# Database Opgave — Execution & SSMS Verification Report

- **Student / Operator:** Gustav Fausing (`EXOPI-908\Gustav Fausing`)
- **Date:** November 11, 2026
- **SQL Server Instance:** `EXOPI-908\EXO` (Named Instance `MSSQL$EXO`)
- **SQL Server Version:** Microsoft SQL Server 2025 Enterprise Developer Edition (64-bit) — `17.0.5045.3` (`RTM-CU10-GDR` / `KB5129299`)
- **Client Tool:** Microsoft SQL Server Management Studio (SSMS)
- **Target Directory:** `C:\Exo\Database`

---

## Executive Summary

All tasks across **`SQL.BACKUP.RESTORE.1`**, **`SQL.BACKUP.RESTORE.2`**, **`SQL.MISC.1`**, and **`SQL.MISC.2`** were executed in SSMS and verified directly against SQL Server's internal system catalogs, `msdb` backup/restore history, SQL Server Agent logs, the SQL Server Default Trace, and the SQL Server Error Log (`ERRORLOG`).

| Section | Step | Task Description | Status | Verification Source |
| :--- | :--- | :--- | :--- | :--- |
| **Part 1** | **1.1** | Create `C:\Exo` and `C:\Exo\Database` | Completed | Windows Filesystem |
| **Part 1** | **1.2** | Change default Data, Log, and Backup paths & restart service | Completed | Registry + `SERVERPROPERTY` + Default Trace (`10:01:39`) |
| **Part 1** | **1.3** | Create, verify, drop, and test recovering `DummyDB` | Completed | `ERRORLOG.2` (`10:06:33`) + Default Trace (`10:07:11`) |
| **Part 1** | **1.4** | Download & restore `AdventureWorksLT2022` and `AdventureWorks2022` (`WITH MOVE`) | Completed | `msdb.dbo.restorehistory` (`10:11:46`, `10:12:06`) |
| **Part 1** | **1.5** | Backup `AdventureWorksLT2022` with `COPY_ONLY` | Completed | `msdb.dbo.backupset` (`10:23:02`, `is_copy_only = 1`) |
| **Part 1** | **1.6** | Set `AdventureWorksLT2022` recovery model to `FULL` & take Full backup | Completed | `ERRORLOG` (`10:20:51`) + `msdb.dbo.backupset` (`10:20:59`) |
| **Part 1** | **1.7** | Create & run SQL Agent job `Backup_Log_AdventureWorksLT2022` | Completed | `msdb.dbo.sysjobhistory` (`10:27:14`) |
| **Part 1** | **1.8** | Create `SalesLT.TableA` $\rightarrow$ Log backup $\rightarrow$ Wait 1m $\rightarrow$ `SalesLT.TableB` $\rightarrow$ Log backup | Completed | Default Trace (`10:36:49`, `10:38:43`) + `sysjobhistory` |
| **Part 1** | **1.9** | Point-in-time restore (`STOPAT = '2026-11-11 10:37:30.000'`) | Completed | `msdb.dbo.restorehistory` (`10:44:15`–`10:44:32`) + `sys.tables` |
| **Part 2** | **2.1** | Backup system databases (`master`, `model`, `msdb`, and `tempdb` error check) | Completed | `msdb.dbo.backupset` (`10:47:22`–`10:47:40`) |
| **Part 2** | **2.2** | Create dynamic SQL Agent job `"Point-in-time.restore"` | Completed | `msdb.dbo.sysjobs` & `msdb.dbo.sysjobsteps` |
| **Part 3** | **3.1–3.4** | Theory & Troubleshooting (`SQL.MISC.1`) | Completed | Documented in [README.md](./README.md) |
| **Part 4** | **4.1–4.2** | Database Integrity & `DBCC CHECKDB` (`SQL.MISC.2`) | Completed | `ERRORLOG` (`11:10:46`, `11:10:56`, `11:11:11`) |

---

## Detailed Step-by-Step Evidence & SSMS / SQL Server Logs

### Step 1.1 & Step 1.2 — Directory Creation & Instance Default Paths
- **Action Taken:** Created `C:\Exo\Database`, configured default Data, Log, and Backup directories in SSMS Server Properties, and restarted `MSSQL$EXO` so the new registry settings took effect in memory.
- **Active Instance Verification (`SERVERPROPERTY` & Registry):**
  ```text
  ActiveDataPath   : C:\Exo\Database\
  ActiveLogPath    : C:\Exo\Database\
  ActiveBackupPath : C:\Exo\Database
  RegistryDataPath : C:\Exo\Database
  RegistryLogPath  : C:\Exo\Database
  RegistryBackupPath: C:\Exo\Database
  ```
- **Default Trace Log (Service Restart):**
  ```text
  2026-11-11 10:01:39.210 | Event: Audit Server Starts And Stops
  2026-11-11 10:01:42.467 | sqlserver_start_time (sys.dm_os_sys_info)
  ```

---

### Step 1.3 — Create, Verify, Drop, and Attempt Recovery of `DummyDB`
- **Action Taken:** Created `DummyDB` (verifying `DummyDB.mdf` and `DummyDB_log.ldf` in `C:\Exo\Database`), dropped `DummyDB` via SSMS (`DROP DATABASE [DummyDB];`), and verified that `SELECT name FROM sys.databases WHERE name = N'DummyDB'` returned `0 rows`.
- **SQL Server Error Log (`ERRORLOG.2`) & Default Trace Evidence:**
  ```text
  [ERRORLOG.2]
  2026-11-11 10:06:33.730 spid53 [DBMgr::FindFreeDatabaseID] Next available DbId EX locked: 5
  2026-11-11 10:06:33.810 spid53 Starting up database 'DummyDB'.
  2026-11-11 10:06:33.830 spid53 Parallel redo is started for database 'DummyDB' with worker pool size [4].
  2026-11-11 10:06:33.840 spid53 Parallel redo is shutdown for database 'DummyDB' with worker pool size [4].

  [SQL Server Default Trace - SSMS Query]
  2026-11-11 10:06:33.860 | Event: Object:Created | DB: DummyDB | App: Microsoft SQL Server Management Studio - Query
  2026-11-11 10:07:11.883 | Event: Object:Deleted | DB: DummyDB | App: Microsoft SQL Server Management Studio - Query
  ```
- **Observation (Intentional Trap):** Dropping a database deletes the physical `.mdf` and `.ldf` files directly from the OS filesystem (bypassing the Recycle Bin). Because no backup was taken prior to `DROP DATABASE`, `DummyDB` cannot be recovered.

---

### Step 1.4 — Download and Restore `AdventureWorksLT2022` and `AdventureWorks2022`
- **Action Taken:** Downloaded official `.bak` files to `C:\Exo\Database`, inspected logical filenames with `RESTORE FILELISTONLY`, and restored both databases using `WITH MOVE` to relocate their physical files into `C:\Exo\Database`.
- **SQL Server Error Log (`ERRORLOG.2`) Evidence:**
  ```text
  2026-11-11 10:11:47.040 spid53 Starting up database 'AdventureWorksLT2022'.
  2026-11-11 10:11:47.280 Backup Restore is complete on database 'AdventureWorksLT2022'. The database is now available.
  2026-11-11 10:11:47.290 Backup Database was restored: Database: AdventureWorksLT2022, creation date(time): 2023/05/23(11:55:47), first LSN: 59:328:1, last LSN: 59:352:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorksLT2022.bak'}).
  2026-11-11 10:11:47.310 Backup RESTORE DATABASE successfully processed 890 pages in 0.012 seconds (579.101 MB/sec).

  2026-11-11 10:12:07.130 spid53 Starting up database 'AdventureWorks2022'.
  2026-11-11 10:12:07.390 Backup Restore is complete on database 'AdventureWorks2022'. The database is now available.
  2026-11-11 10:12:07.400 Backup Database was restored: Database: AdventureWorks2022, creation date(time): 2023/05/23(11:55:10), first LSN: 53:29688:1, last LSN: 53:29712:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorks2022.bak'}).
  2026-11-11 10:12:07.420 Backup RESTORE DATABASE successfully processed 25378 pages in 0.167 seconds (1187.195 MB/sec).
  ```
- **Physical File Verification (`sys.master_files`):**
  | DatabaseName | State |LogicalName | PhysicalName |
  | :--- | :--- | :--- | :--- |
  | `AdventureWorksLT2022` | `ONLINE` | `AdventureWorksLT2022_Data` | `C:\Exo\Database\AdventureWorksLT2022.mdf` |
  | `AdventureWorksLT2022` | `ONLINE` | `AdventureWorksLT2022_Log` | `C:\Exo\Database\AdventureWorksLT2022_log.ldf` |
  | `AdventureWorks2022` | `ONLINE` | `AdventureWorks2022` | `C:\Exo\Database\AdventureWorks2022.mdf` |
  | `AdventureWorks2022` | `ONLINE` | `AdventureWorks2022_log` | `C:\Exo\Database\AdventureWorks2022_log.ldf` |

---

### Step 1.5 & Step 1.6 — `COPY_ONLY` Backup, Recovery Model Change (`FULL`), and Full Backup Base
- **Action Taken:**
  - Changed `AdventureWorksLT2022` recovery model from `SIMPLE` to `FULL`.
  - Created a standard Full backup (`AdventureWorksLT2022_Full.bak`, `is_copy_only = 0`) to establish the transaction log chain base (`FirstLSN: 63000000016000001`, `LastLSN: 63000000018400001`).
  - Created a `COPY_ONLY` Full backup (`AdventureWorksLT2022_CopyOnly.bak`, `is_copy_only = 1`), verifying that a `COPY_ONLY` backup does not disrupt an existing Full/Log backup chain.
- **SQL Server Error Log (`ERRORLOG.0`) Evidence:**
  ```text
  2026-11-11 10:20:51.440 spid89 Setting database option RECOVERY to FULL for database 'AdventureWorksLT2022'.
  2026-11-11 10:20:59.130 Backup Database backed up. Database: AdventureWorksLT2022, creation date(time): 2026/11/11(10:11:46), pages dumped: 1036, first LSN: 63:160:1, last LSN: 63:184:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorksLT2022_Full.bak'}).
  2026-11-11 10:20:59.150 Backup BACKUP DATABASE successfully processed 890 pages in 0.018 seconds (386.067 MB/sec).
  2026-11-11 10:23:02.100 Backup Database backed up. Database: AdventureWorksLT2022, creation date(time): 2026/11/11(10:11:46), pages dumped: 1036, first LSN: 63:264:1, last LSN: 63:288:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorksLT2022_CopyOnly.bak'}).
  2026-11-11 10:23:02.110 Backup BACKUP DATABASE successfully processed 890 pages in 0.018 seconds (386.067 MB/sec).
  ```

---

### Step 1.7 & Step 1.8 — SQL Agent Log Backup Job, `SalesLT.TableA`, 1-Minute Wait, and `SalesLT.TableB`
- **Action Taken:**
  1. Started the `SQLAgent$EXO` service (`10:25:59`).
  2. Created SQL Agent job `Backup_Log_AdventureWorksLT2022` and executed initial test run (`10:27:14`).
  3. Created `SalesLT.TableA` (`10:36:49.947`) and ran `Backup_Log_AdventureWorksLT2022` (`10:37:03`).
  4. Waited >1 minute (establishing target `STOPAT` window at `2026-11-11 10:37:30.000`).
  5. Created `SalesLT.TableB` (`10:38:43.293`) and ran `Backup_Log_AdventureWorksLT2022` (`10:38:51`).
- **SQL Server Error Log & Default Trace Evidence:**
  ```text
  [ERRORLOG.0 — SQL Server Agent Startup]
  2026-11-11 10:25:59.750 spid65 Configuration option 'Agent XPs' changed from 0 to 1. Run the RECONFIGURE statement to install.

  [Default Trace — Table Creation in SSMS]
  2026-11-11 10:36:49.960 | Object:Created | DB: AdventureWorksLT2022 | Object: TableA | App: Microsoft SQL Server Management Studio - Query
  2026-11-11 10:38:43.303 | Object:Created | DB: AdventureWorksLT2022 | Object: TableB | App: Microsoft SQL Server Management Studio - Query
  ```
- **SQL Server Agent Job Execution Log (`msdb.dbo.sysjobhistory`):**
  ```text
  [Run #1 — Step 1.7 Initial Log Backup (10:27:14)]
  Job: Backup_Log_AdventureWorksLT2022 | Step 1 (Run Transaction Log Backup) | Status: Succeeded
  Message: Executed as user: NT SERVICE\SQLAgent$EXO. Processed 8 pages for database 'AdventureWorksLT2022', file 'AdventureWorksLT2022_Log' on file 1. BACKUP LOG successfully processed 8 pages in 0.003 seconds (20.833 MB/sec).
  Output File: C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn (LSN: 63:160:1 -> 63:288:1)

  [Run #2 — Step 1.8 After SalesLT.TableA (10:37:03)]
  Job: Backup_Log_AdventureWorksLT2022 | Step 1 (Run Transaction Log Backup) | Status: Succeeded
  Message: Executed as user: NT SERVICE\SQLAgent$EXO. Processed 5 pages for database 'AdventureWorksLT2022', file 'AdventureWorksLT2022_Log' on file 1. BACKUP LOG successfully processed 5 pages in 0.003 seconds (11.718 MB/sec).
  Output File: C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn (LSN: 63:288:1 -> 63:360:1)

  [Run #3 — Step 1.8 After SalesLT.TableB (10:38:51)]
  Job: Backup_Log_AdventureWorksLT2022 | Step 1 (Run Transaction Log Backup) | Status: Succeeded
  Message: Executed as user: NT SERVICE\SQLAgent$EXO. Processed 7 pages for database 'AdventureWorksLT2022', file 'AdventureWorksLT2022_Log' on file 1. BACKUP LOG successfully processed 7 pages in 0.003 seconds (18.229 MB/sec).
  Output File: C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn (LSN: 63:360:1 -> 63:472:1)
  ```

---

### Step 1.9 — Point-in-Time Restore (`STOPAT`)
- **Action Taken:** Overwrote `AdventureWorksLT2022` with `AdventureWorksLT2022_Full.bak` (`WITH REPLACE, NORECOVERY`), applied `102714.trn` (`WITH NORECOVERY`), applied `103703.trn` (`WITH NORECOVERY`), and applied `103851.trn` (`WITH STOPAT = N'2026-11-11 10:37:30.000', RECOVERY`).
- **SQL Server Error Log (`ERRORLOG.0`) & `msdb.dbo.restorehistory` Evidence:**
  ```text
  2026-11-11 10:44:15.650 Backup Database was restored: Database: AdventureWorksLT2022, first LSN: 63:160:1, last LSN: 63:184:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Full.bak'}.
  2026-11-11 10:44:21.590 Backup Log was restored. Database: AdventureWorksLT2022, first LSN: 63:160:1, last LSN: 63:288:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn'}.
  2026-11-11 10:44:26.800 Backup Log was restored. Database: AdventureWorksLT2022, first LSN: 63:288:1, last LSN: 63:360:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn'}.
  2026-11-11 10:44:32.470 Backup Restore is complete on database 'AdventureWorksLT2022'. The database is now available.
  2026-11-11 10:44:32.470 Backup Log was restored. Database: AdventureWorksLT2022, first LSN: 63:360:1, last LSN: 63:472:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn'} (stop_at = 2026-11-11 10:37:30.000).
  2026-11-11 10:44:40.170 spid89 Setting database option MULTI_USER to ON for database 'AdventureWorksLT2022'.
  ```
- **Post-Restore Verification Query Output (`SalesLT` Tables):**
  ```text
  name   | create_date
  -------+------------------------
  TableA | 2026-11-11 10:36:49.947
  (1 row affected — TableB is absent!)

  ID | CreatedAt                   | Note
  ---+-----------------------------+------------------------------------
  1  | 2026-11-11 10:36:49.9675232 | TableA created before STOPAT target
  ```

---

### Step 2.1 — Backup the Four System Databases (`master`, `model`, `msdb`, `tempdb`)
- **Action Taken:** Backed up `master`, `model`, and `msdb` to `C:\Exo\Database`, and attempted `BACKUP DATABASE [tempdb]`, observing intentional error **Msg 3147** (*"Backup and restore operations are not allowed on database tempdb."*).
- **SQL Server Error Log (`ERRORLOG.0`) Evidence:**
  ```text
  2026-11-11 10:47:22.450 Backup Database backed up. Database: master, pages dumped: 588, first LSN: 247:408:1, last LSN: 247:432:1, device: {'C:\Exo\Database\master_Full.bak'}.
  2026-11-11 10:47:22.460 Backup BACKUP DATABASE successfully processed 578 pages in 0.021 seconds (214.843 MB/sec).
  2026-11-11 10:47:30.070 Backup Database backed up. Database: model, pages dumped: 419, first LSN: 43:2808:1, last LSN: 43:2832:1, device: {'C:\Exo\Database\model_Full.bak'}.
  2026-11-11 10:47:30.080 Backup BACKUP DATABASE successfully processed 410 pages in 0.019 seconds (168.379 MB/sec).
  2026-11-11 10:47:40.100 Backup Database backed up. Database: msdb, pages dumped: 2318, first LSN: 194:1872:1, last LSN: 194:1896:1, device: {'C:\Exo\Database\msdb_Full.bak'}.
  2026-11-11 10:47:40.120 Backup BACKUP DATABASE successfully processed 2154 pages in 0.037 seconds (454.708 MB/sec).
  ```

---

### Step 2.2 — Parameterized Dynamic Restore Job (`"Point-in-time.restore"`)
- **Action Taken:** Created SQL Agent job `Point-in-time.restore` in `msdb` with step `Dynamic Point-in-Time Restore`, which uses `@DatabaseName`, `@BackupFolder`, `@FullBackupFile`, and `@StopAt` parameters, inspects `LastLSN` via `RESTORE HEADERONLY`, enumerates `*.trn` files via `sys.dm_os_enumerate_filesystem`, and applies all subsequent log backups in strict `FirstLSN` order.
- **`msdb.dbo.sysjobs` Verification:**
  ```text
  Job Name              | Enabled | Step Name                     | Target DB
  ----------------------+---------+-------------------------------+----------
  Point-in-time.restore | 1       | Dynamic Point-in-Time Restore | master
  ```

---

### Part 4 (`SQL.MISC.2`) — Database Integrity Verification (`DBCC CHECKDB`)
- **Action Taken:** Executed all three `DBCC CHECKDB` variations against `AdventureWorks2022` in SSMS:
  1. Standard full check (`WITH NO_INFOMSGS, ALL_ERRORMSGS`)
  2. Physical-only check (`WITH PHYSICAL_ONLY, NO_INFOMSGS, ALL_ERRORMSGS`)
  3. Extended logical check (`WITH EXTENDED_LOGICAL_CHECKS, NO_INFOMSGS, ALL_ERRORMSGS`)
- **SQL Server Error Log (`ERRORLOG.0`) Evidence:**
  ```text
  2026-11-11 11:10:46.150 spid89 DBCC CHECKDB (AdventureWorks2022) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 3 seconds. Internal database snapshot has split point LSN = 00000035:00007aa0:0001 and first LSN = 00000035:00007a90:0001.

  2026-11-11 11:10:56.250 spid89 DBCC CHECKDB (AdventureWorks2022) WITH all_errormsgs, no_infomsgs, physical_only executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 0 seconds. Internal database snapshot has split point LSN = 00000035:00007ae8:0001 and first LSN = 00000035:00007ad8:0001.

  2026-11-11 11:11:07.680 spid89 AppDomain 3 (master.sys[runtime].2) created.
  2026-11-11 11:11:11.990 spid89 DBCC CHECKDB (AdventureWorks2022) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 6 seconds. Internal database snapshot has split point LSN = 00000035:00007b30:0001 and first LSN = 00000035:00007b20:0001.
  ```

---

## Complete Backup & Restore Catalog (`msdb`)

### All Backups Recorded in `msdb.dbo.backupset` (`2026-11-11`)
| Database | Type | Copy-Only | Recovery Model | Finish Time | First LSN | Last LSN | Size (Bytes) | Physical Path |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `AdventureWorksLT2022` | `D` (Full) | `0` | `FULL` | `10:20:59` | `63000000016000001` | `63000000018400001` | `8,482,816` | `C:\Exo\Database\AdventureWorksLT2022_Full.bak` |
| `AdventureWorksLT2022` | `D` (Full) | `1` | `FULL` | `10:23:02` | `63000000026400001` | `63000000028800001` | `8,482,816` | `C:\Exo\Database\AdventureWorksLT2022_CopyOnly.bak` |
| `AdventureWorksLT2022` | `L` (Log) | `0` | `FULL` | `10:27:14` | `63000000016000001` | `63000000028800001` | `86,016` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn` |
| `AdventureWorksLT2022` | `L` (Log) | `0` | `FULL` | `10:37:03` | `63000000028800001` | `63000000036000001` | `86,016` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn` |
| `AdventureWorksLT2022` | `L` (Log) | `0` | `FULL` | `10:38:51` | `63000000036000001` | `63000000047200001` | `86,016` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn` |
| `master` | `D` (Full) | `0` | `SIMPLE` | `10:47:22` | `247000000040800001` | `247000000043200001` | `4,813,824` | `C:\Exo\Database\master_Full.bak` |
| `model` | `D` (Full) | `0` | `FULL` | `10:47:30` | `43000000280800001` | `43000000283200001` | `3,431,424` | `C:\Exo\Database\model_Full.bak` |
| `msdb` | `D` (Full) | `0` | `SIMPLE` | `10:47:40` | `194000000187200001` | `194000000189600001` | `18,989,056` | `C:\Exo\Database\msdb_Full.bak` |

### All Restores Recorded in `msdb.dbo.restorehistory` (`2026-11-11`)
| ID | Restore Time | Destination DB | Type | Replace | `STOPAT` Timestamp | Source Backup File |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `1` | `10:11:46.963` | `AdventureWorksLT2022` | `D` | `0` | `NULL` | `C:\Exo\Database\AdventureWorksLT2022.bak` |
| `2` | `10:12:06.773` | `AdventureWorks2022` | `D` | `0` | `NULL` | `C:\Exo\Database\AdventureWorks2022.bak` |
| `4` | `10:44:15.540` | `AdventureWorksLT2022` | `D` | `1` | `NULL` | `C:\Exo\Database\AdventureWorksLT2022_Full.bak` |
| `5` | `10:44:21.527` | `AdventureWorksLT2022` | `L` | `0` | `NULL` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn` |
| `6` | `10:44:26.740` | `AdventureWorksLT2022` | `L` | `0` | `NULL` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn` |
| `7` | `10:44:32.287` | `AdventureWorksLT2022` | `L` | `0` | `2026-11-11 10:37:30.000` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn` |

---

## Part 3 & Part 4 — Questions & Answers (`SQL.MISC.1` & `SQL.MISC.2`)

### 1. How many SQL Server Instances can be active on the same server?
- **Answer:** Up to **50 instances** on a standalone server (1 Default instance + up to 49 Named instances, or 50 Named instances).
  - *(Note: On a Windows Server Failover Cluster (WSFC) using shared Cluster Disks, the limit is **25 instances**, or up to 50 if using SMB file shares).*
- **Key Takeaway:** Each instance has its own independent system databases (`master`, `model`, `msdb`, `tempdb`), memory allocation (`max server memory`), and Windows services (`MSSQLSERVER` for the default instance, or `MSSQL$<InstanceName>` and `SQLAgent$<InstanceName>` for named instances—such as `MSSQL$EXO` and `SQLAgent$EXO` on `EXOPI-908\EXO`).

---

### 2. SQL Server Patching
#### Q: Are Cumulative Updates (CUs) installed and managed per SQL instance?
- **Answer:** **Yes** (with shared components updated machine-wide). When running a SQL Server Cumulative Update installer on a server with multiple instances, the setup wizard allows selecting which specific SQL Server instance(s) to patch, or patching all instances at once. This lets a DBA patch a Dev/Test instance before a Production instance on the same host, while shared machine-level components (Client Tools, SQL Writer, Integration Services) are updated once for the machine.

#### Q: Are security updates included in Cumulative Updates (CUs)?
- **Answer:** **Yes.** Cumulative Updates are *cumulative*—each CU contains all previously released bug fixes, performance improvements, **and all security updates (GDRs)** since the initial RTM release of that SQL Server version.
- **Live Evidence from Today's Session (`ERRORLOG.1` at `10:16:28`):**
  During today's session on `EXOPI-908\EXO`, the instance was updated from `17.0.5035.5` (`RTM-CU10`, `KB5125471`) to `17.0.5045.3` (`RTM-CU10-GDR`, `KB5129299`), demonstrating the **CU+GDR** servicing model in action.

---

### 3. SQL Server Browser Service
#### Q: Describe the purpose of the SQL Server Browser Service.
- **Answer:** The **SQL Server Browser Service** runs as a Windows service listening on **UDP port 1434**. Its purpose is to help clients locate **Named Instances** (such as `EXOPI-908\EXO`). Because named instances use **dynamic TCP ports** assigned at startup by default, a client connecting only by instance name does not know which TCP port to use. The client sends a UDP request to port 1434 asking for the instance's port, and the Browser Service responds with the active TCP port (or named pipe) for that named instance.

#### Q: Can you connect to the local SQL instance without the Browser Service running? Please explain why / why not.
- **Answer:** **Yes, you can!** (In fact, throughout today's entire lab on `EXOPI-908\EXO`, the `SQLBrowser` service was **Stopped / Disabled**, and SSMS connected without issue). There are three reasons why:
  1. **Shared Memory Protocol (Local Connections):** When connecting locally on the same machine (e.g., from SSMS or `sqlcmd -S "lpc:localhost\EXO"`), the client uses the **Shared Memory** protocol (`lpc:`), which communicates directly through local memory and bypasses TCP/IP ports and the Browser Service completely.
  2. **Default Instance Uses Static Port 1433:** Even over TCP/IP, the **Default Instance** (`MSSQLSERVER`) listens on well-known static **TCP port 1433**, which clients try automatically without querying the Browser Service.
  3. **Explicit Port Number for Named Instances:** Even for a **Named Instance** over TCP/IP with the Browser Service stopped, a client can connect by specifying the instance's TCP port number explicitly in the connection string using a comma (e.g., `ServerName,50123`).

---

### 4. Authentication, Authorization & Connection Troubleshooting
#### Q: What is the difference between Windows Authentication and SQL Server Authentication?
| Feature | Windows Authentication (Integrated Security) | SQL Server Authentication |
| :--- | :--- | :--- |
| **Where credentials live** | Active Directory (Domain Controller) or Local Windows OS | Stored inside SQL Server's `master` database (`sys.sql_logins`) |
| **How identity is verified** | Windows verifies the user via **Kerberos** or **NTLM** token; SQL Server never sees the password | User sends username & password directly to SQL Server to hash and compare |
| **Security & Policies** | Enforces Active Directory policies (MFA, password rotation, complexity, account lockout, SSO) | Subject only to local Windows password policy if `CHECK_POLICY = ON`; requires Mixed Mode |
| **Best Use Case** | **Recommended default** for all domain-joined users and application service accounts | Legacy applications, non-domain clients (Linux/containers without Kerberos), or third-party tools |

#### Q: A client machine cannot connect to SQL Server. Everyone else is connecting without issues. Describe your troubleshooting approach.
Because **everyone else is connecting without issues**, we immediately know the SQL Server service is online, TCP/IP is enabled, and the server's firewall is allowing traffic on the SQL port. The issue is isolated to **this specific client machine, its network path/VLAN, or this user's credentials**:

1. **Get the exact error message from the client** to split the investigation into **Network** vs. **Authentication**:
   - *Network Error (Error 26 / Error 53 / Error 10060 — "A network-related or instance-specific error occurred")* $\rightarrow$ Client cannot reach SQL Server.
   - *Authentication Error (Error 18456 — "Login failed for user '...'")* $\rightarrow$ Client reached SQL Server, but the login was rejected.
2. **If Network Error (Cannot reach server):**
   - **Verify connection string:** Check for typos in the server name, instance name, or port.
   - **Check DNS & IP reachability:** Run `ping <ServerName>` and `nslookup <ServerName>` on the client machine.
   - **Test TCP Port connectivity:** Run `Test-NetConnection -ComputerName <ServerName> -Port 1433` (or the named instance port) in PowerShell on the client. If blocked, check if the client is on a restricted VLAN/VPN/subnet or has a local outbound firewall rule blocking the port. If connecting to a named instance by name, verify UDP port 1434 (SQL Browser) is reachable.
   - **Check Client Network Configuration / Aliases:** Check `cliconfg.exe` or ODBC Data Sources on the client for stale SQL Server aliases or disabled client protocols.
3. **If Authentication Error (Error 18456 — Login failed):**
   - **Inspect the SQL Server Error Log (`ERRORLOG`):** While the client only sees a generic error message for security reasons, the SQL Server Error Log records the exact **State number** (e.g., `Error: 18456, Severity: 14, State: X`):
     - *State 2 / 5:* Invalid userid (Login does not exist in SQL Server or user is missing from the AD group).
     - *State 7:* Login is disabled.
     - *State 8:* Password mismatch.
     - *State 11 / 12:* Valid login, but server access failed / denied.
     - *State 16:* User lacks permission to access their configured **Default Database** (or the default database is offline/dropped).
   - **Check Active Directory:** Verify the user's Windows account is not locked out or expired.

#### Q: What is the difference between Authentication and Authorization?
- **Authentication (AuthN) — *"Who are you?"***
  - Verifies the **identity** of a user or service connecting to the SQL Server instance.
  - Configured at the **Server Instance level** via **Logins** (`sys.server_principals`) using Windows Authentication or SQL Server Authentication.
- **Authorization (AuthZ) — *"What are you allowed to do?"***
  - Determines which **databases, schemas, tables, views, or procedures** an already-authenticated user is permitted to read, modify, or execute.
  - Configured via **Database Users** (`sys.database_principals`), **Server/Database Roles** (`sysadmin`, `db_datareader`, `db_datawriter`), and **`GRANT` / `DENY` / `REVOKE`** permissions.

---

### 5. Database Integrity & `DBCC CHECKDB` (`SQL.MISC.2`)
#### Q: What is the difference between Physical and Logical Database Integrity?
| Integrity Type | What It Checks | Examples of Corruption |
| :--- | :--- | :--- |
| **Physical Integrity** | Verifies that physical 8 KB pages on disk can be read and that internal page headers, checksums, and allocation bitmaps are intact. | **Torn Page** (partial 8 KB page write before power loss), **Checksum Failure** (disk bit-flip), corrupt allocation maps (**GAM, SGAM, PFS, IAM**). |
| **Logical Integrity** | Verifies that higher-level relational and index structures across pages make logical sense and match metadata rules. | A **Non-Clustered Index** pointing to a missing/mismatched row in the Clustered Index; broken page pointers in a B-Tree; catalog metadata inconsistencies. |

#### Q: How does `DBCC CHECKDB` work, and how should corruption be repaired?
- **How it works:** `DBCC CHECKDB` creates an internal, hidden copy-on-write **Database Snapshot** for transactional consistency (`Internal database snapshot has split point LSN...` as seen in today's `ERRORLOG`) and executes three checks:
  1. `DBCC CHECKALLOC` (disk space allocation structures)
  2. `DBCC CHECKTABLE` (physical & logical integrity of all tables and indexes)
  3. `DBCC CHECKCATALOG` (system metadata consistency)
- **Recovery & Repair Hierarchy:**
  1. **Restore from a Clean Backup (Preferred — Zero Data Loss):** Use your Full + Transaction Log (`.trn`) backup chain (or Page-Level Restore `RESTORE DATABASE ... PAGE = 'file:page'`) to recover without losing data.
  2. **`REPAIR_REBUILD` (Safe — No Data Loss):** Rebuilds corrupted non-clustered indexes when the underlying base table (Clustered Index / Heap) is still intact.
  3. **`REPAIR_ALLOW_DATA_LOSS` (Last Resort Only):** Deallocates (deletes) unreadable 8 KB data pages and patches pointers around the gap so the database can come online, permanently deleting the rows that lived on those pages.
