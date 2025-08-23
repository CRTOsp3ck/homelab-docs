# OpenProject Self-Hosting Setup Guide

## Prerequisites

- Proxmox VE running
- Cloudflare account with domain configured
- DDNS updater already running on Proxmox
- Basic knowledge of Linux command line

## Phase 1: Proxmox VM Setup

### 1.1 Create a New VM

1. **Access Proxmox Web Interface**
   - Navigate to your Proxmox web interface
   - Click "Create VM" in the top-right corner

2. **VM Configuration**
   - **General Tab:**
     - Node: Select your Proxmox node
     - VM ID: Choose an available ID (e.g., 103)
     - Name: `openproject-server`
   
   - **OS Tab:**
     - Use CD/DVD disc image file (iso)
     - Select Ubuntu Server 22.04 LTS or Debian 12 ISO
   
   - **System Tab:**
     - Keep defaults (BIOS: Default, Machine: q35)
   
   - **Disks Tab:**
     - Bus/Device: SCSI
     - Storage: local-lvm (or your preferred storage)
     - Disk size: 32 GB minimum (recommended: 50GB+)
     - Cache: Write back
   
   - **CPU Tab:**
     - Sockets: 1
     - Cores: 2 minimum (recommended: 4)
   
   - **Memory Tab:**
     - Memory: 4096 MB minimum (recommended: 8192 MB)
   
   - **Network Tab:**
     - Bridge: vmbr0
     - Model: VirtIO (paravirtualized)

3. **Start VM and Install OS**
   - Start the VM and complete Ubuntu/Debian installation
   - Create a user account (e.g., `admin`)
   - Enable SSH during installation

### 1.2 Initial VM Configuration

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget git nano htop net-tools ufw

# Configure firewall
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443

# Set static IP (optional but recommended)
sudo nano /etc/netplan/00-installer-config.yaml
```

**Static IP Configuration Example:**
```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
      addresses:
        - 192.168.1.100/24  # Adjust to your network
      gateway4: 192.168.1.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Apply network configuration:
```bash
sudo netplan apply
```

## Phase 2: Docker Installation

### 2.1 Install Docker and Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Logout and login again to apply group changes
exit
```

### 2.2 Verify Installation

```bash
docker --version
docker-compose --version
```

## Phase 3: OpenProject Installation

### 3.1 Create Project Directory

```bash
mkdir -p ~/openproject
cd ~/openproject
```

### 3.2 Create Docker Compose Configuration

```bash
nano docker-compose.yml
```

**Docker Compose File:**
```yaml
version: '3.8'

services:
  db:
    image: postgres:14
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: p4ssw0rd
      POSTGRES_DB: openproject
      POSTGRES_USER: openproject
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - openproject

  cache:
    image: redis:7
    restart: unless-stopped
    networks:
      - openproject

  web:
    image: openproject/community:14
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://openproject:p4ssw0rd@db:5432/openproject
      RAILS_CACHE_STORE: "redis"
      OPENPROJECT_CACHE__REDIS__URL: "redis://cache:6379"
      OPENPROJECT_RAILS__RELATIVE__URL__ROOT: ""
      # Security
      SECRET_KEY_BASE: your-secret-key-base-change-this-to-something-random
      # Email configuration (optional)
      EMAIL_DELIVERY_METHOD: smtp
      SMTP_ADDRESS: your.smtp.server
      SMTP_PORT: 587
      SMTP_DOMAIN: yourdomain.com
      SMTP_AUTHENTICATION: plain
      SMTP_ENABLE_STARTTLS_AUTO: "true"
      SMTP_USER_NAME: your-email@yourdomain.com
      SMTP_PASSWORD: your-email-password
    ports:
      - "8080:80"
    volumes:
      - openproject_data:/var/openproject/assets
      - openproject_pgdump:/var/openproject/pgdump
    depends_on:
      - db
      - cache
    networks:
      - openproject

volumes:
  postgres_data:
  openproject_data:
  openproject_pgdump:

networks:
  openproject:
```

### 3.3 Generate Secret Key

```bash
# Generate a secure secret key
openssl rand -hex 64
```

Replace `your-secret-key-base-change-this-to-something-random` in the docker-compose.yml file with the generated key.

### 3.4 Start OpenProject

```bash
# Start services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f web
```

### 3.5 Initial Setup

1. Wait for all services to start (this may take several minutes)
2. Access OpenProject at `http://your-vm-ip:8080`
3. Default login: `admin` / `admin`
4. Follow the setup wizard to configure your instance

## Phase 4: Reverse Proxy Setup (Nginx)

### 4.1 Install Nginx

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

### 4.2 Create Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/openproject
```

**Nginx Configuration:**
```nginx
server {
    listen 80;
    server_name openproject.yourdomain.com;  # Replace with your domain

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name openproject.yourdomain.com;  # Replace with your domain

    # SSL Configuration (will be configured by Certbot)
    ssl_certificate /etc/letsencrypt/live/openproject.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/openproject.yourdomain.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;

    # Proxy settings
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Server $host;
        
        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # Security
    location ~ /\.ht {
        deny all;
    }
}
```

### 4.3 Enable Site and Test Configuration

```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/openproject /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## Phase 5: Cloudflare DNS Configuration

### 5.1 Add DNS Record

1. **Login to Cloudflare Dashboard**
   - Go to your domain's DNS settings

2. **Add A Record**
   - Type: A
   - Name: openproject (or your preferred subdomain)
   - IPv4 address: Your public IP (should be updated by your DDNS)
   - Proxy status: DNS only (gray cloud) - important for Let's Encrypt
   - TTL: Auto

### 5.2 Configure SSL/TLS Mode

1. **Go to SSL/TLS → Overview**
   - Set encryption mode to "Full (strict)" after SSL certificate is installed

## Phase 6: SSL Certificate Setup

### 6.1 Obtain Let's Encrypt Certificate

```bash
# Stop nginx temporarily
sudo systemctl stop nginx

# Obtain certificate
sudo certbot certonly --standalone -d openproject.yourdomain.com

# Start nginx
sudo systemctl start nginx

# Test automatic renewal
sudo certbot renew --dry-run
```

### 6.2 Update Cloudflare Proxy Settings

After SSL is working:
1. Go back to Cloudflare DNS settings
2. Change proxy status to "Proxied" (orange cloud) for additional security and performance

## Phase 7: Security Hardening

### 7.1 Update OpenProject Configuration

```bash
cd ~/openproject
nano docker-compose.yml
```

Remove the ports section from the web service (since we're using reverse proxy):
```yaml
# Remove or comment out:
# ports:
#   - "8080:80"
```

Restart OpenProject:
```bash
docker-compose down
docker-compose up -d
```

### 7.2 Configure Firewall

```bash
# Remove direct access to port 8080
sudo ufw delete allow 8080

# Verify firewall rules
sudo ufw status
```

### 7.3 Regular Maintenance

Create a backup script:
```bash
nano ~/backup-openproject.sh
```

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/home/$USER/openproject-backups"

mkdir -p $BACKUP_DIR

# Backup database
docker-compose -f /home/$USER/openproject/docker-compose.yml exec -T db pg_dump -U openproject openproject > $BACKUP_DIR/openproject_db_$DATE.sql

# Backup volumes
docker run --rm -v openproject_openproject_data:/data -v $BACKUP_DIR:/backup alpine tar czf /backup/openproject_data_$DATE.tar.gz -C /data .

# Keep only last 7 backups
find $BACKUP_DIR -name "openproject_*" -mtime +7 -delete

echo "Backup completed: $DATE"
```

Make it executable and add to cron:
```bash
chmod +x ~/backup-openproject.sh

# Add to crontab (daily backup at 2 AM)
crontab -e
# Add: 0 2 * * * /home/admin/backup-openproject.sh >> /var/log/openproject-backup.log 2>&1
```

## Phase 8: Final Configuration

### 8.1 OpenProject Admin Configuration

1. **Access your instance**: https://openproject.yourdomain.com
2. **Initial setup**:
   - Change admin password
   - Configure system settings
   - Set up your first project
   - Configure user authentication (LDAP/SAML if needed)
   - Set up email notifications

### 8.2 Performance Optimization

```bash
# Optimize Docker
echo '{"log-driver": "json-file", "log-opts": {"max-size": "10m", "max-file": "3"}}' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker

# Monitor resources
docker stats
htop
```

## Troubleshooting

### Common Issues

**Services won't start:**
```bash
# Check logs
docker-compose logs -f

# Restart services
docker-compose restart
```

**SSL certificate issues:**
```bash
# Check certificate status
sudo certbot certificates

# Renew manually
sudo certbot renew
```

**Database connection issues:**
```bash
# Access database container
docker-compose exec db psql -U openproject -d openproject
```

**Performance issues:**
- Increase VM resources (CPU/RAM)
- Monitor disk space: `df -h`
- Check Docker resource usage: `docker stats`

### Monitoring Commands

```bash
# System resources
htop
df -h
free -h

# Docker status
docker-compose ps
docker stats

# Nginx status
sudo systemctl status nginx
sudo nginx -t

# OpenProject logs
docker-compose logs -f web
```

## Maintenance Schedule

- **Daily**: Automated backups
- **Weekly**: Check system updates, review logs
- **Monthly**: Test backup restoration, review security
- **Quarterly**: Update OpenProject version, review SSL certificates

## Security Notes

1. **Change default passwords** immediately
2. **Enable 2FA** for admin accounts
3. **Regular updates** for both system and OpenProject
4. **Monitor logs** for suspicious activity
5. **Backup regularly** and test restoration
6. **Use strong passwords** and consider a password manager
7. **Limit access** to administrative interfaces

Your OpenProject instance should now be accessible at https://openproject.yourdomain.com with proper SSL encryption and Cloudflare protection!
