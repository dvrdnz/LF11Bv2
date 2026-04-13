# Teil X: Hardening – Konsistenzdurchsetzung (Nachzug)
 
## Grundkonzept
 
Die Kapitel 07–09 haben die Observability-Schicht funktional aufgebaut:
 
* **MonitoringVM (.20)** – Zeitreihenanalyse mit Prometheus und Grafana
* **CaptureVM (.21)** – Wire-Level-Debugging mit `tcpdump`
* **AnalysisVM (.22)** – aktiver Messpunkt für NTP-Offset, DNS-Antwortzeiten und ICMP-Erreichbarkeit
 
Diese Systeme arbeiten bereits mit klar definierten Rollen und wurden bewusst lokal sowie schrittweise gehärtet. Jedes Kapitel dokumentierte die jeweilige VM als eigenständige, nachvollziehbare Einheit und erfüllte damit den Lernzweck des isolierten Infrastrukturaufbaus.
 
Mit Abschluss dieser Phase bleiben jedoch bewusst einige logische Querverbindungen offen, die sich erst aus der Chronologie des Gesamtaufbaus ergeben: Prometheus benötigt am Ende verifizierbare Identitäten per mTLS, dafür wird eine dedizierte CA-Instanz benötigt; die sichere Verteilung dieses Schlüsselmaterials verlangt einen eindeutig definierten SSH-Verwaltungspfad; die spätere Umstellung auf Hostnamen setzt konsistente DNS-Overrides voraus.
 
Genau diese zusammenhängenden Nachzüge bündelt dieses Kapitel. Es konsolidiert die offenen Punkte des abgeschlossenen Aufbaus und überführt die Observability-Schicht in einen konsistenten Endzustand: ihre Einsatzbereitschaft als vertrauenswürdige Akteursschicht für die Phase 5 – Infrastructure Enforcement by Observability.
 
Die Umsetzung folgt dieser Reihenfolge:
 
* Sicherstellung eines funktionierenden Admin-Zugangs über RDP über VPN vom Schulcomputer direkt zu `Linux Mint`
* Einführung einer **Bastion (.99)** als zentraler und erzwungener SSH-Verwaltungspfad
* Aufbau einer dedizierten **PKI-VM (.199)** zur Erstellung einer CA und zur Zertifikatsausstellung
* Einrichtung konsistenter DNS Host Overrides für hostnamenbasierte Kommunikation
* Verteilung und strukturierte Ablage von Zertifikaten auf allen relevanten Systemen
* Umstellung der Exporter und Blackbox-Komponenten auf mTLS
* Absicherung von pfSense und Prometheus über TLS
* Bereinigung und Finalisierung von Firewall- und DHCP-Konfiguration
 
---
 
## Schritt 1 – Admin-Zugang auf Linux Mint dokumentieren (XRDP)
 
In `01-firewall.md` wurde für TCP-Port 3389 eine Portweiterleitung auf `Linux Mint` eingerichtet. Was dort nicht dokumentiert wurde, ist der dazugehörige RDP-Dienst auf dem Zielsystem selbst.
 
Eine Portweiterleitung stellt ausschließlich den Netzwerkpfad bereit – ohne aktiven RDP-Dienst wird die Verbindung zwar weitergeleitet, dort jedoch nicht angenommen.
 
Damit entsteht eine Lücke in der bisherigen Dokumentation: Wer die Kapitel von vorne durcharbeitet, hätte nach `01-firewall.md` keinen funktionierenden RDP-Zugang auf `Linux Mint` erhalten. Dieser Schritt schließt diese Lücke.
 
Der direkte Pfad `Schulcomputer → VPN → RDP → Linux Mint` ist nicht mehr als Komfortfunktion zu verstehen, sondern als die definierte Admin-Schnittstelle für alle weiteren Schritte dieses Kapitels. Der Pfad `Schulcomputer → VPN → RDP → Windows Server 2019 Datacenter → Hyper-V Manager → Linux Mint` bleibt davon unberührt und erfüllt weiterhin wichtige Eingriffsfunktionen – auch ins Firmennetzwerk.
 
### 1.1 – XFCE und xrdp installieren
 
```bash
sudo apt install -y xrdp xfce4 xfce4-goodies
```
 
> `xrdp` allein liefert keinen Desktop. Ohne Desktop-Umgebung zeigt jede RDP-Verbindung nur einen schwarzen Bildschirm. <y>
 
### 1.2 – XFCE als Session festlegen
 
```bash
echo xfce4-session > ~/.xsession
chmod +x ~/.xsession
```
 
 
### 1.3 – Zugriff auf RDP-Benutzer einschränken
 
```bash
sudo groupadd rdpusers
sudo usermod -aG rdpusers student
```
 
```bash
sudo nano /etc/xrdp/sesman.ini
```
 
```ini
TerminalServerUsers=rdpusers
```
 
> Nur Benutzer in der Gruppe `rdpusers` können sich per RDP anmelden.
 
### 1.4 – xrdp Zugriff auf SSL-Zertifikat erlauben
 
```bash
sudo adduser xrdp ssl-cert
```
 
### 1.5 – Session-Auflösung in `startwm.sh` anpassen
 
```bash
sudo nano /etc/xrdp/startwm.sh
```
 
Die letzten Zeilen ersetzen durch:
 
```bash
# User-defined session
if [ -r ~/.xsession ]; then
  exec ~/.xsession
fi
 
# XFCE fallback
if command -v xfce4-session >/dev/null 2>&1; then
  exec xfce4-session
fi
 
# Default fallback
test -x /etc/X11/Xsession && exec /etc/X11/Xsession
exec /bin/sh /etc/X11/Xsession
```
 
### 1.6 – xrdp aktivieren und starten
 
```bash
sudo systemctl enable xrdp
sudo systemctl restart xrdp
sudo systemctl status xrdp
```
 
Erwartung: `Active: active (running)`