# Development VM/Container Options for Proxmox Management

## Overview
You need a development environment that can:
- Run Cursor IDE effectively
- Manage Proxmox (via web UI, CLI, or API)
- Handle development tasks
- Integrate with your Tailscale network
- Access your ARR services

## Option 1: Ubuntu LXC Container (Recommended)

### Why LXC is Best for This Use Case
- **Lightweight**: Uses minimal resources compared to full VMs
- **Fast**: Near-native performance
- **Easy Management**: Can be managed like any other Proxmox container
- **Resource Efficient**: Perfect for development work
- **GUI Support**: Can run GUI applications with X11 forwarding

### LXC Container Setup
```bash
# Create Ubuntu 22.04 LXC container
pct create 300 local:vztmpl/ubuntu-22.04-standard_22.04-1_amd64.tar.zst \
    --hostname dev-workstation \
    --memory 4096 \
    --cores 4 \
    --rootfs local-zfs:32 \
    --net0 name=eth0,bridge=vmbr1,ip=192.168.1.150/24,gw=192.168.1.1 \
    --unprivileged 0 \
    --onboot 1 \
    --features nesting=1

# Start the container
pct start 300

# Enter the container
pct enter 300
```

### LXC Container Configuration
```bash
# Update system
apt update && apt upgrade -y

# Install development tools
apt install -y curl wget git vim nano htop tree \
    build-essential software-properties-common \
    apt-transport-https ca-certificates gnupg lsb-release

# Install Node.js (for Cursor)
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Install Python and pip
apt install -y python3 python3-pip python3-venv

# Install Docker (for container management)
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker root

# Install Proxmox tools
apt install -y proxmox-ve-tools

# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
```

## Option 2: Ubuntu VM (Full Virtualization)

### When to Choose VM Over LXC
- **Hardware Passthrough**: Need GPU acceleration for development
- **Complex GUI**: Running heavy desktop environments
- **Docker-in-Docker**: Need to run nested virtualization
- **Windows Development**: Need Windows for specific tools

### VM Setup
```bash
# Create Ubuntu VM
qm create 400 \
    --name dev-vm \
    --memory 8192 \
    --cores 4 \
    --net0 virtio,bridge=vmbr1 \
    --scsihw virtio-scsi-pci \
    --scsi0 local-zfs:32 \
    --bootdisk scsi0 \
    --boot order=scsi0 \
    --ostype l26

# Download Ubuntu ISO
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.3-desktop-amd64.iso -P /mnt/isos/

# Attach ISO to VM
qm set 400 --cdrom local:iso/ubuntu-22.04.3-desktop-amd64.iso

# Start VM
qm start 400
```

## Option 3: Fedora LXC (Alternative)

### Why Fedora for Development
- **Cutting Edge**: Latest packages and tools
- **Developer Friendly**: Great for modern development
- **Container Native**: Excellent Docker/Podman support

```bash
# Create Fedora LXC
pct create 301 local:vztmpl/fedora-38-standard_38-1_amd64.tar.zst \
    --hostname fedora-dev \
    --memory 4096 \
    --cores 4 \
    --rootfs local-zfs:32 \
    --net0 name=eth0,bridge=vmbr1,ip=192.168.1.151/24,gw=192.168.1.1 \
    --unprivileged 0 \
    --onboot 1 \
    --features nesting=1
```

## Development Environment Setup

### Essential Development Tools
```bash
#!/bin/bash
# /usr/local/bin/setup-dev-env.sh

# Update system
apt update && apt upgrade -y

# Install development essentials
apt install -y \
    curl wget git vim nano htop tree \
    build-essential software-properties-common \
    apt-transport-https ca-certificates gnupg lsb-release \
    jq yq unzip zip

# Install Node.js (for Cursor)
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Install Python development tools
apt install -y python3 python3-pip python3-venv python3-dev
pip3 install --upgrade pip

# Install Go (optional)
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> /etc/profile

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker root

# Install Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Install Proxmox CLI tools
apt install -y proxmox-ve-tools

# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Install VS Code Server (alternative to Cursor)
curl -fsSL https://code-server.dev/install.sh | sh

# Install additional development tools
apt install -y \
    tmux screen \
    neofetch \
    tree \
    rsync \
    openssh-server \
    net-tools \
    dnsutils
```

### Cursor IDE Installation
```bash
# Download and install Cursor
wget https://download.cursor.sh/linux/appImage/x64 -O cursor.AppImage
chmod +x cursor.AppImage
mv cursor.AppImage /usr/local/bin/cursor

# Create desktop entry
cat > /usr/share/applications/cursor.desktop << EOF
[Desktop Entry]
Name=Cursor
Comment=AI-powered code editor
Exec=/usr/local/bin/cursor
Icon=cursor
Type=Application
Categories=Development;TextEditor;
EOF
```

## Proxmox Management Tools

### Proxmox CLI Tools
```bash
# Install Proxmox CLI
apt install -y proxmox-ve-tools

# Configure Proxmox API access
cat > ~/.proxmoxrc << EOF
export PVE_HOST=192.168.1.10
export PVE_USER=root@pam
export PVE_PASSWORD=your-password
export PVE_REALM=pam
EOF

# Source the configuration
echo 'source ~/.proxmoxrc' >> ~/.bashrc
```

### Proxmox Python API
```bash
# Install Proxmox Python API
pip3 install proxmoxer

# Create Proxmox management script
cat > /usr/local/bin/proxmox-manager.py << 'EOF'
#!/usr/bin/env python3
import proxmoxer
import os

# Connect to Proxmox
proxmox = proxmoxer.ProxmoxAPI(
    os.getenv('PVE_HOST'),
    user=os.getenv('PVE_USER'),
    password=os.getenv('PVE_PASSWORD'),
    verify_ssl=False
)

# List VMs
print("VMs:")
for vm in proxmox.cluster.resources.get(type='vm'):
    print(f"  {vm['vmid']}: {vm['name']} ({vm['status']})")

# List Containers
print("\nContainers:")
for ct in proxmox.cluster.resources.get(type='lxc'):
    print(f"  {ct['vmid']}: {ct['name']} ({ct['status']})")
EOF

chmod +x /usr/local/bin/proxmox-manager.py
```

## Tailscale Integration

### Tailscale Configuration
```bash
# Start Tailscale
tailscale up

# Configure Tailscale for development
tailscale set --advertise-tags=tag:dev-workstation

# Create Tailscale hosts file
cat > /etc/tailscale/hosts << EOF
# Development workstation
dev.tailnet-name.ts.net    dev-workstation

# Proxmox management
proxmox.tailnet-name.ts.net   192.168.1.10

# ARR services
arr-proxy.tailnet-name.ts.net   192.168.2.100
EOF
```

## Development Workflow Setup

### Git Configuration
```bash
# Configure Git
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Install GitHub CLI
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list
apt update
apt install -y gh
```

### Development Scripts
```bash
# Create development helper scripts
mkdir -p /usr/local/bin/dev-scripts

# Proxmox management script
cat > /usr/local/bin/dev-scripts/proxmox-status << 'EOF'
#!/bin/bash
echo "=== Proxmox Status ==="
pct list
echo ""
qm list
echo ""
echo "=== Resource Usage ==="
pvesm status
EOF

# ARR services management script
cat > /usr/local/bin/dev-scripts/arr-status << 'EOF'
#!/bin/bash
echo "=== ARR Services Status ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep -E "(sonarr|radarr|prowlarr|bazarr|plex|qbittorrent)"
EOF

# Tailscale status script
cat > /usr/local/bin/dev-scripts/tailscale-status << 'EOF'
#!/bin/bash
echo "=== Tailscale Status ==="
tailscale status
echo ""
echo "=== Tailscale IPs ==="
tailscale ip -4
EOF

chmod +x /usr/local/bin/dev-scripts/*
```

## Resource Allocation Recommendations

### LXC Container (Recommended)
- **Memory**: 4-8GB
- **CPU**: 4-6 cores
- **Storage**: 32-64GB
- **Network**: Bridge to management VLAN

### VM (If Needed)
- **Memory**: 8-16GB
- **CPU**: 4-8 cores
- **Storage**: 64-128GB
- **Network**: Bridge to management VLAN

## Security Considerations

### LXC Security
```bash
# Configure LXC security
pct set 300 --features nesting=1,keyctl=1

# Set up SSH key authentication
mkdir -p /root/.ssh
# Add your public key to /root/.ssh/authorized_keys
```

### Firewall Rules
```bash
# Configure UFW for development
ufw allow from 192.168.1.0/24 to any port 22
ufw allow from 192.168.2.0/24 to any port 22
ufw allow from 100.64.0.0/10 to any port 22  # Tailscale network
```

## Backup Strategy

### Development Environment Backup
```bash
# Create backup script for dev environment
cat > /usr/local/bin/backup-dev-env.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/mnt/backups/dev-env"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup development files
tar -czf $BACKUP_DIR/dev-env-${DATE}.tar.gz \
    /root/.ssh \
    /root/.gitconfig \
    /usr/local/bin/dev-scripts \
    /etc/tailscale/hosts

# Cleanup old backups
find $BACKUP_DIR -name "dev-env-*.tar.gz" -mtime +7 -delete
EOF

chmod +x /usr/local/bin/backup-dev-env.sh
```

## Recommendation

**Use Ubuntu LXC Container** for your development environment because:

1. **Resource Efficient**: Uses minimal resources
2. **Fast Performance**: Near-native speed
3. **Easy Management**: Can be managed like any Proxmox container
4. **GUI Support**: Can run Cursor with X11 forwarding
5. **Cost Effective**: Doesn't waste resources on full virtualization

The LXC approach gives you everything you need for development while being much more efficient than a full VM. You can always create a VM later if you need specific hardware passthrough or Windows development.

Would you like me to help you set up the LXC container with all the development tools configured?