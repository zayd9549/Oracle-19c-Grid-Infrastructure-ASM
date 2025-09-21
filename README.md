
## Oracle 19c Grid Infrastructure & ASM – Practical Guide

This document is a **step-by-step practical guide** to **Oracle ASM (Automatic Storage Management)** in **Oracle Grid Infrastructure 19c**.
It covers ASM concepts, diskgroup management, ASMCA usage, manual SQL/CLI, adding/dropping disks, and rebalance operations, in proper practical sequence.

---

## 🔑 Logging in to ASM Instance

📘 **Definition**
ASM is Oracle’s **volume manager + filesystem** running in a separate **+ASM instance**.

💻 **Command**

```bash
su - grid
sqlplus / as sysasm
```

⚙️ **Notes**

* Always connect with `SYSASM` privileges.
* `SYSDBA` cannot perform storage operations.

---

## 🖥️ ASMCA – GUI Management

📘 **Definition**
ASMCA is a **graphical tool** for ASM administration: creating, modifying, or dropping diskgroups.

⚠️ **Pre-requisites**

* Disks must be **Candidate or Provisioned**.
* ASM instance must be **running and ONLINE**.

🚀 **Practical Sequence Using ASMCA**

1. **Launch ASMCA**

```bash
asmca
```

2. **Creating a Diskgroup**

   * Go to **File → Create Disk Group**.
   * Select **candidate or provisioned disks**.
   * Set **name**, **redundancy type** (External/Normal/High).
   * Click **OK**.

3. **Adding a Disk**

   * Select diskgroup → **Add Disks**.
   * Choose **candidate or provisioned disks**.
   * ASMCA automatically starts **rebalance**.

4. **Dropping a Disk**

   * Select diskgroup → **Drop Disk**.
   * ASM automatically rebalances remaining extents.

5. **Modifying Diskgroup**

   * Change **attributes** like rebalance power, compatibility, or redundancy.

⚙️ **Notes**

* ASMCA checks disk health automatically.
* Manual rebalance power adjustments are possible.

---

## 🐚 ASMCMD – CLI Navigation

📘 **Definition**
ASMCMD provides a **command-line interface** for ASM navigation and management.

💻 **Commands**

```bash
asmcmd
ASMCMD> lsdg       -- List all diskgroups
ASMCMD> lsdsk      -- List all disks
ASMCMD> cd DATA     -- Navigate into diskgroup
ASMCMD> du         -- Check disk usage
ASMCMD> rm file     -- Remove file from ASM
ASMCMD> exit
```

⚙️ **Notes**

* ASMCMD is useful for scripts and file-level checks without SQL.

---

## ⚠️ Preparing Disks for ASM

📘 **Definition**
Disks must be **Candidate or Provisioned** before ASM can use them.

⚠️ **Pre-requisites**

* Identify raw devices at OS level.
* Ensure disks are **not already members** of another diskgroup.

💻 **Commands**

```bash
# List OS disks
fdisk -l | grep sd

# Register with ASM
oracleasm createdisk DISK1 /dev/sdb
oracleasm createdisk DISK2 /dev/sdc

# Verify registered ASM disks
oracleasm listdisks
```

⚙️ **Notes**

* Only Candidate/Provisioned disks are usable for creating or expanding diskgroups.
* Disk size consistency is recommended.

---

## 📦 Creating a Diskgroup – Manual SQL

📘 **Definition**
A diskgroup is a storage pool in ASM for database files.

⚠️ **Pre-requisites**

* ASM instance must be running.
* Candidate or Provisioned disks identified and registered.

💻 **Commands**

```sql
-- Check candidate or provisioned disks
SET LINES 200
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15

SELECT PATH, HEADER_STATUS, TOTAL_MB, FREE_MB
FROM V$ASM_DISK
WHERE HEADER_STATUS IN ('CANDIDATE','PROVISIONED');

-- Create diskgroup
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

⚙️ **Clarifications**

* Redundancy options: External, Normal, High.
* ASM stripes data automatically.
* Disk names must be unique.

🚀 **Practical Sequence**

1. Identify candidate/provisioned disks.
2. Create diskgroup manually or via ASMCA.
3. Verify diskgroup status (ONLINE, free space, redundancy).

---

## ➕ Adding a Disk

📘 **Definition**
Adding a disk expands capacity and triggers automatic rebalance.

⚠️ **Pre-requisites**

* Disk must be Candidate/Provisioned.
* Diskgroup must be ONLINE.

💻 **Commands**

```sql
-- Check available disks
SET LINES 200
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15

SELECT PATH, HEADER_STATUS, TOTAL_MB, FREE_MB
FROM V$ASM_DISK
WHERE HEADER_STATUS IN ('CANDIDATE','PROVISIONED');

-- Add disk
ALTER DISKGROUP DATA ADD DISK '/dev/oracleasm/disks/DISK3' NAME DATA3;

-- Verify diskgroup
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB FROM V$ASM_DISKGROUP;
```

⚙️ **Clarifications**

* ASM automatically rebalances data.
* Ensure disk health before adding.

---

## ➖ Dropping a Disk

📘 **Definition**
Dropping a disk removes it from a diskgroup; ASM redistributes extents.

⚠️ **Pre-requisites**

* Diskgroup ONLINE.
* Disk can be safely removed (not holding critical extents).

💻 **Commands**

```sql
ALTER DISKGROUP DATA DROP DISK DATA2;

-- Verify diskgroup
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB FROM V$ASM_DISKGROUP;
```

⚙️ **Notes**

* ASM automatically rebalances remaining data.

---

## ⚖️ Rebalancing Diskgroup

📘 **Definition**
Rebalance redistributes data after add/drop disk operations.

⚠️ **Pre-requisites**

* Diskgroup ONLINE.
* No ongoing conflicting operations.

💻 **Commands**

```sql
ALTER DISKGROUP DATA REBALANCE POWER 5;

-- Monitor
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;
```

⚙️ **Clarifications**

* Power 1–11; higher = faster but more CPU/IO.
* Production: 3–5; Maintenance: 8–11.
* Can pause or cancel rebalance if needed.

---

## 📊 ASM Monitoring Views

💻 **Commands**

```sql
-- Diskgroup info
SET LINES 200
COL NAME FORMAT A15
COL TYPE FORMAT A10
COL STATE FORMAT A10
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB FROM V$ASM_DISKGROUP;

-- Disk info
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15
SELECT NAME, PATH, HEADER_STATUS, STATE, TOTAL_MB, FREE_MB FROM V$ASM_DISK;

-- ASM clients
SELECT INSTANCE_NAME, DB_NAME, STATUS FROM V$ASM_CLIENT;

-- Rebalance operations
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES FROM V$ASM_OPERATION;
```

⚙️ **Notes**

* Always check views before performing diskgroup modifications.

---

## 🚀 Complete Practical Sequence Summary

1. Prepare disks (OS → ASM registration).
2. Verify **Candidate/Provisioned** disks.
3. Create diskgroup (ASMCA or SQL).
4. Add disks to diskgroup.
5. Drop disks from diskgroup.
6. Manual rebalance if needed.
7. Use ASMCMD for file/disk navigation.
8. Monitor diskgroup status via ASM views.

---

