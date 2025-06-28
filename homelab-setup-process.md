Let’s create a complete, **step-by-step guide** to transform your spare desktop into a **home datacenter**, including:

1. Formatting the computer
2. Installing the hypervisor
3. Creating and managing VMs
4. Setting up a domain with subdomains
5. Routing traffic with a reverse proxy
6. Securing access from the internet

We’ll be using **Proxmox VE** as the hypervisor (it’s free, robust, has an easy-to-use web UI, and supports nested virtualization). Let’s go!

---

## 🧱 PART 1: Format Your Desktop & Install Proxmox VE

### 🖥️ Requirements

* Spare desktop (64-bit CPU, ideally 16GB+ RAM, SSD preferred)
* USB flash drive (8GB+)
* A second computer to access the Proxmox web interface
* Stable internet connection
* Access to your router settings

---

### ✅ Step 1. Download Proxmox VE ISO

1. Go to [https://www.proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
2. Download the latest **Proxmox VE ISO Installer** (e.g., `proxmox-ve_8.x.iso`)

---

### ✅ Step 2. Flash ISO to USB

Use [Balena Etcher](https://etcher.io/) or [Rufus](https://rufus.ie/en/) on another computer:

* Select the Proxmox ISO
* Select your USB drive
* Flash the image

---

### ✅ Step 3. Boot Your Desktop from USB

1. Plug in the USB
2. Boot your desktop and enter BIOS/UEFI (`F2`, `DEL`, `F10`, etc.)
3. Set USB drive as first boot option
4. Save & reboot

---

### ✅ Step 4. Install Proxmox VE

Follow the on-screen installer:

* Accept license agreement
* Choose a drive to format (⚠️ this will wipe everything)
* Set:

  * Country, Time zone, Keyboard
  * Admin password
  * Email (for alerts)
* Assign a **hostname** (e.g., `proxmox.local`)
* Network setup:

  * Choose your main network interface
  * Assign a **static IP** (e.g., `192.168.1.100`)
  * Gateway = your router’s IP (e.g., `192.168.1.1`)
  * DNS = `1.1.1.1` or `8.8.8.8`

After install, remove USB and reboot.

---

## 🕹️ PART 2: Access Proxmox and Create VMs

### ✅ Step 5. Access Proxmox Web UI

From another computer on the same network:

```
https://192.168.1.100:8006
```

* Login as `root` using the password you set during install.

---

### ✅ Step 6. Upload Linux ISOs to Proxmox

1. Go to **Datacenter → local → ISO Images**
2. Click **Upload**
3. Upload OS ISOs (Ubuntu Server, Alpine, Debian, etc.)

---

### ✅ Step 7. Create Your First VM

1. Click **Create VM**
2. VM ID = auto-assigned
3. Name = `app1` (or your project name)
4. OS = choose ISO (Ubuntu Server, etc.)
5. System = default (UEFI or BIOS)
6. Disk = 16–32GB or more (SSD-backed if possible)
7. CPU = 2–4 cores
8. RAM = 2–4GB
9. Network = **Bridged Mode (vmbr0)** ← critical!

Finish the wizard and start the VM. Use **Console** to install the OS normally.

Repeat this process for multiple VMs (`app2`, `dev`, etc.).

---

## 🌍 PART 3: External Access + Domain Setup

### ✅ Step 8. Set Static IPs for Each VM

Inside each VM:

* Configure static IP or reserve IP in your router’s DHCP settings
* Example VM IPs:

  * `app1`: `192.168.1.101`
  * `app2`: `192.168.1.102`

---

### ✅ Step 9. Get a Domain Name

Buy a domain from:

* [Cloudflare](https://dash.cloudflare.com/)
* [Namecheap](https://namecheap.com)

Example: `yourdomain.com`

---

### ✅ Step 10. Use Dynamic DNS (DDNS)

If your ISP gives you a dynamic IP (check at [https://whatismyipaddress.com/](https://whatismyipaddress.com/)):

#### Option A: Cloudflare DDNS

1. Create a Cloudflare account

2. Add your domain to Cloudflare

3. Point nameservers to Cloudflare

4. Create DNS records:

   * `A` record for `app1.yourdomain.com` → your public IP
   * `A` record for `app2.yourdomain.com` → your public IP

5. Install a Cloudflare DDNS script on your Proxmox host:

   * [https://github.com/oznu/cloudflare-ddns](https://github.com/oznu/cloudflare-ddns)

> This keeps your IP updated automatically.

---

## 🔁 PART 4: Reverse Proxy + HTTPS

### ✅ Step 11. Create a Reverse Proxy VM (or container)

Install **NGINX Proxy Manager** (NPM) on a dedicated lightweight VM or LXC container.

**NPM Features:**

* Simple web UI
* Automatic SSL (Let’s Encrypt)
* Subdomain routing

#### Use Docker to install NPM:

```bash
docker volume create npm_data

docker run -d \
  --name=nginx-proxy-manager \
  -p 80:80 -p 81:81 -p 443:443 \
  -v npm_data:/data \
  -v /etc/localtime:/etc/localtime:ro \
  jc21/nginx-proxy-manager:latest
```

> Web UI: `http://<vm-ip>:81` → default login: `admin@example.com / changeme`

---

### ✅ Step 12. Route Subdomains to VMs

In **NGINX Proxy Manager**:

* Add **Proxy Host**

  * Domain: `app1.yourdomain.com`
  * Forward Hostname/IP: `192.168.1.101`
  * Forward Port: `80` or `443`
  * Enable **Websockets**, **Block Common Exploits**
  * Enable **SSL** + Auto Let’s Encrypt

Repeat for other subdomains like `app2.yourdomain.com`.

---

## 🔒 PART 5: Security

### ✅ Step 13. Secure Your Setup

* Use strong passwords or SSH keys on all VMs
* Enable **UFW** or `firewalld` on each VM
* Use **fail2ban** to block brute force
* Keep Proxmox and VMs updated
* Avoid exposing unnecessary ports (only expose via reverse proxy)
* Use **Cloudflare proxy (orange cloud)** to mask your IP and enable rate limiting

---

## 📦 Bonus: Optional Tools

| Tool        | Purpose                     |
| ----------- | --------------------------- |
| Portainer   | Docker container management |
| Netdata     | Real-time monitoring        |
| Gitea       | Lightweight Git server      |
| Nextcloud   | Personal cloud storage      |
| Uptime Kuma | Self-hosted uptime monitor  |

---

## ✅ Final Result

| Subdomain             | Points To       | Hosted App |
| --------------------- | --------------- | ---------- |
| `app1.yourdomain.com` | `192.168.1.101` | Project A  |
| `app2.yourdomain.com` | `192.168.1.102` | Project B  |

Everything routes securely via:

```
[Internet] → [Public IP] → [Router] → [NGINX Proxy Manager] → [VMs]
```
