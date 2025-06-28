To SSH into your **Alpine Linux** VM instead, the process is very similar — but Alpine is super minimal by default, so there are a few extra steps you may need to take first.

---

### ✅ Step-by-Step: Enable SSH Access on Alpine Linux

#### 🥅 Goal

SSH from your main computer into the Alpine VM running on Proxmox.

---

### 🧰 1. **Install OpenSSH Server (if not installed)**

Log into the Alpine VM via the Proxmox console, then run:

```sh
apk update
apk add openssh
```

---

### 🔧 2. **Enable & Start SSH Service**

To start it immediately:

```sh
rc-service sshd start
```

To start it **automatically on boot**:

```sh
rc-update add sshd
```

---

### 🛡️ 3. **Allow Password Authentication (if needed)**

Edit the SSH config file:

```sh
vi /etc/ssh/sshd_config
```

Ensure this line is present and not commented:

```ini
PasswordAuthentication yes
```

Then restart SSH:

```sh
rc-service sshd restart
```

---

### 🌐 4. **Check Alpine's IP Address**

Run:

```sh
ip a
```

Look for an IP under `eth0`, `eth1`, or similar — e.g. `192.168.1.150`.

---

### 🔐 5. **SSH from Your Main Machine**

Now, from your main machine:

```sh
ssh youruser@192.168.x.x
```

---

### 📝 Note

Alpine doesn't create a user by default unless you do it manually. If you're using `root`, make sure root login is allowed:

```ini
PermitRootLogin yes
```

Otherwise, you can create a new user:

```sh
adduser alice
passwd alice
adduser alice wheel   # Optional: add to sudoers group
```

Give `sudo` permissions (Alpine uses `doas` or install `sudo`):

```sh
apk add sudo
```

Then:

```sh
visudo
```

Add:

```bash
alice ALL=(ALL) ALL
```

---

Let me know if:

* You’re getting an error
* You want to use SSH keys instead of passwords
* You want to expose it outside your LAN via port forwarding or reverse proxy
