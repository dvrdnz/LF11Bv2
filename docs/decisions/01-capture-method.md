## ADR-01 – Capture-Methode für Wire-Level-Debugging

**Datum:** 04.04.2026  
**Status:** Entschieden  
**Betrifft:** `08-capture.md`, Phase 3 Observability

---

### Kontext

Die Observability-Pipeline (Kapitel 07-08) soll neben Zeitreihen-Monitoring auch Wire-Level-Debugging ermöglichen: Paketinhalt, Protokollverhalten, Antwortausbleiben. Dafür muss die Capture-VM Fremdtraffic sehen können - Traffic, der nicht an sie adressiert ist.

Die Umgebung ist eine **verschachtelte Hyper-V-Umgebung** (Hyper-V unter Windows Server 2019 Datacenter). Diese Randbedingung schränkt die verfügbaren Optionen erheblich ein.

---

### Entscheidungskriterien

Für die Auswahl der Capture-Methode wurden folgende Kriterien priorisiert:

- Muss in verschachteltem Hyper-V zuverlässig funktionieren
- Vollständige L2-Sichtbarkeit inklusive MAC- und Frame-Informationen
- Deterministisches und reproduzierbares Verhalten
- Kein dauerhafter Produktivbetrieb erforderlich
- Temporärer Single Point of Failure während Debugging ist akzeptabel

---

### Evaluierte Optionen

#### Option 1 – Hyper-V Port Mirroring

**Ansatz:** Hyper-V bietet `Set-VMNetworkAdapter -PortMirroring Source/Destination` als native Funktion. Die Capture-VM würde als Destination konfiguriert und erhält gespiegelten Traffic passiv - ohne im Pfad zu liegen.

**Ergebnis: Verworfen.**

Der Befehl ließ sich zwar ohne Fehlermeldung ausführen:

```powershell
Set-VMNetworkAdapter -VMName "pfsense router" -Name "LAN" -PortMirroring Source
Set-VMNetworkAdapter -VMName "CaptureVM"       -Name "LAN" -PortMirroring Destination
````

Eine anschließende Validierung zeigte jedoch, dass die Konfiguration in der verschachtelten Hyper-V-Umgebung keine belastbare Wirkung hatte. Port Mirroring wird in diesem Szenario durch den Host-Hypervisor nicht zuverlässig weitergereicht. Das Verhalten ist eine bekannte Einschränkung nested Hyper-V. Die Methode ist deshalb für diese Umgebung nicht belastbar nutzbar.

---

#### Option 2 – pfSense `dup-to` (pf-Regelwerk)

**Ansatz:** FreeBSD pf unterstützt `dup-to`: Jedes matchende Paket wird dupliziert und eine Kopie an die Capture-VM geschickt. Das Original läuft unverändert weiter - die Capture-VM liegt nicht im Pfad.

```text
pass in on em1 all dup-to (em2, 192.168.10.21)
```

**Ergebnis: Verworfen.**

Die Option scheitert an der fachlichen Anforderung: `dup-to` arbeitet auf Layer 3. Für Wire-Level-Debugging wird hier jedoch Layer-2-Sichtbarkeit benötigt, also MAC-Adressen, Frame-Struktur und die vollständige Sicht auf das tatsächliche Durchleiten zwischen den Segmenten. Genau diese Informationen gehen bei `dup-to` verloren. Die Methode ist daher für dieses Projekt nicht ausreichend.

---

### Entscheidung – Option 3: Inline-Capture via transparente Bridge

**Implementierung:** `08-capture.md`

Die Capture-VM bridget zwei Hyper-V-vSwitches (`Firmennetzwerk` ↔ `pcap-br`) über eine Linux-Bridge (`br0`). pfSense LAN-NIC liegt auf `pcap-br`. Der gesamte Traffic zwischen LAN-Clients und pfSense fließt physisch durch `br0` und ist mit `tcpdump -i br0` vollständig sichtbar - inklusive L2.

```text
LAN-Clients (Firmennetzwerk) ↔ CaptureVM (br0) ↔ pcap-br ↔ pfSense LAN
```

**Begründung:**

* Funktioniert in verschachtelten Hyper-V-Umgebungen ohne Einschränkung
* Keine Abhängigkeit von Hypervisor-Features
* Vollständige L2/L3/L4-Sichtbarkeit
* Deterministisch und reproduzierbar
* Kein dauerhafter Betrieb - Capture nur bei Anlass

Ein Dauerbetrieb wurde bewusst verworfen, da die Bridge im Fehlerfall zum Single Point of Failure für das gesamte LAN-Segment wird. Maßnahmen zu Redundanz, Failover, Watchdog oder Rollback sind **nicht Teil dieser Entscheidung** und werden separat dokumentiert.

---

### Bekannter Tradeoff: Kritischer Pfad

In einer verschachtelten Hyper-V-Umgebung ist eine transparente Capture-Bridge kein ausfallsicherer Dienst, sondern ein kritischer Layer-2-Pfad. Fällt die Capture-VM aus, ist der Verkehr zwischen den Segmenten unterbrochen; DHCP/DNS hinter der Bridge brechen entsprechend weg. Eine echte Redundanz ist ohne sauberes Loop-/STP-Design nicht sinnvoll abbildbar.

Dieser Tradeoff ist bewusst akzeptiert. Die Capture-VM ist kein Produktivsystem - sie läuft passiv und wird nur bei gezieltem Debugging-Bedarf aktiviert. Das Ausfallrisiko ist entsprechend gering.

---

### Notfall-Workaround und Rückbau

Fällt die Capture-VM während einer Analyse aus oder muss neu gestartet werden, kann die pfSense-LAN-NIC direkt auf den produktiven vSwitch `Firmennetzwerk` zurückgeschaltet werden.

**Service Restore (Bypass):**

```powershell
$nic = Get-VMNetworkAdapter -VMName "pfsense router" |
    Where-Object SwitchName -eq "pcap-br"

Connect-VMNetworkAdapter -VMNetworkAdapter $nic -SwitchName "Firmennetzwerk"
```

Damit ist die LAN-Konnektivität kurzfristig vollständig wiederhergestellt.

**Rückbau nach Wiederherstellung der Capture-VM:**

Nach erfolgreichem Neustart der Bridge-VM wird die pfSense-LAN-NIC wieder auf den dedizierten Capture-vSwitch `pcap-br` gelegt.

```powershell
Connect-VMNetworkAdapter -VMNetworkAdapter $nic -SwitchName "pcap-br"
```

Damit wird der definierte Soll-Zustand der Observability-Architektur wiederhergestellt.

---

### Zusammenfassung

| Option                    | Ergebnis    | Hauptgrund                                                    |
| ------------------------- | ----------- | ------------------------------------------------------------- |
| Hyper-V Port Mirroring    | Verworfen   | In verschachtelter Umgebung nicht zuverlässig funktionsfähig  |
| pfSense `dup-to`          | Verworfen   | Nur Layer 3, aber Layer-2-Sichtbarkeit erforderlich           |
| **Inline-Bridge (`br0`)** | **Gewählt** | Nested-tauglich, vollständige L2-Sichtbarkeit, reproduzierbar |
