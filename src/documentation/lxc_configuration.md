1. PC starten
2. mit Benutzer anmelden, der Admin Rechte bekommen kann.
3. Applications -> System Tools -> Mate Terminal
5. sudo su -
6. sudo su - & *Benutzerpasswort angeben*
7. apt update && apt upgrade && apt autoremove
8. apt install lxc lxc-templates -y & *warten bis der Download fertig ist*
9. lxc init
10. lxc launch ubuntu:24:04 *Clientname*


Container starten: lxc start *Containername*
Container löschen: lxc delete *Containername*
Container stoppen: lxc stop *Containername*
