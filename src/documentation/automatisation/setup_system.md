# 1. Ansible installieren und Container erstellen
`sudo su -` <br>
`apt update && apt upgrade && apt autoremove` <br>
`apt install -y ansible` <br>
der Ordner muss heruntergeladen werden mit folgenden Dateien [1](url), [2](url)<br>
`sudo ansible-playbook -i inventory.ini setup_environment.yml` <br>

## Was das Skript vollautomatisch erledigt:
- installiert LXD via Snap <br>
- führt `lxd init --minimal` aus <br>
> berechtigt den aktuellen Benutzer und konfiguriert UFW für lxdbr0.

Netzwerk-Setup: Setzt `lxdbr0` fest auf `10.10.10.1/24` und erstellt das isolierte `dhcpnet-Netzwerk`.<br>

## 1. Container-Erstellung:
> startet dhcp, monitoring und client mit Ubuntu 24.04.

**IP- und Interface-Konfiguration:** <br>
dhcp: erhält statische `IP 10.10.10.206` auf `eth0` sowie `eth1` verbunden mit `dhcpnet` <br>
monitoring: Erhält statische `IP 10.10.10.178` auf `eth0` <br>
client: Wird mit `eth0` an `dhcpnet` gehängt <br>

## 2. Dienste-Einrichtung:
**In dhcp:** <br>
- stellt `eth1` via Netplan auf `192.168.50.1/24` ein, <br>
- installiert/konfiguriert `dnsmasq` als DHCP-Server für den Bereich `.100–.200` <br>
- richtet den `node_exporter` ein <br>

**In client:** <br>
- erstellt die vorgegebene Ordnerstruktur `/root/Testordner/file1.txt`

**In monitoring:** <br>
- installiert Prometheus <br>
- hinterlegt die `prometheus.yml` mit Target 10.10.10.206:9100  <br>
- begrenzt die Datenhaltung (TSDB Retention: 1 Tag / 1 GB) <br>
- bindet die Recording- & Alerting-Rules in `rules.yml` ein <br>
- startet Setup mit `promtool` und validiert
