## 1. Introduction

This guide provides a comprehensive, step-by-step process for setting up a secure, scalable, and robust web hosting environment on a Proxmox VE server. The goal is to leverage modern tools to host multiple websites, each in its own isolated environment, and make them securely accessible to the public internet.

By using a Proxmox LXC container as a lightweight Docker host and Nginx Proxy Manager (NPM) as a reverse proxy, you gain full control over your web projects, from simple static portfolios to complex multi-service applications. This architecture provides an excellent foundation for a personal lab (homelab) or small-scale production environment, emphasizing security, organization, and scalability.

## 2. Environment & Prerequisites

**Before you begin, ensure you have the following in place:**

- **Hardware:** A computer of any form factor (desktop, server, NUC) with Proxmox VE installed and running.
- **Software:** A standard, up-to-date installation of Proxmox VE.
- **Networking:**
    - A home or office network with a router you can administrate.
    - The ability to configure **Port Forwarding** on your router.
- **Services:**
    - A registered **domain name** (e.g., `your-domain.com`).
    - A free **Cloudflare account** managing the DNS for your domain.
    - A **Dynamic DNS (DDNS)** hostname (e.g., from your router manufacturer like TP-Link, or a service like No-IP) that is kept updated with your home's public IP address.
- **Skills:** Basic familiarity with the Linux command line.

## 3. Part 1: Preparing the Proxmox VE Environment

**Our foundation will be a lightweight and efficient LXC (Linux Container) which will serve as our dedicated host for Docker.**

### 3.1. Step 1: Create a Dedicated LXC Container

1. **Download a Template:** In the Proxmox web interface, navigate to your server node -> `local` storage -> `CT Templates`. Click the `Templates` button and download the **Debian 12 Standard** template.
2. **Create the Container:** Click the `**Create CT**` button in the top-right corner and configure the following tabs:
    - **General:**
        - **Hostname:** `docker-host`
        - **Password:** Set a strong root password for this container.
    - **Template:** Select the `debian-12-standard` template you just downloaded.
    - **Disks:** The default 8GB is sufficient to start.
    - **CPU:** 1 or 2 cores is a good starting point.
    - **Memory:** 1024MB (1GB) is sufficient for running several web services.
    - **Network:**
        - **IPv4:** `Static`
        - **IPv4/CIDR:** Assign a static IP address from your local network, e.g., `192.168.0.110/24`. Make sure this IP is not in use.
        - **Gateway:** Enter your router's IP address, e.g., `192.168.0.1`.
    - **DNS:** Leave as-is or use your router's IP address.
3. Review the summary and click `**Finish**`.

### 3.2. Step 2: Initial System Update

1. Start your new `docker-host` container from the Proxmox interface.
2. Open its console by selecting the container and clicking `**>_ Console**`.
3. Log in as `root` with the password you just set.
4. Run a full system update to ensure all packages are current:
	```bash
	apt update && apt upgrade -y
	```

## 4. Part 2: Installing and Configuring the Docker Environment

**With the LXC prepared, we will now install Docker and create our project structure.**

### 4.1. Step 1: Application Installation (Docker)

1. Inside the `docker-host` console, run the official Docker installation script. This is the most reliable way to get the latest version.
    ```bash
    curl -fsSL https://get.docker.com -o get-docker.sh
    sh get-docker.sh`
    ```

2. Install the Docker Compose plugin, which allows us to manage multi-container applications easily.
	```bash
	apt install -y docker-compose-plugin
	```

### 4.2. Step 2: Application Configuration (Project Structure)

To keep all projects organized, we will create a central directory structure on the `docker-host`.
1. Create a parent directory for all your Docker configurations and data:
    `mkdir /opt/docker-projects
    `cd /opt/docker-projects/portfolio-site`
    
2. Create your master configuration file. This file will define all of your website containers.
    Bash
    `nano docker-compose.yml`

### 4.3. Step 3: Critical - Understanding Storage Configuration

The key to Docker's power is how it handles data using **volumes**. In our `docker-compose.yml` file, we will link a folder on our `docker-host` to a folder inside a container.

The syntax `- "/opt/docker-projects/my-website:/usr/share/nginx/html:ro"` means:

- `"/opt/docker-projects/portfolio-site/my-website"`: A folder on the `docker-host`. This is where we will place our actual website files.
- `"/usr/share/nginx/html"`: The webroot folder inside the Nginx container.
- `:ro`: A crucial security flag that makes the folder **read-only** inside the container, preventing the website from modifying its own source files.
    

## 5. Part 3: Deploying Your First Website

**Let's deploy a simple static website. This pattern can be repeated for every new site.**

#### 5.1. Step 1: Prepare Website Files on the Host

1. On the `docker-host`, create a folder for your portfolio's data:
    ```bash
    mkdir -p /opt/docker-projects/portfolio-data
    ```

2. Transfer your website files (e.g., `index.html`, `styles.css`) into this new folder. You can do this using the `pct push` command from the Proxmox host, as we troubleshooted earlier.

### **5.2. Step 2: Define the Service in Docker Compose**

1. Open your central configuration file:

    `nano /opt/docker-projects/docker-compose.yml`
    
2. Add the service definition for your portfolio website:
    ```yaml
    `version: '3.8' # This line is optional in modern versions but adds clarity.`
    `services:`
      `portfolio:`
        `image: "nginx:alpine"`
        `container_name: "portfolio-website"`
        `restart: unless-stopped`
        `ports:`
          `- "8081:80" # Map host port 8081 to container port 80`
        `volumes:`
          `- "/opt/docker-projects/portfolio-data:/usr/share/nginx/html:ro"`
    ```
    
3. Save and exit the file.
    

### 5.3. Step 3: Launch the Container

1. Make sure you are in the directory containing your configuration file:

    `cd /opt/docker-projects`
    
2. Launch the service:
    
    `docker compose up -d`
    
Your website is now running in a container, accessible on your local network at `http://<DOCKER_HOST_IP>:8081`.

### 5.4. Step 4: Configure the Reverse Proxy (NPM)

## Part 4: Configuring the Network Flow (Port Forwarding & Reverse Proxy)

Now that your website is running in a container on your local network, it's time to make it securely accessible to the outside world. We will do this in three steps: first, we'll tell our router where all web traffic should go; second, we'll point our domain to our network; and finally, we'll configure our "traffic cop" (NPM).

**The Core Principle:** Your router should **always** forward all web traffic to NPM. NPM then decides which specific website to display.

**Flow:** `Internet` -> `Your Router` -> `Nginx Proxy Manager (NPM)` -> `Your Website Container`

#### Step 5.4.1: Configure Port Forwarding in Your Router

This is the first and most critical step to allow external traffic into your network.

1. Log in to your router (the one that manages your internal network, e.g., your TP-Link).
2. Find the section for **"Port Forwarding"** or "Virtual Servers".
3. Create the following two rules. Both rules should point to the **IP address of your Nginx Proxy Manager (NPM) container**.
    - **Rule 1 (for secure HTTPS traffic):**
        - **External Port:** `443`
        - **Internal Port:** `443`
        - **Internal IP:** `[The IP address of your NPM container]` **(NOT the IP of your docker-host container!)**
        - **Protocol:** `TCP`
    - **Rule 2 (for HTTP traffic, needed for certificates):**
        - **External Port:** `80`
        - **Internal Port:** `80`
        - **Internal IP:** `[The same IP address for your NPM container]`
        - **Protocol:** `TCP`

#### Step 5.4.2: Configure DNS in Cloudflare

Now we need to tell the internet where to find your network.

1. In Cloudflare, ensure you have a `CNAME` record pointing to your DDNS hostname.
    - **Type:** `CNAME`
    - **Name:** `portfolio` (or another subdomain)
    - **Content/Target:** `your-ddns-name.tplinkdns.com`
    - **Proxy status:** For maximum security and performance, make sure the 'Proxy status' is set to **"Proxied" (orange cloud)**. This will hide your home IP address.

#### Step 5.4.3: Configure the Proxy Host in Nginx Proxy Manager (NPM)

Now that your router is forwarding all traffic to NPM, we must tell NPM what to do with the traffic specifically addressed to `portfolio.your-domain.com`.

1. Log in to your Nginx Proxy Manager web interface.
2. Go to `Hosts -> Proxy Hosts` and click `Add Proxy Host`.
3. Fill in the fields:
    - **Domain Names:** `portfolio.your-domain.com`
    - **Scheme:** `http`
    - **Forward Hostname / IP:** `<DOCKER_HOST_IP>` **(HERE, you enter the IP address of your** `**docker-host**` **container)**, as this is where your portfolio service is actually running.
    - **Forward Port:** `8081` (The unique port you exposed for this specific service in your `docker-compose.yml` file).
    - **SSL Tab:** Go to the SSL tab, select `Request a new SSL Certificate` (if it's the first time), and enable `Force SSL`.
    - **Access List:** Ensure it is set to "Publicly Accessible".
4. Click **Save**.
## 6. Solution: The Final Architecture

**You have now built a robust, multi-layered hosting solution. The flow for a public visitor is as follows:**

1. **User** requests `https://portfolio.your-domain.com`.
2. **Cloudflare DNS** resolves the domain via a `CNAME` record to your DDNS hostname.
3. The **DDNS Service** resolves the hostname to your home's current public IP address.
4. The request arrives at your **Router**, which forwards port 443 to your **Nginx Proxy Manager container**.
5. **NPM** processes the request, terminates the SSL, and uses its proxy rule to forward the traffic to the `**docker-host**` **container** at `http://192.168.0.110:8081`.
6. **Docker** receives the traffic on port 8081 and routes it to the correct **Nginx container**, which series the website files.
## 7. Preventive Measures & Best Practices

- **Backups:** Regularly use Proxmox's built-in backup feature to create snapshots of your critical containers (`NPM` and `docker-host`). Also, ensure you have a separate backup of your project data folders (e.g., `/opt/docker-projects`).
- **Updates:** Periodically log in to both your Proxmox host and your LXC containers to run system updates (`apt update && apt upgrade -y`).
- **Security:** Use the Proxmox firewall to restrict access to management ports. Do not expose any ports on your router unless they are managed by a reverse proxy like NPM.
- **Secrets Management:** For applications that require API keys or passwords, use a `.env` file in your project directory and the `env_file` directive in your `docker-compose.yml` file to securely inject them into the container. Never hard-code secrets in your source code or Docker images.
- **Organization:** Continue to use the "one container per service" model. For each new website, create a new data folder and add a new service to your central` `docker-compose.yml` `file, assigning it a new, unique host port.


#### Context 1: Proxmox Host Shell

| - Command                                                              | - Description                                                                                                              |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `pct list`                                                             | Lists all LXC containers on the node and shows their status (running/stopped).                                             |
| `pct enter [CT_ID]`                                                    | Opens a command-line shell **inside** a running container. (e.g., `pct enter 115`).                                        |
| `pct start [CT_ID]`                                                    | Starts a container.                                                                                                        |
| `pct stop [CT_ID]`                                                     | Safely shuts down a container.                                                                                             |
| `pct reboot [CT_ID]`                                                   | Reboots a container.                                                                                                       |
| `pct push [CT_ID] [source] [dest]`                                     | Pushes a file from the host into a container. **Example:** `pct push 115 /var/lib/vz/template/iso/file.zip /root/file.zip` |
| `find /var/lib/vz -name "*.iso"`                                       | Searches the Proxmox storage for a specific file (e.g., your uploaded ISOs).                                               |
| `tar -czvf portfolio.tar.gz PersonalPortfolio`                         | Packs up files to be sent to the correct LXC container.                                                                    |
| `pct push [ID] portfolio.tar.gz /opt/docker-projects/portfolio.tar.gz` | Cmd for pushing files to container.                                                                                        |
| `ls /opt/docker-projects/PersonalPortfolio`                            | Lists all running docker images.                                                                                           |
| `tar -xzvf portfolio.tar.gz`                                           | Assembles files for use.                                                                                                   |
| `docker compose up -d --force-recreate`                                | Recreates the running containers.                                                                                          |
#### Context 2: Docker Host LXC Shell 

| **Command**                       | **Description**                                                                                                                                     |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `docker ps`                       | Lists all running Docker containers. Use this to check if your websites are up.                                                                     |
| `docker ps -a`                    | Lists all Docker containers, including stopped ones.                                                                                                |
| `docker logs [container_name]`    | Shows the logs for a specific container. Essential for debugging.                                                                                   |
| `docker logs -f [container_name]` | Follows the logs in real-time. (Press `Ctrl+C` to exit).                                                                                            |
| `cd /opt/docker-projects`         | Navigates to the directory where your main `docker-compose.yml` file and project data are located.                                                  |
| `docker compose up -d`            | The most common command. Starts or updates all services defined in your `docker-compose.yml` file. Run this from the directory containing the file. |
| `docker compose down`             | Stops and removes all containers, networks, etc., defined in your `docker-compose.yml`. Your data volumes are not touched.                          |
| `docker compose restart`          | Restarts all services in your compose file.                                                                                                         |
| `docker compose pull`             | Downloads the latest versions of all images used in your compose file (e.g., `nginx:alpine`).                                                       |
| `docker stop [container_name]`    | Manually stops a specific container.                                                                                                                |
| `docker rm [container_name]`      | Manually removes a stopped container.                                                                                                               |

#### How to find files 
- To find any file named `docker-compose.yml` anywhere on the system, use the `find` command:
	- `find / -name "docker-compose.yml"`
- To list the contents of your project directory with details:    
	- `ls -lA /opt/docker-projects/`
- (Inside an Nginx container) Lists the files in the web root to verify they were mounted correctly.
	- `ls -l /usr/share/nginx/html`
- Exits the container's shell and returns you to the `docker-host` shell.
	- `exit`

 