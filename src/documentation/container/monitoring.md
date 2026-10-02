# Monitoring-Container einrichten
lxc exec monitoring -- bash

## 1 Prometheus installieren
apt update
apt install -y prometheus <br>

## 2 Hauptkonfiguration prometheus.yml
> Trage bei targets die IP des DHCP-Containers ein:

cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
scrape_interval: 60s
evaluation_interval: 60s


rule_files:
- rules.yml
  
scrape_configs:
- job_name: dhcp
  
static_configs:
- targets: ['10.10.10.206:9100'] <br>

### EOF
| Eintrag | Bedeutung |
|---------|-----------|
| scrape_interval | Alle 60 s holt Prometheus die Messwerte ab |
| evaluation_interval | Alle 60 s werden die Regeln ausgewertet |
| rule_files | Verweist auf Dateien mit Regeln (Pfad relativ zur prometheus.yml) |
| scrape_configs | Liste der Ziele: der node_exporter auf Port 9100 im DHCPContainer 
> Was machen rule_files? Die Prometheus-Konfiguration kennt von sich aus nur Ziele und Intervalle <br>
> Alles, was Prometheus aus den Messwerten ableiten soll, steht in Regeldateien. rule_files bindet diese ein. <br>
> Es gibt zwei Arten von Regeln: <br>
> • Recording Rules berechnen eine Kennzahl dauerhaft vor und speichern sie unter einem eigenen kurzen Namen. Statt der langen CPU-Formel fragst du nur dhcp:cpu_percent ab. <br>
> • Alerting Rules prüfen eine Bedingung (z. B. „Ziel ist ausgefallen“). Der Zustand erscheint in der Weboberfläche im Menü Alerts. Eine Benachrichtigung per E-Mail o. Ä. gibt es erst mit dem zusätzlichen Alertmanager. <br>

## 3 Aufbewahrung der Daten begrenzen

cat > /etc/default/prometheus << 'EOF'
ARGS="--storage.tsdb.retention.time=1d --storage.tsdb.retention.size=1GB"

### EOF
Daten älter als 1 Tag werden gelöscht, und die Datenbank wird nie größer als 1 GB. Es gilt, was zuerst erreicht wird. Die Löschung erfolgt blockweise, alte Daten können deshalb noch einige Stunden länger sichtbar sein. <br>

## 4 Regeldatei rules.yml
cat > /etc/prometheus/rules.yml << 'EOF'
groups:
- name: dhcp-kennzahlen
  
### rules:
- record: dhcp:cpu_percent
  
expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
- record: dhcp:ram_percent
  
expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
* 100
- record: dhcp:disk_percent
  
expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} /
node_filesystem_size_bytes{mountpoint="/"}) * 100
- record: dhcp:net_in_bytes
  
expr: rate(node_network_receive_bytes_total{device="eth0"}[5m])
- record: dhcp:net_out_bytes
  
expr: rate(node_network_transmit_bytes_total{device="eth0"}[5m])
- name: dhcp-alarme <br>
  
### rules:
- alert: DhcpContainerDown
expr: up{job="dhcp"} == 0
for: 2m

- alert: DhcpCpuHoch
expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) >
90
for: 5m
- alert: DhcpPlatteVoll

expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} /
node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
for: 10m <br>

### EOF
• record legt eine neue Kennzahl an (CPU, RAM, Platte, Netzwerk ein/aus in %, bzw. Bytes pro Sekunde). <br>
• alert löst aus, wenn expr für die Dauer for wahr ist: Ausfall (2 Min.), CPU über 90 % 5 Min.), Platte über 90 % (10 Min.) <br>

## 5 Prüfen und starten

promtool check config /etc/prometheus/prometheus.yml <br>
systemctl enable prometheus <br>
systemctl restart prometheus <br>
systemctl is-active prometheus <br>
promtool muss SUCCESS für Konfiguration und Regeldatei melden, is-active muss <br>
active ausgeben. Fehler sind meist falsche Einrückungen (nur Leerzeichen, keine Tabs)
