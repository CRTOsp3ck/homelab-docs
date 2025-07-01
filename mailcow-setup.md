Setting up a **secure, self-hosted mail server** in your **Proxmox homelab** with **Cloudflare DNS** and **NGINX Proxy Manager** is a complex but rewarding task. Here's a **detailed step-by-step guide**, covering everything from VM creation to mail client access, with a focus on reliability and security.

---

## 🔧 OVERVIEW

You will be installing a mail server stack using **Mailcow** (a full-featured Dockerized mailserver suite). It includes:

* Postfix (SMTP)
* Dovecot (IMAP)
* Rspamd (Spam protection)
* SoGo (Webmail)
* ACME/Let’s Encrypt SSL
* DNS & DKIM/SPF/DMARC automation

---

## ✅ PREREQUISITES

### Hardware & Networking

* A **Proxmox VE** host with at least 2 vCPUs and 4–8 GB RAM for the VM
* **Static public IP address** (preferred) or reliable Dynamic DNS
* **Cloudflare DNS** for domain management
* **NGINX Proxy Manager** running (you already have this)
* Optional: IPv6 if your ISP supports it

### Domain

* You need a **domain name** (e.g. `yourdomain.com`)
* Ensure it's managed in Cloudflare

---

## 🧱 STEP 1: CREATE MAIL VM IN PROXMOX

### 1. Provision a VM

* OS: Ubuntu Server 22.04 LTS (Recommended)
* VM Settings:

  * CPU: 2 cores minimum
  * RAM: 4 GB minimum
  * Disk: 20 GB minimum (SSD recommended)
  * Network: bridged mode

### 2. Install the OS

During installation:

* Set hostname to something like `mail.yourdomain.com`
* Configure static IP or reserve it via DHCP lease on your router

---

## ⚙️ STEP 2: INSTALL MAILCOW

### 1. Update & Install Dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl docker.io docker-compose unzip
```

### 2. Clone Mailcow

```bash
git clone https://github.com/mailcow/mailcow-dockerized
cd mailcow-dockerized
```

### 3. Generate Config

```bash
./generate_config.sh
```

* Enter `mail.yourdomain.com` as the hostname

### 4. Configure Timezone (Optional)

Edit `mailcow.conf`:

```bash
vi mailcow.conf
```

Set your timezone, e.g. `TZ=Asia/Kuala_Lumpur`

### 5. Start Mailcow

```bash
docker compose pull
docker compose up -d
```

Mailcow will now be accessible at:
➡️ `https://mail.yourdomain.com` (once DNS & reverse proxy are configured)

---

## 🌐 STEP 3: CONFIGURE CLOUDFLARE DNS

Go to your domain’s DNS in Cloudflare and add the following:

| Type | Name             | Content                                                   | TTL  | Proxy      |
| ---- | ---------------- | --------------------------------------------------------- | ---- | ---------- |
| A    | mail             | Your public IP                                            | Auto | ❌ DNS only |
| MX   | @                | mail.yourdomain.com                                       | Auto | -          |
| TXT  | @                | `v=spf1 mx ~all`                                          | Auto | -          |
| TXT  | \_dmarc          | `v=DMARC1; p=quarantine; rua=mailto:dmarc@yourdomain.com` | Auto | -          |
| TXT  | mail.\_domainkey | <Mailcow will provide this>                               | Auto | -          |

**Important:**

* Keep the `mail.yourdomain.com` A record unproxied (DNS only), otherwise SMTP/IMAP will fail.
* Use Mailcow admin interface later to get the correct **DKIM TXT records**.

---

## 🚪 STEP 4: CONFIGURE PORT FORWARDING (ROUTER / FIREWALL)

Forward the following ports from your router to the Mail VM's **internal IP**:

| Port | Protocol | Purpose                 |
| ---- | -------- | ----------------------- |
| 25   | TCP      | SMTP (inbound mail)     |
| 465  | TCP      | SMTPs                   |
| 587  | TCP      | Submission              |
| 993  | TCP      | IMAPs                   |
| 4190 | TCP      | Sieve                   |
| 80   | TCP      | Let's Encrypt challenge |
| 443  | TCP      | Webmail & UI            |

If your ISP blocks **port 25**, you'll need a **SMTP relay service** (e.g., SendGrid, Mailgun, Mailjet).

---

## 🔒 STEP 5: NGINX PROXY MANAGER SETUP

### 1. Add a new proxy host:

* **Domain Names**: `mail.yourdomain.com`
* **Scheme**: `http`
* **Forward Hostname/IP**: internal IP of mail VM (e.g. `192.168.0.100`)
* **Forward Port**: `80`
* Enable **Websockets**
* Enable **SSL**, request Let’s Encrypt cert
* Enable **Force SSL**

> Mailcow listens on 80/443 internally, so NPM handles HTTPS termination externally.

---

## 🔐 STEP 6: MAIL SERVER SECURITY

Mailcow sets up:

* **DKIM keys**: Available under `Mailcow UI → Configuration → ARC/DKIM`
* **SPF, DMARC, DKIM**: Confirm using external tools like [https://mxtoolbox.com](https://mxtoolbox.com)
* **TLS**: Auto-managed by Mailcow + NGINX Proxy Manager
* **Fail2Ban**: Built into Mailcow
* **Antivirus/Antispam**: Rspamd + ClamAV are included

---

## 📬 STEP 7: CREATE USERS & TEST EMAIL

### 1. Access Admin Panel

* Go to: `https://mail.yourdomain.com`
* Default user: `admin` (you’ll set password during setup)

### 2. Create Mailboxes

e.g., `you@yourdomain.com`

### 3. Configure Mail Client

* **Incoming Mail (IMAP)**:

  * Server: `mail.yourdomain.com`
  * Port: 993
  * SSL/TLS: Yes
* **Outgoing Mail (SMTP)**:

  * Server: `mail.yourdomain.com`
  * Port: 587
  * STARTTLS: Yes
* **Username**: full email address
* **Password**: as set in Mailcow UI

Test sending and receiving using Thunderbird or Outlook.

---

## 🧪 STEP 8: VERIFY & MONITOR

### External Testing Tools:

* [https://mxtoolbox.com](https://mxtoolbox.com)
* [https://www.mail-tester.com](https://www.mail-tester.com)
* [https://dmarcian.com](https://dmarcian.com)

### Logs & Troubleshooting:

```bash
docker compose logs -f
```

Or view logs via Mailcow UI.

---

## 📌 OPTIONAL: BACKUPS

Set up daily backups:

```bash
cd /opt/mailcow-dockerized
./helper-scripts/backup_and_restore.sh backup all
```

Automate with cron or Proxmox scheduled tasks.

---

## 🧠 NOTES

* Hosting your own mail server is complex — deliverability is heavily tied to DNS, IP reputation, and anti-spam measures.
* Ensure your IP is **not blacklisted** (check with mxtoolbox).
* Avoid using residential dynamic IPs if possible.

---

Would you like me to:

* Help you pick a backup strategy for mail?
* Set up SMTP relay (if port 25 is blocked)?
* Create a monitoring dashboard for your Mail VM?

Let me know!
