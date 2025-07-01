Here's the updated, **Alpine/OpenRC-compatible full guide** for setting up MinIO on your Proxmox homelab — copy-paste ready:

---

## 🧾 Overview

You will:

* Create a dedicated **Alpine LXC** or **Debian VM** for MinIO
* Install and configure MinIO in standalone mode
* Set it to auto-start on boot using **OpenRC** (not `systemd`)
* Secure it with HTTPS via **NGINX Proxy Manager** and **Let's Encrypt**
* Expose it via a custom subdomain using **Cloudflare** DNS
* Optionally configure **MinIO Client (`mc`)** and storage buckets

---

## ⚙️ Step-by-Step Setup

---

### 1️⃣ Proxmox: Create a VM or LXC for MinIO

#### ✅ Option A: Use an **Alpine LXC** (recommended for your setup)

1. **Download Alpine LXC Template** via Proxmox:

   * Go to: `Node → CT Templates → Templates`
   * Download: `alpine-3.19-default_20240418_amd64.tar.xz`

2. **Create LXC Container**:

   * Basic:

     * Hostname: `minio`
     * Password: set strong root password
   * Template: choose the downloaded Alpine template
   * Disk: 10-20 GB (depending on file size expectations)
   * CPU/RAM: 1 core, 1-2 GB RAM minimum
   * Network: use `vmbr0` or `vmbr1` (bridged) with DHCP/static IP

3. **Enable nesting and key features**:

   ```bash
   pct set <CTID> -features nesting=1,keyctl=1
   ```

#### ✅ Option B: Debian VM (for `systemd` users)

* Same process, but choose Debian ISO.
* Assign 2 cores, 2 GB RAM, 20 GB+ storage.

---

### 2️⃣ Install MinIO Server

#### 🔐 Login via SSH

```bash
ssh root@<minio_lxc_ip>
```

#### 🧰 Install Dependencies

For Alpine:

```sh
apk update && apk add curl bash
```

For Debian:

```sh
apt update && apt install -y curl wget
```

#### 📦 Download MinIO Server Binary

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
mv minio /usr/local/bin/
```

#### 📁 Create MinIO User and Directories

```bash
adduser -D -g "MinIO User" minio
mkdir -p /data/minio
chown -R minio:minio /data/minio
```

#### 🧪 Test Running It

```bash
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=verystrongpassword
minio server /data/minio --console-address ":9001"
```

* Access: `http://<IP>:9000` (S3) and `http://<IP>:9001` (Admin console)

---

### 3️⃣ OpenRC Service for MinIO (Alpine-specific)

#### 📁 Create Config File

```bash
nano /etc/conf.d/minio
```

Add:

```bash
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="verystrongpassword"
```

#### 🧾 Create Init Script

```bash
nano /etc/init.d/minio
```

Paste:

```bash
#!/sbin/openrc-run

. /etc/conf.d/minio

name="MinIO"
description="S3-compatible object storage server"

command="/usr/local/bin/minio"
command_args="server /data/minio --console-address :9001"
command_user="minio:minio"

pidfile="/run/${RC_SVCNAME}.pid"

depend() {
    need net
    use dns logger
}
```

#### ✅ Make Executable and Enable

```bash
chmod +x /etc/init.d/minio
rc-update add minio default
rc-service minio start
```

---

### 4️⃣ DNS + SSL via NGINX Proxy Manager

#### 🌐 DNS Setup via Cloudflare

* Create an A record: `minio.example.com` → your **public IP**
* Enable Cloudflare proxy (orange cloud)

#### 🔁 NGINX Proxy Manager (running in your homelab)

* Go to UI: `http://<nginxpm_ip>:81`
* Login
* Add a new **Proxy Host**:

  * Domain Names: `minio.example.com`
  * Scheme: `http`
  * Forward Hostname/IP: `<MinIO_LXC_IP>`
  * Forward Port: `9001`
  * Enable Websockets
  * SSL:

    * Request a new SSL cert (Let's Encrypt)
    * Force SSL, HTTP/2

Test it:

* Open: `https://minio.example.com`
* You should see the MinIO Console

---

### 5️⃣ Optional: Use MinIO Client (`mc`)

Install on any client (e.g. local PC):

```bash
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```

Add your server:

```bash
mc alias set myminio https://minio.example.com minioadmin verystrongpassword
```

Create a bucket:

```bash
mc mb myminio/mybucket
```

Upload a file:

```bash
mc cp file.txt myminio/mybucket
```

List buckets:

```bash
mc ls myminio
```

---

## 🛡️ Optional: Harden the Setup

* ❗ Change default credentials (`minioadmin`) after first login
* Set resource limits on LXC (RAM, CPU)
* Enable firewall rules via UFW or Proxmox-level
* Configure TLS passthrough (optional advanced config)
* Setup automatic backups of `/data/minio`

---

## 📦 Optional: S3-Compatible Use in Apps

* Use your MinIO endpoint like any S3:

```env
S3_ENDPOINT=https://minio.example.com
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=verystrongpassword
S3_BUCKET=mybucket
```

Apps like:

* Jenkins (with S3 plugin)
* Nextcloud (for external storage)
* Backups
* Static file hosting

---

## 🧹 Maintenance Tips

* Monitor disk usage (`df -h`)
* Rotate logs if needed
* Back up `/data/minio` regularly
* Keep `minio` binary updated:

```bash
mc admin update myminio
```

---

## ✅ Summary

| Task                         | Status     |
| ---------------------------- | ---------- |
| Proxmox LXC or VM            | ✅ Done     |
| MinIO Installed & Running    | ✅ Done     |
| OpenRC Service for Autostart | ✅ Done     |
| Reverse Proxy via NGINX PM   | ✅ Done     |
| SSL via Let's Encrypt        | ✅ Done     |
| Cloudflare DNS Routing       | ✅ Done     |
| Optional MinIO Client Setup  | ✅ Optional |

---

Would you like to:

* Set up **MinIO in distributed mode**?
* Connect **Jenkins/CI** to upload artifacts?
* Mount MinIO as a **filesystem** (via `s3fs`)?

Let me know and I’ll extend the guide.

This error:

```
Error: unable to rename (/data/minio/.minio.sys/tmp -> /data/minio/.minio.sys/tmp-old/...)
Error: file access denied, drive may be faulty
```

means that **MinIO doesn’t have the correct ownership or permissions** to write into `/data/minio`.

Even if you `chown -R` previously, it's possible:

* the init script is running as `minio:minio`, but the directory/files are not owned by that user/group.
* the permissions on `/data/minio` or its parent (`/data`) prevent access.

---

## ✅ Fix Step-by-Step

### 1️⃣ Ensure `minio` user owns the full path

```sh
chown -R minio:minio /data
chmod -R u+rwX /data
```

If you want to be extra safe:

```sh
ls -ld /data /data/minio
ls -l /data/minio
```

All should show:

```
drwxr-xr-x  minio minio ...
```

---

### 2️⃣ Double-check `/data` mount (if any)

If `/data` is a bind-mount from the host, and the **host directory is owned by `root`**, the container may not have write access. You can:

* Use `pct set <CTID> -mp0 /host/path,mp=/data,min=rw`
* Or ensure that inside the container, `/data` is truly writable by `minio`

---

### 3️⃣ Retry Manually First

Try this as the `minio` user:

```sh
sudo -u minio touch /data/minio/testfile
```

If it fails, permissions are still incorrect.

---

### 4️⃣ Cleanup any partial temp files

Sometimes corrupted `.minio.sys/tmp` contents can cause issues:

```sh
rm -rf /data/minio/.minio.sys/tmp
```

**Only do this if you’re starting fresh and not storing real data yet.**

---

### 5️⃣ Restart the service

```sh
rc-service minio restart
```

Then check logs:

```sh
tail -f /var/log/minio.log
```

(or whatever file you send logs to)

---

## 🔄 Summary Fix Commands

```sh
chown -R minio:minio /data
chmod -R u+rwX /data
rm -rf /data/minio/.minio.sys/tmp
rc-service minio restart
```

---

Let me know if this is a bind mount or if you're doing distributed setup — I can tailor the fix accordingly.
