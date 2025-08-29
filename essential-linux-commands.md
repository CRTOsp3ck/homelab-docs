# Essential Linux Commands Cheat Sheet

## 🔥 Kill Processes & Ports

### Kill process by port
```bash
# Find process using specific port
sudo lsof -i :8080
sudo netstat -tulpn | grep :8080
sudo ss -tulpn | grep :8080

# Kill process by port
sudo fuser -k 8080/tcp
sudo lsof -ti:8080 | xargs kill -9
```

### Kill processes
```bash
# Kill by process ID
kill 1234
kill -9 1234        # Force kill

# Kill by process name
killall firefox
pkill -f "process_name"

# Kill all processes by user
pkill -u username
```

## 📁 File & Directory Operations

### Navigation
```bash
pwd                 # Print working directory
ls                  # List files
ls -la              # List with details and hidden files
ls -lh              # List with human-readable sizes
cd /path/to/dir     # Change directory
cd ..               # Go up one directory
cd ~                # Go to home directory
cd -                # Go to previous directory
```

### File Operations
```bash
# Create
touch file.txt      # Create empty file
mkdir dirname       # Create directory
mkdir -p path/to/dir # Create nested directories

# Copy & Move
cp file1 file2      # Copy file
cp -r dir1 dir2     # Copy directory recursively
mv file1 file2      # Move/rename file
mv file1 /path/     # Move to directory

# Delete
rm file.txt         # Delete file
rm -r dirname       # Delete directory recursively
rm -rf dirname      # Force delete directory
rmdir dirname       # Delete empty directory
```

### File Viewing
```bash
cat file.txt        # Display entire file
head file.txt       # First 10 lines
head -n 20 file.txt # First 20 lines
tail file.txt       # Last 10 lines
tail -f file.txt    # Follow file (real-time)
less file.txt       # Page through file
more file.txt       # Page through file
wc file.txt         # Word, line, character count
```

## 🔍 Search & Find

### Find files
```bash
find /path -name "*.txt"           # Find by name
find /path -type f -name "*.log"   # Find files by extension
find /path -type d -name "dirname" # Find directories
find /path -size +100M             # Find files larger than 100MB
find /path -mtime -7               # Files modified in last 7 days
```

### Search in files
```bash
grep "pattern" file.txt            # Search in file
grep -r "pattern" /path/           # Recursive search
grep -i "pattern" file.txt         # Case-insensitive
grep -n "pattern" file.txt         # Show line numbers
grep -v "pattern" file.txt         # Invert match (exclude)
```

## 📊 System Information

### System status
```bash
ps aux              # List all processes
top                 # Real-time process viewer
htop                # Enhanced process viewer
df -h               # Disk space usage
du -h /path         # Directory size
du -sh *            # Size of each item in current dir
free -h             # Memory usage
uptime              # System uptime
whoami              # Current user
id                  # User and group IDs
```

### Hardware info
```bash
lscpu               # CPU information
lsmem               # Memory information
lsblk               # Block devices
lsusb               # USB devices
lspci               # PCI devices
uname -a            # Kernel and system info
hostnamectl         # System hostname and info
```

## 🌐 Network Commands

### Network status
```bash
ping google.com     # Test connectivity
wget https://url    # Download file
curl https://api    # Make HTTP request
netstat -tulpn      # Network connections
ss -tulpn           # Modern netstat alternative
ip addr show        # Show IP addresses
ip route show       # Show routing table
```

### Network tools
```bash
nslookup domain.com # DNS lookup
dig domain.com      # DNS lookup (detailed)
traceroute google.com # Trace network path
iftop               # Network usage by connection
netstat -i          # Network interface statistics
```

## 🔐 Permissions & Ownership

### Change permissions
```bash
chmod 755 file.txt  # rwxr-xr-x
chmod +x script.sh  # Add execute permission
chmod -x file.txt   # Remove execute permission
chmod u+r file.txt  # Add read for user
chmod go-w file.txt # Remove write for group/others
```

### Change ownership
```bash
chown user file.txt        # Change owner
chown user:group file.txt  # Change owner and group
chgrp group file.txt       # Change group
chown -R user:group dir/   # Recursive ownership change
```

### Permission values
```bash
# 4 = read (r)
# 2 = write (w)  
# 1 = execute (x)
# 755 = rwxr-xr-x (owner: rwx, group: r-x, others: r-x)
# 644 = rw-r--r-- (owner: rw-, group: r--, others: r--)
```

## 📝 Text Processing

### Text manipulation
```bash
sort file.txt       # Sort lines
sort -r file.txt    # Reverse sort
sort -n file.txt    # Numeric sort
uniq file.txt       # Remove duplicate lines
cut -d',' -f1,3 file.csv # Extract columns 1,3 from CSV
awk '{print $1}' file.txt # Print first column
sed 's/old/new/g' file.txt # Replace text
tr 'a-z' 'A-Z' < file.txt # Convert to uppercase
```

### File comparison
```bash
diff file1 file2    # Show differences
comm file1 file2    # Compare sorted files
cmp file1 file2     # Compare files byte by byte
```

## 🗜️ Archives & Compression

### tar archives
```bash
tar -czf archive.tar.gz /path/    # Create gzipped tar
tar -xzf archive.tar.gz           # Extract gzipped tar
tar -tf archive.tar.gz            # List contents
tar -czf backup.tar.gz --exclude='*.log' /path/ # Exclude files
```

### Other compression
```bash
zip -r archive.zip /path/    # Create zip
unzip archive.zip            # Extract zip
gzip file.txt               # Compress file
gunzip file.txt.gz          # Decompress file
```

## 📦 Package Management

### Ubuntu/Debian (apt)
```bash
sudo apt update             # Update package list
sudo apt upgrade            # Upgrade packages
sudo apt install package   # Install package
sudo apt remove package    # Remove package
sudo apt search keyword     # Search packages
apt list --installed       # List installed packages
```

### CentOS/RHEL (yum/dnf)
```bash
sudo yum update            # Update packages
sudo yum install package   # Install package
sudo yum remove package    # Remove package
sudo yum search keyword    # Search packages
```

## 🔄 Process Management

### Background processes
```bash
command &              # Run in background
nohup command &         # Run immune to hangups
jobs                    # List background jobs
fg %1                   # Bring job 1 to foreground
bg %1                   # Send job 1 to background
disown %1               # Remove job from shell
```

### Process monitoring
```bash
ps aux | grep process   # Find specific process
pgrep process_name      # Find process ID by name
pstree                  # Show process tree
watch -n 2 'ps aux'     # Monitor processes every 2 seconds
```

## ⏰ Scheduling & Time

### Cron jobs
```bash
crontab -e              # Edit cron jobs
crontab -l              # List cron jobs
crontab -r              # Remove all cron jobs

# Cron format: minute hour day month weekday command
# 0 2 * * * /path/to/script    # Run daily at 2 AM
# */15 * * * * command         # Run every 15 minutes
```

### Time commands
```bash
date                    # Current date and time
date '+%Y-%m-%d %H:%M'  # Formatted date
timedatectl             # System time settings
```

## 🔗 Useful Shortcuts

### Command line shortcuts
```bash
Ctrl+C                  # Kill current command
Ctrl+Z                  # Suspend current command
Ctrl+D                  # End of file/exit
Ctrl+L                  # Clear screen
Ctrl+A                  # Move to beginning of line
Ctrl+E                  # Move to end of line
Ctrl+R                  # Search command history
!!                      # Repeat last command
!n                      # Execute command n from history
history                 # Show command history
```

### Redirection & Pipes
```bash
command > file          # Redirect output to file
command >> file         # Append output to file
command < file          # Use file as input
command 2> file         # Redirect errors to file
command &> file         # Redirect all output to file
command1 | command2     # Pipe output of cmd1 to cmd2
command | tee file      # Output to both screen and file
```

## 🌍 Environment & Variables

### Environment
```bash
env                     # Show all environment variables
echo $PATH              # Show PATH variable
export VAR=value        # Set environment variable
unset VAR               # Remove environment variable
which command           # Show command location
whereis command         # Show command locations
alias ll='ls -la'       # Create alias
unalias ll              # Remove alias
```

## 🔧 System Services (systemd)

```bash
sudo systemctl start service    # Start service
sudo systemctl stop service     # Stop service
sudo systemctl restart service  # Restart service
sudo systemctl enable service   # Enable at boot
sudo systemctl disable service  # Disable at boot
sudo systemctl status service   # Check service status
journalctl -u service           # View service logs
```

## 💡 Pro Tips

- Use `man command` to get detailed help for any command
- Use `command --help` for quick help
- Use `which command` to find where a command is located  
- Use `history | grep keyword` to search your command history
- Use `!!` to repeat the last command
- Use `sudo !!` to repeat the last command with sudo
- Use Tab for auto-completion
- Use `*` and `?` as wildcards in file operations
