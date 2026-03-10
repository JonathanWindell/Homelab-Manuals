
## 1. Introduction

This guide provides a comprehensive walkthrough for deploying Uptime Kuma, a self-hosted monitoring tool, within a Proxmox VE environment. The primary objective is to configure Uptime Kuma to reliably monitor internal services that are not exposed to the public internet.

The architecture described herein leverages a secure Tailscale network and a private Split DNS server (AdGuard Home or Pi-hole) to resolve custom local domains (e.g., https://service.yourdomain.com). This guide details the complete process, from initial deployment to the critical network configurations required to ensure Tailscale can permanently manage DNS resolution within a Proxmox LXC container, enabling robust, long-term monitoring.

## 2. Environment & Prerequisites

**This guide assumes the following components are already configured and operational:**

- **Proxmox VE Host:** A running Proxmox server for hosting LXC containers.
- **Secure Network:** An active Tailscale network (tailnet) connecting your devices.
- **Reverse Proxy:** An Nginx Proxy Manager (NPM) instance running in an LXC container, with Tailscale installed.
- **Private DNS:** A local DNS server (e.g., AdGuard Home) running in an LXC container, with Tailscale installed and configured as a Split DNS nameserver in the Tailscale Admin Console.
- **Configured Services:** At least one service (e.g., TrueNAS) is already configured in NPM and accessible via a custom domain from a Tailscale-connected client.

## 3. Part 1: Deploying and Configuring the Uptime Kuma Container

**This part covers the initial creation and critical network configuration of the Uptime Kuma container to prevent common DNS conflicts.**

### 3.1. Step 1: Create and Configure the LXC Container

This first step is crucial for ensuring network stability. We will configure the container with a static IP address directly in Proxmox to prevent its internal DHCP client from interfering with Tailscale's DNS management.

1. In the Proxmox web interface, create a new LXC container.
    - **Template:** Use a recent Debian or Ubuntu template.
    - **Resources:** Uptime Kuma is lightweight. 512MB of RAM, 1 CPU core, and an 8GB disk are sufficient.
2. After creation, select the new container and go to the **Network** tab.
3. Select the net0 device and click **Edit**.
4. Configure the network settings as follows:
    - **IPv4:** Change the setting from DHCP to **Static**.
    - **IPv4/CIDR:** Enter a static IP address from your local network (e.g., 192.168.0.50/24).
    - **Gateway (IPv4):** Enter your router's IP address (e.g., 192.168.0.1).
5. Click **OK** to save the network settings.
6. Now, go to the **DNS** tab for the same container.
7. Double-click the **DNS server** field. To prevent Proxmox from pushing your router's DNS settings, enter the loopback address.
    - **DNS server:** 127.0.0.1
8. Click **OK**.
    

### 3.2. Step 2: Install Uptime Kuma

1. Start the newly configured container and open its console.
2. Update the package lists: apt update && apt upgrade -y.

**Execute the official Uptime Kuma installation script. This script will install all necessary dependencies and set up Uptime Kuma to run as a persistent service.**
```Bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/uptimekuma.sh)"
```


### 3.3. Step 3: Initial Access and User Setup

1. Once the installation is complete, access the Uptime Kuma web interface using its static local IP address and the default port, 3001.
    - Example: http://192.168.0.50:3001
2. Follow the on-screen instructions to create your administrator account.

## 4. Part 2: Integrating Uptime Kuma into the Secure Network

**Now, we will make the Uptime Kuma container a full member of the Tailscale network, allowing it to take control of its own DNS.**

### 4.1. Step 1: Install Tailscale in the Uptime Kuma Container

Use the Proxmox VE Helper Scripts from the Proxmox host for a seamless installation.

1. Open the **Proxmox Host Shell** (select your node in the Proxmox UI, then >_ Shell).
	- Execute the helper script:  
	    Bash  
	    bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/add-tailscale-lxc.sh)"
2. When prompted, enter the Container ID (CTID) for your Uptime Kuma container.

### 4.2. Step 2: Authenticate and Finalize DNS Configuration

1. Open the Uptime Kuma container's console.
	- Run the tailscale up command with the --accept-dns=true flag. This explicitly tells Tailscale to manage the container's DNS settings, which is now possible because we disabled DHCP and Proxmox's DNS push.  
	- ```bash
	   tailscale up --accept-dns=true
	  ```

2. Follow the authentication URL to authorize the new device in your Tailscale account.
3. **Verify:** Run cat /etc/resolv.conf. The nameserver should now be 100.100.100.100 and will remain so permanently.
    
## 5. Part 3: Creating the First Monitor

**With Uptime Kuma fully integrated, it can now correctly resolve and monitor your private services.**

### 5.1. Step 1: Add a New Monitor

1. Navigate to your Uptime Kuma dashboard.
2. Click "Add New Monitor" and select HTTP(s).
3. **Friendly Name:** Enter a descriptive name, e.g., TrueNAS Web UI.
4. **URL:** Enter the full, secure custom domain for the service.
    - Example: https://proxmox.jonathans-labb.org

### 5.2. Step 2: Configure Certificate Expiry Monitoring

1. Scroll down and enable **"Certificate Expiry Notification"**.
2. Set a reasonable timeframe, such as 21 days, to get advance warning if your NPM-managed certificate fails to renew.
    

### 5.3. Step 3: Save and Verify

1. Click **Save**.
2. The monitor will appear on the dashboard and should turn green ("Up") within a minute. This confirms Uptime Kuma can use the Split DNS system to resolve the domain and successfully monitor the service through the entire proxy chain.

## 6. Solution: The Final Architecture

**The monitoring data flow is robust:**

1. **Uptime Kuma** initiates a check for a service.
2. The **Tailscale client** inside the Uptime Kuma container sees the .yourdomain.com address and uses Split DNS.
3. The DNS query is sent to the **AdGuard Home container** via Tailscale.
4. **AdGuard Home** resolves the domain to the **NPM container's** Tailscale IP.
5. **Uptime Kuma** connects securely to **NPM** over Tailscale.
6. **NPM** proxies the request to the final service's **local LAN IP**.
7. The successful response travels back, and the monitor is marked as "Up".

## 7. Preventive Measures & Best Practices

- **Configure Notifications:** A monitoring tool is only useful if it can alert you. Configure at least one notification provider (e.g., Email, Discord) in Uptime Kuma's settings.
- **Monitor Your Infrastructure:** Create monitors for your core components, such as the admin pages for NPM and AdGuard Home, using their Tailscale MagicDNS names (e.g., http://npm-hostname:81). This helps detect failures in the infrastructure itself.
- **Use Heartbeats:** For critical scripts or backup jobs, use Uptime Kuma's "Push" type monitor to ensure they are running on schedule.