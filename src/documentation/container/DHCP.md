4. DHCP-Container vorbereiten
4.1 node_exporter installieren (Host)
lxc exec dhcp -- apt update
lxc exec dhcp -- apt install -y prometheus-node-exporter
lxc exec dhcp -- systemctl is-active prometheus-node-exporter
Wozu? Prometheus misst nicht selbst, sondern holt Werte von Programmen ab. Der
node_exporter stellt Systemwerte des Containers (CPU, RAM, Platte, Netzwerk) als Text auf
Port 9100 bereit. Prometheus ruft diese Seite alle 60 Sekunden ab.
Test vom Monitoring-Container aus:
lxc exec monitoring -- curl -s http://10.10.10.206:9100/metrics | head -5
Es müssen Zeilen mit # HELP ... erscheinen. Der node_exporter überwacht den
Container, nicht den DHCP-Dienst selbst.
4.2 Zweites Netz dhcpnet anlegen und als eth1 anhängen (Host)
lxc network create dhcpnet ipv4.address=none ipv6.address=none
lxc config device add dhcp eth1 nic network=dhcpnet name=eth1
lxc exec dhcp -- ip -br a
In der Ausgabe muss eth1 (Status UP, noch ohne IPv4) neben eth0 stehen. Der Container muss
dafür laufen.
Warum ein eigenes Netz? In einem Netz darf nur ein DHCP-Server antworten. In lxdbr0
antwortet bereits LXD. dhcpnet hat bewusst keinen LXD-DHCP, sodass dort allein dein DHCPServer
Adressen verteilt. eth0 bleibt für das Monitoring erreichbar.
4.3 Feste IP auf eth1 per Netplan
lxc exec dhcp -- bash
Im Container (Prompt root@dhcp:~#):
cat > /etc/netplan/60-dhcpnet.yaml << 'EOF'
network:
version: 2
ethernets:
eth1:
addresses: [192.168.50.1/24]
EOF
chmod 600 /etc/netplan/60-dhcpnet.yaml
netplan apply
ip -br a
Bei eth1 muss 192.168.50.1/24 stehen.
Warum fest? Ein DHCP-Server kann seine eigene Adresse nicht per DHCP beziehen, weil er ja der
Einzige wäre, der antworten könnte. Clients müssen ihn außerdem unter einer gleichbleibenden
Adresse erreichen. Die Adresse legt zugleich das Netz (192.168.50.0/24) fest, für das er
Adressen verteilt.
4.4 DHCP-Server dnsmasq auf eth1 einrichten
Weiter im Container:
apt install -y dnsmasq
cat > /etc/dnsmasq.d/dhcpnet.conf << 'EOF'
port=0
interface=eth1
bind-interfaces
dhcp-range=192.168.50.100,192.168.50.200,12h
EOF
systemctl restart dnsmasq
systemctl is-active dnsmasq
exit
Eine Fehlermeldung bei der Installation (address already in use) ist normal, weil Port 53
vom Systemdienst belegt ist. Sie erledigt sich durch port=0 und den Neustart. Erwartet wird
active.

Zeile Bedeutung
port=0 DNS-Teil abschalten. Er würde mit dem Systemdienst auf Port
53 kollidieren, gebraucht wird nur DHCP.
interface=eth1, bindinterfaces
Nur auf dhcpnet lauschen, nicht im Netz 10.10.10.0/24,
wo LXD selbst DHCP macht.
dhcp-range=... Adresspool .100 bis .200, Lease-Dauer 12 Stunden. .1
bleibt dem Server vorbehalten.
Warum dnsmasq? Er ist mit einer einzigen kleinen Konfigurationsdatei der geringste Aufwand für
einen funktionsfähigen DHCP-Server
