# Linux Permissions & Ownership

---

## The Permission Model

Every file/directory in Linux has three permission sets, for three categories of users:

| Category | Symbol | Meaning |
| --- | --- | --- |
| User (owner) | `u` | The person who owns the file |
| Group | `g` | Users belonging to the file's group |
| Others | `o` | Everyone else |

Each category can have three permissions:

| Permission | Symbol | On a File | On a Directory |
| --- | --- | --- | --- |
| Read | `r` = 4 | View file contents | List directory contents (`ls`) |
| Write | `w` = 2 | Modify/delete file contents | Create/delete/rename files inside it |
| Execute | `x` = 1 | Run file as a program/script | Enter directory (`cd`) / access files inside |

**Important frequently tested:** Read and Write on a *directory* don't matter if you don't also have Execute — without `x` you can't even `cd` into it or access anything inside, regardless of `r`/`w`.

### Viewing permissions

```bash
ls -l file.txt
-rwxr-xr--  1 arjun  devops  1024 Jul 27 10:00 file.txt
```

Breaking this down:

- 1st char: file type (`-` file, `d` directory, `l` symlink, `c`/`b` device files)
- Next 9 chars: `rwx r-x r--` → owner=rwx, group=r-x, others=r--
- Then: owner, group, size, modified date, filename

---

## Changing Permissions — `chmod`

### Symbolic mode

```bash
chmod u+x file.sh      # add execute for owner
chmod g-w file.sh      # remove write for group
chmod o=r file.sh      # set others to read-only
chmod a+x file.sh      # add execute for everyone (a = all)
chmod ug+rw file.sh    # add rw for user and group together
```

### Numeric (octal) mode — MUST memorize this cold

Each permission has a value: `r=4, w=2, x=1` (no permission = 0). Sum them per category.

| Number | Permission | Binary |
| --- | --- | --- |
| 7 | rwx | 111 |
| 6 | rw- | 110 |
| 5 | r-x | 101 |
| 4 | r-- | 100 |
| 0 | --- | 000 |

```bash
chmod 755 script.sh   # owner=rwx, group=r-x, others=r-x  (common for scripts)
chmod 644 file.txt    # owner=rw-, group=r--, others=r--  (common for regular files)
chmod 700 secret.sh   # owner=rwx, group=none, others=none (private script)
chmod -R 755 folder/  # recursive, apply to all files/subfolders
```

---

## Changing Ownership — `chown` / `chgrp`

```bash
chown arjun file.txt              # change owner to arjun
chown arjun:devops file.txt       # change owner AND group in one shot
chown -R arjun:devops /var/www/   # recursive
chgrp devops file.txt             # change only the group
```

Note: only **root** (or the file owner, with restrictions) can change ownership — a regular user cannot give away files they don't own.

---

## umask (default permission mask)

When a new file/directory is created, Linux starts with a base permission and subtracts the `umask`.

- Default base for files: `666` (rw-rw-rw-, no execute by default for safety)
- Default base for directories: `777`

```bash
umask         # shows current mask, e.g. 0022
```

With umask `022`: new file = `666 - 022 = 644`, new directory = `777 - 022 = 755`.

Set it temporarily: `umask 027` (common on more secure/shared servers, blocks "others" almost entirely).

---

## Special Permissions (High-value interview topic)

### SUID (Set User ID) — `4`

When set on an **executable**, the program runs with the **file owner's** privileges, not the user running it.

- Classic example: `/usr/bin/passwd` has SUID set to root, so any user can update `/etc/shadow` (which normally only root can write) via the `passwd` command.

```bash
chmod u+s file          # symbolic
chmod 4755 file         # numeric (leading 4)
ls -l → -rwsr-xr-x      # note the 's' instead of 'x' in owner slot
```

### SGID (Set Group ID) — `2`

- On an **executable**: runs with the group's privileges.
- On a **directory**: any new file/folder created inside inherits the *directory's group* instead of the creating user's default group. Very useful for shared team directories.

```bash
chmod g+s /shared/folder
chmod 2775 /shared/folder
ls -ld → drwxrwsr-x
```

### Sticky Bit — `1`

Mainly used on **directories**. When set, only the file's owner (or root) can delete/rename files inside that directory, even if others have write permission on the directory.

- Classic example: `/tmp` has the sticky bit set — everyone can write files there, but users can't delete each other's files.

```bash
chmod +t /tmp
chmod 1777 /tmp
ls -ld /tmp → drwxrwxrwt   # note the 't' at the end
```

**Combined numeric example:** `chmod 6755 file` = SUID(4) + SGID(2) + rwxr-xr-x.

---

## Access Control Lists (ACLs)

Standard permissions only allow ONE owner and ONE group. ACLs let you grant permissions to **multiple specific users/groups** on the same file.

```bash
setfacl -m u:john:rw file.txt     # give user john read+write, beyond normal owner/group/other
getfacl file.txt                  # view ACLs
setfacl -x u:john file.txt        # remove that rule
```

`ls -l` will show a `+` at the end of the permission string (e.g. `-rw-rw-r--+`) if ACLs are applied.

---

## `chattr` / `lsattr` — Filesystem-Level Attributes

Unlike `chmod` and `chown`, which control **who can do what**, `chattr` controls **filesystem attributes** that can override normal permission checks — even for the **root user**.

These attributes are enforced at the **filesystem/kernel level**, making them a deeper layer of protection than standard file permissions.

---

## Common Attributes

| Flag | Name | Effect |
| ------ | ------ | -------- |
| `+i` | Immutable | File cannot be modified, deleted, renamed, or linked to — not even by `root` — until the flag is removed. |
| `+a` | Append-only | File can only be opened in append mode for writing. Existing content cannot be modified or deleted, only new content can be added. |
| `+u` | Undeletable | When deleted, data is saved to allow undeletion (filesystem-dependent; not always supported). |
| `+s` | Secure deletion | When deleted, data blocks are overwritten immediately (filesystem-dependent; not always supported). |

---

## Commands

### Make a file immutable

```bash
chattr +i /etc/important.conf
```

### View file attributes

```bash
lsattr /etc/important.conf
```

Example output:

```text
----i---------
```

### Remove the immutable flag

```bash
chattr -i /etc/important.conf
```

> **Note:** You must remove the immutable flag before editing or deleting the file.

---

### Make a file append-only

Useful for log files.

```bash
chattr +a /var/log/audit.log
lsattr /var/log/audit.log
```

---

## Why This Matters (Key Insight)

Normal permissions (`chmod` and `chown`) only determine access based on **user identity**.

`chattr +i` works at a **lower level**, directly in the filesystem.

Even the **root user**, who normally bypasses permission checks, **cannot modify or delete an immutable file** until the immutable attribute is removed.

This makes `chattr` ideal for protecting critical system files.

---

## Real-World Use Cases

- Protect critical configuration files from accidental modification.
- Prevent scripts running as `root` from overwriting important files.
- Make audit and security logs append-only (`+a`) so attackers cannot erase or modify previous log entries.
- Add an extra layer of protection on hardened or security-focused Linux servers.

---

## Important

Suppose a file has the immutable attribute set:

```bash
chattr +i important_file.conf
```

Even the root user cannot delete it:

```bash
rm -f important_file.conf
```

Output:

```text
rm: cannot remove 'important_file.conf': Operation not permitted
```

This often surprises administrators who forget they previously set the `+i` attribute.

To delete or modify the file:

```bash
chattr -i important_file.conf
rm important_file.conf
```

---

### `chmod` vs `chattr`

| `chmod` | `chattr` |
| --------- | ---------- |
| Controls permissions based on user identity (owner, group, others). | Controls filesystem-level attributes. |
| Root can bypass most permission checks. | Even root must remove attributes like `+i` before modifying the file. |
| Used for regular permission management. | Used for protecting critical files and logs. |

---

### Interview-Ready Answer

> **"`chattr` operates below the normal permission model at the filesystem level. Unlike `chmod` and `chown`, which control access based on user identity, `chattr` can make a file immutable (`+i`) or append-only (`+a`). An immutable file cannot be modified, renamed, or deleted—even by the root user—until the attribute is removed."**

---

## Inodes, Hard Links & Soft (Symbolic) Links

### What is an Inode?

Every file and directory in Linux is represented by an **inode** (index node) — a data structure the filesystem uses to store all metadata about a file, EXCEPT its name and actual data content.

### What an inode stores

- File type (regular file, directory, symlink, etc.)
- Permissions (rwx for owner/group/others)
- Owner (UID) and group (GID)
- File size
- Timestamps: `atime` (last accessed), `mtime` (last modified), `ctime` (last metadata change — NOT "creation time", a common misconception)
- Number of hard links pointing to it
- Pointers to the actual data blocks on disk where the file's content is stored.

### What an inode does NOT store

- The **filename** itself. Filenames are stored separately, in the parent **directory's data**, as a mapping: `filename → inode number`.
This is the single most important concept to understand: **a filename is just a label pointing to an inode. The inode is the actual file.**

### Viewing inode information

```bash
ls -i file.txt              # shows the inode number of a file
ls -li                       # long listing WITH inode numbers
stat file.txt                 # detailed inode metadata: size, permissions, timestamps, inode number, link count
df -i                          # shows inode usage per filesystem (yes, you can run out of inodes even with free disk space!)
```

Example `stat` output:

```bash
File: file.txt
Size: 1024        Blocks: 8       IO Block: 4096   regular file
Inode: 1234567     Links: 1
Access: (0644/-rw-r--r--)  Uid: (1000/arjun)  Gid: (1000/arjun)
Access: 2026-07-27 10:00:00
Modify: 2026-07-27 09:45:00
Change: 2026-07-27 09:45:00
```

### Interesting/interview-relevant inode facts

- **You can run out of inodes even if disk space is free** — every filesystem has a FIXED number of inodes created at format time. If you have millions of tiny files, you can exhaust available inodes while `df -h` still shows plenty of free space. This is a real production issue interviewers like to test: "df shows space available but I can't create new files — why?" → answer: check `df -i`, you've likely exhausted inodes.
- Deleting a file doesn't necessarily free its inode immediately if another process still has it open (this connects to how log rotation and disk-space "phantom usage" issues happen — a deleted-but-open file still consumes space until the process closes it).

---

### Hard Links — Multiple names, SAME inode

A hard link creates another directory entry that points to the **exact same inode** as the original file. There is no "original" vs "copy" distinction after creation — they are equally valid names for the same underlying data.
 
```bash
ln original.txt hardlink.txt
ls -li original.txt hardlink.txt
# Both show the SAME inode number, confirming they're the same file
```

### Key characteristics:

- Both names share the same inode, same permissions, same data, same size — because it is the same file, just referenced by two names.
- The `Links` count in `stat` output tracks how many hard links point to that inode. A brand-new regular file starts at `Links: 1`.
- **Deleting one hard link does NOT delete the data** — the data is only actually freed from disk when the link count drops to **zero** (i.e., ALL hard links to that inode are removed).
- **Cannot span filesystems/partitions** — because inode numbers are only unique within a single filesystem; a hard link literally can't point to an inode number on a different filesystem.
- **Cannot hard-link a directory** — this is a deliberate restriction. Directories use hard links internally in a specific limited way (`.` and `..` entries), and allowing arbitrary hard links to directories could create loops that break tools that traverse the filesystem tree (like `find`, backup tools, `du`).

```bash
rm original.txt              # hardlink.txt still works perfectly — data isn't gone, link count just decreased
cat hardlink.txt              # file content still fully accessible
```

---

## 3. Soft Links / Symbolic Links — A separate file that POINTS to a path

A symbolic link (symlink) is a completely separate, small file with its **own inode**, whose content is simply the **path/string** to another file.

```bash
ln -s original.txt softlink.txt
ls -li original.txt softlink.txt
# DIFFERENT inode numbers — softlink.txt has its own inode, pointing to the path "original.txt"
ls -l softlink.txt
# lrwxrwxrwx 1 arjun arjun 12 Jul 27 10:00 softlink.txt -> original.txt
```

### Key characteristics

- Has its OWN inode, separate from the target file's inode.
- Its "content" is literally just the path string to the target.
- **Can span filesystems and partitions** — since it just stores a path, not an inode reference, it doesn't matter which filesystem the target lives on.
- **Can link to directories** (unlike hard links).
- **Breaks if the original is deleted or moved** — becomes a "dangling"/"broken" symlink, since the path it points to no longer resolves to anything. `ls -l` typically shows broken symlinks in a different color (often red) in most terminal configs.

```bash
rm original.txt
cat softlink.txt        # ERROR: No such file or directory — the symlink is now broken/dangling
ls -l softlink.txt        # still shows the symlink entry, but it's dead — points nowhere
```

---

### Hard Link vs Soft Link — Full Comparison Table

| Aspect | Hard Link | Soft Link (Symlink) |
| --- | --- | --- |
| Command | `ln target linkname` | `ln -s target linkname` |
| Inode | SAME inode as original | Own, SEPARATE inode |
| Points to | The actual data on disk directly | The path/name of the target file |
| Cross filesystem/partition | Not possible | Works fine |
| Can link to a directory | No (restricted) | Yes |
| If original is deleted | Link still works — data persists until ALL hard links are removed | Breaks — becomes a dangling/broken link |
| If original is renamed/moved (same fs) | Hard link unaffected (still points to same inode) | Breaks — the stored path no longer resolves |
| File size shown | Same as original (it's the same data) | Small — just the length of the stored path string |
| Permissions | Shares the exact same permissions as original (same inode = same metadata) | Symlink itself typically shows `lrwxrwxrwx`; actual access permission enforcement happens on the TARGET file |
| Identifying it | `ls -li` shows matching inode numbers | `ls -l` shows `->` arrow notation pointing to target |

---

### Practical / Real-World Use Cases

**Hard links** are used for:

- Space-efficient backups where you want multiple "copies" without actually duplicating data (some backup tools like `rsync --link-dest` use this technique for incremental backups)
- Keeping the same file accessible under two different names in the same location without duplicating storage

**Soft links** are used for:

- Version management: e.g., `/usr/bin/python -> python3.11`, allowing you to switch which version is "active" by just repointing the symlink
- Config file management: e.g., `/etc/nginx/sites-enabled/mysite -> ../sites-available/mysite` (classic Nginx pattern)
- Providing shortcuts to deeply nested paths without duplicating data
- Linking directories, which hard links can't do at all

---

## Quick Reference Commands

```bash
ls -i file.txt                  # show inode number
stat file.txt                    # full inode metadata
df -i                             # inode usage per filesystem (can run out even with free disk space!)
ln target.txt hardlink.txt          # create hard link
ln -s target.txt softlink.txt         # create soft/symbolic link
ls -li                                 # list with inode numbers, to compare hard-linked files
readlink softlink.txt                    # show what path a symlink points to
readlink -f softlink.txt                   # resolve the FULL absolute path, following the link
find . -inum 1234567                        # find all files sharing this specific inode number (find all hard links to a file)
```

---

## Quick Command Reference Sheet

```bash
ls -l                     # view permissions
chmod 755 file            # numeric change
chmod u+x file            # symbolic change
chown user:group file     # change owner+group
chgrp group file          # change group only
umask                     # view default mask
umask 022                 # set default mask
setfacl / getfacl         # ACL management
ln vs ln -s               # hard vs soft link
find / -perm -4000         # find all SUID files on system (security audit use-case)
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What do the numbers in `chmod 755` actually mean?**
> A: Each digit represents permission for owner, group, and others respectively, calculated as r=4, w=2, x=1 summed together. 755 = owner:7(rwx), group:5(r-x), others:5(r-x). This is a common permission for scripts and executables — owner can fully control it, everyone else can only read and execute.

**Q2: What's the difference between `chmod 777` and `chmod 755`, and why is `777` considered risky?**
> A: 777 gives read/write/execute to everyone including other users on the system — anyone can modify or delete the file. 755 restricts write access to only the owner, which is safer. In interviews, mentioning that `777` is a red flag in security audits shows good practice awareness.

**Q3: Explain SUID with a real-world example.**
> A: SUID lets a program run with the file owner's permissions instead of the executing user's. The classic real-world example is `/usr/bin/passwd` — it's owned by root and has SUID set, so when a regular user runs `passwd` to change their own password, the program temporarily runs as root to write to `/etc/shadow`, which normal users can't edit directly.

**Q4: What is the sticky bit and where is it used by default on most Linux systems?**
> A: The sticky bit, when set on a directory, ensures only the file's owner (or root) can delete or rename files within it, even if others have write access to the directory. It's used by default on `/tmp`, since many users/processes write temp files there but shouldn't be able to delete each other's files.

**Q5: If a directory has permission `d rwx r-x r--`, can a user in the "others" category list the files in it?**
> A: Yes — read (`r`) on a directory allows listing contents (`ls`). But they would NOT be able to `cd` into it or access files inside, since execute (`x`) is required for that, and "others" only has `r--` (read only, no execute).

**Q6: What's the difference between hard links and soft links?**
> A: A hard link points to the same inode as the original file — it's essentially another name for the same data, so deleting the original doesn't break it. A soft link is a pointer to the file's path/name — if the original is deleted or moved, the soft link becomes a "dangling link" and breaks. Soft links can also cross filesystems and link to directories; hard links cannot.

**Q7: How would you find all files with SUID set on a system, and why would you do this?**
> A: `find / -perm -4000 -type f 2>/dev/null`. This is a common security audit step — SUID binaries are a common privilege escalation vector if misconfigured or if a vulnerable SUID binary exists, so security teams periodically audit for unexpected SUID files.

**Q8: What does `umask 022` actually do to newly created files and directories?**
> A: It subtracts from the default base permissions. Files default to 666, so `666 - 022 = 644` (rw-r--r--). Directories default to 777, so `777 - 022 = 755` (rwxr-xr-x). It ensures new files/folders aren't world-writable by default.

**Q9: You `chmod 700` a script but it still won't run when another user calls it. Why?**
> A: 700 means only the owner has rwx; group and others have zero permissions — so any other user is correctly denied. This is expected behavior, not a bug. If it also fails for the owner, check the shebang line (`#!/bin/bash`) and that the execute bit is actually applied (`ls -l` to confirm), or check if the filesystem is mounted with `noexec`.

**Q10: What's the practical use case for SGID on a directory (as opposed to a file)?**
> A: On a shared team directory (e.g., `/shared/project`), setting SGID ensures every new file or subfolder created inside automatically inherits the directory's group ownership, rather than defaulting to the creating user's primary group. This keeps shared folders consistently accessible to the whole team without everyone manually running `chgrp` after creating files.

**Q11: Difference between `chmod` and `chown`, and who is allowed to run each?**
> A: `chmod` changes what actions (read/write/execute) are allowed for owner/group/others. `chown` changes WHO owns the file (user and/or group). Any file owner can `chmod` their own file, but only `root` (or the owner in some restricted cases, depending on system config) can `chown` to change the owner to someone else — a regular user can't "give away" a file's ownership arbitrarily.

**Q12: What does the `+` sign at the end of `ls -l` output (e.g., `-rw-rw-r--+`) indicate?**
> A: It indicates the file has an ACL (Access Control List) applied — meaning permissions beyond the standard owner/group/other model have been set using `setfacl`, granting specific extra permissions to additional users or groups.

**Q13: What is an inode, and what does it NOT contain?**
> A: An inode is the data structure that stores all metadata about a file — permissions, owner, size, timestamps, link count, and pointers to the actual data blocks on disk. It does NOT store the filename; the filename-to-inode mapping is kept in the parent directory's own data instead. So fundamentally, a filename is just a label pointing at an inode.

**Q14: Can a system run out of disk space while `df -h` still shows free space available? Explain.**
> A: Yes — this happens when you run out of inodes rather than actual disk blocks. Every filesystem has a fixed number of inodes allocated at creation time, so if you have millions of very small files, you can exhaust all available inodes while plenty of raw disk space technically remains free. You'd diagnose this with `df -i` instead of `df -h`, and it's a real production scenario, often caused by something like an application generating huge numbers of tiny temp/cache files.

**Q15: What's the fundamental difference between a hard link and a soft link, at the inode level?**
> A: A hard link is another directory entry pointing to the exact SAME inode as the original — they're indistinguishable copies of the same underlying file. A soft link is a completely separate file with its OWN inode, whose content is just the text path to the target file. This is why deleting the original breaks a soft link (the path no longer resolves) but doesn't break a hard link (the inode and data are still there, referenced by the remaining link).

**Q16: If you delete the original file that a hard link points to, does the data get deleted?**
> A: No — data is only actually freed from disk when the inode's link count drops to zero, meaning ALL hard links (including the "original") pointing to it have been removed. As long as at least one hard link remains, the data and inode persist and remain fully accessible through that remaining link.

**Q17: Why can't you create a hard link across two different partitions/filesystems, but you can with a soft link?**
> A: Inode numbers are only unique within a single filesystem — the same inode number could exist independently on two different filesystems referring to two completely different files. A hard link needs to directly reference an inode, so it can't cross that boundary. A soft link just stores a path string, which works regardless of which filesystem the target lives on, so there's no such restriction.

**Q18: Why are hard links to directories not allowed?**
> A: It's a deliberate restriction to prevent circular references/loops in the filesystem tree. Directories already use a limited, controlled form of hard linking internally (`.` for itself and `..` for parent), and allowing arbitrary hard links to directories could create infinite loops that would break tools relying on tree traversal, like `find`, `du`, or backup utilities.

**Q19: How would you check if two filenames actually point to the same underlying file?**
> A: `ls -li` on both filenames — if they show the same inode number, they're hard links to the same file. Alternatively, `stat` on both files would show identical inode numbers and a link count greater than 1.

**Q20: What happens to a symlink if you rename or move the file it points to?**
> A: The symlink breaks and becomes "dangling" — since it only stores the original path as a string, and that path no longer resolves to anything once the target is moved or renamed. The symlink file itself still exists as an entry, but accessing it returns a "No such file or directory" error.

**Q21: Give a real production example of where symbolic links are commonly used.**
> A: A classic example is Nginx's `sites-enabled` directory, where config files are typically symlinked from `sites-available` — e.g. `/etc/nginx/sites-enabled/mysite -> ../sites-available/mysite`. This lets you "enable" or "disable" a site just by adding/removing the symlink, without duplicating or losing the actual config file. Another common one is version switching, like `/usr/bin/python` symlinked to a specific Python version, so updating which version is "active" just means repointing one symlink.

**Q22: What does the `Links` field in `stat` output actually represent, and why does a new file start with `Links: 1`?**
> A: It represents the number of hard links currently pointing to that inode. A brand-new regular file starts at `Links: 1` because the file's own name in its directory already counts as the first hard link to that inode — every file is technically already "hard linked" once, even before you manually create any additional links.