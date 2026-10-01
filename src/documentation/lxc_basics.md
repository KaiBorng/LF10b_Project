# LXC Basic Debian Befehle

| Befehl | Funktion |
|----------|----------|
| `lxc list` | Listet alle Container auf |
| `lxc launch images:debian/12 mein-container` | Erstellt und startet einen neuen Debian-Container |
| `lxc start mein-container` | Startet einen Container |
| `lxc stop mein-container` | Stoppt einen Container |
| `lxc restart mein-container` | Startet einen Container neu |
| `lxc delete mein-container` | Löscht einen Container |
| `lxc delete mein-container --force` | Erzwingt das Löschen eines Containers |
| `lxc rename altname neuname` | Benennt einen Container um |
| `lxc copy container1 container2` | Erstellt eine Kopie eines Containers |
| `lxc exec mein-container -- bash` | Öffnet eine Bash-Shell im Container |
| `lxc exec mein-container -- sh` | Öffnet eine SH-Shell im Container |
| `lxc exec mein-container -- apt update` | Führt einen Befehl im Container aus |
| `lxc info mein-container` | Zeigt Informationen zum Container |
| `lxc config show mein-container` | Zeigt die Konfiguration des Containers |
| `lxc snapshot mein-container snap1` | Erstellt einen Snapshot |
| `lxc restore mein-container snap1` | Stellt einen Snapshot wieder her |
| `lxc file push datei.txt mein-container/root/` | Kopiert eine Datei in den Container |
| `lxc file pull mein-container/root/datei.txt .` | Kopiert eine Datei aus dem Container |
| `lxc network list` | Zeigt verfügbare Netzwerke an |
| `lxc network show lxdbr0` | Zeigt Details eines Netzwerks |
| `lxc storage list` | Zeigt verfügbare Storage-Pools |
| `lxc storage show default` | Zeigt Informationen zum Storage-Pool |
| `lxc image list images:` | Listet verfügbare Container-Images auf |
| `lxc image list` | Zeigt lokal gespeicherte Images |
| `lxc monitor` | Überwacht Ereignisse und Logs in Echtzeit |
| `lxc config set mein-container boot.autostart true` | Aktiviert den automatischen Start eines Containers |
| `lxc profile list` | Zeigt vorhandene Profile an |
| `lxc profile show default` | Zeigt die Einstellungen eines Profils |
