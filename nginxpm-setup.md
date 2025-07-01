Here’s a complete **start-to-finish guide** to setting up **NGINX Proxy Manager (NPM)** in a **Proxmox LXC container**, ideal for a home lab environment. This setup is leaner than a full VM, saves on resources, and works well for reverse proxy + Let's Encrypt SSL with a web UI.

---

## ✅ OVERVIEW

You will:

1. Create an LXC container on Proxmox.
2. Configure it to support Docker (optional but recommended).
3. Install Docker and Docker Compose.
4. Deploy NGINX Proxy Manager using Docker Compose.
5. Expose the LXC's ports to your LAN/public network.
6. Optionally, set up DDNS and Cloudflare integration.

---

## 🧱 PREREQUISITES

* **Proxmox installed** and working.
* **Bridge network** set up (typically `vmbr0`) so LXC can get LAN IP.
* Root or sudo access in Proxmox.
* Port **80 and 443** open on your router/firewall and forwarded to your LXC IP if exposing to internet.

---

## ⚙️ STEP 1: Create the LXC Container

1. **Login to Proxmox Web UI**.
2. **Create CT (Container Template):**

   * Template: Use Debian 12 (preferred) or Ubuntu 22.04.
   * Hostname: `nginxpm`
   * Resources:

     * Disk: 6–10 GB minimum
     * RAM: 512MB+ (1GB ideal)
     * CPU: 1–2 cores
   * Network:

     * Use `vmbr0` or your bridged adapter.
     * DHCP or static IP (static preferred for routing).
   * Enable **unprivileged container** (recommended).
   * Under “Features”:

     * Enable `nesting` (required for Docker in LXC)
     * Enable `keyctl`, `fuse`, `cgroups`, and `mounts` (optional, helpful).

### 🛠 Enable required features manually (if missed)

SSH into your Proxmox host and run:

```bash
pct set <CTID> -features nesting=1,keyctl=1
```

---

## 📦 STEP 2: Start and Access the LXC

```bash
pct start <CTID>
pct console <CTID>
```

Update and install basic packages:

```bash
apt update && apt upgrade -y
apt install -y curl wget nano gnupg2 ca-certificates lsb-release apt-transport-https software-properties-common
```

---

## 🐳 STEP 3: Install Docker in the LXC

Install Docker via official script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```

Add your user to the Docker group (if you created a non-root user):

```bash
usermod -aG docker <your-username>
```

Install Docker Compose:

```bash
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

Check:

```bash
docker -v
docker-compose -v
```

---

## 📁 STEP 4: Deploy NGINX Proxy Manager

1. Create a folder:

```bash
mkdir -p /opt/nginxpm && cd /opt/nginxpm
```

2. Create `docker-compose.yml`:

```yaml
version: '3'
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginxpm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"     # Web UI
      - "443:443"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

3. Start the container:

```bash
docker-compose up -d
```

Check that it's running:

```bash
docker ps
```

---

## 🌐 STEP 5: Access NGINX Proxy Manager

* Visit: `http://<LXC-IP>:81`
* Default login:

  * **Email:** `admin@example.com`
  * **Password:** `changeme`

Change the credentials and start configuring your proxy hosts.

---

## 🔐 STEP 6: (Optional) Setup DNS / Cloudflare / DDNS

If you’re exposing to the internet:

* Use **Cloudflare DNS** or another provider.
* Point your domain/subdomain to your **public IP**.
* In your router:

  * Forward **ports 80 and 443** to your LXC IP.

Inside NPM:

* Add an SSL cert via **Let's Encrypt** (automatic).
* Enable **Force SSL**, **HTTP/2**, and HSTS if needed.

---

## 🛡️ Optional Hardening Tips

* Use UFW to limit inbound ports (only allow 80/443).
* Use Cloudflare Access rules (Geo IP filtering, WAF, etc.).
* Run NPM behind another reverse proxy if chaining security layers.

---

## 🔄 Useful Commands

* Restart NPM:

  ```bash
  docker-compose restart
  ```

* Update NPM:

  ```bash
  docker-compose down
  docker-compose pull
  docker-compose up -d
  ```

---

## 🧠 TROUBLESHOOTING

| Problem                        | Solution                                                         |
| ------------------------------ | ---------------------------------------------------------------- |
| LXC cannot run Docker          | Ensure `nesting=1`, `cgroups` are enabled                        |
| NPM port 80/443 not accessible | Check router port forwarding, LXC IP, and firewall               |
| Let’s Encrypt fails            | Domain not resolving to public IP, or port 80 blocked            |
| SSL cert doesn’t renew         | Set email + agree to terms in NPM settings, or use DNS challenge |

---

## 📌 Final Notes

* LXC containers are great for performance but be aware of Docker-in-LXC limitations. It’s supported, but not officially encouraged by all Docker projects.
* If something fails, running NPM on a VM (Debian/Ubuntu minimal) is always a fallback.

---

Would you like me to generate a `cloud-init` template or preconfigured LXC config file for faster setup next time?
