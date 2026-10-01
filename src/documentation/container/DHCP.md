# DHCP-Container vorbereiten

## 1. node_exporter installieren <br>
lxc exec dhcp -- apt update <br>
lxc exec dhcp -- apt install -y prometheus-node-exporter <br>
lxc exec dhcp -- systemctl is-active prometheus-node-exporter <br>
> Wozu? Prometheus misst nicht selbst, sondern holt Werte von Programmen ab. <br>
Der node_exporter stellt Systemwerte des Containers (CPU, RAM, Platte, Netzwerk) als Text auf Port 9100 bereit <br> Prometheus ruft diese Seite alle 60 Sekunden ab. <br>

### Test vom Monitoring-Container aus:
lxc exec monitoring -- curl -s http://10.10.10.206:9100/metrics | head -5 <br>
> Es müssen Zeilen mit # HELP ... erscheinen. Der node_exporter überwacht den Container, nicht den DHCP-Dienst selbst.

## 2. Zweites Netz dhcpnet anlegen und als eth1 anhängen
lxc network create dhcpnet ipv4.address=none ipv6.address=none <br>
lxc config device add dhcp eth1 nic network=dhcpnet name=eth1 <br>
lxc exec dhcp -- ip -br a <br>
In der Ausgabe muss eth1 (Status UP, noch ohne IPv4) neben eth0 stehen. Der Container muss dafür laufen. <br>

## 3. Feste IP auf eth1 per Netplan
lxc exec dhcp -- bash <br>
Im Container (Prompt root@dhcp:~#): <br>
cat > /etc/netplan/60-dhcpnet.yaml << 'EOF' <br>
network: <br>
version: 2 <br>
ethernets: <br>
eth1: <br>
addresses: [192.168.50.1/24] <br>

### EOF
chmod 600 /etc/netplan/60-dhcpnet.yaml <br>
netplan apply <br>
ip -br a <br>
> Bei eth1 muss 192.168.50.1/24 stehen.

## 4. DHCP-Server dnsmasq auf eth1 einrichten
Weiter im Container: <br>
apt install -y dnsmasq <br>
cat > /etc/dnsmasq.d/dhcpnet.conf << 'EOF' <br>
port=0 <br>
interface=eth1 <br>
bind-interfaces <br>
dhcp-range=192.168.50.100,192.168.50.200,12h <br>
EOF <br>
systemctl restart dnsmasq <br>
systemctl is-active dnsmasq <br>
exit <br>
> Eine Fehlermeldung bei der Installation (address already in use) ist normal, weil Port 53
vom Systemdienst belegt ist. Sie erledigt sich durch port=0 und den Neustart. <br>
Erwartet wird **active**.

| Zeile | Bedeutung |
|-------|-----------|
| port=0 | DNS-Teil abschalten. Er würde mit dem Systemdienst auf Port 53 kollidieren, gebraucht wird nur DHCP |
| interface=eth1, bindinterfaces | Nur auf dhcpnet lauschen, nicht im Netz 10.10.10.0/24, wo LXD selbst DHCP macht |
| dhcp-range=... | Adresspool .100 bis .200, Lease-Dauer 12 Stunden. .1 bleibt dem Server vorbehalten |
