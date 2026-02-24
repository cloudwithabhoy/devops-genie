# Linux — Basic Questions

---

## 1. What is the difference between a Hard Link and a Soft Link?

Both create a reference to a file, but they work at fundamentally different levels of the filesystem.

**Hard Link:**

A hard link is a second directory entry pointing to the **same inode** (same data on disk). Both the original filename and the hard link are equal — neither is "the original." Deleting one does not delete the data; the data is deleted only when the last hard link to the inode is removed.

```bash
ls -i file.txt          # 1234567 file.txt  (inode number)
ln file.txt hardlink.txt
ls -i hardlink.txt      # 1234567 hardlink.txt  (same inode!)

rm file.txt             # Data still exists — hardlink.txt still works
cat hardlink.txt        # Still readable
```

**Soft Link (Symbolic Link):**

A soft link (symlink) is a separate file that contains a **path** to another file. It's a pointer. If the target is deleted or moved, the symlink breaks — it points to nothing.

```bash
ln -s /etc/nginx/nginx.conf nginx.conf.link
ls -la nginx.conf.link
# lrwxrwxrwx 1 root root 22 Jan 15 10:30 nginx.conf.link -> /etc/nginx/nginx.conf

rm /etc/nginx/nginx.conf
cat nginx.conf.link     # Error: No such file or directory (dangling symlink)
```

**Key differences at a glance:**

| | Hard Link | Soft Link |
|---|---|---|
| Points to | Inode (data) | Path (filename) |
| Works if original deleted? | Yes — data survives | No — becomes dangling |
| Cross-filesystem? | No — same filesystem only | Yes — can link across filesystems |
| Can link directories? | No (prevents cycles) | Yes |
| `ls -la` output | Same as regular file | Shows `->` target |

**When you use each in practice:**

- **Hard links:** Backup tools like `rsync --link-dest` use hard links to create space-efficient snapshots. If a file didn't change, the backup directory has a hard link to the previous snapshot's inode — no disk space wasted.
- **Soft links:** Version management (`/usr/bin/python3 -> /usr/bin/python3.12`), nginx site configs (`/etc/nginx/sites-enabled/ -> ../sites-available/my-site`), shared libraries with versioned names.

**The inode number test:** `ls -i file hardlink softlink` — hard links share the same inode number; soft links have a different inode.

---

## 2. What is the latest version of Ubuntu?

As of 2024, the latest LTS (Long-Term Support) release is **Ubuntu 24.04 LTS "Noble Numbat"**, released April 2024. LTS releases are supported for 5 years (standard) and up to 10 years with ESM (Extended Security Maintenance). This is the version used in most production server environments.

The latest non-LTS release is **Ubuntu 24.10 "Oracular Oriole"** — short-term release, 9 months of support, used for testing newer kernels and desktop features.

**Why LTS matters for servers:**

```bash
lsb_release -a
# Distributor ID: Ubuntu
# Release:        24.04
# Codename:       noble
```

In production, always use LTS. Non-LTS releases go EOL in 9 months — a running server on a non-LTS version will stop receiving security patches quickly.

---

## 3. What are the most common Linux commands you use daily?

These cover navigation, file management, process management, networking, and system info:

```bash
# --- Navigation ---
pwd               # Print current directory
ls -la            # List files with permissions, hidden files, sizes
cd /var/log       # Change directory
find / -name "*.log" -mtime -1   # Find files modified in last 1 day

# --- File operations ---
cp file.txt /tmp/       # Copy file
mv file.txt /backup/    # Move / rename file
rm -rf /tmp/old/        # Delete directory recursively
mkdir -p /opt/app/conf  # Create nested directories
cat file.txt            # View file contents
less file.txt           # Page through large files
head -20 file.txt       # First 20 lines
tail -f /var/log/syslog # Follow log in real-time

# --- Text processing ---
grep "error" app.log           # Search for pattern
grep -v "INFO" app.log         # Show lines NOT matching
awk '{print $1, $3}' file.txt  # Print columns 1 and 3
sed 's/old/new/g' file.txt     # Replace text

# --- Permissions ---
chmod 755 script.sh     # rwxr-xr-x
chown www-data:www-data /var/www/html/

# --- Process management ---
ps aux                  # All running processes
top / htop              # Interactive process viewer
kill -9 <PID>           # Force kill a process
pkill nginx             # Kill by name

# --- System info ---
df -h                   # Disk usage (human-readable)
du -sh /var/log/        # Directory size
free -h                 # RAM usage
uname -r                # Kernel version
uptime                  # System uptime + load average

# --- Networking ---
curl -I https://google.com    # HTTP headers
netstat -tlnp / ss -tlnp      # Listening ports
ping 8.8.8.8                  # Connectivity test
```

---

## 4. How do you check and manage running processes in Linux?

**Check if a specific process is running:**

```bash
ps aux | grep nginx
# USER   PID  %CPU %MEM  COMMAND
# www    1234  0.0  0.1  nginx: worker process

# Or use pgrep (returns just the PID):
pgrep nginx         # Returns: 1234
pgrep -l nginx      # Returns: 1234 nginx (with name)
```

**List all running processes:**

```bash
ps aux              # All processes, all users (snapshot)
top                 # Real-time, sorted by CPU
htop                # Interactive, color-coded (needs install)

# Sort ps output by memory:
ps aux --sort=-%mem | head -10

# Sort by CPU:
ps aux --sort=-%cpu | head -10
```

**Find a PID and kill it — in a single command:**

```bash
# Option 1: pkill (kill by name)
pkill nginx         # Sends SIGTERM (graceful)
pkill -9 nginx      # Sends SIGKILL (force kill)

# Option 2: kill + pgrep (single line)
kill -9 $(pgrep nginx)

# Option 3: killall
killall nginx

# Option 4: pidof
kill $(pidof nginx)
```

The difference between SIGTERM (`kill -15` / default) and SIGKILL (`kill -9`):
- **SIGTERM** — asks the process to shut down gracefully. The process can clean up, close connections, flush buffers. Always try this first.
- **SIGKILL** — immediately terminates the process. No cleanup. Use only if SIGTERM doesn't work.

```bash
# Check if process is still alive after kill:
pgrep nginx && echo "still running" || echo "stopped"
```

> **Also asked as:** "How to check one running process?" — covered above (`ps aux | grep <name>` or `pgrep <name>`).

---

## 5. How do you check disk usage and free memory?

**Disk usage:**

```bash
# Overall filesystem usage
df -h
# Filesystem      Size  Used Avail Use%  Mounted on
# /dev/xvda1       20G   12G  7.5G  62%  /
# tmpfs           1.9G     0  1.9G   0%  /dev/shm

# Size of a specific directory
du -sh /var/log/
# 2.3G  /var/log/

# Top 10 largest directories under /var
du -h /var/ | sort -rh | head -10

# Find large files
find / -size +100M -type f 2>/dev/null
```

**Free memory:**

```bash
free -h
#               total    used    free  shared  buff/cache  available
# Mem:           7.6G    3.2G    1.1G    212M        3.3G       4.0G
# Swap:          2.0G    0.0G    2.0G

# "available" is what matters — not "free"
# Linux uses free RAM for disk cache (buff/cache)
# "available" = free + memory that can be reclaimed from cache
```

`available` is the real number — a server showing `free: 100M` but `available: 4G` has plenty of memory. The 3.9G is used as disk cache and will be freed immediately if a process needs it.

```bash
# Real-time memory monitoring
watch -n 1 free -h   # Refresh every 1 second

# Detailed memory info
cat /proc/meminfo | grep -E "MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree"
```

**Swap usage to watch:**

```bash
# If swap is being used significantly, the system is under memory pressure
# Processes reading/writing to swap = performance degradation
swapon --show
vmstat -s | grep swap
```

---

## 6. How do you archive and compress a directory in Linux?

```bash
# Create a compressed archive (most common)
tar -czvf archive.tar.gz /path/to/directory/
# -c = create
# -z = compress with gzip
# -v = verbose (show files being added)
# -f = specify filename

# Examples:
tar -czvf backup-2024-01-15.tar.gz /var/www/html/
tar -czvf logs.tar.gz /var/log/nginx/

# Extract archive
tar -xzvf archive.tar.gz              # Extract in current directory
tar -xzvf archive.tar.gz -C /opt/     # Extract to /opt/

# List contents without extracting
tar -tzvf archive.tar.gz

# bzip2 compression (better compression ratio, slower)
tar -cjvf archive.tar.bz2 directory/

# xz compression (best compression ratio, slowest)
tar -cJvf archive.tar.xz directory/
```

**Compression comparison:**

| Format | Command flag | Speed | Compression ratio |
|---|---|---|---|
| gzip | `-z` | Fast | Moderate |
| bzip2 | `-j` | Slower | Better |
| xz | `-J` | Slowest | Best |

For most use cases (log rotation, backups, transferring files), `tar.gz` is the standard — a balance of speed and compression.

```bash
# Quick size comparison on /var/log/nginx:
tar -czf - /var/log/nginx/ | wc -c    # gzip size in bytes
tar -cjf - /var/log/nginx/ | wc -c    # bzip2 size in bytes
```

---

## 7. What does `chmod 755` mean, and what exactly happens when you set it?

`chmod` controls file permissions. Linux permissions are split into three groups: **owner**, **group**, and **others**. Each group gets three bits: **read (r=4)**, **write (w=2)**, **execute (x=1)**.

**Decoding 755:**

```
7 = 4+2+1 = rwx → owner can read, write, AND execute
5 = 4+0+1 = r-x → group can read and execute, NOT write
5 = 4+0+1 = r-x → others can read and execute, NOT write
```

**What happens in practice:**

```bash
chmod 755 deploy.sh

ls -la deploy.sh
# -rwxr-xr-x  1 ubuntu ubuntu  512 Jan 15 10:00 deploy.sh
#  ^^^------  = owner: rwx
#     ---r-x  = group: r-x
#        r-x  = others: r-x

# Owner (ubuntu): can run the script, edit it, read it
# Group members:  can run the script and read it, cannot edit
# Everyone else:  same as group — run and read, cannot edit
```

**Common chmod values:**

```bash
chmod 644 config.yml    # Owner: rw-, Group/Others: r-- (read-only for others)
chmod 600 id_rsa        # Owner: rw-, Group/Others: --- (private key — only owner reads)
chmod 755 /var/www/html # Web directory — everyone can read/traverse, only owner writes
chmod 777 /tmp/shared   # Everyone full access (dangerous — avoid in production)
chmod 400 key.pem       # AWS requires this — owner read-only, others nothing
```

**Symbolic notation (same result as numeric):**

```bash
chmod u=rwx,g=rx,o=rx deploy.sh   # Equivalent to chmod 755
chmod u+x script.sh                # Add execute for owner only
chmod o-w file.txt                 # Remove write for others
```

**SSH private key must be 600 or 400:**

```bash
chmod 400 my-ec2-key.pem
# If permissions are too open (e.g., 644), SSH refuses to use the key:
# "Permissions 0644 for 'my-ec2-key.pem' are too open."
```

> **Also asked as:** "If I provide chmod 755, what exactly will happen?" — covered above (owner: rwx, group: r-x, others: r-x).

---

## 8. What is `chown` and how do you use it?

`chown` (change ownership) changes who owns a file or directory. Every file in Linux has an **owner** (a user) and a **group** (a group). `chown` changes one or both.

```bash
# Syntax: chown [owner]:[group] file
chown ubuntu file.txt           # Change owner to ubuntu, group unchanged
chown ubuntu:www-data file.txt  # Change owner to ubuntu, group to www-data
chown :www-data file.txt        # Change group only, owner unchanged

# Recursive — change ownership of all files in directory
chown -R www-data:www-data /var/www/html/

# Check current ownership
ls -la /var/www/html/index.html
# -rw-r--r-- 1 www-data www-data 1234 Jan 15 index.html
```

**Why it matters in practice:**

```bash
# Apache/Nginx run as www-data user
# Web files must be owned by www-data (or readable by it) for the server to serve them
sudo chown -R www-data:www-data /var/www/html/

# Application deployed as ubuntu user but needs to write to /opt/app/logs
sudo chown -R ubuntu:ubuntu /opt/app/

# Wrong ownership → permission denied errors in application logs
# Most common cause: deploying files as root, app runs as non-root
```

**`chown` vs `chmod`:**

| Command | Changes | Example |
|---|---|---|
| `chown` | Who owns the file | `chown ubuntu file` |
| `chmod` | What the owner/group/others can do | `chmod 755 file` |

---

## 9. How do you list all SSH users in Linux?

SSH users are simply Linux system users. There's no separate "SSH user" concept — any user account that isn't explicitly denied SSH access can use SSH (subject to `sshd_config` rules).

```bash
# List all users on the system
cat /etc/passwd

# Each line: username:password:UID:GID:comment:home:shell
# root:x:0:0:root:/root:/bin/bash
# ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
# www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin

# List only users with actual shell access (exclude service accounts)
cat /etc/passwd | grep -v "nologin\|false" | cut -d: -f1
# root
# ubuntu

# Who is currently logged in via SSH
who
# ubuntu   pts/0   2024-01-15 10:30 (203.0.113.45)

# Recent login history
last | head -20

# List users allowed in sshd_config (if AllowUsers is set)
grep "AllowUsers\|DenyUsers\|AllowGroups\|DenyGroups" /etc/ssh/sshd_config

# Check which users have SSH keys configured
find /home -name "authorized_keys" -exec echo "User: {}" \;
# /home/ubuntu/.ssh/authorized_keys
# /home/deploy/.ssh/authorized_keys
```

**Service accounts (nologin shell) cannot SSH in:**

```bash
# www-data, nobody, daemon, etc. have /usr/sbin/nologin as shell
# Even if someone tries to SSH as www-data, the system rejects it
# because the shell is not a real shell
```

---

## 10. An Apache server is running — where do you check logs?

Apache logs are in different locations depending on the OS:

```bash
# Ubuntu / Debian
/var/log/apache2/
├── access.log     # Every HTTP request: who, what, when, response code
├── error.log      # Errors: 404s, 500s, config issues, module errors
└── other_vhosts_access.log  # Requests to virtual hosts

# CentOS / RHEL / Amazon Linux
/var/log/httpd/
├── access_log
└── error_log
```

**Reading the logs:**

```bash
# Watch access log in real-time (like tail -f)
sudo tail -f /var/log/apache2/access.log
# 203.0.113.45 - - [15/Jan/2024:10:30:01 +0000] "GET /index.html HTTP/1.1" 200 1234

# Search for errors
sudo grep "error\|Error\|ERROR" /var/log/apache2/error.log | tail -20

# Find 404s (not found)
sudo grep " 404 " /var/log/apache2/access.log | tail -10

# Find 500s (server errors)
sudo grep " 500 " /var/log/apache2/access.log | tail -10

# Count requests by IP (who's hitting the server most)
sudo awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn | head -10
```

**Virtual host logs (if configured):**

```bash
# Check Apache config for custom log paths
grep -r "ErrorLog\|CustomLog" /etc/apache2/sites-enabled/
# ErrorLog /var/log/apache2/mysite-error.log
# CustomLog /var/log/apache2/mysite-access.log combined
```

---

## 11. What kinds of logs can you find in `/var/log`?

`/var/log` is the central directory for all system and application logs.

```bash
ls /var/log/
```

**Common log files:**

| File | Contents |
|---|---|
| `/var/log/syslog` (Ubuntu) | General system messages — kernel, init, services |
| `/var/log/messages` (CentOS) | Same as syslog on RHEL-based systems |
| `/var/log/auth.log` | SSH logins, sudo usage, authentication attempts, PAM |
| `/var/log/kern.log` | Kernel messages (hardware, driver issues) |
| `/var/log/dmesg` | Boot-time kernel messages (hardware detection) |
| `/var/log/apt/` | Package installation/upgrade history (Ubuntu) |
| `/var/log/nginx/` | Nginx access and error logs |
| `/var/log/apache2/` | Apache access and error logs |
| `/var/log/mysql/` | MySQL/MariaDB errors and slow queries |
| `/var/log/cron` | Cron job execution history |
| `/var/log/faillog` | Failed login attempts |
| `/var/log/wtmp` | Login history (read with `last` command) |
| `/var/log/btmp` | Failed login attempts (read with `lastb`) |

```bash
# Most useful for security investigation:
sudo tail -f /var/log/auth.log
# Jan 15 10:30:01 server sshd[1234]: Accepted publickey for ubuntu from 203.0.113.45
# Jan 15 10:31:55 server sshd[1235]: Failed password for root from 185.x.x.x

# Check for brute force attempts
sudo grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn
```

---

## 12. What is the purpose of `grep` and how do you use the `-v` flag?

`grep` searches for a pattern in files (or stdin) and prints matching lines. It's one of the most-used commands in Linux for filtering log output, searching config files, and processing text.

```bash
# Basic usage
grep "error" /var/log/syslog         # Lines containing "error"
grep -i "error" /var/log/syslog      # Case-insensitive
grep -n "error" app.log              # Show line numbers
grep -r "TODO" /opt/app/src/         # Recursive through directories
grep -c "ERROR" app.log              # Count matching lines (not print them)

# With pipes (most common real usage)
ps aux | grep nginx                  # Find nginx process
tail -f /var/log/app.log | grep -i "exception"   # Watch for exceptions in real-time
```

**`-v` flag — invert match (exclude lines matching the pattern):**

```bash
# List all lines from linux.txt EXCEPT lines containing "linux"
grep -v "linux" linux.txt

# Practical examples:
cat /etc/passwd | grep -v "nologin"          # Show only users with real shells
ps aux | grep -v "grep"                       # Show processes, exclude the grep itself
grep -v "^#" /etc/nginx/nginx.conf           # Show config without comment lines
grep -v "^$" file.txt                         # Remove blank lines
grep -v "DEBUG\|INFO" app.log                 # Show only WARN/ERROR lines
```

**Combining flags:**

```bash
grep -vi "debug" app.log          # Invert + case-insensitive
grep -v "linux" linux.txt -c      # Count lines NOT containing "linux"
```

---

## 13. What is the difference between `cron` and `at`, and how do you schedule a script every 5 minutes?

Both schedule commands to run at a specific time, but they serve different purposes:

| | `cron` | `at` |
|---|---|---|
| Purpose | Recurring schedules | One-time future execution |
| Persistence | Runs indefinitely on schedule | Runs once, then done |
| Config | `crontab` file | Command-line or stdin |
| Use case | Backups, monitoring, log rotation | Run a script once at 2 AM tonight |

**`cron` — for recurring jobs:**

```bash
# Edit the current user's crontab
crontab -e

# Crontab syntax:
# ┌─── minute (0-59)
# │  ┌─── hour (0-23)
# │  │  ┌─── day of month (1-31)
# │  │  │  ┌─── month (1-12)
# │  │  │  │  ┌─── day of week (0-7, 0=Sun)
# │  │  │  │  │
# *  *  *  *  *  command

# Run a script every 5 minutes:
*/5 * * * * /opt/scripts/check-health.sh

# Examples:
0 2 * * * /opt/backup/backup.sh          # Daily at 2 AM
0 * * * * /opt/scripts/hourly-sync.sh    # Every hour at :00
*/5 * * * * /usr/bin/python3 /opt/monitor.py  # Every 5 minutes

# View scheduled cron jobs
crontab -l

# System-wide cron jobs
cat /etc/crontab
ls /etc/cron.d/       # Per-application cron files
ls /etc/cron.daily/   # Scripts run daily by cron
```

**`at` — for one-time jobs:**

```bash
# Install if not present
sudo apt install at

# Schedule a script to run at 11 PM tonight
at 23:00 today
at> /opt/scripts/maintenance.sh
at> Ctrl+D (to submit)

# Other time formats
at now + 2 hours
at 2:30 AM tomorrow
at midnight

# View scheduled at jobs
atq
# 1  2024-01-15 23:00 a ubuntu

# Remove an at job
atrm 1
```

**The 5-minute cron entry in detail:**

```bash
*/5 * * * * /opt/scripts/check-health.sh >> /var/log/health-check.log 2>&1
# */5 = "every 5 minutes" (0, 5, 10, 15, ... 55)
# >> = append output to log file
# 2>&1 = redirect stderr to stdout (capture errors too)
```

Always use absolute paths in crontab — cron runs with a minimal environment and doesn't know your `PATH`.


---

## 14. What is the one-liner command to delete or wipe out a log file?

```bash
# Option 1: Truncate the file to zero bytes (file remains, content wiped)
> /var/log/app/error.log

# Option 2: Same using truncate command
truncate -s 0 /var/log/app/error.log

# Option 3: Using tee
: | tee /var/log/app/error.log

# Option 4: Permanently delete the file
rm /var/log/app/error.log
```

**The difference between truncating and deleting:**

```bash
# Truncate (> filename) — keeps the file, wipes content
# Use this when: a running process (nginx, apache) has the file open
# If you rm a file that a process has open, the process keeps writing to the
# deleted inode — disk space is NOT freed until the process closes the file
# Truncating the file frees the content but keeps the file descriptor valid

# Delete (rm) — removes the file entirely
# Use this when: no process has the file open, or you want to recreate it fresh
```

**Wipe all log files in a directory (one-liner):**

```bash
# Truncate all .log files in a directory
find /var/log/myapp/ -name "*.log" -exec truncate -s 0 {} \;

# Delete all .log files older than 7 days
find /var/log/myapp/ -name "*.log" -mtime +7 -delete
```

**For production log rotation, use `logrotate` instead of manual deletion:**

```bash
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 7        # Keep 7 days of logs
    compress        # gzip old logs
    missingok       # Don't error if log file missing
    notifempty      # Don't rotate if file is empty
    postrotate
        systemctl reload myapp   # Tell the app to reopen its log file
    endscript
}
```

---

## 15. What is the command to check listening ports in Linux?

```bash
# Option 1: ss (modern, preferred over netstat)
ss -tlnp
# -t = TCP
# -l = listening only
# -n = show port numbers (not service names)
# -p = show process name/PID

# Output:
# State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port  Process
# LISTEN  0       128     0.0.0.0:22            0.0.0.0:*          sshd
# LISTEN  0       511     0.0.0.0:80            0.0.0.0:*          nginx
# LISTEN  0       128     127.0.0.1:5432        0.0.0.0:*          postgres

# Option 2: netstat (older, may need net-tools package)
netstat -tlnp

# Option 3: check a specific port
ss -tlnp | grep :8080

# Option 4: lsof — show which process is using a port
lsof -i :8080
# COMMAND  PID     USER  FD  TYPE  DEVICE  SIZE  NODE  NAME
# java     1234    app   45u IPv4  12345   0t0   TCP   *:8080 (LISTEN)

# Option 5: check if a specific port is open (from the same machine)
nc -zv localhost 8080
# Connection to localhost 8080 port [tcp/*] succeeded!
```

**Check all listening ports (TCP + UDP):**

```bash
ss -tulnp
# -u = include UDP as well
```
