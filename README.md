# Homelab Manuals

A collection of hardware-agnostic, step-by-step guides for building, configuring, and operating self-hosted services and infrastructure.

## Purpose
These guides are designed to serve as universal templates. Whether you are deploying on an old laptop, a Raspberry Pi, or an enterprise-grade cluster, these manuals provide a stable, secure, and reproducible foundation for your homelab architecture.

## Index of Guides

### Network & Security
* [Secure Homelab Access with Tailscale and Nginx Proxy Manager](./network/secure-access-tailscale-npm.md)
* [Configuring a VLAN-based Firewall Switch](./network/vlan-switch-configuration.md)
* [Creating a Web-Honeypot with Integrated Wazuh Agent](./network/web-honeypot-wazuh.md)

### Infrastructure & Proxmox VE
* [Emergency Migration from a Failing Host Disk](./infrastructure/emergency-disk-migration.md)
* [Backing Up to an External USB Drive](./infrastructure/external-usb-backup.md)
* [Stable & Automated LXC Container Backups](./infrastructure/automated-lxc-backups.md)

### Services & Docker
* [Self-Hosting Websites with Docker and a Reverse Proxy](./services/self-hosting-websites.md)
* [Private Service Monitoring with Uptime Kuma and Tailscale](./services/uptime-kuma-monitoring.md)
* [Secure Torrenting Appliance with Docker](./services/secure-torrenting-vpn.md)

## Prerequisites
To get the most out of these guides, it is recommended that you have:
* A basic understanding of the Linux command line.
* A machine with SSH access.
* Docker and Docker Compose installed for the service-related guides.
* Administrative access to your network routing equipment.

## Author
I'm Jonathan, and I develop projects in my spare time that help myself and others become better and more efficient developers!
* [LinkedIn](https://www.linkedin.com/in/jonathan-windell-418a55232/)
* [Portfolio](https://portfolio.jonathans-labb.org/)

## License
This project is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/). You are free to share and adapt the material, provided you give appropriate credit and distribute your contributions under the same license.
