# Einrichtung eines Backups
## 1. Export als .tar.gz auf den Desktop

Gestoppt ist das Backup konsistent: <br>
`lxc stop dhcp monitoring` <br>
`lxc export dhcp /root/Desktop/dhcp-$(date +%F).tar.gz` <br>
`lxc export monitoring /root/Desktop/monitoring-$(date +%F).tar.gz` <br>
`ls -lh /root/Desktop/*.tar.gz` <br>
`lxc start dhcp monitoring` <br>
`lxc list` <br>

## 2. Import und Wiederherstellung
> Auf demselben oder einem anderen Rechner (LXD muss installiert und initialisiert sein) <br>

`lxc network create dhcpnet ipv4.address=none ipv6.address=none` <br>
`lxc import /root/Desktop/dhcp-2026-10-01.tar.gz` <br>
`lxc import /root/Desktop/monitoring-2026-10-01.tar.gz` <br>
`lxc start dhcp monitoring` <br>
`lxc list` <br>

> Das Datum im Dateinamen passt du an das Netz `dhcpnet` muss vor dem Start von dhcp existieren, weil der Container es als eth1 verwendet. <br>
> Existiert schon ein Container mit gleichem Namen, lösche ihn vorher mit `lxc delete -f <name>` oder gib beim Import einen anderen Namen an `lxc import <datei> dhcp-neu`. <br>
> Prüfe nach dem Start das Scrape-Ziel in `/etc/prometheus/prometheus.yml` gegen die aktuelle IP von dhcp

________________________________________________________________________

## 1. LXD installieren und initialisieren
`sudo snap install lxd` <br>
`sudo lxd init --minimal` <br>
`sudo usermod -aG lxd` [`$USER`](https://github.com/KaiBorng/LF10b_Project/blob/main/src/documentation/passwords.md) <br>

Melde dich danach ab und wieder an. Prüfe dann: <br>
`export PATH=$PATH:/snap/bin` <br>
`lxc list` <br>
`uname -m` <br>
`snap list lxd` <br>
`sudo ufw allow in on lxdbr0` <br>
`sudo ufw route allow in on lxdbr0` <br>

## 2. Stick einbinden und Backup prüfen
Stick einstecken. Wird er nicht automatisch eingebunden: <br>
`lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS` <br>
`sudo mkdir -p /mnt/usb` <br>
`sudo mount /dev/partitinsname/mnt/usb` <br>
> Unser Pfad: /media/user/usb-14/backup_project. Prüfen:

`cd /pfad/zu/backup_project` <br>
`cat versions.txt` <br>
`sha256sum -c pruefsummen.txt` <br>

## 3. Netze anlegen (vor dem Import)
`lxc network set lxdbr0 ipv4.address=10.10.10.1/24` <br>
`lxc network set lxdbr0 ipv6.address=none` <br>
> Ohne diese zwei Netze starten dhcp und client nicht oder bekommen falsche Adressen <br>
> `dhcpnet` ist das isolierte Netz ohne LXD-DHCP, in dem die dnsmasq Adressen verteiltwerden `lxc network create dhcpnet ipv4.address=none ipv6.address=none` <br>

Kontrolle: <br>
`lxc network list` <br>
> lxdbr0 muss 10.10.10.1/24 zeigen, dhcpnet bei IPV4 und IPV6 none  <br>
> Die Dateien lxdbr0.yaml und dhcpnet.yaml auf dem Stick zeigen die Einstellungen des alten Hosts, falls du etwas nachschlagen willst

## 4. Container importieren
`cd /pfad/zu/backup_project` <br>
`lxc import dhcp.tar.gz` <br>
`lxc import monitoring.tar.gz` <br>
`lxc import client.tar.gz` <br>
`lxc list` <br>

Prüfe, ob die Netzwerkkarten der Container am richtigen Netz hängen: <br>
`lxc config show dhcp --expanded | grep -B1 -A4 "eth"` <br>
`lxc profile show default | grep -A3 eth0` <br>

> Der Container dhcp braucht eth0 an lxdbr0 (über das Profil) und eth1 an dhcpnet <br>
> Der Client hängt mit eth0 an dhcpnet. Sind sie dort falsch (zum Beispiel weil du ein anderes Netz als lxdbr0 benutzt), korrigierst du sie: <br>
`lxc config device set dhcp eth1 network=dhcpnet` <br>
`lxc config device set client eth0 network=dhcpnet` <br>

Gib dem DHCP-Container wieder seine feste Adresse. Das Ziel in der prometheus.yml (10.10.10.206) hängt daran: <br>
`lxc config device set dhcp eth0 ipv4.address=10.10.10.206` <br>
Prüfe außerdem das Speicherlimit von dhcp gegen den RAM des neuen Hosts: <br>
`lxc config get dhcp limits.memory` <br>
`free -m` <br>

> Optional bekommt auch monitoring eine feste Adresse `lxc config device override monitoring eth0 ipv4.address=10.10.10.178`, damit Grafana unter einer gleichbleibenden Adresse erreichbar bleibt

## 5. Container starten
Der DHCP-Container zuerst, danach die anderen:
`lxc start dhcp` <br>
`sleep 15` <br>
`lxc start monitoring client` <br>
`sleep 20` <br>
`lxc list` <br>
> Alle drei müssen RUNNING sein. Bei dhcp stehen 10.10.10.206 (eth0) und 192.168.50.1 (eth1), beim Client eine Adresse aus 192.168.50.100 bis .200.  <br>

## 6. Funktion prüfen <br>
`lxc exec dhcp -- systemctl is-active prometheus-node-exporter dnsmasq` <br>
`lxc exec monitoring -- systemctl is-active prometheus grafana-server` <br>
`lxc exec monitoring -- curl -s 'http://localhost:9090/api/v1/query?query=up'` <br>
`lxc exec client -- ip -br a` <br>
`lxc exec dhcp -- cat /var/lib/misc/dnsmasq.leases` <br>

## 7.Funktion Monitoring (Backup) prüfen <br>
`printf 'backup_last_success 1\nbackup_last_run_timestamp_seconds %s\nbackup_size_bytes 1000000\n' "$(date +%s)" | curl -sS --data-binary @- http://10.10.10.178:9091/metrics/job/backup`<br>
