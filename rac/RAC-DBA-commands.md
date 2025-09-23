**daily checklist of Oracle RAC DBA commands** 

---

# 🔹 Oracle RAC Daily Use Commands (19c on RHEL)

---

## **1. Cluster Health & Resource Status**

```bash
# Check clusterware status on all nodes
crsctl check cluster -all

# Check Oracle High Availability Service
crsctl check has

# Cluster resources (databases, listeners, ASM, etc.)
crsctl stat res -t

# Cluster resources in detail
crsctl stat res -init

# Verify node membership
olsnodes -n -s -t
```

✅ **Use Case:** Run after login to check cluster status and ensure all nodes are UP.

---

## **2. Database and Instance Management**

```bash
# Check RAC database status
srvctl status database -d oradb

# Check instance status
srvctl status instance -d oradb -i oradb1

# Start RAC database
srvctl start database -d oradb

# Stop RAC database
srvctl stop database -d oradb

# Start/stop specific instance
srvctl start instance -d oradb -i oradb1
srvctl stop instance -d oradb -i oradb2
```

✅ **Use Case:** Daily monitoring of RAC database and instances.

---

## **3. Listener & SCAN Listener Management**

```bash
# Check listener status
srvctl status listener -n oranode1
srvctl status listener -n oranode2

# Check SCAN listener
srvctl status scan_listener

# Start/Stop listeners
srvctl start listener -n oranode1
srvctl stop listener -n oranode1
```

✅ **Use Case:** Ensure SCAN listener is working for load balancing.

---

## **4. ASM (Automatic Storage Management)**

```bash
# Check ASM status
srvctl status asm -n oranode1

# List ASM disks
asmcmd lsdg
asmcmd lsct
asmcmd lsdsk

# Show disk groups
asmcmd lsdg
```

✅ **Use Case:** Verify ASM disk groups are mounted and healthy.

---

## **5. Services Management**

```bash
# Check services
srvctl status service -d oradb

# Start/stop a service
srvctl start service -d oradb -s app_svc
srvctl stop service -d oradb -s app_svc
```

✅ **Use Case:** Check load balancing services for OLTP/reporting workloads.

---

## **6. Cluster Logs & Diagnostics**

```bash
# Cluster alert log
tail -f $GRID_HOME/log/`hostname`/alert.log

# ASM alert log
tail -f $GRID_HOME/diag/asm/+asm/+ASM1/trace/alert_+ASM1.log

# Database alert log (for node1)
tail -f $ORACLE_BASE/diag/rdbms/oradb/oradb1/trace/alert_oradb1.log

# CRS logs
tail -f $GRID_HOME/log/`hostname`/crsd/crsd.log
```

✅ **Use Case:** Daily error checking and troubleshooting.

---

## **7. Cluster Verification (CVU)**

```bash
# Check cluster configuration
cluvfy comp health -n all -verbose

# Check node connectivity
cluvfy comp nodecon -n oranode1,oranode2,oranode3 -verbose
```

✅ **Use Case:** Validate cluster networking and node health.

---

## **8. SQL-Level Checks**

```sql
-- Connected to RAC DB
sqlplus / as sysdba

-- Show instances in cluster
select instance_number, instance_name, host_name from gv$instance;

-- Show database role
select database_role, open_mode from v$database;

-- Show connected services
select name, network_name from dba_services;
```

✅ **Use Case:** Verify all RAC instances are open and available.

---

## **9. Node Management**

```bash
# Show all nodes
olsnodes

# Show node status
olsnodes -s -t
```

✅ **Use Case:** Confirm all nodes are part of the cluster.

---

# ✅ Daily RAC DBA Routine (Checklist)

1. Check **cluster status** → `crsctl stat res -t`
2. Check **database status** → `srvctl status database -d oradb`
3. Verify **SCAN listener** → `srvctl status scan_listener`
4. Confirm **ASM disk groups** → `asmcmd lsdg`
5. Review **alert logs** for RAC, ASM, CRS
6. Validate **services running** → `srvctl status service -d oradb`
7. Run **SQL checks** → `gv$instance`, `v$database`

---
