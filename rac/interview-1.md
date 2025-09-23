**Oracle RAC interview questions with solutions** 
---

# 🔹 Oracle RAC Interview Questions and Solutions

---

## **1. What is Oracle RAC?**

**Answer:**

* **Oracle RAC (Real Application Clusters)** allows multiple servers (nodes) to run Oracle instances simultaneously while accessing a single database.
* Provides **high availability, scalability, and load balancing**.
* Use Case: If one node fails, users automatically failover to another node without downtime.

---

## **2. Key Benefits of RAC**

**Answer:**

* **High Availability:** No single point of failure.
* **Scalability:** Add nodes to increase performance.
* **Load Balancing:** Workload is distributed across nodes.
* **Fault Tolerance:** Survives node/network/storage failures.
* **Use Case:** Banking applications where downtime = revenue loss.

---

## **3. RAC Architecture Components**

**Answer:**

* **Clusterware:** Provides node membership, voting disks, cluster communication.
* **ASM (Automatic Storage Management):** Manages shared storage.
* **Interconnect (Private Network):** High-speed communication between nodes.
* **SCAN (Single Client Access Name):** Provides single name for client connections.
* **OCR (Oracle Cluster Registry):** Stores cluster configuration.
* **Use Case:** ASM ensures shared disks are visible and balanced across nodes.

---

## **4. Difference Between RAC and Single Instance**

**Answer:**

* **Single Instance:** One DB, one instance, one server.
* **RAC:** Multiple instances across nodes accessing the same DB.
* **Use Case:** RAC = resilience + horizontal scalability, Single Instance = cost-effective for small workloads.

---

## **5. What is SCAN in RAC?**

**Answer:**

* **Single Client Access Name (SCAN):** A single name clients use to connect to RAC DB.
* Provides **load balancing and failover** without reconfiguring clients.
* Backed by **3 SCAN listeners** running on cluster nodes.
* **Use Case:** Clients connect via `mydb-scan:1521` instead of individual node IPs.

---

## **6. Voting Disk and OCR**

**Answer:**

* **Voting Disk:** Maintains cluster node membership (who is in/out of cluster).
* **OCR (Oracle Cluster Registry):** Stores cluster configuration (resources, services).
* **Use Case:** If a node loses access to voting disk → node eviction happens to protect DB consistency.

---

## **7. RAC Installation Prerequisites**

**Answer:**

* Shared storage (ASM or NFS).
* Public, private, and virtual IPs for each node.
* DNS setup for SCAN.
* Passwordless SSH between nodes.
* Clusterware installed before DB software.
* **Use Case:** Installing 2-node RAC on Oracle Linux with shared ASM disks.

---

## **8. How Clients Connect to RAC?**

**Answer:**

* Via **SCAN Listener** (recommended).
* Client-side connection string uses **SERVICE\_NAME** not SID.

  ```sql
  (DESCRIPTION=
    (ADDRESS=(PROTOCOL=TCP)(HOST=mydb-scan)(PORT=1521))
    (CONNECT_DATA=(SERVICE_NAME=prodservice)))
  ```
* Load balancing across nodes is automatic.

---

## **9. How is Load Balancing Achieved in RAC?**

**Answer:**

* **Server-Side Load Balancing:** SCAN distributes sessions to least-loaded instance.
* **Client-Side Load Balancing:** Connection string includes multiple addresses.
* **Use Case:** 500 users connect → distributed across 2 RAC nodes instead of overloading one.

---

## **10. What Happens if a Node Fails in RAC?**

**Answer:**

* Clusterware detects failure via **heartbeat**.
* Node evicted → sessions failed over to surviving nodes.
* Transactions in-flight → rolled back.
* **Use Case:** One node crash, banking transactions continue via other node with minimal disruption.

---

## **11. RAC Monitoring Tools**

**Answer:**

* **Oracle Enterprise Manager (OEM Cloud Control).**
* **Cluster Verification Utility (cluvfy).**
* **crsctl, srvctl commands.**
* **AWR, ASH reports.**
* **Use Case:** DBA monitors interconnect latency using `cluvfy comp nodecon -n all`.

---

## **12. How Do You Patch RAC?**

**Answer:**

* Use **OPatch** or **GI patching tools**.
* Apply patch on one node → rolling patching to minimize downtime.
* **Use Case:** Patch PSU on Node1, failover sessions to Node2, then patch Node2.

---

## **13. Split Brain in RAC**

**Answer:**

* **Split Brain:** Multiple nodes think they are primary → leads to data corruption.
* Avoided by **voting disks + clusterware**.
* **Node eviction** resolves conflict.
* **Use Case:** If network split occurs, losing node is evicted.

---

## **14. RAC Services**

**Answer:**

* Services are used to direct workloads to specific nodes.
* Example: Reporting workload → Node2, OLTP workload → Node1.
* `srvctl add service -d db -s oltp_svc -r instance1`
* **Use Case:** Reduce load by separating analytics and OLTP.

---

## **15. RAC vs Data Guard**

**Answer:**

* **RAC:** High availability + scalability within same site.
* **Data Guard:** Disaster recovery across geographic regions.
* **Use Case:** Bank uses RAC for local HA + Data Guard for DR in remote site.

---

## **16. RAC Recovery Scenarios**

**Q:** What do you do if OCR or Voting Disk gets corrupted?
**Answer:**

* OCR: Restore using `ocrconfig -restore`.
* Voting Disk: ASM mirroring provides redundancy.
* **Best Practice:** Keep multiple voting disks and OCR backups.

---

## **17. Commands for RAC DBA**

**Answer:**

* `crsctl stat res -t` → Show cluster resources.
* `srvctl status database -d db` → Show DB status.
* `olsnodes -n` → List cluster nodes.
* **Use Case:** DBA quickly checks if SCAN listener is up using `srvctl status listener`.

---

## **18. Real-Time Scenario**

**Q:** Users complain DB sessions hang. How do you troubleshoot RAC?
**Answer:**

1. Check interconnect → `ifconfig`, `ping -s`, `traceroute`.
2. Check `crsctl stat res -t` for node health.
3. Review alert logs, AWR reports.
4. Validate load balancing using services.

* **Use Case:** Found Node2 had NIC issue → traffic rerouted to Node1.

---
