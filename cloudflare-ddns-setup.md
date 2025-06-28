Here’s a **comprehensive, step-by-step breakdown** of setting up **Cloudflare Dynamic DNS (DDNS)** so your home server’s IP stays updated, even if your ISP changes it:

---

## 🌍 Why You Need DDNS

Most home internet connections use **dynamic IP addresses**, meaning your public IP can change at any time (especially after router reboots or ISP maintenance). If you're routing domains/subdomains to your home, that’s a problem unless you keep your DNS records updated automatically.

**Cloudflare DDNS** solves this by detecting your changing IP and updating your Cloudflare DNS records.

---

## ✅ Step-by-Step Guide: Cloudflare DDNS Using `oznu/cloudflare-ddns`

---

### 🔧 Prerequisites

* You own a domain (e.g. `yourdomain.com`)
* Your domain is managed by Cloudflare
* You have a Cloudflare account
* Your Proxmox host (or a VM on it) has Docker installed (or you’re comfortable running Python/Bash scripts)

---

## 🛠️ Step 1: Add Domain to Cloudflare

1. Go to [https://dash.cloudflare.com](https://dash.cloudflare.com)
2. Create an account or sign in
3. Click **"Add a Site"**
4. Enter your domain (e.g. `yourdomain.com`)
5. Choose the **Free Plan**
6. Cloudflare will show DNS records to import. You can skip or clean them up later.
7. You’ll be given **nameservers** like:

   * `dana.ns.cloudflare.com`
   * `yichun.ns.cloudflare.com`
8. Log in to your **domain registrar (e.g., Namecheap, GoDaddy, etc.)**
9. Replace the existing nameservers with Cloudflare’s
10. Wait for propagation (usually < 1 hour)

---

## 🔁 Step 2: Create DNS Records for VMs

In the Cloudflare Dashboard:

1. Navigate to your domain → **DNS** tab
2. Click **“Add record”**

   * Type: `A`
   * Name: `app1`
   * IPv4 address: Use your current **public IP** (get it from [https://whatismyipaddress.com](https://whatismyipaddress.com))
   * TTL: `Auto`
   * Proxy status: **DNS only (gray cloud)** for now (can switch to proxy later)
3. Repeat for each VM (e.g., `app2.yourdomain.com`)

---

## 🔑 Step 3: Generate Cloudflare API Token

1. In Cloudflare Dashboard → Click your profile (top right) → **My Profile**
2. Go to **API Tokens** tab
3. Click **Create Token**
4. Choose **"Edit zone DNS"** template
5. Set:

   * Zone Resources: **Include** → Select specific domain → your domain
   * Permissions: Leave default (`Zone:DNS:Edit`)
6. Click **Continue** and **Copy the token** (save it in a secure place)

---

## ⚙️ Step 4: Set Up DDNS on Your Proxmox Host

We’ll use the `oznu/cloudflare-ddns` Docker container.

### 🔹 Option A: Using Docker

If Docker is already installed on your Proxmox host or on a lightweight VM/LXC:

#### 1. Create a folder for the config:

```bash
mkdir -p ~/cloudflare-ddns
cd ~/cloudflare-ddns
```

#### 2. Create a config file `.env`:

```bash
nano .env
```

Paste and modify:

```env
ZONE=yourdomain.com
PROXIED=false
API_KEY=your_cloudflare_api_token
EMAIL=your_cloudflare_account_email
RECORDS=app1.yourdomain.com,app2.yourdomain.com
```

> **Explanation**:
>
> * `ZONE`: Your base domain
> * `PROXIED`: Set to `true` if you want to use Cloudflare proxy (orange cloud)
> * `API_KEY`: Cloudflare API token you generated
> * `EMAIL`: The Cloudflare account email (required only for legacy API keys; optional if using a token)
> * `RECORDS`: Comma-separated list of subdomains to update

#### 3. Run the Docker container:

```bash
docker run -d \
  --name=cf-ddns \
  --env-file .env \
  --restart unless-stopped \
  oznu/cloudflare-ddns
```

#### 4. Confirm It’s Working

Check logs:

```bash
docker logs cf-ddns
```

You should see something like:

```
Updated: app1.yourdomain.com → x.x.x.x
Updated: app2.yourdomain.com → x.x.x.x
```

> The container rechecks every 5 minutes by default.

---

### 🔹 Option B: Using Python or Shell Script (No Docker)

If you don’t want Docker, you can use [Cloudflare’s API](https://developers.cloudflare.com/api/) directly in a shell script or Python script.

Let me know if you want this route — I’ll write a custom script for you.

---

## 📦 Optional: Auto-run on Boot (non-Docker)

If you don’t want Docker and use a shell script instead:

* Add it as a cron job:

  ```bash
  crontab -e
  ```

  Add:

  ```bash
  */5 * * * * /path/to/cloudflare-ddns.sh
  ```

---

## 🔐 Step 5: Switch to Cloudflare Proxy (Optional)

Once it’s working:

* Go to DNS tab in Cloudflare
* Click the gray cloud next to your subdomains to turn it **orange**
* This enables:

  * Cloudflare proxying (hides your real IP)
  * DDoS protection
  * HTTPS via Cloudflare

---

## ✅ Result

| Subdomain             | DNS Auto-Updated | Points to VM  |
| --------------------- | ---------------- | ------------- |
| `app1.yourdomain.com` | ✅ via DDNS       | 192.168.1.101 |
| `app2.yourdomain.com` | ✅ via DDNS       | 192.168.1.102 |

---

Would you like me to:

* Generate a full Python/Go/Bash-based Cloudflare DDNS updater (no Docker)?
* Show how to route these subdomains through NGINX Proxy Manager?
* Or automate this whole setup using Ansible?

Let me know how deep you want to go.
