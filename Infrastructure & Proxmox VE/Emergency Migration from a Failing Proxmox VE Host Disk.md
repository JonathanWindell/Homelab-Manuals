## 1. Introduction

This document provides a formal, procedural guide for diagnosing and recovering from a critical Proxmox VE host failure caused by a degrading system disk. It details the process of identifying the root cause, migrating all virtualized services (LXC Containers and VMs) to a healthy secondary disk, and troubleshooting advanced issues that may arise during the migration. This manual serves as a comprehensive record of a real-world recovery scenario.

## 2. Scenario & Objective

- **Scenario:** A Proxmox VE host began experiencing nightly, system-wide freezes. Initial symptoms pointed towards an I/O overload caused by scheduled backup jobs. However, further investigation revealed critical PCIe and NVMe-related hardware errors in the system logs, indicating a failing primary system disk.
- **Objective:** The primary objective is to perform an emergency evacuation of all active services from the failing NVMe disk (`local-lvm`) to a secondary, healthy SATA SSD (`data-storage`) with zero data loss, ensuring service continuity and restoring host stability.

## 3. Rationale & Benefits
SStable & Automated LXC Container Backups in Proxmox VE
Migrating services off a failing disk is a critical, time-sensitive operation. The rationale is to prevent catastrophic data loss that would occur upon the disk's complete failure. The benefits of this procedure are:

- **Data Preservation:** Securing all container and VM data before the source disk becomes unreadable.
- **Service Continuity:** Restoring all services to a fully operational state on stable hardware.
- **System Stability:** Eliminating the root cause of the system freezes.

## 4. Prerequisites & Environment

- **Proxmox VE Host:** A single Proxmox node experiencing system instability.
- **Failing Primary Disk:** An NVMe SSD hosting the Proxmox OS and the default `local-lvm` storage for VMs/Containers.
- **Healthy Secondary Disk:** A SATA SSD configured in Proxmox as an additional storage pool (e.g., `data-storage`).
- **Access:** Full administrative access to the Proxmox VE web interface and the underlying command-line shell.

## 5. Step-by-Step Migration & Troubleshooting Process

**This process details the evacuation of all virtual guests. Each guest (VM or LXC) should be processed individually to minimize load on the unstable host.**

### 5.1. Phase 1: Standard Guest Migration

**This is the standard procedure for moving a guest's disk between storage locations.**

1. **Shutdown Guest:** In the Proxmox web UI, right-click the target guest (e.g., CT 101) and select **Shutdown**. Wait for its status to change to "stopped".
2. **Initiate Move:** Select the stopped guest, navigate to its **Resources** (for LXC) or **Hardware** (for VMs) tab.
3. Select the `Root Disk` (or `Hard Disk`) that resides on the failing storage (`local-lvm`).
4. Click the **Move storage** button.
5. In the dialog box, set the **Target Storage** to the healthy disk (`data-storage`) and **tick the** `Delete source` **checkbox**.
6. Click **Move** and wait for the `TASK OK` confirmation.
7. Start the guest and verify its functionality.

### 5.2. Phase 2: Advanced Troubleshooting During Migration

During the migration of a system with an unstable disk, errors are likely. The following steps address the specific issues encountered.

**Troubleshooting Step 2.1: Error - "you can't move a volume with snapshots"**

- **Cause:** Proxmox's safety mechanism prevents moving a disk with active snapshots when "Delete source" is selected.
- **Solution:**
    1. Select the guest and navigate to its **Snapshots** tab.
    2. Select each snapshot individually and click **Remove**. Confirm the action.
    3. Repeat until the snapshot list is empty.
    4. Re-attempt the migration from Phase 1.

**Troubleshooting Step 2.2: Error - "CT is locked (snapshot-delete)"**

- **Cause:** A failed operation (like deleting a snapshot on the unstable disk) can leave a persistent lock on the guest, preventing further actions. A padlock icon will be visible next to the guest ID.
- **Solution:**
    1. Open the Proxmox node's shell (`>_ Shell`).
    2. Execute the unlock command: `pct unlock <CT_ID>` (e.g., `pct unlock 110`).
    3. The lock icon will disappear, allowing you to re-attempt the previous action.
        

**Troubleshooting Step 2.3: Error - "Failed to find logical volume"**

- **Cause:** This indicates a metadata inconsistency. The guest's configuration file believes a snapshot exists, but the underlying storage system (LVM) reports that it does not. This is a common result of a crash during a snapshot operation.
- **Solution:** Manually edit the guest's configuration file to remove the "ghost" snapshot reference.
    1. Open the Proxmox node's shell.
    2. Create a backup of the configuration file: `cp /etc/pve/lxc/<CT_ID>.conf /etc/pve/lxc/<CT_ID>.conf.bak`
    3. Open the file for editing: `nano /etc/pve/lxc/<CT_ID>.conf`
    4. Locate and delete the entire line that references the non-existent snapshot (e.g., a line starting with `snap_...`). Use `Ctrl+K` in nano to delete the line.
    5. Save and exit the editor (`Ctrl+X`, then `Y`, then `Enter`).
    6. The snapshot will now be gone from the UI, and you can proceed with the migration.

### 5.3. Phase 3: GUI-Based File Migration & Finalization

1. After migrating all guests, perform a final check.
2. Navigate to **Datacenter -> Storage**.
3. Select the failing storage (`local-lvm`). Its usage should now be close to zero.
4. Select the healthy storage (`data-storage`). Its usage should have increased accordingly.

## 6. Final Root Cause Summary

The root cause of the system-wide freezes was a **critical hardware failure of the primary NVMe system disk**. This was definitively diagnosed through:

- **System Logs:** Repeated `PCIe Bus Error` and `nvme Timeout` messages during periods of high I/O load.
- **SMART Data:** Analysis of the NVMe disk's SMART values revealed an `Available Spare` **of only 64%** and **39** `Media and Data Integrity Errors`, indicating severe physical degradation and data corruption.

The backup job was not the cause, but merely the trigger that exposed the underlying hardware instability through sustained I/O load.

## 7. Solution: Final Architecture

The immediate solution involved a full evacuation of all virtual guests to the secondary, healthy 1TB SATA SSD. All services are now running stably from this disk. The long-term solution involves:

1. Physically removing the failing NVMe disk.
2. Performing a fresh installation of Proxmox VE onto either a new, reliable NVMe disk or directly onto the 1TB SATA SSD.
3. Re-attaching the existing storage pool (`data-storage`) to the new Proxmox installation to bring all services back online without restoring from backups.

## 8. Preventive Measures

- **Regular SMART Monitoring:** Implement a routine to periodically check the SMART health data of all disks in the system (`smartctl -a /dev/sdX`).
- **Physical Disk Separation:** For critical systems, run the Proxmox OS on a separate physical disk from the VM and container data. This simplifies recovery from an OS disk failure.
- **Maintain Off-Host Backups:** Ensure that automated backups are stored on a separate physical device (e.g., a NAS) to protect against a complete host hardware failure.

## 9. Appendix: Key Commands & Paths

- **Unlock a locked LXC container:** `pct unlock <CT_ID>`
- **Path to LXC configuration files:** `/etc/pve/lxc/`
- **Edit a file via command line:** `nano /path/to/file.conf`
- **Check NVMe SMART health:** `smartctl -a /dev/nvme0n1`
- **Check SATA SMART health:** `smartctl -a /dev/sda`