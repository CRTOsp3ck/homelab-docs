If you're getting **"Permission denied"** despite using the correct password, here’s a step-by-step checklist to diagnose and fix the issue:

---

### ✅ Step-by-Step Troubleshooting

#### 🔍 1. **Confirm Username is Correct**

Make sure you're using the **exact** username defined in the Debian VM.

Try checking in the VM terminal (via Proxmox web console):

```bash
whoami
```

Or list all users:

```bash
cut -d: -f1 /etc/passwd
```

If you created a user like `admin`, but you're typing `root`, it won't work unless SSH is configured to allow root logins (which is disabled by default).

---

#### 🔐 2. **Check SSH Server Configuration**

In your VM, open the SSH config file:

```bash
sudo nano /etc/ssh/sshd_config
```

Look for the following settings:

```ini
PermitRootLogin yes     # if you want to allow root login (not recommended)
PasswordAuthentication yes
```

✅ Make sure `PasswordAuthentication` is set to `yes`. If it was `no`, change it and then:

```bash
sudo systemctl restart ssh
```

---

#### 🚫 3. **Check for SSH Key Requirement**

If your `sshd_config` has this:

```ini
PasswordAuthentication no
```

...then it’s **forcing SSH key authentication**, and password logins will always fail.

To re-enable passwords:

1. Set `PasswordAuthentication yes`
2. Restart the SSH service:

```bash
sudo systemctl restart ssh
```

---

#### 🔒 4. **Check for Fail2Ban or UFW**

* If you've enabled **fail2ban**, you might be temporarily blocked due to failed attempts.
* If **UFW** or another firewall is active, make sure port 22 is open:

```bash
sudo ufw allow ssh
sudo ufw status
```

---

#### 📡 5. **Try Verbose SSH for More Info**

On your client machine:

```bash
ssh -vvv youruser@your_vm_ip
```

This gives a detailed debug log. Look for lines like:

```
debug1: Authentications that can continue: password,publickey
```

...or any errors.

---

#### 🧪 6. **Temporarily Try Root Login**

If your regular user fails and you just need access:

* Set `PermitRootLogin yes` in `/etc/ssh/sshd_config`
* Restart SSH
* SSH in as root:

```bash
ssh root@your_vm_ip
```

But **disable this after fixing the main issue**, since root SSH access is a security risk.

---

### Let Me Know:

1. What user you’re logging in as
2. What the `/etc/ssh/sshd_config` file says for `PasswordAuthentication`
3. What the verbose SSH output (`ssh -vvv`) shows

I can guide you more precisely based on those.
