Hier ist ein vollständiges Ansible Playbook inklusive Inventar- und Konfigurationsdatei. Es deckt deine gesamte Infrastruktur ab – von der LXD-Installation auf dem Host über das Anlegen der Netzwerke, das Erstellen/Konfigurieren der 3 Container (dhcp, monitoring, client) bis hin zur Installation von dnsmasq, prometheus und prometheus-node-exporter.

Wenn du das Betriebssystem neu installiert hast, musst du lediglich Ansible installieren und dieses Playbook einmal ausführen.

File 1: inventory.ini
Erstelle eine Datei namens inventory.ini auf deinem Steuerungs-PC (oder lokal auf dem Debian-Host):

Ini, TOML
[lxd_hosts]
localhost ansible_connection=local
File 2: setup_environment.yml
Speichere dieses Ansible Playbook als setup_environment.yml:

YAML
---
- name: Automatisierte LXD- und Container-Einrichtung (DHCP, Client, Monitoring)
  hosts: lxd_hosts
  become: true
  vars:
    dhcp_ip_lxd: "10.10.10.206"
    monitoring_ip_lxd: "10.10.10.178"

  tasks:
    # -------------------------------------------------------------------------
    # 1. HOST-VORBEREITUNG & LXD INSTALLATION
    # -------------------------------------------------------------------------
    - name: Systempakete aktualisieren
      ansible.builtin.apt:
        update_cache: yes
        upgrade: yes
        autoremove: yes

    - name: Core-Pakete für Snap und UFW installieren
      ansible.builtin.apt:
        name:
          - snapd
          - ufw
          - curl
        state: present

    - name: Snapd-Dienst aktivieren und starten
      ansible.builtin.systemd:
        name: snapd
        state: started
        enabled: yes

    - name: LXD via Snap installieren
      community.general.snap:
        name: lxd
        state: present

    - name: LXD initialisieren (minimal)
      ansible.builtin.command: lxd init --minimal
      args:
        creates: /var/snap/lxd/common/lxd/unix.socket

    - name: Sudo-Benutzer zur LXD-Gruppe hinzufügen
      ansible.builtin.user:
        name: "{{ ansible_env.SUDO_USER | default(ansible_user_id) }}"
        groups: lxd
        append: yes

    - name: UFW-Regeln für lxdbr0 setzen
      community.general.ufw:
        rule: allow
        direction: in
        interface: lxdbr0
      ignore_errors: yes

    - name: UFW-Routing-Regel für lxdbr0 setzen
      ansible.builtin.command: ufw route allow in on lxdbr0
      changed_when: false
      ignore_errors: yes

    # -------------------------------------------------------------------------
    # 2. NETZWERKE KONFIGURIEREN
    # -------------------------------------------------------------------------
    - name: lxdbr0 Netzwerkeigenschaften anpassen (10.10.10.1/24)
      ansible.builtin.command: "{{ item }}"
      loop:
        - lxc network set lxdbr0 ipv4.address=10.10.10.1/24
        - lxc network set lxdbr0 ipv6.address=none
      ignore_errors: yes

    - name: Isoliertes Netzwerk dhcpnet anlegen
      ansible.builtin.command: lxc network create dhcpnet ipv4.address=none ipv6.address=none
      register: dhcpnet_create
      failed_when: false
      changed_when: "'already exists' not in dhcpnet_create.stderr"

    # -------------------------------------------------------------------------
    # 3. CONTAINER ERSTELLEN
    # -------------------------------------------------------------------------
    - name: Container aus Ubuntu 24.04 starten
      ansible.builtin.command: "lxc launch ubuntu:24.04 {{ item }}"
      register: launch_res
      failed_when: false
      changed_when: "'already exists' not in launch_res.stderr"
      loop:
        - dhcp
        - monitoring
        - client

    - name: Warten, bis Container vollständig gestartet sind
      ansible.builtin.pause:
        seconds: 15

    # -------------------------------------------------------------------------
    # 4. CONTAINER-NETZWERKE & STATISCHE IPS EINRICHTEN
    # -------------------------------------------------------------------------
    - name: Feste IP für DHCP-Container auf lxdbr0 zuweisen
      ansible.builtin.command: "lxc config device override dhcp eth0 ipv4.address={{ dhcp_ip_lxd }}"
      register: override_dhcp
      failed_when: false
      changed_when: "'override' in override_dhcp.stdout or override_dhcp.rc == 0"

    - name: Feste IP für Monitoring-Container auf lxdbr0 zuweisen
      ansible.builtin.command: "lxc config device override monitoring eth0 ipv4.address={{ monitoring_ip_lxd }}"
      register: override_mon
      failed_when: false
      changed_when: "'override' in override_mon.stdout or override_mon.rc == 0"

    - name: eth1 (dhcpnet) an DHCP-Container anhängen
      ansible.builtin.command: lxc config device add dhcp eth1 nic network=dhcpnet name=eth1
      register: add_eth1_dhcp
      failed_when: false
      changed_when: "'added' in add_eth1_dhcp.stdout"

    - name: eth0 von Client auf dhcpnet umstellen
      ansible.builtin.command: lxc config device set client eth0 network=dhcpnet
      register: set_eth0_client
      failed_when: false

    - name: Container neustarten, um Netzanpassungen zu übernehmen
      ansible.builtin.command: "lxc restart {{ item }}"
      loop:
        - dhcp
        - client
        - monitoring

    - name: Warten nach Neustart
      ansible.builtin.pause:
        seconds: 10

    # -------------------------------------------------------------------------
    # 5. CONTAINER 1: DHCP EINRICHTEN (node_exporter, Netplan, dnsmasq)
    # -------------------------------------------------------------------------
    - name: [DHCP] Pakete aktualisieren & node_exporter + dnsmasq installieren
      ansible.builtin.command: >
        lxc exec dhcp -- sh -c "apt-get update && apt-get install -y prometheus-node-exporter dnsmasq"

    - name: [DHCP] Netplan-Konfiguration für eth1 schreiben (192.168.50.1/24)
      ansible.builtin.command:
        cmd: >
          lxc exec dhcp -- sh -c 'cat << "EOF" > /etc/netplan/60-dhcpnet.yaml
          network:
            version: 2
            ethernets:
              eth1:
                addresses: [192.168.50.1/24]
          EOF
          chmod 600 /etc/netplan/60-dhcpnet.yaml
          netplan apply'

    - name: [DHCP] dnsmasq-Konfiguration schreiben
      ansible.builtin.command:
        cmd: >
          lxc exec dhcp -- sh -c 'cat << "EOF" > /etc/dnsmasq.d/dhcpnet.conf
          port=0
          interface=eth1
          bind-interfaces
          dhcp-range=192.168.50.100,192.168.50.200,12h
          EOF
          systemctl restart dnsmasq'

    # -------------------------------------------------------------------------
    # 6. CONTAINER 2: CLIENT EINRICHTEN
    # -------------------------------------------------------------------------
    - name: [CLIENT] Pakete aktualisieren & Testordner anlegen
      ansible.builtin.command: >
        lxc exec client -- sh -c "apt-get update && mkdir -p /root/Testordner && echo 'Hallo Welt' > /root/Testordner/file1.txt"

    # -------------------------------------------------------------------------
    # 7. CONTAINER 3: MONITORING EINRICHTEN (Prometheus, Rules, Retention)
    # -------------------------------------------------------------------------
    - name: [MONITORING] Prometheus installieren
      ansible.builtin.command: >
        lxc exec monitoring -- sh -c "apt-get update && apt-get install -y prometheus"

    - name: [MONITORING] prometheus.yml schreiben
      ansible.builtin.command:
        cmd: >
          lxc exec monitoring -- sh -c 'cat << "EOF" > /etc/prometheus/prometheus.yml
          global:
            scrape_interval: 60s
            evaluation_interval: 60s
            rule_files:
              - rules.yml

          scrape_configs:
            - job_name: dhcp
              static_configs:
                - targets: ["10.10.10.206:9100"]
          EOF'

    - name: [MONITORING] TSDB Retention in /etc/default/prometheus setzen
      ansible.builtin.command:
        cmd: >
          lxc exec monitoring -- sh -c 'cat << "EOF" > /etc/default/prometheus
          ARGS="--storage.tsdb.retention.time=1d --storage.tsdb.retention.size=1GB"
          EOF'

    - name: [MONITORING] rules.yml schreiben (Recording & Alerting Rules)
      ansible.builtin.command:
        cmd: >
          lxc exec monitoring -- sh -c 'cat << "EOF" > /etc/prometheus/rules.yml
          groups:
            - name: dhcp-kennzahlen
              rules:
                - record: dhcp:cpu_percent
                  expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
                - record: dhcp:ram_percent
                  expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
                - record: dhcp:disk_percent
                  expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
                - record: dhcp:net_in_bytes
                  expr: rate(node_network_receive_bytes_total{device="eth0"}[5m])
                - record: dhcp:net_out_bytes
                  expr: rate(node_network_transmit_bytes_total{device="eth0"}[5m])

            - name: dhcp-alarme
              rules:
                - alert: DhcpContainerDown
                  expr: up{job="dhcp"} == 0
                  for: 2m
                - alert: DhcpCpuHoch
                  expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
                  for: 5m
                - alert: DhcpPlatteVoll
                  expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
                  for: 10m
          EOF'

    - name: [MONITORING] Prometheus Konfiguration validieren und neustarten
      ansible.builtin.command:
        cmd: >
          lxc exec monitoring -- sh -c "promtool check config /etc/prometheus/prometheus.yml && systemctl enable prometheus && systemctl restart prometheus"

    # -------------------------------------------------------------------------
    # 8. ABSCHLUSS-PRÜFUNG
    # -------------------------------------------------------------------------
    - name: Status-Übersicht anzeigen
      ansible.builtin.command: lxc list
      register: lxc_status

    - name: Ergebnisse ausgeben
      ansible.builtin.debug:
        msg: "{{ lxc_status.stdout_lines }}"
Anwendung / Ausführung
1. Einmalige Installation von Ansible auf dem frisch installierten Debian-Host:
Bash
sudo apt update
sudo apt install -y ansible
2. Ausführen des Playbooks:
Speichere inventory.ini und setup_environment.yml im selben Ordner und führe folgenden Befehl aus:

Bash
sudo ansible-playbook -i inventory.ini setup_environment.yml
Was das Skript vollautomatisch erledigt:
LXD-Setup: Installiert LXD via Snap, führt lxd init --minimal aus, berechtigt den aktuellen Benutzer und konfiguriert UFW für lxdbr0.

Netzwerk-Setup: Setzt lxdbr0 fest auf 10.10.10.1/24 und erstellt das isolierte dhcpnet-Netzwerk.

Container-Erstellung: Startet dhcp, monitoring und client mit Ubuntu 24.04.

IP- und Interface-Konfiguration:

dhcp: Erhält statische IP 10.10.10.206 auf eth0 sowie eth1 verbunden mit dhcpnet.

monitoring: Erhält statische IP 10.10.10.178 auf eth0.

client: Wird mit eth0 an dhcpnet gehängt.

Dienste-Einrichtung:

In dhcp: Stellt eth1 via Netplan auf 192.168.50.1/24 ein, installiert/konfiguriert dnsmasq als DHCP-Server für den Bereich .100–.200 und richtet den node_exporter ein.

In client: Erstellt die vorgegebene Ordnerstruktur /root/Testordner/file1.txt.

In monitoring: Installiert Prometheus, hinterlegt die prometheus.yml mit Target 10.10.10.206:9100, begrenzt die Datenhaltung (TSDB Retention: 1 Tag / 1 GB) und bindet die Recording- & Alerting-Rules in rules.yml ein. Danach wird das Setup mit promtool validiert und gestartet.
