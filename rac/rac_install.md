**Step-by-step Oracle RAC installation guide** on **RHEL 9** with **Oracle 19c** across **3 nodes**:

* **Nodes:**

  * oranode1 → `10.10.10.1`
  * oranode2 → `10.10.10.2`
  * oranode3 → `10.10.10.3`

Below is a **detailed RAC installation procedure** (enterprise-grade).

---

# 🔹 Oracle RAC 19c Installation on RHEL 9 (3-Node Cluster)

---

## **1. Prerequisites**

### **1.1 OS Requirements**

* OS: RHEL 9 (with UEK kernel recommended).
* Packages (install on all nodes):

  ```bash
  yum install -y bc binutils compat-libcap1 compat-libstdc++ gcc gcc-c++ glibc \
  glibc-devel ksh libaio libaio-devel libX11 libXau libXi libXtst libXrender \
  libXrender-devel libgcc libstdc++ libstdc++-devel libxcb make smartmontools sysstat \
  net-tools nfs-utils unzip
  ```

### **1.2 Create Required Groups and Users**

On all nodes:

```bash
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
groupadd -g 54324 asmadmin
groupadd -g 54325 asmdba
groupadd -g 54326 asmoper

useradd -u 54321 -g oinstall -G dba,oper,asmadmin,asmdba,asmoper oracle
useradd -u 54322 -g oinstall -G asmadmin,asmdba,asmoper grid
```

Set passwords:

```bash
passwd oracle
passwd grid
```

### **1.3 Kernel Parameters**

Add to `/etc/sysctl.conf`:

```conf
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 4294967296
kernel.shmmax = 8589934592
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
```

Apply:

```bash
sysctl -p
```

### **1.4 Security Config**

Disable firewall and SELinux:

```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
```

Edit `/etc/selinux/config` → `SELINUX=disabled`.

---

## **2. Network Configuration**

### **2.1 Hostname Setup**

```bash
hostnamectl set-hostname oranode1
hostnamectl set-hostname oranode2
hostnamectl set-hostname oranode3
```

### **2.2 /etc/hosts File**

Add on all nodes:

```ini
# Public IPs
10.10.10.1 oranode1
10.10.10.2 oranode2
10.10.10.3 oranode3

# Private Interconnect (example subnet)
192.168.17.1 oranode1-priv
192.168.17.2 oranode2-priv
192.168.17.3 oranode3-priv

# Virtual IPs
10.10.10.11 oranode1-vip
10.10.10.12 oranode2-vip
10.10.10.13 oranode3-vip

# SCAN IPs (3 recommended)
10.10.10.20 rac-scan
10.10.10.21 rac-scan
10.10.10.22 rac-scan
```

### **2.3 SSH Equivalence**

Generate and distribute SSH keys for `grid` and `oracle`:

```bash
ssh-keygen -t rsa
ssh-copy-id oranode1
ssh-copy-id oranode2
ssh-copy-id oranode3
```

Test:

```bash
ssh oranode2 date
```

---

## **3. Shared Storage Setup**

### **3.1 ASM Disks**

* Present shared disks from SAN/iSCSI to all nodes.
* Example devices: `/dev/asm-disk1`, `/dev/asm-disk2`.
* Mark as ASM disks:

  ```bash
  oracleasm init
  oracleasm createdisk DATA1 /dev/sdb
  oracleasm createdisk FRA1 /dev/sdc
  ```
* Verify:

  ```bash
  oracleasm listdisks
  ```

---

## **4. Install Grid Infrastructure (GI)**

### **4.1 Prepare Directories**

On all nodes:

```bash
mkdir -p /u01/app/19.0.0/grid
mkdir -p /u01/app/grid
mkdir -p /u01/app/oracle/product/19.0.0/dbhome_1
chown -R grid:oinstall /u01/app/19.0.0/grid /u01/app/grid
chown -R oracle:oinstall /u01/app/oracle
chmod -R 775 /u01
```

### **4.2 Run Grid Setup**

As `grid` user on `oranode1`:

```bash
./gridSetup.sh
```

* Choose **Cluster Installation**.
* Add all nodes (oranode1, oranode2, oranode3).
* Configure **SCAN** (`rac-scan`).
* Select ASM storage for OCR & Voting Disk.
* Run root scripts after installer prompts:

  ```bash
  /u01/app/oraInventory/orainstRoot.sh
  /u01/app/19.0.0/grid/root.sh
  ```

### **4.3 Verify Cluster**

```bash
crsctl check cluster -all
crsctl stat res -t
```

---

## **5. Install Oracle Database Software**

### **5.1 Run Installer**

As `oracle` user:

```bash
./runInstaller
```

* Select **Install Database Software Only**.
* Choose RAC nodes (oranode1, oranode2, oranode3).

---

## **6. Create RAC Database**

### **6.1 DBCA**

Run:

```bash
dbca
```

* Choose **Create Database → RAC Database**.
* Add all 3 nodes.
* Define DB name (`oradb`).
* Create required services.

---

## **7. Post Installation Checks**

### **7.1 Cluster Status**

```bash
crsctl stat res -t
```

### **7.2 Database Status**

```bash
srvctl status database -d oradb
```

### **7.3 SCAN Test**

```bash
nslookup rac-scan
sqlplus sys@//rac-scan:1521/oradb as sysdba
```

---

# ✅ Final Setup Summary

* **3 Nodes:** oranode1, oranode2, oranode3.
* **Networks:** Public, Private, Virtual IPs, SCAN.
* **Shared Storage:** ASM disks for DATA, FRA, OCR, Voting.
* **Grid Infrastructure Installed:** Manages cluster.
* **Database Created:** 19c RAC DB running across 3 nodes.

---
