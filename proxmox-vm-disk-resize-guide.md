# Proxmox VM Disk Resize Guide

This guide covers how to resize a VM disk in Proxmox and extend the filesystem within the VM to use the additional space.

## Prerequisites

- VM is powered off (recommended) or has hot-plug support enabled
- Backup of the VM (strongly recommended before any disk operations)
- SSH access to the VM
- Root/sudo access on the VM

## Step 1: Resize Disk in Proxmox

1. In Proxmox web interface, navigate to your VM
2. Go to **Hardware** tab
3. Select the disk you want to resize
4. Click **Resize disk**
5. Enter the additional size (e.g., +48 to add 48GB)
6. Click **Resize disk**
7. Restart the VM if it was powered off

## Step 2: Verify Disk Size in VM

SSH into your VM and check the current disk usage:

```bash
df -h
```

Check if the disk size has increased:

```bash
sudo fdisk -l /dev/sda
```

## Step 3: Extend Partition (if using LVM)

Check partition table:

```bash
sudo parted /dev/sda print
```

If there's unallocated space, extend the partition:

```bash
sudo parted /dev/sda resizepart 3 100%
```

When prompted:
- **Fix/Ignore?** → Type `Fix`
- **Partition number?** → Type `3` (or the appropriate LVM partition number)
- **End?** → Type `100%`

## Step 4: Extend Physical Volume

Resize the physical volume to use the extended partition:

```bash
sudo pvresize /dev/sda3
```

Verify the change:

```bash
sudo pvdisplay
```

## Step 5: Extend Logical Volume

Extend the logical volume to use all available space:

```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
```

Replace `ubuntu--vg-ubuntu--lv` with your actual logical volume path (visible in `df -h` output).

Verify the change:

```bash
sudo lvdisplay
```

## Step 6: Resize Filesystem

Resize the filesystem to use the extended logical volume:

```bash
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

## Step 7: Verify Results

Check the final disk usage:

```bash
df -h
```

You should now see the full allocated space available.

## Common Issues and Solutions

### Issue: "No space left on device" errors persist

**Solution**: Services may need to be restarted after disk expansion:

```bash
sudo systemctl restart [service-name]
```

### Issue: Partition table shows wrong size

**Solution**: Reboot the VM to ensure the kernel recognizes the new disk size:

```bash
sudo reboot
```

### Issue: LVM commands fail

**Solution**: Check if the correct physical volume device is being used:

```bash
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay
```

## Alternative: Non-LVM Systems

If your system doesn't use LVM, the process is simpler:

1. Extend partition: `sudo parted /dev/sda resizepart 1 100%`
2. Resize filesystem: `sudo resize2fs /dev/sda1`

## Safety Notes

- **Always backup your VM** before performing disk operations
- **Test in a non-production environment** first
- **Partition operations can be destructive** - double-check device names
- **Consider using VM snapshots** before starting the resize process

## Verification Commands

Use these commands to verify each step:

```bash
# Check disk hardware size
sudo fdisk -l /dev/sda

# Check partition table
sudo parted /dev/sda print

# Check physical volumes
sudo pvdisplay

# Check volume groups
sudo vgdisplay

# Check logical volumes
sudo lvdisplay

# Check filesystem usage
df -h
```