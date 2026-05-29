# Oracle Installation

A collection of Oracle Database installation guides across various platforms and OS versions.

---

## Guide List

### Oracle Database 19c on AlmaLinux 8.10

| Stage | Guide | Description |
|-------|-------|-------------|
| Stage 1 | [AlmaLinux 8.10 OS Installation Guide](https://github.com/seeomkus/linux-installation/blob/main/almalinux-8-for-oracle-database/almalinux_8_10_os_installation_guide.md) | AlmaLinux 8.10 operating system installation on VMware Workstation 16 |
| Stage 2 | [Oracle Database 19c Pre-Installation Guide](oracle-database-19c-linux-al8/oracle_database_19c_pre_installation_guide_almalinux_8_10.md) | OS pre-configuration: packages, kernel parameters, storage, Oracle user, and VNC Server setup |
| Stage 3 | [Oracle Database 19c Installation Guide](oracle-database-19c-linux-al8/oracle_database_19c_installation_guide_almalinux_8_10.md) | Oracle Universal Installer (OUI) with Release Update, netca listener, DBCA database creation |
| Stage 4 | Post-Installation Guide *(coming soon)* | Listener configuration, installation validation, and initial tuning |

---

### Oracle Database 11g R2 on Oracle Linux 6

| Stage | Guide | Description |
|-------|-------|-------------|
| — | [Oracle Database 11g R2 Installation on Oracle Linux 6](oracle-database-11g-linux-ol6/oracle-database-11g-installation-linux-ol6.md) | Complete installation guide for Oracle Database 11g Release 2 (11.2.0.1) on Oracle Linux 6 |

---

## Directory Structure

```
oracle-installation/
├── oracle-database-19c-linux-al8/
│   ├── oracle_database_19c_pre_installation_guide_almalinux_8_10.md
│   ├── oracle_database_19c_installation_guide_almalinux_8_10.md
│   └── images/
│       ├── image1_oracle_database_19c_pre_installation_guide_almalinux_8_10.png
│       ├── ... (image2 – image8)
│       ├── image1_oracle_database_19c_installation_guide_almalinux_8_10.png
│       └── ... (image2 – image42)
└── oracle-database-11g-linux-ol6/
    └── oracle-database-11g-installation-linux-ol6.md
```
