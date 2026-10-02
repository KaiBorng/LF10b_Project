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

> Das Datum im Dateinamen passt du an <br>
> Das Netz `dhcpnet` muss vor dem Start von dhcp existieren, weil der Container es als eth1 verwendet. <br>
> Existiert schon ein Container mit gleichem Namen, lösche ihn vorher mit `lxc delete -f <name>` oder gib beim Import einen anderen Namen an `lxc import <datei> dhcp-neu`. <br>
> Prüfe nach dem Start das Scrape-Ziel in `/etc/prometheus/prometheus.yml` gegen die aktuelle IP von dhcp

________________________________________________________________________

## 1. LXD installieren und initialisieren
`sudo snap install lxd` <br>
`sudo lxd init --minimal` <br>
`sudo usermod -aG lxd` [$USER](https://github.com/KaiBorng/LF10b_Project/blob/main/src/documentation/passwords.md) <br>

Melde dich danach ab und wieder an. Prüfe dann: <br>
`export PATH=$PATH:/snap/bin` <br>
`lxc list` <br>
`uname -m` <br>
`snap list lxd` <br>

---- lxc list läuft ohne Fehler (leere Tabelle), und uname -m stimmt mit der Datei versions.txt vom Stick überein. Ist auf dem Host ufw aktiv:

`sudo ufw allow in on lxdbr0`
`sudo ufw route allow in on lxdbr0`

## 2. Stick einbinden und Backup prüfen
Stick einstecken. Wird er nicht automatisch eingebunden:

`lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINTS`
`sudo mkdir -p /mnt/usb`
`sudo mount /dev/sdX /mnt/usb`

Ersetze sdX durch den Namen des Sticks aus lsblk (z. B. sda, bei Partitionen sda1). Die Systemplatte (nvme0n1 oder sda mit Partitionen und /) fasst du nicht an. Der Pfad zum Backup lautet dann /mnt/usb/backup_project bzw. /media/BENUTZER/usb-14/backup_project. Prüfen:

`cd /pfad/zu/backup_project` <br>
`cat versions.txt` <br>
`sha256sum -c pruefsummen.txt` <br>

Jede Datei muss OK zeigen.

## 3. Netze anlegen (vor dem Import)
Ohne diese zwei Netze starten dhcp und client nicht oder bekommen falsche Adressen. <br>

lxdbr0 hat nach lxd init --minimal einen zufälligen Adressbereich. Setze den alten Bereich, damit der DHCP-Container wieder seine bekannte IP bekommen kann:

`lxc network set lxdbr0 ipv4.address=10.10.10.1/24` <br>
`lxc network set lxdbr0 ipv6.address=none` <br>

Lief auf dem neuen Host schon etwas anderes auf lxdbr0, ändere den Bereich nicht. Lege stattdessen ein eigenes Netz an (z. B. lxc network create lxdbr1 ipv4.address=10.10.10.1/24 ipv4.nat=true ipv6.address=none) und weise es später den Containern zu (siehe B5).

dhcpnet ist das isolierte Netz ohne LXD-DHCP, in dem dein dnsmasq Adressen verteilt:

lxc network create dhcpnet ipv4.address=none ipv6.address=none

Kontrolle:
`lxc network list` <br>

lxdbr0 muss 10.10.10.1/24 zeigen, dhcpnet bei IPV4 und IPV6 none.
Die Dateien lxdbr0.yaml und dhcpnet.yaml auf dem Stick zeigen die Einstellungen des alten Hosts, falls du etwas nachschlagen willst.

## 4. Container importieren
`cd /pfad/zu/backup_project` <br>
`lxc import dhcp.tar.gz` <br>
`lxc import monitoring.tar.gz` <br>
`lxc import client.tar.gz` <br>
`lxc list` <br>

Jeder Import dauert einige Minuten. Danach stehen alle drei als STOPPED in der Liste. Zieh den Stick nicht ab, solange ein Import läuft.

+ Tabelle
Meldung 	Lösung
Storage pool not found 	Pool mit lxc storage list nachsehen, dann lxc import datei.tar.gz --storage POOLNAME
Container existiert schon 	Mit neuem Namen importieren (lxc import datei.tar.gz neuername) oder alten mit lxc delete -f NAME löschen
Fehlende Datei im Archiv (z. B. backup.yaml) 	Die Datei ist kein Container-Backup, sondern ein Image
B5. Konfiguration prüfen und feste IP setzen
+++++++

Prüfe, ob die Netzwerkkarten der Container am richtigen Netz hängen:

`lxc config show dhcp --expanded | grep -B1 -A4 "eth"`
`lxc profile show default | grep -A3 eth0`

Der Container dhcp braucht eth0 an lxdbr0 (über das Profil) und eth1 an dhcpnet. Der Client hängt mit eth0 an dhcpnet. Sind sie dort falsch (zum Beispiel weil du ein anderes Netz als lxdbr0 benutzt), korrigierst du sie:

`lxc config device set dhcp eth1 network=dhcpnet`
`lxc config device set client eth0 network=dhcpnet`

Gib dem DHCP-Container wieder seine feste Adresse. Das Ziel in der prometheus.yml (10.10.10.206) hängt daran:

lxc config device override dhcp eth0 ipv4.address=10.10.10.206

Meldet LXD, dass das Gerät eth0 schon existiert, dann setze den Wert so:

lxc config device set dhcp eth0 ipv4.address=10.10.10.206

Prüfe außerdem das Speicherlimit von dhcp gegen den RAM des neuen Hosts:

`lxc config get dhcp limits.memory` <br>
`free -m` <br>

Optional bekommt auch monitoring eine feste Adresse (lxc config device override monitoring eth0 ipv4.address=10.10.10.178), damit Grafana unter einer gleichbleibenden Adresse erreichbar bleibt.
B6. Container starten

Der DHCP-Container zuerst, danach die anderen:

`lxc start dhcp` <br>
`sleep 15` <br>
`lxc start monitoring client` <br>
`sleep 20` <br>
`lxc list` <br>

Alle drei müssen RUNNING sein. Bei dhcp stehen 10.10.10.206 (eth0) und 192.168.50.1 (eth1), beim Client eine Adresse aus 192.168.50.100 bis .200.
B7. Funktion prüfen

`lxc exec dhcp -- systemctl is-active prometheus-node-exporter dnsmasq` <br>
`lxc exec monitoring -- systemctl is-active prometheus grafana-server` <br>
`lxc exec monitoring -- curl -s 'http://localhost:9090/api/v1/query?query=up'` <br>
`lxc exec client -- ip -br a` <br>
`lxc exec dhcp -- cat /var/lib/misc/dnsmasq.leases` <br>
