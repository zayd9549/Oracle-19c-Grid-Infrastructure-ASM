
## Oracle 19c Grid Infrastructure & ASM – Complete Practical Guide

This document is a **step-by-step practical guide** to **Oracle ASM (Automatic Storage Management)** in **Oracle Grid Infrastructure 19c**, including **disk preparation, ASMCA, ASMCMD, manual SQL diskgroup operations, adding/dropping disks, rebalancing, and Failure Groups**.

---

## 🔑 Logging in to ASM Instance

⚠️ **Pre-requisites**

* ASM instance is running.
* OS user `grid` (or equivalent) exists.
* User has `SYSASM` privileges.

💻 **Commands**

```bash
# Switch to Grid user
su - grid

# Connect to ASM instance with SYSASM
export ORACLE_SID=+ASM
sqlplus / as sysasm
```

## 📊 ASM Monitoring Views

⚠️ **Pre-requisites**

* ASM instance running.
* Diskgroup exists.

💻 **Commands**

```sql
-- Diskgroup info
SET LINES 200
COL NAME FORMAT A15
COL TYPE FORMAT A10
COL STATE FORMAT A10
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB FROM V$ASM_DISKGROUP;

-- Disk info with FGs
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15
SELECT NAME AS DISK_NAME, PATH, FAILGROUP, HEADER_STATUS, STATE
FROM V$ASM_DISK;

-- ASM clients
SELECT INSTANCE_NAME, DB_NAME, STATUS FROM V$ASM_CLIENT;

-- Rebalance operations
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;
```

## ⚠️ Preparing Disks for ASM

⚠️ **Pre-requisites**

* Identify new raw disks to be added.
* Disks must **not be members** of any existing diskgroup.
* Disks must be **Candidate or Provisioned**.
* ASM instance must be running.

💻 **Commands**

```bash
# List all attached disks at OS level
fdisk -l | grep sd
lsblk

# List disks already registered with ASM
oracleasm listdisks

# Check if disk is part of existing diskgroup
sqlplus / as sysasm

SET LINES 200
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15
SELECT PATH, HEADER_STATUS, NAME AS DISKGROUP_NAME
FROM V$ASM_DISK;

# Register new disks with ASM
oracleasm createdisk DISK1 /dev/sdb
oracleasm createdisk DISK2 /dev/sdc

# Verify registration
oracleasm listdisks
```

---

## 📦 Creating a Diskgroup

⚠️ **Pre-requisites**

* ASM instance is running and ONLINE.
* Candidate or Provisioned disks are available.
* Disks are not members of any diskgroup.
* Decide on redundancy: External, Normal, or High.

💻 **Commands**

```sql
-- Check candidate/provisioned disks
SET LINES 200
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15
SELECT PATH, HEADER_STATUS, TOTAL_MB, FREE_MB
FROM V$ASM_DISK
WHERE HEADER_STATUS IN ('CANDIDATE','PROVISIONED');

-- Create diskgroup (example: external redundancy)
CREATE DISKGROUP DATA EXTERNAL REDUNDANCY
DISK '/dev/oracleasm/disks/DISK1' NAME DATA1,
     '/dev/oracleasm/disks/DISK2' NAME DATA2;

-- Verify diskgroup
SET LINES 200
COL NAME FORMAT A15
COL TYPE FORMAT A10
COL STATE FORMAT A10
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB
FROM V$ASM_DISKGROUP;
```

---

## 🛡️ Creating a Diskgroup with Failure Groups

⚠️ **Pre-requisites**

* ASM instance is running and ONLINE.
* Candidate or Provisioned disks are available.
* Diskgroup redundancy must be **Normal or High**.
* Disks must be assigned to **failure groups** based on physical layout.

💻 **Commands**

```sql
-- Create diskgroup with failure groups
CREATE DISKGROUP DATA NORMAL REDUNDANCY
FAILGROUP FG1 DISK '/dev/oracleasm/disks/DISK1' NAME DATA1,
              '/dev/oracleasm/disks/DISK2' NAME DATA2
FAILGROUP FG2 DISK '/dev/oracleasm/disks/DISK3' NAME DATA3,
              '/dev/oracleasm/disks/DISK4' NAME DATA4;

-- Verify diskgroup and FGs
SELECT GROUP_NUMBER, NAME AS DISKGROUP_NAME, TYPE, STATE, TOTAL_MB, FREE_MB
FROM V$ASM_DISKGROUP;

SELECT PATH, NAME AS DISK_NAME, FAILGROUP, HEADER_STATUS, STATE
FROM V$ASM_DISK;
```

⚙️ **Clarifications**

* Normal redundancy: 2 copies stored in different FGs.
* High redundancy: 3 copies in different FGs.
* External redundancy ignores FGs.

---

## ➕ Adding a Disk to Diskgroup / Failure Group

⚠️ **Pre-requisites**

* Diskgroup is ONLINE.
* Disk is Candidate/Provisioned.
* Assign disk to a proper FG.

💻 **Commands**

```sql
-- Add disk to specific failure group
ALTER DISKGROUP DATA ADD DISK '/dev/oracleasm/disks/DISK5' NAME DATA5 FAILGROUP FG1;

-- Verify disk and FG
SELECT PATH, NAME AS DISK_NAME, FAILGROUP, HEADER_STATUS, STATE
FROM V$ASM_DISK;
```

⚙️ **Notes**

* Adding disks to different FGs improves redundancy and rebalance efficiency.

---

## ➖ Dropping a Disk from Diskgroup / Failure Group

⚠️ **Pre-requisites**

* Diskgroup is ONLINE.
* Disk can be safely removed.
* Remaining disks in other FGs maintain redundancy.

💻 **Commands**

```sql
-- Drop disk
ALTER DISKGROUP DATA DROP DISK DATA2;

-- Verify diskgroup and FGs
SELECT PATH, NAME AS DISK_NAME, FAILGROUP, HEADER_STATUS, STATE
FROM V$ASM_DISK;
```

⚙️ **Notes**

* ASM automatically rebalances remaining data across other FGs.

---

## ⚖️ Rebalancing Diskgroup / Failure Group

⚠️ **Pre-requisites**

* Diskgroup is ONLINE.
* No conflicting operations ongoing.

💻 **Commands**

```sql
-- Start rebalance with specified power
ALTER DISKGROUP DATA REBALANCE POWER 5;

-- Monitor rebalance
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;

-- Pause or cancel rebalance if needed
ALTER DISKGROUP DATA REBALANCE CANCEL;
```

⚙️ **Notes**

* ASM redistributes mirrored extents across different FGs.
* Proper FG assignment reduces risk of data loss in hardware failures.

---

## 🐚 ASMCMD – Practical CLI Usage

⚠️ **Pre-requisites**

* ASM instance is ONLINE.
* Diskgroup exists.

💻 **Commands**

```bash
# Start persistent ASMCMD shell
asmcmd -p

# List all diskgroups
lsdg

# List all disks
lsdsk

# Navigate diskgroup
cd DATA

# List files/directories
ls
du

# Create directory and files
mkdir arch
touch arch/file1.dbf

# Copy, move, remove files
cp arch/file1.dbf arch/file2.dbf
mv arch/file2.dbf arch/file2_old.dbf
rm arch/file1.dbf

# Show diskgroup storage parameters
sp DATA

# Exit ASMCMD
exit
```

---



