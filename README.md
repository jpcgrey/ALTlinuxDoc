# ALTlinuxDoc
Analyse russischer Linux-Distributionen

Modul 1: Netzwerkinfrastruktur (Grundkonfiguration)

    Charakteristik: Vollständiger Aufbau der Infrastruktur ohne vorbereitete Konfiguration.

1.1 Hostnamen setzen
ALT Linux (HQ-SRV, BR-SRV, HQ-CLI, ISP)
```bash

hostnamectl set-hostname <rechnername>.au-team.irpo
exec bash

EcoRouterOS (HQ-RTR, BR-RTR)
```
```bash

enable
configure terminal
hostname <rechnername>
ip domain-name au-team.irpo
write memory

```
1.2 IPv4-Adressierung
ALT Linux
```bash

echo "<IP-Adresse>/<Präfix>" > /etc/net/ifaces/<schnittstelle>/ipv4address
echo "default via <gateway-ip>" > /etc/net/ifaces/<schnittstelle>/ipv4route
systemctl restart network

EcoRouterOS
```
```bash

configure terminal
interface <name>
 ip address <IP>/<Präfix>
 exit
port <portname>
 service-instance <name>
  encapsulation untagged|dot1q <VID>
  connect ip interface <name>
  exit
 exit
write memory

```
1.3 Internetzugang (ISP)
```bash

# DHCP auf externem Interface
echo "BOOTPROTO=dhcp" > /etc/net/ifaces/<ext-interface>/options

# IP-Weiterleitung aktivieren
sysctl -w net.ipv4.ip_forward=1

# NAT mit iptables
iptables -t nat -A POSTROUTING -o <ext-if> -j MASQUERADE

```
1.4 Benutzerverwaltung
ALT Linux
```bash

useradd <benutzername> -u <UID>
passwd <benutzername>
usermod -aG wheel <benutzername>
echo "<benutzername> ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers

EcoRouterOS
```
```bash

configure terminal
username <name>
 password <passwort>
 role admin
 exit
write memory

```
1.5 SSH-Dienst (gesichert)
```bash

# /etc/openssh/sshd_config
Port 2026
AllowUsers <benutzername>
MaxAuthTries 2
Banner /etc/openssh/banner

systemctl restart sshd

```
1.6 Tunnel (GRE oder IP-in-IP)
EcoRouterOS
```bash

configure terminal
interface tunnel.0
 ip address <IP>/<Präfix>
 ip tunnel <quell-IP> <ziel-IP> mode gre|ipip
 exit
write memory

```
1.7 Dynamisches Routing (OSPF)
```bash

configure terminal
router ospf 1
 passive-interface default
 no passive-interface tunnel.0
 network <netz>/<präfix> area 0
 area 0 authentication
 interface tunnel.0
  ip ospf authentication-key <schlüssel>
  exit
 exit
write memory

```
1.8 DHCP-Server (EcoRouterOS)
```bash

configure terminal
ip pool <name> <start-IP>-<end-IP>
dhcp-server 1
 pool <name> 1
  mask <präfix>
  gateway <gateway-IP>
  dns <dns-server-IP>
  domain-name <domain>
  exit
 exit
interface <interface>
 dhcp-server 1
 exit
write memory

```
1.9 DNS-Server (BIND auf ALT Linux)
```bash

apt-get update && apt-get install -y bind bind-utils

# /var/lib/bind/etc/options.conf
listen-on { <lokale-IP>; };
forwarders { <öffentlicher-DNS>; };
allow-query { <erlaubte-netze>; };

# Zonenkonfiguration in /var/lib/bind/etc/rfc1912.conf
zone "<domain>" { type master; file "master/<zone>"; };

systemctl enable --now bind

```
1.10 Zeitzone
```bash

# ALT Linux
timedatectl set-timezone Europe/Moscow

# EcoRouterOS
configure terminal
ntp timezone utc+3

```
Modul 2: Netzwerkverwaltung (vorbereitete Umgebung)

    Charakteristik: Die Umgebung ist teilweise vorbereitet (IPs, Routing, NAT, DNS, DHCP sind bereits konfiguriert).

2.1 Samba-Domänencontroller (BR-SRV)
```bash

apt-get update && apt-get install -y task-samba-dc

systemctl stop krb5kdc slapd bind
systemctl disable krb5kdc slapd bind

rm -f /etc/samba/smb.conf
rm -rf /var/lib/samba

samba-tool domain provision --use-rfc2307 --interactive

cp /var/lib/samba/private/krb5.conf /etc/krb5.conf
systemctl enable --now samba

```
2.2 Benutzer und Gruppen in der Domäne
```bash

samba-tool group add <gruppe>
samba-tool user add <benutzer> <passwort>
samba-tool group addmembers <gruppe> <benutzer>

```
2.3 Domänenbeitritt (HQ-CLI)
```bash

# DNS auf BR-SRV ausrichten
echo "nameserver 192.168.0.2" > /etc/net/ifaces/<if>/resolv.conf

apt-get update && apt-get install -y task-auth-ad-sssd
# Grafische Konfiguration über Zentrales Steuerungssystem (CSC)

apt-get install -y libnss-role
roleadd <domänengruppe> wheel

echo "%wheel ALL=(ALL:ALL) NOPASSWD: /bin/cat, /bin/grep, /usr/bin/id" >> /etc/sudoers

```
2.4 RAID 0 (HQ-SRV)
```bash

apt-get install -y mdadm
mdadm --create --verbose /dev/md0 -l 0 -n 2 /dev/sdb /dev/sdc
mkfs.ext4 /dev/md0
mkdir /raid
echo "/dev/md0 /raid ext4 defaults 0 0" >> /etc/fstab
mount -av

```
2.5 NFS-Freigabe (HQ-SRV → HQ-CLI)
```bash

# Server
apt-get install -y nfs-server nfs-utils
mkdir /raid/nfs
echo "/raid/nfs 192.168.200.0/24(rw,no_root_squash)" >> /etc/exports
systemctl enable --now nfs-server

# Client
apt-get install -y nfs-utils nfs-clients
mkdir /mnt/nfs
echo "192.168.100.2:/raid/nfs /mnt/nfs nfs defaults 0 0" >> /etc/fstab
mount -av

```
2.6 Ansible (Steuerknoten: BR-SRV)
```bash

apt-get update && apt-get install -y ansible sshpass
ansible-galaxy collection install ansible.netcommon cisco.ios

# Inventar (/etc/ansible/hosts)
[servers]
hq-srv ansible_host=192.168.100.2
hq-cli ansible_host=192.168.200.2
[routers]
hq-rtr ansible_host=192.168.100.1
br-rtr ansible_host=192.168.0.1
[all:vars]
ansible_user=sshuser

```
2.7 Docker-Container (testapp auf BR-SRV)
```bash

apt-get install -y docker-engine docker-compose-v2
systemctl enable --now docker

docker load < /mnt/docker/site_latest.tar
docker load < /mnt/docker/mariadb_latest.tar

docker-compose up -d

```
2.8 LAMP-Webanwendung (HQ-SRV)
```bash

apt-get install -y lamp-server
systemctl enable --now mariadb httpd2

mysql -u root
CREATE DATABASE webdb;
CREATE USER 'web'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON webdb.* TO 'web'@'localhost';
EXIT;

mysql -u web -p webdb < /mnt/web/dump.sql

```
2.9 Statisches NAT (Portweiterleitung)
```bash

configure terminal
ip nat source static tcp <lokale-IP> <lokaler-port> <externe-IP> <externer-port>
write memory

```
2.10 Reverse-Proxy (Nginx auf ISP)
```bash

apt-get update && apt-get install -y nginx

# Konfiguration in /etc/nginx/sites-available.d/default.conf
server {
    listen 80;
    server_name <domain>;
    location / {
        proxy_pass http://<ziel-IP>:<port>;
    }
}

systemctl enable --now nginx

```
2.11 HTTP-Basisauthentifizierung (Nginx)
```bash

htpasswd -c /etc/nginx/.htpasswd <benutzername>
# auth_basic und auth_basic_user_file im location-Block ergänzen
systemctl reload nginx

```
2.12 Yandex.Browser (HQ-CLI)
```bash

apt-get update && apt-get install -y yandex-browser-stable

```
Modul 3: Fortgeschrittene Verwaltung (Sicherheit, Monitoring, Backup)

    Charakteristik: Erweiterte Themen: Migration, Zertifikate (GOST), IPSec, Firewall, Logging, Monitoring, Inventarisierung, Fail2ban, Backup.

3.1 Massenimport von Benutzern (CSV → Samba)
```bash

mount /dev/cdrom /mnt
cp /mnt/Users.csv /root
iconv -f UTF-8 -t UTF-8//IGNORE /root/Users.csv > /root/Users_fixed.csv

# Import-Skript (siehe Originaldokument)
bash /root/import.sh

```
3.2 GOST-Zertifikate (CA auf HQ-SRV, HTTPS auf ISP)
```bash

# CA erstellen
control openssl-gost enabled
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:TCB -out ca.key
openssl req -new -x509 -md_gost12_256 -days 90 -key ca.key -out ca.crt

# Serverzertifikate
openssl genpkey -algorithm gost2012_256 -pkeyopt paramset:A -out <name>.key
openssl req -new -md_gost12_256 -key <name>.key -out <name>.csr
openssl x509 -req -in <name>.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out <name>.crt -days 30

# Kopie zu ISP
scp *.crt *.key root@172.16.1.1:/etc/nginx/

# Nginx-Konfiguration: Port 443 mit ssl-Parametern

```
3.3 IPSec (gesicherter Tunnel zwischen HQ-RTR und BR-RTR)
```bash

configure terminal
crypto-ipsec ike enable
crypto-ipsec profile IPSEC ike-v2
 mode tunnel
 ike-phase1 proposal aes256-sha256-modp2048 auth pre-shared-key <key>
 ike-phase2 protocol esp proposal aes256-sha256
 local-ts <lokale-IP> remote-ts <entfernte-IP>
exit
crypto-map CMAP 10 match peer <entfernte-IP> set crypto-ipsec profile IPSEC
filter-map ipv4 FMAP 10 match gre host <lokal> host <entfernt> set crypto-map CMAP peer <entfernt>
interface tunnel.0 set filter-map in FMAP 10
write memory

```
3.4 Firewall (EcoRouterOS)
```bash

# Standard-Allow-Regel entfernen
no filter-map ipv4 FMAP 30

# Erlaubnisregeln für spezifische Protokolle (HTTP, HTTPS, DNS, NTP, ICMP, Kerberos, LDAP, SMB)
filter-map ipv4 FMAP 21
 match tcp any any eq 88
 match udp any any eq dns
 # ... (weitere Protokolle nach Vorgabe)
 set accept
exit

```
3.5 NTP-Server (chrony auf ISP)
```bash

apt-get install -y chrony
# /etc/chrony.conf: server <öffentlicher-ntp>, local stratum 5, allow 0.0.0.0/0
systemctl restart chronyd

```
3.6 CUPS-Druckserver (HQ-SRV)
```bash

apt-get install -y cups cups-pdf
# /etc/cups/cupsd.conf: Zugriff in <Location />, <Location /admin> erlauben
systemctl enable --now cups

```
3.7 Rsyslog (zentraler Logserver: HQ-SRV)
```bash

# Server
module(load="imudp")
input(type="imudp" port="514")
if $fromhost-ip == '<client-IP>' then /opt/<hostname>/log

# Client (BR-SRV, BR-RTR)
*.warn @@<server-IP>:514

```
3.8 Monitoring (Prometheus + Grafana auf HQ-SRV)
```bash

apt-get install -y grafana prometheus prometheus-node_exporter
# prometheus.yml: targets [localhost:9090, <hq-srv>:9100, <br-srv>:9100]
systemctl enable --now grafana-server prometheus prometheus-node_exporter

```
3.9 Inventarisierung mit Ansible (Playbook)
```yaml

- name: Get host info
  hosts: hq-srv, hq-cli
  tasks:
    - name: Save info
      copy:
        dest: "/etc/ansible/PC-INFO/{{ ansible_hostname }}.yml"
        content: |
          Hostname: {{ ansible_hostname }}
          IP_Address: {{ ansible_default_ipv4.address }}
      delegate_to: localhost

```
3.10 Fail2ban (SSH-Schutz)
```bash

apt-get install -y fail2ban
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 2026
maxretry = 3
bantime = 60
logpath = %(sshd_log)s

systemctl enable --now fail2ban

```
3.11 Cyber Backup (Installation und Grundkonfiguration)
```bash

# Kernel-Update erforderlich (update-kernel && reboot)
# Installation über ISO: Management Server, Agent for Linux, Agent for MySQL
# Benutzer iproadmin mit Passwort P@ssw0rd
# Speicherknoten auf HQ-CLI mit Verzeichnis /backup
# Zwei Sicherungspläne: /etc (Dateien) und webdb (MySQL)

```
Anhang: Systemanforderungen (Auszug)
Komponente	Minimalanforderung
CPU	8 Kerne pro Arbeitsplatz
RAM	10 GB pro Arbeitsplatz
Speicher	SSD mit ≥450 MB/s linearer Lese
Netzwerk	≥1 Gbit/s
Virtualisierung	KVM, Proxmox VE, Alt Virtualisierung
