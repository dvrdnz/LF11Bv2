# LF11Bv2 – Betrieb und Sicherheit vernetzter Systeme gewährleisten

> Fortsetzung von [LF10Bv2](https://github.com/dvrdnz/LF10Bv2)

## Ausgangslage

In LF10Bv2 wurde eine kleine virtuelle Unternehmensumgebung auf einem Windows Host (Windows Server 2019 Datacenter) unter Hyper-V betrieben:

| Komponente    | Rolle                                                        | IP            |
| ------------- | ------------------------------------------------------------ | ------------- |
| Debian Router | NAT-Gateway, iptables                                        | 192.168.10.1  |
| Proxmox       | Hypervisor (Nested Virtualization)                           | 192.168.10.10 |
| SRV1          | DNS (bind9), DHCP (Kea)                                      | 192.168.10.11 |
| TrueNAS       | Storage (ZFS)                                                | 192.168.10.12 |
| SRV2          | Webserver (Apache2), Nextcloud, Docker, Reverse Proxy        | 192.168.10.13 |
| Linux Mint    | Admin-Client                                                 | 192.168.10.50 |

SRV1 und SRV2 liefen als VMs in Proxmox. TrueNAS, Debian Router und Linux Mint liefen direkt unter Hyper-V.

Die Firewall-Policy wurde über `scripts/fw_policy.sh` (iptables) auf dem Debian Router verwaltet.

## Migration

Die Netzwerkdienste von Debian Router und SRV1 werden auf **pfSense 2.8.1** konsolidiert – ein einzelnes System übernimmt Gateway, Firewall, DHCP und DNS. Debian Router und SRV1 entfallen.

Proxmox, TrueNAS und SRV2 bleiben erhalten, werden aber auf Gateway und DNS-Server umgestellt.

| Abgelöste Komponente | Abgelöster Dienst        | pfSense-Äquivalent           |
| -------------------- | ------------------------ | ---------------------------- |
| Debian Router        | iptables / NAT           | Firewall → Rules & NAT       |
| SRV1                 | Kea DHCPv4               | Services → DHCP Server (Kea) |
| SRV1                 | bind9 | DNS Resolver (Unbound)       |

## Netzwerkdiagramm

```
Internet
    │
   WAN (DHCP – 10.100.15.x)
┌─────────────┐
│   pfSense   │ 192.168.10.2  ← Gateway, Firewall, DHCP, DNS
└─────────────┘
   LAN  192.168.10.0/24
    │
    ├── .2         pfSense  (Gateway, DNS-Resolver)
    ├── .10        Admin Client   (Linux Mint – statische DHCP-Zuweisung)
    ├── .11        TrueNAS  (Storage)
    ├── .12        Proxmox  (Hypervisor)
    ├── .13        SRV2     (Reverse Proxy, Docker)
    └── .11–.245   DHCP-Pool
```

## Architekturdiagramm

```
Physical Host (Remote Lab – Windows Server 2019 Datacenter)
│
└── Hyper-V
    │
    ├── pfSense VM          → Gateway, Firewall, DHCP, DNS
    │
    ├── Proxmox VM          → Hypervisor (Nested Virtualization)
    │      │
    │      └── SRV2         → Reverse Proxy, Docker
    │
    ├── TrueNAS VM          → Storage (ZFS)
    │
    └── Linux Mint VM       → Admin-Client
```

## Technologien

* pfSense 2.8.1 (FreeBSD 15.0-CURRENT)
* Kea DHCPv4
* Unbound (DNS Resolver)

## Kapitel

- [`00-pfsense.md`](docs/00-pfsense.md) – VM-Setup, Netzwerkkonfiguration, System-Update
- [`01-firewall.md`](docs/01-firewall.md) – iptables → pfSense Firewall & NAT (inkl. RDP-Portforward)
- [`02-dhcp.md`](docs/02-dhcp.md) – Kea DHCP (SRV1) → DHCP Server in pfSense
- [`03-dns.md`](docs/03-dns.md) – bind9 (SRV1) → DNS Resolver (Unbound) in pfSense
- [`04-proxmox.md`](docs/04-proxmox.md) – Gateway-Umstellung Proxmox & SRV2, Legacy-Abschaltung (SRV1, Debian Router)
- [`05-ntp.md`](docs/05-ntp.md) – Chrony ablösen, NTP über pfSense

- [`images/`](images/) – Abbildungen und Diagramme

---


## Lizenz

Dieses Repository dient ausschließlich Lern- und Demonstrationszwecken.
