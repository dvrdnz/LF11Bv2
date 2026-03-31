## Teil VII: Monitoring – Prometheus & Grafana

### Grundkonzept

Monitoring misst – aber was es misst ist nur so verlässlich wie die Umgebung in der es läuft. Die in `06-hardening.md` durchgesetzte Source of Truth ist keine Sicherheitsmaßnahme, sondern die Voraussetzung dafür, dass Monitoring-Daten überhaupt verwertbar sind:

- **Zeitstempel**: Prometheus und Grafana arbeiten mit Zeitreihen. Wenn Systeme unterschiedliche NTP-Quellen nutzen, entstehen Zeitversätze zwischen Metriken – Korrelationen werden unmöglich, Anomalien sehen aus wie Normalbetrieb.
- **Namensauflösung**: Node Exporter-Targets werden per Hostname oder IP angesprochen. Wenn DNS nicht zentral kontrolliert ist, kann ein Target unter verschiedenen Namen erreichbar sein – oder gar nicht.
- **IP-Stabilität**: Prometheus speichert Metriken pro Target-IP. Wenn eine IP wechselt weil kein Static Mapping gesetzt ist, entstehen Datenlücken und doppelte Zeitreihen.

---

### Schritt 1 – Monitoring-VM aufsetzen

**Hyper-V → Neue VM erstellen**

| Feld | Wert |
| --- | --- |
| Name | monitoring |
| OS | Ubuntu Server (LTS) |
| RAM | 2 GB |
| CPU | 2 vCPU |
| Disk | 20 GB |
| Netzwerk | Internes Netz (wie alle anderen VMs) |

Nach der Installation: statische DHCP-Zuweisung in pfSense unter **Services → DHCP Server → LAN → Static Mappings** (MAC → `192.168.10.20`).

---

### Schritt 2 – Basis-Setup

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install curl wget gnupg2 -y
```

---

### Schritt 3 – Prometheus installieren

Dedicated User anlegen:

```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

Version 3.10.0 herunterladen und entpacken:

```bash
wget https://github.com/prometheus/prometheus/releases/download/v3.10.0/prometheus-3.10.0.linux-amd64.tar.gz
tar xvf prometheus-3.10.0.linux-amd64.tar.gz
cd prometheus-3.10.0.linux-amd64/
```

Binaries und Konfigurationsverzeichnisse einrichten:

```bash
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus

sudo cp prometheus promtool /usr/local/bin/

sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

> Ab Prometheus 3.x sind `consoles` und `console_libraries` nicht mehr im Release-Tarball enthalten – der entsprechende `cp`-Befehl entfällt.

---

### Schritt 4 – Prometheus konfigurieren

```bash
sudo nano /etc/prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 5s

scrape_configs:
  - job_name: 'monitoring'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'nodes'
    static_configs:
      - targets:
        - 192.168.10.2:9100    # pfsense
        - 192.168.10.10:9100   # Mint
        - 192.168.10.20:9100   # Monitoring (self)
```

```bash
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

---

### Schritt 5 – Prometheus als systemd-Service einrichten

```bash
sudo nano /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

Funktionsnachweis:

```bash
sudo systemctl status prometheus
```
![DNS-Enforcement validiert](/images/img_53.png)

Erreichbarkeit prüfen: `http://192.168.10.20:9090`

---

### Schritt 6 – Grafana installieren

```bash
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://packages.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt update
sudo apt install grafana -y

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Erreichbarkeit prüfen: `http://192.168.10.20:3000`

Login: `admin / admin` → Passwort ändern.

---

### Schritt 7 – Prometheus als Datenquelle in Grafana einbinden

**Connections → Data Sources → Add data source → Prometheus**

![DNS-Enforcement validiert](/images/img_54.png)

![DNS-Enforcement validiert](/images/img_55.png)
| Feld | Wert |
| --- | --- |
| Type | Prometheus |
| URL | `http://localhost:9090` |

→ **Save & Test**

---

### Schritt 8 – Node Exporter auf allen VMs installieren

Auf jeder VM (`mint`, `monitoring`):

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvf node_exporter-1.10.2.linux-amd64.tar.gz
cd node_exporter-1.10.2.linux-amd64 
sudo cp node_exporter /usr/local/bin/
```

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

Funktionsnachweis:

```
http://<VM-IP>:9100/metrics
```

---


### Schritt 9 – Node Exporter auf pfSense

pfSense läuft auf FreeBSD – der Linux Node Exporter ist nicht kompatibel. pfSense bringt ein eigenes Paket mit.

**System → Package Manager → Available Packages → `node_exporter` → Install**

Node Exporter als Service aktivieren – über **Diagnostics → Command Prompt**:

![DNS-Enforcement validiert](/images/img_58.png)

```bash
sysrc node_exporter_enable="YES"
service node_exporter start
```

Unter **Status → Services** muss `node_exporter` als aktiv erscheinen.

#### Firewall-Regel für Port 9100

LAN-zu-LAN Traffic läuft direkt über den Switch – pfSense sieht ihn nicht. Traffic der pfSense selbst als Ziel hat wird jedoch von pfSense selbst verarbeitet und braucht eine explizite Pass-Regel.

**Firewall → Rules → LAN → ↑ Add**

| Feld | Wert |
| --- | --- |
| Action | Pass |
| Interface | LAN |
| Protocol | TCP |
| Source | `192.168.10.20` |
| Destination | This Firewall (self) |
| Destination Port | 9100 |
| Description | Allow Monitoring → pfSense Node Exporter |

→ **Save** → **Apply Changes**

Funktionsnachweis von der Monitoring-VM:

```bash
curl http://192.168.10.2:9100/metrics
```

#### Sicherheitsarchitektur

Die Monitoring-VM hat damit direkten Zugriff auf pfSense selbst. Das ist bewusst so – Prometheus muss aktiv zu allen Hosts verbinden, das ist seine Aufgabe. Die Monitoring-VM ist ein dedizierter vertrauenswürdiger Management-Host, der Zugriff ist funktional notwendig und architektonisch akzeptiert.

Konsequenz: die Monitoring-VM wird auf das Extremste gehärtet. Ubuntu Server wurde bereits minimal installiert – kein Desktop, keine unnötigen Dienste. Ausgehender Internet-Traffic der Monitoring-VM wird in pfSense blockiert: nach der Installation sind alle Targets intern, Internet-Zugang ist nicht notwendig.

**Firewall → Rules → LAN → ↑ Add**

| Feld | Wert |
| --- | --- |
| Action | Block |
| Interface | LAN |
| Protocol | any |
| Source | `192.168.10.20` |
| Destination | `!192.168.10.0/24` |
| Destination Port | any |
| Description | Block Monitoring-VM → Internet |

→ **Save** → **Apply Changes**

> Hinweis: Da `Protocol` auf `any` steht, ist hier kein spezieller Port notwendig; die Regel blockiert den gesamten ausgehenden Verkehr der Monitoring-VM ins Nicht-LAN.


---

### Schritt 10 – Windows Exporter auf dem Hyper-V Host

Statt einer Firewall-Regel wird ein dedizierter interner vSwitch zwischen Monitoring-VM und Windows Host eingerichtet. Der Windows Host ist damit nicht ins LAN exponiert – die Verbindung bleibt isoliert.

#### 10.1 – vSwitch anlegen

Auf dem Hyper-V Host (PowerShell als Administrator):

```powershell
New-VMSwitch -Name "monitoring-switch" -SwitchType Internal
```

Der neue vSwitch bekommt automatisch einen Netzwerkadapter auf dem Host. Diesem eine statische IP zuweisen:

```powershell
New-NetIPAddress -InterfaceAlias "vEthernet (monitoring-switch)" -IPAddress 10.10.10.1 -PrefixLength 24
```

#### 10.2 – Monitoring-VM mit vSwitch verbinden

```powershell
Add-VMNetworkAdapter -VMName "MonitoringVM" -SwitchName "monitoring-switch"
```

Auf der Monitoring-VM die neue Netzwerkschnittstelle konfigurieren:

```bash
sudo nano /etc/netplan/01-monitoring-switch.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth1:
      addresses:
        - 10.10.10.2/24
```

Berechtigungen setzen und anwenden:

```bash
sudo chmod 600 /etc/netplan/01-monitoring-switch.yaml
sudo netplan apply
```

#### 10.3 – Windows Exporter installieren

Version 0.31.5 herunterladen:
`https://github.com/prometheus-community/windows_exporter/releases/download/v0.31.5/windows_exporter-0.31.5-amd64.msi`

Im Setup-Wizard:

![DNS-Enforcement validiert](/images/img_56.png)

**Firewall Exception:** im MSI-Wizard deaktivieren – Firewall-Regeln werden manuell und interface-spezifisch angelegt.

**Collectors:** `cpu,logical_disk,net,os,system,hyperv`

![DNS-Enforcement validiert](/images/img_57.png)

| Collector | Metriken |
| --- | --- |
| `cpu` | CPU-Auslastung pro Core + gesamt – Grundlage für Spikes und Bottlenecks |
| `logical_disk` | I/O pro Laufwerk (Read/Write, Queue Length) – Storage-Bottlenecks |
| `net` | Netzwerktraffic + Errors pro Interface – VM ↔ Host ↔ Netzwerk Analyse |
| `os` | Uptime, Prozesse, grundlegender Systemzustand |
| `system` | Threads und Prozesse gesamt auf Systemebene |
| `hyperv` | VM-spezifische Metriken direkt vom Hypervisor |


**Port:** 9182

#### 10.4 – Windows Firewall Regeln anlegen

Die Windows Firewall blockiert standardmäßig eingehenden Traffic – auch auf dem internen vSwitch. Regeln explizit auf `vEthernet (monitoring-switch)` beschränken, damit Port 9182 nicht auf allen Interfaces geöffnet wird.

```powershell
New-NetFirewallRule -DisplayName "monitoring-switch ICMP" -Direction Inbound -Protocol ICMPv4 -Action Allow -InterfaceAlias "vEthernet (monitoring-switch)"

New-NetFirewallRule -DisplayName "monitoring-switch 9182" -Direction Inbound -Protocol TCP -LocalPort 9182 -Action Allow -InterfaceAlias "vEthernet (monitoring-switch)"
```

> Beide Regeln sind auf `vEthernet (monitoring-switch)` beschränkt – Port 9182 ist damit ausschließlich über den dedizierten Monitoring-Switch erreichbar, nicht über LAN oder WAN.

#### 10.5 – Funktionsnachweis

Von der Monitoring-VM:

```bash
ping 10.10.10.1
curl http://10.10.10.1:9182/metrics
```
![DNS-Enforcement validiert](/images/img_60.png)
#### 10.6 – Target in prometheus.yml ergänzen

```yaml
- 10.10.10.1:9182   # Windows Host (Hyper-V)
```

```bash
sudo systemctl restart prometheus
```

---

> Hinweis: Dieses Kapitel ist aktuell noch in Arbeit und beschreibt den geplanten Monitoring-Aufbau. Weitere Anpassungen und Detaillierungen folgen.
