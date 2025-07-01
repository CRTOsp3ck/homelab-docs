Here’s a complete **in-depth guide** to set up a **Jenkins CI/CD server** in your **Proxmox homelab**, using either an **LXC container** (for efficiency) or a **VM** (for maximum compatibility). This guide assumes you have Proxmox up and running, and basic networking (via your router and NGINX Proxy Manager) is in place.

---

## ✅ OVERVIEW

We’ll be setting up:

* ✅ Jenkins on a clean Ubuntu/Debian server (LXC or VM)
* ✅ Web access to Jenkins via reverse proxy (NGINX Proxy Manager + Cloudflare)
* ✅ HTTPS using Let's Encrypt
* ✅ Persistent storage for Jenkins jobs & configs
* ✅ Docker engine (optional) so Jenkins can run containerized builds

---

## 🧰 PREREQUISITES

* Proxmox VE up and running.
* SSH access to your Proxmox host.
* Cloudflare + NGINX Proxy Manager (NPM) reverse proxy setup (optional but recommended).
* A valid domain/subdomain (e.g., `jenkins.yourdomain.com`).
* DNS A/AAAA records pointing to your public IP (or DDNS setup).
* A dedicated storage pool (optional) for Jenkins LXC volume persistence.

---

## 🧱 STEP 1: CREATE A JENKINS LXC CONTAINER

> If you prefer using a VM, skip to the VM section after this.

### 1.1 Download Ubuntu/Debian Template

```bash
pveam update
pveam available | grep ubuntu
pveam download local ubuntu-22.04-standard_*.tar.zst
```

### 1.2 Create LXC Container via Proxmox Web UI

* **Node** > **Create CT**
* **CT ID**: `900` (or any number)
* **Hostname**: `jenkins`
* **Password**: (Set root password)
* **Template**: `ubuntu-22.04-standard`
* **Disk Size**: 16-32GB (more if builds are heavy)
* **CPU**: 2 cores
* **Memory**: 4096 MB
* **Network**:

  * Assign static DHCP or manual IP
* **Features**:

  * ✅ Nesting (important for Docker-in-LXC)
  * ❌ Unprivileged (optional, Jenkins works better with privileged containers)

After creation:

```bash
pct start 900
pct console 900
```

---

## 🖥️ STEP 2: INITIAL SERVER SETUP (INSIDE CONTAINER/VM)

```bash
# Update
apt update && apt upgrade -y

# Install basics
apt install -y curl gnupg2 ca-certificates software-properties-common apt-transport-https unzip

# Optional: Set hostname
hostnamectl set-hostname jenkins
```

---

## ⚙️ STEP 3: INSTALL JAVA

Jenkins requires Java 11 or 17.

```bash
apt install -y openjdk-17-jdk
java -version
```

---

## 📦 STEP 4: INSTALL JENKINS

### 4.1 Add Jenkins Repo

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
```

### 4.2 Install Jenkins

```bash
apt update
apt install -y jenkins
```

---

## 🔥 STEP 5: START AND ENABLE JENKINS

```bash
systemctl enable jenkins
systemctl start jenkins
systemctl status jenkins
```

### Jenkins Default Port

By default, Jenkins runs on `http://your-lxc-ip:8080`.

---

## 🐳 OPTIONAL: INSTALL DOCKER FOR CONTAINERIZED BUILDS

```bash
curl -fsSL https://get.docker.com | bash
usermod -aG docker jenkins
systemctl restart jenkins
```

---

## 🔐 STEP 6: SET UP REVERSE PROXY (NGINX PROXY MANAGER)

Assuming you're using NGINX Proxy Manager in another container:

### 6.1 DNS Setup

* In Cloudflare, create an `A` record:

  * `jenkins.yourdomain.com → your public IP`
  * Enable proxy (orange cloud) if using Cloudflare proxy.

### 6.2 Add Proxy Host in NGINX PM

* **Domain Name**: `jenkins.yourdomain.com`
* **Scheme**: `http`
* **Forward Host**: `internal-ip-of-jenkins`
* **Port**: `8080`
* **Websockets**: ✅ enabled
* **SSL**: Request Let's Encrypt certificate
* **Block Common Exploits**: ✅
* **HTTP/2 Support**: ✅

---

## 🔑 STEP 7: INITIAL JENKINS SETUP

Access via browser:

```
https://jenkins.yourdomain.com
```

Unlock it:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

* Paste it into the web interface
* Install suggested plugins
* Create first admin user

---

## 🗄️ STEP 8: PERSISTENT STORAGE (IMPORTANT FOR LXC)

> Ensure your Jenkins job/workspace config persists across reboots.

* Jenkins data is in `/var/lib/jenkins`
* If needed, mount this path to an external volume on creation or post-creation using Proxmox’s bind mount.

Example:

```bash
# On Proxmox Host
mkdir -p /mnt/pve/jenkins-data
pct set 900 -mp0 /mnt/pve/jenkins-data,mp=/var/lib/jenkins
```

---

## 🛡️ STEP 9: HARDENING & MAINTENANCE

* Setup automated backups (snapshot or via Jenkins backup plugin)
* Enable system firewall or UFW:

  ```bash
  apt install ufw
  ufw allow 8080
  ufw allow ssh
  ufw enable
  ```
* Install `fail2ban` (optional):

  ```bash
  apt install fail2ban
  ```

---

## 🖥️ VM SETUP ALTERNATIVE

If Alpine/Debian LXC is giving issues (esp. with Docker):

1. Download Ubuntu 22.04 ISO
2. Create a VM (e.g., 2 vCPU, 4GB RAM, 32GB disk)
3. Boot ISO, install Ubuntu
4. Follow steps **2 → 9** above inside the VM

---

## 🧪 STEP 10: CREATE A SIMPLE BUILD PIPELINE

1. Go to **New Item** → Freestyle Project
2. Set GitHub Repo → `https://github.com/user/project.git`
3. Add build steps (e.g., shell script):

   ```bash
   echo "Building..."
   ./build.sh
   ```
4. Save and trigger build manually or via webhook

---

## ✅ OPTIONAL: ADD GITHUB/DOCKER INTEGRATIONS

* Add **GitHub Webhooks** under project settings
* Use **Docker Plugin** to build and push images to your private registry

---

## 🎯 RECOMMENDATIONS

| Feature                 | Recommendation                                                 |
| ----------------------- | -------------------------------------------------------------- |
| Persistent Jenkins data | Use external mounted volume                                    |
| Network access          | Use static DHCP reservation                                    |
| HTTPS                   | Use NGINX Proxy Manager with Let's Encrypt                     |
| Reverse Proxy Auth      | Protect Jenkins with Cloudflare Zero Trust or NPM Access Rules |
| Backup                  | Proxmox snapshot + Jenkins job backup plugin                   |
| Updates                 | Regularly update Jenkins & system packages                     |
| Secrets Management      | Use Jenkins credentials store                                  |

---

Would you like a **Docker-based Jenkins deployment**, or **Jenkins with Kubernetes agent integration** next?
