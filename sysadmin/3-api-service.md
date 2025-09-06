---
title: API service
layout: page
---

## Create the systemd service

Instead of running emms4 manually, configure it as a systemd service.

```bash
sudo nano /etc/systemd/system/emms4.service
```
**Add the following:**
```ini
[Unit]
Description=EMMS4 API Service
After=network.target mariadb.service nginx.service
Requires=mariadb.service

[Service]
ExecStart=/home/emms/emms4/emms4
WorkingDirectory=/home/emms/emms4
User=emms
Group=emms
Restart=always

# Log output
StandardOutput=append:/home/emms/emms4/log/emms4.log
StandardError=append:/home/emms/emms4/log/emms4.log

# Allow binding to privileged ports (<1024)
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```
Save and exit.

### Reload Systemd
```bash
sudo systemctl daemon-reload
```

## Enable the Service at boot
```bash
sudo systemctl enable emms4
```

## Service operations

Start EMMS4:
```bash
sudo systemctl start emms4
```

Stop EMMS4:
```bash
sudo systemctl stop emms4
```

Check EMMS4 status:
```bash
sudo systemctl status emms4
```

View logs (tail):
```bash
sudo journalctl -u emms4 -f
```

## Manual operations (debug/test)

Run EMMS4 manually from its directory:
```bash
cd /home/emms/emms4
sudo ./emms4
```

Start EMMS4 before daily database maintenance has completed (e.g., after a fresh install or migration):
```bash
cd /home/emms/emms4
sudo ./emms4 --skip-cron-check
```
WARNING:

The --skip-cron-check flag is for testing or migration only.
Do not use it in normal production operations.

Stop EMMS4 when running manually:
Press Ctrl + C.
