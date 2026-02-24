# Linux — Medium Questions

---

## 1. How do you troubleshoot network connectivity on a Linux server?

Work layer by layer — start from the network interface and move outward to the destination.

**Step 1: Check if the network interface is up.**

```bash
ip a                        # Show all interfaces and their IPs
ip link show eth0           # Check if eth0 is UP
```

If the interface is `DOWN`, bring it up:
```bash
ip link set eth0 up
```

**Step 2: Check the default route.**

```bash
ip route
# default via 10.0.0.1 dev eth0 — this is your gateway
```

If there's no default route, you can't reach anything outside the local subnet.

**Step 3: Ping the gateway.**

```bash
ping -c 3 10.0.0.1         # Replace with your gateway IP
```

If the gateway doesn't respond: local network issue (VLAN, physical cable, NIC).

**Step 4: Ping a public IP (bypass DNS).**

```bash
ping -c 3 8.8.8.8
```

If this fails but the gateway responds: routing or firewall issue between server and internet.

**Step 5: Check DNS resolution.**

```bash
nslookup google.com        # Or
dig google.com
cat /etc/resolv.conf       # Check configured DNS servers
```

If ping 8.8.8.8 works but DNS fails: DNS misconfiguration.

**Step 6: Test a specific port.**

```bash
curl -v telnet://hostname:443    # Test TCP port directly
nc -zv hostname 443              # netcat — connect test
curl -I https://hostname         # Full HTTP test
```

**Step 7: Check local firewall rules.**

```bash
iptables -L -n -v          # Check iptables rules
ufw status                 # If UFW is in use
```

**Step 8: In AWS — check Security Groups and NACLs.**

Security Groups block at instance level. NACLs block at subnet level. Neither shows up in `iptables`. Check the AWS console or:

```bash
# Check VPC flow logs to see if traffic is being accepted/rejected
```

---

**Quick diagnostic sequence:**

```
1. ip a                → interface up?
2. ip route            → default route set?
3. ping <gateway>      → local network ok?
4. ping 8.8.8.8        → routing ok?
5. dig google.com      → DNS ok?
6. nc -zv host port    → specific port reachable?
7. iptables -L         → local firewall blocking?
```

---

## 2. How do you identify and fix disk errors in Linux?

Disk errors fall into two categories: **filesystem errors** (corrupted metadata, orphaned inodes) and **hardware errors** (bad sectors, failing drive). Each needs a different approach.

**Step 1: Check if the disk is full first.**

```bash
df -h
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/xvda1       50G   49G  100M  99% /
```

A full disk causes "no space left on device" errors that look like filesystem corruption. Clear space before running fsck.

**Step 2: Check kernel messages for disk errors.**

```bash
dmesg | grep -i "error\|I/O error\|failed\|bad sector"
journalctl -k --since "1 hour ago" | grep -i "error\|disk\|sda"
```

Look for lines like:
```
[12345.678] end_request: I/O error, dev sda, sector 1234567
[12345.890] Buffer I/O error on device sda1, logical block 12345
```

These indicate hardware-level read/write failures.

**Step 3: Check SMART data (hardware health).**

```bash
smartctl -a /dev/sda       # Full SMART report
smartctl -H /dev/sda       # Quick health check
```

A failing drive will show: `Reallocated_Sector_Ct` increasing, or `SMART overall-health self-assessment: FAILED`.

**Step 4: Run fsck to check and fix filesystem errors.**

`fsck` must be run on an **unmounted** filesystem:

```bash
# Unmount first (or boot into rescue mode if it's the root disk)
umount /dev/xvda1

# Run fsck — -y auto-answers yes to all repairs
fsck -y /dev/xvda1

# Remount after
mount /dev/xvda1 /mount/point
```

For the root filesystem, schedule fsck on next reboot:
```bash
touch /forcefsck          # Some distros
# Or set fsck pass in /etc/fstab (last field = 1 for root, 2 for others)
```

**Step 5: If hardware is failing — replace the disk.**

No amount of fsck fixes bad sectors. If SMART reports reallocated sectors or the drive is failing:
- In AWS: take a snapshot, create a new EBS volume from the snapshot, attach it
- On bare metal: replace the physical drive, restore from backup

---

**Decision flowchart:**

```
dmesg I/O errors?
  YES → smartctl -H → FAILED → replace disk
  NO  → df -h full? → clear space
        → fsck shows errors? → fsck -y to repair
```

