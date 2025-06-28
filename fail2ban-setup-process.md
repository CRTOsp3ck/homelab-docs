## ✅ Set Up Fail2Ban on the NPM Host (Debian/Ubuntu)

Fail2Ban monitors log files for brute-force attempts and blocks IPs via `iptables` or `ufw`.

### 🔧 A. Install Fail2Ban

```bash
sudo apt update
sudo apt install fail2ban -y
```

### 🔧 B. Configure Fail2Ban for NGINX

NPM logs to `./data/logs/`, so we’ll monitor the **access** and **error** logs.

#### 1. Create a custom jail file

```bash
sudo nano /etc/fail2ban/jail.d/npm.conf
```

#### 2. Paste the following:

```ini
[nginx-http-auth]
enabled = true
port    = http,https
filter  = nginx-http-auth
logpath = /path/to/your/npm/data/logs/*access.log
maxretry = 5
bantime  = 3600

[nginx-badbots]
enabled  = true
port     = http,https
filter   = nginx-badbots
logpath  = /path/to/your/npm/data/logs/*access.log
maxretry = 2
bantime  = 86400

[nginx-dos]
enabled = true
port    = http,https
filter  = nginx-dos
logpath = /path/to/your/npm/data/logs/*access.log
maxretry = 20
findtime = 60
bantime = 600
```

> Replace `/path/to/your/npm/data/logs/*access.log` with the full path to your NPM data volume on the host machine. For example, if you followed earlier steps:

```bash
/home/youruser/nginx-proxy-manager/data/logs/*access.log
```

#### 3. Restart Fail2Ban

```bash
sudo systemctl restart fail2ban
```

#### 4. Check active jails

```bash
sudo fail2ban-client status
sudo fail2ban-client status nginx-http-auth
```

---

## 🔐 Optional: Monitor Fail2Ban Logs

To check banned IPs:

```bash
sudo fail2ban-client status nginx-http-auth
```

To unban an IP:

```bash
sudo fail2ban-client set nginx-http-auth unbanip <IP_ADDRESS>
```

---

## ✅ Recap Summary

| Feature             | Method                               |
| ------------------- | ------------------------------------ |
| Restrict by IP      | Use Access Lists in NPM UI           |
| Restrict by Country | Requires GeoIP & NGINX customization |
| Block brute-force   | Install and configure Fail2Ban       |

---

Would you like to go deeper into **GeoIP country blocking**, or integrate **Cloudflare Firewall Rules** to block countries at DNS-level (before it hits your server)?

You're getting that error because **Fail2Ban expects filters (like `nginx-badbots` and `nginx-dos`) to exist in `/etc/fail2ban/filter.d/`**, but these are **not provided by default**.

Let's fix this step-by-step.

---

## ✅ Step 1: Fix Missing Filters

### 1.1: Create the missing `nginx-badbots` filter

```bash
sudo nano /etc/fail2ban/filter.d/nginx-badbots.conf
```

Paste this:

```ini
[Definition]
failregex = ^<HOST> -.*"(GET|POST).*(?:acunetix|sqlmap|nikto|fimap|nmap|nessus|whatweb|zmeu|jaascois|morfeus|arachni|netsparker|libwww-perl).*"$
ignoreregex =
```

Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### 1.2: Create the missing `nginx-dos` filter

```bash
sudo nano /etc/fail2ban/filter.d/nginx-dos.conf
```

Paste this:

```ini
[Definition]
failregex = ^<HOST> -.*"(GET|POST).*
ignoreregex =
```

> ⚠️ Note: This is a **very broad pattern** for basic DoS — it bans on lots of repeated requests in a short period (controlled by `maxretry` + `findtime`). It’s useful for early brute-force protection but may need tuning later.

Save and exit.

---

## ✅ Step 2: Fix the `sshd` Jail Error (Optional)

Fail2Ban tries to enable `sshd` jail by default, but your system might not be running SSH or its logs aren't present.

**Option A: Disable the sshd jail globally**

```bash
sudo nano /etc/fail2ban/jail.local
```

Add this:

```ini
[sshd]
enabled = false
```

Save and exit.

---

## ✅ Step 3: Verify and Restart

### Reload Fail2Ban:

```bash
sudo systemctl daemon-reexec
sudo systemctl restart fail2ban
```

### Check status:

```bash
sudo fail2ban-client status
```

You should see something like:

```
Status
|- Number of jail: 3
`- Jail list: nginx-http-auth, nginx-badbots, nginx-dos
```

### Check individual jail:

```bash
sudo fail2ban-client status nginx-badbots
```

---

## ✅ Bonus: Verify Log Path

Make sure `/root/nginx-proxy-manager/data/logs/*access.log` is correct. If you're running NPM via Docker, your real logs might be mounted elsewhere.

To confirm:

```bash
ls /root/nginx-proxy-manager/data/logs/
```

You should see something like:

```
default.log  proxy-host-1_access.log  proxy-host-2_access.log ...
```

If it’s not working, switch `logpath` to:

```ini
logpath = /root/nginx-proxy-manager/data/logs/*.log
```

And restart Fail2Ban again.

---

Would you like to add **email alerts** when IPs get banned, or integrate this with a **dashboard or Cloudflare ban automation**?
