Here’s a full **start-to-finish guide** to set up a **MinIO server** in your **Proxmox homelab** using a **VM**, including storage configuration, secure access, and optional S3 browser UI.

---

## ✅ Overview

* **Type:** VM-based installation (for isolation and flexibility)
* **MinIO Type:** Standalone (good for homelabs; distributed can be added later)
* **Use Case:** Self-hosted S3-compatible object storage
* **Target Audience:** You, with working Proxmox, Cloudflare/NginxPM routing

---

## 🧱 Prerequisites

| Component           | Requirement                               |
| ------------------- | ----------------------------------------- |
| Proxmox             | Installed and running                     |
| ISO Image           | Ubuntu Server 22.04 or Debian 12          |
| Cloudflare          | (Optional) for domain + SSL               |
| NGINX Proxy Manager | (Optional) for HTTPS reverse proxy        |
| VM Storage          | At least 10–50GB for MinIO bucket storage |

---

## 🖥️ 1. Create the VM in Proxmox

1. **Upload ISO:**

   * Go to Proxmox Web UI → `Datacenter > Storage > ISO Images`
   * Upload `ubuntu-22.04-live-server-amd64.iso` or `debian-12.iso`

2. **Create VM:**

   * Click **Create VM**

     * **Name:** `minio-server`
     * **OS:** Use uploaded ISO
     * **System:**

       * BIOS: `OVMF (UEFI)` (optional for newer features)
       * Machine: Default
     * **Disks:**

       * Add **2 disks**:

         1. **Primary Disk** for OS (20GB)
         2. **Storage Disk** for MinIO data (e.g., 100GB)
       * Use `SCSI` with `VirtIO SCSI` controller
     * **CPU:** 2 cores minimum
     * **RAM:** 2GB minimum
     * **Network:**

       * Use `virtio` NIC, attach to your LAN bridge (e.g., `vmbr0`)
   * **Start after creation**

---

## 💿 2. Install OS

1. Boot the VM and install Ubuntu/Debian normally.
2. **Set static IP** during install or later via:

   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```

   Example:

   ```yaml
   network:
     version: 2
     ethernets:
       ens18:
         dhcp4: no
         addresses: [192.168.1.100/24]
         gateway4: 192.168.1.1
         nameservers:
           addresses: [1.1.1.1, 8.8.8.8]
   ```

   Then:

   ```bash
   sudo netplan apply
   ```

---

## 📦 3. Install MinIO (Binary method)

1. SSH into the VM:

   ```bash
   ssh youruser@192.168.1.100
   ```

2. Create a dedicated user:

   ```bash
   sudo useradd -r minio-user -s /sbin/nologin
   ```

3. Install MinIO:

   ```bash
   wget https://dl.min.io/server/minio/release/linux-amd64/minio
   chmod +x minio
   sudo mv minio /usr/local/bin/
   ```

4. Create directories:

   ```bash
   sudo mkdir -p /data/minio
   sudo chown -R minio-user:minio-user /data/minio
   ```

---

## 🧩 4. Configure MinIO as a Systemd Service

```bash
sudo nano /etc/systemd/system/minio.service
```

Paste:

```ini
[Unit]
Description=MinIO Object Storage
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target

[Service]
User=minio-user
Group=minio-user
ExecStart=/usr/local/bin/minio server /data/minio --console-address ":9001"
Restart=always
LimitNOFILE=65536
Environment="MINIO_ROOT_USER=minioadmin"
Environment="MINIO_ROOT_PASSWORD=minioadminpass"

[Install]
WantedBy=multi-user.target
```

> 🔐 Replace `minioadmin` and `minioadminpass` with strong credentials.

Then:

```bash
sudo systemctl daemon-reexec
sudo systemctl enable minio
sudo systemctl start minio
```

---

## 🔒 5. Optional: Set Up NGINX Proxy Manager for HTTPS

1. In your NGINX Proxy Manager (NPM):

   * **Add Proxy Host**

     * Domain: `minio.yourdomain.com`
     * Forward Hostname/IP: `192.168.1.100`
     * Port: `9001`
     * Enable `Websockets`
     * Enable SSL → Request Let's Encrypt cert

2. In Cloudflare:

   * Point A/CNAME record for `minio.yourdomain.com` to your public IP
   * Ensure **orange cloud (proxy)** is **enabled**

3. (Optional) Add firewall rules or VPN access for added security.

---

## 📂 6. Access and Use MinIO

* Open in browser:

  ```
  https://minio.yourdomain.com
  ```
* Login with your MinIO credentials
* You’ll land in the **MinIO Console** where you can:

  * Create buckets
  * Upload/download files
  * Generate pre-signed URLs
  * Manage access keys

---

## 📦 7. MinIO Client (`mc`) CLI Tool (Optional)

1. Install `mc`:

   ```bash
   curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
   chmod +x mc
   sudo mv mc /usr/local/bin/
   ```

2. Configure:

   ```bash
   mc alias set minio https://minio.yourdomain.com minioadmin minioadminpass
   mc ls minio
   ```

---

## 🛡️ 8. (Optional) Harden MinIO

* Use a non-default port if exposing directly.
* Use Fail2Ban or UFW to limit brute force.
* Use a Cloudflare Access rule for Zero Trust login.
* Rotate access keys regularly.

---

## 🧪 9. Test MinIO with an S3 SDK

You can test MinIO using any S3-compatible SDK:

```python
import boto3

s3 = boto3.client('s3',
    endpoint_url='https://minio.yourdomain.com',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadminpass'
)

response = s3.list_buckets()
print(response)
```

---

## 📋 Final Checklist

| Task                                     | Status |
| ---------------------------------------- | ------ |
| VM created with extra storage disk       | ✅      |
| OS installed + static IP configured      | ✅      |
| MinIO installed and running as service   | ✅      |
| NGINXPM reverse proxy configured         | ✅      |
| HTTPS SSL via Cloudflare & Let's Encrypt | ✅      |
| Accessed MinIO Console                   | ✅      |
| Set up buckets or tested S3 client       | ✅      |

---

## 🚀 Optional Extras

* 🎛️ **MinIO in Distributed Mode** (across multiple VMs/Disks)
* 🛡️ **Object Locking and WORM**
* 📈 **Prometheus + Grafana for metrics**
* 📦 **Backup integration (Velero, Restic, etc.)**

---

Would you like me to generate a **bash script** or **Ansible playbook** to automate this entire setup?
