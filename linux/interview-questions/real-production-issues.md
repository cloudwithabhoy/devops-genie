# Linux — Real Production Issues

---

## 1. A user can't access a file — how do you grant permissions?

This is a structured diagnosis, not just running `chmod 777` and moving on. That's the wrong answer.

**Step 1 — Find out why they can't access it.**

```bash
# What are the current permissions?
ls -la /path/to/file
# -rw-r----- 1 appuser appgroup 2048 Jan 15 10:30 config.yaml
# Owner: appuser, Group: appgroup, Others: no access

# Who is the user trying to access it?
id username
# uid=1001(devuser) gid=1002(developers) groups=1002(developers),1003(docker)
```

The user (`devuser`) is in group `developers` (gid 1002). The file's group is `appgroup`. Since `devuser` is not in `appgroup` and is not the owner, they fall into "others" — which has no read permission (`---`).

**Step 2 — Choose the right fix.**

Option A — Add the user to the file's group (least privilege, recommended):

```bash
# Check the file's group
stat /path/to/file | grep Gid
# Gid: ( 1003/ appgroup)

# Add the user to the group
usermod -aG appgroup devuser

# User must log out and log back in for group change to take effect
# Verify
id devuser   # Should now show appgroup in groups list
```

Option B — Change the file's group to one the user is already in:

```bash
chgrp developers /path/to/file
```

Option C — Change file permissions to allow group read:

```bash
chmod g+r /path/to/file
# Now: -rw-r----- becomes the group can read it
```

Option D — Change ownership (if the user should own the file):

```bash
chown devuser:developers /path/to/file
```

**Never do `chmod 777` in production.** It grants read + write + execute to everyone on the system — including other processes, web servers, and potential attackers. If you find yourself reaching for 777, that's a sign to diagnose properly instead.

**Step 3 — Check directory permissions too.**

A file can have read permissions but still be inaccessible if the parent directory doesn't allow execute (traverse):

```bash
ls -la /path/to/
# drwxr-x--- 2 appuser appgroup 4096 Jan 15 10:00 .
# The directory requires appgroup membership to traverse it
```

```bash
# Permission breakdown for /path/to/file:
# User needs: execute on each parent directory, read on the file
# Missing execute on a parent dir → "Permission denied" even with correct file perms
```

**Real scenario:** A developer reported they couldn't read an application config file despite having `r--` on the file itself. The parent directory had `0700` permissions — only root could traverse it. A senior engineer had set that during an incident to lock down the directory, then forgotten. Fix: `chmod 755` on the directory. The developer didn't need to be added to any group.

---

## 2. A process is using high CPU/memory — how do you monitor and manage it?

The goal is: identify the process, understand why it's high, and decide between tuning, killing, or escalating — not just kill everything that's high.

**Step 1 — Identify the process.**

```bash
top          # Live view, sorted by CPU by default. Press 'M' to sort by memory
htop         # Better UI, can kill processes directly, shows per-core CPU usage
```

```bash
# Get the top 10 CPU consumers
ps aux --sort=-%cpu | head -10
# USER       PID  %CPU %MEM    VSZ   RSS TTY  STAT START   TIME COMMAND
# www-data  1234  95.0  2.3  234567 47890 ?   R    10:00  12:34 python worker.py

# Get the top 10 memory consumers
ps aux --sort=-%mem | head -10
```

**Step 2 — Understand what the process is doing.**

```bash
# What files does it have open?
lsof -p 1234 | head -20

# What system calls is it making?
strace -p 1234 -c    # Summary of syscalls (% time in each)

# What connections does it have?
ss -tp | grep 1234

# For Java — thread dump to see what's running
kill -3 1234    # Sends SIGQUIT to Java → prints thread dump to stdout
```

**Step 3 — Check if it's expected or runaway.**

```bash
# How long has this high CPU been going on?
# Check if it's a spike (expected: batch job, startup, heavy request) or sustained
sar -u 1 5    # CPU usage every 1 second, 5 samples

# Check for OOM situation alongside high memory
dmesg | grep -i "killed process"    # Kernel OOM killer logs
journalctl -k | grep -i oom
```

**Step 4 — Action based on diagnosis.**

```bash
# If it's a known runaway (stuck job, infinite loop, memory leak):
kill -15 1234      # SIGTERM — graceful shutdown, app can clean up
kill -9 1234       # SIGKILL — force kill, last resort (no cleanup, may corrupt state)

# If it's a legitimate process that's just too resource-hungry:
# Renice — lower its CPU priority (nice value: -20 highest, 19 lowest)
renice 10 -p 1234       # Run at lower priority, Linux scheduler gives it less CPU
ionice -c 3 -p 1234     # Idle I/O class — only gets disk I/O when nothing else needs it

# If it's a production service that needs to keep running:
# Don't kill — investigate the root cause first
# Is there a query running without an index? A loop that never terminates?
# Does restarting the service help (memory leak — yes; logic bug — probably not)
```

**Step 5 — Long-term: set resource limits.**

```bash
# Linux cgroups v2 (systemd services)
# In /etc/systemd/system/myapp.service:
[Service]
MemoryMax=512M
CPUQuota=200%    # 200% = 2 cores maximum

systemctl daemon-reload && systemctl restart myapp

# Kubernetes — resource limits in pod spec (no cgroups config needed)
resources:
  limits:
    cpu: "2"
    memory: "512Mi"
```

**Real scenario:** A Python Celery worker was pegging one CPU at 100% for 30 minutes. `strace -p <pid> -c` showed 90% of time in `select()` syscalls — the worker was in a busy-wait loop because the Redis broker connection had dropped silently. The worker thought it was waiting for tasks but was actually spinning. Fix: added heartbeat timeout to the Celery config (`BROKER_HEARTBEAT = 10`). Worker now detects dead connections and reconnects instead of spinning.

---

## 3. A user can't SSH into a server — how do you troubleshoot the issue?

SSH failures have a short list of root causes. Work through them systematically, from network to authentication.

**Step 1 — Is SSH reachable at all?**

```bash
# From the user's machine:
nc -zv <server-ip> 22
# Connection to 10.0.1.5 22 port [tcp/ssh] succeeded!  → SSH port is open
# nc: connect to 10.0.1.5 port 22 (tcp) failed: Connection refused  → sshd not running or port blocked

telnet <server-ip> 22
# SSH-2.0-OpenSSH_8.9p1 Ubuntu  → sshd is responding
```

If the port is blocked:
- **Security Group / Firewall:** On AWS, check the EC2 Security Group — is port 22 allowed from the user's IP? On-prem: check `iptables -L -n` or `ufw status`.
- **sshd not running:** `systemctl status sshd` on the server.

**Step 2 — Is the SSH key correct?**

```bash
# From the user's machine — verbose output shows exactly where authentication fails:
ssh -v user@server-ip
# debug1: Offering public key: /home/user/.ssh/id_rsa RSA
# debug1: Authentications that can continue: publickey
# debug1: No more authentication methods to try.
# → Key was offered but rejected
```

Check on the server:
```bash
cat ~/.ssh/authorized_keys    # Is the user's public key in here?
# The key must match EXACTLY — one extra space or newline breaks it
```

**Step 3 — Check file permissions on the server.**

SSH is extremely strict about permissions. If any permissions are too open, sshd silently rejects the key:

```bash
stat ~/.ssh
# Must be 700 (drwx------)
stat ~/.ssh/authorized_keys
# Must be 600 (-rw-------)

# Fix if wrong:
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R username:username ~/.ssh
```

**Step 4 — Check sshd configuration.**

```bash
sudo grep -E "PermitRootLogin|PasswordAuthentication|AllowUsers|DenyUsers|PubkeyAuthentication" /etc/ssh/sshd_config
```

Common causes:
- `PermitRootLogin no` — user is trying to SSH as root
- `PasswordAuthentication no` — user is trying password auth, only keys work
- `AllowUsers alice bob` — user's name is not in the AllowUsers list → rejected
- `PubkeyAuthentication no` — keys disabled (rare but happens after accidental config changes)

**Step 5 — Check auth logs on the server.**

```bash
sudo journalctl -u sshd -n 50    # Last 50 sshd log lines
sudo tail -50 /var/log/auth.log  # Debian/Ubuntu
sudo tail -50 /var/log/secure    # RHEL/CentOS

# What to look for:
# "Invalid user developer from 10.0.1.10"  → username wrong or not on system
# "Connection closed by authenticating user alice ...  [preauth]"  → key rejected
# "error: maximum authentication attempts exceeded"  → too many bad attempts, temporarily banned
# "User alice not allowed because not listed in AllowUsers"  → sshd config restriction
```

**Step 6 — Is the user locked out by fail2ban?**

```bash
sudo fail2ban-client status sshd
# Shows banned IPs — if the user's IP is in here, they're blocked

# Unban:
sudo fail2ban-client unban <user-ip>
```

**My quick triage order:**
1. Can I `nc -zv server 22`? (network/firewall)
2. `ssh -v user@server` — what does the verbose output show? (authentication)
3. Check `~/.ssh` permissions on the server (700/600)
4. Check `/var/log/auth.log` — it tells you exactly why SSH rejected the connection
5. Check `sshd_config` for AllowUsers restrictions

**Real scenario:** A new engineer couldn't SSH into a server despite having their key in `authorized_keys`. The verbose SSH output showed: `Authentication succeeded... but connection closed`. The server's `/etc/ssh/sshd_config` had `AllowUsers alice bob` — the new engineer's username wasn't in the list. Added their username to AllowUsers, reloaded sshd (`systemctl reload sshd`), and they could connect immediately. The auth log showed the rejection clearly: `User charlie not allowed because not listed in AllowUsers`.

> **Also asked as:** "If your EC2 instance is not reachable via SSH, what troubleshooting steps would you perform?" — covered above (network reachability → Security Group port 22 → key authentication → sshd_config → auth.log).

---

## 4. Which logs would you check if a user can't access your application?

Work from the outside in — start where the user's request first enters your system and follow it inward until you find the failure.

**Layer 1: DNS — does the domain resolve?**

```bash
# From your machine or the user's location:
dig app.example.com
nslookup app.example.com

# Check: does it return the expected IP? Is there a CNAME chain that's broken?
```

**Layer 2: Load balancer logs — ALB access logs.**

ALB logs every request: status code, response time, target IP, error reason.

```
# In AWS: S3 bucket configured as ALB access log target
# Log entry format:
https 2024-01-15T10:23:45 app/my-alb/abc123 1.2.3.4:51234 10.0.1.45:8080 0.001 0.003 0.000 200 200 456 1234 "GET https://app.example.com/login HTTP/1.1" ...

# 502 = target returned error or connection refused
# 503 = no healthy targets in target group
# 504 = target timed out (60s default)
```

**Layer 3: Application logs — what did your app see?**

```bash
# Nginx / Apache — access and error logs:
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# Application-level (varies by stack):
journalctl -u myapp -f           # systemd service
docker logs myapp --tail 100     # Docker container
kubectl logs deploy/myapp        # Kubernetes pod

# Look for: 4xx (client error), 5xx (server error), stack traces, OOM messages
```

**Layer 4: System logs — is the server itself healthy?**

```bash
# Disk full? (common cause of 500 errors)
df -h

# OOM killer killed the app?
dmesg | grep -i "killed process\|oom"
journalctl -k | grep -i oom

# System errors:
/var/log/syslog          # General system messages (Debian/Ubuntu)
/var/log/messages        # General system messages (RHEL/CentOS)
/var/log/auth.log        # Authentication failures (if login-gated)
```

**Layer 5: AWS-specific — Security Groups, NACLs, VPC flow logs.**

```bash
# VPC flow logs show accepted/rejected traffic at the subnet level
# Check in CloudWatch Logs if flow logs are enabled:
# REJECT entries mean NACL or SG is blocking

# Also check: WAF logs if WAF is attached to the ALB
# WAF can block specific IPs, user agents, or geo-locations
```

---

**Decision tree:**

```
User can't access the app
  ↓
DNS resolves? No → Fix DNS record
  ↓ Yes
ALB returning error? Yes → Check target group health, ALB error logs
  ↓ No error at ALB
App log showing errors? Yes → Check stack trace, fix application bug
  ↓ No app errors
System healthy? df -h, dmesg → disk full, OOM → fix
  ↓
VPC flow logs showing REJECT? → Security Group / NACL blocking the user's IP
```

**Real scenario:** A user reported they couldn't access the app but others could. ALB logs showed their requests were reaching the load balancer and returning 200 — but the user saw a 403. Root cause: WAF had a geo-blocking rule. The user was connecting via a VPN that exit-nodded through a blocked country. Disabling WAF geo-blocking for their IP unblocked them in minutes.

