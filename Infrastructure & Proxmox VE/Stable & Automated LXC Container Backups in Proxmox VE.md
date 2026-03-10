## 1. Introduction

This guide provides a formal, step-by-step procedure for creating scheduled, automated backups of LXC (Linux Containers) within a Proxmox Virtual Environment (VE). The primary objective is to establish a "set-and-forget" backup strategy that ensures data integrity and recoverability.

A critical focus of this guide is **maintaining system stability**. Unrestricted backup operations can generate intense I/O (Input/Output) load, potentially causing system freezes on some hardware. This manual details how to configure a "throttled" backup job that runs safely without compromising the stability of the Proxmox host.
## 2. Environment & Prerequisites

- **Proxmox VE Host:** A functioning Proxmox VE server with one or more running LXC containers.
- **Storage Target:** A pre-configured storage location for the backups. This guide will primarily use the `local` storage on the Proxmox host.
- **Permissions:** Administrative access to the Proxmox VE web interface.

## 3. Part 1: Preparing the Backup Storage

**This step ensures that your chosen storage location is configured to accept Proxmox backup files (`VZDump`).**

### 3.1. Step 1: Verify Storage Content Type

1. Navigate to the **Datacenter** view in the Proxmox web interface.
2. From the side menu, select **Storage**.
3. Select your target storage (e.g., `local`).
4. Click the **Edit** button.
5. In the `Edit Storage` window, locate the **Content** field.
6. Ensure that `VZDump backup file` is selected from the dropdown list. If it is not, select it and click `OK`.

## 4. Part 2: Creating and Configuring the Automated Backup Job

**This is the core section where the scheduled task is defined, with a critical focus on performance tuning.**

### 4.1. Step 1: Navigate to the Backup Interface

1. In the **Datacenter** view, select **Backup** from the side menu.

### 4.2. Step 2: Initiate New Backup Job

1. Click the **Add** button. This will open the `Create: Backup Job` configuration window.

### 4.3. Step 3: Configure General Parameters

1. In the **General** tab, define the basic parameters:
    - **Node:** Ensure this is set to your Proxmox node (e.g., `pve`).
    - **Storage:** Select your designated backup storage (e.g., `local`).
    - **Schedule:** Define a time for the job to run during off-peak hours (e.g., `02:00`).
    - **Selection mode:** Choose `Include selected VMs/CTs`.
    - **Target Selection:** Check the box next to **each LXC container** you wish to back up.
    - **Mode:** Set to `Snapshot`. This is the recommended method for live backups of LXC containers.
    - **Compression:** Set to `ZSTD (fast)`.
    - **Enable:** Ensure the `Enable` checkbox is ticked.

### 4.4. Step 4: Configure Retention Policy

1. Click on the **Retention** tab.
2. Define how many backups to keep. This is critical to prevent your storage from filling up.
    - Example: Set `Keep Last` to `3`. This instructs Proxmox to retain the three most recent backups.

### 4.5. Step 5: Configure Performance Limits (Critical for Stability)

1. Click on the **Advanced** tab. This is the most important step for preventing system freezes.
2. Locate the **Performance** section.
3. **Set Bandwidth Limit:** This setting limits the I/O speed of the backup process. It is the most effective tool for preventing storage controller overloads.
    - In the `Bandwidth Limit (MB/s)` field, enter a safe value. A starting value of `50` is highly recommended. This limits the backup to a gentle, consistent speed that the system can handle without stress.

### 4.6. Step 6: Finalize the Job

1. Review all configured settings across the **General**, **Retention**, and **Advanced** tabs.
2. Click the **Create** button to save and activate the throttled backup job.

## 5. Part 3: Managing and Verifying the Job

1. **Verification:** The new job will appear in the list under `Datacenter -> Backup`.
2. **Manual Execution:** To test the job, select it and click **Run now**. Monitor the `Tasks` log at the bottom of the interface. Note that the job will take longer to complete due to the bandwidth limit, which is expected and desired behavior.
3. **Viewing Backups:** Once complete, backups are visible under each container's **Backup** tab.

## 6. Solution: The Final Architecture

**Upon completion of this guide, you will have a fully automated and  stable backup system where:**
- A scheduled job runs daily at a specified time.
- The job is **throttled to a safe speed**, preventing I/O-related system freezes.
- A complete, compressed backup of each selected LXC container is created.
- An automated retention policy manages disk space by purging old backups.
- The entire process requires no manual intervention, providing consistent and reliable recovery points without compromising host stability.

## 7. Preventive Measures & Best Practices

- **Start with a Safe Limit:** Always begin with a conservative bandwidth limit (e.g., 25-50 MB/s). You can cautiously increase it later if your system remains stable and you wish to shorten backup times.
- **Test Your Backups:** A backup strategy is only valid if it is tested. Periodically, perform a test restore of a container to ensure the backups are viable.
- **External Storage:** For true disaster recovery, plan to migrate your backup storage target from `local` to a separate physical device (like a NAS). This protects your data against a complete failure of the Proxmox host.
- **Split Large Jobs:** If you have dozens of containers, consider creating multiple smaller backup jobs scheduled at different times (e.g., one at 02:00, another at 03:00) to further distribute the load.