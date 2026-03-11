## 1. Introduction

A web honeypot acts as a decoy, appearing as a legitimate service (like a router admin panel) to attract and log unauthorized access attempts. By placing this decoy in an isolated network segment, you can safely monitor attacker behavior and gain early warnings of scanning activity.

This setup uses a lightweight **Nginx** container on a **Raspberry Pi 3** to serve a fake login page. All interaction data is captured by the **Wazuh Agent**, sent to the **Wazuh Manager**, and flagged as a high-priority security event when specific "traps" (like a `/login-attempt` URL) are triggered.

## 2. Environment & Prerequisites

**This guide assumes the following:**

- **Raspberry Pi:** A Pi 3 or newer running a Linux distribution with **Docker** and **Wazuh Agent** installed.
- **Wazuh Environment:** A running Wazuh Manager (e.g. LXC container) accessible by the Pi.
- **Network Isolation:** The Raspberry Pi should be placed in an isolated **VLAN** (e.g., VLAN 20) to prevent lateral movement.

## 3. Part 1: Deploying the Web Honeypot

#### 3.1. Create the Honeypot Directory

On your Raspberry Pi, create a dedicated directory for the honeypot files:
- `mkdir ~/web-honeypot && cd ~/web-honeypot`
- `mkdir html logs`
#### 3.2. Create the Fake Login Page

Create an `index.html` file that mimics a sensitive login portal:
- `nano html/index.html`

```html
<!DOCTYPE html>
<html>
<head><title>Router Admin Panel</title></head>
<body style="background:#f4f4f4; text-align:center; padding-top:50px;">
    <h2>UniFi Gateway - Login</h2>
    <form action="/login-attempt" method="POST">
        <input type="text" name="user" placeholder="Username"><br><br>
        <input type="password" name="pass" placeholder="Password"><br><br>
        <button type="submit">Login</button>
    </form>
</body>
</html>
```

#### 3.3. Configure Docker Compose

Create a `docker-compose.yml` file to run the Nginx service:
- `nano docker-compose.yml`

```yaml
services:
  honeypot-web:
    image: nginx:stable-alpine
    container_name: web-trap
    restart: always
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./logs:/var/log/nginx
```

- Start the container: `docker compose up -d`.

## 4. Part 2: Network Configuration (Port Forwarding)
To make the honeypot visible to the internet, you must forward a "tempting" port through your routers.
#### 4.1. Step 1: Secondary Gateway (UCG-Ultra)

In the UniFi Network interface, create a **Port Forwarding** rule:
- **Name:** Raspberry Pi - Web Honeypot
- **WAN Port:** 8081
- **Forward IP:** Forward IP: `<RASPBERRY_PI_IP>`
- **Forward Port:** 8080
- **Protocol:** TCP

## 5. Part 3: Integrating with Wazuh

#### 5.1. Configure the Wazuh Agent (Raspberry Pi)

The agent must be told to monitor the Nginx access log.
1. Open the agent configuration: `sudo nano /var/ossec/etc/ossec.conf`
2. Add the following block before the closing `</ossec_config>` tag:


```xml
<localfile>
  <log_format>apache</log_format>
  <location>/home/<your-username>/web-honeypot/logs/access.log</location>
</localfile>
```

3. Restart the agent: `sudo systemctl restart wazuh-agent`.

#### 5.2. Configure the Wazuh Manager (VM 108)

Create custom rules to elevate the priority of honeypot triggers.
1. In the Wazuh Web UI, go to **Server Management > Rules > local_rules.xml**.
2. Add the following rules:

```xml
<group name="web,honeypot,">
  <rule id="100010" level="10">
    <if_sid>31101</if_sid>
    <match>/login-attempt</match>
    <description>Honeypot Trigger: Login try detected on false portal!</description>
  </rule>
</group>
```

3. Save and restart the Wazuh Manager.

## 6. Verification & Testing

#### 6.1. Test the Access
From an external device (e.g., a smartphone on a mobile network), navigate to `http://[Your-Public-IP]:8081`. You should see the fake login page.

#### 6.2. Verify in Wazuh
1. Enter dummy credentials into the honeypot and click **Login**.
2. Open the **Wazuh Dashboard** and navigate to **Threat Hunting** or **Discover**.
3. Search for `rule.id: 100010`.
4. You should see a **Level 10** alert with the description defined in Step 5.2.
