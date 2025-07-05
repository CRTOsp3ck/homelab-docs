# Harbor Docker Registry Setup Guide for Proxmox Homelab

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [VM Preparation](#vm-preparation)
3. [Docker Installation](#docker-installation)
4. [Harbor Installation](#harbor-installation)
5. [SSL/TLS Configuration](#ssltls-configuration)
6. [Harbor Configuration](#harbor-configuration)
7. [User Management](#user-management)
8. [Testing the Registry](#testing-the-registry)
9. [Backup and Maintenance](#backup-and-maintenance)
10. [Troubleshooting](#troubleshooting)

## Prerequisites

- Proxmox VE server running
- Domain name or subdomain for the registry (e.g., `registry.yourdomain.com`)
- DNS record pointing to your server's IP
- Basic knowledge of Linux command line
- Internet connection for downloading packages

## VM Preparation

### 1. Create a New VM in Proxmox

1. In Proxmox web interface, click "Create VM"
2. **General Tab:**
   - VM ID: Choose available ID (e.g., 200)
   - Name: `harbor-registry`
   - Resource Pool: (optional)

3. **OS Tab:**
   - ISO Image: Ubuntu Server 22.04 LTS (recommended)
   - Guest OS: Linux 6.x - 2.6 Kernel

4. **System Tab:**
   - Graphic card: Default
   - Machine: Default (q35)
   - BIOS: Default (SeaBIOS)
   - Add EFI Disk: No (unless needed)

5. **Hard Disk Tab:**
   - Storage: Choose your storage
   - Disk size: **100GB minimum** (Harbor requires significant space)
   - Format: qcow2
   - Cache: Write back
   - Discard: Yes
   - SSD emulation: Yes (if using SSD storage)

6. **CPU Tab:**
   - Sockets: 1
   - Cores: **4 minimum** (Harbor is resource-intensive)
   - Type: host

7. **Memory Tab:**
   - Memory: **8GB minimum** (Harbor needs at least 4GB, more recommended)
   - Ballooning: Yes

8. **Network Tab:**
   - Bridge: vmbr0 (or your main bridge)
   - Model: VirtIO (paravirtualized)
   - Firewall: Yes (optional)

### 2. Install Ubuntu Server

1. Start the VM and install Ubuntu Server 22.04 LTS
2. During installation:
   - Choose minimal installation
   - Enable SSH server
   - Create a user account (e.g., `harbor`)
   - Do NOT install Docker during OS installation

3. After installation, update the system:
```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

## Docker Installation

### 1. Install Docker Engine

```bash
# Remove any old Docker installations
sudo apt remove docker docker-engine docker.io containerd runc

# Update package index
sudo apt update

# Install packages to allow apt to use repository over HTTPS
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up the repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install Docker Engine
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add current user to docker group
sudo usermod -aG docker $USER

# Log out and back in, or run:
newgrp docker

# Test Docker installation
docker --version
docker run hello-world
```

### 2. Install Docker Compose (Standalone)

```bash
# Download Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make it executable
sudo chmod +x /usr/local/bin/docker-compose

# Test installation
docker-compose --version
```

## Harbor Installation

### 1. Download Harbor

```bash
# Create harbor directory
mkdir -p ~/harbor
cd ~/harbor

# Download latest Harbor release (check https://github.com/goharbor/harbor/releases for latest version)
wget https://github.com/goharbor/harbor/releases/download/v2.9.1/harbor-online-installer-v2.9.1.tgz

# Extract the archive
tar xvf harbor-online-installer-v2.9.1.tgz

# Move into harbor directory
cd harbor
```

### 2. Configure Harbor

```bash
# Copy the configuration template
cp harbor.yml.tmpl harbor.yml

# Edit the configuration file
nano harbor.yml
```

**Key configuration changes in `harbor.yml`:**

```yaml
# The IP address or hostname to access admin UI and registry service
hostname: registry.yourdomain.com  # Change this to your domain

# HTTP settings
http:
  port: 80

# HTTPS settings (comment out for initial installation)
# https:
#   port: 443
#   certificate: /data/cert/registry.yourdomain.com.crt
#   private_key: /data/cert/registry.yourdomain.com.key

# Harbor admin password (change this!)
harbor_admin_password: YourStrongPasswordHere

# Database settings
database:
  password: YourDbPasswordHere
  max_idle_conns: 100
  max_open_conns: 900

# Data volume
data_volume: /data

# Log settings
log:
  level: info
  local:
    rotate_count: 50
    rotate_size: 200M
    location: /var/log/harbor

# Uncomment external_url if you have a reverse proxy
# external_url: https://registry.yourdomain.com

# Chart repository service
chart:
  absolute_url: disabled

# Jobservice
jobservice:
  max_job_workers: 10

# Notification settings
notification:
  webhook_job_max_retry: 3

# Registry settings
registry:
  max_replica: 3
  poll_interval: 10s
  storage:
    filesystem:
      rootdirectory: /storage
```

### 3. Install Harbor

```bash
# Run the installation script
sudo ./install.sh

# If you want to install with additional components:
# sudo ./install.sh --with-notary --with-trivy --with-chartmuseum
```

The installation will:
- Download required Docker images
- Create necessary directories
- Generate configuration files
- Start Harbor services

**Note:** Harbor will initially run on HTTP only. We'll add SSL/TLS in the next section.

## SSL/TLS Configuration (After Installation)

### Option 1: Self-Signed Certificate (Development/Testing)

```bash
# Create certificate directory
sudo mkdir -p /data/cert

# Generate private key
sudo openssl genrsa -out /data/cert/registry.yourdomain.com.key 4096

# Generate certificate signing request
sudo openssl req -new -key /data/cert/registry.yourdomain.com.key -out /data/cert/registry.yourdomain.com.csr

# Generate self-signed certificate
sudo openssl x509 -req -days 365 -in /data/cert/registry.yourdomain.com.csr -signkey /data/cert/registry.yourdomain.com.key -out /data/cert/registry.yourdomain.com.crt

# Set proper permissions
sudo chmod 644 /data/cert/registry.yourdomain.com.crt
sudo chmod 600 /data/cert/registry.yourdomain.com.key
```

### Option 2: Let's Encrypt Certificate (Production)

```bash
# Install Certbot
sudo apt install -y certbot

# Stop Harbor temporarily
cd ~/harbor/harbor
sudo docker-compose down

# Get certificate (replace with your domain and email)
sudo certbot certonly --standalone -d registry.yourdomain.com --email your-email@domain.com --agree-tos

# Copy certificates to Harbor directory
sudo cp /etc/letsencrypt/live/registry.yourdomain.com/fullchain.pem /data/cert/registry.yourdomain.com.crt
sudo cp /etc/letsencrypt/live/registry.yourdomain.com/privkey.pem /data/cert/registry.yourdomain.com.key

# Set proper permissions
sudo chmod 644 /data/cert/registry.yourdomain.com.crt
sudo chmod 600 /data/cert/registry.yourdomain.com.key
```

### 4. Reconfigure Harbor with SSL

```bash
# Stop Harbor
sudo docker-compose down

# Edit harbor.yml to uncomment and configure HTTPS section
nano harbor.yml

# Uncomment the HTTPS section:
# https:
#   port: 443
#   certificate: /data/cert/registry.yourdomain.com.crt
#   private_key: /data/cert/registry.yourdomain.com.key

# Prepare Harbor with new configuration
sudo ./prepare

# Start Harbor with SSL
sudo docker-compose up -d
```

## Harbor Configuration

### 1. Access Harbor Web Interface

1. **Initial HTTP Access:**
   - Open your browser and navigate to `http://registry.yourdomain.com`
   - Login with:
     - Username: `admin`
     - Password: (the password you set in harbor.yml)

2. **After SSL Configuration:**
   - Access via `https://registry.yourdomain.com`
   - You may need to accept the self-signed certificate warning

### 2. Initial Configuration

1. **Change Admin Password:**
   - Go to "Users" → Click on "admin" → "Change Password"

2. **Configure Registry Settings:**
   - Go to "Configuration" → "System Settings"
   - Set appropriate values for:
     - Project creation restriction
     - Self-registration
     - Verify remote certificate

3. **Configure Authentication (Optional):**
   - Go to "Configuration" → "Authentication"
   - Configure LDAP/OIDC if needed

## User Management

### 1. Create Users

```bash
# Via Web Interface:
# 1. Go to "Users" → "New User"
# 2. Fill in user details
# 3. Assign appropriate roles

# Via CLI (optional):
# First, get Harbor CLI tool
wget https://github.com/goharbor/harbor/releases/download/v2.9.1/harbor-cli-linux-amd64.tar.gz
tar xvf harbor-cli-linux-amd64.tar.gz
sudo mv harbor-cli /usr/local/bin/
```

### 2. Create Projects

1. Go to "Projects" → "New Project"
2. Set project name (e.g., "my-app")
3. Configure access level (Public/Private)
4. Set vulnerability scanning if enabled

### 3. Assign User Roles

1. Go to "Projects" → Select project → "Members"
2. Add users with appropriate roles:
   - **Project Admin**: Full project control
   - **Developer**: Push/pull images
   - **Guest**: Pull images only
   - **Maintainer**: Manage project settings

## Testing the Registry

### 1. Login to Registry

```bash
# Login to Harbor registry
docker login registry.yourdomain.com

# Enter username and password when prompted
```

### 2. Push a Test Image

```bash
# Pull a test image
docker pull nginx:latest

# Tag the image for your registry
docker tag nginx:latest registry.yourdomain.com/my-app/nginx:latest

# Push the image
docker push registry.yourdomain.com/my-app/nginx:latest
```

### 3. Pull the Image

```bash
# Remove local image
docker rmi registry.yourdomain.com/my-app/nginx:latest

# Pull from registry
docker pull registry.yourdomain.com/my-app/nginx:latest
```

## Backup and Maintenance

### 1. Regular Backups

Create a backup script:

```bash
#!/bin/bash
# Harbor backup script

BACKUP_DIR="/backup/harbor"
DATE=$(date +%Y%m%d_%H%M%S)
HARBOR_DIR="/home/harbor/harbor"

# Create backup directory
mkdir -p $BACKUP_DIR

# Stop Harbor
cd $HARBOR_DIR
sudo docker-compose down

# Backup data directory
sudo tar -czf $BACKUP_DIR/harbor_data_$DATE.tar.gz /data

# Backup Harbor configuration
sudo cp -r $HARBOR_DIR $BACKUP_DIR/harbor_config_$DATE

# Start Harbor
sudo docker-compose up -d

# Keep only last 7 days of backups
find $BACKUP_DIR -name "harbor_*" -mtime +7 -delete

echo "Backup completed: $DATE"
```

### 2. Auto-renewal for Let's Encrypt

```bash
# Create renewal script
sudo nano /etc/cron.d/harbor-ssl-renewal

# Add content:
0 3 * * * root /usr/bin/certbot renew --quiet --deploy-hook "cd /home/harbor/harbor && docker-compose restart proxy"
```

### 3. Log Rotation

Harbor logs are automatically rotated based on the configuration in `harbor.yml`. Monitor disk usage:

```bash
# Check Harbor logs
sudo docker-compose logs -f

# Check disk usage
df -h
du -sh /data
```

### 4. Regular Maintenance

```bash
# Update Harbor (when new version is available)
cd ~/harbor/harbor
sudo docker-compose down
cd ..
# Download new version, backup old config, update, and restart

# Clean up Docker system
docker system prune -f

# Check Harbor health
docker-compose ps
```

## Troubleshooting

### Common Issues

1. **Harbor services not starting:**
   ```bash
   # Check logs
   sudo docker-compose logs

   # Check system resources
   free -h
   df -h
   ```

2. **SSL certificate issues:**
   ```bash
   # Check certificate validity
   openssl x509 -in /data/cert/registry.yourdomain.com.crt -text -noout

   # Verify certificate chain
   openssl verify /data/cert/registry.yourdomain.com.crt
   ```

3. **Docker login fails:**
   ```bash
   # Check if Harbor is running
   docker-compose ps

   # Test connectivity
   curl -k https://registry.yourdomain.com

   # Check firewall
   sudo ufw status
   ```

4. **Performance issues:**
   ```bash
   # Monitor resources
   htop
   iotop

   # Check Docker stats
   docker stats
   ```

### Useful Commands

```bash
# Harbor management
cd ~/harbor/harbor
sudo docker-compose start    # Start Harbor
sudo docker-compose stop     # Stop Harbor
sudo docker-compose restart  # Restart Harbor
sudo docker-compose ps       # Check status

# View logs
sudo docker-compose logs -f [service_name]

# Update configuration
sudo ./prepare
sudo docker-compose up -d

# Database operations
sudo docker exec -it harbor-db-1 psql -U postgres -d registry
```

## Security Considerations

1. **Firewall Configuration:**
   ```bash
   sudo ufw enable
   sudo ufw allow 22/tcp      # SSH
   sudo ufw allow 80/tcp      # HTTP
   sudo ufw allow 443/tcp     # HTTPS
   ```

2. **Regular Updates:**
   - Keep Harbor updated
   - Update base OS regularly
   - Monitor security advisories

3. **Access Control:**
   - Use strong passwords
   - Implement proper RBAC
   - Regular audit of users and permissions

4. **Network Security:**
   - Use VPN for remote access
   - Implement proper network segmentation
   - Monitor access logs

## Conclusion

Your Harbor registry is now ready for use! You can:
- Push and pull Docker images
- Manage users and projects
- Scan images for vulnerabilities (if Trivy is installed)
- Set up automated builds and deployments

Remember to:
- Regularly backup your registry data
- Monitor system resources
- Keep Harbor and the underlying system updated
- Review and rotate credentials periodically

For advanced features like replication, webhook notifications, and integration with CI/CD systems, refer to the official Harbor documentation.
