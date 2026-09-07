# Oracle Database 26ai Post-Installation Guide — Oracle Linux 9.6

> **Platform:** Oracle Linux 9.6 on VMware Workstation 16.0.0 | **Purpose:** Configure Oracle Database 26ai for automatic startup, verify installation, and complete essential post-installation tasks

| | |
|---|---|
| **Document** | Post-Installation Guide |
| **OS Version** | Oracle Linux 9.6 |
| **Platform** | VMware Workstation 16.0.0 |
| **Oracle Version** | Oracle Database 26ai (23.26.1.0.0) |
| **Edition** | Enterprise Edition |
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
   - [7.2 Enterprise Manager — Not Configured in This Deployment](#72-enterprise-manager--not-configured-in-this-deployment)
   - [7.3 Verify /etc/oratab Entry](#73-verify-etcoratab-entry)
   - [7.4 Configure Oracle Environment for root User](#74-configure-oracle-environment-for-root-user)
   - [7.5 Set Oracle Password Expiration Policy](#75-set-oracle-password-expiration-policy)
   - [7.6 Enable Archiving Before Running RMAN Backups](#76-enable-archiving-before-running-rman-backups)
   - [7.7 Perform an Initial RMAN Backup](#77-perform-an-initial-rman-backup)
10. [Summary of Post-Installation Configurations](#summary-of-post-installation-configurations)
11. [Next Steps](#next-steps)
12. [References](#references)

---

## 1. Overview

This guide covers the essential post-installation configuration steps for **Oracle Database 26ai** on **Oracle Linux 9.6** after the database software has been installed and the initial database (`orcl26ai`) has been created.

**Objectives of this guide:**

| Objective | Description |
|-----------|-------------|
| **Auto-start on boot** | Configure Oracle Database and Listener to start automatically when the Linux server boots |
| **systemd integration** | Register Oracle as a managed systemd service for reliable startup, shutdown, and status monitoring |
| **PDB persistence** | Ensure the Pluggable Database (`orcl26aipdb1`) opens automatically with the CDB |
| **Verification** | Confirm the database, listener, and background processes are fully operational after reboot |

> **User context:**
> - Commands prefixed with `#` are run as the **`root`** OS user
> - Commands prefixed with `$` are run as the **`oracle`** OS user

---

## 2. Prerequisites

| Item | Status |
|------|--------|
| Oracle Linux 9.6 OS installed | ✅ Complete — see [Oracle Linux 9.6 OS Installation Guide](https://github.com/seeomkus/linux-installation/blob/main/oraclelinux-9-for-oracle-database/oraclelinux_9_6_os_installation_guide.md) |
| Oracle 26ai pre-installation completed | ✅ Complete — see [Pre-Installation Guide](oracle_database_26ai_pre_installation_guide_oraclelinux_9_6.md) |
| Oracle 26ai software installed and database created | ✅ Complete — see [Installation Guide](oracle_database_26ai_installation_guide_oraclelinux_9_6.md) |
| Oracle SID | `orcl26ai` |
| Oracle Home | `/u01/app/oracle/product/26ai/dbhome_1` |
| Oracle Base | `/u01/app/oracle` |
| Listener | `LISTENER` on port `1521` |
| PDB | `orcl26aipdb1` |
| Archive Mode | Not enabled during installation (addressed in [7.6](#76-enable-archiving-before-running-rman-backups)) |

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

View the current entry:

```bash
[root@oradb26 ~]# cat /etc/oratab
orcl26ai:/u01/app/oracle/product/26ai/dbhome_1:N
```

Open the file for editing:

```bash
[root@oradb26 ~]# vi /etc/oratab
```

Change the trailing flag from `N` to `Y`.

---

### 1.3 Verify the Change

```bash
[root@oradb26 ~]# cat /etc/oratab
orcl26ai:/u01/app/oracle/product/26ai/dbhome_1:Y
[root@oradb26 ~]#
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
[root@oradb26 ~]# tee /etc/systemd/system/oracle-dbora.service >/dev/null <<'EOF'
[Unit]
Description=Oracle Database & Listener Startup/Shutdown
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
User=oracle
Group=oinstall
Environment="ORACLE_BASE=/u01/app/oracle"
Environment="ORACLE_HOME=/u01/app/oracle/product/26ai/dbhome_1"
Environment="ORACLE_SID=orcl26ai"
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
[root@oradb26 ~]# cat /etc/systemd/system/oracle-dbora.service
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
Environment="ORACLE_HOME=/u01/app/oracle/product/26ai/dbhome_1"
Environment="ORACLE_SID=orcl26ai"
ExecStart=/bin/sh -c '${ORACLE_HOME}/bin/lsnrctl start && ${ORACLE_HOME}/bin/dbstart ${ORACLE_HOME}'
ExecStop=/bin/sh -c '${ORACLE_HOME}/bin/dbshut ${ORACLE_HOME} && ${ORACLE_HOME}/bin/lsnrctl stop'
Restart=no
TimeoutSec=600

[Install]
WantedBy=multi-user.target
[root@oradb26 ~]#
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
| `Environment="ORACLE_HOME=..."` | `/u01/app/oracle/product/26ai/dbhome_1` | Sets `ORACLE_HOME` — points to Oracle binaries |
| `Environment="ORACLE_SID=..."` | `orcl26ai` | Identifies which database instance to start/stop — must match the SID in `/etc/oratab` |
| `ExecStart` | `lsnrctl start && dbstart $ORACLE_HOME` | Startup sequence: start the Listener first, then start the database |
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
[root@oradb26 ~]# chmod 750 /u01/app/oracle/product/26ai/dbhome_1/bin/dbstart
[root@oradb26 ~]# chmod 750 /u01/app/oracle/product/26ai/dbhome_1/bin/dbshut
```

| Permission | Meaning |
|-----------|---------|
| `7` (owner = `oracle`) | Read + Write + Execute |
| `5` (group = `oinstall`) | Read + Execute |
| `0` (others) | No access |

> This ensures only the `oracle` user and members of the `oinstall` group can execute these scripts — consistent with Oracle security best practices.

---

## Part 4 — Clean Shutdown Before Enabling systemd Service

Before enabling the systemd service, perform a **clean shutdown** of the database and listener. This ensures the systemd service starts Oracle from a fully stopped state on its first run, with no orphaned processes.

### 4.1 Shut Down the Oracle Database

```bash
[oracle@oradb26 ~]$ sqlplus / as sysdba

SQL*Plus: Release 23.26.1.0.0 - Production on Mon Sep 7 14:20:32 2026
Version 23.26.1.0.0

Copyright (c) 1982, 2025, Oracle.  All rights reserved.

Connected to:
Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0

SQL> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SQL> exit
Disconnected from Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0
```

> **`shutdown immediate`** closes all active connections, rolls back any uncommitted transactions, and shuts down the database cleanly. It is the recommended shutdown mode for planned maintenance.

---

### 4.2 Stop the Oracle Listener

```bash
[oracle@oradb26 ~]$ lsnrctl stop

LSNRCTL for Linux: Version 23.26.1.0.0 - Production on 07-SEP-2026 14:20:45

Copyright (c) 1991, 2026, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oradb26.company.com)(PORT=1521)))
The command completed successfully
```

---

## Part 5 — Enable and Start the Oracle systemd Service

### 5.1 Reload systemd Daemon

After creating a new service unit file, reload the systemd daemon to register the new service:

```bash
[root@oradb26 ~]# systemctl daemon-reload
```

---

### 5.2 Enable and Start the Service

Enable the service to start automatically on boot AND start it immediately in the current session:

```bash
[root@oradb26 ~]# systemctl enable --now oracle-dbora
Created symlink /etc/systemd/system/multi-user.target.wants/oracle-dbora.service → /etc/systemd/system/oracle-dbora.service.
```

| Flag | Effect |
|------|--------|
| `enable` | Creates the symlink in `/etc/systemd/system/multi-user.target.wants/` — service will start on every boot |
| `--now` | Also starts the service immediately without requiring a reboot |

---

### 5.3 Verify Service Status

```bash
[root@oradb26 ~]# systemctl status --no-pager oracle-dbora
```

**Expected output (active/running state):**

```
● oracle-dbora.service - Oracle Database & Listener Startup/Shutdown
     Loaded: loaded (/etc/systemd/system/oracle-dbora.service; enabled; preset: disabled)
     Active: active (running) since Mon 2026-09-07 14:21:29 WIB; 1s ago
    Process: 14821 ExecStart=/bin/sh -c ${ORACLE_HOME}/bin/lsnrctl start && ${ORACLE_HOME}/bin/dbstart ${ORACLE_HOME} (code=exited, status=0/SUCCESS)
      Tasks: 109 (limit: 34557)
     Memory: 2.1G (peak: 2.1G)
        CPU: 8.604s
     CGroup: /system.slice/oracle-dbora.service
             ├─14825 /u01/app/oracle/product/26ai/dbhome_1/bin/tnslsnr LISTENER -inherit
             ├─14931 ora_pmon_orcl26ai
             ├─14935 ora_clmn_orcl26ai
             ├─14939 ora_psp0_orcl26ai
             ├─14943 ora_vktm_orcl26ai
             ├─14952 ora_gen0_orcl26ai
             ├─14956 ora_mman_orcl26ai
             ├─14962 ora_gen2_orcl26ai
             ├─14964 ora_diag_orcl26ai
             ├─14966 ora_ofsd_orcl26ai
             ├─14968 ora_gwpd_orcl26ai
             ├─14970 ora_dbrm_orcl26ai
             ├─14972 ora_vkrm_orcl26ai
             ├─14974 ora_svcb_orcl26ai
             ├─14976 ora_pman_orcl26ai
             ├─14978 ora_dia0_orcl26ai
             ├─14980 ora_lmhb_orcl26ai
             ├─14982 ora_dbw0_orcl26ai
             ├─14984 ora_lgwr_orcl26ai
             ├─14986 ora_ckpt_orcl26ai
             ├─14990 ora_smon_orcl26ai
             ├─14992 ora_smco_orcl26ai
             ├─14994 ora_reco_orcl26ai
             ├─14996 ora_lreg_orcl26ai
             ├─14998 ora_pxmn_orcl26ai
             ├─15004 ora_mmon_orcl26ai
             ├─15006 ora_mmnl_orcl26ai
             ├─15026 ora_bg00_orcl26ai
             ├─15028 ora_nmon_orcl26ai
             ├─15030 ora_lg00_orcl26ai
             ├─15043 ora_w000_orcl26ai
             ├─15045 ora_bg01_orcl26ai
             ├─15047 ora_m000_orcl26ai
             ├─15052 ora_lg01_orcl26ai
             ├─15058 ora_w001_orcl26ai
             ├─15060 ora_bg02_orcl26ai
             ├─15066 ora_bg03_orcl26ai
             ├─15079 ora_dt00_orcl26ai
             ├─15081 ora_dt01_orcl26ai
             ├─15088 ora_d000_orcl26ai
             ├─15090 ora_s000_orcl26ai
             ├─15092 ora_tmon_orcl26ai
             ├─15098 ora_rcbg_orcl26ai
             ├─15101 ora_tt00_orcl26ai
             ├─15103 ora_tt01_orcl26ai
             ├─15107 ora_gcw0_orcl26ai
             ├─15109 ora_gcr0_orcl26ai
             ├─15114 ora_p000_orcl26ai
             ├─15116 ora_p001_orcl26ai
             ├─15118 ora_p002_orcl26ai
             ├─15120 ora_p003_orcl26ai
             ├─15122 ora_gcr1_orcl26ai
             ├─15124 ora_cjq0_orcl26ai
             ├─15143 /bin/sh .../QOpatch/qopiprep.bat ...
             ├─15192 /bin/sh .../OPatch/opatch lsinventory -customLogDir ...
             ├─15206 ora_aqpc_orcl26ai
             ├─15296 ora_j000_orcl26ai
             ├─15299 ora_j001_orcl26ai
             └─15358 .../jdk/bin/java -javaagent:.../OPatch/jlib/oracle.opatch.classpath.jar ...

Sep 07 14:21:23 oradb26.company.com sh[14823]: SNMP                      OFF
Sep 07 14:21:23 oradb26.company.com sh[14823]: Listener Parameter File   /u01/app/oracle/product/26ai/dbhome_1/network/admin/listener.ora
Sep 07 14:21:23 oradb26.company.com sh[14823]: Listener Log File         /u01/app/oracle/diag/tnslsnr/oradb26/listener/alert/log.xml
Sep 07 14:21:23 oradb26.company.com sh[14823]: Listening Endpoints Summary...
Sep 07 14:21:23 oradb26.company.com sh[14823]:   (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oradb26.company.com)(PORT=1521)))
Sep 07 14:21:23 oradb26.company.com sh[14823]:   (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
Sep 07 14:21:23 oradb26.company.com sh[14823]: The listener supports no services
Sep 07 14:21:23 oradb26.company.com sh[14823]: The command completed successfully
Sep 07 14:21:23 oradb26.company.com sh[14873]: Processing Database instance "orcl26ai": log file /u01/app/oracle/product/26ai/dbhome_1/rdbms/log/startup.log
Sep 07 14:21:29 oradb26.company.com systemd[1]: Started Oracle Database & Listener Startup/Shutdown.
[root@oradb26 ~]#
```

**Status field meanings:**

| Status | Meaning |
|--------|---------|
| `enabled` | Service is registered to start at boot |
| `active (running)` | Service started successfully and Oracle processes are running |
| `inactive (dead)` | Service has not started — check logs with `journalctl -u oracle-dbora` |
| `failed` | Service encountered an error — review logs for details |

> **Larger process tree than earlier Oracle releases.** Oracle Database 26ai starts noticeably more background processes than 19c (`ora_ofsd`, `ora_gwpd`, `ora_dbrm`, `ora_vkrm`, `ora_pxmn`, and others), along with an automatic `OPatch lsinventory` run and a Java-based OPatch agent during instance startup. This is expected — it reflects the additional AI Database and diagnostic infrastructure bundled with this release.

---

## Part 6 — Reboot and Verify Auto-Start

Reboot the server to confirm that Oracle Database and the Listener start automatically without manual intervention.

### 6.1 Reboot

```bash
[root@oradb26 ~]# reboot now
```

Wait for the server to fully restart. Then reconnect via VNC or SSH.

---

### 6.2 Verify Listener Auto-Start

After reboot, switch to the `oracle` user and check the listener:

```bash
[oracle@oradb26 ~]$ lsnrctl status

LSNRCTL for Linux: Version 23.26.1.0.0 - Production on 07-SEP-2026 14:23:30

Copyright (c) 1991, 2026, Oracle.  All rights reserved.

Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oradb26.company.com)(PORT=1521)))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 23.26.1.0.0 - Production
Start Date                07-SEP-2026 14:22:43
Uptime                    0 days 0 hr. 0 min. 47 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/oracle/product/26ai/dbhome_1/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/oradb26/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oradb26.company.com)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=EXTPROC1521)))
The listener supports no services
The command completed successfully
```

> **"The listener supports no services"** at this point is expected — the database instance registers its services with the listener a short time after startup (dynamic registration). Re-run `lsnrctl status` a minute later, or check `lsnrctl services`, to see `orcl26ai` and `orcl26aipdb1` listed once registration completes.

---

### 6.3 Verify Database Auto-Start

Connect to the database as SYSDBA:

```bash
[oracle@oradb26 ~]$ sqlplus / as sysdba

SQL*Plus: Release 23.26.1.0.0 - Production on Mon Sep 7 14:24:10 2026
Version 23.26.1.0.0

Copyright (c) 1982, 2025, Oracle.  All rights reserved.

Connected to:
Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0

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
ORCL26AI  READ WRITE           PRIMARY
```

Exit SQL*Plus:

```sql
SQL> exit
Disconnected from Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0
[oracle@oradb26 ~]$
```

The database is running in `READ WRITE` mode — it is open and ready to accept connections.

---

## Part 7 — Additional Post-Installation Tasks

---

### 7.1 Configure PDB to Open Automatically on Startup

By default, the Pluggable Database (`orcl26aipdb1`) does not automatically open when the CDB (`orcl26ai`) starts — it comes up in `MOUNTED` state. Configure it to open on startup using `SAVE STATE`.

Connect as SYSDBA and check the current PDB state:

```bash
[oracle@oradb26 ~]$ sqlplus / as sysdba

SQL*Plus: Release 23.26.1.0.0 - Production on Mon Sep 7 14:25:20 2026
Version 23.26.1.0.0

Copyright (c) 1982, 2025, Oracle.  All rights reserved.

Connected to:
Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0

SQL> SELECT NAME, OPEN_MODE FROM V$PDBS;

NAME
--------------------------------------------------------------------------------
OPEN_MODE
----------
PDB$SEED
READ ONLY

ORCL26AIPDB1
MOUNTED
```

Open the PDB:

```sql
SQL> ALTER PLUGGABLE DATABASE ORCL26AIPDB1 OPEN;

Pluggable database altered.

SQL> SELECT NAME, OPEN_MODE FROM V$PDBS;

NAME
--------------------------------------------------------------------------------
OPEN_MODE
----------
PDB$SEED
READ ONLY

ORCL26AIPDB1
READ WRITE
```

Save the open state so the PDB automatically opens on the next CDB startup:

```sql
SQL> ALTER PLUGGABLE DATABASE ORCL26AIPDB1 SAVE STATE;

Pluggable database altered.

SQL> SELECT NAME, OPEN_MODE FROM V$PDBS;

NAME
--------------------------------------------------------------------------------
OPEN_MODE
----------
PDB$SEED
READ ONLY

ORCL26AIPDB1
READ WRITE

SQL> exit
Disconnected from Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0
[oracle@oradb26 ~]$
```

> **Why `SAVE STATE`?** Without this, every time the CDB restarts, the PDB remains in `MOUNTED` state and must be opened manually. `SAVE STATE` stores the current open mode in the CDB's data dictionary and restores it automatically on the next startup. To confirm the saved state persisted, query `DBA_PDB_SAVED_STATES` on your system:
> ```sql
> SQL> SELECT CON_NAME, OPEN_MODE FROM DBA_PDB_SAVED_STATES;
> ```

---

### 7.2 Enterprise Manager — Not Configured in This Deployment

During database creation ([Installation Guide, Step 10](oracle_database_26ai_installation_guide_oraclelinux_9_6.md#step-10-specify-management-options)), **"Register with Enterprise Manager (EM) cloud control"** was left unchecked, and the DBCA wizard for this release did not present a separate EM Express port-configuration screen.

As a result, no Enterprise Manager console — Cloud Control or Express — was configured for `orcl26ai` in this installation. If a web-based management console is needed:

- **Enterprise Manager Cloud Control**: requires a separately installed OMS (Oracle Management Service) repository; register this target from an existing Cloud Control installation using `emcli` or the Cloud Control console.
- **Database Express (if available for this release)**: check whether `DBMS_XDB_CONFIG.GETHTTPSPORT()` returns a configured port:
  ```sql
  SQL> SELECT DBMS_XDB_CONFIG.GETHTTPSPORT() FROM DUAL;
  ```
  A `NULL` or `0` result confirms no HTTPS port is currently assigned, consistent with this database not having been configured for a browser-based console during creation.

---

### 7.3 Verify /etc/oratab Entry

Confirm the `/etc/oratab` entry is correctly set with the `Y` flag:

```bash
[oracle@oradb26 ~]$ cat /etc/oratab
```

**Expected output:**

```
orcl26ai:/u01/app/oracle/product/26ai/dbhome_1:Y
```

If the PDB was created with DBCA, an additional line for the PDB may also appear — this is normal.

---

### 7.4 Configure Oracle Environment for root User

To conveniently run Oracle commands as `root` when needed (e.g., checking service status, inspecting logs), add the Oracle environment variables to `/root/.bash_profile`:

```bash
[root@oradb26 ~]# echo ". /home/oracle/scripts/setEnv.sh" >> /root/.bash_profile
```

Or manually export the key variables:

```bash
[root@oradb26 ~]# export ORACLE_HOME=/u01/app/oracle/product/26ai/dbhome_1
[root@oradb26 ~]# export ORACLE_SID=orcl26ai
[root@oradb26 ~]# export PATH=$ORACLE_HOME/bin:$PATH
```

---

### 7.5 Set Oracle Password Expiration Policy

By default, Oracle's `DEFAULT` profile sets passwords to expire every 180 days. For a dedicated internal server or development environment, you may want to set passwords to never expire.

```bash
[oracle@oradb26 ~]$ sqlplus / as sysdba

SQL*Plus: Release 23.26.1.0.0 - Production on Mon Sep 7 14:28:23 2026
Version 23.26.1.0.0

Copyright (c) 1982, 2025, Oracle.  All rights reserved.

Connected to:
Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0
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
PROFILE                                                                         
--------------------------------------------------------------------------------
RESOURCE_NAME
--------------------------------
LIMIT
--------------------------------------------------------------------------------
DEFAULT
PASSWORD_LIFE_TIME
180
```

To set passwords to never expire (development/test environment):

```sql
SQL> ALTER PROFILE DEFAULT LIMIT PASSWORD_LIFE_TIME UNLIMITED;

Profile altered.
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
PROFILE                                                                         
--------------------------------------------------------------------------------
RESOURCE_NAME
--------------------------------
LIMIT
--------------------------------------------------------------------------------
DEFAULT
PASSWORD_LIFE_TIME
UNLIMITED
```

```sql
SQL> exit
Disconnected from Oracle AI Database 26ai Enterprise Edition Release 23.26.1.0.0 - Production
Version 23.26.1.0.0
[oracle@oradb26 ~]$
```

> **For production environments:** Keep the password expiration policy active and implement a proper password rotation process. Setting `UNLIMITED` is appropriate only for non-production, isolated environments.

---

### 7.6 Enable Archiving Before Running RMAN Backups

The database was created with **"Enable archiving" left unchecked** ([Installation Guide, Step 6](oracle_database_26ai_installation_guide_oraclelinux_9_6.md#step-6-select-fast-recovery-option)), so `orcl26ai` is currently in `NOARCHIVELOG` mode. Online (hot) RMAN backups and `BACKUP ... PLUS ARCHIVELOG` require `ARCHIVELOG` mode.

Check the current log mode:

```bash
[oracle@oradb26 ~]$ sqlplus / as sysdba
```

```sql
SQL> SELECT LOG_MODE FROM V$DATABASE;
```

If the result is `NOARCHIVELOG`, enable archiving:

```sql
SQL> SHUTDOWN IMMEDIATE;
SQL> STARTUP MOUNT;
SQL> ALTER DATABASE ARCHIVELOG;
SQL> ALTER DATABASE OPEN;
SQL> SELECT LOG_MODE FROM V$DATABASE;
```

The result should now show `ARCHIVELOG`. Skip this step if your environment intentionally remains in `NOARCHIVELOG` mode (for example, a disposable development database using only cold/offline backups).

> **Why enable archiving?** `ARCHIVELOG` mode ensures that all redo log changes are archived before the redo log files are overwritten. This is a prerequisite for online (hot) backups with RMAN and for Oracle Data Guard (standby database).

---

### 7.7 Perform an Initial RMAN Backup

Establish a backup baseline immediately after database creation. This ensures a recovery point exists before any data is loaded.

```bash
[oracle@oradb26 ~]$ rman target /
```

If archiving was enabled in [7.6](#76-enable-archiving-before-running-rman-backups):

```
RMAN> BACKUP DATABASE PLUS ARCHIVELOG;
```

If the database remains in `NOARCHIVELOG` mode, take a consistent (cold) backup instead:

```
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;
RMAN> BACKUP DATABASE;
RMAN> ALTER DATABASE OPEN;
```

After the backup completes, verify it:

```
RMAN> LIST BACKUP SUMMARY;
```

Exit RMAN:

```
RMAN> EXIT;
```

> **Backup output location:** RMAN backups are stored in the Flash Recovery Area (`/u04/orafra/`) by default, as configured during DBCA. Backup files are managed automatically by RMAN within the FRA size limit (100 GB).

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
| **Management** | Enterprise Manager | Not configured (see [7.2](#72-enterprise-manager--not-configured-in-this-deployment)) |
| **Security** | Script permissions | `dbstart`, `dbshut` — chmod 750 |
| **Archiving** | Log mode | Addressed in [7.6](#76-enable-archiving-before-running-rman-backups) — `NOARCHIVELOG` by default, can be switched to `ARCHIVELOG` |
| **Backup** | Initial RMAN backup | Baseline established in `/u04/orafra/` |

---

## Next Steps

The Oracle Database 26ai installation and post-installation configuration is now complete. The full installation series for Oracle Database 26ai on Oracle Linux 9.6 is finished:

| Stage | Document | Status | Description |
|-------|----------|--------|-------------|
| **1** | [Oracle Linux 9.6 OS Installation Guide](https://github.com/seeomkus/linux-installation/blob/main/oraclelinux-9-for-oracle-database/oraclelinux_9_6_os_installation_guide.md) | ✅ Complete | Operating system installation |
| **2** | [Oracle Database 26ai Pre-Installation Guide](oracle_database_26ai_pre_installation_guide_oraclelinux_9_6.md) | ✅ Complete | OS preparation |
| **3** | [Oracle Database 26ai Installation Guide](oracle_database_26ai_installation_guide_oraclelinux_9_6.md) | ✅ Complete | Software + database creation |
| **4** | **Post-Installation Guide** *(this document)* | ✅ Complete | Auto-start, PDB persistence, verification |

**Recommended ongoing tasks:**

| Task | Frequency | Description |
|------|-----------|-------------|
| RMAN database backup | Daily | Back up the database (and archive logs, if `ARCHIVELOG` is enabled) to the FRA |
| Review alert log | Weekly | Check `/u01/app/oracle/diag/rdbms/orcl26ai/orcl26ai/trace/alert_orcl26ai.log` for errors |
| Apply Oracle patches | Quarterly | Apply Oracle Release Updates via OPatch |
| Check tablespace usage | Weekly | Ensure data tablespaces have sufficient free space |
| Monitor FRA usage | Daily | Ensure the FRA (`/u04/orafra`) does not exceed its configured 100 GB size limit |

---

## References

### 1. Oracle Database 26ai — Official Documentation

| Document | URL |
|----------|-----|
| **Oracle Database Administrator's Guide** | https://docs.oracle.com/en/database/oracle/oracle-database/ |
| **Oracle Database Backup and Recovery (RMAN)** | https://docs.oracle.com/en/database/oracle/oracle-database/ |
| **Oracle Multitenant Administrator's Guide (CDB/PDB)** | https://docs.oracle.com/en/database/oracle/oracle-database/ |
| **Oracle Net Services Administrator's Guide** | https://docs.oracle.com/en/database/oracle/oracle-database/ |
| **Oracle Database Security Guide** | https://docs.oracle.com/en/database/oracle/oracle-database/ |
| **Oracle Enterprise Manager Documentation** | https://docs.oracle.com/en/enterprise-manager/ |

### 2. Oracle Linux 9 — Related Documentation

| Document | URL |
|----------|-----|
| **Oracle Linux Documentation** | https://docs.oracle.com/en/operating-systems/oracle-linux/ |
| **Oracle Linux 9 Release Notes** | https://docs.oracle.com/en/operating-systems/oracle-linux/9/relnotes9.6/ |

### 3. Supporting Tools

| Tool | Purpose | URL |
|------|---------|-----|
| **RealVNC Viewer** | VNC client — remote desktop access | https://www.realvnc.com/en/connect/download/viewer/ |
| **PuTTY** | SSH client — terminal access | https://www.putty.org/ |
| **MobaXterm** | All-in-one SSH + SFTP + X11 | https://mobaxterm.mobatek.net/ |
