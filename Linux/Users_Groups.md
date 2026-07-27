# Users & Groups

## 1. Why Users & Groups Exist

Linux is a **multiuser** system by design — it needs a way to identify who owns what, who can access what, and to isolate users from each other for security. Every process and file is tied to a user (UID) and group (GID), which is the foundation permissions (chmod/chown from earlier topics) actually enforces.

---

## 2. Types of Users

| Type | UID Range (typical) | Purpose |
| --- | --- | --- |
| **Root (superuser)** | 0 | Full unrestricted access to everything on the system |
| **System users** | 1–999 (varies by distro) | Non-human accounts created for services (e.g., `www-data`, `mysql`, `nginx`) — used to run daemons with limited privileges instead of root, for security |
| **Regular/Normal users** | 1000+ (varies by distro) | Actual human user accounts |

**Why system users matter:** Running a web server as `www-data` instead of `root` means that if the web server is compromised, the attacker only gets `www-data`'s limited privileges, not full root access. This is a core security principle called **least privilege**.

---

## 3. Key Files Behind Users & Groups

### `/etc/passwd` — User account information (NOT actually passwords despite the name; that's historical)

```bash
cat /etc/passwd
# arjun:x:1000:1000:Arjun K:/home/arjun:/bin/bash
```

Format (colon-separated, 7 fields):

```bash
username : password-placeholder : UID : GID : comment/full-name : home-directory : login-shell
```

- 2nd field is just `x` — actual password hash is stored elsewhere in `/etc/shadow` for security (so `/etc/passwd` can remain world-readable, since it's needed for basic ID lookups, without exposing password hashes)
- Last field, the login shell — if set to `/sbin/nologin` or `/bin/false`, that account **cannot log in interactively** (common for system/service accounts)

### `/etc/shadow` — Actual password hashes (root-only readable, for security)

```bash
sudo cat /etc/shadow
# arjun:$6$abc123...:19500:0:99999:7:::
```

Format:

```bash
username : hashed-password : last-changed : min-days : max-days : warn-days : inactive : expire : reserved
```

- Only root can read this file (`chmod 600` or `640` typically) — this is exactly why `passwd` needs SUID (covered in Topic 3!) to let regular users update their own password despite not having direct write access to `/etc/shadow`.

### `/etc/group` — Group information

```bash
cat /etc/group
# devops:x:1001:arjun,priya
```

Format:

```bash
groupname : password-placeholder : GID : comma-separated-list-of-member-usernames
```

### `/etc/gshadow` — Group password info (rarely used in practice, but exists)

---

## 4. Primary Group vs Secondary (Supplementary) Groups

- **Primary group**: The default group assigned to a user, recorded in `/etc/passwd`. New files a user creates default to this group (unless SGID is involved, from Topic 3).
- **Secondary/supplementary groups**: Additional groups a user belongs to, granting extra permissions (e.g., being added to a `docker` group to run Docker commands without `sudo`, or a `sudo`/`wheel` group for admin privileges).

```bash
groups arjun               # shows all groups arjun belongs to (primary + secondary)
id arjun                     # shows UID, GID, and all group memberships
```

---

## 5. User Management Commands

### `useradd` — Create a new user

```bash
sudo useradd arjun                          # basic user creation (may not create home dir by default, distro-dependent)
sudo useradd -m arjun                        # -m creates home directory (/home/arjun)
sudo useradd -m -s /bin/bash arjun             # specify shell explicitly
sudo useradd -m -G devops,sudo arjun             # add to secondary groups at creation time
sudo useradd -m -u 1050 arjun                      # specify a custom UID
```

### `passwd` — Set/change password

```bash
sudo passwd arjun          # set password for another user (as root)
passwd                       # change your OWN password
passwd -l arjun                # lock a user account (disable login without deleting it)
passwd -u arjun                 # unlock account
passwd -e arjun                   # force password expiry — user must change password at next login
```

### `usermod` — Modify an existing user

```bash
sudo usermod -aG docker arjun       # ADD arjun to the docker group (⚠️ -a is CRITICAL — append, don't replace)
sudo usermod -s /bin/zsh arjun        # change login shell
sudo usermod -L arjun                  # lock account (similar to passwd -l)
sudo usermod -d /new/home -m arjun       # change home directory (and move contents with -m)
sudo usermod -g newgroup arjun             # change PRIMARY group
```

**Critical interview:** `usermod -G docker arjun` (WITHOUT `-a`) **replaces ALL** of a user's secondary groups with just `docker`, potentially removing them from other important groups accidentally. Always use `-aG` (append) unless you specifically intend to wipe all existing group memberships. This is a genuinely common real-world mistake that causes access issues.

### `userdel` — Delete a user

```bash
sudo userdel arjun               # delete user, but leaves home directory behind
sudo userdel -r arjun              # -r also removes home directory and mail spool
```

---

## 6. Group Management Commands

### `groupadd` — Create a new group

```bash
sudo groupadd devops
sudo groupadd -g 1050 devops        # specify custom GID
```

### `groupdel` — Delete a group

```bash
sudo groupdel devops     # fails if it's still someone's PRIMARY group
```

### `gpasswd` — Manage group membership (alternative to usermod)

```bash
sudo gpasswd -a arjun devops       # add arjun to devops group
sudo gpasswd -d arjun devops        # remove arjun from devops group
```

---

## 7. Switching Users & Privilege Escalation

### `su` — Switch user

```bash
su arjun               # switch to arjun (asks for arjun's password)
su - arjun               # switch to arjun AND load their full environment/shell profile (recommended over plain su)
su                          # no argument — switches to root (asks for ROOT's password)
```

### `sudo` — Run a single command with elevated privileges

```bash
sudo apt update              # run one command as root, asks for YOUR OWN password (not root's)
sudo -i                        # get a root shell/login session (similar effect to 'su -')
sudo -u arjun whoami              # run a command as a DIFFERENT specific user, not necessarily root
sudo -l                            # list what commands you're allowed to run with sudo
```

### `su` vs `sudo` (frequently asked distinction)

| | `su` | `sudo` |
| --- | --- | --- |
| Password required | Target user's password (e.g., root's password) | YOUR OWN password |
| Scope | Switches your whole session to another user | Typically runs a single command with elevated privileges, then returns |
| Logging | Less granular audit trail | Every sudo command is logged (`/var/log/auth.log` or `/var/log/secure`), including who ran what |
| Security best practice | Less preferred — requires sharing/knowing root's actual password | Preferred — no need to share root's password; access is controlled via `/etc/sudoers`, individually per user |

### `/etc/sudoers` — Controls WHO can use sudo and for WHAT

```bash
sudo visudo                # ALWAYS use visudo to edit this file — it validates syntax before saving, preventing a broken sudoers file that could lock everyone out
```

Example line granting a user full sudo access:

```bash
arjun ALL=(ALL:ALL) ALL
```

Example granting passwordless sudo for a SPECIFIC command only (common in automation/CI):

```bash
deployuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart myapp
```

---

## 8. Useful Identity/Info Commands

```bash
whoami                # current username
id                    # UID, GID, and all group memberships
id -u                 # just the UID number
id -Gn                # just the group NAMES you belong to
groups                # groups current user belongs to
who                   # who is currently logged in
w                     # who is logged in + what they're doing (more detail than 'who')
last                  # login history
finger arjun          # detailed user info (not installed by default on many modern systems)
```

---

## Quick Reference Cheat Sheet

```bash
cat /etc/passwd                     # user account info
sudo cat /etc/shadow                # password hashes (root only)
cat /etc/group                      # group info
useradd -m -s /bin/bash arjun       # create user with home dir + shell
passwd arjun                        # set password
usermod -aG docker arjun            # ADD to group (append, don't overwrite!)
userdel -r arjun                    # delete user + home dir
groupadd devops                     # create group
gpasswd -a arjun devops             # add user to group
su - arjun                          # switch user (full environment)
sudo apt update                     # run single command as root
sudo visudo                         # safely edit sudoers file
id arjun                            # full identity info
groups arjun                        # group memberships
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What's actually stored in `/etc/passwd`, and why isn't the real password there despite the name?**
> A: `/etc/passwd` stores user account metadata — username, UID, GID, home directory, and login shell — in colon-separated fields. The password field is just a placeholder `x`; the actual hashed password lives in `/etc/shadow`, which is readable only by root. This split exists because `/etc/passwd` needs to remain world-readable for basic system lookups (like resolving UID to username), while password hashes need much stricter protection.

**Q2: Why does `passwd` need the SUID bit set, connecting back to permissions?**
> A: `/etc/shadow` is only writable by root, but regular users need to be able to change their own password. Since `passwd` has SUID set (owned by root), it temporarily runs with root's privileges when executed by any user, allowing it to write to `/etc/shadow` on the user's behalf, even though the user themselves has no direct write access to that file.

**Q3: What's the critical difference between `usermod -G` and `usermod -aG`, and why does this matter practically?**
> A: `usermod -G docker arjun` REPLACES all of a user's existing secondary group memberships with just the groups listed — potentially removing them from other groups they needed access through. `usermod -aG docker arjun` APPENDS the new group while keeping existing memberships intact. Forgetting `-a` is a genuinely common real-world mistake that silently breaks a user's access to other resources they were previously part of.

**Q4: Explain the core difference between `su` and `sudo`.**
> A: `su` switches your entire session to another user and requires that TARGET user's password — commonly root's password, meaning it has to be shared or known. `sudo` runs a single command with elevated privileges using YOUR OWN password, with access controlled granularly per-user via `/etc/sudoers`, and every command is logged for auditing. `sudo` is generally the preferred, more secure approach in modern systems since it avoids sharing the root password and provides better accountability.

**Q5: Why should you always use `visudo` instead of directly editing `/etc/sudoers` with a normal text editor?**
> A: `visudo` validates the syntax of the sudoers file BEFORE saving changes. If you edit `/etc/sudoers` directly and introduce a syntax error, you could lock every user (including yourself) out of using sudo, since a malformed sudoers file breaks privilege escalation system-wide. `visudo` prevents saving broken syntax in the first place.

**Q6: What's the difference between a primary group and a secondary/supplementary group?**
> A: The primary group is the default group recorded in `/etc/passwd` for a user — new files they create are owned by this group by default (unless SGID overrides it, as we covered with directories). Secondary groups are additional group memberships that grant extra permissions, like being added to a `docker` or `sudo` group, without changing the user's default/primary group.

**Q7: Why would you run a web server process as a dedicated system user like `www-data` instead of root?**
> A: This follows the principle of least privilege — if the web server has a security vulnerability and gets compromised, the attacker only gains the limited privileges of `www-data`, not full root access to the entire system. System users like `www-data` typically have restricted permissions and often can't log in interactively at all (shell set to `/sbin/nologin`), further limiting risk.

**Q8: How would you completely remove a user, including their home directory?**
> A: `sudo userdel -r username` — the `-r` flag removes the user's home directory and mail spool along with the account itself. Without `-r`, `userdel` only removes the account entry but leaves the home directory and its files behind.

**Q9: A user says they were just added to the `docker` group but Docker commands still ask for `sudo`/still get "permission denied." What would you check?**
> A: Group membership changes typically require the user to log out and log back in (or start a new shell session) to take effect, since the current shell session's group membership was established at login time. I'd verify with `groups username` that the group was actually applied, and remind them to re-login rather than assuming the group assignment itself failed.

**Q10: What does `sudo -l` show, and why is it useful?**
> A: It lists the specific sudo permissions the current user has been granted — which commands they're allowed to run with sudo, and whether a password is required. It's useful for verifying exactly what privileges have been configured for a user without needing to read through `/etc/sudoers` directly, especially helpful when troubleshooting "permission denied" issues on specific commands.
