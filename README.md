# Executive Summary
This project demonstrates a cost-optimized Oracle RAC 12c architecture using virtualization, designed to deliver high availability
without reliance on expensive SAN infrastructure. The implementation simulates enterprise clustering behavior suitable for mid-scale
environments and testing scenarios.

## Oracle Virtual Box: To ensure it is successful on open-source virtualization solutions.

## 2-Node architecture was seleted due to resource limitation and implementation time. 

## NOTE: OL6 was used to match the client's production environment at the time of implementation.

# 2-Node Oracle RAC 12c on VirtualBox

Oracle Real Application Clusters 12c R1 (12.1.0.2) implemented on Oracle Linux 6
using VirtualBox. This guide covers the full stack: Grid Infrastructure, ASM shared
storage, clusterware networking, and RAC database creation.

---

## Architecture

### Network Topology (4-IP Design)

| Type | Node 1 (rr1) | Node 2 (rr2) | Purpose |
|---|---|---|---|
| Public | 10.5.1.221 | 10.5.1.222 | Client communication |
| Private | 10.0.6.221 | 10.0.6.222 | Interconnect (clusterware) |
| Virtual | 10.5.1.223 | 10.5.1.224 | Failover / DB access |
| SCAN | 10.5.1.225/226/227 | — | Round-robin client access |

> SCAN requires 3 IPs as recommended by Oracle for proper RAC installation.

### Storage (ASM Shared Disks)

4× VDI shared disks (50 GB each), fixed size, stored outside the VM directory,
attached to both nodes as shareable via VirtualBox SATA controller.

---

## Software Stack

| Component | Version |
|---|---|
| OS | Oracle Linux 6 |
| Database | Oracle 12c R1 (12.1.0.2) Enterprise Edition |
| Grid Infrastructure | 12c R1 |
| Storage | Oracle ASM (Diskgroup: DATA) |
| DNS | dnsmasq |
| Virtualisation | VirtualBox |

---

## Key Configuration Snippets

### /etc/hosts (both nodes)
Public
10.5.1.221   rr1.fatimagroup.com   rr1
10.5.1.222   rr2.fatimagroup.com   rr2
Private
10.0.6.221   rr1-priv.fatimagroup.com   rr1-priv
10.0.6.222   rr2-priv.fatimagroup.com   rr2-priv
Virtual
10.5.1.223   rr1-vip.fatimagroup.com   rr1-vip
10.5.1.224   rr2-vip.fatimagroup.com   rr2-vip
SCAN
10.5.1.225   rr-scan.fatimagroup.com   rr-scan
10.5.1.226   rr-scan.fatimagroup.com   rr-scan
10.5.1.227   rr-scan.fatimagroup.com   rr-scan
### ASM Shared Disk Creation (VBoxManage)

```bash
mkdir -p /opt/sharedisks && cd /opt/sharedisks

VBoxManage createhd --filename asm1.vdi --size 50120 --format VDI --variant Fixed
VBoxManage createhd --filename asm2.vdi --size 50120 --format VDI --variant Fixed
VBoxManage createhd --filename asm3.vdi --size 50120 --format VDI --variant Fixed
VBoxManage createhd --filename asm4.vdi --size 50120 --format VDI --variant Fixed

VBoxManage modifyhd asm1.vdi --type shareable
VBoxManage modifyhd asm2.vdi --type shareable
VBoxManage modifyhd asm3.vdi --type shareable
VBoxManage modifyhd asm4.vdi --type shareable
```

### udev Rules for ASM Disks (/etc/udev/rules.d/99-oracle-asmdevices.rules)
KERNEL=="sd?1", BUS=="scsi", PROGRAM=="/sbin/scsi_id -g -u -d /dev/$parent",
RESULT=="<DISK1_ID>", NAME="asm-disk1", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", BUS=="scsi", PROGRAM=="/sbin/scsi_id -g -u -d /dev/$parent",
RESULT=="<DISK2_ID>", NAME="asm-disk2", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", BUS=="scsi", PROGRAM=="/sbin/scsi_id -g -u -d /dev/$parent",
RESULT=="<DISK3_ID>", NAME="asm-disk3", OWNER="oracle", GROUP="dba", MODE="0660"
KERNEL=="sd?1", BUS=="scsi", PROGRAM=="/sbin/scsi_id -g -u -d /dev/$parent",
RESULT=="<DISK4_ID>", NAME="asm-disk4", OWNER="oracle", GROUP="dba", MODE="0660"

> Get each disk's RESULT ID with: `/sbin/scsi_id -g -u -d /dev/sdb` (repeat for sdc, sdd, sde)

### Oracle Environment Profile (~/.bash_profile)

```bash
export ORACLE_BASE=/opt/app/oracle
export GRID_HOME=/opt/app/12.1.0.2/grid
export DB_HOME=$ORACLE_BASE/product/12.1.0.2/db_1
export ORACLE_HOME=$DB_HOME
export ORACLE_SID=cdbrac1          # cdbrac2 on node 2
export ORACLE_UNQNAME=CDBRAC

alias grid_env='. /home/oracle/grid_env'
alias db_env='. /home/oracle/db_env'
```

### Lock resolv.conf (prevents DHCP overwrite)

```bash
chattr +i /etc/resolv.conf
# Unlock with: chattr -i /etc/resolv.conf
```

---

## Installation Phases

1. **Pre-install** — OS packages, oracle user/groups (oinstall, dba, asmdba, asmoper, asmadmin), OFA directories
2. **Networking** — Static IPs, dnsmasq DNS, SCAN resolution, `resolv.conf` lock
3. **Shared Storage** — 4× ASM VDIs created and attached to both VMs as shareable
4. **udev Rules** — ASM disks mapped by SCSI ID, owned by oracle:dba
5. **SSH Equivalence** — Key-based auth between rr1 and rr2 as oracle user
6. **Pre-check** — `runcluvfy.sh stage -pre crsinst -n rr1,rr2 -verbose` (must pass)
7. **Grid Infrastructure** — Standard cluster, ASM storage, DATA diskgroup
8. **Database Software** — RAC installation across both nodes
9. **DBCA** — Container database `cdbrac` with PDB `pdb1` on ASM

---

## Verification

### Clusterware Status

```bash
. /home/oracle/grid_env
crsctl stat res -t
```

Expected output (all resources ONLINE STABLE on both nodes):
ora.DATA.dg        ONLINE  ONLINE  rr1  STABLE
ONLINE  ONLINE  rr2  STABLE
ora.asm            ONLINE  ONLINE  rr1  Started,STABLE
ONLINE  ONLINE  rr2  Started,STABLE
ora.rr1.vip        ONLINE  ONLINE  rr1  STABLE
ora.rr2.vip        ONLINE  ONLINE  rr2  STABLE
ora.scan1.vip      ONLINE  ONLINE  rr2  STABLE
ora.scan2.vip      ONLINE  ONLINE  rr1  STABLE
ora.scan3.vip      ONLINE  ONLINE  rr1  STABLE
### RAC Database Status

```bash
srvctl status database -d cdbrac
```
### RAC Database Status

```bash
srvctl status database -d cdbrac
```
### Active Instances (from SQL*Plus)

```sql
SELECT inst_name FROM v$active_instances;
```
### Active Instances (from SQL*Plus)

```sql
SELECT inst_name FROM v$active_instances;
```
---

## Full Implementation Guide

The complete step-by-step SOP with screenshots is available in
[docs/oracle-rac-12c-implementation-guide.pdf](docs/oracle-rac-12c-implementation-guide.pdf)

## Video Tutotrial https://youtu.be/_hh6R26YiAs
