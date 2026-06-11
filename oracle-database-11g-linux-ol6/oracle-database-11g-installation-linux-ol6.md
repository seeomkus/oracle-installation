# Oracle Database 11g Release 2 (11.2) Installation On Oracle Linux 6 (OL6)

---

## Prerequisites

Before proceeding with this guide, ensure that Oracle Linux 6.10 has been installed and configured on the target server.

| Stage | Guide | Status |
|-------|-------|--------|
| **1** | [Oracle Linux 6.10 OS Installation Guide](https://github.com/seeomkus/linux-installation/blob/main/oraclelinux-6-for-oracle-database/oraclelinux_6_10_os_installation_guide.md) | ✅ Complete — prerequisite for this guide |
| **2** | **Oracle Database 11g R2 Installation** *(this document)* | ⬜ Current |

---

## SOURCE ARTICLE AND DOWNLOAD FILE:

1. Installation OS Oracle Linux 6.10:

   https://oracle-base.com/articles/11g/oracle-db-11gr2-installation-on-oracle-linux-6

5. Download Oracle Database 11g

   https://edelivery.oracle.com/

---

## Virtual Machine Specifications

| Component | Minimum (Testing) | Recommended (Production) |
|-----------|-------------------|--------------------------|
| **Memory** | 4 GB (works) | 64 GB (better for DB workloads) |
| **CPU** | 2 cores | 32 cores (improves performance) |
| **Disk 1 (OS & Database Software)** | 300 GB | 500 GB |
| **Disk 2 (Datafile)** | 300 GB | 500 GB |
| **Disk 3 (Datafile) is Optional** | 300 GB | 500 GB |
| **Disk 4 (Backup/RMAN/Archivelog)** | 300 GB | 1 TB |
| **Network** | Bridged (OK) | IP Static & Direct Access from Server Database to Storage |

## **Suggested Disk Partitioning Scheme (for Oracle Database)**

- `/dev/sda1` → `/`, `/boot`, `/tmp`, `/home`, `swap` OS
- `/dev/sda2` → `/u01` (Oracle software home)
- `/dev/nvme0n1` → `/u02/oradata` (datafiles, redo logs, control files)
- `/dev/nvme1n1` → `/u03/oraindx` (datafiles, redo logs, control files)
- `/dev/nvme2n1` → `/u04/orafra` (RMAN backups, archive logs)
- `/dev/nvme2n1` → `/u04/dump` (RMAN backups, archive logs)
- `/dev/nvme2n1` → `/u04/backup` (RMAN backups, archive logs)

---

## **A. Oracle Installation Prerequisites**

Complete the prerequisites using the Automatic Setup method described below. The Additional Setup steps are required for all installations.

1. Update OS

   ```bash
   [root@localhost ~]# date
   Fri Oct 24 14:46:07 WIB 2025
   [root@localhost ~]# yum clean all
   [root@localhost ~]# yum repolist
   [root@localhost ~]# yum update -y
   ```

2. Verify the network configuration and confirm internet access

   ```bash
   [root@localhost ~]# ifconfig
   eth0      Link encap:Ethernet  HWaddr xx:xx:xx:xx:xx:xx
             inet addr:<IP_ADDRESS>  Bcast:<BROADCAST_ADDRESS>  Mask:<NETMASK>
             inet6 addr: <IPv6_ADDRESS>/64 Scope:Link
             UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
             RX packets:811580 errors:0 dropped:0 overruns:0 frame:0
             TX packets:119353 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:1000
             RX bytes:1166423197 (1.0 GiB)  TX bytes:9391911 (8.9 MiB)

   lo        Link encap:Local Loopback
             inet addr:127.0.0.1  Mask:255.0.0.0
             inet6 addr: ::1/128 Scope:Host
             UP LOOPBACK RUNNING  MTU:65536  Metric:1
             RX packets:19 errors:0 dropped:0 overruns:0 frame:0
             TX packets:19 errors:0 dropped:0 overruns:0 carrier:0
             collisions:0 txqueuelen:0
             RX bytes:1224 (1.1 KiB)  TX bytes:1224 (1.1 KiB)
   [root@localhost ~]# ping 8.8.8.8
   PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
   64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=12.6 ms
   64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=17.2 ms
   64 bytes from 8.8.8.8: icmp_seq=3 ttl=117 time=12.9 ms
   64 bytes from 8.8.8.8: icmp_seq=4 ttl=117 time=22.4 ms
   ^C
   --- 8.8.8.8 ping statistics ---
   4 packets transmitted, 4 received, 0% packet loss, time 3346ms
   rtt min/avg/max/mdev = 12.648/16.331/22.443/3.978 ms
   ```

3. The `/etc/hosts` file must contain a fully qualified name for the server.

   Before:

   ```bash
   [root@localhost ~]# cat /etc/hosts
   127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
   ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   ```

   Edit the file:

   ```bash
   [root@localhost ~]# vi /etc/hosts
   ```

   After:

   ```bash
   [root@localhost ~]# cat /etc/hosts
   127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
   <IP_ADDRESS>  oradb11g.example.com oradb11g
   ```

4. Change the hostname

   ```bash
   [root@localhost ~]# hostname
   localhost.localdomain
   [root@localhost ~]# cat /etc/sysconfig/network
   NETWORKING=yes
   HOSTNAME=localhost.localdomain
   NTPSERVERARGS=iburst
   [root@localhost ~]# vi /etc/sysconfig/network
   [root@localhost ~]# cat /etc/sysconfig/network
   NETWORKING=yes
   HOSTNAME=oradb11g.example.com
   NTPSERVERARGS=iburst
   ```

   Apply the new hostname immediately with:

   ```bash
   [root@localhost ~]# hostname oradb11g.example.com
   ```

   Then restart the network service:

   ```bash
   [root@localhost ~]# service network restart
   [root@localhost ~]# hostname oradb11g.example.com
   [root@localhost ~]# service network restart
   Shutting down interface eth0:  Device state: 3 (disconnected)
                                                              [  OK  ]
   Shutting down loopback interface:                          [  OK  ]
   Bringing up loopback interface:                            [  OK  ]
   Bringing up interface eth0:  Active connection state: activating
   Active connection path: /org/freedesktop/NetworkManager/ActiveConnection/1
   state: activated
   Connection activated
                                                              [  OK  ]
   [root@localhost ~]# hostname
   oradb11g.example.com
   ```

   You should now see the new hostname (e.g., `oradb11g.example.com`).

   Disconnect from the session and log back in. The prompt should now reflect the new hostname:

   ```bash
   [root@oradb11g ~]# hostname
   oradb11g.example.com
   ```

5. Automatic Setup

   If you plan to use the `oracle-rdbms-server-11gR2-preinstall` package to perform all prerequisite setup automatically, follow the instructions at http://public-yum.oracle.com to set up the yum repository for OL, then run the following command.

   ```bash
   [root@oradb11g ~]# yum install -y oracle-rdbms-server-11gR2-preinstall
   ```

   All necessary prerequisites will be performed automatically.

   It is probably worth doing a full update as well, but this is not strictly speaking necessary.

   ```bash
   [root@oradb11g ~]# yum -y update
   ```

   Verify the current kernel parameters:

   ```bash
   [root@oradb11g ~]# /sbin/sysctl -p
   net.ipv4.ip_forward = 0
   net.ipv4.conf.default.accept_source_route = 0
   kernel.sysrq = 0
   kernel.core_uses_pid = 1
   net.ipv4.tcp_syncookies = 1
   kernel.msgmnb = 65536
   kernel.msgmax = 65536
   fs.file-max = 6815744
   kernel.sem = 250 32000 100 128
   kernel.shmmni = 4096
   kernel.shmall = 4294967296
   kernel.shmmax = 4398046511104
   kernel.panic_on_oops = 1
   net.core.rmem_default = 262144
   net.core.rmem_max = 4194304
   net.core.wmem_default = 262144
   net.core.wmem_max = 1048576
   net.ipv4.conf.all.rp_filter = 2
   net.ipv4.conf.default.rp_filter = 2
   fs.aio-max-nr = 1048576
   net.ipv4.ip_local_port_range = 9000 65500
   ```

   Verify the `/etc/security/limits.conf` file:

   ```bash
   [root@oradb11g ~]# cat /etc/security/limits.conf
   # /etc/security/limits.conf
   #
   #Each line describes a limit for a user in the form:
   #<domain>        <type>  <item>  <value>
   #Where:
   #<domain> can be:
   #        - a user name
   #        - a group name, with @group syntax
   #        - the wildcard *, for default entry
   #        - the wildcard %, can be also used with %group syntax,
   #                 for maxlogin limit
   #<type> can have the two values:
   #        - "soft" for enforcing the soft limits
   #        - "hard" for enforcing hard limits
   #<item> can be one of the following:
   #        - core - limits the core file size (KB)
   #        - data - max data size (KB)
   #        - fsize - maximum filesize (KB)
   #        - memlock - max locked-in-memory address space (KB)
   #        - nofile - max number of open file descriptors
   #        - rss - max resident set size (KB)
   #        - stack - max stack size (KB)
   #        - cpu - max CPU time (MIN)
   #        - nproc - max number of processes
   #        - as - address space limit (KB)
   #        - maxlogins - max number of logins for this user
   #        - maxsyslogins - max number of logins on the system
   #        - priority - the priority to run user process with
   #        - locks - max number of file locks the user can hold
   #        - sigpending - max number of pending signals
   #        - msgqueue - max memory used by POSIX message queues (bytes)
   #        - nice - max nice priority allowed to raise to values: [-20, 19]
   #        - rtprio - max realtime priority
   #<domain>      <type>  <item>         <value>

   #*               soft    core            0
   #*               hard    rss             10000
   #@student        hard    nproc           20
   #@faculty        soft    nproc           20
   #@faculty        hard    nproc           50
   #ftp             hard    nproc           0
   #@student        -       maxlogins       4
   # End of file

   # oracle-rdbms-server-11gR2-preinstall setting for nofile soft limit is 1024
   oracle   soft   nofile    1024

   # oracle-rdbms-server-11gR2-preinstall setting for nofile hard limit is 65536
   oracle   hard   nofile    65536

   # oracle-rdbms-server-11gR2-preinstall setting for nproc soft limit is 16384
   # refer orabug15971421 for more info.
   oracle   soft   nproc    16384

   # oracle-rdbms-server-11gR2-preinstall setting for nproc hard limit is 16384
   oracle   hard   nproc    16384

   # oracle-rdbms-server-11gR2-preinstall setting for stack soft limit is 10240KB
   oracle   soft   stack    10240

   # oracle-rdbms-server-11gR2-preinstall setting for stack hard limit is 32768KB
   oracle   hard   stack    32768

   # oracle-rdbms-server-11gR2-preinstall setting for memlock hard limit is maximum of 128GB on x86_64 or 3GB on x86 OR 90 % of RAM
   oracle   hard   memlock    134217728

   # oracle-rdbms-server-11gR2-preinstall setting for memlock soft limit is maximum of 128GB on x86_64 or 3GB on x86 OR 90 % of RAM
   oracle   soft   memlock    134217728
   ```

5. The following package groups were selected during OS installation:

   - Base System > Base
   - Base System > Client management tools
   - Base System > Compatibility libraries
   - Base System > Hardware monitoring utilities
   - Base System > Large Systems Performance
   - Base System > Network file system client
   - Base System > Performance Tools
   - Base System > Perl Support
   - Servers > Server Platform
   - Servers > System administration tools
   - Desktops > Desktop
   - Desktops > Desktop Platform
   - Desktops > Fonts
   - Desktops > General Purpose Desktop
   - Desktops > Graphical Administration Tools
   - Desktops > Input Methods
   - Desktops > X Window System
   - Development > Additional Development
   - Development > Development Tools
   - Applications > Internet Browser

   ```bash
   # Default packages from Oracle Linux 6
   yum -y install elfutils-libelf
   yum -y install elfutils-libelf-devel
   yum -y install glibc
   yum -y install glibc-common
   yum -y install glibc-headers
   yum -y install pdksh
   yum -y install redhat-lsb
   yum -y install compat-libcap1
   yum -y install nfs-utils
   yum -y install xorg-x11-server-Xorg
   yum -y install xorg-x11-xauth
   yum -y install xorg-x11-utils
   yum -y install xdg-utils
   yum -y install xorg-x11-apps
   yum -y install xorg-x11-fonts-*
   yum -y install gnome-desktop
   yum -y install gnome-session
   yum -y install gnome-terminal
   yum -y install nautilus
   yum -y install libXext
   yum -y install libXtst
   yum -y install libX11
   yum -y install libXmu
   yum -y install libXp
   yum -y install libXt
   yum -y install libXrender
   yum -y install libXrandr
   yum -y install libXinerama
   yum -y install openmotif
   yum -y install openmotif22
   yum -y install xterm
   yum -y install wget
   yum -y install curl
   yum -y install firefox
   yum -y install gnome-system-monitor
   yum -y install gnome-panel
   yum -y install vte
   yum -y install gnome-applets
   yum -y install gnome-themes-standard
   ```

   Install the following required packages if not already present:

   ```bash
   # Default packages used for the Oracle Installation
   yum install binutils -y
   yum install compat-libstdc++-33 -y
   yum install compat-libstdc++-33.i686 -y
   yum install gcc -y
   yum install gcc-c++ -y
   yum install glibc -y
   yum install glibc.i686 -y
   yum install glibc-devel -y
   yum install glibc-devel.i686 -y
   yum install ksh -y
   yum install libgcc -y
   yum install libgcc.i686 -y
   yum install libstdc++ -y
   yum install libstdc++.i686 -y
   yum install libstdc++-devel -y
   yum install libstdc++-devel.i686 -y
   yum install libaio -y
   yum install libaio.i686 -y
   yum install libaio-devel -y
   yum install libaio-devel.i686 -y
   yum install libXext -y
   yum install libXext.i686 -y
   yum install libXtst -y
   yum install libXtst.i686 -y
   yum install libX11 -y
   yum install libX11.i686 -y
   yum install libXau -y
   yum install libXau.i686 -y
   yum install libxcb -y
   yum install libxcb.i686 -y
   yum install libXi -y
   yum install libXi.i686 -y
   yum install make -y
   yum install sysstat -y
   yum install unixODBC -y
   yum install unixODBC-devel -y
   yum install zlib-devel -y
   yum install elfutils-libelf-devel -y
   ```

   > This will install all the necessary 32-bit packages for 11.2.0.1. From 11.2.0.2 onwards many of these are unnecessary, but having them present does not cause a problem.

   Create the required OS groups and Oracle user:

   ```bash
   groupadd -g 501 oinstall
   groupadd -g 502 dba
   groupadd -g 503 oper
   groupadd -g 504 asmadmin
   groupadd -g 506 asmdba
   groupadd -g 505 asmoper

   useradd -u 502 -g oinstall -G dba,asmdba,oper oracle
   passwd oracle
   ```

   > We are not going to use the "asm" groups, since this installation will not use ASM.

6. Additional Setup

   Set the password for the `oracle` OS user:

   ```bash
   [root@oradb11g ~]# passwd oracle
   Changing password for user oracle.
   New password:          # Enter your secure password
   Retype new password:   # Re-enter your secure password
   passwd: all authentication tokens updated successfully.
   ```

   Update the `/etc/security/limits.d/90-nproc.conf` file as follows. See [MOS Note ID 1487773.1](https://support.oracle.com/epmos/faces/DocContentDisplay?id=1487773.1)

   ```bash
   [root@oradb11g ~]# cat /etc/security/limits.d/90-nproc.conf
   # Default limit for number of user's processes to prevent
   # accidental fork bombs.
   # See rhbz #432903 for reasoning.

   *          soft    nproc     1024
   root       soft    nproc     unlimited
   ```

7. Verify Hard Disk Partitions

   ```bash
   [root@oradb11g ~]# df -h
   Filesystem      Size  Used Avail Use% Mounted on
   /dev/sda2        50G  5.7G   41G  13% /
   tmpfs           2.0G   80K  2.0G   1% /dev/shm
   /dev/sda1       380M  147M  209M  42% /boot
   /dev/sda3        20G   33M   20G   1% /home
   /dev/sda5        20G   33M   20G   1% /tmp
   /dev/sda7       202G   33M  202G   1% /u01
   [root@oradb11g ~]# fdisk -l

   Disk /dev/ram0: 16 MB, 16777216 bytes
   255 heads, 63 sectors/track, 2 cylinders
   Units = cylinders of 16065 * 512 = 8225280 bytes
   Sector size (logical/physical): 512 bytes / 4096 bytes
   I/O size (minimum/optimal): 4096 bytes / 4096 bytes
   Disk identifier: 0x00000000

   ...

   Disk /dev/sda: 322.1 GB, 322122547200 bytes
   255 heads, 63 sectors/track, 39162 cylinders
   Units = cylinders of 16065 * 512 = 8225280 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 512 bytes / 512 bytes
   Disk identifier: 0x<DISK_ID>

      Device Boot      Start         End      Blocks   Id  System
   /dev/sda1   *           1          52      409600   83  Linux
   Partition 1 does not end on cylinder boundary.
   /dev/sda2              52        6579    52428800   83  Linux
   /dev/sda3            6579        9190    20971520   83  Linux
   /dev/sda4            9190       39163   240761856    5  Extended
   /dev/sda5            9190       11800    20971520   83  Linux
   /dev/sda6           11801       12845     8388608   82  Linux swap / Solaris
   /dev/sda7           12845       39163   211398656   83  Linux

   Disk /dev/sdb: 536.9 GB, 536870912000 bytes
   255 heads, 63 sectors/track, 65270 cylinders
   Units = cylinders of 16065 * 512 = 8225280 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 512 bytes / 512 bytes
   Disk identifier: 0x<DISK_ID>

      Device Boot      Start         End      Blocks   Id  System

   Disk /dev/sdc: 536.9 GB, 536870912000 bytes
   255 heads, 63 sectors/track, 65270 cylinders
   Units = cylinders of 16065 * 512 = 8225280 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 512 bytes / 512 bytes
   Disk identifier: 0x<DISK_ID>

      Device Boot      Start         End      Blocks   Id  System

   Disk /dev/sdd: 536.9 GB, 536870912000 bytes
   255 heads, 63 sectors/track, 65270 cylinders
   Units = cylinders of 16065 * 512 = 8225280 bytes
   Sector size (logical/physical): 512 bytes / 512 bytes
   I/O size (minimum/optimal): 512 bytes / 512 bytes
   Disk identifier: 0x<DISK_ID>

      Device Boot      Start         End      Blocks   Id  System
   ```

9. List All Disks

   ```bash
   [root@oradb11g ~]# lsblk
   NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
   sda      8:0    0   300G  0 disk
   ├─sda1   8:1    0   400M  0 part /boot
   ├─sda2   8:2    0    50G  0 part /
   ├─sda3   8:3    0    20G  0 part /home
   ├─sda4   8:4    0     1K  0 part
   ├─sda5   8:5    0    20G  0 part /tmp
   ├─sda6   8:6    0     8G  0 part [SWAP]
   └─sda7   8:7    0 201.6G  0 part /u01
   sdb      8:16   0   500G  0 disk
   sdc      8:32   0   500G  0 disk
   sdd      8:48   0   500G  0 disk
   sr0     11:0    1  1024M  0 rom
   ```

9. Format the Additional Disks with XFS

   ```bash
   [root@oradb11g ~]# mkfs.xfs /dev/sdb
   -bash: mkfs.xfs: command not found
   [root@oradb11g ~]# yum -y install mkfs
   [root@oradb11g ~]# mkfs.xfs -f /dev/sdb
   meta-data=/dev/sdb               isize=256    agcount=4, agsize=32768000 blks
            =                       sectsz=512   attr=2, projid32bit=1
            =                       crc=0        finobt=0
   data     =                       bsize=4096   blocks=131072000, imaxpct=25
            =                       sunit=0      swidth=0 blks
   naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
   log      =internal log           bsize=4096   blocks=64000, version=2
            =                       sectsz=512   sunit=0 blks, lazy-count=1
   realtime =none                   extsz=4096   blocks=0, rtextents=0
   [root@oradb11g ~]# mkfs.xfs -f /dev/sdc
   meta-data=/dev/sdc               isize=256    agcount=4, agsize=32768000 blks
            =                       sectsz=512   attr=2, projid32bit=1
            =                       crc=0        finobt=0
   data     =                       bsize=4096   blocks=131072000, imaxpct=25
            =                       sunit=0      swidth=0 blks
   naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
   log      =internal log           bsize=4096   blocks=64000, version=2
            =                       sectsz=512   sunit=0 blks, lazy-count=1
   realtime =none                   extsz=4096   blocks=0, rtextents=0
   [root@oradb11g ~]# mkfs.xfs -f /dev/sdd
   meta-data=/dev/sdd               isize=256    agcount=4, agsize=32768000 blks
            =                       sectsz=512   attr=2, projid32bit=1
            =                       crc=0        finobt=0
   data     =                       bsize=4096   blocks=131072000, imaxpct=25
            =                       sunit=0      swidth=0 blks
   naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
   log      =internal log           bsize=4096   blocks=64000, version=2
            =                       sectsz=512   sunit=0 blks, lazy-count=1
   realtime =none                   extsz=4096   blocks=0, rtextents=0
   ```

10. Mount the New Disks and Register Them in `/etc/fstab`

    ```bash
    [root@oradb11g ~]# mkdir /u02 /u03 /u04
    [root@oradb11g ~]# mount /dev/sdb /u02
    [root@oradb11g ~]# mount /dev/sdc /u03
    [root@oradb11g ~]# mount /dev/sdd /u04
    [root@oradb11g ~]# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda2        50G  5.9G   41G  13% /
    tmpfs           2.0G   80K  2.0G   1% /dev/shm
    /dev/sda1       380M  147M  209M  42% /boot
    /dev/sda3        20G   33M   20G   1% /home
    /dev/sda5        20G   33M   20G   1% /tmp
    /dev/sda7       202G   33M  202G   1% /u01
    /dev/sdb        500G   33M  500G   1% /u02
    /dev/sdc        500G   33M  500G   1% /u03
    /dev/sdd        500G   33M  500G   1% /u04
    [root@oradb11g ~]# blkid /dev/sdb
    /dev/sdb: UUID="<UUID_SDB>" TYPE="xfs"
    [root@oradb11g ~]# blkid /dev/sdc
    /dev/sdc: UUID="<UUID_SDC>" TYPE="xfs"
    [root@oradb11g ~]# blkid /dev/sdd
    /dev/sdd: UUID="<UUID_SDD>" TYPE="xfs"
    [root@oradb11g ~]# cat /etc/fstab

    #
    # /etc/fstab
    # Created by anaconda on Fri Oct 24 18:10:27 2025
    #
    # Accessible filesystems, by reference, are maintained under '/dev/disk'
    # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
    #
    UUID=<UUID_ROOT>  /                       ext4    defaults        1 1
    UUID=<UUID_BOOT>  /boot                   ext4    defaults        1 2
    UUID=<UUID_HOME>  /home                   xfs     defaults        1 2
    UUID=<UUID_TMP>   /tmp                    xfs     defaults        1 2
    UUID=<UUID_U01>   /u01                    xfs     defaults        1 2
    UUID=<UUID_SWAP>  swap                    swap    defaults        0 0
    tmpfs                   /dev/shm                tmpfs   defaults        0 0
    devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
    sysfs                   /sys                    sysfs   defaults        0 0
    proc                    /proc                   proc    defaults        0 0
    [root@oradb11g ~]# vi /etc/fstab
    [root@oradb11g ~]# cat /etc/fstab

    #
    # /etc/fstab
    # Created by anaconda on Fri Oct 24 18:10:27 2025
    #
    # Accessible filesystems, by reference, are maintained under '/dev/disk'
    # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
    #
    UUID=<UUID_ROOT>  /                       ext4    defaults        1 1
    UUID=<UUID_BOOT>  /boot                   ext4    defaults        1 2
    UUID=<UUID_HOME>  /home                   xfs     defaults        1 2
    UUID=<UUID_TMP>   /tmp                    xfs     defaults        1 2
    UUID=<UUID_U01>   /u01                    xfs     defaults        1 2
    UUID=<UUID_U02>   /u02                    xfs     defaults        1 2
    UUID=<UUID_U03>   /u03                    xfs     defaults        1 2
    UUID=<UUID_U04>   /u04                    xfs     defaults        1 2
    UUID=<UUID_SWAP>  swap                    swap    defaults        0 0
    tmpfs                   /dev/shm                tmpfs   defaults        0 0
    devpts                  /dev/pts                devpts  gid=5,mode=620  0 0
    sysfs                   /sys                    sysfs   defaults        0 0
    proc                    /proc                   proc    defaults        0 0
    [root@oradb11g ~]# reboot
    ```

11. After Rebooting, Verify That All Mount Points Persist

    ```bash
    [root@oradb11g ~]# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda2        50G  5.8G   41G  13% /
    tmpfs           2.0G   72K  2.0G   1% /dev/shm
    /dev/sda1       380M  147M  209M  42% /boot
    /dev/sda3        20G   33M   20G   1% /home
    /dev/sda5        20G   33M   20G   1% /tmp
    /dev/sda7       202G   33M  202G   1% /u01
    /dev/sdb        500G   33M  500G   1% /u02
    /dev/sdc        500G   33M  500G   1% /u03
    /dev/sdd        500G   33M  500G   1% /u04
    ```

9. Create the directories in which the Oracle software will be installed.

   ```bash
   [root@oradb11g ~]# mkdir -p /u01/app/oracle/product/11g/dbhome_1
   [root@oradb11g ~]# mkdir -p /u02/oradata/dbtest11g
   [root@oradb11g ~]# mkdir -p /u03/oradata/dbtest11g
   [root@oradb11g ~]# mkdir -p /u03/oraindx/dbtest11g
   [root@oradb11g ~]# mkdir -p /u04/orafra/dbtest11g
   [root@oradb11g ~]# mkdir /u04/dump
   [root@oradb11g ~]# mkdir -p /u04/backup/scripts
   [root@oradb11g ~]# mkdir /u04/backup/logs
   [root@oradb11g ~]# mkdir /u04/backup/daily
   [root@oradb11g ~]# chown -R oracle:oinstall /u01 /u02 /u03 /u04
   [root@oradb11g ~]# chmod -R 775  /u01 /u02 /u03 /u04
   ```

   > Putting mount points directly under root without mounting separate disks to them is typically a bad idea. It's done here for simplicity, but for a real installation "/" storage should be reserved for the OS.

10. Create the DBA Scripts Directory

    ```bash
    [root@oradb11g ~]# su - oracle
    [oracle@oradb11g ~]$ mkdir /home/oracle/scripts
    ```

11. Configure the Oracle user environment variables in `.bash_profile`

    ```bash
    [oracle@oradb11g ~]$ cat /home/oracle/.bash_profile
    # .bash_profile

    # Get the aliases and functions
    if [ -f ~/.bashrc ]; then
            . ~/.bashrc
    fi

    # User specific environment and startup programs

    PATH=$PATH:$HOME/bin

    export PATH
    [oracle@oradb11g ~]$ vi /home/oracle/.bash_profile
    [oracle@oradb11g ~]$ cat /home/oracle/.bash_profile
    # .bash_profile

    # Get the aliases and functions
    if [ -f ~/.bashrc ]; then
            . ~/.bashrc
    fi

    # User specific environment and startup programs

    PATH=$PATH:$HOME/bin

    export PATH

    # Oracle Settings
    TMP=/tmp; export TMP
    TMPDIR=$TMP; export TMPDIR

    ORACLE_HOSTNAME=oradb11g.example.com; export ORACLE_HOSTNAME
    ORACLE_UNQNAME=DBTEST11G; export ORACLE_UNQNAME
    ORACLE_BASE=/u01/app/oracle; export ORACLE_BASE
    ORACLE_HOME=$ORACLE_BASE/product/11g/dbhome_1; export ORACLE_HOME
    ORACLE_SID=DBTEST11G; export ORACLE_SID

    PATH=/usr/sbin:$PATH; export PATH
    PATH=$ORACLE_HOME/bin:$PATH; export PATH

    LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib; export LD_LIBRARY_PATH
    CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH
    ```

---

## **B. Oracle 11g Installation On Oracle Linux 6**

1. Extract the Oracle Database 11g 64-bit installer

   ```bash
   [oracle@oradb11g ~]$ cd Downloads/
   [oracle@oradb11g Downloads]$ cd Oracle11gLinux64bit/
   [oracle@oradb11g Oracle11gLinux64bit]$ unzip V17530-01_1of2.zip
   [oracle@oradb11g Oracle11gLinux64bit]$ unzip V17530-01_2of2.zip
   [oracle@oradb11g Oracle11gLinux64bit]$ ls
   database  V17530-01_1of2.zip  V17530-01_2of2.zip
   ```

2. Launch the Oracle Database 11g installer

   ```bash
   [oracle@oradb11g database]$ cd /home/oracle/Downloads/Oracle11gLinux64bit/database
   [oracle@oradb11g database]$ ./runInstaller
   ```

3. Uncheck "I wish to receive security updates via My Oracle Support", then press Next button to continue.

   ![Step 3 - Security Updates](image.png)

4. To continue installation, press Yes button.

   ![Step 4 - Confirm](image-1.png)

5. Select "Install database software only" then press Next button to continue.

   ![Step 5 - Install Option](image-2.png)

6. Select "Single instance database installation"

   ![Step 6 - Installation Type](image-3.png)

7. Press Next button to continue.

   ![Step 7 - Language Selection](image-4.png)

8. Select the Oracle Database edition. In this example, "Standard Edition (4.22 GB)" is chosen. Press Next to continue.

   ![Step 8 - Database Edition](image-5.png)

9. Press Next button to continue

   ![Step 9 - Installation Location](image-6.png)

10. Press Next button to continue

    ![Step 10 - OS Groups](image-7.png)

11. Press Next button to continue

    ![Step 11 - Summary](image-8.png)

12. Prerequisite Checks in progress

    ![Step 12 - Prerequisite Checks](image-9.png)

13. Some package warnings can be safely ignored — the Oracle Universal Installer (OUI) may not recognize newer package versions. Check "Ignore All" and press Next to continue.

    ![Step 13 - Ignore Warnings](image-10.png)

14. Review the installation summary and press Finish to begin the software installation.

    ![Step 14 - Summary](image-11.png)

15. The Oracle Database 11g software-only installation is now in progress

    ![Step 15 - Installation Progress](image-12.png)

16. A non-critical error may appear during the linking phase. Press Continue to proceed.

    ![Step 16 - Error Prompt](image-13.png)

17. When prompted, run the configuration scripts as the root OS user

    ![Step 17 - Execute Scripts](image-14.png)

    Execute the following configuration scripts as the root OS user:

    ```bash
    [root@oradb11g ~]# /u01/app/oraInventory/orainstRoot.sh
    Changing permissions of /u01/app/oraInventory.
    Adding read,write permissions for group.
    Removing read,write,execute permissions for world.

    Changing groupname of /u01/app/oraInventory to oinstall.
    The execution of the script is complete.
    [root@oradb11g ~]# /u01/app/oracle/product/11g/dbhome_1/root.sh
    Running Oracle 11g root.sh script...

    The following environment variables are set as:
        ORACLE_OWNER= oracle
        ORACLE_HOME=  /u01/app/oracle/product/11g/dbhome_1

    Enter the full pathname of the local bin directory: [/usr/local/bin]: 
       Copying dbhome to /usr/local/bin ...
       Copying oraenv to /usr/local/bin ...
       Copying coraenv to /usr/local/bin ...


    Creating /etc/oratab file...
    Entries will be added to the /etc/oratab file as needed by
    Database Configuration Assistant when a database is created
    Finished running generic part of root.sh script.
    Now product-specific root actions will be performed.
    Finished product-specific root actions.
    ```

18. The Oracle Database 11g software installation is complete. Press Close to exit the installer.

    ![Step 18 - Installation Complete](image-15.png)

19. Configure a new listener on the default port 1521 using Oracle Net Configuration Assistant (netca)

    ```bash
    [oracle@oradb11g database]$ netca
    ```

35. Choose "Listener configuration" and press Next to continue.

    ![Step 35 - NETCA Welcome](image-16.png)

36. Select "Add" to create a new listener

    ![Step 36 - Listener Configuration](image-17.png)

37. Enter the listener name

    ![Step 37 - Listener Name](image-18.png)

38. Press Next button to continue.

    ![Step 38 - Protocols](image-19.png)

39. Use default port 1521

    ![Step 39 - Port Number](image-20.png)

40. Select No

    ![Step 40 - More Listeners](image-21.png)

41. Press Next and Finish button to finish the configuration

    ![Step 41a - Done](image-22.png)

    ![Step 41b - Finish](image-23.png)

42. Create a new Oracle database using the Database Configuration Assistant (dbca)

    ```bash
    [oracle@oradb11g database]$ dbca
    ```

43. Press Next

    ![Step 43 - DBCA Welcome](image-24.png)

44. Select "Create a Database" then press Next button to continue.

    ![Step 44 - Operations](image-25.png)

45. Select "General Purpose or Transaction Processing" then press Next to continue.

    ![Step 45 - Database Templates](image-26.png)

46. Enter the Global Database Name (with domain) and the SID

    ![Step 46 - Database Identification](image-27.png)

47. Uncheck "Configure Enterprise Manager"

    ![Step 47 - Management Options](image-28.png)

48. Set a password for all administrative accounts. Use a strong password and press Next to continue.

    ![Step 48 - Database Credentials](image-29.png)

49. Press Next button

    ![Step 49 - Storage Options](image-30.png)

50. Enable "Specify Flash Recovery Area" and set the path and size. Press Next to continue.

    ![Step 50 - Recovery Configuration](image-31.png)

51. Press Next button.

    ![Step 51 - Database Content](image-32.png)

52. Choose "Typical" memory configuration and allocate approximately 60% of total server memory to Oracle. Uncheck "Use Automatic Memory Management".

    ![Step 52 - Initialization Parameters](image-33.png)

53. Adjust the initialization parameters for processes, open_cursors, and sessions

    ![Step 53a - Parameters](image-34.png)

    ![Step 53b - Parameters](image-35.png)

    Set `open_cursors` to 1000, `processes` to 900, and `sessions` to 995. Press Next to continue.

54. Place the control files on separate disks for redundancy. In this example, `control01.ctl` resides under `$ORACLE_BASE` and `control02.ctl` under `/u04`

    ![Step 54 - Control Files](image-36.png)

55. Set Maximum Datafiles to 900, and press Next button to continue.

    ![Step 55 - Database Storage](image-37.png)

56. Set the data file directory to `/u02/oradata/{DB_UNIQUE_NAME}/`

    ![Step 56 - Data Files Location](image-38.png)

57. Resize the redo log files to 2 GB and distribute them across two disks (`/u02` and `/u03`) for redundancy

    ![Step 57a - Redo Log Groups](image-39.png)

    ![Step 57b - Redo Log Members](image-40.png)

    ![Step 57c - Redo Log Size](image-41.png)

    ![Step 57d - Redo Log Config](image-42.png)

    ![Step 57e - Redo Log Complete](image-43.png)

58. Select "Create Database" and press Finish to begin the database creation.

    ![Step 58 - Creation Options](image-44.png)

59. Press OK to continue.

    ![Step 59 - Confirmation](image-45.png)

60. Oracle Database 11g creation is now in progress

    ![Step 60 - Database Creation Progress](image-46.png)

61. Database creation is complete. Press Exit to close the Database Configuration Assistant

    ![Step 61 - Database Created](image-47.png)

62. Verify the listener status

    ```bash
    [oracle@oradb11g ~]$ lsnrctl status

    LSNRCTL for Linux: Version 11.2.0.1.0 - Production on 23-OCT-2025 15:46:22

    Copyright (c) 1991, 2009, Oracle.  All rights reserved.

    Connecting to (DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=oradb11g.example.com)(PORT=1521)))
    STATUS of the LISTENER
    ------------------------
    Alias                     LISTENER
    Version                   TNSLSNR for Linux: Version 11.2.0.1.0 - Production
    Start Date                23-OCT-2025 15:09:18
    Uptime                    0 days 0 hr. 37 min. 4 sec
    Trace Level               off
    Security                  ON: Local OS Authentication
    SNMP                      OFF
    Listener Parameter File   /u01/app/oracle/product/11g/dbhome_1/network/admin/listener.ora
    Listener Log File         /u01/app/oracle/diag/tnslsnr/oradb11g/listener/alert/log.xml
    Listening Endpoints Summary...
      (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=oradb11g.example.com)(PORT=1521)))
    Services Summary...
    Service "DBTEST11G.example.com" has 1 instance(s).
      Instance "DBTEST11G", status READY, has 1 handler(s) for this service...
    Service "DBTEST11GXDB.example.com" has 1 instance(s).
      Instance "DBTEST11G", status READY, has 1 handler(s) for this service...
    The command completed successfully
    ```

63. Test the database connection using SQL*Plus as SYSDBA

    ```bash
    [oracle@oradb11g ~]$ export ORACLE_SID=DBTEST11G
    [oracle@oradb11g ~]$ sqlplus / as sysdba

    SQL*Plus: Release 11.2.0.1.0 Production on Thu Oct 23 15:47:17 2025

    Copyright (c) 1982, 2009, Oracle.  All rights reserved.


    Connected to:
    Oracle Database 11g Release 11.2.0.1.0 - 64bit Production
    SQL> select * from dual;

    D
    -
    X
    ```

---

## **C. Post Installation Oracle Database 11g on Oracle Linux 6**

On Oracle Linux 6 (which uses **SysV init**, not systemd), you can configure Oracle Database 11g to start automatically at boot by:

1. Enabling the database in `/etc/oratab`
2. Creating an init script `/etc/init.d/oracle`
3. Registering it with `chkconfig`
4. Testing with `service oracle start|stop|status`
5. Rebooting to verify automatic startup

---

1. Edit `/etc/oratab`

   Open the file `/etc/oratab` and ensure your entry looks like this:

   ```
   ORCL:/u01/app/oracle/product/11.2.0/dbhome_1:Y
   ```

   The **last field must be `Y`** to allow automatic startup using `dbstart`.

2. Create the init script `/etc/init.d/oracle`

   Create a new file `/etc/init.d/oracle` and paste the following script.

   (Adjust `ORACLE_OWNER` and `ORACLE_HOME` as needed.)

   ```bash
   #!/bin/sh
   #
   # oracle  Startup script for Oracle DB (uses dbstart/dbshut)
   #
   # chkconfig: 345 99 10
   # description: Start and stop Oracle Database using /etc/oratab and dbstart/dbshut
   #
   ### BEGIN INIT INFO
   # Provides: oracle
   # Required-Start: $local_fs $network $syslog
   # Required-Stop:  $local_fs $network $syslog
   # Default-Start:  3 4 5
   # Default-Stop:   0 1 2 6
   # Short-Description: Start Oracle DB
   ### END INIT INFO

   ORACLE_OWNER=oracle
   # Optionally hardcode ORACLE_HOME
   # ORACLE_HOME=/u01/app/oracle/product/11g/dbhome_1

   export PATH=/usr/sbin:/usr/bin:/sbin:/bin

   DBSTART_CMD="dbstart"
   DBSHUT_CMD="dbshut"
   LSNRCTL=/u01/app/oracle/product/11g/dbhome_1/bin/lsnrctl

   start() {
     echo "Starting Oracle databases (using dbstart)..."
     su - ${ORACLE_OWNER} -c "export ORACLE_HOME=$(awk -F: '/^[^#]/ { if ($3=="Y" && $1 !~ /^\+/) { print $2; exit } }' /etc/oratab) ; ${DBSTART_CMD} $ORACLE_HOME"
     # Uncomment if you want to explicitly start listener:
     # su - ${ORACLE_OWNER} -c "${LSNRCTL} start"
     echo "Startup complete."
   }

   stop() {
     echo "Stopping Oracle databases (using dbshut)..."
     su - ${ORACLE_OWNER} -c "export ORACLE_HOME=$(awk -F: '/^[^#]/ { if ($3=="Y" && $1 !~ /^\+/) { print $2; exit } }' /etc/oratab) ; ${DBSHUT_CMD} $ORACLE_HOME"
     # Uncomment to stop listener explicitly:
     # su - ${ORACLE_OWNER} -c "${LSNRCTL} stop"
     echo "Shutdown complete."
   }

   status() {
     echo "Checking Oracle processes..."
     ps -ef | grep pmon | grep -v grep
   }

   case "$1" in
     start)
       start
       ;;
     stop)
       stop
       ;;
     restart)
       stop
       sleep 2
       start
       ;;
     status)
       status
       ;;
     *)
       echo "Usage: $0 {start|stop|restart|status}"
       exit 1
   esac

   exit 0
   ```

3. Set permissions and enable the service

   Run these commands as `root`:

   ```bash
   chmod 755 /etc/init.d/oracle
   chkconfig --add oracle
   chkconfig oracle on
   chkconfig --list oracle
   ```

   Test the service:

   ```bash
   service oracle start
   ps -ef | grep pmon | grep -v grep
   su - oracle -c "lsnrctl status"
   ```

4. Test automatic startup

   Reboot your server:

   ```bash
   reboot
   ```

   After the system boots, check if the database started automatically:

   ```bash
   ps -ef | grep pmon | grep -v grep
   su - oracle -c "lsnrctl status"
   ```

5. Optional – Handle multiple ORACLE_HOME entries

   If you have more than one database entry with `Y` in `/etc/oratab`, modify the script to loop through them:

   ```bash
   for OH in $(awk -F: '/^[^#]/ && $3=="Y" {print $2}' /etc/oratab); do
     su - oracle -c "ORACLE_HOME=$OH; PATH=\$ORACLE_HOME/bin:\$PATH; dbstart \$ORACLE_HOME"
   done
   ```

6. Quick Alternative (Crontab Method)

   Add this line to the Oracle user's crontab (`crontab -e`):

   ```bash
   @reboot /u01/app/oracle/product/11g/dbhome_1/bin/dbstart /u01/app/oracle/product/11g/dbhome_1
   ```

   > **Note:** This is not recommended for production because it doesn't integrate cleanly with system startup and shutdown order.

7. Troubleshooting Tips

   | Problem | Solution |
   |---------|----------|
   | `dbstart` says "ORACLE_HOME not set" | Ensure `/etc/oratab` entry is not commented and ends with `:Y` |
   | Database not starting | Check `/etc/oratab`, `.bash_profile`, and `$ORACLE_HOME/bin/dbstart` permissions |
   | Listener not starting | Uncomment the `lsnrctl start` line in the script |
   | SELinux blocks startup | Temporarily set SELinux to permissive (`setenforce 0`) to test, then adjust policy properly |
   | Logs | Check `/var/log/messages` and `/home/oracle/.bash_profile` for environment setup issues |
