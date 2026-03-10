## 1. Introduction

This guide provides a comprehensive walkthrough for connecting, formatting, and configuring an external USB-attached drive (HDD or SSD) as a dedicated backup storage target within a Proxmox VE environment.

Maintaining backups on a physically separate and easily detachable device is a fundamental component of a robust 3-2-1 backup strategy. This process involves identifying the new device from the Linux command line, preparing it with a suitable filesystem, mounting it into the host's directory structure, and finally, adding it as a valid backup storage location through the Proxmox web interface. This enables the creation of reliable, portable backups of all VMs and containers.

## 2. Environment & Prerequisites

**This guide assumes the following:**

- **Proxmox VE Host:** A running Proxmox server.
- **Physical Access:** The ability to physically connect a USB device to the Proxmox host.
- **External Storage:** An external HDD or SSD, connected via a USB adapter (e.g., a USB-to-SATA cable).
-  **Important:** The external drive **will be completely erased** during this process. Ensure it contains no critical data.

## 3. Part 1: Preparing the External Drive via the Command Line

**This part covers the necessary command-line operations to make the physical drive usable by the Proxmox host.**

#### 3.1. Step 1: Connect and Identify the Drive

**First, we must safely identify the device name assigned to the new drive by the Linux kernel.**

1. Physically connect your external SSD to a USB port on the Proxmox host.
2. In the Proxmox web interface, select your server node in the left-hand tree, then open the `>_ Shell`.
3. Execute the `lsblk` command to list all block devices.
	- `lsblk`
4. Examine the output. Your system drive will likely be `sda` or an NVMe device (`nvme0n1`). Your newly connected USB drive will appear as a new device, typically `sdb` or `sdc`. You can confirm by its size. In this example, we will assume the device is `sdb`.

#### 3.2. Step 2: Format the Drive

The drive needs a Linux-compatible filesystem. We will use `ext4`, which is robust and universally supported.
1. **WARNING:** This command will permanently erase all data on the target device. Double-check that you are using the correct device name identified in the previous step.
2. In the Proxmox Shell, execute the `mkfs.ext4` command:
	- `mkfs.ext4 /dev/sdb`
3. The command will format the drive and create a new `ext4` filesystem on it.

#### 3.3. Step 3: Mount the Drive

**Now, we will make the formatted filesystem accessible to the operating system by "attaching" it to a folder.**

1. Create a mount point. This is simply an empty folder where the contents of the drive will appear.
	- `mkdir /mnt/external_backup`
2. Mount the drive to the newly created folder.
	- `mount /dev/sdb /mnt/external_backup`
3. **Verify:** Run `lsblk` again. You should now see that `/dev/sdb` has a `MOUNTPOINT` of `/mnt/external_backup`.

## 4. Part 2: Configuring Storage in the Proxmox Web Interface

**With the drive prepared in the shell, the final step is to inform Proxmox that this location can be used for backups.**

#### 4.1. Step 1: Add the Directory Storage

1. In the Proxmox web interface, navigate to `Datacenter` -> `Storage`.
2. Click the `Add` button and select `Directory` from the list.
3. Fill out the configuration window as follows:
    - **ID:** Enter a descriptive name for the storage, e.g., `External-SSD-Backup`.
    - **Directory:** Enter the full path to the mount point you created: `/mnt/external_backup`.
    - **Content:** Click the dropdown menu and select **only** the `Backup` option.
    - **Nodes:** Ensure it is available on your current Proxmox node.
4. Click `Add`.

Your external drive will now appear in the storage list on the left and is ready to be used as a backup target.

## 5. Part 3: Running a Backup Job

**You can now create and run a backup job targeting your new external drive.**

1. Navigate to `Datacenter` -> `Backup`.
2. Click `Add` to create a new backup job.
3. In the configuration window:
    - **Storage:** Select your new `External-SSD-Backup` from the dropdown list.
    - **Selection mode:** Choose which VMs and containers to include.
    - **Mode:** Select `Snapshot` to perform a live backup with no downtime.
4. Click `Create`.
5. To start the backup immediately, select the newly created job and click `**Run now**`. The progress will be visible in the "Tasks" log at the bottom of the screen.

## 6. Best Practices: Safely Disconnecting the Drive

**When you want to physically disconnect the drive for off-site storage, you must unmount it first to prevent data corruption.**

1. Open the Proxmox `**>_ Shell**`.
2. Execute the `umount` command:
	- `umount /mnt/external_backup`
3. Once the command completes without errors, it is 100% safe to physically unplug the USB cable.