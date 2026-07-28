# Disk & Storage Management

---

## 1. Checking Disk Usage: `df` and `du`

### `df` — Disk Free (filesystem-level usage)

```bash
df                      # basic output, sizes in blocks (not human friendly)
df -h                      # human-readable (GB/MB) — almost always what you actually want
df -T                         # show filesystem TYPE (ext4, xfs, etc.) per mount
df -i                            # inode usage instead of block/space usage (covered in Topic 3 — worth remembering df -i exists here specifically)
df -h /var                          # usage for the filesystem containing /var specifically
```

Shows usage **per mounted filesystem/partition** — how full each mounted disk/partition is.

### `du` — Disk Usage (per file/directory)

```bash
du -h file.txt                   # size of a specific file, human readable
du -sh /var/log                     # -s = summary (total only, not every subfolder), -h = human readable — VERY commonly used combo
du -h --max-depth=1 /var               # show size of each immediate subfolder, one level deep — great for hunting what's eating disk space
du -ah /home/arjun | sort -rh | head -10   # find the top 10 largest files/folders under a path (extremely common real troubleshooting one-liner)
```

### `df` vs `du` — frequently asked distinction

| | `df` | `du` |
| --- | --- | --- |
| Scope | Whole filesystem/mounted partition | Specific files/directories you point it at |
| Answers | "How full is this disk/partition?" | "What's taking up space inside this folder?" |
| Common use | Quick check: is the disk nearly full? | Drill down: WHAT is filling up the disk? |

**Real troubleshooting flow interviewers love hearing:** "I'd first run `df -h` to confirm which partition is actually full, then `cd` into that mount point and run `du -h --max-depth=1` repeatedly, drilling down level by level until I find the specific directory or file responsible."

### Common gotcha: `df` shows full disk but `du` doesn't add up

This happens when a file has been **deleted but is still open by a running process** — the space isn't actually freed until the process closes the file handle, but `du` (which scans the directory tree) won't see it since it's unlinked from any directory. `df` still counts it as used space, since the actual disk blocks aren't freed yet.

```bash
lsof | grep deleted          # find processes holding open file handles to deleted files
lsof +L1                        # find files with a link count of 0 (i.e., deleted but still open) — genuinely useful command
```

---

## 2. Mounting & Unmounting: `mount`, `umount`

Linux doesn't use drive letters (like Windows' C:\, D:\) — instead, storage devices are **mounted** onto a directory (mount point) within the single unified filesystem tree.

```bash
mount                       # show ALL currently mounted filesystems
mount | grep sda                # filter to a specific device
mount /dev/sdb1 /mnt/data          # manually mount a device to a directory
umount /mnt/data                      # unmount (note: it's "umount", NOT "unmount" — classic typo trap)
umount -l /mnt/data                      # lazy unmount — detaches immediately, cleans up once it's no longer busy (useful when a normal umount fails with "device is busy")
```

**"Device is busy" scenario (commonly asked):** If `umount` fails saying the device is busy, it means some process still has an open file handle or working directory on that mount. Use `lsof /mnt/data` or `fuser -m /mnt/data` to find what's using it, close/kill that process, then retry.

---

## 3. Viewing Disk/Partition Layout: `lsblk`, `fdisk`, `blkid`

### `lsblk` — List Block Devices (tree view, read-only, safe to run anytime)

```bash
lsblk                     # shows all disks/partitions in a tree, with sizes and mount points
lsblk -f                     # also shows filesystem type and UUID — very useful when setting up fstab
```

### `fdisk` — Partition management tool (for MBR-style, and works with GPT too on modern versions)

```bash
sudo fdisk -l                   # list all disks and partitions with details
sudo fdisk /dev/sdb                # enter interactive mode to create/delete/modify partitions on /dev/sdb
# inside fdisk: n = new partition, d = delete, p = print table, w = write changes (ACTUALLY applies them), q = quit without saving
```

**Important safety note:** `fdisk` changes aren't applied until you explicitly press `w` (write) — this is intentional, giving you a chance to review before committing destructive partition changes.

### `parted` — More modern alternative to `fdisk`, better GPT support for very large disks (>2TB)

### `blkid` — Show UUIDs and filesystem types of block devices

```bash
sudo blkid                # lists device, UUID, and filesystem TYPE for each block device — needed for fstab entries
```

---

## 4. Filesystems — Types & Concepts

A filesystem defines HOW data is organized and stored on a partition — file structure, metadata, permissions, journaling.

| Filesystem | Common Use | Notes |
| --- | --- | --- |
| **ext4** | Default on most Linux distros (Ubuntu, Debian) | Mature, reliable, journaling filesystem — journaling means it logs changes before committing them, so it can recover cleanly after a crash/power loss without corruption |
| **XFS** | Default on RHEL/CentOS | Excellent for large files and high-performance workloads, handles large filesystems very well, commonly seen in enterprise/RHEL environments |
| **Btrfs** | Growing in adoption (some SUSE, Fedora setups) | Supports snapshots, built-in volume management, copy-on-write |
| **NTFS** | Windows filesystem | Linux can read/write it (via ntfs-3g driver) but it's not a native Linux filesystem |
| **FAT32/exFAT** | USB drives, cross-platform compatibility | Simple, universally compatible, but FAT32 has a 4GB single-file size limit |
| **NFS** | Network File System — not a disk format, but a protocol for sharing filesystems over a network | Common in enterprise environments for shared storage across multiple servers |

### Formatting a partition with a filesystem

```bash
sudo mkfs.ext4 /dev/sdb1        # format a partition with ext4
sudo mkfs.xfs /dev/sdb1            # format a partition with xfs
```

**Interview-relevant nuance on journaling:** A journaling filesystem (ext4, xfs) writes intended changes to a log ("journal") before actually committing them to the main filesystem structure. If the system crashes mid-write, on reboot the filesystem replays or discards incomplete journal entries, avoiding the kind of corruption that plagued older non-journaling filesystems (like ext2).

---

## 5. `/etc/fstab` — Persistent Mount Configuration

Manually running `mount` only lasts until reboot. `/etc/fstab` (File System Table) defines mounts that should happen **automatically at every boot**.

```bash
cat /etc/fstab
```

Example line, field by field:

```bash
UUID=1234-5678-ABCD  /data  ext4  defaults  0  2
```

| Field | Meaning |
| --- | --- |
| 1st | Device to mount — by UUID (preferred, stable) or device path like `/dev/sdb1` (can shift/change on reboot, less reliable) |
| 2nd | Mount point (the directory it attaches to) |
| 3rd | Filesystem type (ext4, xfs, nfs, etc.) |
| 4th | Mount options (`defaults`, or specifics like `ro` for read-only, `noexec` to block execution from that mount) |
| 5th | Dump flag — legacy backup utility flag, almost always `0` today |
| 6th | fsck order — filesystem check order at boot; `0` = don't check, `1` = check first (root filesystem), `2` = check after (other filesystems) |

**Why UUID is preferred over `/dev/sdX`:** Device names like `/dev/sdb` can shift between reboots depending on detection order, especially with multiple disks or after hardware changes — a drive that was `/dev/sdb` could become `/dev/sdc` after adding a new disk. UUIDs are unique and stable identifiers tied to the filesystem itself, so `fstab` entries using UUID keep working correctly regardless of device naming order.

```bash
sudo blkid                          # get the UUID to use in fstab
sudo nano /etc/fstab                   # edit fstab to add a persistent mount
sudo mount -a                             # test fstab WITHOUT rebooting — mounts everything listed that isn't already mounted, and importantly, will ERROR if there's a syntax mistake, letting you catch problems before an actual reboot
```

**Critical safety habit (real production practice):** ALWAYS test with `sudo mount -a` after editing `/etc/fstab`, before rebooting. A broken `fstab` entry (bad UUID, wrong filesystem type, typo) can cause the system to fail to boot properly or drop into an emergency/recovery shell — `mount -a` catches these mistakes safely while you can still fix them.

---

## 6. LVM — Logical Volume Manager

LVM adds a flexible abstraction layer between raw physical disks and the filesystems you actually use, allowing you to resize storage, add disks, and manage volumes without the rigid constraints of traditional fixed partitioning.

### The LVM hierarchy (must understand the layering)

```text
Physical Volume (PV)  →  Volume Group (VG)  →  Logical Volume (LV)  →  Filesystem (ext4/xfs) on top
```

1. **Physical Volume (PV)**: A raw disk or partition initialized for LVM use (e.g., `/dev/sdb`)
2. **Volume Group (VG)**: A pool combining one or more PVs into a single storage pool — this is where the real flexibility comes from, since a VG can span multiple physical disks
3. **Logical Volume (LV)**: A "virtual partition" carved out of a VG's pooled space — this is what you actually format with a filesystem and mount

### Why LVM matters practically

> "With traditional partitioning, if a partition runs out of space, resizing it is disruptive and constrained by physical disk layout. With LVM, if a Logical Volume runs low on space, I can extend it — even by adding an entirely new physical disk to the Volume Group first — and grow the filesystem, often without unmounting or rebooting. This flexibility is exactly why LVM is common in cloud/VM environments, where storage needs change over time and you want to resize without downtime."

### Basic LVM commands

```bash
# Create a Physical Volume
sudo pvcreate /dev/sdb

# Create a Volume Group from one or more PVs
sudo vgcreate myvg /dev/sdb

# Create a Logical Volume from the Volume Group
sudo lvcreate -L 10G -n mylv myvg

# Format the Logical Volume with a filesystem
sudo mkfs.ext4 /dev/myvg/mylv

# Mount it
sudo mount /dev/myvg/mylv /mnt/data

# View existing PVs, VGs, LVs
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay
pvs / vgs / lvs        # shorter summary versions of the above

# EXTEND storage — the key practical benefit
sudo vgextend myvg /dev/sdc            # add a new physical disk to an existing volume group
sudo lvextend -L +5G /dev/myvg/mylv       # grow the logical volume by 5GB
sudo resize2fs /dev/myvg/mylv                # grow the ext4 filesystem to fill the new LV space (use xfs_growfs instead for XFS)
```

**Important detail interviewers check for:** Extending an LV doesn't automatically resize the filesystem sitting on top of it — you must ALSO run `resize2fs` (ext4) or `xfs_growfs` (XFS) afterward to actually let the filesystem use the newly available space.

---

## Quick Reference Cheat Sheet

```bash
df -h                              # disk usage per filesystem
df -i                                 # inode usage
du -sh /path                             # total size of a directory
du -h --max-depth=1 /path                   # size of each immediate subfolder
mount / umount                                 # mount/unmount a device
lsblk -f                                          # list block devices + filesystem type + UUID
sudo fdisk -l                                        # list partitions in detail
sudo blkid                                              # UUIDs of devices
sudo mkfs.ext4 /dev/sdb1                                   # format with ext4
cat /etc/fstab                                                # persistent mount config
sudo mount -a                                                    # test fstab without reboot

# LVM
pvcreate / vgcreate / lvcreate           # create PV → VG → LV
pvs / vgs / lvs                             # quick summaries
vgextend / lvextend                            # grow storage
resize2fs / xfs_growfs                            # grow filesystem to match extended LV
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What's the difference between `df` and `du`?**
> A: `df` reports usage at the filesystem/partition level — how full each mounted disk is overall. `du` reports usage for specific files or directories you point it at, letting you drill down into what's actually consuming space inside a filesystem. I'd use `df -h` first to identify which partition is full, then `du` to find exactly what's taking up the space within it.

**Q2: `df -h` shows a disk at 100% full, but when you add up file sizes with `du`, it doesn't match. What's happening?**
> A: This typically happens when a file has been deleted but is still held open by a running process. The directory entry is gone (so `du`, which scans the visible directory tree, doesn't count it), but the actual disk blocks aren't freed until the process closes the file handle, so `df` still reports that space as used. I'd use `lsof +L1` or `lsof | grep deleted` to find the process holding the phantom space and restart/kill it to actually reclaim the disk space.

**Q3: Why is using UUID preferred over `/dev/sdX` device names in `/etc/fstab`?**
> A: Device names like `/dev/sdb` are assigned based on detection order at boot, which can shift if disks are added, removed, or detected in a different order — a drive that was `/dev/sdb` could become `/dev/sdc` after a hardware change. UUIDs are unique identifiers tied directly to the filesystem itself, so they remain stable and correct in `/etc/fstab` regardless of device naming changes.

**Q4: Why should you run `mount -a` after editing `/etc/fstab`, before rebooting?**
> A: `mount -a` attempts to mount everything listed in fstab that isn't already mounted, and critically, it will surface any syntax errors, invalid UUIDs, or misconfigurations immediately, while you're still able to fix them. If you skip this and reboot with a broken fstab entry, the system can fail to boot cleanly or drop into an emergency recovery shell, which is a much more disruptive way to discover the same mistake.

**Q5: Explain the LVM hierarchy: what's a Physical Volume, Volume Group, and Logical Volume?**
> A: A Physical Volume (PV) is a raw disk or partition initialized for LVM use. A Volume Group (VG) is a storage pool combining one or more PVs — this is where the flexibility comes from, since a VG can span multiple physical disks. A Logical Volume (LV) is a "virtual partition" carved out of that pooled VG space, which you then format with a filesystem and actually mount and use.

**Q6: Why is LVM particularly valuable in cloud/VM environments?**
> A: Because it decouples logical storage from rigid physical partition boundaries. If a Logical Volume runs low on space, you can extend it — even by adding an entirely new disk to the Volume Group first — and grow the underlying filesystem, often without downtime or unmounting. This matches how cloud storage needs actually evolve over time, unlike traditional fixed partitioning which is much more disruptive to resize.

**Q7: If you extend a Logical Volume with `lvextend`, does the filesystem automatically use the new space?**
> A: No — extending the LV only grows the underlying block device; the filesystem sitting on top of it still thinks it's the old size until you explicitly grow it too, using `resize2fs` for ext4 or `xfs_growfs` for XFS. Forgetting this step is a common mistake — the LV shows more space in `lvdisplay`, but `df -h` still shows the old, smaller size until the filesystem itself is resized.

**Q8: What's the difference between ext4 and XFS, and where would you typically see each?**
> A: Both are journaling filesystems, meaning they log intended changes before committing them, allowing clean recovery after a crash. ext4 is the default on most Debian-based distros (Ubuntu, Debian) and is a solid general-purpose choice. XFS is the default on RHEL/CentOS and tends to perform especially well with large files and high-throughput workloads, which is why it's common in enterprise storage-heavy environments.

**Q9: What does "journaling" mean in the context of filesystems, and why does it matter?**
> A: A journaling filesystem writes a record of intended changes to a separate log (the journal) before actually applying them to the main filesystem structure. If the system crashes or loses power mid-operation, on reboot the filesystem can replay or safely discard incomplete journal entries, avoiding the kind of severe corruption that could occur with older non-journaling filesystems where an interrupted write could leave the filesystem structure in an inconsistent state.

**Q10: Your `umount` command fails with "device is busy." How do you troubleshoot it?**
> A: This means some process still has an open file handle or is using that mount point as its working directory. I'd run `lsof /mount/point` or `fuser -m /mount/point` to identify which process is holding it open, then either close that process gracefully or kill it if appropriate, and retry the unmount. Alternatively, `umount -l` (lazy unmount) can detach it immediately and clean up once it's no longer actually busy, though that's more of a workaround than addressing the root cause.
