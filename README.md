# MySQL Database Migration Guide
### Olist Portfolio Project - Laptop-to-Laptop Transfer

> **Context:** Migrating the Olist Brazilian E-Commerce dataset and schema from a primary development machine to a secondary machine running MySQL 8.0.45 on Windows 10.

---

## Table of Contents

1. [Overview & Decision Matrix](#1-overview--decision-matrix)
2. [Option A - Manual Export/Import (mysqldump)](#2-option-a--manual-exportimport-mysqldump)
3. [Option B - MySQL Workbench Data Export/Import](#3-option-b--mysql-workbench-data-exportimport)
4. [Option C - Direct File Copy (Data Directory)](#4-option-c--direct-file-copy-data-directory)
5. [Option D - OCI MySQL HeatWave (Cloud Staging)](#5-option-d--oci-mysql-heatwave-cloud-staging)
6. [Option E - Shared Network Drive / USB Transfer](#6-option-e--shared-network-drive--usb-transfer)
7. [Option F - GitHub as an Intermediary](#7-option-f--github-as-an-intermediary)
8. [Post-Migration Validation](#8-post-migration-validation)
9. [Recommended Approach](#9-recommended-approach)

---

## 1. Overview & Decision Matrix

| Option | Difficulty | Speed | Requires Network | Best For |
|--------|------------|-------|-----------------|----------|
| A - mysqldump (CLI) | Low | Medium | No | Reliable, portable, version-safe |
| B - Workbench GUI | Very Low | Medium | No | Non-CLI users, visual control |
| C - Data directory copy | Medium | Fast | No | Same MySQL version on both machines |
| D - OCI HeatWave | Medium | Slow (upload/download) | Yes | Cloud backup + migration in one step |
| E - USB / Network share | Low | Fast | Optional | Quick local transfer |
| F - GitHub | Low | Slow for large data | Yes | Small schemas; keeps history in repo |

**Dataset size note:** The full Olist dataset (9 CSV files + schema) is approximately 120–150 MB uncompressed. All options below are viable at this size.

---

## 2. Option A - Manual Export/Import (mysqldump)

The most portable and reliable method. Works across MySQL versions and operating systems.

### Step 1 - Export on source machine

Open Command Prompt or PowerShell and run:

```bash
# Export entire database
mysqldump -u root -p olist_db > olist_backup.sql

# Export with additional safety flags (recommended)
mysqldump -u root -p \
  --single-transaction \
  --routines \
  --triggers \
  --add-drop-table \
  olist_db > olist_backup.sql
```

> `--single-transaction` ensures a consistent snapshot without locking tables.  
> `--routines` includes stored procedures and functions.  
> `--add-drop-table` makes the import idempotent (safe to re-run).

The output file `olist_backup.sql` will be created in your current directory.

### Step 2 - Transfer the file

Copy `olist_backup.sql` to the target machine via USB, shared folder, or any method from Options E/F.

### Step 3 - Import on target machine (Dell Inspiron)

```bash
# Create the target database first
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS olist_db;"

# Import the dump
mysql -u root -p olist_db < olist_backup.sql
```

**Expected output:** No errors. MySQL will print nothing on success.

### Pros
- Version-safe (SQL text format, not binary)
- Single portable file
- Human-readable - useful for portfolio/learning purposes

### Cons
- Slower than binary methods for very large databases
- Requires CLI familiarity

---

## 3. Option B - MySQL Workbench Data Export/Import

A GUI-based alternative to mysqldump. Produces the same `.sql` output but through a visual interface.

### Export (source machine)

1. Open **MySQL Workbench** → connect to your local instance
2. Go to **Server** → **Data Export**
3. Select `olist_db` from the schema list
4. Choose **"Export to Self-Contained File"** → set output path
5. Check **"Include Create Schema"**
6. Click **Start Export**

### Import (target machine - Dell Inspiron)

1. Open **MySQL Workbench** → connect to MySQL 8.0.45
2. Go to **Server** → **Data Import/Restore**
3. Select **"Import from Self-Contained File"** → browse to your `.sql` file
4. Under **Default Target Schema**, type `olist_db` (or create it first)
5. Click **Start Import**

### Pros
- No command line required
- Progress bar and error reporting built in
- Familiar interface for Workbench users

### Cons
- Slower for large databases compared to CLI
- Workbench must be installed on source machine

---

## 4. Option C - Direct File Copy (Data Directory)

Copies the raw MySQL data files instead of exporting SQL. **Only safe when both machines run the same MySQL version (8.0.45 in this case).**

### Locate the data directory

On the source machine, run in MySQL Workbench or CLI:

```sql
SHOW VARIABLES LIKE 'datadir';
```

Default on Windows: `C:\ProgramData\MySQL\MySQL Server 8.0\Data\`

### Steps

1. **Stop MySQL service** on the source machine:
   ```
   net stop MySQL80
   ```

2. **Copy the database folder:**  
   Navigate to the data directory and copy the entire `olist_db\` subfolder.

3. **Stop MySQL service** on the target machine:
   ```
   net stop MySQL80
   ```

4. **Paste** the `olist_db\` folder into the same data directory on the target machine.

5. **Start MySQL service** on the target machine:
   ```
   net start MySQL80
   ```

6. Verify in Workbench:
   ```sql
   SHOW DATABASES;
   USE olist_db;
   SHOW TABLES;
   ```

### Pros
- Fastest method
- No SQL generation required

### Cons
- **Version-sensitive** - only safe if both machines run identical MySQL versions
- Risk of corruption if service is not properly stopped first
- Not suitable for cross-OS or cross-version migration

---

## 5. Option D - OCI MySQL HeatWave (Cloud Staging)

Uses Oracle Cloud Infrastructure (OCI) as an intermediary. Exports from source → uploads to OCI HeatWave → imports on target. Also serves as a cloud backup.

> **Note:** Requires an OCI account. Free Tier includes MySQL HeatWave with limited resources.

### Architecture

```
Source Machine → mysqldump → OCI Object Storage → OCI MySQL HeatWave
                                                          ↓
Target Machine ← mysql import ← OCI MySQL HeatWave dump ←
```

### Steps

#### 1. Export from source (same as Option A)
```bash
mysqldump -u root -p --single-transaction olist_db > olist_backup.sql
```

#### 2. Upload to OCI Object Storage

Using OCI CLI (install from [docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm)):

```bash
oci os object put \
  --bucket-name your-bucket-name \
  --file olist_backup.sql \
  --name olist_backup.sql
```

#### 3. Create a MySQL HeatWave instance on OCI

- Go to OCI Console → **Databases** → **MySQL HeatWave**
- Click **Create DB System** → choose **Development / Testing** (free tier eligible)
- Note the endpoint hostname

#### 4. Import into HeatWave

```bash
mysql -h <heatwave-endpoint> -u admin -p \
  -e "CREATE DATABASE IF NOT EXISTS olist_db;"

mysql -h <heatwave-endpoint> -u admin -p olist_db < olist_backup.sql
```

#### 5. Export from HeatWave and import on target machine

```bash
# Dump from HeatWave
mysqldump -h <heatwave-endpoint> -u admin -p \
  --single-transaction olist_db > olist_from_cloud.sql

# Import on target (Dell Inspiron)
mysql -u root -p olist_db < olist_from_cloud.sql
```

### Pros
- Acts as cloud backup simultaneously
- Useful for OCI familiarity (relevant for Oracle support background)
- Can test HeatWave analytics features on the Olist dataset

### Cons
- Most complex option
- Requires OCI account and CLI setup
- Upload/download time depends on connection speed
- Overkill for a local laptop-to-laptop transfer

---

## 6. Option E - Shared Network Drive / USB Transfer

The simplest approach for physically co-located machines. Used in combination with Option A or B.

### Via USB

1. Run the `mysqldump` export (Option A, Step 1) on the source machine
2. Copy `olist_backup.sql` to a USB drive
3. Plug into the Dell Inspiron
4. Run the import (Option A, Step 3)

### Via Local Network Share (Windows)

1. On the **source machine**, right-click the folder containing `olist_backup.sql` → **Properties** → **Sharing** → **Share**
2. Note the network path (e.g., `\\SOURCEMACHINE\SharedFolder`)
3. On the **target machine**, open File Explorer → type the network path → copy the file
4. Run the import

### Pros
- No internet required
- Very fast for local network (gigabit = ~100 MB/s)
- Simple and reliable

### Cons
- Requires physical proximity or local network access
- Not scalable beyond local setups

---

## 7. Option F - GitHub as an Intermediary

Suitable for schema-only migrations or small datasets. Large data files should be handled via Git LFS or kept out of version control entirely.

### Schema only (recommended for portfolio)

Export just the structure, no data:

```bash
mysqldump -u root -p \
  --no-data \
  --routines \
  olist_db > schema_only.sql
```

Commit to your repository:

```bash
git add schema_only.sql
git commit -m "Add Olist DB schema export"
git push
```

On the target machine:

```bash
git pull
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS olist_db;"
mysql -u root -p olist_db < schema_only.sql
```

Then reload the raw CSVs from the [Olist Kaggle dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) and re-run your data loading scripts.

### With Git LFS (data included)

For files over 50 MB, GitHub requires [Git Large File Storage](https://git-lfs.com/):

```bash
git lfs install
git lfs track "*.sql"
git add .gitattributes
git add olist_backup.sql
git commit -m "Add full Olist DB dump via LFS"
git push
```

### Pros
- Keeps migration history in version control
- Target machine only needs `git pull`
- Schema file is readable and portfolio-friendly

### Cons
- GitHub has a 100 MB file limit without LFS
- LFS has storage/bandwidth quotas on free accounts
- Not ideal for frequent large data transfers

---

## 8. Post-Migration Validation

Run these checks on the target machine after any migration method to confirm data integrity.

```sql
-- 1. Confirm all tables are present
USE olist_db;
SHOW TABLES;

-- 2. Check row counts match source
SELECT 
    table_name,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'olist_db'
ORDER BY table_name;

-- 3. Spot-check a key table
SELECT COUNT(*) FROM olist_orders;
SELECT COUNT(*) FROM olist_order_items;

-- 4. Verify no import errors by checking for NULL in key columns
SELECT COUNT(*) FROM olist_orders WHERE order_id IS NULL;

-- 5. Confirm foreign key constraints are intact
SELECT 
    constraint_name,
    table_name,
    referenced_table_name
FROM information_schema.referential_constraints
WHERE constraint_schema = 'olist_db';
```

---

## 9. Recommended Approach

For this specific use case (Olist dataset, two local Windows machines, MySQL 8.0.45 on both):

**Primary recommendation: Option A (mysqldump) + Option E (USB)**

1. Run `mysqldump` on the source machine → produces `olist_backup.sql`
2. Copy to the Dell Inspiron via USB or local network share
3. Import with `mysql` CLI on the target
4. Run validation queries from Section 8

This approach is fast, reliable, version-safe, and requires no external accounts or services. The resulting `.sql` file is also worth committing to GitHub as a schema reference (Option F, schema-only variant) to demonstrate clean database documentation practice in your portfolio.

---

*Created: May 2026 | MySQL 8.0.45 | Windows 10 | Olist Brazilian E-Commerce Dataset*
