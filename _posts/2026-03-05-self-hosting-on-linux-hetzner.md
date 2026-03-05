---
layout: post
title: "Self-Hosting on Any Linux Box — A Practical Guide with Hetzner"
date: 2026-03-05
author: "Finn Rades"
tags: [linux, self-hosting, hetzner, docker, sysadmin]
description: "A step-by-step walkthrough for spinning up your first self-hosted server — from ordering a Hetzner VPS to running your first service behind a reverse proxy."
---

So you want to host your own stuff. Maybe you're tired of relying on third-party SaaS, want full control of your data, or just want to learn how infrastructure actually works. Whatever the reason — this guide will walk you through everything from ordering a server to running your first service, using Hetzner as the example provider.

This isn't just a copy-paste tutorial. I'll explain what each step actually does and why, so you come out the other end understanding your own setup.

---

## Why Hetzner?

There are plenty of VPS providers out there — DigitalOcean, Linode, Vultr, OVH. Hetzner is my go-to for a few reasons:

- **Price** — genuinely hard to beat. A CX22 (2 vCPU, 4GB RAM) starts at ~€4.35/month.
- **Location** — EU-based, relevant if you care about GDPR or latency to European users.
- **Quality** — solid uptime, fast network, good console access via the Cloud dashboard.
- **Simplicity** — the Hetzner Cloud console is clean and doesn't try to upsell you on 40 services.

You can use any other provider and follow along — the Linux steps are identical.

---

## Step 1 — Order Your Server

1. Create an account at [console.hetzner.cloud](https://console.hetzner.cloud)
2. Create a new **Project**
3. Click **+ Add Server**

**Recommended settings for a starter box:**

| Setting | Value |
|---------|-------|
| Location | Nuremberg or Falkenstein (EU) |
| Image | **Ubuntu 24.04** |
| Type | **CX22** (2 vCPU, 4 GB RAM) — enough for multiple services |
| Networking | Public IPv4 + IPv6 |
| SSH Keys | Add your public key (see below) |
| Firewall | Create one — we'll configure it next |

**Generating an SSH key if you don't have one:**

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

Copy your public key and paste it into Hetzner:

```bash
cat ~/.ssh/id_ed25519.pub
```

---

## Step 2 — Secure Your Server

Your server is publicly accessible the moment it boots. First thing to do: lock it down.

### Connect

```bash
ssh root@YOUR_SERVER_IP
```

### Create a non-root user

Never run everything as root. Create a regular user with sudo access:

```bash
useradd -m -s /bin/bash -G sudo finn
passwd finn
```

Copy your SSH key to the new user:

```bash
mkdir -p /home/finn/.ssh
cp /root/.ssh/authorized_keys /home/finn/.ssh/
chown -R finn:finn /home/finn/.ssh
chmod 700 /home/finn/.ssh
chmod 600 /home/finn/.ssh/authorized_keys
```

Test this works before proceeding:

```bash
ssh finn@YOUR_SERVER_IP
```

### Harden SSH

Edit the SSH daemon config:

```bash
sudo nano /etc/ssh/sshd_config
```

Change or add these lines:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

> **Important:** don't close your current session until you've confirmed you can reconnect with the new settings.

### Firewall with UFW

```bash
sudo apt install ufw -y

# Deny everything by default
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH, HTTP, HTTPS
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw enable
sudo ufw status
```

You can also configure a firewall at the Hetzner level (before traffic even hits your server) via the Cloud console under **Firewalls**.

### Automatic security updates

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Step 3 — Install Docker

Docker is the easiest way to run services in isolation. Each app lives in its own container with its own dependencies — no conflicts, easy to update, easy to remove.

```bash
# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Allow your user to run docker without sudo
sudo usermod -aG docker finn
newgrp docker
```

Verify it works:

```bash
docker run hello-world
```

---

## Step 4 — Set Up a Reverse Proxy with Traefik

You'll want to run multiple services on the same server, each on a different subdomain (`grafana.example.com`, `nextcloud.example.com`, etc.). A **reverse proxy** sits in front of them all, terminates HTTPS, and routes traffic to the right container.

[Traefik](https://traefik.io) is great for this because it automatically discovers Docker containers and handles Let's Encrypt certificates without any manual configuration.

### Point a domain at your server

In your DNS provider, create an A record:

```
*.example.com  →  YOUR_SERVER_IP
example.com    →  YOUR_SERVER_IP
```

The wildcard `*` means any subdomain will resolve to your server. Traefik then decides where to route it.

### Create the Docker network

All containers that need to be reachable via Traefik must be on a shared network:

```bash
docker network create traefik-proxy
```

### Traefik compose file

Create a directory for Traefik:

```bash
mkdir -p ~/services/traefik && cd ~/services/traefik
touch acme.json && chmod 600 acme.json
```

Create `docker-compose.yml`:

```yaml
services:
  traefik:
    image: traefik:v3.3
    container_name: traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-proxy"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--certificatesresolvers.letsencrypt.acme.email=your@email.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.tlschallenge=true"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./acme.json:/acme.json
    networks:
      - traefik-proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.example.com`)"
      - "traefik.http.routers.dashboard.entrypoints=websecure"
      - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=admin:$$apr1$$..."  # htpasswd hash

networks:
  traefik-proxy:
    external: true
```

> Replace `your@email.com` and `example.com` with your actual values. Generate a dashboard password with `htpasswd -nb admin yourpassword`.

Start it:

```bash
docker compose up -d
docker compose logs -f
```

---

## Step 5 — Deploy Your First Service

Let's deploy [Uptime Kuma](https://github.com/louislam/uptime-kuma) — a self-hosted status page and uptime monitor. Simple, useful, and a good test case.

```bash
mkdir -p ~/services/uptime-kuma && cd ~/services/uptime-kuma
```

`docker-compose.yml`:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - ./data:/app/data
    networks:
      - traefik-proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.uptime-kuma.rule=Host(`status.example.com`)"
      - "traefik.http.routers.uptime-kuma.entrypoints=websecure"
      - "traefik.http.routers.uptime-kuma.tls.certresolver=letsencrypt"
      - "traefik.http.services.uptime-kuma.loadbalancer.server.port=3001"

networks:
  traefik-proxy:
    external: true
```

```bash
docker compose up -d
```

Navigate to `https://status.example.com` — Traefik will have already provisioned a certificate. Set up your account on first visit.

---

## Step 6 — Keeping Things Updated

### Update a service

```bash
cd ~/services/uptime-kuma
docker compose pull
docker compose up -d
```

### System updates

```bash
sudo apt update && sudo apt upgrade -y
```

---

## What's Next?

You've got a secured server, a reverse proxy with auto-HTTPS, and at least one service running. From here you can:

- Add more services — Nextcloud, Gitea, Vaultwarden, Grafana, anything with a Docker image
- Set up [Portainer](https://www.portainer.io) for a GUI over Docker
- Configure automated backups (Restic + Backblaze B2 is a solid combo)
- Move to Kubernetes if you outgrow Docker Compose — but that's a whole other post

Self-hosting has a learning curve, but it pays off. You understand your own stack, control your own data, and pick up skills that translate directly to production engineering work.

Questions? Ping me on Discord: **zonlykroks** or join [discord.gg/XUn9Kt5dsy](https://discord.gg/XUn9Kt5dsy).
