# 0. Vorbereitung
PC starten <br>
mit Benutzer anmelden, der Admin Rechte bekommen kann <br>
Applications -> System Tools -> Mate Terminal <br>
`sudo su -` & *Benutzerpasswort angeben* <br>
`apt update && apt upgrade && apt autoremove` <br>


# 1. LXD installieren und initialisieren
`sudo snap install lxd` <br>
`sudo lxd init --minimal` <br>
`sudo usermod -aG lxd $USER` <br>
`newgrp lxd` <br>
`lxc network list` <br>
> Ist auf dem Host ufw aktiv, erlaube den Verkehr der Bridge, sonst bekommen die Container keine IP:

`sudo ufw allow in on lxdbr0` <br>
`sudo ufw route allow in on lxdbr0` <br>

# 2. Container anlegen
`lxc launch ubuntu:24.04 containername` <br>
`lxc launch ubuntu:24.04 monitoring` <br>
`sleep 20` <br>
`lxc list` <br>
> Beide müssen RUNNING sein und eine IPv4-Adresse haben <br>
> > Optional: Feste Adresse für dhcp, damit sich die IP nach einem Neustart nicht ändert

`lxc config device override dhcp eth0 ipv4.address=10.10.10.206` <br>
`lxc restart dhcp` <br>
