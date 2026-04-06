# LF11Bv2 – Betrieb und Sicherheit vernetzter Systeme gewährleisten

> **🔗 Ausgangsbasis**: Dieses Projekt ist die Fortsetzung von [LF10Bv2](https://github.com/dvrdnz/LF10Bv2/tree/main)

## Ziel dieses Repos

Das Repository dokumentiert Schritt für Schritt den Betrieb und die Absicherung einer vernetzten Systemumgebung - von der Migration auf pfSense über Enforcement bis hin zu Observability - sodass jeder Schritt nachvollziehbar ist und die gesamte Umgebung reproduzierbar bleibt.

## Phasenmodell (chronologisch)

### Phase 0 – Ausgangslage (LF10Bv2)

In LF10Bv2 wurde eine kleine virtuelle Unternehmensumgebung auf einem Windows Host (Windows Server 2019 Datacenter) unter Hyper-V betrieben:

| Komponente    | Rolle                                                 | IP            |
| ------------- | ----------------------------------------------------- | ------------- |
| Debian Router | NAT-Gateway, iptables                                 | 192.168.10.1  |
| Proxmox       | Hypervisor (Nested Virtualization)                    | 192.168.10.10 |
| SRV1          | DNS (bind9), DHCP (Kea)                               | 192.168.10.11 |
| TrueNAS       | Storage (ZFS)                                         | 192.168.10.12 |
| SRV2          | Webserver (Apache2), Nextcloud, Docker, Reverse Proxy | 192.168.10.13 |
| Linux Mint    | Admin-Client                                          | 192.168.10.50 |

SRV1 und SRV2 liefen als VMs in Proxmox. TrueNAS, Debian Router und Linux Mint liefen direkt unter Hyper-V.
Die Firewall-Policy wurde über `scripts/fw_policy.sh` (iptables) auf dem Debian Router verwaltet.

**Netzwerktopologie**

```text
Internet
    │
    │ [WAN]
┌───┴──────────────────────────────────┐
│ Debian Router                  (.1)  │
│ NAT-Gateway · iptables               │
└───┬──────────────────────────────────┘
    │ [Firmennetzwerk]  192.168.10.0/24
    ├── Proxmox              .10
    │     ├── SRV1           .11
    │     └── SRV2           .13
    ├── TrueNAS              .12
    └── Linux Mint           .50
```


### Phase 1 – Migration (docs/00–04)

Debian Router und SRV1 werden durch **pfSense 2.8.1** ersetzt. Ein einzelnes System übernimmt Gateway, Firewall, DHCP und DNS.

| Abgelöste Komponente | Abgelöster Dienst | pfSense-Äquivalent                |
| -------------------- | ----------------- | --------------------------------- |
| Debian Router        | iptables / NAT    | Firewall → Rules & NAT            |
| SRV1                 | Kea DHCPv4        | Services → DHCP Server (Kea)      |
| SRV1                 | bind9             | Services → DNS Resolver (Unbound) |

**Netzwerktopologie**

```text
Internet
    │
    │ [WAN]
┌───┴──────────────────────────────────┐
│ pfSense 2.8.1                  (.2)  │
│ Gateway · Firewall · DHCP · DNS      │
└───┬──────────────────────────────────┘
    │ [Firmennetzwerk]  192.168.10.0/24
    ├── Debian Router        .1    (läuft noch)
    ├── Proxmox              .12   
    │     └── SRV2           .13
    ├── TrueNAS              .11
    └── Linux Mint           .10
```

**Architekturdiagramm**

### Phase 2 – pfSense als SSOT (docs/04–06)

pfSense wird zur **Single Source of Truth (SSOT)** für IP-Vergabe, DNS und NTP.
Dazu werden die abgelösten Systeme stillgelegt und Enforcement-Regeln gesetzt:

* **IP**: Alle bekannten Hosts erhalten Static Mappings; der DHCP-Pool ist auf den definierten Bereich beschränkt.
* **DNS**: Direkte Anfragen an externe Resolver werden geblockt - alle Clients müssen pfSense als Resolver nutzen.
* **NTP**: Hyper-V-Zeitsynchronisierung wird auf allen VMs deaktiviert; pfSense ist die einzige NTP-Quelle.

**Netzwerktopologie**

```text
Internet
    │
    │ [WAN]
┌───┴──────────────────────────────────┐
│ pfSense 2.8.1                  (.2)  │
│ Gateway · Firewall · DHCP · DNS      │
└───┬──────────────────────────────────┘
    │ [Firmennetzwerk]  192.168.10.0/24
    └── Linux Mint           .10
```

**Architekturdiagramm**

```text
Windows Server 2019 Datacenter
└── Hyper-V
    ├── pfSense                → .2   Gateway, Firewall, DHCP, DNS, NTP
    └── Linux Mint             → .10  Admin-Client
```

### Phase 3 – Observability (docs/07–10)

* **Monitoring-VM** (`.20`): Zeitreihen (Prometheus/Grafana) - Wann / Trend / Korrelation
* **Capture-VM** (`.21`): Wire-Level-Debugging (tcpdump) - was genau auf dem Netzwerk passiert
* **Analysis-VM** (`.22`): Aktiver Messpunkt – NTP-Offset, DNS-Antwortzeiten, ICMP-Erreichbarkeit

**Netzwerktopologie**

```text
Internet
    │
    │ [WAN]
┌───┴──────────────────────────────────────────┐
│ pfSense 2.8.1                          (.2)  │
│ Gateway · Firewall · DHCP · DNS · NTP        │
└───┬──────────────────────────────────────────┘
    │ [pcap-br]
┌───┴──────────────────────────────────────────┐
│ Capture-VM  br0                       (.21)  │
│ Inline-Bridge · gesamter LAN-Traffic         │
└───┬──────────────────────────────────────────┘
    │ [Firmennetzwerk]  192.168.10.0/24
    ├── Linux Mint    .10
    ├── Monitoring    .20 
    └── Analysis      .22  

    [monitoring-switch]  10.10.10.0/24  (host-only, isoliert)
    ├── Windows Host  10.10.10.1   Node Exporter :9182
    └── Monitoring    .20          
```

**Architekturdiagramm**

```text
Windows Server 2019 Datacenter
└── Hyper-V
    ├── pfSense                → .2   Gateway, Firewall, DHCP, DNS, NTP
    ├── Linux Mint             → .10  Admin-Client
    ├── Monitoring             → .20  Prometheus, Grafana
    ├── Capture                → .21  Wire-Level-Debugging (tcpdump)
    └── Analysis               → .22  Aktiver Messpunkt (blackbox_exporter, Node Exporter)
```

#### Observability-Modell

Das Observability-Setup folgt einer strikt getrennten Pipeline:

```text
Data Plane → Monitoring → Analysis ↔ Capture
```

* **Monitoring** erkennt Abweichungen anhand von Zeitreihen | läuft immer |
* **Analysis** quantifiziert und untersucht diese Abweichungen gezielt | läuft gezielt |
* **Capture** erklärt sie auf Paketebene | läuft optional/situativ |

## Kapitel

* **Migration**

  * [`00-pfsense.md`](docs/00-pfsense.md) - VM-Setup, Netzwerkkonfiguration, System-Update
  * [`01-firewall.md`](docs/01-firewall.md) - iptables → pfSense Firewall & NAT (inkl. RDP-Portforward)
  * [`02-dhcp.md`](docs/02-dhcp.md) - Kea DHCP (SRV1) → DHCP Server in pfSense
  * [`03-dns.md`](docs/03-dns.md) - bind9 (SRV1) → DNS Resolver (Unbound) in pfSense

* **pfSense als SSOT**

  * [`04-proxmox.md`](docs/04-proxmox.md) - Gateway-Umstellung Proxmox & SRV2, Stilllegung SRV1 + Debian Router
  * [`05-ntp.md`](docs/05-ntp.md) - Chrony ablösen, NTP über pfSense
  * [`06-hardening.md`](docs/06-hardening.md) - Infrastructure Enforcement Policy

* **Observability**

  * [`07-monitoring.md`](docs/07-monitoring.md) - Monitoring (Prometheus, Grafana)
  * [`08-capture.md`](docs/08-capture.md) - Capture-VM
  * [`09-analysis.md`](docs/09-analysis.md) - Analysis-VM
  * [`10-hardening.md`](docs/10-hardening.md) - Infrastructure Enforcement Policy

* **Entscheidungen**

  * [`decisions/01-capture-method.md`](docs/decisions/01-capture-method.md) - ADR-01: Capture-Methode

[`images/`](images/) - Abbildungen und Diagramme

---

## Lizenz

Dieses Repository dient ausschließlich Lern- und Demonstrationszwecken.
