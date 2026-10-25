# proxmox-vaultwarden-tailscale
Self-hosted password manager deployment documentation.

## Project Goal

Setting up a secure, self-hosted password manager (Vaultwarden/Bitwarden) on a home server. The goal was to run it in a lightweight, isolated environment without exposing local network ports to the public internet, while maintaining the strict HTTPS encryption required by Bitwarden clients.

## Hardware & Technologies

* **Hardware:** Dell Optiplex (Home Server)
* **Hypervisor:** Proxmox VE
* **Containerization:** LXC (Linux Containers), Docker, Docker Compose
* **Software:** Vaultwarden (lightweight Bitwarden server written in Rust)
* **Networking & Security:** Tailscale (Mesh VPN), MagicDNS for automated SSL/HTTPS

## Main Challenges & Setup Process

* **Storage Configuration (LVM to LVM-Thin):** Initially, the secondary drive (`hdd`) was formatted as standard LVM, which did not support LXC templates or snapshots. I reconfigured it as an LVM-Thin pool (`hdd-thin`) to enable container cloning and templating. This required manually removing ghost container configurations via the Proxmox shell.
* **LXC Container for Docker:** To keep the Proxmox host clean and save RAM, I avoided installing Docker directly on the host. Instead, I deployed an unprivileged Debian LXC container with the **Nesting** feature enabled, which is strictly required to run Docker within LXC.
* **Tailscale TUN Device Issue:** To access the vault securely and generate an SSL certificate, I installed Tailscale inside the container. However, the Tailscale daemon kept crashing because LXC blocks access to the virtual network adapter (`/dev/net/tun`) by default. I resolved this by editing the container's configuration file on the Proxmox host.
* **Automated HTTPS:** Once Tailscale was running, I utilized Tailscale's MagicDNS to automatically provision a valid SSL certificate and route secure HTTPS traffic to the local Vaultwarden Docker container, bypassing the need for a reverse proxy.

## Key Commands & Configuration

**Removing Ghost LXC Containers (Proxmox Host)**
After wiping the physical drive, stale container configurations remained. I cleared them via the host shell:

```bash
rm /etc/pve/lxc/101.conf
rm /etc/pve/lxc/102.conf

```

**LXC Container TUN Device Fix (Proxmox Host)**
To allow Tailscale to use the TUN device inside the unprivileged container, I added the following lines to `/etc/pve/lxc/101.conf`:

```text
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file

```

**Vaultwarden Docker Compose Setup**
The container was deployed using the following `docker-compose.yml` configuration:

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - ./vw-data:/data
    ports:
      - "8080:80"

```

**Tailscale HTTPS Routing (Inside LXC)**
Commands used to spin up the mesh VPN and route HTTPS traffic to the Docker container port:

```bash
tailscale up
tailscale serve --bg localhost:8080

```

## Final Result

A highly secure, low-footprint password manager running 24/7. It remains completely invisible to the public internet but instantly synchronizes across all my personal devices via the Tailscale VPN mesh network.
