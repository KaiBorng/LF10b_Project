# 1. Ansible installieren und Container erstellen
`sudo su -` <br>
`apt update && apt upgrade && apt autoremove` <br>
`apt install -y ansible` <br>
`sudo ansible-playbook -i inventory.ini setup_environment.yml` <br>
