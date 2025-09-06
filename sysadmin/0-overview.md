---
title: Overview
layout: page
---

This document provides a high-level overview of EMMS4 architecture and daily operations.  
It is the starting point for sysadmins deploying and maintaining EMMS4.


<br>
## Core Architecture
---
<br>
    

       +-------------------+
       | Angular Frontends |
       |-------------------|
       |  /u  Users        |
       |  /c  Clients      |
       |  /s  Sponsors     |
       |  /p  Public       |
       +-------------------+
                 ^
                 |
                 v
       +-------------------+
       |       Nginx       |
       |     (Reverse      |
       |    Proxy + SSL)   |
       +-------------------+
                 ^
                 |
                 v
       +-------------------+
       |     EMMS4 API     |
       |    (Go Binary)    |        +----------------+
       |-------------------|        |                |
       |      Modules      | <----- |  Config (.ini) |
       |-------------------|        |                |
       |  /u  Users        |        +----------------+
       |  /c  Clients      |                 |
       |  /s  Sponsors     |                 |
       |  /p  Public       |                 |
       +-------------------+                 |
                 ^                           |
                 |                           |
                 v                           v  
       +-------------------+        +----------------+
       |      MariaDB      |        |     Cron       |
       | (Loans, Payments, | <----> |  (Go Binary)   |
       |    Arrears, … )   |        |                |
       +-------------------+        +----------------+


<br>
## Components
---
<br>

EMMS4 is made of six main parts:

1. **MariaDB database**  
   Stores all data: loans, payments, arrears, calendars, users, etc.

2. **EMMS4 API (`backend`)**  
   - Written in Golang, shipped as a binary (`emms4`).  
   - Exposes REST APIs under four modules:
     - `/u` → **Users** (MFI staff: officers, managers, finance, operations).  
     - `/c` → **Clients** (borrowers: check loan status, payments, arrears).  
     - `/s` → **Sponsors** (donors: see use of funds, transparency reports).  
     - `/p` → **Public** (open access: stats for embedding in websites, live MFI stats, borrower stories, donation campaigns, etc.).  
   - Each module has **separate authentication** (except `/p`, which is public).  

3. **Frontends**  
   - Written in Angular, shipped as Angular builds.
   - Four frontends, one per module.  
   - Served via Nginx.  
   - Each consumes only its own API scope (`/u`, `/c`, `/s`, `/p`).

4. **Nginx**  
   - Serves static Angular frontends.  
   - Acts as reverse proxy for the API backend.  
   - Terminates SSL using Let’s Encrypt certificates.

5. **Cron**  
   - Written in Golang, shipped as a binary (`cron`).  
   - Executes ~40 stored procedures in MariaDB once per day.  
   - Handles arrears rebuilding, loan calendar updates, performance metrics, and reporting tables.  
   - **If cron fails, the database can become inconsistent.**  
     That is why daily pre- and post-backups are critical.  
   - EMMS4 will not start unless cron has completed successfully.  

6. **Config**  
   - This is a plain text file in .ini format.  
   - Holds smtp and database connection credentials, domain name, listening port, ssl certs path, the cron stored procedures list, etc.

**WARNING:** 
- Never skip the cron step in production, even if the system seems to run fine.  
- Doing so will accumulate inconsistencies in arrears and reporting data.



<br>
## Daily Cycles
---
<br>

EMMS4 runs in strict daily cycles to ensure consistency, backups, and fresh arrears.  

**Default cycle (via crontab):**

1. **00:00 – Stop APIs**  
   `systemctl stop emms4`

2. **00:30 – Pre-backup**  
   Full dump of the `emms4` database (7-day rotation).  

3. **01:00 – Cron tasks**  
   The `cron` binary runs all maintenance procedures.  
   If it fails, sysadmin must step in.  
   Failure can leave the DB in an inconsistent state.

4. **01:30 – Post-backup**  
   Another full dump of the DB, captured after cron.  

5. **02:00 – Start APIs**  
   `systemctl start emms4`

**Result:**  
- Two backups per day (pre + post).  
- Backups rotate weekly, keeping **7 days of rolling history**.  
- Users always log in to a freshly maintained system in the morning.


<br>
## Responsibilities
---
<br>

Sysadmins are responsible for:

- Ensuring daily cycles run successfully.  
- Checking `/var/log/emms4/cron.log` for cron completion.  
- Verifying backups are rotated properly.  
- Restoring from backup if cron execution causes inconsistencies.  
- Keeping Nginx + certbot healthy for SSL renewals.  
- Restarting services when needed.  


<br>
## Quick Health Check
---
<br>

Run these checks after setup, or anytime you suspect issues:

```bash
# Is the API running?
systemctl status emms4

# Can we reach the DB?
mariadb -e "SHOW DATABASES;"

# Did cron run last night? (look for errors)
tail -n 50 /var/log/emms4/cron.log

# Are backups rotating? (should see 7 weekday files)
ls -lh /home/emms/emms4/backup

# Is Nginx healthy?
systemctl status nginx
```

If any of these fail:

- Restart the failing service.
- If cron errors are found, check procedures and consider restoring from the pre-backup.