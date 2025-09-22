
## Oracle 19c Grid Infrastructure & ASM – Complete Practical Guide

This guide provides a **step-by-step walkthrough** of **Oracle ASM (Automatic Storage Management)** in Oracle 19c.
It includes **concepts, definitions, prerequisites, and practical commands** for **DBAs**.

---

## 🔑 Logging in to ASM Instance

### 📘 Definition

* **ASM Instance**: A lightweight Oracle instance that manages diskgroups. It does not have data dictionary views like a normal DB instance but has its own fixed views (`V$ASM_*`).
* **User Grid**: The OS user `grid` owns the Grid Infrastructure stack and ASM instance.
* **SYSASM Role**: A special role for administering ASM.

### 📝 Why Needed?

Every ASM operation (create, drop, add, rebalance) must be executed inside the ASM instance.

### ⚠️ Pre-requisites

* ASM instance running.
* OS user `grid` exists.
* User has `SYSASM` privileges.

### 💻 Commands (as `grid`)

```bash
# Switch to Grid user
su - grid

# Set ASM instance SID
export ORACLE_SID=+ASM

# Connect with SYSASM
sqlplus / as sysasm
```


---

## 📊 ASM Monitoring Views

### 📘 Definition

ASM provides **dynamic performance views** (`V$ASM_*`) for monitoring diskgroups, disks, rebalancing, and connected clients.

* `V$ASM_DISKGROUP` → Diskgroup-level info.
* `V$ASM_DISK` → Physical disks and failure groups.
* `V$ASM_CLIENT` → Databases using ASM.
* `V$ASM_OPERATION` → Rebalance status.

### 📝 Why Needed?

Monitoring is critical to validate free space, redundancy, and rebalance operations.

### 💻 Commands

```sql
-- Diskgroup info
SET LINES 200
COL NAME FORMAT A15
COL TYPE FORMAT A10
COL STATE FORMAT A10
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB
FROM V$ASM_DISKGROUP;

-- Disk info with Failgroups
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15
SELECT NAME AS DISK_NAME, PATH, FAILGROUP, HEADER_STATUS, STATE
FROM V$ASM_DISK;

-- ASM clients
SELECT INSTANCE_NAME, DB_NAME, STATUS
FROM V$ASM_CLIENT;

-- Rebalance operations
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;
```
## 🐚 ASMCMD – Practical CLI Usage

### 💻 Commands

```bash
# Start ASMCMD
asmcmd -p

# Diskgroup info
lsdg
lsdsk

# Navigate
cd DATA
ls
du

# Create & manage files
mkdir arch
touch arch/file1.dbf
cp arch/file1.dbf arch/file2.dbf
mv arch/file2.dbf arch/file2_old.dbf
rm arch/file1.dbf

---

## ⚠️ Preparing Disks for ASM

### 📘 Definition

* **Disk preparation** means partitioning raw devices and registering them with the Oracle ASM library driver.
* **oracleasm tool** helps Linux OS register disks so ASM can recognize them.

### 📝 Why Needed?

ASM can only use disks labeled and registered through `oracleasm` (or UDEV rules).

### ⚠️ Pre-requisites

* Login as **root** for disk operations.
* Disks must be free (Candidate/Provisioned).
* ASM instance running.
```
Storage addition from VirtualBox:

<img width="700" height="350" alt="image" src="https://github.com/user-attachments/assets/981c18d9-4c8a-4a90-a438-9cbfd50cab3a" />
<img width="700" height="350" alt="image" src="https://github.com/user-attachments/assets/5edc9585-a7dc-4215-b333-e583ab455d9b" />
<img width="700" height="350" alt="image" src="https://github.com/user-attachments/assets/1dd575f5-b075-497c-9f3f-94aa2969321f" />
<img width="700" height="350" alt="image" src="https://github.com/user-attachments/assets/5a9fa7e2-d284-49a5-b85d-573d6894c253" />
<img width="759" height="380" alt="image" src="https://github.com/user-attachments/assets/23b4d589-a51b-4ce8-8286-a968cca03860" />


```bash
### 💻 Commands (as `root`)

```bash
# List available disks
fdisk -l | grep sd
lsblk

# Partition new disks
fdisk /dev/sdc
fdisk /dev/sdd
fdisk /dev/sde
fdisk /dev/sdf

# Register disks with oracleasm
oracleasm createdisk DATA1 /dev/sdc1
oracleasm createdisk DATA2 /dev/sdd1
oracleasm createdisk DATA3 /dev/sde1
oracleasm createdisk DATA4 /dev/sdf1

# Verify registered disks
oracleasm listdisks
```

### 💻 Commands (as `grid`)

```bash
su - grid
export ORACLE_SID=+ASM
sqlplus / as sysasm

-- Check if disks are Candidate
SET LINES 200
COL PATH FORMAT A30
COL HEADER_STATUS FORMAT A15
SELECT PATH, HEADER_STATUS, NAME AS DISKGROUP_NAME
FROM V$ASM_DISK;
```

---

## 🔄 Disk Path Clarification

* **OS level**: `/dev/sdc1`, `/dev/sdd1`, `/dev/sde1`, `/dev/sdf1`
* **oracleasm createdisk**: `oracleasm createdisk DATA1 /dev/sdc1`
* **ASM SQL level**: Disks appear as `ORCL:DATA1`, `ORCL:DATA2`, `ORCL:DATA3`, `ORCL:DATA4`

👉 Always use **`ORCL:DISKNAME`** in `CREATE DISKGROUP` and `ALTER DISKGROUP` SQL commands.

---

## 📦 Creating a Diskgroup

### 📘 Definition

* **Diskgroup**: A logical storage pool in ASM, made up of multiple disks.
* **Redundancy Options**:

  * **External**: No mirroring, depends on hardware RAID.
  * **Normal**: 2-way mirroring.
  * **High**: 3-way mirroring.

### 📝 Why Needed?

Every database file in ASM must reside inside a diskgroup.

### ⚠️ Pre-requisites

* ASM ONLINE.
* Candidate disks available.
* Decide redundancy level.

### 💻 Commands

```sql
-- Create diskgroup with Normal redundancy
CREATE DISKGROUP DATA1 NORMAL REDUNDANCY
DISK 'ORCL:DATA1' NAME DATA1,
     'ORCL:DATA2' NAME DATA2;

-- Verify
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB
FROM V$ASM_DISKGROUP;
```

---

## 📂 Mount / Unmount Diskgroup

### 💻 Commands

```sql
-- Mount
ALTER DISKGROUP DATA1 MOUNT;

-- Dismount
ALTER DISKGROUP DATA1 DISMOUNT;
```

---

## ➕ Adding a Disk to Diskgroup

### 💻 Commands

```sql
ALTER DISKGROUP DATA1 ADD DISK 'ORCL:DATA3' NAME DATA3;

-- Monitor rebalance
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;
```

---

## ➖ Dropping a Disk from Diskgroup

### 💻 Commands

```sql
ALTER DISKGROUP DATA1 DROP DISK DATA3;

-- Monitor rebalance
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;
```

---

## 🗑️ Dropping a Diskgroup

### 💻 Commands

```sql
DROP DISKGROUP DATA1 INCLUDING CONTENTS;

-- Verify
SELECT NAME, STATE, TYPE FROM V$ASM_DISKGROUP;
```

---

## 🛡️ Creating a Diskgroup with Failure Groups

### 📘 Definition

* **Failure Group (FG)**: Logical grouping of disks to protect against simultaneous failures (e.g., all disks in a storage shelf).
* ASM ensures mirrored copies are stored across **different FGs**.

### 💻 Commands

```sql
CREATE DISKGROUP DATA2 NORMAL REDUNDANCY
FAILGROUP FG1 DISK 'ORCL:DATA1' NAME DATA1,
              'ORCL:DATA2' NAME DATA2
FAILGROUP FG2 DISK 'ORCL:DATA3' NAME DATA3,
              'ORCL:DATA4' NAME DATA4;

-- Verify
SELECT NAME, TYPE, STATE, TOTAL_MB, FREE_MB
FROM V$ASM_DISKGROUP;
```

---

## ⚖️ Rebalancing Diskgroup / Failure Groups

### 💻 Commands

```sql
-- Start rebalance
ALTER DISKGROUP DATA REBALANCE POWER 5;

-- Monitor progress
SELECT GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_MINUTES
FROM V$ASM_OPERATION;

-- Stop rebalance
ALTER DISKGROUP DATA REBALANCE POWER 0;
```

---




# Exit
exit
```


