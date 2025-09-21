## Oracle 19c Grid Infrastructure & ASM  🎓


This document is a **step-by-step, practical guide** to **Oracle ASM (Automatic Storage Management)** in **Oracle Grid Infrastructure 19c**.

It covers ASM concepts, diskgroup management, ASMCMD and ASMCA usage, rebalance operations, and monitoring techniques.


---

## 1. Logging in to ASM Instance 🔑

📘 **Definition**
ASM (Automatic Storage Management) is Oracle’s **volume manager + filesystem** for databases.
It runs in its own **+ASM instance**, separate from RDBMS instances.

💻 **Command**

```bash
su - grid
sqlplus / as sysasm
```

⚙️ **Explanation**

* `grid` = OS user owning Grid Infrastructure.
* `SYSASM` = privilege needed for ASM storage operations.
* If connected as `SYSDBA`, **cannot perform storage management**.

---

## 2. ASMCA (ASM Configuration Assistant) 🖥️

📘 **Definition**
ASMCA is a **GUI tool** for ASM diskgroup management.

💻 **Command**

```bash
asmca
```

⚙️ **Explanation**

* Allows creating/managing diskgroups, monitoring usage, configuring redundancy.
* Internally runs SQL and ASMCMD commands.
* Useful for beginners; **CLI preferred in production**.

---

## 3. ASMCMD (ASM Command Line Tool) 🐚

📘 **Definition**
ASMCMD is a **CLI shell** for ASM, like a Unix shell for ASM storage.

💻 **Commands**

```bash
asmcmd
ASMCMD> lsdg     -- list diskgroups
ASMCMD> lsdsk    -- list disks
ASMCMD> du       -- disk usage
ASMCMD> exit
```

⚙️ **Explanation**

* Queries `V$ASM_*` views internally.
* Quick navigation of ASM files/disks.
* Without it, one would rely entirely on SQL queries.

---

## 4. Preparing a Disk for ASM ⚠️

📘 **Definition**
A disk must be **Candidate** or **Provisioned** to be added to ASM.
`header_status = MEMBER` → already part of a diskgroup.

💻 **Steps in VirtualBox / Linux**

1. ➕ Add a **raw virtual disk** in VirtualBox (e.g., 50GB).
2. 🔎 Discover OS-level device:

   ```bash
   fdisk -l | grep sd
   ```
3. ⚙️ Register disk for ASM using ASMLib:

   ```bash
   oracleasm createdisk DISK1 /dev/sdb
   oracleasm createdisk DISK2 /dev/sdc
   ```
4. 📊 List registered ASM disks:

   ```bash
   oracleasm listdisks
   ```

⚙️ **Explanation**

* `oracleasm` maps OS raw disks → ASM-managed disks.
* Only Candidate/Provisioned disks appear in ASM.
* Without this, ASM cannot recognize or use the disk.

---

## 5. Creating a Diskgroup 📦

📘 **Definition**
A **diskgroup** is a storage pool in ASM where database files reside.

💻 **Command**

```sql
create diskgroup DATA external redundancy
disk '/dev/oracleasm/disks/DISK1' name DATA1,
     '/dev/oracleasm/disks/DISK2' name DATA2;
```

📊 **PuTTY-Friendly Query**

```sql
set lines 200
col name format a15
col type format a10
col state format a10

select name, type, state, total_mb, free_mb 
from v$asm_diskgroup;
```

⚙️ **Explanation**

* **Redundancy**: External (hardware RAID), Normal (2-way mirroring), High (3-way).
* ASM **stripes data** for performance.
* Without diskgroup → database cannot store files on ASM.

---

## 6. Adding Disk to Existing Diskgroup ➕💽

⚠️ **Pre-Check**
Disk must be **Candidate** or **Provisioned**.

```sql
set lines 200
col path format a30
col header_status format a15

select path, header_status, total_mb, free_mb 
from v$asm_disk;
```

💻 **Command**

```sql
alter diskgroup DATA add disk '/dev/oracleasm/disks/DISK3' name DATA3;
```

⚙️ **Explanation**

* Triggers **automatic rebalance**.
* Skipping rebalance → uneven utilization → performance issues.

---

## 7. Rebalancing in ASM ⚖️

📘 **Definition**
Rebalance redistributes data across disks after **add/remove/move operations**.

💻 **Command**

```sql
alter diskgroup DATA rebalance power 5;
```

📊 **Monitor Progress (PuTTY formatting)**

```sql
set lines 200
col operation format a12
col state format a10
col est_minutes format 999

select group_number, operation, state, power, sofar, est_work, est_minutes
from v$asm_operation;
```

⚙️ **Explanation**

* Background processes (`RBAL`, `ARBn`) move extents.
* **Power** controls rebalance speed (1 = slow, 11 = max).
* Skipping rebalance → uneven data distribution → hot disks.

---

## 8. Useful ASM Views 📊

💻 **PuTTY-Formatted Queries**

```sql
set lines 200
col name format a15
col path format a30
col state format a10
col header_status format a15

-- Diskgroup info
select name, type, state, total_mb, free_mb
from v$asm_diskgroup;

-- Disk info
select name, path, header_status, state, total_mb, free_mb
from v$asm_disk;

-- Clients using ASM
select instance_name, db_name, status
from v$asm_client;

-- Rebalance operations
select group_number, operation, state, power, sofar, est_minutes
from v$asm_operation;
```

⚙️ **Explanation**

* `v$asm_diskgroup` → overview of diskgroups.
* `v$asm_disk` → disk-level status.
* `v$asm_client` → databases using ASM.
* `v$asm_operation` → ongoing rebalance info.

---

## 9. End-to-End Demo Flow 🚀

1. ⚠️ **Prepare disks** → add VirtualBox disk + `oracleasm createdisk`.
2. 📊 **Check candidate disks**:

```sql
select path, header_status from v$asm_disk;
```

3. 📦 **Create diskgroup**:

```sql
create diskgroup DATA external redundancy disk '/dev/oracleasm/disks/DISK1';
```

4. 📊 **Verify diskgroup**:

```sql
select name, type, total_mb, free_mb from v$asm_diskgroup;
```

5. ➕💽 **Add disk to diskgroup**:

```sql
alter diskgroup DATA add disk '/dev/oracleasm/disks/DISK2';
```

6. ⚖️ **Monitor rebalance**:

```sql
select group_number, operation, sofar, est_minutes from v$asm_operation;
```

7. 🐚 **Navigate ASM with ASMCMD**:

```bash
asmcmd
ASMCMD> lsdg
ASMCMD> lsdsk
```

---
