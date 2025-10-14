**comprehensive collection of Oracle Database performance checking scripts** 

---

## 🔍 **1. Check Database & Instance Information**

```sql
SELECT name, open_mode, database_role, log_mode FROM v$database;
SELECT instance_name, status, startup_time, host_name, version FROM v$instance;
```

---

## ⚙️ **2. Check Initialization Parameters Affecting Performance**

```sql
SELECT name, value 
FROM v$parameter 
WHERE name IN ('db_cache_size','shared_pool_size','pga_aggregate_target','sga_target','cpu_count','optimizer_mode');
```

---

## 📊 **3. Check Tablespace Usage**

```sql
SELECT df.tablespace_name,
       df.file_id,
       df.bytes/1024/1024 AS total_mb,
       (df.bytes - NVL(fs.bytes,0))/1024/1024 AS used_mb,
       NVL(fs.bytes,0)/1024/1024 AS free_mb,
       ROUND(((df.bytes - NVL(fs.bytes,0))/df.bytes)*100,2) AS pct_used
FROM dba_data_files df
LEFT JOIN (
    SELECT tablespace_name, file_id, SUM(bytes) bytes
    FROM dba_free_space
    GROUP BY tablespace_name, file_id
) fs ON df.file_id = fs.file_id AND df.tablespace_name = fs.tablespace_name
ORDER BY pct_used DESC;
```

---

## 💾 **4. Check Top SQL by CPU Time**

```sql
SELECT * FROM (
    SELECT sql_id,
           plan_hash_value,
           executions,
           cpu_time/1000000 cpu_sec,
           elapsed_time/1000000 elapsed_sec,
           sql_text
    FROM v$sql
    WHERE executions > 0
    ORDER BY cpu_time DESC
) WHERE ROWNUM <= 10;
```

---

## 🧠 **5. Top SQL by Buffer Gets (Logical Reads)**

```sql
SELECT * FROM (
    SELECT sql_id, buffer_gets, executions, 
           ROUND(buffer_gets/DECODE(executions,0,1,executions)) AS gets_per_exec,
           sql_text
    FROM v$sqlarea
    ORDER BY buffer_gets DESC
) WHERE ROWNUM <= 10;
```

---

## 🕒 **6. Top SQL by Elapsed Time**

```sql
SELECT * FROM (
    SELECT sql_id, elapsed_time/1000000 elapsed_sec, executions,
           sql_text
    FROM v$sql
    WHERE executions > 0
    ORDER BY elapsed_time DESC
) WHERE ROWNUM <= 10;
```

---

## 🧩 **7. Wait Events (System Level)**

```sql
SELECT event, total_waits, time_waited/100 time_waited_secs
FROM v$system_event
WHERE wait_class <> 'Idle'
ORDER BY time_waited DESC;
```

---

## 📈 **8. Session Waits (Active Session Bottlenecks)**

```sql
SELECT sid, event, state, wait_time, seconds_in_wait, p1text, p1, p2text, p2
FROM v$session_wait
WHERE wait_class <> 'Idle';
```

---

## 🔄 **9. Active Sessions by CPU or Wait**

```sql
SELECT s.sid, s.serial#, s.username, s.program, s.sql_id, q.sql_text,
       s.status, s.event, s.state, s.last_call_et
FROM v$session s
LEFT JOIN v$sql q ON s.sql_id = q.sql_id
WHERE s.status = 'ACTIVE'
ORDER BY s.last_call_et DESC;
```

---

## ⚡ **10. Check System Statistics (CPU, IO, Memory)**

```sql
SELECT name, value 
FROM v$sysstat 
WHERE name IN ('CPU used by this session','DB time','physical reads','physical writes');
```

---

## 🧮 **11. Check I/O Performance by Datafile**

```sql
SELECT df.name, phyrds, phywrts, readtim, writetim,
       ROUND(readtim/DECODE(phyrds,0,1,phyrds),2) avg_read_ms,
       ROUND(writetim/DECODE(phywrts,0,1,phywrts),2) avg_write_ms
FROM v$filestat fs, v$datafile df
WHERE fs.file# = df.file#
ORDER BY avg_read_ms DESC;
```

---

## 🧍‍♂️ **12. Top 10 Wait Events (Current Snapshot)**

```sql
SELECT * FROM (
    SELECT event, total_waits, time_waited/100 time_waited_secs
    FROM v$system_event
    WHERE wait_class <> 'Idle'
    ORDER BY time_waited DESC
) WHERE ROWNUM <= 10;
```

---

## 🧰 **13. Check PGA Memory Usage**

```sql
SELECT name, value/1024/1024 AS mb
FROM v$pgastat
WHERE name IN ('aggregate PGA target parameter','total PGA allocated','total PGA inuse');
```

---

## 📦 **14. Check SGA Breakdown**

```sql
SELECT pool, name, bytes/1024/1024 AS mb
FROM v$sgastat
ORDER BY bytes DESC;
```

---

## 🧾 **15. Check Fragmentation in Tablespaces**

```sql
SELECT tablespace_name,
       ROUND((SUM(bytes)/1024/1024),2) AS free_mb,
       COUNT(*) AS free_extents
FROM dba_free_space
GROUP BY tablespace_name
ORDER BY free_mb DESC;
```

---

## 🔍 **16. Check Long Running Queries**

```sql
SELECT s.sid, s.serial#, s.username, q.sql_text, s.last_call_et/60 AS running_minutes
FROM v$session s
JOIN v$sql q ON s.sql_id = q.sql_id
WHERE s.status='ACTIVE' AND s.last_call_et > 60;
```

---

## 📚 **17. Library Cache Efficiency**

```sql
SELECT namespace, gets, gethits, ROUND(gethits*100/gets,2) AS gethit_ratio,
       pins, pinhits, ROUND(pinhits*100/pins,2) AS pinhit_ratio
FROM v$librarycache;
```

---

## 📊 **18. Buffer Cache Hit Ratio**

```sql
SELECT ROUND((1 - (phy.value / (cur.value + con.value))) * 100, 2) AS buffer_cache_hit_ratio
FROM v$sysstat cur, v$sysstat con, v$sysstat phy
WHERE cur.name = 'db block gets'
AND con.name = 'consistent gets'
AND phy.name = 'physical reads';
```

---

## ⚙️ **19. Shared Pool Free Space**

```sql
SELECT ROUND((free_space/pool_size)*100,2) AS pct_free
FROM (
  SELECT (SELECT SUM(bytes) FROM v$sgastat WHERE pool='shared pool' AND name='free memory') free_space,
         (SELECT SUM(bytes) FROM v$sgastat WHERE pool='shared pool') pool_size
  FROM dual
);
```

---

## 🔧 **20. Find Contention on Latches**

```sql
SELECT name, gets, misses, ROUND((misses/gets)*100,2) miss_ratio
FROM v$latch
WHERE gets > 0
ORDER BY miss_ratio DESC;
```

---

## ✅ **21. Check Top Segments by Physical Reads**

```sql
SELECT owner, segment_name, segment_type, value AS physical_reads
FROM v$segment_statistics
WHERE statistic_name = 'physical reads'
AND value > 0
ORDER BY value DESC FETCH FIRST 10 ROWS ONLY;
```

---

## 🧩 **22. Check Undo Tablespace Usage**

```sql
SELECT d.tablespace_name, d.file_name, 
       (d.bytes - NVL(s.bytes,0))/1024/1024 used_mb,
       NVL(s.bytes,0)/1024/1024 free_mb
FROM dba_data_files d
LEFT JOIN (SELECT file_id, SUM(bytes) bytes FROM dba_free_space GROUP BY file_id) s 
ON d.file_id = s.file_id
WHERE d.tablespace_name LIKE '%UNDO%';
```

---

## 📤 **23. Temp Tablespace Usage**

```sql
SELECT tablespace_name, 
       ROUND(SUM(used_blocks)*8/1024,2) used_mb, 
       ROUND(SUM(free_blocks)*8/1024,2) free_mb
FROM v$sort_segment
GROUP BY tablespace_name;
```

---

## 🧾 **24. Check Redo Log Switches**

```sql
SELECT TO_CHAR(first_time,'YYYY-MM-DD HH24:MI') log_time, COUNT(*) AS switches
FROM v$log_history
WHERE first_time > SYSDATE - 1
GROUP BY TO_CHAR(first_time,'YYYY-MM-DD HH24:MI')
ORDER BY log_time DESC;
```

---

## 🧰 **25. Check AWR Baseline (if enabled)**

```sql
SELECT * FROM dba_hist_baseline;
```

---

## 📑 Export Tip

You can spool these results:

```sql
SET LINESIZE 200
SET PAGESIZE 100
SPOOL performance_report.txt
-- Run above scripts here
SPOOL OFF
```

---

