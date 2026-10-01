
## 1. Export als .tar.gz auf den Desktop

Gestoppt ist das Backup konsistent: <br>
lxc stop dhcp monitoring <br>
lxc export dhcp /root/Desktop/dhcp-$(date +%F).tar.gz <br>
lxc export monitoring /root/Desktop/monitoring-$(date +%F).tar.gz <br>
ls -lh /root/Desktop/*.tar.gz <br>
lxc start dhcp monitoring <br>
lxc list <br>

## 2. Import und Wiederherstellung
Auf demselben oder einem anderen Rechner (LXD muss installiert und initialisiert sein, siehe Schritt 1): <br>
lxc network create dhcpnet ipv4.address=none ipv6.address=none <br>
lxc import /root/Desktop/dhcp-2026-10-01.tar.gz <br>
lxc import /root/Desktop/monitoring-2026-10-01.tar.gz <br>
lxc start dhcp monitoring <br>
lxc list <br>

> Das Datum im Dateinamen passt du an. <br>

> Das Netz dhcpnet muss vor dem Start von dhcp existieren, weil der Container es als
eth1 verwendet. <br>

> Existiert schon ein Container mit gleichem Namen, lösche ihn vorher mit lxc delete -
f <name> oder gib beim Import einen anderen Namen an (lxc import <datei>
dhcp-neu). <br>

> Prüfe nach dem Start das Scrape-Ziel in /etc/prometheus/prometheus.yml gegen
die aktuelle IP von dhcp.
