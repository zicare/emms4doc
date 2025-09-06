# Cloud server setup

EMMS4 requires a Linux cloud server. We recommend a Digital Ocean droplet with Ubuntu 24.04 (LTS) x64 or later. Tested with Ubuntu 24.04 (LTS) x64. 

Recommended/Tested droplet size: 8 GB Memory / 4 AMD vCPUs / 160 GB 

## Server configuration

Run these commands as root:

```bash
apt-get update && apt-get dist-upgrade -y
reboot
```

### Set locale

Edit /etc/default/locale

```bash
LANG=en_US.UTF-8
LC_ALL="en_US.UTF-8"
```

Edit /etc/environment

```bash
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
LANG="en_US.UTF-8"
LC_ALL="en_US.UTF-8"
```

Verify:
```bash
locale
```

Expected output (all set to en_US.UTF-8):
```bash
LANG=en_US.UTF-8
LANGUAGE=
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=en_US.UTF-8
```

### Set timezone and NTP

```bash
apt-get install mc chrony -y
dpkg-reconfigure tzdata
systemctl restart chrony
```

Verify:
```bash
date
```

### Upload ssh public keys

Copy SSH public keys to /root:

* /root/id_rsa_sysadmin.pub
* /root/id_rsa_tunnel.pub

### Create users

Run as root

```bash
adduser sysadmin
adduser tunnel
adduser emms
gpasswd -a sysadmin sudo
usermod -s /bin/false tunnel
```

NOTES:

1. emms → service account that owns EMMS4 files and runs the API service.
2. tunnel → restricted (no-shell) account, used only for SSH tunneling to allow secure remote database access.
3. sysadmin → primary sysadmin account with sudo privileges.

Setup .ssh directories and keys:

```bash
mkdir /home/sysadmin/.ssh
mkdir /home/tunnel/.ssh

mv /root/id_rsa_sysadmin.pub /home/sysadmin/.ssh/authorized_keys
mv /root/id_rsa_tunnel.pub /home/tunnel/.ssh/authorized_keys

chown -R sysadmin:sysadmin /home/sysadmin/.ssh
chown -R tunnel:tunnel /home/tunnel/.ssh

chmod 700 /home/sysadmin/.ssh
chmod 700 /home/tunnel/.ssh
chmod 600 /home/sysadmin/.ssh/authorized_keys
chmod 600 /home/tunnel/.ssh/authorized_keys
```

### Harden SSH

Edit /etc/ssh/sshd_config:

* Disable password authentication
* Disable root login
* Ensure PubkeyAuthentication yes is enabled

IMPORTANT:

Ensure you can log in with your new users before rebooting, otherwise you may lock yourself out.”

Restart to apply:

```bash
reboot
```

## Nginx

### Install nginx and cerbot
```bash
apt install -y nginx certbot python3-certbot-nginx
```

### Start and enable nginx
```bash
systemctl start nginx
systemctl enable nginx
```

### Obtain an SSL Certificate
```bash
certbot --nginx -d api.example.com --redirect
```

NOTE:

Certbot installs an automatic renewal timer, no manual action is required. Certificates auto-renew silently; nginx will pick them up without manual reload.

### Site config

Edit /etc/nginx/sites-available/api.example.com
```nginx
server {

    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate     /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8443;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Enable and reload
```bash
ln -s /etc/nginx/sites-available/api.example.com /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## EMMS4 API deploy

Create the required directory structure under /home/emms:

### Create directory tree

```bash
mkdir -p /home/emms/emms4/backup
mkdir -p /home/emms/emms4/config
mkdir -p /home/emms/emms4/images/banks
mkdir -p /home/emms/emms4/images/clients
mkdir -p /home/emms/emms4/images/officers
mkdir -p /home/emms/emms4/log
mkdir -p /home/emms/emms4/tpl/email
```

### Upload required EMMS4 files

Copy emms4 binaries, configuration and template files to the server:

* /home/emms/emms4/cron
* /home/emms/emms4/emms4
* /home/emms/emms4/config/production.json
* /home/emms/emms4/tpl/email/pin.tpl

### Set ownership and permissions
```bash
chown -R emms:emms /home/emms/emms4
chmod -R 755 /home/emms/emms4
chmod +x /home/emms/emms4/cron
chmod +x /home/emms/emms4/emms4
```
### Warning!

Even if EMMS4 files are in place, do not start the API yet.
The database must be fully set up first.

## Firewall

Use the DigitalOcean cloud firewall instead of `ufw`.  

Inbound rules:

* ICMP
* SSH (22/tcp)
* HTTP (80/tcp) – required for certbot
* HTTPS (443/tcp) – required for nginx

Outbound rules:

* ICMP open
* All TCP/UDP open

IMPORTANT: Digital Ocean uses to block 465 and 587 ports. If that is the case, request DO to open those ports, otherwise features that involve sending an email like the pin request for password reset won't work.