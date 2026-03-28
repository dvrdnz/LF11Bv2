## Teil IV: Proxmox & SRV2 – Gateway-Umstellung und Legacy-Abschaltung

### Ausgangslage

Bis zu diesem Punkt lief die Umgebung noch im Übergangszustand: pfSense ist zwar bereits als Firewall, DHCP- und DNS-Server aktiv, aber Proxmox und SRV2 zeigen noch auf die alten Dienste:

| System | Einstellung | Alter Wert (LF10B) | Neuer Wert (pfSense) |
| --- | --- | --- | --- |
| Proxmox | Gateway | `192.168.10.1` (Debian Router) | `192.168.10.2` (pfSense LAN) |
| Proxmox | DNS-Server | `192.168.10.11` (SRV1 / bind9) | `192.168.10.2` (pfSense / Unbound) |
| SRV2 | Gateway | `192.168.10.1` (Debian Router) | `192.168.10.2` (pfSense LAN) |
| SRV2 | DNS-Server | `192.168.10.11` (SRV1 / bind9) | `192.168.10.2` (pfSense / Unbound) |

Erst nach der Umstellung beider Systeme können SRV1 und der Debian Router sauber abgeschaltet werden.

---

### Schritt 1 – Proxmox: Gateway ändern

Die Netzwerkkonfiguration des Proxmox-Hosts erfolgt entweder über die WebGUI oder direkt in `/etc/network/interfaces`.

#### Option A – WebGUI

**Node → System → Network** → Bridge auswählen (üblicherweise `vmbr0`) → **Edit**

| Feld | Alter Wert | Neuer Wert |
| --- | --- | --- |
| Gateway (IPv4) | `192.168.10.1` | `192.168.10.2` |

→ **OK** → **Apply Configuration**

> ⚠️ Proxmox wendet Netzwerkänderungen erst nach Klick auf **Apply Configuration** an. Der Button erscheint oben in der Netzwerkliste, sobald ungespeicherte Änderungen vorliegen.

#### Option B – CLI

```bash
nano /etc/network/interfaces
```

Relevanter Abschnitt vorher:

```
iface vmbr0 inet static
    address  192.168.10.12/24
    gateway  192.168.10.1
    bridge-ports ens18
    bridge-stp off
    bridge-fd 0
```

Nach der Anpassung:

```
iface vmbr0 inet static
    address  192.168.10.12/24
    gateway  192.168.10.2
    bridge-ports ens18
    bridge-stp off
    bridge-fd 0
```

Änderung anwenden:

```bash
sudo systemctl restart networking
```

---

### Schritt 2 – Proxmox: DNS-Server ändern

**Node → System → DNS**

| Feld | Alter Wert | Neuer Wert |
| --- | --- | --- |
| DNS server 1 | `192.168.10.11` | `192.168.10.2` |
| Search domain | `example.internal` | `example.internal` |

→ **OK**

Die Änderung wirkt sofort – kein Neustart nötig. Intern schreibt Proxmox die Einstellung in `/etc/resolv.conf`:

```
search example.internal
nameserver 192.168.10.2
```

---

### Schritt 3 – Host Override in pfSense für Proxmox anlegen

Host Override im DNS Resolver eingtragen:

**Services → DNS Resolver → Host Overrides → + Add**

| Feld | Wert |
| --- | --- |
| Host | `proxmox` |
| Domain | `example.internal` |
| IP Address | `192.168.10.12` |
| Description | proxmox |

→ **Save** → **Apply Changes**

---

### Schritt 4 – Funktionsnachweis Proxmox

Auf dem Proxmox-Host (Shell: **Node → Shell**) testen:

```bash
nslookup google.com pfsense.example.internal
```

Erwartete Ausgabe:

```
Server:         pfsense.example.internal
Address:        192.168.10.2#53

Non-authoritative answer:
Name:   google.com
Address: 172.217.16.174
...
```



Zusätzlich ist die Proxmox-WebGUI über den internen Hostnamen erreichbar:

```
https://proxmox.example.internal:8006/
```

> Der Aufruf im Browser beweist, dass `proxmox.example.internal → 192.168.10.12` korrekt aufgelöst wird.



---

### Schritt 5 – SRV2: Gateway und DNS umstellen

SRV2 (`192.168.10.13`) ist eine Debian-VM in Proxmox mit Interface `ens18`. Die Netzwerkkonfiguration erfolgt klassisch über `/etc/network/interfaces`. DNS wird über /etc/resolv.conf` geändert.

#### 5.1 Gateway und DNS ändern

```bash
nano /etc/resolv.conf
```

```
# vorher
nameserver 192.168.10.12

# nachher
nameserver 192.168.10.2
```

```bash
nano /etc/network/interfaces
```

Gateway-Zeile anpassen:

```
# vorher
gateway 192.168.10.1

# nachher
gateway 192.168.10.2
```



Anwenden:

```bash
sudo systemctl restart networking
```


Prüfen:

```bash
sudo nslookup google.com pfsense.exapmle.internal
```
![Gateway-Umstellung SRV2](/images/img_33.png)




---

### Schritt 6 – TrueNAS-VM umstellen

Die Umstellung von TrueNAS erfolgt direkt in der VM im Hyper-V-Manager über dessem eigenes Konsolen-Menü zur Netzwerkkonfiguration.

#### 6.1 – Netzwerkinterface zurücksetzen

Im Konsolen-Menü Option **1) Configure Network Interfaces** wählen:

```
Select an interface (q to quit): 1   ← hn0 auswählen
Remove the current settings of this interface? (y/n) y
```

TrueNAS entfernt die alte Konfiguration und startet Netzwerk und Routing neu:

```
Removing interface configuration: Ok
Restarting network: ok
Restarting routing: ok
```

![Netzwerkinterface zurücksetzen](/images/img_35.png)

#### 6.2 – Standard-Gateway setzen

Im Konsolen-Menü Option **4) Configure Default Route** wählen:

```
Configure IPv4 Default Route? (y/n) y
IPv4 Default Route: 192.168.10.2
Saving IPv4 gateway: Ok
Configure IPv6 Default Route? (y/n) n
Restarting routing: ok
```

![Standard-Gateway setzen](/images/img_37.png)

#### 6.3 – DNS-Server setzen

Im Konsolen-Menü Option **6) Configure DNS** wählen:

```
DNS Domain [local]: example.internal
DNS Nameserver 1: 192.168.10.2
DNS Nameserver 2:              ← leer lassen, Enter
Saving DNS configuration: ok
Reloading network config: ok
```

![DNS-Server setzen](/images/img_36.png)

Nach diesen drei Schritten verwendet TrueNAS als Gateway und DNS-Server jeweils `192.168.10.2` (pfSense) – analog zur Umstellung von Proxmox und SRV2.

---

### Schritt 7 – Host Overrides

Da SRV2 einen Reverse Proxy betreibt, bekommt jeder Dienst einen eigenen DNS-Eintrag – alle zeigen auf `192.168.10.13`. Der Reverse Proxy routet anhand des Hostnamens.

**Services → DNS Resolver → Host Overrides → + Add**

| Host | Domain | IP Address | Description |
| --- | --- | --- | --- |
| `portainer` | `example.internal` | `192.168.10.13` | Portainer |
| `nextcloud` | `example.internal` | `192.168.10.13` | Nextcloud |
| `storage` | `example.internal` | `192.168.10.13` | TrueNAS |

Jeden Eintrag einzeln anlegen: → **Save** → abschließend **Apply Changes**

![Host Overrides pfSense](/images/img_34.png)

---

### Schritt 8 – SRV1 herunterfahren

DHCP (Kea) und DNS (bind9) laufen vollständig auf pfSense. Proxmox und SRV2 nutzen die neuen Dienste. SRV1 kann abgeschaltet werden.

**Proxmox WebGUI → SRV1 (VM auswählen) → Shutdown**

Autostart deaktivieren:

**SRV1 → Options → Start at boot → No**

---

### Schritt 9 – Debian Router herunterfahren

Der Debian Router läuft als VM in **Hyper-V** (nicht in Proxmox) und ist durch pfSense vollständig abgelöst (NAT, iptables, Gateway). Er wird über den Hyper-V Manager abgeschaltet.

**Hyper-V Manager → VM auswählen → Shut Down**

Damit der Debian Router nach einem Host-Neustart nicht automatisch startet:

**Hyper-V Manager → VM → Settings → Automatic Start Action → Nothing**

---

