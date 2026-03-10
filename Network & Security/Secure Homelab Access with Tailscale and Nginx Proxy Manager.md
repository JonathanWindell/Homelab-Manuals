## 1. Introduction

This guide provides a step-by-step process for configuring a secure, private, and user-friendly setup for accessing homelab services. The primary goal is to use a custom domain with a valid SSL certificate to access internal services (hosted in Proxmox LXC containers) without exposing any ports to the public internet. This is achieved by leveraging Tailscale for secure network access and Nginx Proxy Manager (NPM) as a reverse proxy for SSL termination and service routing. This document details the entire setup process, including common pitfalls, specific error messages, and their corresponding solutions.

## 2. Problem Summary

The initial objective was to establish remote access to services like Proxmox, Jellyfin, and TrueNAS. While seemingly straightforward, the integration of multiple technologies—Proxmox LXC, Tailscale, NPM, Cloudflare for domain management, and Let's Encrypt for SSL—led to a series of challenges. Issues encountered included fundamental connectivity problems, DNS resolution failures, SSL handshake errors between internal services, and application-specific proxy failures. The debugging process revealed several critical misunderstandings about how these technologies interact within a private network.

## 3. Symptoms

**Throughout the configuration process, several distinct errors were encountered. This guide provides solutions for the following symptoms:**

- **Browser Error:** ERR_NAME_NOT_RESOLVED or DNS_PROBE_FINISHED_NXDOMAIN when trying to access a custom domain.
- **Browser Error:** ERR_SSL_UNRECOGNIZED_NAME_ALERT when accessing the proxy address.
- **Browser Error:** 502 Bad Gateway after a seemingly successful proxy configuration.
- **Connection Fails:** Inability to connect to a service's Tailscale IP address (e.g., https://100.x.y.z:8006).
- **NPM Status:** The Proxy Host status shows as **"Offline"** in the NPM dashboard.
- **Application Failure:** Specific application features, like the Proxmox noVNC console, fail to work through the proxy.
- **Terminal Error:** tailscale up command fails inside an LXC container with Unit tailscale.service not found.

## 4. Environment Overview

**The final, working solution is composed of the following components:**

- **Hypervisor:** Proxmox VE
- **Services:** Various applications (e.g., Proxmox Web UI, Jellyfin, TrueNAS) running in their own LXC Containers.
- **Reverse Proxy:** Nginx Proxy Manager running in a dedicated LXC Container.
- **Secure Networking:** Tailscale installed on the end-user client device (e.g., a Windows PC), the Proxmox host, and the Nginx Proxy Manager container.
- **Domain & SSL:** A custom domain managed via Cloudflare, with SSL certificates provided by Let's Encrypt using the DNS-01 challenge method.

## 5. Step-by-Step Debugging & Configuration Process

**This process outlines the logical steps from basic connectivity to a full reverse proxy setup.**

### 5.1. Baseline Connectivity with Tailscale

The first step is to ensure all core components can communicate over the private Tailscale network.

1. **Installation:** Install the Tailscale client on your primary computer (the client) and on the target server (e.g., the Proxmox host).
2. **Verification:** From the client, use ping <Server's Tailscale IP> to confirm a basic network path exists.
3. **Direct Access:** Attempt to access the service directly via its Tailscale MagicDNS name and port (e.g., https://proxmox-hostname:8006).


- **Encountered Problem:** DNS_PROBE_FINISHED_NXDOMAIN.
- **Root Cause & Solution:** This error occurs because the client device is not part of the private network. To resolve any Tailscale address (100.x.y.z or MagicDNS name), the device making the request **must** have Tailscale installed and running.

### 5.2. Preparing the Nginx Proxy Manager (NPM) Container

For Nginx Proxy Manager (NPM) to function as the central entry point, it must be a full and reliable member of the Tailscale network. The recommended method for installing Tailscale in a Proxmox LXC container is to use the Proxmox VE Helper Scripts, as this automates all necessary prerequisites, including container permissions and service configuration.

**Note:** This process is performed from the **Proxmox host shell**, not from inside the target LXC container.

#### 5.2.1. Step 1: Open the Proxmox Host Shell

Navigate to your Proxmox web interface. In the server view tree on the left, select your main Proxmox node (e.g., pve), not the NPM container itself. Then, open the >_ Shell.

#### 5.2.2. Step 2: Execute the Tailscale LXC Helper Script

Paste and run the following command in the Proxmox host shell. This will download and execute the script that guides you through the installation.

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/add-tailscale-lxc.sh)"
```

#### 5.2.3. Step 3: Follow the On-Screen Prompts

The script is interactive. It will ask for your confirmation and then prompt you to enter the **Container ID (CTID)** of the container where you wish to install Tailscale.

1. When prompted, enter the CTID for your Nginx Proxy Manager container.
2. The script will then perform all necessary actions automatically.
    

#### 5.2.4. Step 4: Understanding the Automation

This single helper script automates several crucial steps that are difficult to perform manually:

- **Host-side Configuration:** It automatically edits the container's configuration file (e.g., /etc/pve/lxc/101.conf) on the Proxmox host to grant permissions for the necessary /dev/net/tun network device. This is a critical step that cannot be done from inside the container.
- **Container-side Installation:** It then enters the container and runs the standard Tailscale installation process.
- **Service & Persistence:** It correctly configures the tailscaled service to start automatically on boot, resolving potential issues with systemd in minimal LXC environments. This eliminates the need for manual workarounds like cron jobs.

#### 5.2.5. Step 5: Authenticate the Container

After the script finishes, the final step is to authenticate the new device.

1. Open the console for your NPM container.
2. Run the command tailscale up.
3. You will be given a URL. Copy this URL and open it in a browser on your primary computer to log in to your Tailscale account and authorize the new machine.

### 5.3. SSL Certificate Configuration via DNS Challenge

To get a valid SSL certificate for a private service, the DNS-01 challenge is required.

- **Create a Cloudflare API Token:** In your Cloudflare dashboard, create an API token with Zone:DNS:Edit permissions for your domain.
- **Add Certificate in NPM:** In the NPM SSL Certificates section, add a new Let's Encrypt certificate.
    - For flexibility, use a wildcard: yourdomain.com and *.yourdomain.com.
    - Enable **"Use a DNS Challenge"**.
    - Select Cloudflare as the provider and paste your API token into the credentials field using the format dns_cloudflare_api_token = <YOUR_TOKEN>.
    - The warning "These domains must be... configured to point to this installation" can be ignored, as it only applies to the HTTP challenge.

### 5.4. Local DNS Resolution with the hosts File

Your computer must be told how to find your private services using their public-facing domain names.

1. **Identify the Target IP:** Find the **Tailscale IP address of your NPM container** (e.g., 100.x.y.z). All your services will point to this single IP.
2. **Edit** **hosts** **File:** On your client machine, edit the system's hosts file.
    - **Windows:** C:\Windows\System32\drivers\etc\hosts. Use PowerShell as Administrator: Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "[NPM Tailscale IP] service.yourdomain.com"
    - **macOS/Linux:** /etc/hosts
3. **Add Entries:** For every new service you configure in NPM, you must add a corresponding new line to your hosts file, all pointing to the same NPM Tailscale IP.

### 5.5. Configuring the Proxy Host

This is the final step where all pieces are tied together.

1. **Create Proxy Host:** In NPM, create a new proxy host for your service (e.g., proxmox.yourdomain.com).
2. **Destination:** The Forward Hostname / IP should point to the service's **Local LAN IP address** (e.g., 192.168.1.127). This was found to be more reliable than using the service's Tailscale address, bypassing potential routing complexities between LXC containers over the Tailscale network.
3. **SSL:** Select the wildcard certificate you created in step 5.3 and enable Force SSL.

**Backend SSL Verification (Critical Fix):** If the backend service (like Proxmox) uses HTTPS with a self-signed certificate, NPM will show it as "Offline" and connections will fail. To fix this, go to the Advanced tab of the Proxy Host and add the following line to disable backend certificate verification:  
Nginx  
proxy_ssl_verify off;

1. **Application-Specific Settings:** Some applications like the Proxmox console require WebSocket support. If enabling the "Websocket Support" toggle or adding the full advanced snippet causes issues, the most reliable workaround is to access the application via the proxy for general use, and via its direct Tailscale address (https://hostname:port) when a specific feature (like the console) is needed.

### 5.6 Implementing a Centralized DNS Solution (Split DNS)

While editing the hosts file on each client device is effective for testing or for a single user, it quickly becomes cumbersome to maintain across multiple devices (laptops, phones, desktops). A more robust and scalable solution is to use Split DNS.

The concept is to run a private DNS server on your network that all your Tailscale devices will consult _only_ for queries related to your private domain. This provides a single source of truth for your internal services, eliminating the need for any client-side configuration.

#### 5.6.1. Step 1: Deploy a Local DNS Server

The easiest way to do this is by using a dedicated application like **Pi-hole** or **AdGuard Home**, which provide user-friendly web interfaces for managing DNS.

1. **Create a new LXC Container** in Proxmox dedicated to your DNS server (e.g., "adguard-lxc").
2. **Install AdGuard Home or Pi-hole** inside this container by following their official installation guides.
3. **Install Tailscale** inside this new container using the same method as for the NPM container (curl... | sh, tailscaled &, and setting up the cron job for persistence).
4. Take note of the new container's **Tailscale IP address**.

#### 5.6.2. Step 2: Configure Local DNS Records

In your AdGuard Home or Pi-hole web interface, find the section for "DNS Rewrites" or "Local DNS Records". Here, you will map your custom domain names to your Nginx Proxy Manager container.

The most efficient method is to use a wildcard record. This single rule will cover proxmox., jellyfin., and any other service you add in the future.

- Create a new DNS record:
    - **Domain:** *.jonathans-labb.org
    - **IP Address:** 100.89.67.32 (The **Tailscale IP of your Nginx Proxy Manager container**)

This rule tells your local DNS server: "Any request for a subdomain of jonathans-labb.org should be answered with the IP address of the NPM proxy."

#### 5.6.3. Step 3: Configure Tailscale to Use Your DNS Server

Now, you must instruct your entire Tailscale network to use your new DNS server for your private domain.

1. Go to the **DNS** page in your Tailscale Admin Console.
2. Under the **Nameservers** section, click Add nameserver.
3. Select the **"Custom"** option and enter the **Tailscale IP of your AdGuard/Pi-hole container**.
4. After adding the nameserver, a toggle for **"Restrict to search domain (Split DNS)"** will appear. **Enable this toggle.**
5. In the text box that appears, enter your private domain: jonathans-labb.org.

Your configuration should look like this:

This tells every device on your Tailnet: "For any address ending in .jonathans-labb.org, ask the private DNS server. For everything else (like google.com), use the regular internet DNS."

#### 5.6.4. Step 4: Cleanup and Verification

1. **Remove the custom entries** you previously added to the hosts file on your Windows desktop, laptop, and any other clients. They are no longer needed and can cause conflicts. You may need to flush your computer's DNS cache (ipconfig /flushdns in Windows Command Prompt) or restart your browser.
2. **Verify:** From any device connected to your Tailnet, you should now be able to access https://proxmox.jonathans-labb.org, https://truenas.jonathans-labb.org, etc., without any local configuration. The name resolution is now handled centrally and automatically for your entire private network.
    
## 6. Final Root Cause Summary

**The series of issues stemmed from a few key principles of this architecture:**

1. **Network Membership:** A device cannot communicate on the Tailscale network unless it is running the Tailscale client.
2. **Container Environments:** Minimalist LXC containers may lack a full systemd init system, requiring manual service management for tailscaled.
3. **DNS Resolution:** A public domain name must be manually mapped to a private Tailscale IP on each client device (via hosts file or Split DNS) to resolve correctly.
4. **Proxy Trust:** A reverse proxy will, by default, not trust a backend server that uses a self-signed SSL certificate. This verification must be explicitly disabled for private setups.
5. **Internal Routing:** The most reliable path for the final hop (NPM -> Service) was the local LAN bridge, bypassing potential complexities with Tailscale's routing inside the Proxmox host.

## 7. Solution: The Final Architecture Recipe

**To add any new service to this secure setup, follow this repeatable recipe:**

1. **In Nginx Proxy Manager:**
    - Create a new Proxy Host (e.g., service.yourdomain.com).
    - Set the destination to the service's **Local LAN IP** and port.
    - Select your existing wildcard Let's Encrypt certificate and force SSL.
    - If the backend uses a self-signed certificate, add proxy_ssl_verify off; to the Advanced tab.
2. **On your Client PC:**
    - Add a new line to your hosts file: [NPM's Tailscale IP] service.yourdomain.com.
3. **Access your service** securely at https://service.yourdomain.com.

## 8. Preventive Measures

- **Graceful Shutdowns:** Before rebooting the Proxmox host, it is a recommended best practice to gracefully shut down the LXC containers (especially NPM). This ensures network services terminate cleanly and helps prevent network state corruption or connectivity issues upon restart.
- **Network Priority:** If you experience issues accessing general local network devices (e.g., your router, other PCs) while Tailscale is active, check your client's Tailscale settings. Ensure that "Use Tailscale DNS settings" is only enabled if needed. Crucially, verify that you are not using an "Exit Node" on your Tailnet, as this feature routes _all_ of your computer's traffic through a different Tailscale machine, which can prevent access to your immediate local network.
    

## 9. Appendix: Commands Used

- 1: **Test Connectivity:** ping <ip_address>
- 2: **Check Tailscale Status:** tailscale status
- 3: **Get Tailscale IP:** tailscale ip -4
- 4: **Detailed Connection Test:** curl -v -k 
- 5: **Start Tailscale daemon manually:** tailscaled &
- 6: **Add startup job:** crontab -e then add @reboot /usr/sbin/tailscaled >/dev/null 2>&1


Detailed Connection Test:
	curl -v -k https://<hostname>:<port>
Force DNS Lookup via MagicDNS:
	nslookup <hostname> 100.100.100.100
Add to Windows hosts file:
	Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "<IP> <hostname>"