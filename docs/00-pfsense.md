## Teil 0: pfSense Router

### VM-Konfiguration (Hyper-V)

| Parameter              | Wert                              |
| ---------------------- | --------------------------------- |
| VM-Name                | pfsense router                    |
| Arbeitsspeicher        | 1024 MB                           |
| Prozessoren            | 1                                 |
| Netzwerkkarte WAN      | R-LAB Internet    |
| Netzwerkkarte LAN      | Firmennetzwerk                    |


---

### Netzwerkkonfiguration

| Interface | NIC  | Typ    | Adresse           |
| --------- | ---- | ------ | ----------------- |
| WAN       | hn0  | DHCP4  | WAN-IP   |
| LAN       | hn1  | Statisch | 192.168.10.2/24 |

WebConfigurator: `https://192.168.10.2/`

---

### Software & System

| Parameter  | Wert                                                     |
| ---------- | -------------------------------------------------------- |
| OS         | pfSense 2.7.2-RELEASE (amd64)                            |
| Basis      | FreeBSD 14.0-CURRENT                                     |
| Build      | Wed Dec 6 21:10:00 CET 2023                              |
| Hostname   | pfSense.example.internal                                     |
| BIOS       | Microsoft Corporation – Hyper-V UEFI v4.1 (20.06.2024)  |
| DNS-Server | 127.0.0.1, 1.1.1.1, 8.8.8.8                |

---

### WebConfigurator

Erreichbar unter `https://192.168.10.2/` – Zertifikatswarnung erscheint (selbstsigniert, erwartet).

![pfSense Dashboard – System Information](/images/img_19.png)

---

### Verfügbares System-Update

Im Dashboard wird unter **Version** ein verfügbares Update auf **2.8.1** angezeigt. Das Update kann unter **System → Update → System Update** eingespielt werden.

| Parameter           | Wert  |
| ------------------- | ----- |
| Current Base System | 2.7.2 |
| Latest Base System  | 2.8.1 |
| Branch              | Current Stable Version (2.8.1) |

Systeminformationen nach Update:


| Parameter  | Wert                                                     |
| ---------- | -------------------------------------------------------- |
| OS         | 2.8.1-RELEASE (amd64)                         |
| Basis      | FreeBSD 15.0-CURRENT                                     |
| Build      | built on Mon Dec 15 18:31:00 CET 2025                             |


![System Update – Confirmation](/images/img_20.png)

---

### WAN-Interface prüfen

Falls im Dashboard **keine Update-Empfehlung** erscheint, hat pfSense möglicherweise keine WAN-Verbindung. In diesem Fall prüfen, ob das WAN-Interface korrekt konfiguriert ist:

**Interfaces → WAN → IPv4 Configuration Type: DHCP**


![WAN Interface – IPv4 Configuration Type DHCP](/images/img_18.png)

---
