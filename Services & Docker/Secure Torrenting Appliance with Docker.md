## 1. Introduction

This guide provides a comprehensive, step-by-step process for deploying a secure and robust torrenting service on a dedicated server (like a Raspberry Pi or a VM). The goal is to create an always-on appliance where all BitTorrent traffic is forced through a secure VPN tunnel, completely isolating the activity from your main home network and devices.

This document covers the entire process, from OS preparation and Docker installation to the final configuration of a Gluetun (VPN client) and qBittorrent stack.

## 2. Environment & Prerequisites

- **Hardware:** A server device (e.g., Raspberry Pi 4/5, or a VM running Debian/Ubuntu).
- **Storage:** A boot drive for the OS and a separate, larger drive for downloads.
- **VPN Subscription:** An active account with a provider that supports port forwarding in Gluetun (e.g., **Private Internet Access (PIA)**).
- **Network:** The server is connected to your router via an Ethernet cable.
- **Client Device:** A separate computer for setup via SSH.

## 3. Part 1: Preparing the Host Environment

**A clean and correctly configured host OS is the foundation of a stable deployment.**

### 3.1. Step 1: Install OS and Docker

1. **Install OS:** Install a minimal, 64-bit server OS (e.g., **Debian 12 Server** or **Raspberry Pi OS Lite (64-bit)**).
2. **Enable SSH** and create a standard user with `sudo` privileges.
3. **Update System:** Connect via SSH and run a full system update:
	- `sudo apt update && sudo apt full-upgrade -y`
4. **Install Docker:**
	- `curl -sSL https://get.docker.com | sh`
    `sudo usermod -aG docker ${USER}`
    `sudo apt install docker-compose -y`

**Important:** Log out (`exit`) and log back in for the `usermod` changes to take effect.

### 3.2. Step 2: Prepare the External Storage

Mount the external drive to a permanent location.

1. **Connect the drive**, find its partition name with `lsblk`.
2. **Format the partition** to `ext4` (e.g., `sudo mkfs.ext4 /dev/sda1`). **This erases all data.**
3. **Create a mount point:** `sudo mkdir /mnt/downloads`
4. **Configure auto-mount** by editing `sudo nano /etc/fstab` and adding a line using the partition's `UUID` (found with `sudo blkid`).
    `UUID=YOUR_UUID_HERE /mnt/downloads ext4 defaults,auto,users,rw,nofail 0 0`
5. **Test the mount** and set permissions:
	- `sudo mount -a`
    `sudo chown ${USER}:${USER} /mnt/downloads`

## 4. Part 2: Deploying the Docker Stack

### 4.1. Step 1: Create the Project Directory and File
- `cd ~`
- `mkdir pia-stack`
- `cd pia-stack`
- `nano docker-compose.yml`

### 4.2. Step 2: Create the Docker Compose Configuration

Paste the following configuration into the `nano` editor, replacing the placeholders with your PIA credentials.

```yaml
version: "3.8"
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    ports:
      - 8080:8080`
      - 54321:54321/tcp # Temporary P2P Port
      - 54321:54321/udp # Temporary P2P Port
    environment:
      - VPN_SERVICE_PROVIDER=private internet access
      - VPN_TYPE=openvpn
      - OPENVPN_USER=YOUR_PIA_USERNAME_HERE
      - OPENVPN_PASSWORD=YOUR_PIA_PASSWORD_HERE
      - VPN_PORT_FORWARDING=on
      - SERVER_REGIONS=Sweden
      - TZ=Europe/Stockholm
    volumes:
      - ./gluetun-data:/gluetun
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent
    container_name: qbittorrent
    network_mode: "service:gluetun"
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Stockholm
      - WEBUI_PORT=8080
    volumes:
      - ./qb-config:/config
      - /mnt/downloads:/downloads
    depends_on:
      - gluetun
    restart: unless-stopped
```

## 5. Part 3: Final Configuration and Verification

### 5.1. Step 1: Initial Launch and Port Discovery

1. **Launch the stack:** `docker-compose up -d`
2. **Find your forwarded port:** Wait a minute, then check the logs:
	- `docker-compose logs gluetun | grep 'port forwarded'`

	Note the five-digit port number (`XXXXX`).

### 5.2. Step 2: Update Final Settings

1. **Update** `docker-compose.yml`**:** Replace all three instances of `54321` with your new, real port number (`XXXXX`).
2. **Update qBittorrent:** Access the WebUI at `http://<your-server-ip>:8080`, log in, and change the "Port used for incoming connections" in the Connection settings to your new port number.
3. **Restart the Stack:** Apply the final port configuration:
	- `docker-compose up -d --force-recreate`

## 6. Solution: The Final Architecture

You now have a secure torrenting appliance where all traffic from qBittorrent is forced through a secure VPN tunnel with an open port, and all downloaded files are saved directly to your dedicated external storage.

## 7. Preventive Measures & Best Practices

- **Check Logs:** If you experience issues, always check `docker-compose logs gluetun`.
- **Keep Updated:** Regularly run `sudo apt update && sudo apt full-upgrade -y` on the host and `docker-compose pull` in your project directory.
- **Port Persistence:** The forwarded port is "sticky" but not permanent. If you have connection issues after a restart, re-check the logs to see if you've been assigned a new port.

## 8. Appendix: Useful Management Commands

Here are the essential commands for managing your stack and files. All commands are run from your project directory (e.g., `~/pia-stack`) unless otherwise specified.

### 8.1. Docker Stack Management

- **Command:** `docker-compose logs qbittorrent`
    - **Purpose:** To view the log output of the qBittorrent container. This is essential for finding the **initial temporary password** for the WebUI.
- **Command:** `docker-compose up -d --force-recreate`
    - **Purpose:** To apply changes made to your `docker-compose.yml` file. The `--force-recreate` flag ensures that containers are updated with new settings, such as corrected port mappings.
- **Command:** docker rmi $(docker images -q)
    - **Purpose:** To remove unused resources such as images & more. Container must be shutdown with docker-compose down before executing the cmd. Run df -h to see that it has been successful.

### 8.2. File & Directory Management

- **Command:** `ls -lh /mnt/downloads`
    - **Purpose:** To **l**i**s**t the contents of your download directory in a **h**uman-readable, **l**ong format. Use this to verify that files are being saved to your SSD.
- **Command:** `find . -type f`
    - **Purpose:** To recursively list all **f**iles within the current directory (`.`) and all of its subdirectories. This is extremely useful for getting a complete overview of a download's contents to look for unexpected or suspicious file types.
- **Command:** `sha256sum /mnt/downloads/your-file-name.iso`
    - **Purpose:** To calculate the unique SHA256 "fingerprint" of a file for verification on sites like VirusTotal. Note: This is slow for large files.
- **Command:** `rm /mnt/downloads/your-file-name.iso`
    - **Purpose:** To permanently remove a file.
    - Warning: This command is permanent and does not use a trash bin. Double-check the filename before executing.`