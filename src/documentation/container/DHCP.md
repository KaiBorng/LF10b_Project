# DHCP-Container vorbereiten

## 1. node_exporter installieren <br>
`lxc exec dhcp -- bash` <br>
`apt update && apt upgrade && apt autoremove` <br>
`apt install -y prometheus-node-exporter` <br>
`systemctl is-active prometheus-node-exporter` <br>
> Wozu? Prometheus misst nicht selbst, sondern holt Werte von Programmen ab. <br>
Der node_exporter stellt Systemwerte des Containers (CPU, RAM, Platte, Netzwerk) als Text auf Port 9100 bereit <br> Prometheus ruft diese Seite alle 60 Sekunden ab. <br>

### Test vom Monitoring-Container aus:
`lxc exec monitoring -- curl -s http://10.10.10.206:9100/metrics | head -5` <br>
> Es müssen Zeilen mit # HELP ... erscheinen. Der node_exporter überwacht den Container, nicht den DHCP-Dienst selbst.

## 2. Zweites Netz dhcpnet anlegen und als eth1 anhängen
`lxc network create dhcpnet ipv4.address=none ipv6.address=none` <br>
`lxc config device add dhcp eth1 nic network=dhcpnet name=eth1` <br>
`lxc exec dhcp -- ip -br a` <br>
> In der Ausgabe muss eth1 (Status UP, noch ohne IPv4) neben eth0 stehen. Der Container muss dafür laufen. <br>

## 3. Feste IP auf eth1 per Netplan
`lxc exec dhcp -- bash
Im Container (Prompt root@dhcp:~#):
cat > /etc/netplan/60-dhcpnet.yaml << 'EOF'
network:
version: 2
ethernets:
eth1:
addresses: [192.168.50.1/24]` <br>

### EOF
`chmod 600 /etc/netplan/60-dhcpnet.yaml
netplan apply
ip -br a` <br>
> Bei eth1 muss 192.168.50.1/24 stehen.

## 4. DHCP-Server dnsmasq auf eth1 einrichten
Weiter im Container: <br>
`apt install -y dnsmasq
cat > /etc/dnsmasq.d/dhcpnet.conf << 'EOF'
port=0
interface=eth1
bind-interfaces
dhcp-range=192.168.50.100,192.168.50.200,12h
EOF
systemctl restart dnsmasq
systemctl is-active dnsmasq
exit` <br>
> Eine Fehlermeldung bei der Installation (address already in use) ist normal, weil Port 53
vom Systemdienst belegt ist. Sie erledigt sich durch port=0 und den Neustart. <br>
Erwartet wird **active**.

| Zeile | Bedeutung |
|-------|-----------|
| port=0 | DNS-Teil abschalten. Er würde mit dem Systemdienst auf Port 53 kollidieren, gebraucht wird nur DHCP |
| interface=eth1, bindinterfaces | Nur auf dhcpnet lauschen, nicht im Netz 10.10.10.0/24, wo LXD selbst DHCP macht |
| dhcp-range=... | Adresspool .100 bis .200, Lease-Dauer 12 Stunden. .1 bleibt dem Server vorbehalten |
