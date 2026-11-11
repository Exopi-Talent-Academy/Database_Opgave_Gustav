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
ALTER DATABASE [AdventureWorksLT2022]
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
ALTER DATABASE [AdventureWorksLT2022]
SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
GO

-- 1. Restore the FULL backup WITH NORECOVERY and REPLACE
RESTORE DATABASE [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Full.bak'
WITH REPLACE, NORECOVERY, STATS = 10;
GO

-- 2. Restore the initial test .trn file from Step 1.7 WITH NORECOVERY
RESTORE LOG [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_142800.trn'
WITH NORECOVERY;
GO

-- 3. Restore the .trn file taken after TableA WITH NORECOVERY
RESTORE LOG [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_143010.trn'
WITH NORECOVERY;
GO

-- 4. Restore the final .trn file (which contains TableB) WITH STOPAT and RECOVERY!
RESTORE LOG [AdventureWorksLT2022]
FROM DISK = N'C:\Exo\Database\AdventureWorksLT2022_Log_20261111_143115.trn'
WITH STOPAT = N'2026-11-11 14:30:30.000', -- Timestamp between TableA and TableB
     RECOVERY;
GO

-- Bring database back to MULTI_USER mode
ALTER DATABASE [AdventureWorksLT2022]
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
DECLARE @StopAt         DATETIME      = ''2026-11-11 14:30:30.000'';

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
IF DB_ID(@DatabaseName) IS NOT NULL
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
