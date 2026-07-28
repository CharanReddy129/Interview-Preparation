# Interview-Preparation

Personal notes for DevOps / Cloud Engineer interview preparation.

---

## 📁 Linux

| # | File | Covers |
| --- | --- | --- |
| 1 | [Basics_architecture.md](./Linux/Basics_architecture.md) | What Linux is and why it's used; kernel vs shell vs userspace; terminal vs shell; Debian vs RHEL vs Amazon Linux vs Alpine distro differences; the Linux Filesystem Hierarchy Standard (`/etc`, `/var`, `/proc`, `/sys`, `/bin` vs `/sbin`, etc.) |
| 2 | [Basic_commands.md](./Linux/Basic_commands.md) | Navigation (`pwd`, `cd`, `ls`), file operations (`cp`, `mv`, `rm`, `mkdir`, `touch`), wildcards, absolute vs relative paths, `find` vs `locate`, viewing files (`cat`, `less`, `head`, `tail -f`), plus utility commands — `echo`, `man`, `date`, `sort`, `split`, `shuf`, `diff`, `wc`, `cmp`, `which`, `bc`, and more |
| 3 | [files_links.md](./Linux/files_links.md) | File permissions (`rwx`, numeric vs symbolic `chmod`), `chown`/`chgrp`, special permissions (SUID, SGID, sticky bit), `umask`, ACLs (`setfacl`/`getfacl`), `chattr`/`lsattr`, and inodes/hard links/soft links in depth |
| 4 | [Users_Groups.md](./Linux/Users_Groups.md) | User types and UID ranges, `/etc/passwd` vs `/etc/shadow` vs `/etc/group`, primary vs secondary groups, `useradd`/`usermod`/`userdel`, `su` vs `sudo`, `/etc/sudoers` and `visudo` |
| 5 | [Process_management.md](./Linux/Process_management.md) | `ps`/`top`/`htop`, process states, foreground vs background jobs (`&`, `jobs`, `fg`, `bg`, `nohup`), zombie vs orphan processes, signals (`SIGTERM`/`SIGKILL`/`SIGHUP`), `kill`/`killall`/`pkill`, `nice`/`renice` |
| 6 | [Package_manager.md](./Linux/Package_manager.md) | Debian (`apt`/`dpkg`), RHEL (`yum`/`dnf`/`rpm`), Alpine (`apk`), and compiling software from source (`./configure && make && make install`) |
| 7 | [Disk_management.md](./Linux/Disk_management.md) | `df` vs `du`, `mount`/`umount`, `lsblk`/`fdisk`/`blkid`, filesystem types (ext4, XFS, etc.), `/etc/fstab`, and LVM (Physical/Volume/Logical volumes) |
| 8 | [System_logs_monitoring.md](./Linux/System_logs_monitoring.md) | `/var/log` structure, `journalctl`, `dmesg`, resource monitoring (`uptime`, `free -h`, swap deep-dive, `vmstat`, `iostat`), the full boot sequence (BIOS/UEFI → GRUB → kernel → systemd), and `systemctl`/units/targets |
| 9 | [Networking_basics.md](./Linux/Networking_basics.md) | `ip`/`ss` (modern) vs `ifconfig`/`netstat` (legacy), `ping`/`traceroute`, `curl`/`wget`, `dig`/`nslookup`, `/etc/hosts` & `/etc/resolv.conf`, TCP vs UDP, common ports, and firewalls (`iptables`/`ufw`/`firewalld`) |
| 10 | [SSH_security.md](./Linux/SSH_security.md) | SSH key pairs, `ssh-keygen`/`ssh-copy-id`, `~/.ssh/config`, `known_hosts`, SCP/SFTP/rsync, file permission hardening, SELinux/AppArmor concepts, and server hardening basics (disabling root login, key-only auth) |
| 11 | [Others.md](./Linux/Others.md) | Namespaces & cgroups (the Linux foundations of containers), `vim`/`nano` basics, `sysctl` kernel tuning, time/timezone management (`timedatectl`, NTP/chrony), and debugging tools (`strace`, `tcpdump`) |

---

## 📁 Shell Scripting

| # | File | Covers |
| --- | --- | --- |
| 1 | [Shell_Scripting.md](./Shell_Scripting/Shell_Scripting.md) | Shebang/permissions, variables & quoting, special variables (`$1`, `$@`, `$?`), reading input, conditionals (`if`/`case`, numeric vs string operators), loops (`for`/`while`/`until`), functions, exit codes & `set -euo pipefail`, arrays, and real script examples (disk alert, backup script, user creation, log monitor) |
