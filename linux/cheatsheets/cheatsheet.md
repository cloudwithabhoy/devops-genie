# Linux Cheatsheet

> Quick-reference commands and concepts for Linux administration.

## Core Commands / Concepts

### Navigation
```bash
pwd               # print working directory
ls -lah           # list files with details
cd /path/to/dir   # change directory
find / -name "file.txt"  # find files
```

### File Operations
```bash
cp src dst        # copy
mv src dst        # move / rename
rm -rf dir        # remove directory recursively
chmod 755 file    # set permissions
chown user:group file  # change ownership
ln -s target link # create symlink
```

### Process Management
```bash
ps aux            # list all processes
top / htop        # interactive process viewer
kill -9 <PID>     # force kill process
systemctl start|stop|restart|status <service>
journalctl -u <service> -f   # follow service logs
```

### Package Management
```bash
# Debian/Ubuntu
apt update && apt upgrade
apt install <pkg>
apt remove <pkg>

# RHEL/CentOS/Fedora
dnf install <pkg>
dnf remove <pkg>
```

### Disk & Memory
```bash
df -h             # disk usage
du -sh /path      # directory size
free -h           # memory usage
lsblk             # list block devices
```

<!-- Add more commands here -->

## File Permissions

```
rwxrwxrwx  =  owner | group | others
chmod 644 file  →  rw-r--r--
chmod 755 file  →  rwxr-xr-x
```

## Common Patterns

<!-- Add common sysadmin patterns, one-liners, and pipe combos here -->
