 **deep-dive Oracle ASM (Automatic Storage Management) interview questions with answers and real-world use cases**. 

Here’s a structured guide:

---

# 🔹 Oracle ASM Deep-Dive Interview Questions & Answers

---

## **1. What is ASM and Why Do We Use It?**

**Answer:**

* **ASM (Automatic Storage Management)** is Oracle’s volume manager + file system for Oracle DBs.
* It manages DB files directly on raw disks/diskgroups.
* Benefits:

  * Striping (I/O distribution).
  * Mirroring (redundancy).
  * Online add/remove disks.
  * No OS-level file system needed.

**Use Case:**

* In a RAC setup, ASM ensures all nodes can access the same shared storage (OCR, Voting, DB files).

---

## **2. ASM Architecture**

**Answer:**

* Runs as background processes:

  * **RBAL**: Coordinates disk rebalance.
  * **ARBn**: Performs rebalance operations.
  * **GMON**: Monitors disk group.
* ASM Instance: Lightweight instance with no data dictionary.
* Uses **ASM Disk Groups** (logical collections of disks).

**Use Case:**

* DBA adds a new disk → RBAL/ARBn automatically rebalances data across all disks online.

---

## **3. What are ASM Disk Groups?**

**Answer:**

* Logical containers of ASM disks.
* DB files are placed inside disk groups.
* Redundancy types:

  * **External:** No mirroring (relies on storage).
  * **Normal:** 2-way mirroring.
  * **High:** 3-way mirroring.

**Use Case:**

* Finance DB → uses **High redundancy** for maximum protection.
* Dev/Test → **External redundancy** to save space.

---

## **4. ASM Striping Types**

**Answer:**

* **Fine Striping:** 128 KB chunks, good for redo logs (small I/O).
* **Coarse Striping:** 1 MB chunks, good for datafiles (large I/O).

**Use Case:**

* Place redo logs in **fine striping** for faster sequential writes.

---

## **5. How Do You Add or Drop a Disk in ASM?**

**Answer:**

```sql
-- Add disk
ALTER DISKGROUP DATA ADD DISK '/dev/asm-disk4';

-- Drop disk
ALTER DISKGROUP DATA DROP DISK DATA_0004;
```

* ASM automatically rebalances data.

**Use Case:**

* Adding new SAN LUN for DB growth without downtime.

---

## **6. What is ASM Rebalancing?**

**Answer:**

* When disks are added/removed, ASM redistributes extents evenly.
* Controlled by **POWER limit** (default 1, max 11).

```sql
ALTER DISKGROUP DATA REBALANCE POWER 6;
```

* Higher power = faster rebalance, more CPU usage.

**Use Case:**

* Increase POWER to 6 during maintenance for faster rebalance.

---

## **7. ASM File Types Stored in ASM**

**Answer:**

* DB files: datafiles, controlfiles, redo logs, tempfiles.
* Cluster files: OCR, Voting Disks.
* Flashback logs, RMAN backups (if configured).

**Use Case:**

* In RAC, store OCR + Voting Disk in ASM for cluster consistency.

---

## **8. ASM vs OS File System**

**Answer:**

* ASM provides:

  * Striping, mirroring, rebalance.
  * Oracle-optimized storage.
* OS File Systems (ext4, XFS) lack Oracle awareness.

**Use Case:**

* High-performance RAC DB → ASM preferred.
* Small standalone DB → file system may be enough.

---

## **9. How to Check ASM Diskgroup Space?**

**Answer:**

```sql
SELECT name, total_mb, free_mb, usable_file_mb, type
FROM v$asm_diskgroup;
```

**Use Case:**

* DBA checks **usable\_file\_mb** before adding new datafiles.

---

## **10. How to Migrate DB from File System to ASM?**

**Answer:**

* Use **RMAN**:

```sql
BACKUP AS COPY DATABASE FORMAT '+DATA';
SWITCH DATABASE TO COPY;
```

* Or use **DBCA**.

**Use Case:**

* Migrating a legacy single-instance DB to ASM before RAC implementation.

---

## **11. What is ASM Metadata and Where is it Stored?**

**Answer:**

* ASM metadata tracks file-to-extent-to-disk mapping.
* Stored inside ASM disk headers, mirrored as per redundancy.

**Use Case:**

* If one disk header corrupts, ASM still retrieves metadata from mirrored copy.

---

## **12. How Do You Backup ASM Metadata?**

**Answer:**

```bash
asmcmd md_backup /tmp/asm_metadata.bak
asmcmd md_restore /tmp/asm_metadata.bak
```

**Use Case:**

* Backup before major storage changes to restore diskgroup config if needed.

---

## **13. What is ASM Diskgroup Compatibility Attribute?**

**Answer:**

* `COMPATIBLE.ASM`: ASM instance compatibility.
* `COMPATIBLE.RDBMS`: Minimum DB version allowed.
* `COMPATIBLE.ADVM`: For ASM Dynamic Volume Manager.

**Use Case:**

* Before upgrading DB to 19c, set diskgroup COMPATIBLE.RDBMS=19.0.

---

## **14. Difference Between ASMLIB and UDEV**

**Answer:**

* **ASMLIB:** Oracle-provided library to mark ASM disks.
* **UDEV:** Linux device management, widely used now.
* Oracle recommends **UDEV rules** in modern deployments.

**Use Case:**

* In RHEL9 + 19c RAC, configure ASM disks via `udev` for persistence.

---

## **15. How to Move OCR and Voting Disk to ASM?**

**Answer:**

```bash
ocrconfig -add +OCR_VOTE
crsctl replace votedisk +OCR_VOTE
```

* Ensures cluster config resides in ASM.

**Use Case:**

* Move OCR from file system to ASM for easier cluster storage management.

---

## **16. How to Monitor ASM Performance?**

**Answer:**

* Views:

  * `v$asm_diskgroup` → space usage.
  * `v$asm_operation` → rebalancing progress.
  * `v$asm_disk` → disk health.

**Use Case:**

* Check rebalancing progress after adding new 1 TB disk.

---

## **17. ASM Failures**

**Q:** What happens if a disk fails in ASM?
**Answer:**

* **External redundancy:** Disk loss = data loss.
* **Normal redundancy:** Data still available (mirrored copy).
* **High redundancy:** Two failures tolerated.

**Use Case:**

* In 3-way mirrored setup, two SAN disks can fail and DB still runs.

---

## **18. ASM Commands (asmcmd)**

```bash
asmcmd ls
asmcmd du
asmcmd lsdg
asmcmd lsdsk
asmcmd cp +DATA/oradb/datafile/users.259.12345 /backup/
```

**Use Case:**

* DBA copies DB files from ASM to local FS for troubleshooting.

---

## **19. ASM in RAC Environment**

**Answer:**

* ASM instance runs on each RAC node.
* DB instances connect to ASM instances for storage.
* Diskgroup must be visible to **all nodes**.

**Use Case:**

* RAC cluster on 3 nodes → each node has +ASM instance managing shared storage.

---

## **20. Real-Time Scenario**

**Q:** You add a new 500 GB disk to ASM, but DB users complain of performance issues. What do you do?
**Answer:**

1. Check rebalance operation in progress → `select * from v$asm_operation;`
2. Increase rebalance power to finish faster → `ALTER DISKGROUP DATA REBALANCE POWER 6;`
3. Inform users rebalancing causes temporary I/O overhead.
4. Monitor completion and confirm performance recovery.

---



👉 Do you want me to also **bundle these into a Word/PDF “Oracle ASM Deep Dive Interview Guide”** with **command examples + diagrams** (ASM architecture, striping, mirroring)?
