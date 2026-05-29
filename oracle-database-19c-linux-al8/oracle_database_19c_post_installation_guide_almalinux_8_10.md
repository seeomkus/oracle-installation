# Oracle Database 19c Post-Installation Guide — AlmaLinux 8.10 (Cerulean Leopard)

> **Platform:** AlmaLinux 8.10 (Cerulean Leopard) on VMware Workstation 16.0.0 | **Purpose:** Configure Oracle Database 19c for automatic startup, verify installation, and complete essential post-installation tasks

| | |
|---|---|
| **Document** | Post-Installation Guide |
| **OS Version** | AlmaLinux 8.10 (Cerulean Leopard) |
| **Platform** | VMware Workstation 16.0.0 |
| **Oracle Version** | Oracle Database 19c (19.3.0.0.0) |
| **Patch Level** | Release Update 19.28.0.0.0 |
| **Kernel** | Linux 4.18.0-553.el8_10.x86_64 |
| **Architecture** | x86-64 |

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Part 1 — Configure /etc/oratab for Auto-Start](#part-1--configure-etcoratab-for-auto-start)
4. [Part 2 — Create systemd Service Unit for Oracle Database](#part-2--create-systemd-service-unit-for-oracle-database)
5. [Part 3 — Set Permissions on dbstart and dbshut](#part-3--set-permissions-on-dbstart-and-dbshut)
6. [Part 4 — Clean Shutdown Before Enabling systemd Service](#part-4--clean-shutdown-before-enabling-systemd-service)
7. [Part 5 — Enable and Start the Oracle systemd Service](#part-5--enable-and-start-the-oracle-systemd-service)
8. [Part 6 — Reboot and Verify Auto-Start](#part-6--reboot-and-verify-auto-start)
9. [Part 7 — Additional Post-Installation Tasks](#part-7--additional-post-installation-tasks)
   - [7.1 Configure PDB to Open Automatically on Startup](#71-configure-pdb-to-open-automatically-on-startup)
   - [7.2 Verify Oracle Enterprise Manager Express](#72-verify-oracle-enterprise-manager-express)
   - [7.3 Verify /etc/oratab Entry](#73-verify-etcoratab-entry)
   - [7.4 Configure Oracle Environment for root User](#74-configure-oracle-environment-for-root-user)
   - [7.5 Set Oracle Password Expiration Policy](#75-set-oracle-password-expiration-policy)
   - [7.6 Perform an Initial RMAN Backup](#76-perform-an-initial-rman-backup)
10. [Summary of Post-Installation Configurations](#summary-of-post-installation-configurations)
11. [Next Steps](#next-steps)
12. [References](#references)

---

## 1. Overview

This guide covers the essential post-installation configuration steps for **Oracle Database 19c** on **AlmaLinux 8.10** after the database software has been installed and the initial database (`orcl19c`) has been created.

**Objectives of this guide:**

| Objective | Description |
|-----------|-------------|
| **Auto-start on boot** | Configure Oracle Database and Listener to start automatically when the Linux server boots |
| **systemd integration** | Register Oracle as a managed systemd service for reliable startup, shutdown, and status monitoring |
| **PDB persistence** | Ensure the Pluggable Database (`orclpdb`) opens automatically with the CDB |
| **Verification** | Confirm the database, listener, and EM Express are fully operational after reboot |

> **User context:**
> - Commands prefixed with `#` are run as the **`root`** OS user
> - Commands prefixed with `$` are run as the **`oracle`** OS user

---

## 2. Prerequisites

| Item | Status |
|------|--------|
| AlmaLinux 8.10 OS installed | ✅ Complete — see [AlmaLinux 8.10 OS Installation Guide](https://github.com/seeomkus/linux-installation/blob/main/almalinux-8-for-oracle-database/almalinux_8_10_os_installation_guide.md) |
| Oracle 19c pre-installation completed | ✅ Complete — see [Pre-Installation Guide](oracle_database_19c_pre_installation_guide_almalinux_8_10.md) |
| Oracle 19c software installed and database created | ✅ Complete — see [Installation Guide](oracle_database_19c_installation_guide_almalinux_8_10.md) |
| Oracle SID | `orcl19c` |
| Oracle Home | `/u01/app/oracle/product/19c/dbhome_1` |
| Oracle Base | `/u01/app/oracle` |
| Listener | `LISTENER` on port `1521` |
| PDB | `orclpdb` |

---

## Part 1 — Configure /etc/oratab for Auto-Start

The `/etc/oratab` file is the system-wide registry of Oracle databases installed on this host. It is created automatically by the Oracle installer and updated by DBCA when a database is created. Each entry controls whether `dbstart` and `dbshut` include that database in automatic start and stop operations.

### 1.1 Understand the /etc/oratab Format

Each line in `/etc/oratab` follows this format:

```
$ORACLE_SID:$ORACLE_HOME:<N|Y>
```

| Field | Description |
|-------|-------------|
| `$ORACLE_SID` | Oracle System Identifier — unique name of the database instance |
| `$ORACLE_HOME` | Full path to the Oracle Home directory |
| `Y` | Include this database in `dbstart` / `dbshut` automatic operations |
| `N` | Exclude this database — must be started/stopped manually |

By default, DBCA sets the flag to **`N`**. This must be changed to **`Y`** to enable automatic startup.

---

### 1.2 Edit /etc/oratab

Open the file for editing as the `oracle` user:

```bash
$ vi /etc/oratab
```

Locate the line for `orcl19c` and change the trailing flag from `N` to `Y`:

**Before:**
```
orcl19c:/u01/app/oracle/product/19c/dbhome_1:N
```

**After:**
```
orcl19c:/u01/app/oracle/product/19c/dbhome_1:Y
```

> **Vi quick reference:** Press `i` to enter insert mode → change `N` to `Y` → press `Esc` → type `:wq` → press `Enter` to save and exit.

---

### 1.3 Verify the Change

```bash
$ cat /etc/oratab
```

**Expected output:**

```
orcl19c:/u01/app/oracle/product/19c/dbhome_1:Y
```

---

## Part 2 — Create systemd Service Unit for Oracle Database

A **systemd service unit** registers Oracle Database and the Oracle Listener as a managed Linux service. This ensures:

- Oracle starts automatically when the server boots (after the network is online)
- Oracle shuts down gracefully when the server is powered off or rebooted
- The service can be managed with standard `systemctl` commands (`start`, `stop`, `restart`, `status`)

### 2.1 Create the Service Unit File

Run the following command as **`root`** to create the service unit file:

```bash
# tee /etc/systemd/system/oracle-dbora.service >/dev/null <<'EOF'
[Unit]
Description=Oracle Database & Listener Startup/Shutdown
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
User=oracle
Group=oinstall
Environment="ORACLE_BASE=/u01/app/oracle"
Environment="ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1"
Environment="ORACLE_SID=orcl19c"
ExecStart=/bin/sh -c '${ORACLE_HOME}/bin/lsnrctl start && ${ORACLE_HOME}/bin/dbstart ${ORACLE_HOME}'
ExecStop=/bin/sh -c '${ORACLE_HOME}/bin/dbshut ${ORACLE_HOME} && ${ORACLE_HOME}/bin/lsnrctl stop'
Restart=no
TimeoutSec=600

[Install]
WantedBy=multi-user.target
EOF
```

---

### 2.2 Verify the Service File Content

```bash
# cat /etc/systemd/system/oracle-dbora.service
```

**Expected output:**

```ini
[Unit]
Description=Oracle Database & Listener Startup/Shutdown
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
User=oracle
Group=oinstall
Environment="ORACLE_BASE=/u01/app/oracle"
Environment="ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1"
Environment="ORACLE_SID=orcl19c"
ExecStart=/bin/sh -c '${ORACLE_HOME}/bin/lsnrctl start && ${ORACLE_HOME}/bin/dbstart ${ORACLE_HOME}'
ExecStop=/bin/sh -c '${ORACLE_HOME}/bin/dbshut ${ORACLE_HOME} && ${ORACLE_HOME}/bin/lsnrctl stop'
Restart=no
TimeoutSec=600

[Install]
WantedBy=multi-user.target
```

---

### 2.3 Service Unit Configuration Reference

#### [Unit] Section

| Directive | Value | Purpose |
|-----------|-------|---------|
| `Description` | `Oracle Database & Listener Startup/Shutdown` | Human-readable service description shown in `systemctl status` |
| `After` | `network-online.target remote-fs.target nss-lookup.target` | Oracle must start **after** these targets are active — ensures network interfaces, remote filesystems, and name resolution are ready |
| `Wants` | `network-online.target` | Declares a soft dependency on network-online — Oracle requires the network to be available for listener connections |

#### [Service] Section

| Directive | Value | Purpose |
|-----------|-------|---------|
| `Type=forking` | — | The service starts a parent process that forks background child processes — Oracle's standard startup model |
| `User=oracle` | — | All service processes run as the `oracle` OS user |
| `Group=oinstall` | — | Primary group for the service processes |
| `Environment="ORACLE_BASE=..."` | `/u01/app/oracle` | Sets `ORACLE_BASE` for the service environment — required by `dbstart`/`dbshut` |
| `Environment="ORACLE_HOME=..."` | `/u01/app/oracle/product/19c/dbhome_1` | Sets `ORACLE_HOME` — points to Oracle binaries |
| `Environment="ORACLE_SID=..."` | `orcl19c` | Identifies which database instance to start/stop — must match the SID in `/etc/oratab` |
| `ExecStart` | `lsnrctl start && dbstart $ORACLE_HOME` | Startup sequence: start the Listener first, then start the database. The Listener must be running before the database attempts to register its services |
| `ExecStop` | `dbshut $ORACLE_HOME && lsnrctl stop` | Shutdown sequence: shut down the database first (gracefully), then stop the Listener |
| `Restart=no` | — | systemd will not automatically restart Oracle after a stop — manual intervention required for unexpected stops |
| `TimeoutSec=600` | — | Allow up to 600 seconds (10 minutes) for startup and shutdown to complete before systemd declares a timeout |

#### [Install] Section

| Directive | Value | Purpose |
|-----------|-------|---------|
| `WantedBy=multi-user.target` | — | The service is enabled in run level 3 (multi-user, network, non-graphical) — the standard server run level |

---

## Part 3 — Set Permissions on dbstart and dbshut

The `dbstart` and `dbshut` Oracle utilities must be executable by the systemd service runner. Set the correct permissions as **`root`**:

```bash
# chmod 750 /u01/app/oracle/product/19c/dbhome_1/bin/dbstart
# chmod 750 /u01/app/oracle/product/19c/dbhome_1/bin/dbshut
```

| Permission | Meaning |
|-----------|---------|
| `7` (owner = `oracle`) | Read + Write + Execute |
| `5` (group = `oinstall`) | Read + Execute |
| `0` (others) | No access |

> This ensures only the `oracle` user and members of the `oinstall` group can execute these scripts — consistent with Oracle security best practices.

---

## Part 4 — Clean Shutdown Before Enabling systemd Service

Before enabling the systemd service, perform a **clean shutdown** of the database and listener. This ensures:

1. The systemd service starts Oracle from a fully stopped state on its first run
2. There are no orphaned Oracle processes that could conflict with the service startup
3. The initial service enable is tested from a known clean state

### 4.1 Shut Down the Oracle Database

Connect as the `oracle` user and shut down the database:

```bash
$ sqlplus / as sysdba
```

```sql
SQL> shutdown immediate
```

```sql
SQL> exit
```

> **`shutdown immediate`** closes all active connections, rolls back any uncommitted transactions, and shuts down the database cleanly. It is the recommended shutdown mode for planned maintenance.

---

### 4.2 Stop the Oracle Listener

```bash
$ lsnrctl stop
```

**Expected output:**

```
Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1521)))
The command completed successfully
```

---

## Part 5 — Enable and Start the Oracle systemd Service

### 5.1 Reload systemd Daemon

After creating a new service unit file, reload the systemd daemon to register the new service:

```bash
# systemctl daemon-reload
```

---

### 5.2 Enable and Start the Service

Enable the service to start automatically on boot AND start it immediately in the current session:

```bash
# systemctl enable --now oracle-dbora
```

| Flag | Effect |
|------|--------|
| `enable` | Creates the symlink in `/etc/systemd/system/multi-user.target.wants/` — service will start on every boot |
| `--now` | Also starts the service immediately without requiring a reboot |

---

### 5.3 Verify Service Status

```bash
# systemctl status --no-pager oracle-dbora
```

**Expected output (active/running state):**

```
● oracle-dbora.service - Oracle Database & Listener Startup/Shutdown
   Loaded: loaded (/etc/systemd/system/oracle-dbora.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2026-05-26 17:00:00 WIB; 30s ago
  Process: XXXX ExecStart=/bin/sh -c ${ORACLE_HOME}/bin/lsnrctl start && ...
 Main PID: XXXX
   CGroup: /system.slice/oracle-dbora.service
```

**Status field meanings:**

| Status | Meaning |
|--------|---------|
| `enabled` | Service is registered to start at boot |
| `active (running)` | Service started successfully and Oracle processes are running |
| `inactive (dead)` | Service has not started — check logs with `journalctl -u oracle-dbora` |
| `failed` | Service encountered an error — review logs for details |

---

## Part 6 — Reboot and Verify Auto-Start

Reboot the server to confirm that Oracle Database and the Listener start automatically without manual intervention.

### 6.1 Reboot

```bash
# reboot now
```

Wait for the server to fully restart (approximately 2–5 minutes). Then reconnect via VNC or SSH.

---

### 6.2 Verify Listener Auto-Start

After reboot, switch to the `oracle` user and check the listener:

```bash
$ lsnrctl status
```

**Expected output:**

```
LSNRCTL for Linux: Version 19.0.0.0.0 - Production on 26-MAY-2026 17:10:00

Copyright (c) 1991, 2025, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=IPC)(KEY=EXTPROC1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux x86_64: Version 19.0.0.0.0
Start Date                26-MAY-2026 17:08:00
Uptime                    0 days 0 hr. 2 min. 0 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/19c/dbhome_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/al8dbora/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=al8dbora.company.com)(PORT=1521)))
Services Summary...
Service "orcl19c" has 1 instance(s).
  Instance "orcl19c", status READY, has 1 handler(s) for this service...
Service "orclpdb" has 1 instance(s).
  Instance "orcl19c", status READY, has 1 handler(s) for this service...
The command completed successfully
```

> Confirm that:
> - `Status: READY` — listener is active
> - Both `orcl19c` (CDB) and `orclpdb` (PDB) services are listed and status is `READY`

---

### 6.3 Verify Database Auto-Start

Connect to the database as SYSDBA:

```bash
$ sqlplus / as sysdba
```

**Expected output:**

```
SQL*Plus: Release 19.0.0.0.0 - Production on Tue May 26 17:10:00 2026
Version 19.28.0.0.0

Copyright (c) 1982, 2025, Oracle.  All rights reserved.

Connected to:
Oracle Database 19c Standard Edition 2 Release 19.0.0.0.0 - Production
Version 19.28.0.0.0

SQL>
```

Check the database status:

```sql
SQL> SELECT NAME, OPEN_MODE, DATABASE_ROLE FROM V$DATABASE;
```

**Expected output:**

```
NAME      OPEN_MODE            DATABASE_ROLE
--------- -------------------- ----------------
ORCL19C   READ WRITE           PRIMARY
```

Exit SQL*Plus:

```sql
SQL> exit
```

The database is running in `READ WRITE` mode — it is open and ready to accept connections.

---

## Part 7 — Additional Post-Installation Tasks

---

### 7.1 Configure PDB to Open Automatically on Startup

By default, the Pluggable Database (`orclpdb`) does not automatically open when the CDB (`orcl19c`) starts. You must configure it to open on startup using `SAVE STATE`.

Connect as SYSDBA and open the PDB:

```bash
$ sqlplus / as sysdba
```

```sql
SQL> ALTER PLUGGABLE DATABASE orclpdb OPEN;

SQL> SELECT NAME, OPEN_MODE FROM V$PDBS;
```

**Expected output after opening:**

```
NAME          OPEN_MODE
------------- ----------
PDB$SEED      READ ONLY
ORCLPDB       READ WRITE
```

Save the open state so the PDB automatically opens on the next CDB startup:

```sql
SQL> ALTER PLUGGABLE DATABASE orclpdb SAVE STATE;
```

Verify the saved state:

```sql
SQL> SELECT CON_NAME, OPEN_MODE FROM DBA_PDB_SAVED_STATES;
```

**Expected output:**

```
CON_NAME     OPEN_MODE
------------ ----------
ORCLPDB      READ WRITE
```

```sql
SQL> exit
```

> **Why `SAVE STATE`?** Without this, every time the CDB restarts, all PDBs remain in `MOUNTED` state and must be opened manually. `SAVE STATE` stores the current open mode in the CDB's data dictionary and restores it automatically on the next startup. This is the recommended approach for PDB auto-open in Oracle 12c and later.

---

### 7.2 Verify Oracle Enterprise Manager Express

Oracle Enterprise Manager Database Express (EM Express) provides a browser-based management console. Verify it is accessible after reboot.

From a browser on your client machine, open:

```
https://192.168.159.134:5500/em
```

| Item | Value |
|------|-------|
| **URL** | `https://al8dbora.company.com:5500/em` |
| **Username** | `sys` |
| **Password** | Password set during DBCA |
| **Container Name** | Leave blank for CDB, or enter `orclpdb` for PDB |
| **Connect as** | `SYSDBA` |

> **Note:** EM Express uses a self-signed SSL certificate by default. Your browser may display a certificate warning — this is expected. Accept the certificate to proceed.

To confirm the EM Express port from SQL*Plus:

```bash
$ sqlplus / as sysdba
```

```sql
SQL> SELECT DBMS_XDB_CONFIG.GETHTTPSPORT() FROM DUAL;
```

**Expected output:**

```
DBMS_XDB_CONFIG.GETHTTPSPORT()
-------------------------------
                           5500
```

```sql
SQL> exit
```

---

### 7.3 Verify /etc/oratab Entry

Confirm the `/etc/oratab` entry is correctly set with the `Y` flag:

```bash
$ cat /etc/oratab
```

**Expected output:**

```
orcl19c:/u01/app/oracle/product/19c/dbhome_1:Y
```

If the PDB was created with DBCA, an additional line for the PDB may also appear — this is normal.

---

### 7.4 Configure Oracle Environment for root User

To conveniently run Oracle commands as `root` when needed (e.g., checking service status, inspecting logs), add the Oracle environment variables to `/root/.bash_profile`:

```bash
# echo ". /home/oracle/scripts/setEnv.sh" >> /root/.bash_profile
```

Or manually export the key variables:

```bash
# export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
# export ORACLE_SID=orcl19c
# export PATH=$ORACLE_HOME/bin:$PATH
```

---

### 7.5 Set Oracle Password Expiration Policy

By default, Oracle's `DEFAULT` profile sets passwords to expire every 180 days. For a dedicated internal server or development environment, you may want to set passwords to never expire.

```bash
$ sqlplus / as sysdba
```

Check the current password expiration setting:

```sql
SQL> SELECT PROFILE, RESOURCE_NAME, LIMIT
     FROM DBA_PROFILES
     WHERE RESOURCE_NAME = 'PASSWORD_LIFE_TIME'
     AND PROFILE = 'DEFAULT';
```

**Expected default output:**

```
PROFILE    RESOURCE_NAME        LIMIT
---------- -------------------- -------
DEFAULT    PASSWORD_LIFE_TIME   180
```

To set passwords to never expire (development/test environment):

```sql
SQL> ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;
```

Verify:

```sql
SQL> SELECT PROFILE, RESOURCE_NAME, LIMIT
     FROM DBA_PROFILES
     WHERE RESOURCE_NAME = 'PASSWORD_LIFE_TIME'
     AND PROFILE = 'DEFAULT';
```

**Expected output after change:**

```
PROFILE    RESOURCE_NAME        LIMIT
---------- -------------------- ---------
DEFAULT    PASSWORD_LIFE_TIME   UNLIMITED
```

```sql
SQL> exit
```

> **For production environments:** Keep the password expiration policy active and implement a proper password rotation process. Setting `UNLIMITED` is appropriate only for non-production, isolated environments.

---

### 7.6 Perform an Initial RMAN Backup

Establish a backup baseline immediately after database creation. This ensures a recovery point exists before any data is loaded.

```bash
$ rman target /
```

```
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

RMAN will back up:
1. All datafiles (CDB and PDB)
2. The current control file and server parameter file (SPFILE)
3. All archived redo log files

After the backup completes, verify it:

```
RMAN> LIST BACKUP SUMMARY;
```

Exit RMAN:

```
RMAN> EXIT;
```

> **Backup output location:** RMAN backups are stored in the Flash Recovery Area (`/u04/orafra/`) by default, as configured during DBCA. Backup files are managed automatically by RMAN within the FRA size limit.

---

## Summary of Post-Installation Configurations

| Category | Item | Configured Value / Status |
|----------|------|--------------------------|
| **Auto-Start** | `/etc/oratab` flag | `Y` — database included in `dbstart`/`dbshut` |
| **Auto-Start** | systemd service | `oracle-dbora.service` — enabled in `multi-user.target` |
| **Auto-Start** | Service startup sequence | `lsnrctl start` → `dbstart $ORACLE_HOME` |
| **Auto-Start** | Service shutdown sequence | `dbshut $ORACLE_HOME` → `lsnrctl stop` |
| **Auto-Start** | Service timeout | 600 seconds |
| **Database** | CDB open mode (after reboot) | `READ WRITE` |
| **Database** | PDB open mode (after reboot) | `READ WRITE` (via `SAVE STATE`) |
| **Network** | Listener status (after reboot) | `READY` on port 1521 |
| **Management** | EM Express | Accessible at `https://192.168.159.134:5500/em` |
| **Security** | Script permissions | `dbstart`, `dbshut` — chmod 750 |
| **Backup** | Initial RMAN backup | Complete — baseline established in `/u04/orafra/` |

---

## Next Steps

The Oracle Database 19c installation and post-installation configuration is now complete. The full installation series for Oracle Database 19c on AlmaLinux 8.10 is finished:

| Stage | Document | Status | Description |
|-------|----------|--------|-------------|
| **1** | [AlmaLinux 8.10 OS Installation Guide](https://github.com/seeomkus/linux-installation/blob/main/almalinux-8-for-oracle-database/almalinux_8_10_os_installation_guide.md) | ✅ Complete | Operating system installation |
| **2** | [Pre-Installation Guide](oracle_database_19c_pre_installation_guide_almalinux_8_10.md) | ✅ Complete | OS preparation |
| **3** | [Installation Guide](oracle_database_19c_installation_guide_almalinux_8_10.md) | ✅ Complete | Software + database creation |
| **4** | **Post-Installation Guide** *(this document)* | ✅ Complete | Auto-start, PDB persistence, verification |

**Recommended ongoing tasks:**

| Task | Frequency | Description |
|------|-----------|-------------|
| RMAN database backup | Daily | Back up the database and archive logs to FRA |
| Review alert log | Weekly | Check `/u01/app/oracle/diag/rdbms/orcl19c/orcl19c/trace/alert_orcl19c.log` for errors |
| Apply Oracle patches | Quarterly | Apply Oracle Release Updates via OPatch |
| Check tablespace usage | Weekly | Ensure data and index tablespaces have sufficient free space |
| Monitor FRA usage | Daily | Ensure the FRA (`/u04/orafra`) does not exceed its configured size limit |

---

## References

### 1. Oracle Database 19c — Official Documentation

| Document | URL |
|----------|-----|
| **Oracle Database Administrator's Guide** | https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/ |
| **Oracle Database Backup and Recovery (RMAN)** | https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/ |
| **Oracle Multitenant Administrator's Guide (CDB/PDB)** | https://docs.oracle.com/en/database/oracle/oracle-database/19/multi/ |
| **Oracle Net Services Administrator's Guide** | https://docs.oracle.com/en/database/oracle/oracle-database/19/netag/ |
| **Oracle Database Security Guide** | https://docs.oracle.com/en/database/oracle/oracle-database/19/dbseg/ |
| **Oracle EM Express** | https://docs.oracle.com/en/database/oracle/oracle-database/19/admqs/getting-started-with-database-administration.html |

### 2. AlmaLinux 8 — Related Documentation

| Document | URL |
|----------|-----|
| **AlmaLinux Wiki** | https://wiki.almalinux.org |
| **AlmaLinux 8.10 Release Notes** | https://wiki.almalinux.org/release-notes/8.10.html |

### 3. Supporting Tools

| Tool | Purpose | URL |
|------|---------|-----|
| **RealVNC Viewer** | VNC client — remote desktop access | https://www.realvnc.com/en/connect/download/viewer/ |
| **PuTTY** | SSH client — terminal access | https://www.putty.org/ |
| **MobaXterm** | All-in-one SSH + SFTP + X11 | https://mobaxterm.mobatek.net/ |
