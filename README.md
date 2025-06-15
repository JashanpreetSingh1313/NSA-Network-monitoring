# NSA-Network-monitoring
“LibreNMS vs SolarWinds”

###LibreNMS:

- Open-source, free to use

- Works with SNMP to monitor network devices

- Web dashboard with graphs, alerts, and discovery

- Runs on Linux (Ubuntu, Debian, CentOS)

- Setup requires some Linux and CLI knowledge

- Best for labs, students, and small businesses

###SolarWinds:

- Commercial tool (not free)

- Known for advanced network monitoring and beautiful dashboards

- Easier setup on Windows

- Features like NetPath, config backups, alerting

- Used by enterprises, requires licensing

# Steps to install LibreNMS and how to add devices to the monitoring system.

# LibreNMS Setup Guide on Ubuntu 24.04

LibreNMS is a powerful open-source network monitoring system that supports auto-discovery, alerting, graphing, and integration with many third-party tools. This guide will walk you through installing LibreNMS on Ubuntu 24.04.

---

## ✅ Prerequisites

- Ubuntu 24.04 server
- Root access or `sudo` privileges
- Internet connection
- SNMP port 161 open

---

## 🧱 Step-by-Step Installation

### 1. Install Required Packages
```bash
apt update && apt install -y acl curl fping git graphviz imagemagick mariadb-client mariadb-server mtr-tiny nginx-full nmap \
php-cli php-curl php-fpm php-gd php-gmp php-json php-mbstring php-mysql php-snmp php-xml php-zip rrdtool snmp snmpd unzip \
python3-command-runner python3-pymysql python3-dotenv python3-redis python3-setuptools python3-psutil python3-systemd \
python3-pip whois traceroute
```

### 2. Create LibreNMS User
```bash
useradd librenms -d /opt/librenms -M -r -s "$(which bash)"
```

### 3.3. Download LibreNMS
```bash
cd /opt
git clone https://github.com/librenms/librenms.git
chown -R librenms:librenms /opt/librenms
chmod 771 /opt/librenms
```

### 4. Set Folder Permissions
```bash

setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
```
### 5. Install PHP Dependencies
```bash

su - librenms
./scripts/composer_wrapper.php install --no-dev
exit
```
### 6. Set Timezone
```bash

Edit
timedatectl set-timezone Etc/UTC
vi /etc/php/8.3/fpm/php.ini
vi /etc/php/8.3/cli/php.ini
```
###### Set: date.timezone = Etc/UTC (MAY VARY )

🛢️ Configure MariaDB

### 7. Edit Configuration
```bash

vi /etc/mysql/mariadb.conf.d/50-server.cnf
```
Add the following under [mysqld]:

```bash
innodb_file_per_table=1
lower_case_table_names=0
```
### 8. Restart MariaDB and Create Database
```bash

systemctl enable mariadb
systemctl restart mariadb
mysql -u root
```
Inside MySQL:
```bash
CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
EXIT;
```
### ⚙️ Configure PHP-FPM

### 9. Create Pool Config
```bash
cp /etc/php/8.3/fpm/pool.d/www.conf /etc/php/8.3/fpm/pool.d/librenms.conf
vi /etc/php/8.3/fpm/pool.d/librenms.conf
```
Change to:
```bash
[librenms]
user = librenms
group = librenms
listen = /run/php/php-fpm-librenms.sock
```
🌐 Configure NGINX
### 10. Create Virtual Host
```bash
vi /etc/nginx/conf.d/librenms.conf
```
Paste:
```bash

server {
    listen      80;
    server_name librenms.local;
    root        /opt/librenms/html;
    index       index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    location ~ \.php$ {
        fastcgi_pass unix:/run/php-fpm-librenms.sock;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include fastcgi.conf;
    }
   location ~ /\.(?!well-known).* {
        deny all;
    }
}
```
### 11. Restart Services
```bash
rm /etc/nginx/sites-enabled/default /etc/nginx/sites-available/default
systemctl restart nginx
systemctl restart php8.3-fpm
```
#### Allow access through firewall

### 12. Configure snmpd (v2c)
```bash
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf

vi /etc/snmp/snmpd.conf
```
Edit the text which says RANDOMSTRINGGOESHERE and set your own community string.
```bash
curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl enable snmpd
systemctl restart snmpd
```
Enable the Scheduler
LibreNMS uses a scheduler for background tasks.
```bash
cp /opt/librenms/dist/librenms-scheduler.service /opt/librenms/dist/librenms-scheduler.timer /etc/systemd/system/
systemctl enable librenms-scheduler.timer
systemctl start librenms-scheduler.timer
```
### 13.Enable Log Rotation (Optional but Recommended)
To prevent logs from filling up disk space:

```bash
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```

### 14. Enable web installer

Open your browser and go to:
```bash
http://<your-server-ip>:80/install
# OR if you used a different port:
http://<your-server-ip>:8080/install
```
Then login to LibreNMS using the credentials.

## 🖥️ How to Add Devices in LibreNMS

Once you’re logged into the dashboard:

1. ➕ To Add a Device:
Go to Devices > Add Device

2. Fill in
Hostname/IP of the device (e.g., 192.168.1.10)

SNMP Version: select v2c

Community String: this must match the snmpd.conf file (e.g., public or your custom one)

3. Click Add Device

If the SNMP service is reachable and port 161 is open, the device will be added and discovered.

#### Make Sure Before Adding Devices
- SNMP service is enabled and running on the device

- Firewall allows SNMP (UDP port 161)

- Community string matches what you configured in
  ```bash
  /etc/snmp/snmpd.conf
  ```
-  You can test SNMP from your LibreNMS server:
```bash
snmpwalk -v2c -c yourCommunityString 192.168.1.10
```
- For better security, consider:

- Enabling SNMPv3

- Setting up HTTPS with Let's Encrypt

- Locking down firewall access

## Closing Remarks

This project helped us explore the differences between an open-source network monitoring solution (LibreNMS) and a commercial enterprise-grade tool (SolarWinds). Through hands-on setup and configuration, we learned the importance of visibility, alerting, and centralized monitoring in managing modern IT environments. LibreNMS provided valuable experience with Linux-based services, SNMP configuration, and web server integration, making it an ideal solution for labs and small setups. While SolarWinds offers powerful features and a refined interface, LibreNMS proves that open-source tools can be just as capable when configured properly.

This project was completed by the following students from team 1 :
- **Armaan Singh Virk**
- **Manjot Singh Gill**
- **Jashanpreet Singh**

Thank you for reading!

