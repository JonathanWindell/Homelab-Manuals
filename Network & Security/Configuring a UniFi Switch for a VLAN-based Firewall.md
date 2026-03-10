## 1. Introduction

This guide provides a comprehensive walkthrough for configuring a UniFi managed switch to support a "Router on a Stick" firewall setup, such as a virtualized OPNsense instance running on Proxmox. The objective is to use a single switch to manage traffic for multiple, isolated networks (e.g., an untrusted WAN network and a trusted LAN network) that are routed by a firewall with only one physical network connection.

This architecture relies on VLAN tagging to logically separate traffic. This guide details the complete process, from the initial adoption of the switch to the specific port profiles required to create a functional and secure multi-network environment.
## 2. Environment & Prerequisites

**This guide assumes the following:**

- **UniFi Switch:** A manageable UniFi switch (e.g., USW-Flex-XG).
- **UniFi Network Application:** Access to a device running the UniFi Controller software (either temporarily on a laptop/mobile or permanently on a server).
- **Existing Network:** A primary, functional network with a DHCP server (e.g., a home router) that the switch can connect to for initial setup.
- **Firewall:** A router/firewall (like OPNsense) that is configured to use VLANs for its WAN and LAN interfaces.

## 3. Part 1: Adopting the Switch (The "Chicken-and-Egg" Problem)

A UniFi switch must be "adopted" by a controller before it can be configured. This creates a classic challenge: the controller and the switch must be on the same network to communicate, but the switch's job is to create new, separate networks.

### 3.1. Step 1: The Temporary Setup

To solve this, we first connect everything to the existing, primary network.

1. Connect your UniFi Switch to a power source.
2. Connect a network cable from your primary home router to any port on the UniFi Switch.
3. Ensure the device running your UniFi Controller (e.g., a laptop) is also connected to this same primary network (via Wi-Fi or cable).

### 3.2. Step 2: Adoption and Troubleshooting

1. Launch the UniFi Network Application. The new switch should appear under the "Devices" tab with a "Pending Adoption" status.
2. Click `Adopt`. The switch will begin provisioning.
3. **Troubleshooting "Device Unreachable":** It is very common for the status to change to "Unreachable". This is almost always caused by the host computer's firewall (e.g., Windows Defender) blocking the switch's attempt to communicate back to the controller.
    - **Solution:** **Temporarily disable the firewall** on the computer running the controller. The switch's status should quickly change to "Online". Remember to **re-enable your firewall** immediately after adoption is complete.

## 4. Part 2: Programming the Switch Ports

**With the switch adopted and online, we can now program its ports for their permanent roles. This is done in the **Port Manager** (`Devices -> Select Switch -> Ports`).**

### 4.1. Step 1: Create the LAN VLAN

1. Go to `Settings` ⚙️ -> `Networks`.
2. Click `Create New Network`.
3. Select the `VLAN Only` network type.
4. **Name:** Give it a descriptive name (e.g., `LabbNetwork`).
5. **VLAN ID:** Enter the ID that matches your firewall's LAN configuration (e.g., `10`).
6. Save the network.

### 4.2. Step 2: Configure Port Profiles

Now, assign the correct profiles to the physical ports.

1. **The WAN Access Port (e.g., Port 1):** This port will receive internet traffic from your primary router.
    - **Name:** `WAN-Input`
    - **Port Profile / Native VLAN:** `Default`
    - **Tagged VLAN Management:** `Block All`
2. **The TRUNK Port (e.g., Port 2):** This is the multi-lane highway to your firewall.
    - **Name:** `TRUNK-to-Firewall`
    - **Port Profile / Native VLAN:** `Default`
    - **Tagged VLAN Management:** Select `Custom` and check the box for your `LabbNetwork [10]`.
3. **The LAN Access Port (e.g., Port 3):** This port is for a device in your lab.
    - **Name:** `LAN-Device`
    - **Port Profile / Native VLAN:** `LabbNetwork [10]`
    - **Tagged VLAN Management:** `Block All`

## 5. Solution: The Final Architecture

Once the ports are programmed, you can perform the final physical connection:

1. A cable from your primary router connects to the **WAN Access Port** (Port 1).
2. A cable from your firewall connects to the **TRUNK Port** (Port 2).
3. Your lab devices connect to their respective **LAN Access Ports** (Port 3, etc.).

The switch will now correctly isolate and direct traffic. Untagged traffic from the primary router will be sent to the firewall on the` `Default network to serve as its WAN. Traffic from your lab devices will be tagged with VLAN 10, sent through the TRUNK port to the firewall, processed, and routed out to the internet, creating a secure, multi-network environment.