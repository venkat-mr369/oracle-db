# Updated Oracle 21c Installation Script for Oracle Linux 9 (Without Preinstall RPMs)

On Oracle Linux 9, the `oracle-database-preinstall-21c` and `compat-openssl10` packages are not available in the default repositories. You must therefore manually install required packages and set kernel/user limits. Below is an updated script and instructions, suitable for running as root, that replaces the automatic preinstall step with their manual equivalents.

## 1. Install Prerequisite Packages Manually

Install all required development tools and library dependencies:

```bash
sudo dnf install -y \
    bc \
    binutils \
    elfutils-libelf \
    elfutils-libelf-devel \
    fontconfig-devel \
    glibc \
    glibc-devel \
    ksh \
    libaio \
    libaio-devel \
    libnsl \
    libnsl2 \
    libstdc++ \
    libstdc++-devel \
    libX11 \
    libXau \
    libXi \
    libXtst \
    libgcc \
    libxcb \
    make \
    smartmontools \
    sysstat \
    libxcrypt-compat
```

> **Note:** If you encounter missing packages (such as `compat-openssl10`), download the .rpm for compatible OL8/CentOS8/RHEL8 from reputable mirrors and install locally with `dnf localinstall `, but be aware this is not officially recommended for production.[1][2]

## 2. Configure Kernel Parameters and Security Limits

Create `/etc/sysctl.d/98-oracle.conf`:

```bash
cat  /etc/sysctl.d/98-oracle.conf
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586
net.ipv4.ip_local_port_range = 9000 65500
EOF

sysctl --system
```

Adjust `/etc/security/limits.d/oracle.conf`:

```bash
cat  /etc/security/limits.d/oracle.conf
oracle   soft   nproc    2047
oracle   hard   nproc    16384
oracle   soft   nofile   1024
oracle   hard   nofile   65536
oracle   soft   stack    10240
oracle   hard   stack    32768
EOF
```

## 3. Create Required Groups and the Oracle User

```bash
getent group oinstall >/dev/null || groupadd -g 54321 oinstall
getent group dba >/dev/null || groupadd -g 54322 dba
getent group oper >/dev/null || groupadd -g 54323 oper
id oracle &>/dev/null || useradd -u 54321 -g oinstall -G dba,oper oracle
echo "Set password for oracle:"
passwd oracle
```

## 4. Create Oracle Directories and Set Permissions

```bash
mkdir -p /u01/app/oracle/product/21.0/db_home
mkdir -p /u01/app/oraInventory
chown -R oracle:oinstall /u01
chmod -R 775 /u01
```

## 5. Set Hostname and Update `/etc/hosts`

```bash
hostnamectl set-hostname myserver.example.com
grep -q "myserver.example.com" /etc/hosts || echo "127.0.0.1 myserver.example.com localhost" >> /etc/hosts
```

## 6. Configure Oracle User Environment

```bash
su - oracle -c "cat > ~/.bash_profile <<EOF
export ORACLE_SID=CDB
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/21.0/db_home
export PATH=\\\$PATH:\\\$HOME/.local/bin:\\\$ORACLE_HOME/bin
EOF"
```

## 7. Unpack and Install Oracle Software

Make sure the Oracle ZIP is present in `/tmp`.

```bash
if [ ! -d "/u01/app/oracle/product/21.0/db_home/install" ]; then
  su - oracle -c "unzip -oq /tmp/LINUX.X64_213000_db_home.zip -d /u01/app/oracle/product/21.0/db_home"
fi

su - oracle -c "
cd /u01/app/oracle/product/21.0/db_home
./runInstaller -ignorePrereq -waitforcompletion -silent \
-responseFile /u01/app/oracle/product/21.0/db_home/install/response/db_install.rsp \
oracle.install.option=INSTALL_DB_SWONLY \
ORACLE_HOSTNAME=myserver.example.com \
UNIX_GROUP_NAME=oinstall \
INVENTORY_LOCATION=/u01/app/oraInventory \
SELECTED_LANGUAGES=en,en_GB \
ORACLE_HOME=/u01/app/oracle/product/21.0/db_home \
ORACLE_BASE=/u01/app/oracle \
oracle.install.db.InstallEdition=EE \
oracle.install.db.OSDBA_GROUP=dba \
oracle.install.db.OSBACKUPDBA_GROUP=dba \
oracle.install.db.OSDGDBA_GROUP=dba \
oracle.install.db.OSKMDBA_GROUP=dba \
oracle.install.db.OSRACDBA_GROUP=dba \
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
DECLINE_SECURITY_UPDATES=true
"
```

## 8. Run Root Scripts

```bash
/u01/app/oraInventory/orainstRoot.sh
/u01/app/oracle/product/21.0/db_home/root.sh
```

## 9. (Optional) Create Database with DBCA

```bash
su - oracle -c "
dbca -silent -createDatabase -templateName General_Purpose.dbc \
-gdbname cdb -sid cdb -responseFile NO_VALUE \
-characterSet AL32UTF8 -totalMemory 2048 \
-createAsContainerDatabase true -numberOfPDBs 1 -pdbName pdb1 \
-createListener LISTENER_ORCL:1521 -emConfiguration DBEXPRESS -emExpressPort 5500 \
-J-Doracle.assistants.dbca.validate.ConfigurationParams=false
"
```

## 10. Verify Installation

```bash
su - oracle -c "ps -ef | grep pmon"
su - oracle -c "sqlplus / as sysdba <<EOF
SELECT name, open_mode FROM v\$database;
EXIT
EOF"
```

## Notes and References

- You must manually install all dependencies and configure system parameters for Oracle on OL9, as the easy preinstall RPM is not available.[3][4][1]
- The missing `compat-openssl10` can be sourced from an OL8/CentOS 8 rpm for testing, but this is *not* officially supported for production. Always verify the library is in place if any builds require legacy SSL.[1][2]
- Full manual dependency lists are provided in the official Oracle docs and guides, and are required for a successful installation.[3][5]
- If new dependencies are needed, install them with `dnf` as root before running the runInstaller.

This approach enables installation on Oracle Linux 9 despite missing automated packages, by manually fulfilling requirements and maintaining best practices.

[1] https://forums.oracle.com/ords/apexds/post/oel-9-image-does-not-have-compat-openssl10-package-in-repos-4811
[2] https://ramoradba.com/2023/03/29/oracle-database-21c-express-edition-xe-rpm-installation-on-oracle-linux-9-2/
[3] https://docs.oracle.com/en/database/oracle/oracle-database/21/lacli/supported-oracle-linux-9-distributions-for-x86-64.html
[4] https://unix.stackexchange.com/questions/732297/installing-oracle-database-preinstall-21c-oracle-linux
[5] https://docs.oracle.com/en/database/oracle/oracle-database/21/xeinl/installing-oracle-database-free.html
[6] https://docs.oracle.com/en/database/oracle/oracle-database/21/ladbi/running-rpm-packages-to-install-oracle-database.html
[7] https://www.reddit.com/r/oracle/comments/w9b5gf/error_installing_oracle_preinstall/
[8] https://oracle-base.com/articles/19c/oracle-db-19c-installation-on-oracle-linux-9
[9] https://docs.oracle.com/en/database/oracle/oracle-database/21/rnrdm/linux-platform-issues.html
[10] https://stackoverflow.com/questions/57966184/libssl-so-10-cannot-open-shared-object-file-no-such-file-or-directory
[11] https://support.oracle.com/knowledge/Oracle%20Database%20Products/3063194_1.html
[12] https://packagecloud.io/trabian/extras/packages/el/8/compat-openssl10-devel-1.0.2o-3.el8.x86_64.rpm?distro_version_id=205
[13] https://support.dbagenesis.com/oracle-database/oracle-21c-installation-on-linux
[14] https://oracle-base.com/articles/21c/oracle-db-21c-installation-on-oracle-linux-8
[15] https://extras.getpagespeed.com/redhat/8/x86_64/repoview/compat-openssl10-devel.html
[16] https://docs.oracle.com/en/database/oracle/oracle-database/21/lacli/supported-red-hat-enterprise-linux-9-distributions-for-x86-64.html
[17] https://mikedietrichde.com/2021/08/16/installation-of-oracle-database-21c-on-oracle-linux/
[18] https://rpmfind.net/linux/rpm2html/search.php?query=compat-openssl10
[19] https://docs.oracle.com/en/database/oracle/oracle-database/21/ladbi/index.html
[20] https://oracledbwr.com/step-by-step-oracle-21c-installation-on-linux/
