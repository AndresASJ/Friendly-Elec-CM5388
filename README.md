# Friendly Elec CM3588 Setup Guide

This guide outlines the process of setting up a Friendly Elec CM3588 single-board computer with Ubuntu, Docker, Samba file sharing, Jellyfin media server, and PhotoPrism photo management software.

## Table of Contents

1. [Hardware Overview](#hardware-overview)
2. [Operating System Installation](#operating-system-installation)
3. [Docker Installation](#docker-installation)
4. [Portainer Setup](#portainer-setup)
5. [Samba File Sharing Setup](#samba-file-sharing-setup)
6. [Jellyfin Media Server Setup](#jellyfin-media-server-setup)
7. [Cloudflare Tunnel Configuration](#cloudflare-tunnel-configuration)
8. [PhotoPrism Setup](#photoprism-setup)

## Hardware Overview

- **SBC**: Friendly Elec CM3588
- **Storage**:
  - Team Group MP44L SSD
  - Sandisk Ultra 512GB SD card

The CM3588 features a Rockchip RK3588 processor and supports up to 16GB RAM and 64GB eMMC storage.

## Operating System Installation

1. Download the Ubuntu Jammy image from the Friendly Elec wiki.
2. Use Win32DiskImager to flash the image to the SD card.
3. Use elfasher to transfer the OS from the SD card to the eMMC memory.

> Note: It's recommended to use the Jammy version over Focal for better hardware acceleration.

## Docker Installation

```bash
# Update package list
sudo apt update

# Install Docker
sudo apt install -y docker.io

# Start and enable Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-linux-aarch64" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installations
docker --version
docker-compose --version
```

## Portainer Setup

1. Create a volume for Portainer data:
   ```bash
   sudo docker volume create portainer_data
   ```

2. Deploy Portainer container:
   ```bash
   sudo docker run -d -p 8000:8000 -p 9443:9443 --name=portainer --restart=always \
   -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data \
   portainer/portainer-ce:latest
   ```

3. Access Portainer at `https://<YOUR_SERVER_IP>:9443` and complete the initial setup.

## Samba File Sharing Setup

1. Install Samba:
   ```bash
   sudo apt install samba -y
   ```

2. Configure Samba:
   ```bash
   sudo nano /etc/samba/smb.conf
   ```
   Add the following configuration:
   ```ini
   [global]
   server string = File Server
   workgroup = WRKGRP
   security = user
   map to guest = Bad User
   name resolve order = bcast host
   include = /etc/samba/shares.conf
   ```

3. Create and configure shares:
   ```bash
   sudo nano /etc/samba/shares.conf
   ```
   Add your share configuration:
   ```ini
   [Your Name]
   path = /share/your_name
   force user = smbuser
   force group = smbgroup
   create mask = 0664
   force create mode = 0664
   directory mask = 0775
   force directory mode = 0775
   public = yes
   writable = yes
   ```

4. Set up directories and permissions:
   ```bash
   sudo mkdir -p /share/your_name
   sudo groupadd --system smbgroup
   sudo useradd --system --no-create-home --group smbgroup -s /bin/false smbuser
   sudo chown -R smbuser:smbgroup /share
   sudo chmod -R g+w /share
   ```

5. Start Samba service:
   ```bash
   sudo systemctl start smbd
   ```

## Jellyfin Media Server Setup

1. Create a directory for Jellyfin:
   ```bash
   mkdir ~/jellyfin
   cd ~/jellyfin
   ```

2. Create a docker-compose.yml file:
   ```bash
   nano docker-compose.yml
   ```
   Add the following content:
   ```yaml
   version: "3.8"
   services:
     jellyfin:
       image: jellyfin/jellyfin:latest
       container_name: jellyfin
       volumes:
         - /path/to/your/config:/config
         - /path/to/your/cache:/cache
         - /path/to/your/media:/media
       ports:
         - 8096:8096
       restart: unless-stopped
   ```

3. Start Jellyfin:
   ```bash
   sudo docker-compose up -d
   ```

4. Allow Jellyfin through the firewall:
   ```bash
   sudo ufw allow 8096/tcp
   ```

## Cloudflare Tunnel Configuration

1. Install Cloudflared:
   ```bash
   wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
   sudo dpkg -i cloudflared-linux-arm64.deb
   ```

2. Authenticate and create a tunnel:
   ```bash
   cloudflared tunnel login
   cloudflared tunnel create jellyfin-tunnel
   ```

3. Configure the tunnel:
   ```bash
   sudo nano /etc/cloudflared/config.yml
   ```
   Add the following content:
   ```yaml
   tunnel: [YOUR_TUNNEL_ID]
   credentials-file: /root/.cloudflared/[YOUR_TUNNEL_ID].json
   ingress:
     - hostname: yourdomain
       service: http://localhost:8096
     - service: http_status:404
   ```

4. Set up Cloudflared as a service:
   ```bash
   sudo nano /etc/systemd/system/cloudflared.service
   ```
   Add the following content:
   ```ini
   [Unit]
   Description=cloudflared tunnel
   After=network.target

   [Service]
   TimeoutStartSec=0
   Type=notify
   ExecStart=/usr/local/bin/cloudflared --config /etc/cloudflared/config.yml --no-autoupdate run
   Restart=on-failure
   RestartSec=5s
   KillMode=process

   [Install]
   WantedBy=multi-user.target
   ```

5. Enable and start the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable cloudflared
   sudo systemctl start cloudflared
   ```

6. Configure DNS in Cloudflare dashboard:
   - Add a CNAME record pointing to your tunnel address.

## PhotoPrism Setup

1. Create required directories:
   ```bash
   sudo mkdir -p /your/drive/photoprism/{storage,import,config}
   ```

2. Create a docker-compose.yml file for PhotoPrism:
   ```bash
   mkdir ~/photoprism
   cd ~/photoprism
   nano docker-compose.yml
   ```
   Add the following content:
   ```yaml
   version: '3.8'
   services:
     photoprism:
       image: photoprism/photoprism:latest
       container_name: photoprism
       environment:
         PHOTOPRISM_ADMIN_USER: "admin"
         PHOTOPRISM_ADMIN_PASSWORD: "your_secure_password"
         PHOTOPRISM_ORIGINALS_LIMIT: 5000
         PHOTOPRISM_HTTP_PORT: 2342
         PHOTOPRISM_STORAGE_PATH: "/photoprism/storage"
         PHOTOPRISM_IMPORT_PATH: "/photoprism/import"
       volumes:
         - /your/drive/photoprism/storage:/photoprism/storage
         - /your/drive/photoprism/import:/photoprism/import
         - /your/drive/photoprism/config:/photoprism/config
       ports:
         - 2342:2342
       restart: unless-stopped
   ```

3. Set up permissions:
   ```bash
   sudo groupadd photoprism
   sudo usermod -aG photoprism $USER
   sudo usermod -aG photoprism photoprism
   sudo chown -R $USER:photoprism /your/drive/photoprism/{storage,import,config}
   sudo chmod -R 775 /your/drive/photoprism/{storage,import,config}
   ```

4. Start PhotoPrism:
   ```bash
   cd ~/photoprism
   sudo docker-compose up -d
   ```

5. Allow PhotoPrism through the firewall:
   ```bash
   sudo ufw allow 2342/tcp
   ```

6. Access PhotoPrism at `http://your_server_ip:2342` and complete the initial setup.

This guide will be updated with more information as the setup evolves.
