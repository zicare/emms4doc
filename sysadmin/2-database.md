---
title: Database
layout: page
---

EMMS4 requires MariaDB v10.11.13 or later, tested with v10.11.13.

## Install mariadb

Run these commands as root

```bash
apt update 
apt install mariadb-server 
mysql_secure_installation 
systemctl start mariadb 
systemctl enable mariadb 
systemctl status mariadb 
```

Recommended hardening flow in mysql_secure_installation

When prompted:

1. Enter current password for root (enter for none): → press Enter
2. Switch to unix_socket authentication [Y/n] → Y
3. Change the root password? [Y/n] → n
4. Remove anonymous users? [Y/n] → Y
5. Disallow root login remotely? [Y/n] → Y
6. Remove test database and access to it? [Y/n] → Y
7. Reload privilege tables now? [Y/n] → Y

Result:

1. Local root access works passwordless (via unix_socket).
2. Remote root logins are blocked.
3. Anonymous/test accounts are removed.
4. Safe, clean, and suitable for cron-based backups.

## Create emms4 databases

**NOTES**

> * We recommend naming the database `emms4`. If you use another name, update the provided SQL scripts accordingly.
> * For EMMS2 migration: restore a backup of the production EMMS2 database on the EMMS4 server as `emms`. If you use another name, adjust the migration scripts.
> * Migration scripts will modify the source database (`emms`). Always restore from a copy, never run on the live production database.
> * To restart from scratch after a failed setup: truncate `emms4` (and `emms` if migrating), then rerun all setup steps.
> * Example restore from a gzip backup:
```bash
gunzip -c /path/to/emms2bk.gz | mysql emms
```

### Steps

Open a MariaDB shell as root:

```bash
mariadb 
```

From the mariadb terminal run these commands

```sql
create database emms; 
create database emms4; 
create user 'sysadmin'@'localhost' identified by 'CHANGE-ME';
grant all on *.* to 'sysadmin'@'localhost';
flush privileges;
exit;
```

## Structure - Tables and Views

Run the scripts in /db/structure/create in this order:

* 1-tables.sql    
* 2-views.sql 

## Data - Fresh install

For a minimal fresh installation, run the scripts in /db/data in this order:

* 1-roles-and-endpoints.sql  
* 2-fresh.sql  
* 3-performance-metrics.sql
* 4-grants.sql

## Data - Migration from EMMS2

If migrating from EMMS2, run the scripts in /db/data in this order:

* 1-roles-and-endpoints.sql  
* 2-migration.sql  
* 3-performance-metrics.sql
* 3-performance-data.sql (optional)
* 4-grants.sql

## Structure - Procedures and Triggers

Run the scripts in /db/structure/create in this order:

* 3-procedures-logic.sql
* 3-procedures-rpt.sql
* 3-procedures-chart.sql
* 3-procedures-nightly.sql
* 3-procedures-extras.sql
* 4-triggers-log.sql  
* 5-triggers-logic.sql

## Payment calendars - Migrating from EMMS2

EMMS2 stores only actual payments, not full calendars. EMMS4 introduces full payment calendars in the payments table. After migrating from EMMS2 you must rebuild calendars for the active loan portfolio.

Endpoints:

1. Rebuild all calendars

    PATCH /u/calendars (JWT required)

2. Rebuild calendar for one loan

    PATCH /u/calendars/:loan_id (JWT required)

3. To obtain a JWT:

    GET /u/jwt (BASIC auth)

4. Password reset workflow:

    POST /u/pin (no auth)
	
    PATCH /u/pwd (no auth)


**IMPORTANT**
> * To run these endpoints you must first start the EMMS4 backend with --skip-cron-check.
> * Stop the backend again once the calendars are rebuilt.

**WARNING** 
> * The --skip-cron-check flag is for installation/migration/test only. Never use it for regular operations.

## Re-building the arrears table

EMMS4 replaces EMMS2’s tblLoansOnDelinquency with the arrears table. To populate it from the payments table, run the rebuild_arrears procedure once.

This is required if you migrate from EMMS2, and may be useful later in specific recovery scenarios.

Procedure definition: /db/structure/create/3-procedures-extras.sql.

## EMMS4 database daily maintenance

EMMS4 database requires daily maintenance. By default, EMMS4 won't start unless this maintenance has completed, even on the very first run.

### Schedule the maintenance

The maintenance is performed by the bundled cron binary. Schedule it under the emms user’s crontab (not root). Example:

```cron
0 0 * * * /bin/systemctl stop emms4
0 1 * * * cd /home/emms/emms4 && ./cron >> /var/log/emms4/cron.log 2>&1
0 2 * * * /bin/systemctl start emms4
```

This stops the API at midnight, runs database maintenance at 01:00, then restarts the API at 02:00.
The maintenance log will be written to /var/log/emms4/cron.log.

### Manual run - equivalent to the schedule

For a manual run, execute the steps below:

```bash
systemctl stop emms4
cd /home/emms/emms4 && ./cron
systemctl start emms4
```

### Configure the maintenance list

Check config/production.json. The "cron" section defines the ~40 stored procedures executed by the cron binary (this is the “daily maintenance” list).

**WARNING**
> * Adjust this only if you know exactly what you’re changing.

### Skipping the maintenance check

If you need to start EMMS4 before the first daily maintenance has run (e.g., after a fresh install or migration), use the --skip-cron-check flag:

```bash
cd /home/emms/emms4
sudo -u emms ./emms4 --skip-cron-check --verbose
```

## Database backup and restoring

### Backup

Schedule daily backups from the root crontab. Example:

```cron
30 0 * * * mysqldump emms4 --single-transaction --quick --routines --triggers --events --add-drop-table --hex-blob | gzip -c > /home/emms/emms4/backup/emms4-$(date +\%a).pre.gz
30 1 * * * mysqldump emms4 --single-transaction --quick --routines --triggers --events --add-drop-table --hex-blob | gzip -c > /home/emms/emms4/backup/emms4-$(date +\%a).post.gz
```

* Two backups are taken daily: one at 00:30 and another at 01:30.
* Each backup overwrites the file from the same weekday, maintaining a 7-day rotation.

Because root@localhost uses the unix_socket plugin, no password is required, and these jobs run non-interactively.

### Restore

1. Ensure the emms4 database exists and is empty.
2. Restore from any backup file (example: Monday pre-backup):

```bash
sudo gunzip -c /home/emms/emms4/backup/emms4-Mon.pre.gz | mysql emms4
```