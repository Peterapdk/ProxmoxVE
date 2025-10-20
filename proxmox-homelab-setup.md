# Proxmox Homelab Setup with ARR Services and Tailscale Integration

## Overview
This guide provides a comprehensive setup for a Proxmox homelab optimized for ARR (Automated Request Router) services with deep Tailscale integration for secure remote access and routing.

## 1. Proxmox Base Configuration

### Initial Setup
```bash
# Update Proxmox
apt update && apt upgrade -y

# Configure hostname
hostnamectl set-hostname proxmox-homelab

# Set timezone
timedatectl set-timezone America/New_York

# Configure static IP (adjust for your network)
nano /etc/network/interfaces
```

### Security Hardening
```bash
# Install fail2ban
apt install fail2ban -y

# Configure firewall
ufw enable
ufw allow 8006/tcp  # Proxmox web interface
ufw allow 22/tcp    # SSH
ufw allow 8000/tcp  # Tailscale web interface

# Disable root login
nano /etc/ssh/sshd_config
# Set: PermitRootLogin no
systemctl restart ssh
```

## 2. Tailscale Deep Integration

### Install Tailscale on Proxmox Host
```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Start and enable Tailscale
systemctl enable --now tailscaled

# Authenticate (run this interactively)
tailscale up
```

### Configure Tailscale Subnet Routing
```bash
# Enable subnet routing
tailscale up --advertise-routes=192.168.1.0/24,10.0.0.0/8

# Set up exit node (optional)
tailscale up --advertise-exit-node
```

### Tailscale Configuration Files
```bash
# Create Tailscale config directory
mkdir -p /etc/tailscale

# Configure Tailscale for persistent settings
cat > /etc/tailscale/tailscaled.conf << EOF
# Tailscale configuration
FLAGS="--state=/var/lib/tailscale/tailscaled.state --socket=/run/tailscale/tailscaled.sock --port=41641"
EOF
```

## 3. Storage Strategy

### ZFS Configuration
```bash
# Install ZFS utilities
apt install zfsutils-linux -y

# Create ZFS pool (adjust drives as needed)
zpool create -f tank mirror /dev/sdb /dev/sdc

# Configure ZFS datasets
zfs create tank/vms
zfs create tank/containers
zfs create tank/media
zfs create tank/backups
zfs create tank/isos

# Set compression and other optimizations
zfs set compression=lz4 tank
zfs set atime=off tank
zfs set sync=standard tank
```

### Storage Mount Points
```bash
# Create mount points
mkdir -p /mnt/media
mkdir -p /mnt/backups
mkdir -p /mnt/isos

# Mount datasets
zfs set mountpoint=/mnt/media tank/media
zfs set mountpoint=/mnt/backups tank/backups
zfs set mountpoint=/mnt/isos tank/isos
```

## 4. Network Architecture

### VLAN Configuration
```bash
# Install VLAN utilities
apt install vlan -y

# Configure VLANs in /etc/network/interfaces
cat >> /etc/network/interfaces << EOF

# Management VLAN
auto vmbr1
iface vmbr1 inet static
    address 192.168.1.10/24
    gateway 192.168.1.1
    bridge_ports none
    bridge_stp off
    bridge_fd 0

# Services VLAN (ARR services)
auto vmbr2
iface vmbr2 inet static
    address 192.168.2.10/24
    bridge_ports none
    bridge_stp off
    bridge_fd 0

# Media VLAN
auto vmbr3
iface vmbr3 inet static
    address 192.168.3.10/24
    bridge_ports none
    bridge_stp off
    bridge_fd 0
EOF
```

## 5. ARR Services Architecture

### LXC Container Template
```bash
# Create LXC template for ARR services
pct create 100 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
    --hostname arr-base \
    --memory 1024 \
    --cores 2 \
    --rootfs local-zfs:8 \
    --net0 name=eth0,bridge=vmbr2,ip=192.168.2.100/24,gw=192.168.2.1 \
    --unprivileged 1 \
    --onboot 1
```

### Docker Compose for ARR Services
```yaml
# /mnt/media/docker-compose.yml
version: '3.8'

services:
  # Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./ssl:/etc/nginx/ssl
    networks:
      - arr-network
    restart: unless-stopped

  # Sonarr
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./config/sonarr:/config
      - /mnt/media/tv:/tv
      - /mnt/media/downloads:/downloads
    networks:
      - arr-network
    restart: unless-stopped

  # Radarr
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./config/radarr:/config
      - /mnt/media/movies:/movies
      - /mnt/media/downloads:/downloads
    networks:
      - arr-network
    restart: unless-stopped

  # Prowlarr
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./config/prowlarr:/config
    networks:
      - arr-network
    restart: unless-stopped

  # Bazarr
  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./config/bazarr:/config
      - /mnt/media/movies:/movies
      - /mnt/media/tv:/tv
    networks:
      - arr-network
    restart: unless-stopped

  # qBittorrent
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - WEBUI_PORT=8080
    volumes:
      - ./config/qbittorrent:/config
      - /mnt/media/downloads:/downloads
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    networks:
      - arr-network
    restart: unless-stopped

  # Plex
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - VERSION=docker
    volumes:
      - ./config/plex:/config
      - /mnt/media/movies:/movies
      - /mnt/media/tv:/tv
      - /mnt/media/music:/music
    ports:
      - "32400:32400"
    networks:
      - arr-network
    restart: unless-stopped

networks:
  arr-network:
    driver: bridge
```

## 6. Tailscale Integration Scripts

### Tailscale Auth Script
```bash
#!/bin/bash
# /usr/local/bin/tailscale-auth.sh

# Tailscale authentication and setup
TAILSCALE_AUTH_KEY="your-auth-key-here"

# Authenticate with Tailscale
tailscale up --authkey="$TAILSCALE_AUTH_KEY" --accept-routes

# Configure subnet routing
tailscale up --advertise-routes=192.168.1.0/24,192.168.2.0/24,192.168.3.0/24

# Set up as exit node (optional)
# tailscale up --advertise-exit-node
```

### Tailscale Service Discovery
```bash
#!/bin/bash
# /usr/local/bin/tailscale-discovery.sh

# Update DNS records for services
tailscale set --advertise-tags=tag:arr-services

# Configure service discovery
cat > /etc/tailscale/hosts << EOF
# ARR Services
arr-proxy.tailnet-name.ts.net    nginx
arr-sonarr.tailnet-name.ts.net   sonarr
arr-radarr.tailnet-name.ts.net   radarr
arr-plex.tailnet-name.ts.net     plex
EOF
```

## 7. Monitoring and Logging

### Install Prometheus and Grafana
```bash
# Create monitoring LXC
pct create 200 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
    --hostname monitoring \
    --memory 2048 \
    --cores 2 \
    --rootfs local-zfs:20 \
    --net0 name=eth0,bridge=vmbr1,ip=192.168.1.200/24,gw=192.168.1.1 \
    --unprivileged 1 \
    --onboot 1
```

### Monitoring Stack
```yaml
# /mnt/media/monitoring/docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
```

## 8. Backup Strategy

### Automated Backup Script
```bash
#!/bin/bash
# /usr/local/bin/backup-homelab.sh

BACKUP_DIR="/mnt/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup Proxmox VMs
for vm in $(qm list | grep -v VMID | awk '{print $1}'); do
    echo "Backing up VM $vm"
    qm backup $vm $BACKUP_DIR/vm-${vm}-${DATE}.tar.zst
done

# Backup LXC containers
for ct in $(pct list | grep -v CTID | awk '{print $1}'); do
    echo "Backing up CT $ct"
    pct backup $ct $BACKUP_DIR/ct-${ct}-${DATE}.tar.zst
done

# Backup configuration files
tar -czf $BACKUP_DIR/proxmox-config-${DATE}.tar.gz /etc/pve /etc/network/interfaces

# Cleanup old backups (keep 30 days)
find $BACKUP_DIR -name "*.tar.zst" -mtime +30 -delete
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete
```

### Cron Job for Backups
```bash
# Add to crontab
0 2 * * * /usr/local/bin/backup-homelab.sh >> /var/log/backup.log 2>&1
```

## 9. Security Considerations

### Firewall Rules
```bash
# /etc/ufw/before.rules
# Tailscale rules
-A ufw-before-input -i tailscale0 -j ACCEPT
-A ufw-before-output -o tailscale0 -j ACCEPT

# ARR services VLAN
-A ufw-before-input -i vmbr2 -j ACCEPT
-A ufw-before-output -o vmbr2 -j ACCEPT
```

### SSL/TLS Configuration
```bash
# Install certbot for Let's Encrypt
apt install certbot -y

# Create SSL certificates for Tailscale domains
certbot certonly --standalone -d arr-proxy.tailnet-name.ts.net
```

## 10. Performance Optimization

### Proxmox Tuning
```bash
# /etc/sysctl.conf optimizations
echo "vm.swappiness=10" >> /etc/sysctl.conf
echo "vm.dirty_ratio=15" >> /etc/sysctl.conf
echo "vm.dirty_background_ratio=5" >> /etc/sysctl.conf
echo "net.core.rmem_max=134217728" >> /etc/sysctl.conf
echo "net.core.wmem_max=134217728" >> /etc/sysctl.conf

# Apply changes
sysctl -p
```

### ZFS Tuning
```bash
# ZFS performance tuning
echo "options zfs zfs_arc_max=2147483648" >> /etc/modprobe.d/zfs.conf
echo "options zfs zfs_arc_min=1073741824" >> /etc/modprobe.d/zfs.conf
```

## 11. Remote Access Setup

### Tailscale ACL Configuration
```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:arr-services"],
      "dst": ["tag:homelab:*"]
    },
    {
      "action": "accept",
      "src": ["tag:admin"],
      "dst": ["tag:proxmox:*"]
    }
  ],
  "tagOwners": {
    "tag:arr-services": ["user:admin@example.com"],
    "tag:admin": ["user:admin@example.com"],
    "tag:proxmox": ["user:admin@example.com"]
  }
}
```

## 12. Maintenance Scripts

### System Update Script
```bash
#!/bin/bash
# /usr/local/bin/update-homelab.sh

# Update Proxmox
apt update && apt upgrade -y

# Update Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Update Docker containers
cd /mnt/media
docker-compose pull
docker-compose up -d

# Cleanup
apt autoremove -y
docker system prune -f
```

## Next Steps

1. **Initial Setup**: Follow the base configuration steps
2. **Tailscale Setup**: Configure Tailscale authentication and routing
3. **Storage Configuration**: Set up ZFS pools and datasets
4. **Network Configuration**: Configure VLANs and bridges
5. **ARR Services**: Deploy Docker containers for ARR services
6. **Monitoring**: Set up Prometheus and Grafana
7. **Backup**: Configure automated backup strategy
8. **Security**: Implement firewall rules and SSL certificates
9. **Testing**: Test all services and remote access
10. **Documentation**: Document your specific configuration

## Troubleshooting

### Common Issues
- **Tailscale not connecting**: Check firewall rules and network configuration
- **ARR services not accessible**: Verify Docker networking and port mappings
- **Storage issues**: Check ZFS pool status and disk health
- **Performance issues**: Review resource allocation and tuning parameters

### Useful Commands
```bash
# Check Tailscale status
tailscale status

# Check ZFS pool status
zpool status

# Check container status
docker ps -a

# Check Proxmox logs
journalctl -u pve-cluster
```

This setup provides a robust, secure, and scalable homelab environment with deep Tailscale integration for remote access and routing.