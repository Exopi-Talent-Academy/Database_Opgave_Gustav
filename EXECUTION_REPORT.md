# Database Opgave — Execution & SSMS Verification Report

- **Student / Operator:** Gustav Fausing (`EXOPI-908\Gustav Fausing`)
- **Dates:** November 11, 2026 (Parts 1–4), November 13, 2026 (Parts 4A, 4B, & 5.1/5.2), November 23, 2026 (Parts 5.2, 6, 7 [Week 1], & 8), & November 26, 2026 (Parts 9–12)
- **SQL Server Instance:** `EXOPI-908\EXO` (Named Instance `MSSQL$EXO`)
- **SQL Server Version:** Microsoft SQL Server 2025 Enterprise Developer Edition (64-bit) — `17.0.5045.3` (`RTM-CU10-GDR` / `KB5129299`)
- **Client Tool:** Microsoft SQL Server Management Studio (SSMS)
- **Target Directory:** `C:\Exo\Database`

---

## Executive Summary

All tasks across **`SQL.BACKUP.RESTORE.1`**, **`SQL.BACKUP.RESTORE.2`**, **`SQL.MISC.1`**, **`SQL.MISC.2`**, **`Corruption3`**, **`Corruption4`**, **`DatabaseMail.papercut.SMTP`**, **`DatabaseMail.Notification`** (`CheckDB`, `DatabaseDML`, `Alert_ProcessKilled`, `Alert_TempDB_RapidGrowth`, & `Alert_TempDB_Shrink`), **Steve Stedman Week 1 (`CorruptionChallenge1`)**, **System-Versioned Temporal Tables (`SalesLT.Product`)**, **`TempDB` Configuration & Resource Governance**, **IT-Revision 12-Month Restart Audit**, and **`AdventureWorks2022` Space Monitoring** were executed in SSMS/PowerShell and verified directly against SQL Server's internal system catalogs, `msdb` backup/restore/Database Mail/Agent history, Extended Events, the SQL Server Default Trace, Windows Firewall rules, and the SQL Server Error Log (`ERRORLOG`).

| Section | Step | Task Description | Status | Verification Source |
| :--- | :--- | :--- | :--- | :--- |
| **Part 1** | **1.1** | Create `C:\Exo` and `C:\Exo\Database` | Completed | Windows Filesystem (`2026-11-11`) |
| **Part 1** | **1.2** | Change default Data, Log, and Backup paths & restart service | Completed | Registry + `SERVERPROPERTY` + Default Trace (`2026-11-11`) |
| **Part 1** | **1.3** | Create, verify, drop, and test recovering `DummyDB` | Completed | `ERRORLOG.2` + Default Trace (`2026-11-11`) |
| **Part 1** | **1.4** | Download & restore `AdventureWorksLT2022` and `AdventureWorks2022` (`WITH MOVE`) | Completed | `msdb.dbo.restorehistory` (`2026-11-11`) |
| **Part 1** | **1.5** | Backup `AdventureWorksLT2022` with `COPY_ONLY` | Completed | `msdb.dbo.backupset` (`2026-11-11`, `is_copy_only = 1`) |
| **Part 1** | **1.6** | Set `AdventureWorksLT2022` recovery model to `FULL` & take Full backup | Completed | `ERRORLOG` + `msdb.dbo.backupset` (`2026-11-11`) |
| **Part 1** | **1.7** | Create & run SQL Agent job `Backup_Log_AdventureWorksLT2022` | Completed | `msdb.dbo.sysjobhistory` (`2026-11-11`) |
| **Part 1** | **1.8** | Create `SalesLT.TableA` $\rightarrow$ Log backup $\rightarrow$ Wait 1m $\rightarrow$ `SalesLT.TableB` $\rightarrow$ Log backup | Completed | Default Trace + `sysjobhistory` (`2026-11-11`) |
| **Part 1** | **1.9** | Point-in-time restore (`STOPAT`) | Completed | `msdb.dbo.restorehistory` + `sys.tables` (`2026-11-11`) |
| **Part 2** | **2.1** | Backup system databases (`master`, `model`, `msdb`, and `tempdb` error check) | Completed | `msdb.dbo.backupset` (`2026-11-11`) |
| **Part 2** | **2.2** | Create dynamic SQL Agent job `"Point-in-time.restore"` | Completed | `msdb.dbo.sysjobs` & `msdb.dbo.sysjobsteps` (`2026-11-11`) |
| **Part 3** | **3.1–3.4** | Theory & Troubleshooting (`SQL.MISC.1`) | Completed | Documented in [README.md](./README.md) |
| **Part 4** | **4.1–4.2** | Database Integrity & `DBCC CHECKDB` (`SQL.MISC.2`) | Completed | `ERRORLOG` (`2026-11-11`) |
| **Part 4A** | **4A.1–4A.3** | Restore `Corruption3`, diagnose 26 `CustomerName` index errors, & rebuild cleanly | Completed | `msdb.dbo.restorehistory` + `ERRORLOG` (`2026-11-13`) |
| **Part 4B** | **4B.1–4B.4** | Power-failure recovery of `Corruption4` (no backups) via Hack-Attach, `EMERGENCY`, & `REPAIR_ALLOW_DATA_LOSS` | Completed | `ERRORLOG` + `sys.databases` (`2026-11-13`) |
| **Part 5** | **5.1** | Install Papercut SMTP, lock firewall to `Private`, & verify PowerShell `Send-MailMessage` | Completed | `Get-NetFirewallRule` (`Private`) + Papercut port 25 (`2026-11-13`) |
| **Part 5** | **5.2** | Configure Database Mail (`Papercut_Local_Profile`), link SQL Server Agent, restart `SQLAgent$EXO`, & verify `sysmail_allitems` (`sent`) | Completed | `ERRORLOG` + `msdb.dbo.sysmail_allitems` (`2026-11-13` & `2026-11-23`) |
| **Part 6** | **6.1** | Create `CheckDB` SQL Agent job, verify clean run, inject `CorruptTestDB` corruption, & verify HTML email alert | Completed | `msdb.dbo.sysjobhistory` + `sysmail_allitems` (`mailitem_id 2`, `2026-11-23`) |
| **Part 6** | **6.2–6.3** | Create `DatabaseDML` job + `DBA_TraceWatermark`, test `EXECUTE AS LOGIN = N'John'` DB/User create/delete alerts | Completed | `sysjobhistory` + `sysmail_allitems` (`mailitem_id 3, 4`, `2026-11-23`) |
| **Part 6** | **6.4** | Theory: Receiving critical alerts when company mail services are offline | Completed | Documented in [README.md](./README.md) |
| **Part 7** | **7 (Week 1)** | Steve Stedman Week 1 (`CorruptionChallenge1`): recover `dbo.Revenue` (54 rows, zero data loss) via NC indexes & verify 0 errors | Completed | `restorehistory` + `ERRORLOG` (`2026-11-23`) |
| **Part 8** | **8.1–8.3** | Check table eligibility, enable `SYSTEM_VERSIONING` on `SalesLT.Product` (`SalesLT.ProductHistory`), & verify `INSERT`/`UPDATE`/`DELETE`/time-travel | Completed | `sys.tables` (`temporal_type = 2`) + `SalesLT.ProductHistory` (`2` rows, `2026-11-23`) |
| **Part 9** | **9.1–9.7** | Set `TempDB` `MAXSIZE` (`1024 MB`/`512 MB`), move all 9 files to `C:\Exo\Database\TempDB`, restart `MSSQL$EXO`, & configure Resource Governor (`256 MB` cap) | Completed | `sys.master_files` + `tempdb.sys.database_files` + `ERRORLOG` + `sys.resource_governor_configuration` (`2026-11-26`) |
| **Part 10** | **10.1–10.2** | Audit available `ERRORLOG` archives, document default 7-file limitation, raise `NumErrorLogs` to `99`, & register `usp_LogInstanceStartup` (`sp_procoption`) | Completed | Registry (`NumErrorLogs = 99`) + `master.sys.procedures` (`is_auto_executed = 1`, `2026-11-26`) |
| **Part 11** | **11.1–11.2** | Configure `DBA_ProcessTermination_Monitor` XE session + `Alert_ProcessKilled` SQL Agent job & test `KILL 51` HTML alert | Completed | `sysjobhistory` + `sysmail_allitems` (`mailitem_id 5`, `sent`, `2026-11-26`) |
| **Part 11** | **11.3** | Create `Alert_TempDB_RapidGrowth` job + 1-min schedule (`Every_1_Minute_TempDB_Watch`) & trigger alert via `#RapidGrowthTest` (`159.25 MB`) | Completed | `sysjobhistory` + `sysmail_allitems` (`mailitem_id 6`, `sent`, `2026-11-26`) |
| **Part 11** | **11.4** | Create `Alert_TempDB_Shrink` job + `DBA_TempDB_FileBaseline`, test `128 MB` $\rightarrow$ `32 MB` shrink, & reset all 9 `tempdb` files back to `8.00 MB` | Completed | `sysjobhistory` + `sysmail_allitems` (`mailitem_id 7, 8`, `sent`, `2026-11-26`) |
| **Part 12** | **12.1–12.3** | Verify `AdventureWorks2022` (`ONLINE`), run 2-row Data vs. Log space query, & compare with `AdventureWorks2022.sys.sp_spaceused` | Completed | `sys.database_files` + `FILEPROPERTY` + `sys.sp_spaceused` (`272.00 MB`, `2026-11-26`) |

---

## Detailed Step-by-Step Evidence & SSMS / SQL Server Logs

### Step 1.1 & Step 1.2 — Directory Creation & Instance Default Paths (`2026-11-11`)
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
  2026-11-11 | Event: Audit Server Starts And Stops
  2026-11-11 | sqlserver_start_time (sys.dm_os_sys_info)
  ```

---

### Step 1.3 — Create, Verify, Drop, and Attempt Recovery of `DummyDB` (`2026-11-11`)
- **Action Taken:** Created `DummyDB` (verifying `DummyDB.mdf` and `DummyDB_log.ldf` in `C:\Exo\Database`), dropped `DummyDB` via SSMS (`DROP DATABASE [DummyDB];`), and verified that `SELECT name FROM sys.databases WHERE name = N'DummyDB'` returned `0 rows`.
- **SQL Server Error Log (`ERRORLOG.2`) & Default Trace Evidence:**
  ```text
  [ERRORLOG.2]
  2026-11-11 spid53 [DBMgr::FindFreeDatabaseID] Next available DbId EX locked: 5
  2026-11-11 spid53 Starting up database 'DummyDB'.
  2026-11-11 spid53 Parallel redo is started for database 'DummyDB' with worker pool size [4].
  2026-11-11 spid53 Parallel redo is shutdown for database 'DummyDB' with worker pool size [4].

  [SQL Server Default Trace - SSMS Query]
  2026-11-11 | Event: Object:Created | DB: DummyDB | App: Microsoft SQL Server Management Studio - Query
  2026-11-11 | Event: Object:Deleted | DB: DummyDB | App: Microsoft SQL Server Management Studio - Query
  ```
- **Observation (Intentional Trap):** Dropping a database deletes the physical `.mdf` and `.ldf` files directly from the OS filesystem (bypassing the Recycle Bin). Because no backup was taken prior to `DROP DATABASE`, `DummyDB` cannot be recovered.

---

### Step 1.4 — Download and Restore `AdventureWorksLT2022` and `AdventureWorks2022` (`2026-11-11`)
- **Action Taken:** Downloaded official `.bak` files to `C:\Exo\Database`, inspected logical filenames with `RESTORE FILELISTONLY`, and restored both databases using `WITH MOVE` to relocate their physical files into `C:\Exo\Database`.
- **SQL Server Error Log (`ERRORLOG.2`) Evidence:**
  ```text
  2026-11-11 spid53 Starting up database 'AdventureWorksLT2022'.
  2026-11-11 Backup Restore is complete on database 'AdventureWorksLT2022'. The database is now available.
  2026-11-11 Backup Database was restored: Database: AdventureWorksLT2022, creation date: 2023/05/23, first LSN: 59:328:1, last LSN: 59:352:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorksLT2022.bak'}).
  2026-11-11 Backup RESTORE DATABASE successfully processed 890 pages in 0.012 seconds (579.101 MB/sec).

  2026-11-11 spid53 Starting up database 'AdventureWorks2022'.
  2026-11-11 Backup Restore is complete on database 'AdventureWorks2022'. The database is now available.
  2026-11-11 Backup Database was restored: Database: AdventureWorks2022, creation date: 2023/05/23, first LSN: 53:29688:1, last LSN: 53:29712:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorks2022.bak'}).
  2026-11-11 Backup RESTORE DATABASE successfully processed 25378 pages in 0.167 seconds (1187.195 MB/sec).
  ```
- **Physical File Verification (`sys.master_files`):**
  | DatabaseName | State | LogicalName | PhysicalName |
  | :--- | :--- | :--- | :--- |
  | `AdventureWorksLT2022` | `ONLINE` | `AdventureWorksLT2022_Data` | `C:\Exo\Database\AdventureWorksLT2022.mdf` |
  | `AdventureWorksLT2022` | `ONLINE` | `AdventureWorksLT2022_Log` | `C:\Exo\Database\AdventureWorksLT2022_log.ldf` |
  | `AdventureWorks2022` | `ONLINE` | `AdventureWorks2022` | `C:\Exo\Database\AdventureWorks2022.mdf` |
  | `AdventureWorks2022` | `ONLINE` | `AdventureWorks2022_log` | `C:\Exo\Database\AdventureWorks2022_log.ldf` |

---

### Step 1.5 & Step 1.6 — `COPY_ONLY` Backup, Recovery Model Change (`FULL`), and Full Backup Base (`2026-11-11`)
- **Action Taken:**
  - Changed `AdventureWorksLT2022` recovery model from `SIMPLE` to `FULL`.
  - Created a standard Full backup (`AdventureWorksLT2022_Full.bak`, `is_copy_only = 0`) to establish the transaction log chain base (`FirstLSN: 63000000016000001`, `LastLSN: 63000000018400001`).
  - Created a `COPY_ONLY` Full backup (`AdventureWorksLT2022_CopyOnly.bak`, `is_copy_only = 1`), verifying that a `COPY_ONLY` backup does not disrupt an existing Full/Log backup chain.
- **SQL Server Error Log (`ERRORLOG.0`) Evidence:**
  ```text
  2026-11-11 spid89 Setting database option RECOVERY to FULL for database 'AdventureWorksLT2022'.
  2026-11-11 Backup Database backed up. Database: AdventureWorksLT2022, creation date: 2026/11/11, pages dumped: 1036, first LSN: 63:160:1, last LSN: 63:184:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorksLT2022_Full.bak'}).
  2026-11-11 Backup BACKUP DATABASE successfully processed 890 pages in 0.018 seconds (386.067 MB/sec).
  2026-11-11 Backup Database backed up. Database: AdventureWorksLT2022, creation date: 2026/11/11, pages dumped: 1036, first LSN: 63:264:1, last LSN: 63:288:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\AdventureWorksLT2022_CopyOnly.bak'}).
  2026-11-11 Backup BACKUP DATABASE successfully processed 890 pages in 0.018 seconds (386.067 MB/sec).
  ```

---

### Step 1.7 & Step 1.8 — SQL Agent Log Backup Job, `SalesLT.TableA`, 1-Minute Wait, and `SalesLT.TableB` (`2026-11-11`)
- **Action Taken:**
  1. Started the `SQLAgent$EXO` service.
  2. Created SQL Agent job `Backup_Log_AdventureWorksLT2022` and executed initial test run.
  3. Created `SalesLT.TableA` and ran `Backup_Log_AdventureWorksLT2022`.
  4. Waited >1 minute to separate `TableA` and `TableB` for the `STOPAT` recovery window.
  5. Created `SalesLT.TableB` and ran `Backup_Log_AdventureWorksLT2022`.
- **SQL Server Error Log & Default Trace Evidence:**
  ```text
  [ERRORLOG.0 — SQL Server Agent Startup]
  2026-11-11 spid65 Configuration option 'Agent XPs' changed from 0 to 1. Run the RECONFIGURE statement to install.

  [Default Trace — Table Creation in SSMS]
  2026-11-11 | Object:Created | DB: AdventureWorksLT2022 | Object: TableA | App: Microsoft SQL Server Management Studio - Query
  2026-11-11 | Object:Created | DB: AdventureWorksLT2022 | Object: TableB | App: Microsoft SQL Server Management Studio - Query
  ```
- **SQL Server Agent Job Execution Log (`msdb.dbo.sysjobhistory`):**
  ```text
  [Run #1 — Step 1.7 Initial Log Backup (2026-11-11)]
  Job: Backup_Log_AdventureWorksLT2022 | Step 1 (Run Transaction Log Backup) | Status: Succeeded
  Message: Executed as user: NT SERVICE\SQLAgent$EXO. Processed 8 pages for database 'AdventureWorksLT2022', file 'AdventureWorksLT2022_Log' on file 1. BACKUP LOG successfully processed 8 pages in 0.003 seconds (20.833 MB/sec).
  Output File: C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn (LSN: 63:160:1 -> 63:288:1)

  [Run #2 — Step 1.8 After SalesLT.TableA (2026-11-11)]
  Job: Backup_Log_AdventureWorksLT2022 | Step 1 (Run Transaction Log Backup) | Status: Succeeded
  Message: Executed as user: NT SERVICE\SQLAgent$EXO. Processed 5 pages for database 'AdventureWorksLT2022', file 'AdventureWorksLT2022_Log' on file 1. BACKUP LOG successfully processed 5 pages in 0.003 seconds (11.718 MB/sec).
  Output File: C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn (LSN: 63:288:1 -> 63:360:1)

  [Run #3 — Step 1.8 After SalesLT.TableB (2026-11-11)]
  Job: Backup_Log_AdventureWorksLT2022 | Step 1 (Run Transaction Log Backup) | Status: Succeeded
  Message: Executed as user: NT SERVICE\SQLAgent$EXO. Processed 7 pages for database 'AdventureWorksLT2022', file 'AdventureWorksLT2022_Log' on file 1. BACKUP LOG successfully processed 7 pages in 0.003 seconds (18.229 MB/sec).
  Output File: C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn (LSN: 63:360:1 -> 63:472:1)
  ```

---

### Step 1.9 — Point-in-Time Restore (`STOPAT`) (`2026-11-11`)
- **Action Taken:** Overwrote `AdventureWorksLT2022` with `AdventureWorksLT2022_Full.bak` (`WITH REPLACE, NORECOVERY`), applied `102714.trn` (`WITH NORECOVERY`), applied `103703.trn` (`WITH NORECOVERY`), and applied `103851.trn` (`WITH STOPAT, RECOVERY`) targeting the window between `TableA` and `TableB`.
- **SQL Server Error Log (`ERRORLOG.0`) & `msdb.dbo.restorehistory` Evidence:**
  ```text
  2026-11-11 Backup Database was restored: Database: AdventureWorksLT2022, first LSN: 63:160:1, last LSN: 63:184:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Full.bak'}.
  2026-11-11 Backup Log was restored. Database: AdventureWorksLT2022, first LSN: 63:160:1, last LSN: 63:288:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn'}.
  2026-11-11 Backup Log was restored. Database: AdventureWorksLT2022, first LSN: 63:288:1, last LSN: 63:360:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn'}.
  2026-11-11 Backup Restore is complete on database 'AdventureWorksLT2022'. The database is now available.
  2026-11-11 Backup Log was restored. Database: AdventureWorksLT2022, first LSN: 63:360:1, last LSN: 63:472:1, device: {'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn'} (STOPAT applied).
  2026-11-11 spid89 Setting database option MULTI_USER to ON for database 'AdventureWorksLT2022'.
  ```
- **Post-Restore Verification Query Output (`SalesLT` Tables):**
  ```text
  name   | create_date
  -------+------------
  TableA | 2026-11-11
  (1 row affected — TableB is absent!)

  ID | CreatedAt  | Note
  ---+------------+------------------------------------
  1  | 2026-11-11 | TableA created before STOPAT target
  ```

---

### Step 2.1 — Backup the Four System Databases (`master`, `model`, `msdb`, `tempdb`) (`2026-11-11`)
- **Action Taken:** Backed up `master`, `model`, and `msdb` to `C:\Exo\Database`, and attempted `BACKUP DATABASE [tempdb]`, observing intentional error **Msg 3147** (*"Backup and restore operations are not allowed on database tempdb."*).
- **SQL Server Error Log (`ERRORLOG.0`) Evidence:**
  ```text
  2026-11-11 Backup Database backed up. Database: master, pages dumped: 588, first LSN: 247:408:1, last LSN: 247:432:1, device: {'C:\Exo\Database\master_Full.bak'}.
  2026-11-11 Backup BACKUP DATABASE successfully processed 578 pages in 0.021 seconds (214.843 MB/sec).
  2026-11-11 Backup Database backed up. Database: model, pages dumped: 419, first LSN: 43:2808:1, last LSN: 43:2832:1, device: {'C:\Exo\Database\model_Full.bak'}.
  2026-11-11 Backup BACKUP DATABASE successfully processed 410 pages in 0.019 seconds (168.379 MB/sec).
  2026-11-11 Backup Database backed up. Database: msdb, pages dumped: 2318, first LSN: 194:1872:1, last LSN: 194:1896:1, device: {'C:\Exo\Database\msdb_Full.bak'}.
  2026-11-11 Backup BACKUP DATABASE successfully processed 2154 pages in 0.037 seconds (454.708 MB/sec).
  ```

---

### Step 2.2 — Parameterized Dynamic Restore Job (`"Point-in-time.restore"`) (`2026-11-11`)
- **Action Taken:** Created SQL Agent job `Point-in-time.restore` in `msdb` with step `Dynamic Point-in-Time Restore`, which uses `@DatabaseName`, `@BackupFolder`, `@FullBackupFile`, and `@StopAt` parameters, inspects `LastLSN` via `RESTORE HEADERONLY`, enumerates `*.trn` files via `sys.dm_os_enumerate_filesystem`, and applies all subsequent log backups in strict `FirstLSN` order.
- **`msdb.dbo.sysjobs` Verification:**
  ```text
  Job Name              | Enabled | Step Name                     | Target DB
  ----------------------+---------+-------------------------------+----------
  Point-in-time.restore | 1       | Dynamic Point-in-Time Restore | master
  ```

---

### Part 4 (`SQL.MISC.2`) — Database Integrity Verification (`DBCC CHECKDB`) (`2026-11-11`)
- **Action Taken:** Executed all three `DBCC CHECKDB` variations against `AdventureWorks2022` in SSMS:
  1. Standard full check (`WITH NO_INFOMSGS, ALL_ERRORMSGS`)
  2. Physical-only check (`WITH PHYSICAL_ONLY, NO_INFOMSGS, ALL_ERRORMSGS`)
  3. Extended logical check (`WITH EXTENDED_LOGICAL_CHECKS, NO_INFOMSGS, ALL_ERRORMSGS`)
- **SQL Server Error Log (`ERRORLOG.0`) Evidence:**
  ```text
  2026-11-11 spid89 DBCC CHECKDB (AdventureWorks2022) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 3 seconds. Internal database snapshot has split point LSN = 00000035:00007aa0:0001 and first LSN = 00000035:00007a90:0001.

  2026-11-11 spid89 DBCC CHECKDB (AdventureWorks2022) WITH all_errormsgs, no_infomsgs, physical_only executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 0 seconds. Internal database snapshot has split point LSN = 00000035:00007ae8:0001 and first LSN = 00000035:00007ad8:0001.

  2026-11-11 spid89 AppDomain 3 (master.sys[runtime].2) created.
  2026-11-11 spid89 DBCC CHECKDB (AdventureWorks2022) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 6 seconds. Internal database snapshot has split point LSN = 00000035:00007b30:0001 and first LSN = 00000035:00007b20:0001.
  ```

---

### Part 4A (`2020.Q.RESTORE.CHECKDB.Database.Integrity.Corruption3`) — Restore & Non-Clustered Index Repair (`2026-11-13`)
- **Action Taken:**
  1. Copied `Corruption3.bak` to `C:\Exo\Database`, inspected logical names via `RESTORE FILELISTONLY` (`SalesDBData`, `SalesDBLog`), and restored `Corruption3` using `WITH MOVE`.
  2. Ran `DBCC CHECKDB (N'Corruption3') WITH NO_INFOMSGS, ALL_ERRORMSGS;`, observing **0 allocation errors and 26 consistency errors** in `dbo.Customers` (`object ID 453576654`), non-clustered index `CustomerName` (`ID 2`), with minimum repair level `repair_rebuild`.
  3. Disabled and rebuilt `CustomerName` (`ALTER INDEX [CustomerName] ON [dbo].[Customers] DISABLE;` followed by `REBUILD`) so SQL Server rebuilt from the intact `CustomerPK` clustered index rather than scanning the corrupted non-clustered index.
  4. Verified `DBCC CHECKDB (N'Corruption3')` returned **0 errors**.
- **SQL Server Error Log (`ERRORLOG`) Evidence:**
  ```text
  2026-11-13 Backup Restore is complete on database 'Corruption3'. The database is now available.
  2026-11-13 Backup Database was restored: Database: Corruption3, creation date: 2026/11/12, first LSN: 374:1728:1, last LSN: 374:1752:1, number of dump devices: 1, device information: (FILE=1, TYPE=DISK: {'C:\Exo\Database\Corruption3.bak'}).
  2026-11-13 spid71 CHECKDB found 0 allocation errors and 26 consistency errors in database 'Corruption3'.
  2026-11-13 spid71 DBCC CHECKDB (Corruption3) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 26 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 3 seconds.
  2026-11-13 spid71 DBCC CHECKDB (Corruption3) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 2 seconds.
  2026-11-13 spid97 DBCC CHECKDB (Corruption3) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors. Elapsed time: 0 hours 0 minutes 2 seconds.
  ```
- **Final State Verification (`sys.databases`, `sys.master_files`, `sys.indexes`):**
  - `Corruption3` is `ONLINE` / `MULTI_USER`, with `SalesDBData` (`C:\Exo\Database\Corruption3.mdf`) and `SalesDBLog` (`C:\Exo\Database\Corruption3_log.ldf`) `ONLINE`.
  - `CustomerPK` (`CLUSTERED`) and `CustomerName` (`NONCLUSTERED`) on `dbo.Customers` both show `is_disabled = 0`.

---

### Part 4B (`Corruption4`) — Power Failure Simulation & Recovery Without Backups (`2026-11-13`)
- **Action Taken:**
  1. Copied `Corruption4.mdf` and `Corruption4_log.ldf` to `C:\Exo\Database`, unblocked files, and configured NTFS Full Control permissions.
  2. Verified `FOR ATTACH` failure (`Msg 824` checksum error on page `(0:0)` in `Corruption4_log.ldf`) and `FOR ATTACH_REBUILD_LOG` failure (`Msg 1813` unclean shutdown).
  3. Performed Hack-Attach + `SET EMERGENCY`, then ran `DBCC CHECKDB (N'Corruption4')`, observing **0 allocation errors and 1 consistency error** in `sys.sysiscols` (`object ID 55`, index `nc1` ID `2`).
  4. Initial `REPAIR_ALLOW_DATA_LOSS` attempt hit `OS error 5 (Access is denied.)` because the original pre-existing `C:\Exo\Database\Corruption4_log.ldf` lacked `NT SERVICE\MSSQL$EXO:(F)`.
  5. Renamed the damaged log file to `C:\Exo\Database\Corruption4_log.ldf.corrupt`, re-ran `SET SINGLE_USER` $\rightarrow$ `DBCC CHECKDB (N'Corruption4', REPAIR_ALLOW_DATA_LOSS)` $\rightarrow$ `SET MULTI_USER`, rebuilding a clean `Corruption4_log.ldf` and repairing `sys.sysiscols`.
  6. Verified `Corruption4` is `ONLINE` / `MULTI_USER` and `DBCC CHECKDB` returned **0 errors**.
- **SQL Server Error Log (`ERRORLOG`) Evidence:**
  ```text
  2026-11-13 spid71 Synchronize Database 'Corruption4' (8) with Resource Database.
  2026-11-13 spid71 CHECKDB found 0 allocation errors and 1 consistency errors in database 'Corruption4'.
  2026-11-13 spid71 CREATE FILE encountered operating system error 5(Access is denied.) while attempting to open or create the physical file 'C:\Exo\Database\Corruption4_log.ldf'.
  2026-11-13 spid71 EMERGENCY MODE DBCC CHECKDB (Corruption4, repair_allow_data_loss) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing terminated abnormally due to error state 4.
  2026-11-13 spid71 Setting database option SINGLE_USER to ON for database 'Corruption4'.
  2026-11-13 spid71 Warning: The log for database 'Corruption4' has been rebuilt. Transactional consistency has been lost.
  2026-11-13 spid71 DBCC DBREPAIR: 'Nonclustered' index restored for 'Corruption4.sysiscols'.
  2026-11-13 spid71 CHECKDB fixed 0 allocation errors and 1 consistency errors in database 'Corruption4'.
  2026-11-13 spid71 Setting database option MULTI_USER to ON for database 'Corruption4'.
  2026-11-13 spid71 DBCC CHECKDB (Corruption4) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors.
  2026-11-13 spid97 DBCC CHECKDB (Corruption4) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors.
  ```
- **Final State Verification (`sys.databases` & `sys.master_files`):**
  - `Corruption4`: `ONLINE`, `MULTI_USER`, `is_read_only = 0`.
  - Files: `Corruption4` (`C:\Exo\Database\Corruption4.mdf`, `ONLINE`) and `Corruption4_log` (`C:\Exo\Database\Corruption4_log.ldf`, `ONLINE`).

---

### Part 5 (`DatabaseMail.papercut.SMTP` & SQL Agent Mail) — Local SMTP & SQL Server Database Mail (`2026-11-13` & `2026-11-23`)
- **Action Taken:**
  1. (`2026-11-13`) Installed **Papercut SMTP** (`Papercut.Service`, PID `20340`, listening on TCP port `25`) and switched its Windows Defender Firewall inbound rules (TCP and UDP) to **`Private` only**.
  2. (`2026-11-13`) Verified SMTP connectivity from PowerShell (`Send-MailMessage -SmtpServer localhost -Port 25 ...`).
  3. (`2026-11-13`) Enabled `Database Mail XPs`, verified `msdb` Service Broker (`is_broker_enabled = 1`), created `Papercut_Local_Account` (`localhost:25`, `enable_ssl = 0`) and `Papercut_Local_Profile` (default public profile), and sent a test email to `test@local.local`.
  4. (`2026-11-23`) Linked SQL Server Agent to `Papercut_Local_Profile`, restarted `SQLAgent$EXO`, and confirmed `sysmail_allitems` delivery status (`mailitem_id 1: sent`).
- **Windows Firewall & Port 25 Verification:**
  ```text
  DisplayName : papercut.service.exe | Enabled : True | Direction : Inbound | Action : Allow | Profile : Private (TCP & UDP)
  LocalAddress: 0.0.0.0 / 127.0.0.1 | LocalPort : 25 | State : Listen / Established | Process : Papercut.Service (PID 20340)
  ```
- **SQL Server `ERRORLOG`, `sp_configure`, & `msdb` Database Mail Evidence:**
  ```text
  [ERRORLOG]
  2026-11-13 spid71 Configuration option 'Database Mail XPs' changed from 0 to 1. Run the RECONFIGURE statement to install.
  2026-11-23 spid53 Configuration option 'Agent XPs' changed from 1 to 0. Run the RECONFIGURE statement to install.
  2026-11-23 spid53 Configuration option 'Agent XPs' changed from 0 to 1. Run the RECONFIGURE statement to install.

  [sp_configure]
  Database Mail XPs | config_value: 1 | run_value: 1

  [msdb.dbo.sysmail_account / sysmail_profile]
  AccountName: Papercut_Local_Account | email_address: sqlserver@local.local | servername: localhost | port: 25 | enable_ssl: 0 | ProfileName: Papercut_Local_Profile | is_default: 1

  [msdb.dbo.sysmail_allitems (mailitem_id 1)]
  mailitem_id: 1 | recipients: test@local.local | subject: SQL Server Database Mail Test | sent_status: sent | sent_date: 2026-11-13
  ```

---

### Part 6 (`DatabaseMail.Notification`) — `CheckDB` Alerts, `DatabaseDML` Auditing & `John` Impersonation (`2026-11-23`)
- **Action Taken:**
  1. Created SQL Agent job `CheckDB` running `DBCC CHECKDB (...) WITH NO_INFOMSGS, ALL_ERRORMSGS, TABLERESULTS` across all `ONLINE` system and user databases.
  2. Verified clean baseline runs succeeded with 0 emails queued.
  3. Created disposable `CorruptTestDB`, corrupted 4 bytes on page `(1:296)` via `DBCC WRITEPAGE`, ran `CheckDB`, observed `3` errors (`Error: 824` on `(1:296)`, `repair_allow_data_loss`), received HTML alert email (`mailitem_id 2`, `sent`), and dropped `CorruptTestDB`.
  4. Created `msdb.dbo.DBA_TraceWatermark` (`DATABASE_DDL`, `DBUSER_DDL`) and 2-step SQL Agent job `DatabaseDML`.
  5. Created SQL Login `John` (`dbcreator` fixed server role), impersonated `John` (`EXECUTE AS LOGIN = N'John'`) to create `JohnsLabDB`, reverted, created/dropped user `LabAppUser` in `JohnsLabDB`, dropped `JohnsLabDB` as `John`, and ran `DatabaseDML`—delivering both HTML audit emails (`mailitem_id 3` and `4`, `sent`).
- **`msdb.dbo.sysjobhistory`, `sysmail_allitems`, & `ERRORLOG` Evidence:**
  ```text
  [ERRORLOG — CorruptTestDB Injection & CheckDB Detection]
  2026-11-23 spid54 User "EXOPI-908\Gustav Fausing" is modifying bytes 0 to 4 of page (1:296) in database "CorruptTestDB".
  2026-11-23 spid56 DBCC CHECKDB (CorruptTestDB) WITH all_errormsgs, no_infomsgs, tableresults executed by NT SERVICE\SQLAgent$EXO found 3 errors and repaired 0 errors.

  [msdb.dbo.sysjobhistory]
  CheckDB     | 2026-11-23 | Succeeded | (Clean baseline: 0 emails)
  CheckDB     | 2026-11-23 | Succeeded | Mail (Id: 2) queued (CorruptTestDB 3 errors)
  DatabaseDML | 2026-11-23 | Succeeded | Step 1: Mail (Id: 3) queued | Step 2: Mail (Id: 4) queued
  CheckDB     | 2026-11-23 | Succeeded | (Post-Part 7 clean baseline across all 8 online DBs)

  [msdb.dbo.sysmail_allitems]
  mailitem_id: 2 | recipients: dba@anywhere.example | subject: [CRITICAL] DBCC CHECKDB Corruption Detected on EXOPI-908\EXO          | HTML | sent | 2026-11-23
  mailitem_id: 3 | recipients: dba@anywhere.example | subject: [SQL Audit] Database Create/Delete Detected on EXOPI-908\EXO      | HTML | sent | 2026-11-23
  mailitem_id: 4 | recipients: dba@anywhere.example | subject: [SQL Audit] Database User Create/Delete Detected on EXOPI-908\EXO | HTML | sent | 2026-11-23
  ```

---

### Part 7 (Week 1 — Steve Stedman's Database Corruption Challenge: `CorruptionChallenge1`) (`2026-11-23`)
- **Action Taken:**
  1. Extracted `CorruptionChallenge1.zip` (`CorruptionChallenge1.bak`) to `C:\Exo\Database` and restored `CorruptionChallenge1` using `WITH MOVE`.
  2. Ran `DBCC CHECKDB (N'CorruptionChallenge1')`, observing **0 allocation errors and 4 consistency errors** in `dbo.Revenue` (`object ID 245575913`, clustered index `clustId` ID `1`, `page (1:280)`, `row 3`, `Msg 8944`: `ColumnOffsets <= (nextRec - pRec)` failed with values `3139` and `288`; minimum repair level `repair_allow_data_loss`).
  3. Inspected page `(1:280)` (`m_slotCnt = 27`, covering `id` 28–54) and confirmed `dbo.Revenue` had two intact non-clustered indexes covering all 5 columns (`ncDeptIdYear` on `DepartmentID, Revenue` + clustered key `id`; `ncBadNameForAnIndex` on `Year` `INCLUDE (Notes)` + `id`).
  4. Verified `id = 31` showed `Notes = NULL` when scanned via `clustId` but retained its intact `Notes` string (`E5DCAE9B-...`) in `ncBadNameForAnIndex`.
  5. Reconstructed all `54` rows into staging `dbo.Revenue_Recovered` by joining `ncDeptIdYear` and `ncBadNameForAnIndex` on `id`, verified `EXCEPT` returned only `id = 31`, truncated `dbo.Revenue` inside a transaction, reloaded all `54` rows with `SET IDENTITY_INSERT dbo.Revenue ON`, dropped `dbo.Revenue_Recovered`, and verified `DBCC CHECKDB` + the `CheckDB` SQL Agent job both returned **0 errors**.
- **SQL Server Error Log (`ERRORLOG`) & Verification Evidence:**
  ```text
  [ERRORLOG]
  2026-11-23 Backup Database was restored: Database: CorruptionChallenge1, creation date: 2015/04/11, device: {'C:\Exo\Database\CorruptionChallenge1.bak'}.
  2026-11-23 spid54 Table error: Object ID 245575913, index ID 1, ..., page (1:280), row 3. Test (ColumnOffsets <= (nextRec - pRec)) failed. Values are 3139 and 288.
  2026-11-23 spid54 CHECKDB found 0 allocation errors and 4 consistency errors in table 'Revenue' (object ID 245575913).
  2026-11-23 spid54 DBCC CHECKDB (CorruptionChallenge1) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing found 0 errors and repaired 0 errors.
  2026-11-23 spid51 DBCC CHECKDB (CorruptionChallenge1) WITH all_errormsgs, no_infomsgs, tableresults executed by NT SERVICE\SQLAgent$EXO found 0 errors and repaired 0 errors.

  [Post-Recovery Verification]
  dbo.Revenue row count: 54 | IDENT_CURRENT('dbo.Revenue'): 54
  id = 31 | DepartmentID: 1 | Revenue: 50000 | Year: 2008 | Notes: E5DCAE9B-D8F0-4BA2-B198-215A2D4671F5This is some varchar dat...
  ```

---

### Part 8 — System-Versioned Temporal Tables (`AdventureWorksLT2022` / `SalesLT.ProductHistory`) (`2026-11-23`)
- **Action Taken:**
  1. Executed the temporal eligibility script across `AdventureWorksLT2022`, listing all eligible schema-prefixed tables.
  2. Added `HIDDEN` period columns `ValidFrom` / `ValidTo` (`DF_Product_ValidFrom`, `DF_Product_ValidTo`) and enabled `SYSTEM_VERSIONING = ON (HISTORY_TABLE = SalesLT.ProductHistory)` on `SalesLT.Product`.
  3. Tested `INSERT` (`ProductID = 1000`, `TT-BIKE-001`, `Black`, `999.00`), `UPDATE` (`Red`, `1299.00`), `DELETE`, and `FOR SYSTEM_TIME AS OF` / `FOR SYSTEM_TIME ALL` queries.
- **Temporal Catalog & `SalesLT.ProductHistory` Verification:**
  ```text
  [sys.tables]
  SalesLT.Product        | SYSTEM_VERSIONED_TEMPORAL_TABLE | HistoryTable: SalesLT.ProductHistory
  SalesLT.ProductHistory | HISTORY_TABLE                   | NULL

  [SalesLT.Product] (Current)
  Row count: 295 (TT-BIKE-001 absent after DELETE)

  [SalesLT.ProductHistory] (2 archived rows)
  ProductID: 1000 | Temporal Test Bike | TT-BIKE-001 | Black | 999.0000  | Archived: 2026-11-23
  ProductID: 1000 | Temporal Test Bike | TT-BIKE-001 | Red   | 1299.0000 | Archived: 2026-11-23
  ```

---

### Part 9 — `TempDB` Sizing, File Relocation (`C:\Exo\Database\TempDB`) & Resource Governor (`2026-11-26`)
- **Action Taken:**
  1. Set explicit `MAXSIZE` limits (`1024.00 MB` per data file, `512.00 MB` for `templog`) and uniform `64.00 MB` fixed autogrowth across all 9 `tempdb` files (`tempdev`, `temp2`–`temp8`, `templog`).
  2. Created `C:\Exo\Database\TempDB`, updated all 9 file paths via `ALTER DATABASE [tempdb] MODIFY FILE`, and restarted `MSSQL$EXO`, verifying fresh physical files on disk (`8.00 MB` each) and matching `sys.master_files` / `tempdb.sys.database_files` paths.
  3. Configured SQL Server 2025 Resource Governor: created Workload Group `wg_Reporting_TempDBCapped` (`GROUP_MAX_TEMPDB_DATA_MB = 256.0`, `REQUEST_MAX_MEMORY_GRANT_PERCENT = 15`, `MAX_DOP = 2`), registered `dbo.fn_RG_Classifier`, and executed `ALTER RESOURCE GOVERNOR RECONFIGURE`.
- **Verification (`ERRORLOG`, `sys.master_files`, & `sys.resource_governor_configuration`):**
  ```text
  [ERRORLOG]
  2026-11-26 Server SQL Server is starting at normal priority base (=7).
  2026-11-26 spid32s Starting up database 'tempdb'.
  2026-11-26 spid32s The tempdb database has 8 data file(s).
  2026-11-26 spid28s Recovery is complete. This is an informational message only.
  2026-11-26 spid56 Resource governor reconfiguration succeeded.

  [tempdb Files — Catalog & Active Paths (after final 8 MB reset on 2026-11-26)]
  file_id 1 (tempdev, ROWS)   : C:\Exo\Database\TempDB\tempdb.mdf          | Size: 8.00 MB | Max: 1024.00 MB | Growth: 64.00 MB
  file_id 2 (templog, LOG)    : C:\Exo\Database\TempDB\templog.ldf         | Size: 8.00 MB | Max:  512.00 MB | Growth: 64.00 MB
  file_id 3..9 (temp2..temp8) : C:\Exo\Database\TempDB\tempdb_mssql_2..8.ndf | Size: 8.00 MB | Max: 1024.00 MB | Growth: 64.00 MB

  [Resource Governor]
  is_enabled: 1 | ClassifierFunc: fn_RG_Classifier | WorkloadGroup: wg_Reporting_TempDBCapped (group_max_tempdb_data_mb: 256.0)
  ```

---

### Part 10 — IT-Revision 12-Month Restart Audit & Remediation (`2026-11-26`)
- **Action Taken:**
  1. Verified current `ERRORLOG` archive span via `xp_enumerrorlogs` (`Archive 0`–`6`, spanning only back to `2026-11-11`—proving the default 7-file limit cannot retain 12 months of history).
  2. Increased `NumErrorLogs` in the instance registry from `6` to `99`.
  3. Created `msdb.dbo.DBA_InstanceRestartAudit` and `master.dbo.usp_LogInstanceStartup`, and enabled automatic startup execution via `sp_procoption` (`is_auto_executed = 1`).
- **Verification:**
  ```text
  [Registry] NumErrorLogs = 99
  [master.sys.procedures] usp_LogInstanceStartup | is_auto_executed = 1 | create_date: 2026-11-26
  [msdb.dbo.DBA_InstanceRestartAudit] Ready (created post-boot on 2026-11-26; will populate automatically on next instance startup)
  ```

---

### Part 11 — Advanced SQL Agent HTML/CSS Alerts (`Alert_ProcessKilled`, `Alert_TempDB_RapidGrowth`, `Alert_TempDB_Shrink`) (`2026-11-26`)
- **Action Taken:**
  1. **Step 11.2 (`Alert_ProcessKilled`):** Created & started Extended Events session `DBA_ProcessTermination_Monitor` (`STARTUP_STATE = ON`, writing to `C:\Exo\Database\DBA_ProcessTermination_Monitor*.xel`), added watermark `'KILL_MONITOR_UTC'` to `msdb.dbo.DBA_TraceWatermark`, created SQL Agent job `Alert_ProcessKilled` (`SET QUOTED_IDENTIFIER ON;`), tested by killing `SPID 51`, and verified `mailitem_id = 5` (`sent`).
  2. **Step 11.3 (`Alert_TempDB_RapidGrowth`):** Created `msdb.dbo.DBA_TempDB_GrowthHistory`, created `Alert_TempDB_RapidGrowth` on 1-minute schedule `Every_1_Minute_TempDB_Watch`, ran `#RapidGrowthTest` (`159.25 MB` internal usage in `SampleID = 2`), and verified `mailitem_id = 6` (`sent`).
  3. **Step 11.4 (`Alert_TempDB_Shrink`):** Created `msdb.dbo.DBA_TempDB_FileBaseline`, created `Alert_TempDB_Shrink` attached to `Every_1_Minute_TempDB_Watch`, grew `tempdev` to `128.00 MB`, shrunk `tempdev` to `32.00 MB` via `DBCC SHRINKFILE` (triggering `mailitem_id = 7`), and then shrank all `tempdb` data files back to `8.00 MB` (triggering `mailitem_id = 8`).
- **`ERRORLOG`, `msdb.dbo.sysjobhistory`, & `msdb.dbo.sysmail_allitems` Evidence:**
  ```text
  [ERRORLOG]
  2026-11-26 spid56 Process ID 51 was killed by hostname EXOPI-908, host process ID 8012.

  [msdb.dbo.sysjobhistory]
  Alert_ProcessKilled      | 2026-11-26 | Succeeded | Mail (Id: 5) queued
  Alert_TempDB_RapidGrowth | 2026-11-26 | Succeeded | Mail (Id: 6) queued (InternalUsedMB: 159.25 MB)
  Alert_TempDB_Shrink      | 2026-11-26 | Succeeded | Mail (Id: 7) queued (tempdev 128.00 -> 32.00 MB)
  Alert_TempDB_Shrink      | 2026-11-26 | Succeeded | Mail (Id: 8) queued (Reset tempdev/temp2..temp8 to 8.00 MB)

  [msdb.dbo.sysmail_allitems]
  mailitem_id: 5 | dba@anywhere.example | [WARNING] SQL Process KILL Executed on EXOPI-908\EXO        | sent | 2026-11-26
  mailitem_id: 6 | dba@anywhere.example | [WARNING] Rapid TempDB Growth / Pressure on EXOPI-908\EXO | sent | 2026-11-26
  mailitem_id: 7 | dba@anywhere.example | [AUDIT] TempDB File Shrink Detected on EXOPI-908\EXO      | sent | 2026-11-26
  mailitem_id: 8 | dba@anywhere.example | [AUDIT] TempDB File Shrink Detected on EXOPI-908\EXO      | sent | 2026-11-26
  ```

---

### Part 12 — `AdventureWorks2022` Space Monitoring & `sys.sp_spaceused` (`2026-11-26`)
- **Action Taken:**
  1. Verified `AdventureWorks2022` already existed (`ONLINE`, `MULTI_USER`) so the conditional restore (`IF DB_ID(N'AdventureWorks2022') IS NULL`) safely skipped overwriting it.
  2. Executed the adjusted 2-row `sys.database_files` + `FILEPROPERTY(..., 'SpaceUsed')` query and `EXECUTE AdventureWorks2022.sys.sp_spaceused;`.
- **Observed Query Output Comparison:**
  ```text
  [Method 1 — 2-Row sys.database_files + FILEPROPERTY('SpaceUsed')]
  Data Files      | FileCount: 1 | Allocated: 200.00 MB | Used: 198.19 MB | Free:  1.81 MB
  Transaction Log | FileCount: 1 | Allocated:  72.00 MB | Used:  16.10 MB | Free: 55.90 MB

  [Method 2 — EXECUTE AdventureWorks2022.sys.sp_spaceused]
  Result Set 1: database_name: AdventureWorks2022 | database_size: 272.00 MB | unallocated space: 1.73 MB
  Result Set 2: reserved: 203032 KB | data: 97208 KB | index_size: 85840 KB | unused: 19984 KB
  ```

---

## Complete Backup & Restore Catalog (`msdb`)

### All Backups Recorded in `msdb.dbo.backupset` (`2026-11-11`)
| Database | Type | Copy-Only | Recovery Model | Backup Date | First LSN | Last LSN | Size (Bytes) | Physical Path |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `AdventureWorksLT2022` | `D` (Full) | `0` | `FULL` | `2026-11-11` | `63000000016000001` | `63000000018400001` | `8,482,816` | `C:\Exo\Database\AdventureWorksLT2022_Full.bak` |
| `AdventureWorksLT2022` | `D` (Full) | `1` | `FULL` | `2026-11-11` | `63000000026400001` | `63000000028800001` | `8,482,816` | `C:\Exo\Database\AdventureWorksLT2022_CopyOnly.bak` |
| `AdventureWorksLT2022` | `L` (Log) | `0` | `FULL` | `2026-11-11` | `63000000016000001` | `63000000028800001` | `86,016` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn` |
| `AdventureWorksLT2022` | `L` (Log) | `0` | `FULL` | `2026-11-11` | `63000000028800001` | `63000000036000001` | `86,016` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn` |
| `AdventureWorksLT2022` | `L` (Log) | `0` | `FULL` | `2026-11-11` | `63000000036000001` | `63000000047200001` | `86,016` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn` |
| `master` | `D` (Full) | `0` | `SIMPLE` | `2026-11-11` | `247000000040800001` | `247000000043200001` | `4,813,824` | `C:\Exo\Database\master_Full.bak` |
| `model` | `D` (Full) | `0` | `FULL` | `2026-11-11` | `43000000280800001` | `43000000283200001` | `3,431,424` | `C:\Exo\Database\model_Full.bak` |
| `msdb` | `D` (Full) | `0` | `SIMPLE` | `2026-11-11` | `194000000187200001` | `194000000189600001` | `18,989,056` | `C:\Exo\Database\msdb_Full.bak` |

### All Restores Recorded in `msdb.dbo.restorehistory` (`2026-11-11`, `2026-11-13`, & `2026-11-23`)
| ID | Restore Date | Destination DB | Type | Replace | `STOPAT` Used | Source Backup File |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `1` | `2026-11-11` | `AdventureWorksLT2022` | `D` | `0` | `No` | `C:\Exo\Database\AdventureWorksLT2022.bak` |
| `2` | `2026-11-11` | `AdventureWorks2022` | `D` | `0` | `No` | `C:\Exo\Database\AdventureWorks2022.bak` |
| `4` | `2026-11-11` | `AdventureWorksLT2022` | `D` | `1` | `No` | `C:\Exo\Database\AdventureWorksLT2022_Full.bak` |
| `5` | `2026-11-11` | `AdventureWorksLT2022` | `L` | `0` | `No` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn` |
| `6` | `2026-11-11` | `AdventureWorksLT2022` | `L` | `0` | `No` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn` |
| `7` | `2026-11-11` | `AdventureWorksLT2022` | `L` | `0` | `Yes` | `C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn` |
| `9` | `2026-11-13` | `Corruption3` | `D` | `0` | `No` | `C:\Exo\Database\Corruption3.bak` |
| `10` | `2026-11-23` | `CorruptionChallenge1` | `D` | `0` | `No` | `C:\Exo\Database\CorruptionChallenge1.bak` |

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
- **Live Evidence (`ERRORLOG.1` on `2026-11-11`):**
  During the `2026-11-11` session on `EXOPI-908\EXO`, the instance was updated from `17.0.5035.5` (`RTM-CU10`, `KB5125471`) to `17.0.5045.3` (`RTM-CU10-GDR`, `KB5129299`), demonstrating the **CU+GDR** servicing model in action.

---

### 3. SQL Server Browser Service
#### Q: Describe the purpose of the SQL Server Browser Service.
- **Answer:** The **SQL Server Browser Service** runs as a Windows service listening on **UDP port 1434**. Its purpose is to help clients locate **Named Instances** (such as `EXOPI-908\EXO`). Because named instances use **dynamic TCP ports** assigned at startup by default, a client connecting only by instance name does not know which TCP port to use. The client sends a UDP request to port 1434 asking for the instance's port, and the Browser Service responds with the active TCP port (or named pipe) for that named instance.

#### Q: Can you connect to the local SQL instance without the Browser Service running? Please explain why / why not.
- **Answer:** **Yes, you can!** (In fact, throughout the lab on `EXOPI-908\EXO`, the `SQLBrowser` service was **Stopped / Disabled**, and SSMS connected without issue). There are three reasons why:
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
- **How it works:** `DBCC CHECKDB` creates an internal, hidden copy-on-write **Database Snapshot** for transactional consistency (`Internal database snapshot has split point LSN...` as seen in `ERRORLOG`) and executes three checks:
  1. `DBCC CHECKALLOC` (disk space allocation structures)
  2. `DBCC CHECKTABLE` (physical & logical integrity of all tables and indexes)
  3. `DBCC CHECKCATALOG` (system metadata consistency)
- **Recovery & Repair Hierarchy:**
  1. **Restore from a Clean Backup (Preferred — Zero Data Loss):** Use your Full + Transaction Log (`.trn`) backup chain (or Page-Level Restore `RESTORE DATABASE ... PAGE = 'file:page'`) to recover without losing data.
  2. **`REPAIR_REBUILD` (Safe — No Data Loss):** Rebuilds corrupted non-clustered indexes when the underlying base table (Clustered Index / Heap) is still intact.
  3. **`REPAIR_ALLOW_DATA_LOSS` (Last Resort Only):** Deallocates (deletes) unreadable 8 KB data pages and patches pointers around the gap so the database can come online, permanently deleting the rows that lived on those pages.

---

## Mistakes, Errors & Troubleshooting Log (`2026-11-13`)

Below is an honest record of every operator mistake, environment error, and intentional lab trap encountered during the `2026-11-13` session (Parts 4A, 4B, & 5.1/5.2), along with the exact log evidence and how each was resolved:

### 1. SSMS Desktop Copy — Side-by-Side Configuration Error (`2026-11-13`)
- **What Happened:** The standalone `SSMS.exe` binary (`900,456` bytes) was copied directly onto `C:\Users\Gustav Fausing\OneDrive - ExOpi\Desktop\SSMS.exe` instead of creating a `.lnk` shortcut to the installation directory. Launching it failed with *"the application has failed to start because its side by side configuration is incorrect"*.
- **Event Log Evidence (`Application` / `SideBySide`, Event ID `33`):**
  ```text
  Activation context generation failed for "C:\Users\Gustav Fausing\OneDrive - ExOpi\Desktop\SSMS.exe". Dependent Assembly Automation,processorArchitecture="amd64",type="win32",version="1.0.0.0" could not be found.
  ```
- **Resolution:** Removed the copied `SSMS.exe` from the Desktop and created `SSMS.lnk` pointing to `C:\Program Files\Microsoft SQL Server Management Studio 22\Release\Common7\IDE\SSMS.exe` with the matching `WorkingDirectory`. *(Also encountered SSMS's normal **AutoRecover** prompt on launch from the earlier unexpected shutdowns.)*

---

### 2. PowerShell Terminal Line-Wrap Copy/Paste Errors (`2026-11-13`)
- **What Happened:** Pasting multi-line or long PowerShell commands into a narrow terminal window wrapped lines and inserted literal newlines mid-command:
  - Shortcut script: `ParserError: Unexpected token '$WshShell'`, `OperationStopped: Value does not fall within the expected range`, and `Get-ChildItem: Missing an argument for parameter 'Filter'`.
  - Log file rename: `Rename-Item: Cannot bind argument to parameter 'NewName' because it is an empty string`.
- **Resolution:** Used short single-line commands or executed the file/shortcut operations directly without line wrapping.

---

### 3. `Corruption4_log.ldf` Missing `NT SERVICE\MSSQL$EXO:(F)` Permission (`2026-11-13`)
- **What Happened:**
  1. When fixing NTFS permissions in Step 4B.1, `NT SERVICE\MSSQL$EXO:(F)` was granted on `C:\Exo\Database\Corruption4.mdf`, **but missed on `C:\Exo\Database\Corruption4_log.ldf`**.
  2. Later in Step 4B.4, when `DBCC CHECKDB (N'Corruption4', REPAIR_ALLOW_DATA_LOSS)` ran in `EMERGENCY` mode, SQL Server attempted to overwrite/recreate `C:\Exo\Database\Corruption4_log.ldf` (the path stored inside `Corruption4.mdf`), which failed with `Access is denied` and aborted the repair (`Error: 7909`), leaving `Corruption4` in `EMERGENCY` mode with `1` consistency error still unfixed.
- **SQL Server Error Log (`ERRORLOG`) Evidence:**
  ```text
  2026-11-13 spid71 Error: 5123, Severity: 16, State: 1.
  2026-11-13 spid71 CREATE FILE encountered operating system error 5(Access is denied.) while attempting to open or create the physical file 'C:\Exo\Database\Corruption4_log.ldf'.
  2026-11-13 spid71 Error: 7909, Severity: 20, State: 1.
  2026-11-13 spid71 The emergency-mode repair failed.You must restore from backup.
  2026-11-13 spid71 EMERGENCY MODE DBCC CHECKDB (Corruption4, repair_allow_data_loss) WITH all_errormsgs, no_infomsgs executed by EXOPI-908\Gustav Fausing terminated abnormally due to error state 4.
  ```
- **Resolution:** Renamed `C:\Exo\Database\Corruption4_log.ldf` to `Corruption4_log.ldf.corrupt` and re-ran `SET SINGLE_USER` $\rightarrow$ `DBCC CHECKDB (N'Corruption4', REPAIR_ALLOW_DATA_LOSS)` $\rightarrow$ `SET MULTI_USER`, allowing SQL Server to create a clean `Corruption4_log.ldf` and repair `sys.sysiscols`.

---

### 4. `Corruption4` Intentional Lab Errors & Batch Output Confusion (`2026-11-13`)
- **Expected Lab Errors Observed in `ERRORLOG`:**
  - `FOR ATTACH`: `Error: 824` — incorrect checksum (`expected: 0x0b407bbf; actual: 0xf4bf8440`) at page `(0:0)` in `C:\Exo\Database\Corruption4_log.ldf`.
  - `FOR ATTACH_REBUILD_LOG`: `Error: 824` at page `(2:0)` + `Error: 5105` (*A file activation error occurred*).
  - Hack-Attach `SET ONLINE`: `Error: 17207` (*OS error 2: The system cannot find the file specified*) for `C:\Exo\Database\Corruption4_rebuilt_log.ldf`.
- **Single-Batch Messages Confusion:** Running `SINGLE_USER`, `REPAIR_ALLOW_DATA_LOSS`, `MULTI_USER`, and the final `DBCC CHECKDB` together in one SSMS execution caused the red diagnostic lines emitted *by* `REPAIR_ALLOW_DATA_LOSS` (`Msg 259`, `Msg 8945`, `Msg 8951`/`8955`) to appear in the same Messages output above the clean `0`-error check, making it look like the repair might have failed even though it succeeded completely.

---

### 5. Papercut Windows Firewall Rules Defaulted to `Public` Profile (`2026-11-13`)
- **What Happened:** Because the active Wi-Fi connection (`DTU SciencePark 2`) is classified as `NetworkCategory = Public`, accepting the Windows Defender Firewall prompt created **two inbound `Allow` rules (`TCP` and `UDP`) under `Profile : Public`** for `papercut.service.exe`, violating the requirement to allow **Private networks only**.
- **Resolution:** Switched both inbound rules for `papercut.service.exe` to **`Profile : Private`** only (leaving `Public` and `Domain` disabled).

---

## Troubleshooting Log (`2026-11-23`)

### 1. Transient Database Mail Delivery Failure Before Retry (`2026-11-23`)
- **What Happened:** When `CheckDB` queued `mailitem_id = 2`, the initial SMTP attempt logged an error in `msdb.dbo.sysmail_event_log` (`The mail could not be sent to the recipients because of the mail server failure`).
- **Resolution:** Database Mail's built-in retry queue automatically retried and delivered `mailitem_id = 2` cleanly (`sent` on `2026-11-23`).

### 2. Lingering SSMS Connection Attempts to Dropped `CorruptTestDB` (`2026-11-23`)
- **What Happened:** After `CorruptTestDB` was dropped following the `CheckDB` test, an open SSMS tab / IntelliSense session continued polling `CorruptTestDB`, recording `Error: 18456, Severity: 14, State: 38` (`Failed to open the explicitly specified database 'CorruptTestDB'`) in `ERRORLOG`.
- **Resolution:** Harmless client-context artifact; switching the SSMS query tab database context back to `master` stops the log entries.

### 3. Step 8.3 Partial-Highlight Execution (`2026-11-23`)
- **What Happened:** During the initial Step 8.3 run, only the `INSERT` statement (`ProductID = 1000`, `TT-BIKE-001`, `Black`, `999.00`) and `WAITFOR DELAY` (`278`-character batch in plan cache) ran, leaving `SalesLT.ProductHistory` empty (`0` rows) and `TT-BIKE-001` in `SalesLT.Product`.
- **Resolution:** Looked up the existing `ProductID = 1000` and ran `UPDATE`, `DELETE`, and `FOR SYSTEM_TIME` queries in a single contiguous execution, archiving both historical versions cleanly in `SalesLT.ProductHistory`.

---

## Troubleshooting Log (`2026-11-26`)

### 1. `Alert_ProcessKilled` Failed with `Error 1934` (`QUOTED_IDENTIFIER OFF`) & `KILL 72` Target Mismatch (`2026-11-26`)
- **What Happened:**
  1. SQL Server Agent T-SQL job steps default to `SET QUOTED_IDENTIFIER OFF`, causing `SELECT ... INTO #KillEvents` (which calls XML `.value()` methods) to fail with **`Error 1934`**.
  2. During the initial test, `KILL 72;` was executed instead of the active Tab 2 `WAITFOR` session (`SPID 85`), so `KILL 72;` failed with `Msg 6106` and did not emit `sql_statement_completed`.
- **Resolution:** Prepended `SET QUOTED_IDENTIFIER ON;` to `Alert_ProcessKilled` (and added it to `Alert_TempDB_RapidGrowth` and `Alert_TempDB_Shrink` in `README.md`), opened a fresh Tab 2 (`SPID 51`), executed `KILL 51;`, and verified `Alert_ProcessKilled` succeeded and delivered `mailitem_id = 5`.

### 2. SSMS Stale IntelliSense Cache on Newly Created `msdb.dbo.DBA_TempDB_FileBaseline` (`2026-11-26`)
- **What Happened:** Pasting Step 11.4 Test #2 right after creating `msdb.dbo.DBA_TempDB_FileBaseline` triggered SSMS IntelliSense warnings (`Invalid object name 'msdb.dbo.DBA_TempDB_FileBaseline'`, `Invalid column name 'LogicalName'`).
- **Resolution:** Verified the table and column existed on `EXOPI-908\EXO`; refreshing SSMS IntelliSense (`Ctrl + Shift + R`) clears the stale cache, and Test #2/#3 executed cleanly (`mailitem_id = 7`).

### 3. Pinned `TempDB` Worktable Pages During Final `8.00 MB` Reset (`2026-11-26`)
- **What Happened:** After `#RapidGrowthTest` grew `temp2`–`temp8` to `72.00 MB`, running `DBCC SHRINKFILE (..., 8)` left `temp3` (`63.44 MB`), `temp4` (`63.56 MB`), and `temp5` (`64.25 MB`) partially unshrunk with `DBCC SHRINKFILE: Page 4:8119 could not be moved because it is a work table page.`
- **Resolution:** Executed `DBCC FREEPROCCACHE;` followed by `DBCC FREESYSTEMCACHE('ALL');` inside `tempdb` and re-ran `DBCC SHRINKFILE (..., 8)`, returning all 9 `tempdb` files (`tempdev`, `templog`, `temp2`–`temp8`) to `8.00 MB`.
