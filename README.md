# Database Opgave — Step-by-Step Guide & Learnings

This guide walks through every task from your database assignment emails in chronological order. Because the assignment states **"Use T-SQL in solutions unless otherwise stated"** and **"Some of the tasks will intentionally give you errors"**, each step below includes:
1. **How to do it** (T-SQL scripts and/or GUI steps)
2. **What it achieves**
3. **Key Learnings & Intentional Errors (Traps)** to watch out for

---

## Phase 0: Installation (Already Completed)
- [x] Install SQL Server 2025
- [x] Install Cumulative Updates (CUs) for SQL Server 2025
- [x] Install SQL Server Management Studio (SSMS)

---

## Part 1: `SQL.BACKUP.RESTORE.1`

### Step 1.1 — Create Windows Directories
**Task:** Create `C:\Exo` and `C:\Exo\Database` in Windows.

#### How to do it
In PowerShell / Command Prompt (or File Explorer):
```powershell
New-Item -ItemType Directory -Path "C:\Exo\Database" -Force
```
Or inside T-SQL (using the undocumented extended stored procedure `xp_create_subdir`):
```sql
EXEC master.dbo.xp_create_subdir N'C:\Exo\Database';
```

#### What it achieves
Creates a dedicated folder structure on the OS filesystem where SQL Server will store database data files (`.mdf`), transaction log files (`.ldf`), and backup files (`.bak`, `.trn`).

#### Key Learnings
- The Windows service account running SQL Server (`NT Service\MSSQLSERVER` or `NT Service\SQLSERVERAGENT`) must have **Read/Write NTFS permissions** on `C:\Exo\Database`. Creating the folder via `xp_create_subdir` guarantees that the SQL Server service account already has permission to write there.

---

### Step 1.2 — Change Default SQL Server Backup, Data, and Log Locations
**Task:** In SSMS, change the default SQL Server backup, data, and log location to `C:\Exo\Database`.

#### How to do it
**Option A: Using SSMS (as requested)**
1. In Object Explorer, **right-click the SQL Server instance** (top node) and select **Properties**.
2. Go to the **Database Settings** page on the left menu.
3. Under **Database default locations**, change all three paths (**Data**, **Log**, and **Backup**) to:
   `C:\Exo\Database`
4. Click **OK**.
5. **Restart the SQL Server service** (Right-click the server instance -> **Restart**).

**Option B: Using T-SQL (what SSMS runs under the hood)**
```sql
USE [master];
GO
EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', N'Software\Microsoft\MSSQLServer\MSSQLServer', N'DefaultData', REG_SZ, N'C:\Exo\Database';
EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', N'Software\Microsoft\MSSQLServer\MSSQLServer', N'DefaultLog', REG_SZ, N'C:\Exo\Database';
EXEC xp_instance_regwrite N'HKEY_LOCAL_MACHINE', N'Software\Microsoft\MSSQLServer\MSSQLServer', N'BackupDirectory', REG_SZ, N'C:\Exo\Database';
GO
```

#### What it achieves
Configures the instance so any new `CREATE DATABASE` or `BACKUP DATABASE` statement that does not explicitly specify a file path will automatically place its `.mdf`, `.ldf`, and `.bak` files in `C:\Exo\Database`.

#### Key Learnings & Intentional Traps
- **Trap:** Changing the default **Data** and **Log** paths in SSMS writes to the Windows Registry, but SQL Server **only reads those registry keys on startup**. If you do not **restart the SQL Server service** before Step 1.3, your new dummy database files will still be created in the old `C:\Program Files\Microsoft SQL Server\...\DATA` folder!
- *(Best Practice Note)*: In production, you would put Data (`.mdf`), Log (`.ldf`), and Backups (`.bak`) on **separate physical drives** so a single disk failure doesn't destroy both the live database and its backups, and so sequential log writes don't compete with random data I/O. Putting them in one folder here is strictly for lab simplicity.

---

### Step 1.3 — Create, Verify, Delete, and "Recover" a Dummy Database
**Task:**
1. Create a new dummy user database.
2. Verify physical files in `C:\Exo\Database`.
3. Verify in SSMS Databases.
4. Delete the database.
5. Verify in SSMS Databases.
6. **"Now. Get the dummy database back."**

#### How to do it
```sql
USE [master];
GO

-- 1. Create the dummy database (uses the new default paths automatically)
CREATE DATABASE [DummyDB];
GO

-- 2 & 3. Verify database and its physical file locations via T-SQL
SELECT name AS LogicalName, physical_name AS PhysicalLocation, type_desc AS FileType
FROM sys.master_files
WHERE database_id = DB_ID(N'DummyDB');
GO

-- 4. Delete (drop) the database
DROP DATABASE [DummyDB];
GO

-- 5. Verify it is gone
SELECT name FROM sys.databases WHERE name = N'DummyDB';
GO
```

#### What it achieves
Creates `DummyDB.mdf` (Primary Data File) and `DummyDB_log.ldf` (Transaction Log File) inside `C:\Exo\Database`, and then permanently drops the database.

#### Key Learnings & Intentional Error
- **INTENTIONAL TRAP ("Now. Get the dummy database back."):**
  **You cannot get it back!**
  - When you execute `DROP DATABASE`, SQL Server removes the database metadata from `master` **and immediately deletes the physical `.mdf` and `.ldf` files from Windows disk** (they do *not* go to the Windows Recycle Bin).
  - Because you **never took a backup** of `DummyDB` before dropping it, there is nothing to restore from.
  - **The Lesson:** A database without a backup cannot be recovered if dropped. Always take a full backup before making destructive changes.

---

### Step 1.4 — Download and Restore `AdventureWorksLT2022` and `AdventureWorks2022`
**Task:** Download and restore two databases from official Microsoft locations: `AdventureWorksLT2022` and `AdventureWorks2022`.

#### How to do it
1. **Download the `.bak` files** from Microsoft's official GitHub release page into `C:\Exo\Database`:
   - `AdventureWorks2022.bak`: `https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorks2022.bak`
   - `AdventureWorksLT2022.bak`: `https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorksLT2022.bak`

   *(Quick PowerShell download command)*:
   ```powershell
   Invoke-WebRequest -Uri "https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorks2022.bak" -OutFile "C:\Exo\Database\AdventureWorks2022.bak"
   Invoke-WebRequest -Uri "https://github.com/Microsoft/sql-server-samples/releases/download/adventureworks/AdventureWorksLT2022.bak" -OutFile "C:\Exo\Database\AdventureWorksLT2022.bak"
   ```

2. **Inspect the logical file names inside the backups first** (because the `.bak` files were created on a Microsoft server with an older directory structure like `C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\...`):
   ```sql
   RESTORE FILELISTONLY FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022.bak';
   RESTORE FILELISTONLY FROM DISK = N'C:\Exo\Database\AdventureWorks2022.bak';
   ```

3. **Restore both databases using `WITH MOVE`** so their `.mdf` and `.ldf` files are placed in `C:\Exo\Database`:
   ```sql
   USE [master];
   GO

   -- Restore AdventureWorksLT2022
   RESTORE DATABASE [AdventureWorksLT2022]
   FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022.bak'
   WITH MOVE N'AdventureWorksLT2022_Data' TO N'C:\Exo\Database\AdventureWorksLT2022.mdf',
        MOVE N'AdventureWorksLT2022_Log'  TO N'C:\Exo\Database\AdventureWorksLT2022_log.ldf',
        RECOVERY, STATS = 10;
   GO

   -- Restore AdventureWorks2022
   RESTORE DATABASE [AdventureWorks2022]
   FROM DISK = N'C:\Exo\Database\AdventureWorks2022.bak'
   WITH MOVE N'AdventureWorks2022'     TO N'C:\Exo\Database\AdventureWorks2022.mdf',
        MOVE N'AdventureWorks2022_log' TO N'C:\Exo\Database\AdventureWorks2022_log.ldf',
        RECOVERY, STATS = 10;
   GO
   ```

#### What it achieves
Brings both sample databases online on your SQL Server 2025 instance with their physical files cleanly stored in `C:\Exo\Database`.

#### Key Learnings & Intentional Error
- **Trap:** If you run `RESTORE DATABASE [AdventureWorks2022] FROM DISK = ...` *without* `WITH MOVE`, SQL Server tries to recreate the `.mdf` and `.ldf` files in the exact folder path where Microsoft originally backed them up (`MSSQL16.MSSQLSERVER`). On SQL Server 2025 (`MSSQL17`), that folder does not exist, causing **"Directory lookup for the file failed with the operating system error 3 (The system cannot find the path specified)."**
- Always run `RESTORE FILELISTONLY` first to find the `LogicalName` of the data and log files, then map them to your target directory using `WITH MOVE`.

---

### Step 1.5 — Backup `AdventureWorksLT2022` with `COPY_ONLY`
**Task:** Backup `AdventureWorksLT2022` with the `COPY_ONLY` switch.

#### How to do it
```sql
BACKUP DATABASE [AdventureWorksLT2022]
TO DISK = N'C:\Exo\Database\AdventureWorksLT2022_CopyOnly.bak'
WITH COPY_ONLY, INIT, STATS = 10;
GO
```

#### What it achieves
Creates an independent full backup file (`AdventureWorksLT2022_CopyOnly.bak`) without resetting the differential bitmap or affecting the transaction log backup chain.

#### Key Learnings
- Normally, a Full Backup resets the **differential base** (so the next Differential backup will look for this new Full backup).
- If you just want an ad-hoc backup (e.g., to restore onto a developer/test server) without breaking your scheduled production backup sequence, you must use `WITH COPY_ONLY`.
- Also crucially: **A `COPY_ONLY` full backup cannot act as the starting base for Transaction Log backups** when switching from `SIMPLE` to `FULL` recovery model!

---

### Step 1.6 — Change Recovery Model to `FULL` and Take a Standard Full Backup
**Task:**
1. Change `AdventureWorksLT2022` recovery model to `FULL`.
2. Backup `AdventureWorksLT2022` **without** the `COPY_ONLY` switch.

#### How to do it
```sql
USE [master];
GO

-- 1. Change Recovery Model from SIMPLE to FULL
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = N'AdventureWorksLT2022' AND state_desc = N'ONLINE') ALTER DATABASE [AdventureWorksLT2022]
SET RECOVERY FULL;
GO

-- 2. Take a standard Full Backup (establishes the log chain base!)
BACKUP DATABASE [AdventureWorksLT2022]
TO DISK = N'C:\Exo\Database\AdventureWorksLT2022_Full.bak'
WITH INIT, NAME = N'AdventureWorksLT2022-Full Database Backup', STATS = 10;
GO
```

#### What it achieves
Switches the database out of `SIMPLE` recovery (which auto-truncates the transaction log at every checkpoint and forbids log backups) into `FULL` recovery, and **starts the transaction log chain** by taking a non-`COPY_ONLY` Full backup.

#### Key Learnings & Intentional Error
- By default, `AdventureWorksLT2022` is in the **`SIMPLE` recovery model**.
- When you execute `ALTER DATABASE ... SET RECOVERY FULL`, the database operates in "**pseudo-simple**" mode until you take a regular (non-`COPY_ONLY`) Full backup.
- If you tried to take a Transaction Log backup *right after* changing to `FULL` (relying only on the `COPY_ONLY` backup from Step 1.5), SQL Server would fail with Error 4214:
  > *`BACKUP LOG cannot be performed because there is no current database backup.`*
- Taking the standard Full backup in Step 1.6 establishes the starting **LSN (Log Sequence Number)** for all subsequent `.trn` log backups.

---

### Step 1.7 — Create and Execute a SQL Agent Job for Transaction Log Backups
**Task:**
1. Create a SQL Agent job (no schedule) that backs up the transaction log.
2. Execute the job and verify physical `*.trn` file exists in `C:\Exo\Database`.

#### How to do it
*(Note: Ensure the **SQL Server Agent** service is running in SQL Server Configuration Manager or SSMS first!)*

We design the job step so that every time it runs, it appends a timestamp (`YYYYMMDD_HHMMSS`) to the `.trn` filename. That way, each run creates a distinct `.trn` file in `C:\Exo\Database`.

```sql
USE [msdb];
GO

-- 1. Create the SQL Agent Job (no schedule attached)
EXEC dbo.sp_add_job
    @job_name = N'Backup_Log_AdventureWorksLT2022',
    @enabled = 1,
    @description = N'Backs up the transaction log of AdventureWorksLT2022 to C:\Exo\Database with a timestamp.';

-- 2. Add the T-SQL Step to generate a timestamped .trn file
EXEC dbo.sp_add_jobstep
    @job_name = N'Backup_Log_AdventureWorksLT2022',
    @step_name = N'Run Transaction Log Backup',
    @subsystem = N'TSQL',
    @database_name = N'master',
    @command = N'
DECLARE @FileName NVARCHAR(500);
DECLARE @TimeStamp NVARCHAR(32) = FORMAT(GETDATE(), ''yyyyMMdd_HHmmss'');
SET @FileName = N''C:\Exo\Database\AdventureWorksLT2022_Log_'' + @TimeStamp + N''.trn'';

BACKUP LOG [AdventureWorksLT2022]
TO DISK = @FileName
WITH INIT, NAME = N''AdventureWorksLT2022 Transaction Log Backup'';
';

-- 3. Target the local SQL Server instance
EXEC dbo.sp_add_jobserver
    @job_name = N'Backup_Log_AdventureWorksLT2022',
    @server_name = N'(local)';
GO

-- 4. Execute the job manually
EXEC msdb.dbo.sp_start_job @job_name = N'Backup_Log_AdventureWorksLT2022';
GO
```

Verify the `.trn` file exists via T-SQL:
```sql
EXEC master.dbo.xp_dirtree N'C:\Exo\Database', 1, 1;
```

#### What it achieves
Creates a reusable on-demand SQL Agent job that backs up active transaction log records since the last backup into a `.trn` file and truncates (frees up for reuse) the inactive portion of the `.ldf` file.

#### Key Learnings & Intentional Error
- **Trap 1 (SQL Server Agent Stopped):** By default after a fresh SQL Server installation, the **SQL Server Agent** service is often **Stopped / Manual**. If you try to run `sp_start_job` while SQL Server Agent is stopped, it throws Error 22022: *"SQLServerAgent is not currently running so it cannot be notified of this action."*
- **Trap 2 (`sp_start_job` is Asynchronous):** `EXEC sp_start_job` returns immediately ("Job started successfully") while the backup runs in the background. Wait a few seconds and check **Job Activity Monitor** (or `msdb.dbo.sysjobhistory`) to confirm it succeeded.

---

### Step 1.8 — Create `SalesLT.TableA`, Backup Log, Wait 1 Minute, Create `SalesLT.TableB`, Backup Log
**Task:**
1. Create a new database object (`SalesLT.TableA`) in `AdventureWorksLT2022`.
2. Execute the transaction log backup job.
3. Wait one minute.
4. Create a new database object (`SalesLT.TableB`) in `AdventureWorksLT2022`.
5. Execute the transaction log backup job.

#### How to do it
```sql
USE [AdventureWorksLT2022];
GO

-- 1. Create SalesLT.TableA
CREATE TABLE SalesLT.TableA (
    ID INT IDENTITY(1,1) PRIMARY KEY,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    Note NVARCHAR(100)
);
INSERT INTO SalesLT.TableA (Note) VALUES (N'TableA created before STOPAT target');
GO

-- 2. Execute Transaction Log Backup Job (#1 after TableA)
EXEC msdb.dbo.sp_start_job @job_name = N'Backup_Log_AdventureWorksLT2022';
GO

-- Wait 5 seconds for the async job to finish, then record our target recovery timestamp!
WAITFOR DELAY '00:00:05';
SELECT CONVERT(VARCHAR(23), GETDATE(), 121) AS [Target_STOPAT_Timestamp_Copy_This];
-- >>> COPY THE TIMESTAMP PRINTED HERE! (e.g. '2026-11-11 14:30:15.000') <<<

-- 3. Wait one minute
WAITFOR DELAY '00:01:00';
GO

-- 4. Create SalesLT.TableB
CREATE TABLE SalesLT.TableB (
    ID INT IDENTITY(1,1) PRIMARY KEY,
    CreatedAt DATETIME2 DEFAULT SYSDATETIME(),
    Note NVARCHAR(100)
);
INSERT INTO SalesLT.TableB (Note) VALUES (N'TableB created AFTER STOPAT target');
GO

-- 5. Execute Transaction Log Backup Job (#2 after TableB)
EXEC msdb.dbo.sp_start_job @job_name = N'Backup_Log_AdventureWorksLT2022';
GO
```

#### What it achieves
Simulates a real-world timeline where a good change (`TableA`) is made at $T_1$, followed 1 minute later by an unwanted or accidental change (`TableB`) at $T_2$, with transaction log backups capturing the exact LSN and timestamp of every single transaction.

#### Key Learnings
- Because `AdventureWorksLT2022` is in the `FULL` recovery model, every `CREATE TABLE` and `INSERT` is fully logged in the transaction log with its exact commit timestamp.
- Waiting 1 minute creates a clean, unmistakable time window between `TableA` and `TableB` that we can target using `STOPAT`.

---

### Step 1.9 — Point-in-Time Restore (`STOPAT`) to Recover `TableA` Without `TableB`
**Task:** Restore the full backup and the `.trn` files including a `STOPAT` command with a timestamp. Overwrite existing database `AdventureWorksLT2022`. Goal is to bring `AdventureWorksLT2022` online so we only see `TableA` but not `TableB`.

#### How to do it
*(Replace the `.trn` filenames and `@StopAtTime` below with the actual filenames in `C:\Exo\Database` and the timestamp you recorded in Step 1.8)*:

```sql
USE [master];
GO

-- Kick out any active connections so we can overwrite the database
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = N'AdventureWorksLT2022' AND state_desc = N'ONLINE') ALTER DATABASE [AdventureWorksLT2022]
SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

-- 1. Restore the FULL backup WITH NORECOVERY and REPLACE
RESTORE DATABASE [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Full.bak'
WITH REPLACE, NORECOVERY, STATS = 10;
GO

-- 2. Restore the initial test .trn file from Step 1.7 WITH NORECOVERY
RESTORE LOG [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_102714.trn'
WITH NORECOVERY;
GO

-- 3. Restore the .trn file taken after TableA WITH NORECOVERY
RESTORE LOG [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103703.trn'
WITH NORECOVERY;
GO

-- 4. Restore the final .trn file (which contains TableB) WITH STOPAT and RECOVERY!
RESTORE LOG [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_103851.trn'
WITH STOPAT = N'2026-11-11 10:37:30.000', -- Timestamp between TableA and TableB
     RECOVERY;
GO

-- Bring database back to MULTI_USER mode
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = N'AdventureWorksLT2022' AND state_desc = N'ONLINE') ALTER DATABASE [AdventureWorksLT2022]
SET MULTI_USER;
GO

-- 5. Verify: TableA exists, TableB does NOT exist!
USE [AdventureWorksLT2022];
GO
SELECT name, create_date
FROM sys.tables
WHERE schema_id = SCHEMA_ID(N'SalesLT')
  AND name IN (N'TableA', N'TableB');
GO
```

#### What it achieves
Rolls the database back to the Full backup, replays the transaction log forward in chronological order, and halts replay at the exact second before `SalesLT.TableB` was created—bringing the database online with `TableA` intact and `TableB` gone.

#### Key Learnings & Intentional Errors
- **Trap 1 (Broken Log Chain — Skipping a `.trn` file):** You **must** restore *every* `.trn` file taken after the Full backup in strict chronological order (including the test `.trn` file you created in Step 1.7!). If you skip the first `.trn` file from Step 1.7 and try to apply the two `.trn` files from Step 1.8 directly onto the Full backup, SQL Server will throw Error 4305:
  > *`The log in this backup set begins at LSN ..., which is too recent to apply to the database. An earlier log backup that includes LSN ... can be restored.`*
- **Trap 2 (`WITH NORECOVERY` vs `WITH RECOVERY`):** Every restore step except the very last one **must** use `WITH NORECOVERY` (which leaves the database in the `Restoring...` state so more `.trn` files can be applied). If you accidentally use `WITH RECOVERY` (the default!) on the Full backup or an intermediate `.trn` file, SQL Server rolls back uncommitted transactions and brings the database online immediately—preventing you from applying the next `.trn` file!
- **Trap 3 (Database in Use / Tail-Log Error):** Restoring over an existing online database fails if your SSMS query tab is still connected to `AdventureWorksLT2022` (`Database is in use`) or if you omit `WITH REPLACE` (SQL Server complains that the tail of the log has not been backed up).

---

## Part 2: `SQL.BACKUP.RESTORE.2`

### Step 2.1 — Backup the Four System Databases
**Task:** Backup the four system databases (`master`, `model`, `msdb`, `tempdb`).

#### How to do it
```sql
USE [master];
GO

-- 1. Backup master
BACKUP DATABASE [master]
TO DISK = N'C:\Exo\Database\master_Full.bak'
WITH INIT, STATS = 10;
GO

-- 2. Backup model
BACKUP DATABASE [model]
TO DISK = N'C:\Exo\Database\model_Full.bak'
WITH INIT, STATS = 10;
GO

-- 3. Backup msdb
BACKUP DATABASE [msdb]
TO DISK = N'C:\Exo\Database\msdb_Full.bak'
WITH INIT, STATS = 10;
GO

-- 4. Attempt to backup tempdb (INTENTIONAL ERROR!)
BACKUP DATABASE [tempdb]
TO DISK = N'C:\Exo\Database\tempdb_Full.bak'
WITH INIT;
GO
```

#### What it achieves
Backs up `master` (instance-level logins, linked servers, system configuration, list of databases), `model` (template for all newly created databases), and `msdb` (SQL Server Agent jobs, schedules, backup/restore history).

#### Key Learnings & Intentional Error
- **INTENTIONAL ERROR (`tempdb`):**
  When you try to back up the 4th system database (`tempdb`), SQL Server intentionally fails with **Msg 3147, Level 16**:
  > *`Backup and restore operations are not allowed on database tempdb.`*
- **Why?** `tempdb` only holds temporary tables (`#temp`), table variables, version stores, and internal query worktables (spills/sorts). It is **destroyed and recreated from scratch (copied from `model`) every single time the SQL Server service starts**. Therefore, backing up or restoring `tempdb` makes no sense and is blocked by SQL Server.

---

### Step 2.2 — Create SQL Agent Job Script `"Point-in-time.restore"`
**Task:** Create a SQL Agent job script `"Point-in-time.restore"` that restores a full database backup and dynamically restores all `*.trn` files taken after the full backup. Use parameters for database name, file locations, etc.

#### How to do it
Below is a complete T-SQL script that creates the `"Point-in-time.restore"` SQL Agent job. Inside the job step:
- Variables (`@DatabaseName`, `@BackupFolder`, `@FullBackupFile`, `@StopAt`) parameterize the restore.
- It inspects the Full backup header (`RESTORE HEADERONLY`) to get its `BackupFinishDate` / `LastLSN`.
- It dynamically scans `@BackupFolder` for all `*.trn` files using `sys.dm_os_enumerate_filesystem`, reads each `.trn` header via `RESTORE HEADERONLY`, filters for log backups belonging to `@DatabaseName` taken after the Full backup, and loops through them in strict `FirstLSN` order—applying `WITH NORECOVERY`, and finally bringing the database online with `STOPAT = @StopAt, RECOVERY`.

```sql
USE [msdb];
GO

-- Drop the job if it already exists so this script is re-runnable
IF EXISTS (SELECT 1 FROM msdb.dbo.sysjobs WHERE name = N'Point-in-time.restore')
BEGIN
    EXEC msdb.dbo.sp_delete_job @job_name = N'Point-in-time.restore', @delete_unused_schedule = 1;
END
GO

-- 1. Create the SQL Agent Job
EXEC dbo.sp_add_job
    @job_name = N'Point-in-time.restore',
    @enabled = 1,
    @description = N'Dynamically restores a full backup and all subsequent *.trn files up to a specified @StopAt point in time.';

-- 2. Add the parameterized dynamic restore job step
EXEC dbo.sp_add_jobstep
    @job_name = N'Point-in-time.restore',
    @step_name = N'Dynamic Point-in-Time Restore',
    @subsystem = N'TSQL',
    @database_name = N'master',
    @command = N'
SET NOCOUNT ON;

-- ============================================================================
-- PARAMETERS (Change these as needed)
-- ============================================================================
DECLARE @DatabaseName   SYSNAME       = N''AdventureWorksLT2022'';
DECLARE @BackupFolder   NVARCHAR(260) = N''C:\Exo\Database'';
DECLARE @FullBackupFile NVARCHAR(500) = N''C:\Exo\Database\AdventureWorksLT2022_Full.bak'';
DECLARE @StopAt         DATETIME      = ''2026-11-11 10:37:30.000'';

-- ============================================================================
-- STEP 1: Read Full Backup Header to find its LastLSN
-- ============================================================================
CREATE TABLE #BackupHeader (
    BackupName NVARCHAR(128), BackupDescription NVARCHAR(255), BackupType SMALLINT,
    ExpirationDate DATETIME, Compressed TINYINT, Position SMALLINT, DeviceType TINYINT,
    UserName NVARCHAR(128), ServerName NVARCHAR(128), DatabaseName NVARCHAR(128),
    DatabaseVersion INT, DatabaseCreationDate DATETIME, BackupSize NUMERIC(20,0),
    FirstLSN NUMERIC(25,0), LastLSN NUMERIC(25,0), CheckpointLSN NUMERIC(25,0),
    DatabaseBackupLSN NUMERIC(25,0), BackupStartDate DATETIME, BackupFinishDate DATETIME,
    SortOrder SMALLINT, [CodePage] SMALLINT, UnicodeLocaleId INT, UnicodeComparisonStyle INT,
    CompatibilityLevel TINYINT, SoftwareVendorId INT, SoftwareVersionMajor INT,
    SoftwareVersionMinor INT, SoftwareVersionBuild INT, MachineName NVARCHAR(128),
    Flags INT, BindingID UNIQUEIDENTIFIER, RecoveryForkID UNIQUEIDENTIFIER,
    Collation NVARCHAR(128), FamilyGUID UNIQUEIDENTIFIER, HasBulkLoggedData BIT,
    IsSnapshot BIT, IsReadOnly BIT, IsSingleUser BIT, HasBackupChecksums BIT,
    IsDamaged BIT, BeginsLogChain BIT, HasIncompleteMetaData BIT, IsForceOffline BIT,
    IsCopyOnly BIT, FirstRecoveryForkID UNIQUEIDENTIFIER, ForkPointLSN NUMERIC(25,0),
    RecoveryModel NVARCHAR(60), DifferentialBaseLSN NUMERIC(25,0),
    DifferentialBaseGUID UNIQUEIDENTIFIER, BackupTypeDescription NVARCHAR(60),
    BackupSetGUID UNIQUEIDENTIFIER, CompressedBackupSize BIGINT, Containment TINYINT,
    KeyAlgorithm NVARCHAR(32), EncryptorThumbprint VARBINARY(20), EncryptorType NVARCHAR(32),
    LastValidRestoreTime DATETIME, TimeZone SMALLINT, CompressionAlgorithm NVARCHAR(32)
);

DECLARE @Sql NVARCHAR(MAX);
INSERT INTO #BackupHeader
EXEC (N''RESTORE HEADERONLY FROM DISK = N'''''' + @FullBackupFile + N'''''''');

DECLARE @FullBackupLastLSN NUMERIC(25,0);
SELECT TOP 1 @FullBackupLastLSN = LastLSN FROM #BackupHeader WHERE BackupType = 1;
TRUNCATE TABLE #BackupHeader;

-- ============================================================================
-- STEP 2: Kick users off and Restore the Full Backup WITH NORECOVERY
-- ============================================================================
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = @DatabaseName AND state_desc = N''ONLINE'')
BEGIN
    SET @Sql = N''ALTER DATABASE '' + QUOTENAME(@DatabaseName) + N'' SET SINGLE_USER WITH ROLLBACK IMMEDIATE;'';
    EXEC (@Sql);
END

SET @Sql = N''RESTORE DATABASE '' + QUOTENAME(@DatabaseName) +
           N'' FROM DISK = N'''''' + @FullBackupFile + N'''''''' +
           N'' WITH REPLACE, NORECOVERY;'';
PRINT @Sql;
EXEC (@Sql);

-- ============================================================================
-- STEP 3: Enumerate all *.trn files in @BackupFolder and inspect their LSNs
-- ============================================================================
CREATE TABLE #TrnFiles (
    FullPath NVARCHAR(500),
    FirstLSN NUMERIC(25,0),
    LastLSN  NUMERIC(25,0),
    BackupFinishDate DATETIME
);

DECLARE @CurrentTrn NVARCHAR(500);
DECLARE TrnCursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT full_filesystem_path
    FROM sys.dm_os_enumerate_filesystem(@BackupFolder, N''*.trn'');

OPEN TrnCursor;
FETCH NEXT FROM TrnCursor INTO @CurrentTrn;

WHILE @@FETCH_STATUS = 0
BEGIN
    TRUNCATE TABLE #BackupHeader;
    INSERT INTO #BackupHeader
    EXEC (N''RESTORE HEADERONLY FROM DISK = N'''''' + @CurrentTrn + N'''''''');

    INSERT INTO #TrnFiles (FullPath, FirstLSN, LastLSN, BackupFinishDate)
    SELECT @CurrentTrn, FirstLSN, LastLSN, BackupFinishDate
    FROM #BackupHeader
    WHERE BackupType = 2 -- Transaction Log Backup
      AND DatabaseName = @DatabaseName
      AND LastLSN >= @FullBackupLastLSN;

    FETCH NEXT FROM TrnCursor INTO @CurrentTrn;
END
CLOSE TrnCursor;
DEALLOCATE TrnCursor;

-- ============================================================================
-- STEP 4: Restore each valid *.trn file in strict LSN order with STOPAT
-- ============================================================================
DECLARE @StopAtStr NVARCHAR(30) = CONVERT(NVARCHAR(30), @StopAt, 121);

DECLARE RestoreCursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT FullPath
    FROM #TrnFiles
    ORDER BY FirstLSN ASC;

OPEN RestoreCursor;
FETCH NEXT FROM RestoreCursor INTO @CurrentTrn;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @Sql = N''RESTORE LOG '' + QUOTENAME(@DatabaseName) +
               N'' FROM DISK = N'''''' + @CurrentTrn + N'''''''' +
               N'' WITH NORECOVERY, STOPAT = N'''''' + @StopAtStr + N'''''''';'';
    PRINT @Sql;
    EXEC (@Sql);

    FETCH NEXT FROM RestoreCursor INTO @CurrentTrn;
END
CLOSE RestoreCursor;
DEALLOCATE RestoreCursor;

-- ============================================================================
-- STEP 5: Bring the database ONLINE (WITH RECOVERY) and set MULTI_USER
-- ============================================================================
SET @Sql = N''RESTORE DATABASE '' + QUOTENAME(@DatabaseName) + N'' WITH RECOVERY;'';
PRINT @Sql;
EXEC (@Sql);

SET @Sql = N''ALTER DATABASE '' + QUOTENAME(@DatabaseName) + N'' SET MULTI_USER;'';
EXEC (@Sql);

DROP TABLE #BackupHeader;
DROP TABLE #TrnFiles;
';

-- 3. Attach job to local server
EXEC dbo.sp_add_jobserver
    @job_name = N'Point-in-time.restore',
    @server_name = N'(local)';
GO
```

#### What it achieves
Automates point-in-time recovery so a DBA doesn't have to manually look up and type 20 different `.trn` filenames during an outage.

#### Key Learnings
- You can pass `WITH NORECOVERY, STOPAT = '...'` to **every** `RESTORE LOG` statement in a loop! If a `.trn` file was taken *before* the `@StopAt` time, SQL Server simply rolls through the entire `.trn` file and waits for the next one. Once it reaches the `.trn` file that spans `@StopAt`, it stops at that exact timestamp, and a final `RESTORE DATABASE [DB] WITH RECOVERY;` brings the database online cleanly.
- Sorting by `FirstLSN` from `RESTORE HEADERONLY` is safer than sorting by filename or OS file modification time because LSNs (Log Sequence Numbers) are how SQL Server internally orders the transaction log chain.

---

## Part 3: `SQL.MISC.1` — Theory & Troubleshooting Questions

### 1. How many SQL Server Instances can be active on the same server?
- **Answer:** Up to **50 instances** on a standalone server (1 Default instance + up to 49 Named instances, or 50 Named instances).
- *(Note: On a Windows Server Failover Cluster (WSFC) using shared Cluster Disks, the limit is **25 instances**, or up to 50 if using SMB file shares).*
- **Learning:** Each instance has its own independent system databases (`master`, `model`, `msdb`, `tempdb`), memory allocation (`max server memory`), and Windows service (`MSSQLSERVER` for the default instance, `MSSQL$<InstanceName>` for named instances).

---

### 2. SQL Server Patching
#### Q: Are Cumulative Updates (CUs) installed and managed per SQL instance?
- **Answer:** **Yes** (with shared components updated machine-wide). When you run a SQL Server Cumulative Update installer on a server with multiple instances, the setup wizard lets you choose which specific SQL Server instance(s) you want to patch, or patch all instances at once. This allows you to patch a Test/Dev instance on a box before patching another instance, though any *shared components* (like Client Tools, Integration Services, or SQL Writer) on that server are updated once for the whole machine.

#### Q: Are security updates included in Cumulative Updates (CUs)?
- **Answer:** **Yes.** Cumulative Updates are *cumulative*—each CU contains all bug fixes, performance improvements, **and all security updates (GDRs)** released since the initial RTM release of that SQL Server version.
- *(Note: Microsoft releases two servicing branches when a security vulnerability is found: a **GDR** branch containing *only* security fixes for conservative environments, and a **CU+GDR** / next **CU** branch containing both cumulative functional fixes and security updates).*

---

### 3. SQL Server Browser Service
#### Q: Describe the purpose of the SQL Server Browser Service.
- **Answer:** The **SQL Server Browser Service** runs as a Windows service listening on **UDP port 1434**. Its purpose is to help clients find **Named Instances** (e.g., `ServerName\SQLEXPRESS` or `ServerName\INST2`). Because named instances typically use **dynamic TCP ports** assigned at startup, a client connecting by instance name doesn't know which TCP port to connect to. The client sends a UDP packet to port 1434 asking *"What port is `INST2` using?"*, and the Browser Service replies with the exact TCP port (or named pipe) for that instance.

#### Q: Can you connect to the local SQL instance without the Browser Service running? Please explain why / why not.
- **Answer:** **Yes, you can!** There are three reasons why:
  1. **Shared Memory Protocol (Local Connections):** When connecting locally on the same machine (e.g., from SSMS installed on the SQL Server box), the client uses the **Shared Memory** network protocol (`lpc:`) by default, which communicates directly in RAM and does not use TCP/IP ports or the Browser Service at all.
  2. **Default Instance Uses Static Port 1433:** Even over TCP/IP, the **Default Instance** (`MSSQLSERVER`) listens on the well-known static **TCP port 1433**. Clients automatically try port 1433 without asking the Browser Service.
  3. **Explicit Port Number for Named Instances:** Even if you have a **Named Instance** over TCP/IP and the Browser Service is stopped, you can still connect if you configure a static port (or look up the current port in SQL Server Configuration Manager / Error Log) and specify the port number explicitly in your connection string using a comma: e.g., `localhost,50123`.

---

### 4. Authentication & Troubleshooting
#### Q: What is the difference between Windows Authentication and SQL Server Authentication?
| Feature | Windows Authentication (Integrated Security) | SQL Server Authentication |
| :--- | :--- | :--- |
| **Where credentials live** | Active Directory (Domain Controller) or Local Windows OS | Stored inside SQL Server's `master` database (`sys.sql_logins`) |
| **How identity is verified** | Windows verifies the user via **Kerberos** or **NTLM** token; SQL Server never sees the password | User sends username & password directly to SQL Server to hash and compare |
| **Security & Policies** | Enforces Active Directory policies (MFA, password rotation, complexity, account lockout, SSO) | Subject only to local Windows password policy if `CHECK_POLICY = ON`; requires Mixed Mode |
| **Best Use Case** | **Recommended default** for all domain-joined users and application service accounts | Legacy applications, non-domain clients (Linux/containers without Kerberos), or third-party tools |

#### Q: A client machine cannot connect to SQL Server. Everyone else is connecting without issues. Describe your troubleshooting approach.
Because **everyone else is connecting without issues**, we immediately know:
- The SQL Server service is **running** and online.
- TCP/IP is enabled on the server, and the server's Windows Firewall is allowing traffic on the SQL port (at least for other clients).

Therefore, the problem is isolated to **this specific client machine, its network path/subnet, or this user's credentials**. Follow this step-by-step troubleshooting approach:

1. **Get the exact error message from the client** (this splits the investigation in half):
   - *Error Type A: "A network-related or instance-specific error occurred (Error 26 / Error 53 / Error 10060)"* -> The client cannot reach SQL Server over the network.
   - *Error Type B: "Login failed for user '...' (Error 18456)"* -> The client reached SQL Server, but SQL Server rejected the login.
2. **If Network Error (Cannot reach server):**
   - **Check typo in Server/Instance Name:** Did the user type the wrong server name, instance name, or port?
   - **Check DNS & Basic Reachability:** Run `ping <ServerName>` and `nslookup <ServerName>` from the client machine. Does it resolve to the right IP address?
   - **Test the TCP Port from the Client:** Run PowerShell on the client:
     `Test-NetConnection -ComputerName <ServerName> -Port 1433`
     If this fails, check if the client is on a different VLAN/subnet (e.g., VPN or Guest Wi-Fi) blocked by a network firewall, or has an outbound rule on the client's local firewall.
   - **Check Client Aliases / Drivers:** Run `cliconfg.exe` (SQL Server Client Network Utility) or check ODBC Data Sources on the client machine to see if a broken SQL Alias or disabled protocol is overriding the connection.
3. **If Authentication Error (`Login failed for user`, Error 18456):**
   - **Check the SQL Server Error Log!** While the client only sees a generic *"Login failed for user 'X'"* message (for security reasons), the **SQL Server Error Log** records the exact **State number** (e.g., `Error: 18456, Severity: 14, State: 8`), which tells you the exact cause:
     - *State 2 / 5:* Login does not exist in SQL Server (User isn't in the AD group or lacks a SQL Login).
     - *State 7:* Login is disabled.
     - *State 8:* Incorrect password.
     - *State 11 / 12:* Valid login, but server access fails / lacks permission.
     - *State 16:* User's **Default Database** is offline, dropped, or the user lacks permission to access it.
   - **Check Active Directory:** Is the user's Windows account locked out or password expired?

#### Q: What is the difference between Authentication and Authorization?
- **Authentication (AuthN) — *"Who are you?"***
  - The process of verifying the **identity** of a user or service attempting to connect to the SQL Server instance.
  - Handled at the **Server Instance level** via a **Login** (`sys.server_principals`), using either Windows Authentication or SQL Server Authentication.
- **Authorization (AuthZ) — *"What are you allowed to do?"***
  - The process of determining which **databases, schemas, tables, views, or commands** an already-authenticated user has permission to access or execute.
  - Handled via **Database Users** (`sys.database_principals`), **Roles** (e.g., `sysadmin`, `db_datareader`, `db_datawriter`), and explicit **`GRANT` / `DENY` / `REVOKE`** permissions.
- **Simple Analogy:** **Authentication** is showing your passport at airport security to prove who you are; **Authorization** is your boarding pass determining which plane and which seat class (Economy vs. First Class) you are allowed to enter.

---

## Part 4: `SQL.MISC.2` (Next Week Preview) — Database Corruption, Integrity & `DBCC CHECKDB`

### 1. Physical vs. Logical Database Integrity
SQL Server stores data in **8 KB pages** organized into **extents** (8 contiguous pages = 64 KB). Corruption happens when hardware (disk controllers, memory bit-flips, storage SAN firmware, or unexpected power loss) damages these 8 KB pages.

| Integrity Type | What It Checks | Examples of Corruption |
| :--- | :--- | :--- |
| **Physical Integrity** | Verifies that the physical 8 KB pages on disk can be read and that their internal page headers, checksums, and allocation bitmaps are intact. | **Torn Page** (only part of an 8 KB page was written before power loss), **Checksum Failure** (page bits flipped on disk so the stored checksum no longer matches the calculated checksum), corrupt allocation maps (**GAM, SGAM, PFS, IAM**). |
| **Logical Integrity** | Verifies that the higher-level relational and index structures across pages make logical sense and match metadata rules. | A **Non-Clustered Index** points to a row in the Clustered Index that does not exist (or has different key values); broken forward/backward page pointers in a B-Tree index; metadata catalog inconsistencies (`sys.objects` vs actual allocations); computed column or data type violations. |

---

### 2. Understanding `DBCC CHECKDB`
`DBCC CHECKDB` (Database Console Command Check Database) is SQL Server's primary tool for detecting both **physical** and **logical** corruption across an entire database.

#### How `DBCC CHECKDB` Works Under the Hood
1. **Creates an Internal Hidden Database Snapshot:** Because a live database is constantly changing while `DBCC CHECKDB` reads it, `DBCC CHECKDB` automatically creates an internal, hidden copy-on-write **Database Snapshot** first so it can scan a transactionally consistent point-in-time view without blocking users.
2. **Runs Three Core Checks in One Command:**
   - `DBCC CHECKALLOC`: Checks disk space allocation structures (GAM, SGAM, PFS, IAM pages).
   - `DBCC CHECKTABLE`: Checks physical and logical integrity of every table and index (clustered, non-clustered, heap, LOB pages).
   - `DBCC CHECKCATALOG`: Checks metadata consistency across system catalog tables.

#### Essential `DBCC CHECKDB` Commands
```sql
-- 1. Standard full integrity check (Suppresses thousands of "OK" informational lines; shows only errors)
DBCC CHECKDB (N'AdventureWorks2022') WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO

-- 2. Lightweight Physical-Only check (Faster for very large databases on weekday nights; skips deep logical index checks)
DBCC CHECKDB (N'AdventureWorks2022') WITH PHYSICAL_ONLY, NO_INFOMSGS, ALL_ERRORMSGS;
GO

-- 3. Extended Logical Checks (Checks indexed views, XML indexes, spatial indexes - slower)
DBCC CHECKDB (N'AdventureWorks2022') WITH EXTENDED_LOGICAL_CHECKS, NO_INFOMSGS, ALL_ERRORMSGS;
GO
```

#### How to Recover from Corruption (Repair vs. Restore)
When `DBCC CHECKDB` finds corruption, its output will state the **minimum repair level** required:

1. **Always Prefer Restoring from a Clean Backup (or Page-Level Restore)!**
   - If you have a valid Full backup and unbroken Transaction Log (`.trn`) chain (like you practiced in `SQL.BACKUP.RESTORE.1`!), you can restore the database—or even restore just the single damaged 8 KB page (`RESTORE DATABASE ... PAGE = '1:245'`)—with **zero data loss**.
2. **`REPAIR_REBUILD` (Safe — No Data Loss):**
   - Used when corruption only exists in a **Non-Clustered Index** (e.g., logical index mismatch) while the base table (Clustered Index / Heap) is still 100% intact. SQL Server simply rebuilds the damaged index from the intact base data.
   *(Tip: Often you can just `ALTER INDEX ... REBUILD` or `DROP` and `CREATE` the non-clustered index online without putting the database in `SINGLE_USER` mode!)*
3. **`REPAIR_ALLOW_DATA_LOSS` (Last Resort Only!):**
   - Used when a **Clustered Index or Heap data page** is physically destroyed and **you have no valid backup**.
   - What it actually does: It **deletes (deallocates) the corrupted 8 KB page(s)** and fixes the page pointers around the missing hole so the database can function again. **Any rows on that page are permanently deleted**, and foreign key constraints may be left broken (which is why you must run `DBCC CHECKCONSTRAINTS` afterward).
   - Requires the database to be in `SINGLE_USER` mode:
     ```sql
     ALTER DATABASE [CorruptDB] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
     DBCC CHECKDB (N'CorruptDB', REPAIR_ALLOW_DATA_LOSS) WITH NO_INFOMSGS, ALL_ERRORMSGS;
     ALTER DATABASE [CorruptDB] SET MULTI_USER;
     ```

---

## Part 4A: `2020.Q.RESTORE.CHECKDB.Database.Integrity.Corruption3`

### Step 4A.1 — Copy and Restore `Corruption3.bak`
**Task:** Restore `Corruption3` from `Corruption3.bak` if it doesn't exist.

#### How to do it
1. **Copy `Corruption3.bak` to `C:\Exo\Database`** (so the SQL Server service account `NT Service\MSSQL$EXO` can read it):
   ```powershell
   Copy-Item "C:\Users\Gustav Fausing\Database-Opgave\Corruption3.bak" "C:\Exo\Database\Corruption3.bak"
   ```

2. **Inspect the logical file names inside `Corruption3.bak` first** (`RESTORE FILELISTONLY`):
   ```sql
   USE [master];
   GO

   RESTORE FILELISTONLY
   FROM DISK = N'C:\Exo\Database\Corruption3.bak';
   GO
   ```
   *(Notice the logical names inside the backup are **`SalesDBData`** and **`SalesDBLog`**, NOT `Corruption3`!)*

3. **Restore `Corruption3` using `WITH MOVE`:**
   ```sql
   IF DB_ID(N'Corruption3') IS NULL
   BEGIN
       RESTORE DATABASE [Corruption3]
       FROM DISK = N'C:\Exo\Database\Corruption3.bak'
       WITH MOVE N'SalesDBData' TO N'C:\Exo\Database\Corruption3.mdf',
            MOVE N'SalesDBLog'  TO N'C:\Exo\Database\Corruption3_log.ldf',
            RECOVERY, STATS = 10;
   END
   GO
   ```

#### What it achieves
Restores the `Corruption3` database onto your SQL Server instance (`EXOPI-908\EXO`) with its `.mdf` and `.ldf` files stored cleanly in `C:\Exo\Database`.

#### Key Learnings & Intentional Traps
- **Trap 1 (NTFS Permissions on User Profile Folder):** If you run `RESTORE FILELISTONLY` or `RESTORE DATABASE` directly against `C:\Users\Gustav Fausing\Database-Opgave\Corruption3.bak`, SQL Server fails with **`Msg 3201: Operating system error 5(Access is denied.)`** because `NT Service\MSSQL$EXO` does not have NTFS read permission inside your personal user profile folder. Always move/copy `.bak` files into `C:\Exo\Database` first.
- **Trap 2 (Unexpected Logical File Names):** Guessing `MOVE N'Corruption3'` fails because the backup's internal logical names are `SalesDBData` and `SalesDBLog`. Always run `RESTORE FILELISTONLY` first!

---

### Step 4A.2 — Run `DBCC CHECKDB` on `Corruption3`
**Task:** Run `DBCC CHECKDB (Corruption3) WITH NO_INFOMSGS, ALL_ERRORMSGS;` in SSMS and inspect the consistency errors.

#### How to do it
```sql
DBCC CHECKDB (N'Corruption3')
WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO
```

**Observed Output Summary:**
```text
Msg 8951, Level 16, State 1, ...
Table error: table 'Customers' (ID 453576654). Data row does not have a matching index row in the index 'CustomerName' (ID 2). Possible missing or invalid keys for the index row matching:
Msg 8955, Level 16, State 1, ...
Data row (1:45:28) identified by (CustomerID = 29) with index values 'LastName = 'Adams' and CustomerID = 29'.
...
Msg 8952, Level 16, State 1, ...
Table error: table 'Customers' (ID 453576654). Index row in index 'CustomerName' (ID 2) does not match any data row. Possible extra or invalid keys for:
Msg 8956, Level 16, State 1, ...
Index row (1:24480:20) with values (LastName = 'Adams' and CustomerID = 2078) pointing to the data row identified by (CustomerID = 2078).
...
CHECKDB found 0 allocation errors and 26 consistency errors in table 'Customers' (object ID 453576654).
CHECKDB found 0 allocation errors and 26 consistency errors in database 'Corruption3'.
repair_rebuild is the minimum repair level for the errors found by DBCC CHECKDB (Corruption3).
```

#### What it achieves
Identifies the exact object, index, error count, and minimum safe repair level required.

---

### Step 4A.3 — Assignment Questions & Solutions (`Corruption3`)

1. **Where does SQL Server save the output produced by `CHECKDB`?**
   - **Primary / Detailed Output:** Returned directly to the **calling client session** (the SSMS **Messages** tab, `sqlcmd` stdout, or a SQL Agent job step's output file if configured). SQL Server does **not** automatically save the full `DBCC CHECKDB` row-by-row output into a permanent system table.
   - **Summary Log:** SQL Server writes a **summary entry** (e.g., `DBCC CHECKDB (Corruption3) executed by ... found 26 errors and repaired 0 errors`) to the **SQL Server Error Log (`ERRORLOG`)** and the **Windows Application Event Log**.
   - *(Tip)*: If you want to persist detailed results in SQL Server, run `DBCC CHECKDB (N'Corruption3') WITH TABLERESULTS, NO_INFOMSGS, ALL_ERRORMSGS;` and `INSERT` the result set into a custom audit table.

2. **Is the `Corruption3` database usable?**
   - **Yes, technically it is `ONLINE` and accessible**, because the corruption is isolated inside a single **non-clustered index (`CustomerName`)** on `dbo.Customers`, while the base data pages (`CustomerPK` clustered index) and allocation maps are intact.
   - Queries that do not touch `dbo.Customers`—or queries that read `dbo.Customers` via the clustered index `CustomerPK`—will succeed normally. However, queries that use the `CustomerName` non-clustered index may fail or silently return wrong results.

3. **If you query the database, can you trust your query results?**
   - **No!** Because `CustomerName` has **26 out-of-sync index rows**:
     - Queries filtering or sorting by `LastName` that choose an Index Seek/Scan on `CustomerName` can **miss valid rows** (`Msg 8951` — row exists in the table, missing from index) or **return phantom/wrong matches** (`Msg 8952` — index entry points to a non-matching row).
     - Depending on which execution plan the Query Optimizer picks (Clustered Index Scan vs. Non-Clustered Index Seek), the *same logical query* can return **different row counts and different data**. Therefore, results **cannot be trusted** until repaired.

4. **What is wrong with the database?**
   - There are **0 allocation errors** and **26 logical consistency errors** in table **`dbo.Customers`** (`object ID 453576654`), specifically inside the **non-clustered index `CustomerName` (`index ID 2`)** on column `LastName` (which includes clustered key `CustomerID`):
     - **Missing index rows (`Msg 8951 / 8955`):** Data rows exist in the clustered index `CustomerPK` that are missing from `CustomerName`.
     - **Extra/invalid index rows (`Msg 8952 / 8956`):** Entries exist in `CustomerName` that do not match any valid row in `CustomerPK`.
   - Because the underlying table data (`CustomerPK`) is healthy, `DBCC CHECKDB` reports **`repair_rebuild is the minimum repair level`** (meaning **0 data loss**).

5. **T-SQL to solve the consistency errors (and verify 0 errors):**

   First, inspect the indexes on `dbo.Customers`:
   ```sql
   USE [Corruption3];
   GO
   EXEC sp_helpindex N'dbo.Customers';
   GO
   ```
   *(Shows `CustomerPK` [clustered, unique, PK on `CustomerID`] and `CustomerName` [nonclustered on `LastName`].)*

   **Option A (Recommended — Online / No `SINGLE_USER` Downtime): `DISABLE` then `REBUILD` the index**
   ```sql
   USE [Corruption3];
   GO

   -- 1. Disable the corrupted non-clustered index first (CRITICAL!)
   ALTER INDEX [CustomerName] ON [dbo].[Customers] DISABLE;
   GO

   -- 2. Rebuild it from the intact clustered index (CustomerPK)
   ALTER INDEX [CustomerName] ON [dbo].[Customers] REBUILD;
   GO

   -- 3. Verify 0 errors
   DBCC CHECKDB (N'Corruption3') WITH NO_INFOMSGS, ALL_ERRORMSGS;
   GO
   ```

   **Option B (Alternative Online Method): `DROP` and `CREATE` the non-clustered index**
   ```sql
   USE [Corruption3];
   GO

   DROP INDEX [CustomerName] ON [dbo].[Customers];
   GO

   CREATE NONCLUSTERED INDEX [CustomerName]
       ON [dbo].[Customers] ([LastName] ASC);
   GO

   DBCC CHECKDB (N'Corruption3') WITH NO_INFOMSGS, ALL_ERRORMSGS;
   GO
   ```

   **Option C (Using `DBCC CHECKDB ... REPAIR_REBUILD` — Requires `SINGLE_USER` downtime)**
   ```sql
   USE [master];
   GO

   ALTER DATABASE [Corruption3] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
   GO

   DBCC CHECKDB (N'Corruption3', REPAIR_REBUILD)
   WITH NO_INFOMSGS, ALL_ERRORMSGS;
   GO

   ALTER DATABASE [Corruption3] SET MULTI_USER;
   GO

   DBCC CHECKDB (N'Corruption3') WITH NO_INFOMSGS, ALL_ERRORMSGS;
   GO
   ```

#### Key Learnings & Intentional Traps
- **INTENTIONAL TRAP (`ALTER INDEX ... REBUILD` Alone Does NOT Fix It!):**
  If you run `ALTER INDEX [CustomerName] ON [dbo].[Customers] REBUILD;` *without* disabling or dropping it first, **SQL Server scans the existing corrupted non-clustered index itself** to build the new index pages—carrying all **26 consistency errors** right into the rebuilt index! You **must** run `DISABLE` + `REBUILD` (forcing SQL Server to scan the healthy `CustomerPK` clustered index), `DROP` + `CREATE`, or `REPAIR_REBUILD`.

---

## Part 4B: `Corruption4` — Power Failure Simulation (No Backup Files!)

**Context:** Power failure simulation. You are given only `Corruption4.mdf` and `Corruption4_log.ldf` and have **no database backup files (`.bak`)**. Your goal is to bring `Corruption4` back `ONLINE` so `DBCC CHECKDB` runs with **0 errors**.

---

### Step 4B.1 — Copy Files to `C:\Exo\Database` and Fix NTFS Permissions
**Task:** Place `Corruption4.mdf` and `Corruption4_log.ldf` in `C:\Exo\Database` with proper NTFS permissions so SQL Server can attach them.

#### How to do it
In PowerShell (keep a clean copy of the original files in `Database-Opgave` in case you want to practice again):
```powershell
Copy-Item "C:\Users\Gustav Fausing\Database-Opgave\Corruption4.mdf" "C:\Exo\Database\Corruption4.mdf"
Copy-Item "C:\Users\Gustav Fausing\Database-Opgave\Corruption4_log.ldf" "C:\Exo\Database\Corruption4_log.ldf"

# Remove Mark-of-the-Web (if downloaded) and grant Full Control to SQL Server + your user
Unblock-File "C:\Exo\Database\Corruption4.mdf", "C:\Exo\Database\Corruption4_log.ldf"
icacls "C:\Exo\Database\Corruption4.mdf"     /grant "NT SERVICE\MSSQL`$EXO:F" "$($env:USERNAME):F"
icacls "C:\Exo\Database\Corruption4_log.ldf" /grant "NT SERVICE\MSSQL`$EXO:F" "$($env:USERNAME):F"
```

#### What it achieves
Places the files in your SQL directory and gives both the SQL Server service account (`NT SERVICE\MSSQL$EXO`) and your Windows user account explicit Full Control.

#### Key Learnings & Intentional Trap
- **Trap (`Msg 5120: Operating system error 5 (Access is denied.)`):** Unlike `RESTORE DATABASE` (where SQL Server creates fresh `.mdf`/`.ldf` files it owns), `CREATE DATABASE ... FOR ATTACH` opens *existing* files you copied manually. If `NT SERVICE\MSSQL$EXO` and your attaching Windows user lack Full Control (e.g., because `BUILTIN\Users` only has Read & Execute on `C:\Exo\Database` and UAC strips `BUILTIN\Administrators`), attaching fails with `Access is denied`.

---

### Step 4B.2 — Attempt Normal Attach & Rebuild-Log Attach (Observe the Failures)
**Task:** Test why standard attach methods fail on a power-failure-damaged database.

#### How to do it
1. **Inspect internal file metadata first (read-only):**
   ```sql
   DBCC CHECKPRIMARYFILE (N'C:\Exo\Database\Corruption4.mdf', 3);
   GO
   ```
   *(Confirms logical names `Corruption4` and `Corruption4_log`.)*

2. **Attempt #1: Standard `FOR ATTACH`:**
   ```sql
   USE [master];
   GO

   CREATE DATABASE [Corruption4]
   ON (FILENAME = N'C:\Exo\Database\Corruption4.mdf'),
      (FILENAME = N'C:\Exo\Database\Corruption4_log.ldf')
   FOR ATTACH;
   GO
   ```
   **Observed Error:**
   ```text
   Msg 824, Level 24, State 6, ...
   SQL Server detected a logical consistency-based I/O error: incorrect checksum (expected: 0x0b407bbf; actual: 0xf4bf8440). It occurred during a read of page (0:0) in database ID 0 at offset 0000000000000000 in file 'C:\Exo\Database\Corruption4_log.ldf'.
   ```
   *(The header page `(0:0)` of `Corruption4_log.ldf` is corrupted!)*

3. **Attempt #2: Attach without the corrupted `.ldf` using `FOR ATTACH_REBUILD_LOG`:**
   ```sql
   CREATE DATABASE [Corruption4]
   ON (FILENAME = N'C:\Exo\Database\Corruption4.mdf')
   FOR ATTACH_REBUILD_LOG;
   GO
   ```
   **Observed Error:**
   ```text
   Msg 1813, Level 16, State 2, ...
   File activation failure. The physical file name "C:\Exo\Database\Corruption4_log.ldf" may be incorrect.
   The log cannot be rebuilt because there were open transactions/users when the database was shutdown, no checkpoint occurred to the database, or the database was read-only.
   ```

#### What it achieves
Diagnoses the exact root cause: the `.ldf` file header is corrupt (`Msg 824`), **and** because the database suffered an unclean shutdown (simulated power failure with open transactions), SQL Server refuses to simply attach and rebuild the log (`Msg 1813`).

---

### Step 4B.3 — Perform the DBA "Hack-Attach" + `EMERGENCY` Mode Recovery
**Task:** Register `Corruption4.mdf` inside SQL Server's system catalogs using a temporary dummy database, switch to `EMERGENCY` mode, rebuild the log, and repair all corruption.

#### How to do it
```sql
USE [master];
GO

-- 1. Create a dummy database with the exact same name & logical file layout
CREATE DATABASE [Corruption4]
ON (NAME = N'Corruption4',     FILENAME = N'C:\Exo\Database\Corruption4_dummy.mdf')
LOG ON (NAME = N'Corruption4_log', FILENAME = N'C:\Exo\Database\Corruption4_dummy_log.ldf');
GO

-- 2. Take the dummy database OFFLINE so SQL Server releases its file handles
ALTER DATABASE [Corruption4] SET OFFLINE WITH ROLLBACK IMMEDIATE;
GO

-- 3. Point [Corruption4] in master's catalog to the REAL Corruption4.mdf and a fresh log filename
ALTER DATABASE [Corruption4]
MODIFY FILE (NAME = N'Corruption4',     FILENAME = N'C:\Exo\Database\Corruption4.mdf');
GO
ALTER DATABASE [Corruption4]
MODIFY FILE (NAME = N'Corruption4_log', FILENAME = N'C:\Exo\Database\Corruption4_rebuilt_log.ldf');
GO

-- 4. Attempt SET ONLINE (Expected to fail because the log does not exist/cannot be rebuilt cleanly)
ALTER DATABASE [Corruption4] SET ONLINE;
GO

-- 5. Put the database into EMERGENCY mode (read-only bypass that ignores the missing/corrupt log)
ALTER DATABASE [Corruption4] SET EMERGENCY;
GO
```

*(Note: If the `MODIFY FILE` log path matches an existing file like `C:\Exo\Database\Corruption4_log.ldf` that lacks `NT SERVICE\MSSQL$EXO:(F)`, rename the damaged log file to `Corruption4_log.ldf.corrupt` before running `REPAIR_ALLOW_DATA_LOSS`. You can also delete unused `Corruption4_dummy.*` files later.)*

#### What it achieves
Tricks SQL Server into attaching `Corruption4.mdf` into `sys.databases` and brings it into **`EMERGENCY`** state (`MULTI_USER`, read-only) so `DBCC CHECKDB` can inspect and repair it.

---

### Step 4B.4 — Diagnose, Rebuild the Log, and Repair `Corruption4`
**Task:** Run `DBCC CHECKDB`, repair the database, bring it `ONLINE` in `MULTI_USER` mode, and confirm `CHECKDB` reports **0 errors**.

#### How to do it
```sql
USE [master];
GO

-- 1. Check what corruption exists inside Corruption4.mdf while in EMERGENCY mode
DBCC CHECKDB (N'Corruption4') WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO
```
**Observed Output:**
```text
Msg 8951, Level 16, State 1, ...
Table error: table 'sys.sysiscols' (ID 55). Data row does not have a matching index row in the index 'nc1' (ID 2).
...
CHECKDB found 0 allocation errors and 1 consistency errors in table 'sys.sysiscols' (object ID 55).
CHECKDB found 0 allocation errors and 1 consistency errors in database 'Corruption4'.
repair_rebuild is the minimum repair level for the errors found by DBCC CHECKDB (Corruption4).
```

Now run the repair and return the database to `ONLINE` / `MULTI_USER`:
```sql
-- 2. Switch to SINGLE_USER (required for DBCC repair)
ALTER DATABASE [Corruption4] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

-- 3. Rebuild the missing transaction log AND fix the sys.sysiscols index error
DBCC CHECKDB (N'Corruption4', REPAIR_ALLOW_DATA_LOSS)
WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO

-- 4. Return database to normal MULTI_USER access
ALTER DATABASE [Corruption4] SET MULTI_USER;
GO

-- 5. Verify database state and confirm 0 CHECKDB errors
SELECT name, state_desc, user_access_desc
FROM sys.databases
WHERE name = N'Corruption4';
GO

DBCC CHECKDB (N'Corruption4') WITH NO_INFOMSGS, ALL_ERRORMSGS;
GO

-- 6. (Recommended) Check foreign-key constraint integrity after REPAIR_ALLOW_DATA_LOSS
USE [Corruption4];
GO
DBCC CHECKCONSTRAINTS WITH ALL_CONSTRAINTS;
GO
```

#### What it achieves
- Creates the brand-new transaction log, resolving the broken `.ldf` from the power failure.
- Rebuilds the corrupted non-clustered index `nc1` on system catalog table `sys.sysiscols` (`CHECKDB fixed 0 allocation errors and 1 consistency errors`).
- Brings `[Corruption4]` back to **`ONLINE` / `MULTI_USER`** so `DBCC CHECKDB` runs cleanly with **0 errors**.

#### Key Learnings & Intentional Traps
- **INTENTIONAL TRAP (`REPAIR_REBUILD` Fails in `EMERGENCY` Mode!):**
  Even though `DBCC CHECKDB` prints `repair_rebuild is the minimum repair level` for the `sys.sysiscols` error, running `DBCC CHECKDB (N'Corruption4', REPAIR_REBUILD)` while in `EMERGENCY` mode fails with:
  > **`Msg 7901: The repair statement was not processed. This level of repair is not supported when the database is in emergency mode.`**
  Why? Because rebuilding a missing/corrupt transaction log breaks transactional ACID guarantees for any uncommitted transactions at the time of the power failure, so SQL Server **requires `REPAIR_ALLOW_DATA_LOSS`** to rebuild the log!
- **Post-Recovery Best Practice:** Immediately take a **new Full Backup** (`BACKUP DATABASE [Corruption4] ...`) after using `REPAIR_ALLOW_DATA_LOSS`, because any previous backup/log chain is permanently broken (`Warning: The log for database 'Corruption4' has been rebuilt. Transactional consistency has been lost.`).

---

## Part 5: `MS SQL Server DatabaseMail Configuration` (`DatabaseMail.papercut.SMTP` + Agent Mail)

Before SQL Server Agent jobs can send email alerts in Part 6, you need a working local SMTP server, Database Mail configured on your SQL Server instance, and SQL Server Agent linked to that Database Mail profile.

### Step 5.1 — Install Papercut SMTP, Configure Firewall & Verify in PowerShell
**Task:**
1. Install a local SMTP server (**Papercut SMTP**).
2. On the Windows Firewall prompt, **only allow Private networks** for Papercut SMTP.
3. Verify SMTP mail delivery using PowerShell (`Send-MailMessage`).

#### How to do it
1. **Download and install Papercut SMTP:**
   - Website: `https://www.papercut-smtp.com/`
   - Direct installer: `https://github.com/ChangemakerStudios/Papercut-SMTP/releases/latest/download/PapercutSMTP-win-x64-stable-Setup.exe`
   ```powershell
   $InstallerPath = "$env:TEMP\PapercutSMTP-win-x64-stable-Setup.exe"
   Invoke-WebRequest -Uri "https://github.com/ChangemakerStudios/Papercut-SMTP/releases/latest/download/PapercutSMTP-win-x64-stable-Setup.exe" -OutFile $InstallerPath
   Start-Process -FilePath $InstallerPath -Wait
   ```
2. **Launch Papercut SMTP & Configure Windows Firewall:**
   - When Windows Defender Firewall prompts on first launch:
     - **Check** `Private networks, such as my home or work network`
     - **Uncheck** `Public networks, such as those in airports and coffee shops`
   - Confirm Papercut is listening on `127.0.0.1:25` (default) and verify in PowerShell that no `Public` inbound allow rule exists:
     ```powershell
     Get-NetFirewallApplicationFilter -Program "*Papercut*" |
         Get-NetFirewallRule |
         Select-Object DisplayName, Enabled, Direction, Action, Profile
     ```
3. **Verify SMTP functionality in PowerShell:**
   ```powershell
   Send-MailMessage -SmtpServer localhost -Port 25 -From "test@local" -To "anyone@anywhere.example" -Subject "Hello from Papercut!" -Body "It works!"
   ```
   Open the **Papercut SMTP** window and confirm `"Hello from Papercut!"` appears in the inbox.

#### What it achieves
Provides a safe, zero-cost local SMTP sink on port `25` that catches every email sent by PowerShell and SQL Server Database Mail without needing external internet relay credentials or spamming real inboxes.

#### Key Learnings
- Restricting Papercut to **Private networks** prevents machines on untrusted Public networks (e.g., café/campus Wi-Fi) from connecting to your local port `25` SMTP listener.
- PowerShell's `Send-MailMessage` prints a deprecation warning in newer .NET versions, which is harmless for a local lab test (production scripts should use `MailKit` or authenticated APIs).

---

### Step 5.2 — Configure SQL Server Database Mail & SQL Agent Mail (T-SQL)
**Task:** Configure SQL Server Database Mail, link it to SQL Server Agent, create a DBA Operator, and verify mail delivery (`test@local.local` and formatted HTML email) through Papercut SMTP.

*(Aligned with your existing `Papercut_Local_Account` and `Papercut_Local_Profile` from Nov 13—safe and idempotent to re-run!)*

#### How to do it
**Option A: Using T-SQL (Recommended)**
```sql
USE [master];
GO

-- 1. Enable Database Mail Extended Stored Procedures
EXEC sp_configure N'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure N'Database Mail XPs', 1;
RECONFIGURE;
GO

-- 2. Ensure Service Broker is enabled in msdb (required for Database Mail queue)
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = N'msdb' AND is_broker_enabled = 0)
BEGIN
    ALTER DATABASE [msdb] SET ENABLE_BROKER WITH ROLLBACK IMMEDIATE;
END
GO

USE [msdb];
GO

-- 3. Ensure a Database Mail Account exists pointing to local Papercut SMTP (port 25)
IF NOT EXISTS (SELECT 1 FROM msdb.dbo.sysmail_account WHERE name = N'Papercut_Local_Account')
BEGIN
    EXEC msdb.dbo.sysmail_add_account_sp
        @account_name            = N'Papercut_Local_Account',
        @description             = N'Local Papercut SMTP server account for lab testing',
        @email_address           = N'sqlserver@local.local',
        @display_name            = N'SQL Server Alerting (EXO)',
        @replyto_address         = N'sqlserver@local.local',
        @mailserver_name         = N'127.0.0.1',
        @port                    = 25,
        @use_default_credentials = 0,
        @enable_ssl              = 0;
END
GO

-- 4. Ensure the Database Mail Profile exists
IF NOT EXISTS (SELECT 1 FROM msdb.dbo.sysmail_profile WHERE name = N'Papercut_Local_Profile')
BEGIN
    EXEC msdb.dbo.sysmail_add_profile_sp
        @profile_name = N'Papercut_Local_Profile',
        @description  = N'Profile for SQL Server DBA alerts and notifications via local Papercut SMTP.';
END
GO

-- 5. Link the Account to the Profile (sequence_number = 1)
IF NOT EXISTS (
    SELECT 1
    FROM msdb.dbo.sysmail_profileaccount pa
    JOIN msdb.dbo.sysmail_profile p ON pa.profile_id = p.profile_id
    JOIN msdb.dbo.sysmail_account a ON pa.account_id = a.account_id
    WHERE p.name = N'Papercut_Local_Profile'
      AND a.name = N'Papercut_Local_Account'
)
BEGIN
    EXEC msdb.dbo.sysmail_add_profileaccount_sp
        @profile_name    = N'Papercut_Local_Profile',
        @account_name    = N'Papercut_Local_Account',
        @sequence_number = 1;
END
GO

-- 6. Grant public access & set as default only if no public default profile already exists
IF NOT EXISTS (
    SELECT 1
    FROM msdb.dbo.sysmail_principalprofile pp
    JOIN msdb.dbo.sysmail_profile p ON pp.profile_id = p.profile_id
    WHERE p.name = N'Papercut_Local_Profile'
      AND pp.principal_sid = 0x00
)
BEGIN
    DECLARE @HasDefault BIT = CASE
        WHEN EXISTS (SELECT 1 FROM msdb.dbo.sysmail_principalprofile WHERE principal_sid = 0x00 AND is_default = 1)
            THEN 0
        ELSE 1
    END;

    EXEC msdb.dbo.sysmail_add_principalprofile_sp
        @profile_name   = N'Papercut_Local_Profile',
        @principal_name = N'public',
        @is_default     = @HasDefault;
END
GO

-- 7. Configure SQL Server Agent to use Database Mail & create a DBA Operator
EXEC msdb.dbo.sp_set_sqlagent_properties
    @email_save_in_sent_folder = 1,
    @databasemail_profile      = N'Papercut_Local_Profile',
    @use_databasemail          = 1;
GO

IF NOT EXISTS (SELECT 1 FROM msdb.dbo.sysoperators WHERE name = N'DBA_Admin')
BEGIN
    EXEC msdb.dbo.sp_add_operator
        @name          = N'DBA_Admin',
        @enabled       = 1,
        @email_address = N'dba@anywhere.example';
END
GO

-- 8. Test Database Mail with a formatted HTML email (to test@local.local)
EXEC msdb.dbo.sp_send_dbmail
    @profile_name = N'Papercut_Local_Profile',
    @recipients   = N'test@local.local',
    @subject      = N'SQL Server Database Mail & Agent Test',
    @body         = N'<h3>Database Mail is working!</h3><p>Sent from <b>SQL Server (EXOPI-908\EXO)</b> to Papercut SMTP.</p>',
    @body_format  = N'HTML';
GO

-- 9. Verify delivery status in msdb (wait a few seconds for Service Broker queue)
WAITFOR DELAY '00:00:05';
SELECT TOP (5) mailitem_id, recipients, subject, sent_status, sent_date
FROM msdb.dbo.sysmail_allitems
ORDER BY mailitem_id DESC;

SELECT TOP (10) log_date, event_type, description
FROM msdb.dbo.sysmail_event_log
ORDER BY log_id DESC;
GO
```

*(Note: After setting `@use_databasemail = 1` on SQL Server Agent, **restart the SQL Server Agent service** — `SQLAgent$EXO` on `EXOPI-908\EXO` — e.g. `Restart-Service 'SQLAgent$EXO'` in an elevated PowerShell window or via SSMS Object Explorer -> Right-click **SQL Server Agent** -> **Restart**, so Agent picks up the new mail profile for job-level notifications.)*

**Option B: Using SSMS GUI (Alternative)**
1. In Object Explorer, expand **Management** $\rightarrow$ right-click **Database Mail** $\rightarrow$ **Configure Database Mail**.
2. Select **Set up Database Mail by performing the following tasks** $\rightarrow$ **Next**.
3. If prompted that the Database Mail feature is not available, click **Yes** to enable it.
4. Enter Profile name **`Papercut_Local_Profile`** $\rightarrow$ click **Add...** to create a new SMTP Account:
   - Account name: `Papercut_Local_Account`
   - E-mail address: `sqlserver@local.local`
   - Server name: `127.0.0.1` (or `localhost`) | Port number: `25`
   - Leave **This server requires a secure connection (SSL)** unchecked and select **Anonymous authentication** $\rightarrow$ **OK** $\rightarrow$ **Next**.
5. Under **Manage Profile Security**, check **Public** and set **Default Profile** = **Yes** $\rightarrow$ **Next** $\rightarrow$ **Next** $\rightarrow$ **Finish**.
6. Right-click **SQL Server Agent** $\rightarrow$ **Properties** $\rightarrow$ **Alert System** $\rightarrow$ check **Enable mail profile**, select **Database Mail** and **`Papercut_Local_Profile`**, click **OK**, and **Restart** SQL Server Agent.

#### What it achieves
Enables SQL Server's asynchronous Service Broker-backed mail subsystem (`DatabaseMail.exe`), registers `Papercut_Local_Profile` as the default public mail profile, and links SQL Server Agent so jobs and alerts can send notifications through Papercut SMTP.

#### Key Learnings & Traps
- **Trap 1 (`Database Mail XPs` disabled by default):** SQL Server disables Database Mail out of the box for attack-surface reduction. Calling `sp_send_dbmail` before enabling `Database Mail XPs` raises **Msg 15281** (*"SQL Server blocked access to procedure 'dbo.sp_send_dbmail'..."*).
- **Trap 2 (Asynchronous Delivery):** `sp_send_dbmail` returns immediately (`Mail (Id: ...) queued.`) even if SMTP is down! Always inspect `msdb.dbo.sysmail_allitems` (`sent_status`: `sent`, `unsent`, `retrying`, `failed`) and `msdb.dbo.sysmail_event_log` when troubleshooting.
- **Trap 3 (SSL & Authentication with Papercut):** Papercut runs unencrypted on port `25` with anonymous authentication. Enabling SSL (`@enable_ssl = 1`) or passing Windows credentials causes delivery to fail.
- **Trap 4 (`127.0.0.1` vs `localhost`):** Using `127.0.0.1` avoids IPv6 `::1` resolution mismatches if Papercut is only bound to IPv4 loopback.
- **Trap 5 (One Default Profile Per Principal):** Calling `sysmail_add_principalprofile_sp` with `@is_default = 1` for `public` fails if another profile is already marked as the `public` default (`sysmail_principalprofile`). Always check before setting `@is_default = 1`.

---

## Part 6: `DatabaseMail.Notification` — Integrity Alerts, DDL Auditing & Impersonation

### Step 6.1 — Create & Test SQL Agent Job `CheckDB`
**Task:**
1. Create a SQL Agent job named **`CheckDB`**.
2. Run integrity checks (`DBCC CHECKDB`) across **all user and system databases**.
3. Get notified by an **HTML/CSS table-formatted email** if `CheckDB` finds any kind of corruption.
4. Test the solution.

#### How to do it
The key to sending a rich HTML table *only when corruption is found* is `DBCC CHECKDB (...) WITH NO_INFOMSGS, ALL_ERRORMSGS, TABLERESULTS`.
- When a database is **healthy**, `WITH NO_INFOMSGS, TABLERESULTS` returns **0 rows**.
- When **corruption exists**, it returns structured rows (`Error`, `Level`, `State`, `MessageText`, `RepairLevel`, `File`, `Page`, `Slot`, etc.) that we collect into a temp table across every `ONLINE` database and render as an HTML `<table>`.
- Wrapping each database check in `BEGIN TRY ... END TRY BEGIN CATCH ... END CATCH` ensures severe metadata/snapshot errors are also captured and emailed rather than aborting the loop prematurely.

```sql
USE [msdb];
GO

IF EXISTS (SELECT 1 FROM msdb.dbo.sysjobs WHERE name = N'CheckDB')
    EXEC msdb.dbo.sp_delete_job @job_name = N'CheckDB', @delete_unused_schedule = 1;
GO

EXEC dbo.sp_add_job
    @job_name    = N'CheckDB',
    @enabled     = 1,
    @description = N'Runs DBCC CHECKDB on all online system and user databases and sends an HTML/CSS email if corruption is detected.';
GO

EXEC dbo.sp_add_jobstep
    @job_name      = N'CheckDB',
    @step_name     = N'Run DBCC CHECKDB and Notify on Corruption',
    @subsystem     = N'TSQL',
    @database_name = N'master',
    @command       = N'
SET NOCOUNT ON;

IF OBJECT_ID(N''tempdb..#CheckDBResults'') IS NOT NULL DROP TABLE #CheckDBResults;
CREATE TABLE #CheckDBResults (
    Error        INT,
    [Level]      INT,
    [State]      INT,
    MessageText  NVARCHAR(4000),
    RepairLevel  NVARCHAR(128),
    [Status]     INT,
    [DbId]       INT,
    DbFragId     INT,
    ObjectId     INT,
    IndexId      INT,
    PartitionId  BIGINT,
    AllocUnitId  BIGINT,
    RidDbId      INT,
    RidPruId     INT,
    [File]       INT,
    [Page]       INT,
    Slot         INT,
    RefDbId      INT,
    RefPruId     INT,
    RefFile      INT,
    RefPage      INT,
    RefSlot      INT,
    Allocation   INT
);

IF OBJECT_ID(N''tempdb..#AllCorruption'') IS NOT NULL DROP TABLE #AllCorruption;
CREATE TABLE #AllCorruption (
    DatabaseName SYSNAME,
    Error        INT,
    [Level]      INT,
    [State]      INT,
    RepairLevel  NVARCHAR(128),
    FilePage     NVARCHAR(50),
    MessageText  NVARCHAR(4000)
);

DECLARE @DbName SYSNAME;
DECLARE @Sql    NVARCHAR(MAX);

DECLARE DbCursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT name
    FROM sys.databases
    WHERE state_desc = N''ONLINE''
      AND source_database_id IS NULL -- exclude database snapshots
    ORDER BY database_id;

OPEN DbCursor;
FETCH NEXT FROM DbCursor INTO @DbName;

WHILE @@FETCH_STATUS = 0
BEGIN
    TRUNCATE TABLE #CheckDBResults;

    BEGIN TRY
        SET @Sql = N''DBCC CHECKDB ('' + QUOTENAME(@DbName) + N'') WITH NO_INFOMSGS, ALL_ERRORMSGS, TABLERESULTS;'';

        INSERT INTO #CheckDBResults
        EXEC (@Sql);

        -- Keep actual error/corruption rows (Level >= 11 or Error > 0)
        INSERT INTO #AllCorruption (DatabaseName, Error, [Level], [State], RepairLevel, FilePage, MessageText)
        SELECT
            @DbName,
            Error,
            [Level],
            [State],
            ISNULL(RepairLevel, N''N/A''),
            CASE
                WHEN [File] IS NOT NULL AND [Page] IS NOT NULL
                THEN CONCAT([File], N'':'', [Page])
                ELSE N''-''
            END,
            ISNULL(MessageText, N''-'')
        FROM #CheckDBResults
        WHERE Error > 0 OR [Level] >= 11 OR MessageText LIKE N''%corrupt%'' OR MessageText LIKE N''%consistency error%'';
    END TRY
    BEGIN CATCH
        INSERT INTO #AllCorruption (DatabaseName, Error, [Level], [State], RepairLevel, FilePage, MessageText)
        VALUES (
            @DbName,
            ERROR_NUMBER(),
            ERROR_SEVERITY(),
            ERROR_STATE(),
            N''Check Failed'',
            N''-'',
            ISNULL(ERROR_MESSAGE(), N''Unknown error'')
        );
    END CATCH

    FETCH NEXT FROM DbCursor INTO @DbName;
END

CLOSE DbCursor;
DEALLOCATE DbCursor;

-- Send HTML/CSS email ONLY if corruption or check errors were found
IF EXISTS (SELECT 1 FROM #AllCorruption)
BEGIN
    DECLARE @TableRows NVARCHAR(MAX);
    DECLARE @HtmlBody  NVARCHAR(MAX);
    DECLARE @Subject   NVARCHAR(255) = N''[CRITICAL] DBCC CHECKDB Corruption Detected on '' + @@SERVERNAME;

    SELECT @TableRows = CAST((
        SELECT
            td = DatabaseName,                                    N'''',
            td = CAST(Error AS NVARCHAR(20)),                     N'''',
            td = CAST([Level] AS NVARCHAR(20)),                   N'''',
            td = FilePage,                                        N'''',
            td = RepairLevel,                                     N'''',
            td = MessageText
        FROM #AllCorruption
        ORDER BY DatabaseName, [Level] DESC, Error
        FOR XML PATH(''tr''), TYPE
    ) AS NVARCHAR(MAX));

    SET @HtmlBody = N''
    <html>
    <head>
        <style>
            body { font-family: Segoe UI, Arial, sans-serif; color: #1f2937; background-color: #f9fafb; margin: 0; padding: 20px; }
            .card { background: #ffffff; border: 1px solid #e5e7eb; border-radius: 8px; padding: 20px; max-width: 960px; }
            h2 { color: #b91c1c; margin-top: 0; }
            .meta { font-size: 13px; color: #4b5563; margin-bottom: 16px; }
            table { border-collapse: collapse; width: 100%; font-size: 13px; }
            th { background-color: #991b1b; color: #ffffff; text-align: left; padding: 10px 12px; border: 1px solid #7f1d1d; }
            td { padding: 8px 12px; border: 1px solid #e5e7eb; vertical-align: top; }
            tr:nth-child(even) td { background-color: #fef2f2; }
        </style>
    </head>
    <body>
        <div class="card">
            <h2>DBCC CHECKDB Corruption Alert</h2>
            <div class="meta">
                <strong>Server:</strong> '' + @@SERVERNAME + N'' &nbsp;|&nbsp;
                <strong>Detected At:</strong> '' + CONVERT(NVARCHAR(30), SYSDATETIME(), 120) + N''
            </div>
            <table>
                <thead>
                    <tr>
                        <th>Database</th>
                        <th>Error</th>
                        <th>Severity</th>
                        <th>File:Page</th>
                        <th>Min Repair Level</th>
                        <th>Details</th>
                    </tr>
                </thead>
                <tbody>'' + ISNULL(@TableRows, N'''') + N''</tbody>
            </table>
        </div>
    </body>
    </html>'';

    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = N''Papercut_Local_Profile'',
        @recipients   = N''dba@anywhere.example'',
        @subject      = @Subject,
        @body         = @HtmlBody,
        @body_format  = N''HTML'',
        @importance   = N''HIGH'';
END
';
GO

EXEC dbo.sp_add_jobserver
    @job_name    = N'CheckDB',
    @server_name = N'(local)';
GO
```

#### Testing the `CheckDB` Job
1. **Baseline clean run:**
   ```sql
   EXEC msdb.dbo.sp_start_job @job_name = N'CheckDB';
   ```
   Verify in `msdb.dbo.sysjobhistory` that the job succeeds and **no email** is sent when all databases are healthy.
2. **Test corruption detection safely in a disposable database:**
   *(Warning: **Never** run `DBCC WRITEPAGE` on a real database—only on a brand-new disposable test database `CorruptTestDB`, or restore a sample corrupt `.bak` from Part 7!)*
   ```sql
   USE [master];
   GO
   CREATE DATABASE [CorruptTestDB];
   GO
   USE [CorruptTestDB];
   GO
   CREATE TABLE dbo.TestData (ID INT IDENTITY(1,1) PRIMARY KEY, Payload CHAR(2000) DEFAULT 'A');
   INSERT INTO dbo.TestData DEFAULT VALUES;
   GO 10

   -- Find the first data page (PageType = 1) of dbo.TestData
   DECLARE @FileId INT, @PageId INT;
   SELECT TOP (1) @FileId = allocated_page_file_id, @PageId = allocated_page_page_id
   FROM sys.dm_db_database_page_allocations(DB_ID(N'CorruptTestDB'), OBJECT_ID(N'dbo.TestData'), NULL, NULL, N'DETAILED')
   WHERE page_type = 1 AND is_allocated = 1;

   -- Intentionally corrupt 4 bytes inside the page header/slot in SINGLE_USER lab mode
   ALTER DATABASE [CorruptTestDB] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
   DBCC WRITEPAGE (N'CorruptTestDB', @FileId, @PageId, 0, 4, 0xFFFFFFFF, 1);
   ALTER DATABASE [CorruptTestDB] SET MULTI_USER;
   GO

   -- Run the CheckDB SQL Agent job and watch Papercut SMTP!
   EXEC msdb.dbo.sp_start_job @job_name = N'CheckDB';
   GO
   ```
   Check **Papercut SMTP**: you should receive `[CRITICAL] DBCC CHECKDB Corruption Detected` with a styled HTML table showing `CorruptTestDB`, the exact `File:Page`, `Min Repair Level`, and error message.
3. **Clean up the test database afterwards:**
   ```sql
   USE [master];
   GO
   IF DB_ID(N'CorruptTestDB') IS NOT NULL
   BEGIN
       ALTER DATABASE [CorruptTestDB] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
       DROP DATABASE [CorruptTestDB];
   END
   GO
   ```

#### What it achieves
Continuously protects every system (`master`, `model`, `msdb`, `tempdb`) and user database by automating integrity verification and immediately alerting the DBA with actionable page/repair metadata.

#### Key Learnings & Traps
- **Trap 1 (`TABLERESULTS` Schema):** `DBCC CHECKDB ... WITH TABLERESULTS` has 23 columns in modern SQL Server versions; your `#CheckDBResults` temp table must match all 23 columns exactly or `INSERT INTO ... EXEC` will fail with a column-count mismatch.
- **Trap 2 (`tempdb` Snapshot Limitation):** `DBCC CHECKDB (N'tempdb')` cannot create an internal database snapshot, so SQL Server runs `CHECKALLOC` and `CHECKCATALOG` directly and skips snapshot-based checks.
- **Trap 3 (XML-Encoding Safety):** Building HTML rows via `FOR XML PATH('tr'), TYPE` automatically escapes `<`, `>`, and `&` inside error messages so malformed text never breaks your HTML table.

---

### Step 6.2 — Create SQL Agent Job `DatabaseDML` (Database & User Create/Delete Alerts)
**Task:**
1. Create a SQL Agent job named **`DatabaseDML`** with two steps:
   - **Step 1:** Get notified by email when a **database** is created or deleted and **who performed the operation**.
   - **Step 2:** Get notified by email when a **database user** is created or deleted and **who performed the operation**.
2. Format emails cleanly using **HTML and CSS tables** (rows and columns).

> [!NOTE]
> **Terminology Note (`DatabaseDML` vs. DDL):** Even though the assignment names the job `DatabaseDML`, `CREATE DATABASE`, `DROP DATABASE`, `CREATE USER`, and `DROP USER` are actually **DDL (Data Definition Language)** / **DCL** operations—not DML (`INSERT`, `UPDATE`, `DELETE`).

#### How to do it
SQL Server's **Default Trace** (`sys.fn_trace_gettable`) is enabled out of the box and automatically records:
- **Database Create / Delete:** `EventClass IN (46, 47)` (`Object:Created` / `Object:Deleted`) where `ObjectType = 16964` (`Database`) and `EventSubClass = 1` (`Commit`).
- **Database User Create / Delete:** `EventClass = 109` (`Audit Add DB User Event`) where `EventSubClass IN (1, 2, 3, 4)` (`1 = Add`, `2 = Drop`, `3 = Grant DB Access`, `4 = Revoke DB Access`).
- **Impersonation Tracking (`LoginName` vs. `SessionLoginName`):** Crucially, the Default Trace logs **both** `LoginName` (the effective/impersonated identity, e.g., `John`) **and** `SessionLoginName` (the real authenticated login that opened the connection and ran `EXECUTE AS`).

First, create a small watermark table in `msdb` so each job step only emails **new** events since the previous run (avoiding duplicate emails):

```sql
USE [msdb];
GO

IF OBJECT_ID(N'dbo.DBA_TraceWatermark', N'U') IS NULL
BEGIN
    CREATE TABLE dbo.DBA_TraceWatermark (
        CheckKey      VARCHAR(50) NOT NULL PRIMARY KEY,
        LastEventTime DATETIME2(3) NOT NULL
    );

    -- Initialize watermarks to current time (or slightly in the past to catch immediate tests)
    INSERT INTO dbo.DBA_TraceWatermark (CheckKey, LastEventTime)
    VALUES ('DATABASE_DDL', DATEADD(MINUTE, -5, GETDATE())),
           ('DBUSER_DDL',   DATEADD(MINUTE, -5, GETDATE()));
END
GO
```

Now create the **`DatabaseDML`** SQL Agent job with **Step 1** (Database Create/Delete) and **Step 2** (Database User Create/Delete):

```sql
USE [msdb];
GO

IF EXISTS (SELECT 1 FROM msdb.dbo.sysjobs WHERE name = N'DatabaseDML')
    EXEC msdb.dbo.sp_delete_job @job_name = N'DatabaseDML', @delete_unused_schedule = 1;
GO

EXEC dbo.sp_add_job
    @job_name    = N'DatabaseDML',
    @enabled     = 1,
    @description = N'Monitors Default Trace for Database and Database User Create/Delete events and sends styled HTML table alerts.';
GO

-- ============================================================================
-- STEP 1: Notify when a Database is created or deleted (and who did it)
-- ============================================================================
EXEC dbo.sp_add_jobstep
    @job_name          = N'DatabaseDML',
    @step_name         = N'Notify Database Create or Delete',
    @step_id           = 1,
    @subsystem         = N'TSQL',
    @database_name     = N'msdb',
    @on_success_action = 3, -- Go to next step
    @on_fail_action    = 3, -- Continue to Step 2 even if Step 1 fails
    @command           = N'
SET NOCOUNT ON;

DECLARE @TracePath NVARCHAR(260);
SELECT @TracePath = [path] FROM sys.traces WHERE is_default = 1;

IF @TracePath IS NULL RETURN;

DECLARE @LastChecked DATETIME2(3);
SELECT @LastChecked = LastEventTime
FROM msdb.dbo.DBA_TraceWatermark
WHERE CheckKey = ''DATABASE_DDL'';

IF OBJECT_ID(N''tempdb..#DbEvents'') IS NOT NULL DROP TABLE #DbEvents;

SELECT
    CASE t.EventClass
        WHEN 46 THEN N''CREATED''
        WHEN 47 THEN N''DELETED''
    END AS Operation,
    COALESCE(t.DatabaseName, t.ObjectName, N''-'') AS DatabaseName,
    ISNULL(t.LoginName, N''-'')                    AS EffectiveLogin,
    ISNULL(t.SessionLoginName, N''-'')             AS OriginalLogin,
    ISNULL(t.HostName, N''-'')                     AS HostName,
    ISNULL(t.ApplicationName, N''-'')              AS ApplicationName,
    t.StartTime
INTO #DbEvents
FROM sys.fn_trace_gettable(@TracePath, DEFAULT) t
WHERE t.EventClass IN (46, 47)   -- 46 = Object:Created, 47 = Object:Deleted
  AND t.ObjectType = 16964       -- 16964 = Database
  AND t.EventSubClass = 1        -- 1 = Commit (avoids duplicate Begin/Commit rows)
  AND ISNULL(t.IsSystem, 0) = 0  -- Ignore internal system/DBCC snapshot events
  AND COALESCE(t.DatabaseName, t.ObjectName, N'''') <> N''tempdb''
  AND COALESCE(t.DatabaseName, t.ObjectName, N'''') NOT LIKE N''%_MSSQL_DBCC%''
  AND t.StartTime > @LastChecked;

IF EXISTS (SELECT 1 FROM #DbEvents)
BEGIN
    DECLARE @TableRows NVARCHAR(MAX);
    DECLARE @HtmlBody  NVARCHAR(MAX);
    DECLARE @Subject   NVARCHAR(255) = N''[SQL Audit] Database Create/Delete Detected on '' + @@SERVERNAME;

    SELECT @TableRows = CAST((
        SELECT
            td = Operation,                                       N'''',
            td = DatabaseName,                                    N'''',
            td = EffectiveLogin,                                  N'''',
            td = OriginalLogin,                                   N'''',
            td = HostName,                                        N'''',
            td = ApplicationName,                                 N'''',
            td = CONVERT(NVARCHAR(23), StartTime, 121)
        FROM #DbEvents
        ORDER BY StartTime ASC
        FOR XML PATH(''tr''), TYPE
    ) AS NVARCHAR(MAX));

    SET @HtmlBody = N''
    <html>
    <head>
        <style>
            body { font-family: Segoe UI, Arial, sans-serif; color: #1f2937; background-color: #f3f4f6; margin: 0; padding: 20px; }
            .card { background: #ffffff; border: 1px solid #d1d5db; border-radius: 8px; padding: 20px; max-width: 1000px; }
            h2 { color: #1d4ed8; margin-top: 0; }
            p.sub { color: #4b5563; font-size: 13px; }
            table { border-collapse: collapse; width: 100%; font-size: 13px; margin-top: 12px; }
            th { background-color: #1e3a8a; color: #ffffff; text-align: left; padding: 10px 12px; border: 1px solid #1e40af; }
            td { padding: 8px 12px; border: 1px solid #e5e7eb; }
            tr:nth-child(even) td { background-color: #f8fafc; }
        </style>
    </head>
    <body>
        <div class="card">
            <h2>Database Creation / Deletion Audit</h2>
            <p class="sub">Server: <strong>'' + @@SERVERNAME + N''</strong> | Note: <em>OriginalLogin</em> reveals the real account if <code>EXECUTE AS</code> impersonation was used.</p>
            <table>
                <thead>
                    <tr>
                        <th>Operation</th>
                        <th>Database Name</th>
                        <th>Effective Login</th>
                        <th>Original Login</th>
                        <th>Host</th>
                        <th>Application</th>
                        <th>Timestamp</th>
                    </tr>
                </thead>
                <tbody>'' + ISNULL(@TableRows, N'''') + N''</tbody>
            </table>
        </div>
    </body>
    </html>'';

    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = N''Papercut_Local_Profile'',
        @recipients   = N''dba@anywhere.example'',
        @subject      = @Subject,
        @body         = @HtmlBody,
        @body_format  = N''HTML'';

    UPDATE msdb.dbo.DBA_TraceWatermark
    SET LastEventTime = (SELECT MAX(StartTime) FROM #DbEvents)
    WHERE CheckKey = ''DATABASE_DDL'';
END
';
GO

-- ============================================================================
-- STEP 2: Notify when a Database User is created or deleted (and who did it)
-- ============================================================================
EXEC dbo.sp_add_jobstep
    @job_name          = N'DatabaseDML',
    @step_name         = N'Notify Database User Create or Delete',
    @step_id           = 2,
    @subsystem         = N'TSQL',
    @database_name     = N'msdb',
    @on_success_action = 1, -- Quit with success
    @command           = N'
SET NOCOUNT ON;

DECLARE @TracePath NVARCHAR(260);
SELECT @TracePath = [path] FROM sys.traces WHERE is_default = 1;

IF @TracePath IS NULL RETURN;

DECLARE @LastChecked DATETIME2(3);
SELECT @LastChecked = LastEventTime
FROM msdb.dbo.DBA_TraceWatermark
WHERE CheckKey = ''DBUSER_DDL'';

IF OBJECT_ID(N''tempdb..#UserEvents'') IS NOT NULL DROP TABLE #UserEvents;

SELECT
    CASE t.EventSubClass
        WHEN 1 THEN N''USER CREATED (Add)''
        WHEN 2 THEN N''USER DELETED (Drop)''
        WHEN 3 THEN N''GRANT DB ACCESS''
        WHEN 4 THEN N''REVOKE DB ACCESS''
        ELSE CONCAT(N''SubClass '', t.EventSubClass)
    END AS Operation,
    ISNULL(t.DatabaseName, N''-'')                         AS DatabaseName,
    COALESCE(t.TargetUserName, t.TargetLoginName, N''-'')  AS TargetDatabaseUser,
    ISNULL(t.LoginName, N''-'')                            AS EffectiveLogin,
    ISNULL(t.SessionLoginName, N''-'')                     AS OriginalLogin,
    ISNULL(t.HostName, N''-'')                             AS HostName,
    ISNULL(t.ApplicationName, N''-'')                      AS ApplicationName,
    t.StartTime
INTO #UserEvents
FROM sys.fn_trace_gettable(@TracePath, DEFAULT) t
WHERE t.EventClass = 109               -- 109 = Audit Add DB User Event
  AND t.EventSubClass IN (1, 2, 3, 4)
  AND t.StartTime > @LastChecked;

IF EXISTS (SELECT 1 FROM #UserEvents)
BEGIN
    DECLARE @TableRows NVARCHAR(MAX);
    DECLARE @HtmlBody  NVARCHAR(MAX);
    DECLARE @Subject   NVARCHAR(255) = N''[SQL Audit] Database User Create/Delete Detected on '' + @@SERVERNAME;

    SELECT @TableRows = CAST((
        SELECT
            td = Operation,                                       N'''',
            td = DatabaseName,                                    N'''',
            td = TargetDatabaseUser,                              N'''',
            td = EffectiveLogin,                                  N'''',
            td = OriginalLogin,                                   N'''',
            td = HostName,                                        N'''',
            td = ApplicationName,                                 N'''',
            td = CONVERT(NVARCHAR(23), StartTime, 121)
        FROM #UserEvents
        ORDER BY StartTime ASC
        FOR XML PATH(''tr''), TYPE
    ) AS NVARCHAR(MAX));

    SET @HtmlBody = N''
    <html>
    <head>
        <style>
            body { font-family: Segoe UI, Arial, sans-serif; color: #1f2937; background-color: #f3f4f6; margin: 0; padding: 20px; }
            .card { background: #ffffff; border: 1px solid #d1d5db; border-radius: 8px; padding: 20px; max-width: 1050px; }
            h2 { color: #0f766e; margin-top: 0; }
            p.sub { color: #4b5563; font-size: 13px; }
            table { border-collapse: collapse; width: 100%; font-size: 13px; margin-top: 12px; }
            th { background-color: #115e59; color: #ffffff; text-align: left; padding: 10px 12px; border: 1px solid #134e4a; }
            td { padding: 8px 12px; border: 1px solid #e5e7eb; }
            tr:nth-child(even) td { background-color: #f0fdfa; }
        </style>
    </head>
    <body>
        <div class="card">
            <h2>Database User Creation / Deletion Audit</h2>
            <p class="sub">Server: <strong>'' + @@SERVERNAME + N''</strong> | Tracked via Default Trace (EventClass 109)</p>
            <table>
                <thead>
                    <tr>
                        <th>Operation</th>
                        <th>Database</th>
                        <th>Target User</th>
                        <th>Effective Login</th>
                        <th>Original Login</th>
                        <th>Host</th>
                        <th>Application</th>
                        <th>Timestamp</th>
                    </tr>
                </thead>
                <tbody>'' + ISNULL(@TableRows, N'''') + N''</tbody>
            </table>
        </div>
    </body>
    </html>'';

    EXEC msdb.dbo.sp_send_dbmail
        @profile_name = N''Papercut_Local_Profile'',
        @recipients   = N''dba@anywhere.example'',
        @subject      = @Subject,
        @body         = @HtmlBody,
        @body_format  = N''HTML'';

    UPDATE msdb.dbo.DBA_TraceWatermark
    SET LastEventTime = (SELECT MAX(StartTime) FROM #UserEvents)
    WHERE CheckKey = ''DBUSER_DDL'';
END
';
GO

EXEC dbo.sp_add_jobserver
    @job_name    = N'DatabaseDML',
    @server_name = N'(local)';
GO
```

*(Optional: Attach a recurring SQL Agent schedule—e.g., every 1 or 5 minutes—if you want `DatabaseDML` to run continuously in the background rather than manually.)*

---

### Step 6.3 — Test `DatabaseDML` Using Identity Impersonation (`John`)
**Task:**
1. As a `sysadmin`, create a server login named **`John`** that can create and delete databases.
2. Switch your identity to **`John`** and create a new database as if you were `John`.
3. Revert back to your initial identity.
4. Also test deleting the database and creating/deleting a database user, then execute `DatabaseDML` and verify the HTML emails in Papercut SMTP.

#### How to do it
```sql
USE [master];
GO

-- 1. Create SQL Login 'John' (requires Mixed Mode Auth if John were logging in interactively,
--    though sysadmin impersonation via EXECUTE AS works even under Windows-only mode!)
IF SUSER_ID(N'John') IS NULL
BEGIN
    CREATE LOGIN [John]
    WITH PASSWORD = N'StrongLabPass!2026',
         CHECK_POLICY = ON;
END
GO

-- Grant John permission to create and delete databases (dbcreator server role)
ALTER SERVER ROLE [dbcreator] ADD MEMBER [John];
GO

-- Check identity BEFORE switching
SELECT SUSER_SNAME() AS EffectiveLogin, ORIGINAL_LOGIN() AS OriginalLogin;

-- 2. Fake your SQL Server identity (Impersonate John)
EXECUTE AS LOGIN = N'John';
GO

-- Confirm we are now acting as John (OriginalLogin still records the real sysadmin!)
SELECT SUSER_SNAME() AS EffectiveLogin, ORIGINAL_LOGIN() AS OriginalLogin;

-- Create a new database as John
CREATE DATABASE [JohnsLabDB];
GO

-- 3. Revert back to your initial sysadmin identity
REVERT;
GO

SELECT SUSER_SNAME() AS EffectiveLoginAfterRevert, ORIGINAL_LOGIN() AS OriginalLogin;
GO

-- 4. Test Database User Create & Delete inside JohnsLabDB
USE [JohnsLabDB];
GO
CREATE USER [LabAppUser] WITHOUT LOGIN;
DROP USER [LabAppUser];
GO

-- 5. Also test Database Delete (as John, or as yourself)
USE [master];
GO
EXECUTE AS LOGIN = N'John';
DROP DATABASE [JohnsLabDB];
REVERT;
GO

-- 6. Run the DatabaseDML job and check Papercut SMTP!
EXEC msdb.dbo.sp_start_job @job_name = N'DatabaseDML';
GO
```

#### What it achieves
Demonstrates `EXECUTE AS LOGIN` impersonation and proves that the `DatabaseDML` job detects both database and user create/delete events—while exposing **both** the impersonated `EffectiveLogin` (`John`) and the real `OriginalLogin` (your Windows sysadmin account) in the HTML email table!

#### Key Learnings & Traps
- **Why `dbcreator` (vs. `CREATE ANY DATABASE` alone):** Adding `John` to the `dbcreator` fixed server role allows `John` to `CREATE`, `ALTER`, `DROP`, and `RESTORE` databases. If you only run `GRANT CREATE ANY DATABASE TO [John]`, `John` can create databases and drop databases *he owns*, but cannot drop databases owned by other logins unless granted `ALTER ANY DATABASE`.
- **Impersonation Security & Auditing (`EXECUTE AS`):** A `sysadmin` can impersonate any server login using `EXECUTE AS LOGIN = N'...'` without knowing their password. However, SQL Server never hides the true caller: `ORIGINAL_LOGIN()` and Default Trace's `SessionLoginName` still record the underlying authenticated session.
- **Default Trace Rollover Caveat:** Default Trace maintains up to 5 rolling 20 MB `.trc` files (`100 MB` total). On high-traffic production servers, events can roll off quickly—for long-term compliance auditing, use **SQL Server Audit** or **Extended Events**.

---

### Step 6.4 — Theory Question: Receiving Critical Alerts When Company Mail Services Are Offline
**Question:** *You rely on mail services for business critical notifications. How do you receive critical information about SQL Server and databases if the company's mail services is offline?*

- **Answer:** Never rely on a single corporate SMTP server as a single point of failure for critical database alerts:
  1. **Failover SMTP Account / Local Store-and-Forward Relay:**
     - A Database Mail **Profile** supports multiple **Accounts** with sequence numbers (`@sequence_number = 1, 2, ...`). Add a secondary/external SMTP gateway as sequence `2`, or route through a local SMTP smart-host relay that spools messages on disk and retries delivery when corporate mail recovers.
  2. **Out-of-Band Monitoring & Non-Email Alert Channels:**
     - Use dedicated monitoring tools (e.g., Redgate SQL Monitor, SolarWinds DPA, Datadog, SCOM, Zabbix, or Prometheus + Alertmanager) that poll SQL Server health independently and dispatch alerts via **SMS / PagerDuty / Opsgenie**, **Microsoft Teams / Slack webhooks**, **mobile push notifications**, or **SNMP traps**—all of which bypass corporate Exchange/SMTP entirely.
  3. **Windows Application Event Log & SQL Server Error Log Forwarding:**
     - Configure SQL Server Agent Alerts (`sp_add_alert`) and `DBCC CHECKDB` jobs (`WITH LOG`) to write critical errors to the **Windows Application Event Log** and **SQL Server Error Log**. Endpoint log shippers (e.g., Azure Monitor Agent, Splunk, Elastic/Winlogbeat) forward those logs out-of-band even when SMTP is dead.
  4. **Local Persistent Audit Table & Dead-Letter Checks:**
     - Log critical findings to a local DBA utility table before calling `sp_send_dbmail`, and monitor `msdb.dbo.sysmail_faileditems` / `sysmail_unsentitems` so alerts aren't silently lost during an outage.

---

## Part 7: Self-Study — Steve Stedman's Database Corruption Challenges

**Reference:** `https://stevestedman.com/server-health/database-corruption-challenge/`

Steve Stedman's **Database Corruption Challenge** is a classic set of 10+ real-world corruption scenarios (corrupted clustered/non-clustered indexes, damaged `LOB` pages, corrupt `sys` metadata, damaged transaction logs, etc.) originally built for hands-on DBA training.

#### How to Safely Practice in Your Lab
1. **Download a challenge `.bak` file** (start with *Week 1* or *Week 2*—note that older challenges were backed up on earlier SQL Server versions and restore cleanly onto SQL Server 2025 with `WITH MOVE`).
2. **Restore it under an isolated name** into `C:\Exo\Database` (inspect `RESTORE FILELISTONLY` first!).
3. **Run your new `CheckDB` SQL Agent job** (from Step 6.1) or `DBCC CHECKDB (...) WITH NO_INFOMSGS, ALL_ERRORMSGS` to see the exact corruption report and receive the HTML alert in Papercut SMTP.
4. **Practice zero-data-loss recovery techniques** before ever resorting to `REPAIR_ALLOW_DATA_LOSS`:
   - Inspect damaged 8 KB pages with `DBCC TRACEON (3604); DBCC PAGE (DBName, FileId, PageId, 3);`
   - Determine whether the corruption is only in a **Non-Clustered Index** (fixable by rebuilding/recreating the index) vs. a **Clustered Index / Heap** (fixable by restoring from a clean backup + log tail, or extracting uncorrupted rows / reconstructing from non-clustered covering indexes).
5. **Drop the challenge database** when finished so your `CheckDB` job returns to a clean baseline.

---

## Part 8: System-Versioned Temporal Tables (`AdventureWorksLT2022` History Table)

A **System-Versioned Temporal Table** automatically maintains a full historical audit trail of every `UPDATE` and `DELETE` in a paired **History Table**, letting you query data *"as of"* any point in time (`FOR SYSTEM_TIME AS OF ...`).

### Step 8.1 — T-SQL Eligibility Check Script Across All Tables in a Database
**Task:** Create a T-SQL script that accepts a database variable, loops through all tables in that database, checks if each table meets the requirements for a system-versioned history table, and outputs a list of eligible table names **prefixed with schema name**.

#### Requirements Checked by the Script
For an existing table in SQL Server to be converted into a **system-versioned temporal table** (with a paired history table):
1. Must be a **user table** (`is_ms_shipped = 0`) and **not already temporal or a history table** (`temporal_type = 0`).
2. Must have a **Primary Key** defined (`OBJECTPROPERTY(object_id, 'TableHasPrimaryKey') = 1`).
3. Must **not** be a `FILETABLE` (`is_filetable = 0`) and must **not** contain `FILESTREAM` columns (`filestream_data_space_id IS NULL` and no `is_filestream = 1` column).
4. Must **not** be an external table (`is_external = 0`), graph node/edge table (`is_node = 0 AND is_edge = 0`), or ledger table (`ledger_type = 0`).
5. Must **not** have any **`INSTEAD OF` triggers** (`is_instead_of_trigger = 1`), because `INSTEAD OF` triggers are prohibited on temporal tables.
*(Note: In SQL Server 2017–2025, `ON DELETE/UPDATE CASCADE` foreign keys and computed columns are supported on the current table, though computed columns become regular static columns in the history table.)*

#### How to do it
```sql
USE [master];
GO

SET NOCOUNT ON;

-- ============================================================================
-- INPUT VARIABLE: Set target database name
-- ============================================================================
DECLARE @DatabaseName SYSNAME = N'AdventureWorksLT2022';

IF DB_ID(@DatabaseName) IS NULL
BEGIN
    RAISERROR(N'Database "%s" does not exist.', 16, 1, @DatabaseName);
    RETURN;
END

-- Staging table for all tables discovered in @DatabaseName
IF OBJECT_ID(N'tempdb..#TableList') IS NOT NULL DROP TABLE #TableList;
CREATE TABLE #TableList (
    RowID          INT IDENTITY(1,1) PRIMARY KEY,
    SchemaName     SYSNAME,
    TableName      SYSNAME,
    FullTableName  NVARCHAR(300),
    IsEligible     BIT DEFAULT 0,
    IneligibleReason NVARCHAR(500) NULL
);

DECLARE @Sql NVARCHAR(MAX);

-- Populate table list from target database
SET @Sql = N'
SELECT
    s.name AS SchemaName,
    t.name AS TableName,
    s.name + N''.'' + t.name AS FullTableName
FROM ' + QUOTENAME(@DatabaseName) + N'.sys.tables t
JOIN ' + QUOTENAME(@DatabaseName) + N'.sys.schemas s ON t.schema_id = s.schema_id
WHERE t.is_ms_shipped = 0
ORDER BY s.name, t.name;';

INSERT INTO #TableList (SchemaName, TableName, FullTableName)
EXEC (@Sql);

-- Loop through every table in @DatabaseName and evaluate temporal requirements
DECLARE @CurrentRowID  INT = 1;
DECLARE @MaxRowID      INT = ISNULL((SELECT MAX(RowID) FROM #TableList), 0);
DECLARE @SchemaName    SYSNAME;
DECLARE @TableName     SYSNAME;
DECLARE @HasPK         BIT;
DECLARE @TemporalType  TINYINT;
DECLARE @IsUnsupported BIT;
DECLARE @HasInsteadOf  BIT;
DECLARE @Reason        NVARCHAR(500);

WHILE @CurrentRowID <= @MaxRowID
BEGIN
    SELECT @SchemaName = SchemaName, @TableName = TableName
    FROM #TableList
    WHERE RowID = @CurrentRowID;

    SET @HasPK         = 0;
    SET @TemporalType  = 0;
    SET @IsUnsupported = 0;
    SET @HasInsteadOf  = 0;
    SET @Reason        = N'';

    SET @Sql = N'
    SELECT
        @pHasPK = CASE WHEN EXISTS (
            SELECT 1 FROM ' + QUOTENAME(@DatabaseName) + N'.sys.key_constraints kc
            WHERE kc.parent_object_id = t.object_id AND kc.type = ''PK''
        ) THEN 1 ELSE 0 END,
        @pTemporalType = t.temporal_type,
        @pIsUnsupported = CASE WHEN (
            t.is_filetable = 1
            OR t.is_external = 1
            OR t.is_node = 1
            OR t.is_edge = 1
            OR t.ledger_type <> 0
            OR t.filestream_data_space_id IS NOT NULL
            OR EXISTS (
                SELECT 1 FROM ' + QUOTENAME(@DatabaseName) + N'.sys.columns c
                WHERE c.object_id = t.object_id AND c.is_filestream = 1
            )
        ) THEN 1 ELSE 0 END,
        @pHasInsteadOf = CASE WHEN EXISTS (
            SELECT 1 FROM ' + QUOTENAME(@DatabaseName) + N'.sys.triggers tr
            WHERE tr.parent_id = t.object_id AND tr.is_instead_of_trigger = 1
        ) THEN 1 ELSE 0 END
    FROM ' + QUOTENAME(@DatabaseName) + N'.sys.tables t
    JOIN ' + QUOTENAME(@DatabaseName) + N'.sys.schemas s ON t.schema_id = s.schema_id
    WHERE s.name = @pSchemaName AND t.name = @pTableName;';

    EXEC sp_executesql
        @Sql,
        N'@pSchemaName SYSNAME, @pTableName SYSNAME, @pHasPK BIT OUTPUT, @pTemporalType TINYINT OUTPUT, @pIsUnsupported BIT OUTPUT, @pHasInsteadOf BIT OUTPUT',
        @pSchemaName    = @SchemaName,
        @pTableName     = @TableName,
        @pHasPK         = @HasPK OUTPUT,
        @pTemporalType  = @TemporalType OUTPUT,
        @pIsUnsupported = @IsUnsupported OUTPUT,
        @pHasInsteadOf  = @HasInsteadOf OUTPUT;

    IF @TemporalType <> 0
        SET @Reason += N'Already a temporal or history table; ';
    IF @HasPK = 0
        SET @Reason += N'Missing Primary Key; ';
    IF @IsUnsupported = 1
        SET @Reason += N'Uses FileTable/FILESTREAM/External/Graph/Ledger; ';
    IF @HasInsteadOf = 1
        SET @Reason += N'Has INSTEAD OF trigger; ';

    UPDATE #TableList
    SET IsEligible       = CASE WHEN LEN(@Reason) = 0 THEN 1 ELSE 0 END,
        IneligibleReason = NULLIF(@Reason, N'')
    WHERE RowID = @CurrentRowID;

    SET @CurrentRowID += 1;
END

-- Required Output: Schema-prefixed list of tables where a history table can be applied
SELECT FullTableName AS [EligibleTables_For_HistoryTable]
FROM #TableList
WHERE IsEligible = 1
ORDER BY FullTableName;

-- Optional Diagnostic View: See all tables and any blockers
-- SELECT FullTableName, IsEligible, IneligibleReason FROM #TableList ORDER BY FullTableName;
GO
```

#### Expected Result on `AdventureWorksLT2022`
Every standard table in `AdventureWorksLT2022` (`dbo.BuildVersion`, `dbo.ErrorLog`, `SalesLT.Address`, `SalesLT.Customer`, `SalesLT.CustomerAddress`, `SalesLT.Product`, `SalesLT.ProductCategory`, `SalesLT.ProductDescription`, `SalesLT.ProductModel`, `SalesLT.ProductModelProductDescription`, `SalesLT.SalesOrderDetail`, `SalesLT.SalesOrderHeader`, plus `SalesLT.TableA` from Step 1.8) has a Primary Key and no `FILESTREAM` or `INSTEAD OF` triggers, so all of them appear in `[EligibleTables_For_HistoryTable]`.

---

### Step 8.2 — Enable System-Versioning (History Table) on `SalesLT.Product`
**Task:** Choose one table from the eligible list (e.g., `SalesLT.Product`) and create a history table for it.

#### How to do it
Because `SalesLT.Product` already contains existing rows, we:
1. Add the two mandatory `DATETIME2` period columns (`ValidFrom` and `ValidTo`) with `DEFAULT` constraints (and `HIDDEN` so existing `SELECT *` queries continue returning the original columns seamlessly).
2. Define `PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)`.
3. Enable `SYSTEM_VERSIONING = ON` and explicitly name the history table `SalesLT.ProductHistory` (so SQL Server doesn't generate a random name like `MSSQL_TemporalHistoryFor_...`).

```sql
USE [AdventureWorksLT2022];
GO

-- 1. Add temporal period columns with defaults for existing rows, and mark them HIDDEN
ALTER TABLE SalesLT.Product
ADD
    ValidFrom DATETIME2 GENERATED ALWAYS AS ROW START HIDDEN NOT NULL
        CONSTRAINT DF_Product_ValidFrom DEFAULT SYSUTCDATETIME(),
    ValidTo   DATETIME2 GENERATED ALWAYS AS ROW END   HIDDEN NOT NULL
        CONSTRAINT DF_Product_ValidTo   DEFAULT CONVERT(DATETIME2, '9999-12-31 23:59:59.9999999'),
    PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo);
GO

-- 2. Enable System-Versioning and create SalesLT.ProductHistory
ALTER TABLE SalesLT.Product
SET (SYSTEM_VERSIONING = ON (HISTORY_TABLE = SalesLT.ProductHistory));
GO

-- 3. Verify temporal metadata in sys.tables
SELECT
    SCHEMA_NAME(t.schema_id) + N'.' + t.name AS TableName,
    t.temporal_type_desc,
    OBJECT_SCHEMA_NAME(t.history_table_id) + N'.' + OBJECT_NAME(t.history_table_id) AS HistoryTableName
FROM sys.tables t
WHERE t.object_id = OBJECT_ID(N'SalesLT.Product');
GO
```

#### What it achieves
Turns `SalesLT.Product` into a live System-Versioned Temporal Table backed by `SalesLT.ProductHistory`. Any future `UPDATE` or `DELETE` automatically archives the previous row version into `SalesLT.ProductHistory` with exact UTC validity timestamps.

#### Key Learnings & Traps
- **Why `HIDDEN`?** Marking `ValidFrom` and `ValidTo` as `HIDDEN` keeps `SELECT * FROM SalesLT.Product` compatible with existing applications/reports; you can still explicitly select `ValidFrom, ValidTo` whenever you want to see them.
- **Why `DEFAULT` constraints are required here:** You cannot add `NOT NULL` `GENERATED ALWAYS AS ROW START/END` columns to a non-empty table without `DEFAULT` values to backfill existing rows.
- **History Table Constraints:** `SalesLT.ProductHistory` automatically mirrors the column types of `SalesLT.Product` (with a clustered index on `(ValidTo, ValidFrom)` created by SQL Server) and has **no Primary Key, Foreign Keys, or triggers**, because multiple historical versions of the same `ProductID` will live inside it.

---

### Step 8.3 — Test the Temporal / History Table Solution
**Task:** Test `INSERT`, `UPDATE`, `DELETE`, and point-in-time temporal queries against `SalesLT.Product` and `SalesLT.ProductHistory`.

#### How to do it
*(Using a dedicated test row so we don't disturb foreign keys on existing AdventureWorks products!)*
```sql
USE [AdventureWorksLT2022];
GO

-- 1. Insert a new test product
INSERT INTO SalesLT.Product (Name, ProductNumber, Color, StandardCost, ListPrice, SellStartDate)
VALUES (N'Temporal Test Bike', N'TT-BIKE-001', N'Black', 500.00, 999.00, SYSUTCDATETIME());

DECLARE @TestProductID INT = SCOPE_IDENTITY();
WAITFOR DELAY '00:00:02';

-- Capture a timestamp while ListPrice was 999.00
DECLARE @TimeWhen999 DATETIME2 = SYSUTCDATETIME();
WAITFOR DELAY '00:00:02';

-- 2. Update the product price (archives the 999.00 row into SalesLT.ProductHistory!)
UPDATE SalesLT.Product
SET ListPrice = 1299.00, Color = N'Red'
WHERE ProductID = @TestProductID;

WAITFOR DELAY '00:00:02';

-- 3. Delete the test product (archives the 1299.00 row into SalesLT.ProductHistory!)
DELETE FROM SalesLT.Product
WHERE ProductID = @TestProductID;

-- 4. Time-travel query: see the product as it existed at @TimeWhen999 (before update & delete)
SELECT ProductID, Name, Color, ListPrice, ValidFrom, ValidTo
FROM SalesLT.Product
FOR SYSTEM_TIME AS OF @TimeWhen999
WHERE ProductID = @TestProductID;
GO

-- 5. Inspect current table (0 rows for TT-BIKE-001) vs. History Table (2 historical versions!)
SELECT ProductID, Name, Color, ListPrice
FROM SalesLT.Product
WHERE ProductNumber = N'TT-BIKE-001';

SELECT ProductID, Name, Color, ListPrice, ValidFrom, ValidTo
FROM SalesLT.ProductHistory
WHERE ProductNumber = N'TT-BIKE-001'
ORDER BY ValidFrom;
GO

-- 6. Query across BOTH current + history automatically using FOR SYSTEM_TIME ALL
SELECT ProductID, Name, Color, ListPrice, ValidFrom, ValidTo
FROM SalesLT.Product
FOR SYSTEM_TIME ALL
WHERE ProductNumber = N'TT-BIKE-001'
ORDER BY ValidFrom;
GO
```

#### Key Learnings & Intentional Traps
- **Trap (`DROP TABLE` Blocked):** While `SYSTEM_VERSIONING = ON`, SQL Server intentionally blocks `DROP TABLE SalesLT.Product`, `TRUNCATE TABLE SalesLT.Product`, and direct `INSERT/UPDATE/DELETE` on `SalesLT.ProductHistory`. To drop or alter a history table, you must first run `ALTER TABLE SalesLT.Product SET (SYSTEM_VERSIONING = OFF);`.
- **UTC Timestamps:** `ValidFrom` and `ValidTo` always use **UTC** (`SYSUTCDATETIME()`), not local server time (`GETDATE()`). Keep this in mind when passing timestamps to `FOR SYSTEM_TIME AS OF`!

