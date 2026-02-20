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

