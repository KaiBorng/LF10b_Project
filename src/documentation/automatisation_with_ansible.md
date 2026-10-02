Hier ist ein vollständiges Ansible Playbook inklusive Inventar- und Konfigurationsdatei. Es deckt deine gesamte Infrastruktur ab – von der LXD-Installation auf dem Host über das Anlegen der Netzwerke, das Erstellen/Konfigurieren der 3 Container (dhcp, monitoring, client) bis hin zur Installation von dnsmasq, prometheus und prometheus-node-exporter.

Wenn du das Betriebssystem neu installiert hast, musst du lediglich Ansible installieren und dieses Playbook einmal ausführen.

File 1: inventory.ini
Erstelle eine Datei namens inventory.ini auf deinem Steuerungs-PC (oder lokal auf dem Debian-Host):

Ini, TOML
[lxd_hosts]
localhost ansible_connection=local
File 2: setup_environment.yml
Speichere dieses Ansible Playbook als setup_environment.yml:


Anwendung / Ausführung
1. Einmalige Installation von Ansible auf dem frisch installierten Debian-Host:
Bash
sudo apt update
sudo apt install -y ansible
2. Ausführen des Playbooks:
Speichere inventory.ini und setup_environment.yml im selben Ordner und führe folgenden Befehl aus:

Bash
sudo ansible-playbook -i inventory.ini setup_environment.yml
Was das Skript vollautomatisch erledigt:
LXD-Setup: Installiert LXD via Snap, führt lxd init --minimal aus, berechtigt den aktuellen Benutzer und konfiguriert UFW für lxdbr0.

Netzwerk-Setup: Setzt lxdbr0 fest auf 10.10.10.1/24 und erstellt das isolierte dhcpnet-Netzwerk.

Container-Erstellung: Startet dhcp, monitoring und client mit Ubuntu 24.04.

IP- und Interface-Konfiguration:

dhcp: Erhält statische IP 10.10.10.206 auf eth0 sowie eth1 verbunden mit dhcpnet.

monitoring: Erhält statische IP 10.10.10.178 auf eth0.

client: Wird mit eth0 an dhcpnet gehängt.

Dienste-Einrichtung:

In dhcp: Stellt eth1 via Netplan auf 192.168.50.1/24 ein, installiert/konfiguriert dnsmasq als DHCP-Server für den Bereich .100–.200 und richtet den node_exporter ein.

In client: Erstellt die vorgegebene Ordnerstruktur /root/Testordner/file1.txt.

In monitoring: Installiert Prometheus, hinterlegt die prometheus.yml mit Target 10.10.10.206:9100, begrenzt die Datenhaltung (TSDB Retention: 1 Tag / 1 GB) und bindet die Recording- & Alerting-Rules in rules.yml ein. Danach wird das Setup mit promtool validiert und gestartet.
