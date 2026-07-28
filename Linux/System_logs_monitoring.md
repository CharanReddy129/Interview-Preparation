# System Monitoring & Logs

## 1. `/var/log/` — Where Linux Logs Live

Almost everything of significance on a Linux system gets logged somewhere under `/var/log/`. Knowing the key files is genuinely useful troubleshooting knowledge, not just interview trivia.

| Log File | Contains |
| --- | --- |
| `/var/log/syslog` (Debian/Ubuntu) or `/var/log/messages` (RHEL) | General system activity log — the "catch-all" for most system events |
| `/var/log/auth.log` (Debian) or `/var/log/secure` (RHEL) | Authentication events — logins, sudo usage, SSH attempts, failed passwords |
| `/var/log/kern.log` | Kernel-specific messages |
| `/var/log/dmesg` | Boot-time and kernel ring buffer messages (also viewable live via the `dmesg` command) |
| `/var/log/boot.log` | Messages generated during system boot |
| `/var/log/cron` | Cron job execution logs |
| `/var/log/apt/` (Debian) | Package installation/update history |
| `/var/log/nginx/`, `/var/log/mysql/`, etc. | Application-specific logs, usually in their own subdirectory |
| `/var/log/wtmp`, `/var/log/btmp` | Binary logs of login history (successful/failed) — read via `last`/`lastb`, not directly readable with `cat` |

```bash
tail -f /var/log/syslog             # live-watch general system activity
tail -f /var/log/auth.log              # live-watch login/auth attempts — genuinely useful for spotting brute-force SSH attempts
grep "Failed password" /var/log/auth.log     # find failed SSH login attempts, a common real security-triage command
```

### Log Rotation — `logrotate`

Log files would grow forever without management. `logrotate` automatically rotates (archives/compresses/deletes old) logs based on rules in `/etc/logrotate.conf` and `/etc/logrotate.d/`.

```bash
cat /etc/logrotate.conf                  # global rotation defaults
ls /etc/logrotate.d/                        # per-application specific rotation rules
```

This is WHY you'll sometimes see `syslog`, `syslog.1`, `syslog.2.gz` etc. in `/var/log` — logrotate rotates the active log periodically and compresses/ages out the older ones.

---

## 2. `journalctl` — The Modern systemd Logging Tool (Critical!)

Since most modern distros use `systemd`, logs are also captured by **systemd-journald** into a structured **binary** journal — not plain text files. `journalctl` is how you read it.

```bash
journalctl                        # show ALL journal logs (oldest first) — usually pipe to less/tail
journalctl -e                        # jump to the END (most recent logs) — very commonly used
journalctl -f                           # FOLLOW mode — live tail, like tail -f but for the journal
journalctl -u nginx                        # logs for a SPECIFIC systemd service/unit — extremely commonly used
journalctl -u nginx -f                        # live-follow logs for a specific service
journalctl -u nginx --since "1 hour ago"         # time-filtered logs for a service
journalctl -p err                                   # filter by PRIORITY level (err, warning, crit, etc.)
journalctl --since "2026-07-27" --until "2026-07-28"   # date range filtering
journalctl -b                                              # logs since the current BOOT only
journalctl -b -1                                              # logs from the PREVIOUS boot (very useful after a crash/reboot to see what happened before it went down)
journalctl -k                                                    # kernel messages only (similar to dmesg)
journalctl --disk-usage                                             # how much disk space the journal itself is consuming
sudo journalctl --vacuum-time=7d                                       # delete journal entries older than 7 days (space management)
sudo journalctl --vacuum-size=500M                                        # shrink journal to under 500MB
```

**Why `journalctl -u <service>` is one of the single most important real-world commands:** When a service fails to start (`systemctl start myapp` fails), `journalctl -u myapp -e` or `journalctl -u myapp --since "5 min ago"` is almost always the very next command you run to actually see WHY it failed — this is the standard troubleshooting reflex for any systemd-managed service.

---

## 3. `dmesg` — Kernel Ring Buffer

```bash
dmesg                     # print kernel messages (boot messages, hardware detection, driver issues)
dmesg -T                     # human-readable TIMESTAMPS instead of raw seconds-since-boot (very commonly needed flag)
dmesg | grep -i error           # filter for error-related kernel messages
dmesg -w                           # watch/follow live, like tail -f but for kernel messages — useful for spotting hardware issues (e.g., disk errors, USB device events) as they happen
```

**Common real use case:** diagnosing hardware issues, disk errors, out-of-memory (OOM) killer events — `dmesg | grep -i "out of memory"` is a classic way to confirm the kernel's OOM killer terminated a process due to memory pressure.

---

## 4. System Resource Monitoring: `uptime`, `free -h`, `vmstat`, `iostat`

### `uptime` — How long the system has been running + load average

```bash
uptime
# 14:32:05 up 10 days, 3:15, 2 users, load average: 0.52, 0.58, 0.61
```

The three numbers are **load average over 1, 5, and 15 minutes** — representing the average number of processes wanting CPU time (running + waiting to run). 

**Interpreting load average:** A load average roughly equal to your number of CPU cores means the system is fully utilized but keeping up. A load average significantly HIGHER than your core count means processes are queuing/waiting — the system is overloaded. E.g., a load average of 8.0 on a 4-core machine means, on average, 4 processes are actively waiting their turn for CPU time.

```bash
nproc                # shows number of CPU cores, to compare against load average
```

### `free -h` — Memory (RAM) and Swap Usage

```bash
free -h
#               total        used        free      shared  buff/cache   available
# Mem:           7.8G        2.1G        3.2G        0.1G        2.5G        5.4G
# Swap:          2.0G        0.3G        1.7G
```

**Column breakdown (commonly asked):**

- `total` — total installed RAM
- `used` — currently in active use
- `free` — completely unused RAM
- `shared` — memory used by tmpfs and shared between processes
- `buff/cache` — memory used for disk caching and buffers — **this is NOT "wasted" memory**; Linux aggressively uses spare RAM to cache disk data for speed, and this memory is instantly reclaimed if an application needs it
- `available` — the REAL number to look at — estimates how much memory is actually available for new applications, accounting for reclaimable cache. **This is the number that actually matters, not `free`**, since a low "free" value with high "available" is completely normal and healthy on Linux.

**Common interview trap:** A new engineer sees `free` is low (e.g., only 200MB "free") and panics that the system is nearly out of memory — but `available` might show 5GB, because most of the "used" memory is actually reclaimable buffer/cache, not truly locked up. This confusion is extremely common and interviewers specifically test for whether you understand this Linux memory management nuance.

### Swap — Deep Dive

Swap is disk space used as an overflow/backup for RAM when physical memory is exhausted — the kernel moves less-actively-used memory pages ("swapped out") from RAM to disk to free up physical RAM for more active processes.

**Why swap matters:**

- Acts as a safety net preventing the system from immediately crashing/OOM-killing processes the instant RAM fills up
- Swap is MUCH slower than RAM (disk speed vs. RAM speed — orders of magnitude difference), so heavy swap usage is a sign of memory pressure and will cause noticeable performance degradation ("thrashing" if it's happening constantly)
- A small amount of swap usage isn't necessarily alarming — the kernel may swap out genuinely idle/inactive memory pages even when plenty of RAM is free, simply to keep more RAM available for active disk caching. **Consistently HIGH or constantly GROWING swap usage is the actual red flag**, not any swap usage at all.

```bash
free -h                       # shows swap total/used/free, as above
swapon --show                    # show active swap devices/files and their usage
cat /proc/swappiness                # or: sysctl vm.swappiness — a kernel tunable (0-100) controlling how AGGRESSIVELY the kernel swaps; lower = prefers keeping things in RAM longer, higher = swaps more readily. Default is often 60.
sudo swapoff -a && sudo swapon -a       # disable then re-enable all swap (occasionally used to forcibly clear swap usage, moving swapped pages back to RAM if there's room)
```

**Cloud/DevOps-relevant swap fact:** Many production database servers explicitly SET `vm.swappiness` very low (even 1 or 0) because swapping database memory to disk causes severe unpredictable latency spikes — databases generally prefer to rely on having enough RAM and avoid swap entirely rather than degrade gracefully into it.

### `vmstat` — Virtual Memory Statistics (broader system performance snapshot)

```bash
vmstat 2 5           # report every 2 seconds, 5 times total
```

Shows columns for processes (r = running, b = blocked/waiting), memory, swap activity (si/so = swap in/out — genuinely useful to spot active swapping happening RIGHT NOW), IO, and CPU (us/sy/id/wa — user/system/idle/iowait time).
**`wa` (I/O wait) is particularly worth knowing:** high `wa` percentage means the CPU is frequently sitting idle waiting on disk I/O to complete — a strong signal of a disk-bound bottleneck rather than a CPU-bound one.

### `iostat` — I/O Statistics per Device (may need `sysstat` package installed)

```bash
iostat -x 2               # extended stats, refresh every 2 seconds
```

Shows per-disk read/write throughput, IOPS, and `%util` (how busy the device is) — useful for pinpointing whether a specific disk is the actual bottleneck.

---

# Boot Process & systemd

## 5. The Boot Sequence (BIOS/UEFI → GRUB → Kernel → systemd)

```text
Power On
   ↓
BIOS/UEFI  (firmware-level hardware initialization + POST)
   ↓
Bootloader (GRUB)  (locates and loads the kernel + initrd)
   ↓
Kernel initialization  (kernel loads into memory, initializes hardware drivers, mounts root filesystem)
   ↓
init process (PID 1) — systemd on modern distros
   ↓
systemd starts targets/services in dependency order
   ↓
Login prompt / GUI
```

### Step-by-step breakdown

1. **BIOS/UEFI**: Firmware built into the motherboard. Performs POST (Power-On Self-Test), checks hardware, then locates a bootable device based on boot order settings. UEFI is the modern replacement for legacy BIOS, supporting larger disks (>2TB) and faster boot via Secure Boot.

2. **GRUB (GRand Unified Bootloader)**: Loaded by BIOS/UEFI from the boot device. Presents the boot menu (if multiple kernels/OS options exist), then loads the selected Linux **kernel** and **initrd/initramfs** (a temporary minimal root filesystem containing drivers needed to mount the REAL root filesystem) into memory.

```bash
cat /boot/grub/grub.cfg            # GRUB configuration (auto-generated, don't hand-edit directly)
sudo update-grub                      # regenerate GRUB config after kernel changes (Debian/Ubuntu)
sudo grub2-mkconfig -o /boot/grub2/grub.cfg   # RHEL equivalent
```

3. **Kernel initialization**: The kernel takes over, initializes hardware/drivers, mounts the actual root filesystem (using initrd's drivers if needed for things like RAID/LVM/encrypted disks), then hands off control to the first userspace process.

4. **init / systemd (PID 1)**: The very first userspace process. On modern distros this is `systemd`, which then starts everything else — services, mounts, targets — according to defined dependencies, largely in parallel for speed (a key improvement over older sequential SysV init).

---

## 6. systemd — Units, Targets, and `systemctl`

### Units
systemd manages "units" — different types of manageable resources:
| Unit Type | Purpose |
|---|---|
| `.service` | A background service/daemon (e.g., `nginx.service`) |
| `.mount` | A filesystem mount point |
| `.timer` | A scheduled task (systemd's modern alternative to cron) |
| `.socket` | Socket-based activation (start a service on-demand when its socket receives a connection) |
| `.target` | A grouping of units — conceptually similar to old "runlevels" |

### `systemctl` — The main systemd management command

```bash
sudo systemctl start nginx           # start a service NOW
sudo systemctl stop nginx               # stop a service NOW
sudo systemctl restart nginx               # stop then start
sudo systemctl reload nginx                   # reload config WITHOUT fully restarting (if the service supports it — gentler than restart)
sudo systemctl status nginx                      # current status + recent log lines + whether it's enabled

sudo systemctl enable nginx                         # make the service START AUTOMATICALLY on future boots (does NOT start it now)
sudo systemctl enable --now nginx                      # enable AND start immediately in one command (very common combo)
sudo systemctl disable nginx                              # stop it from auto-starting on boot (does NOT stop it if currently running)

systemctl is-active nginx                                    # quick check: just prints "active" or "inactive"
systemctl is-enabled nginx                                      # quick check: is it set to autostart on boot?
systemctl list-units --type=service                                # list all currently loaded service units
systemctl list-units --type=service --state=failed                    # list only FAILED services — great for a quick health check
```

**Critical, very commonly tested distinction: `start` vs `enable`**
> "`start` runs the service RIGHT NOW, but if the server reboots, it won't automatically come back unless it's also enabled. `enable` sets it to auto-start on FUTURE boots, but doesn't affect its current running state — enabling a stopped service doesn't start it immediately. This is why `enable --now` is such a common combo when setting up a new service you want both running now AND surviving reboots."

### Unit files

```bash
systemctl cat nginx                     # view the actual unit file content
sudo systemctl edit nginx                  # create an override file for customizing a unit without touching the original
ls /etc/systemd/system/                       # custom/enabled unit files typically live here
ls /usr/lib/systemd/system/                      # default unit files shipped by packages
sudo systemctl daemon-reload                        # MUST run this after manually editing a unit file, so systemd re-reads the updated definition
```

Basic unit file structure example:

```ini
[Unit]
Description=My Custom App
After=network.target

[Service]
ExecStart=/usr/bin/myapp
Restart=on-failure
User=appuser

[Install]
WantedBy=multi-user.target
```

**Interview-relevant note on `daemon-reload`:** If you manually edit a `.service` file and forget to run `sudo systemctl daemon-reload`, systemd keeps using its OLD cached definition of that unit — your changes silently won't take effect until you reload. This is a very common real "why isn't my config change working" mistake.

---

## 7. Runlevels / Targets

Old SysV init used numbered **runlevels** (0-6); systemd replaces this concept with **targets**, though it maintains backward-compatible aliases.

| Old Runlevel | systemd Target | Meaning |
|---|---|---|
| 0 | `poweroff.target` | Shutdown |
| 1 | `rescue.target` | Single-user/rescue mode |
| 3 | `multi-user.target` | Full multi-user mode, NO GUI (typical for servers) |
| 5 | `graphical.target` | Multi-user mode WITH GUI |
| 6 | `reboot.target` | Reboot |

```bash
systemctl get-default              # show the current default target (what boots into by default)
sudo systemctl set-default multi-user.target      # set default boot target (e.g., server without GUI)
sudo systemctl isolate rescue.target                 # switch to a DIFFERENT target immediately (like changing runlevel live)
systemctl list-units --type=target                      # list all currently active targets
```
**Server-relevant fact:** Most production servers boot into `multi-user.target` (no GUI) rather than `graphical.target`, since a GUI consumes unnecessary resources on a headless server accessed purely via SSH.

---

## Quick Reference Cheat Sheet

```bash
# Logs
tail -f /var/log/syslog                 # live system log
tail -f /var/log/auth.log                  # live auth/login log
journalctl -u nginx -f                        # live logs for a specific service
journalctl -e                                    # jump to most recent logs
journalctl -b -1                                    # logs from PREVIOUS boot
journalctl -p err                                      # filter by priority
dmesg -T | grep -i error                                  # kernel errors, human-readable time

# Resource monitoring
uptime                          # load average
nproc                              # CPU core count (compare against load average)
free -h                               # memory + swap usage
swapon --show                            # active swap devices
vmstat 2 5                                  # performance snapshot every 2s, 5 times
iostat -x 2                                    # per-disk I/O stats

# Boot & systemd
systemctl status nginx                # service status
systemctl start / stop / restart nginx    # control service NOW
systemctl enable --now nginx                 # autostart on boot + start now
systemctl is-enabled nginx                      # check autostart config
systemctl daemon-reload                            # reload unit file changes
journalctl -u nginx --since "10 min ago"               # recent logs for a failing service
systemctl get-default                                     # current boot target
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: A service fails to start. What's your exact troubleshooting sequence?**
> A: First `systemctl status servicename` for a quick overview and recent log snippet. Then `journalctl -u servicename -e` or `journalctl -u servicename --since "10 min ago"` to see the full, detailed logs around the failure. That almost always reveals the actual error — a missing config file, permission issue, port conflict, or dependency failure — which then guides the specific fix.

**Q2: What's the difference between `systemctl start` and `systemctl enable`?**
> A: `start` runs the service immediately but has no effect on boot behavior — if the server restarts, the service won't come back unless it's also enabled. `enable` configures the service to auto-start on future boots but doesn't affect whether it's running right now. You often want both, which is why `systemctl enable --now servicename` is a common combined command.

**Q3: Why does Linux show low "free" memory even when the system is healthy, and what should you actually look at instead?**
> A: Linux aggressively uses otherwise-idle RAM for disk caching and buffers, since that memory would otherwise sit unused — this cache is instantly reclaimed the moment an application actually needs it. So "free" memory being low is completely normal and doesn't indicate a problem. The `available` column in `free -h` is the meaningful number — it estimates how much memory is genuinely available for new processes once reclaimable cache is accounted for.

**Q4: Explain what swap is and when high swap usage becomes a real concern.**
> A: Swap is disk space used as overflow when physical RAM is under pressure — the kernel moves less-active memory pages from RAM to disk to free up physical memory. A small amount of swap usage isn't necessarily alarming, since the kernel may swap out genuinely idle pages even with free RAM available. The real red flag is HIGH or continuously GROWING swap usage, since swap is dramatically slower than RAM, and heavy reliance on it causes serious performance degradation, sometimes called "thrashing."

**Q5: Walk through the Linux boot process from power-on to login prompt.**
> A: Power-on triggers BIOS/UEFI, which performs hardware checks (POST) and locates a bootable device. It hands off to the bootloader, GRUB, which loads the selected kernel and an initial temporary filesystem (initrd) into memory. The kernel initializes hardware and drivers, mounts the real root filesystem, then starts the first userspace process, PID 1 — systemd on modern distros. systemd then starts all configured services and targets in dependency order, eventually reaching a login prompt or graphical target.

**Q6: What happens if you edit a systemd unit file but forget to run `daemon-reload`?**
> A: systemd continues using its previously cached, in-memory version of that unit definition — your edits to the file on disk won't actually take effect until `systemctl daemon-reload` tells systemd to re-read all unit files. This is a very common real mistake: someone edits a `.service` file, restarts the service, and is confused why their changes "aren't working," when really the reload step was simply skipped.

**Q7: What does `journalctl -b -1` show, and why is it useful?**
> A: It shows journal logs from the PREVIOUS boot, rather than the current one. This is genuinely useful after an unexpected crash or reboot — you can review exactly what was happening in the logs right before the system went down, which is often impossible to see any other way once the system has restarted.

**Q8: What's the practical difference between `systemctl restart` and `systemctl reload`?**
> A: `restart` fully stops and then starts the service again — briefly interrupting availability, and losing any in-memory state. `reload` asks the service to reload its configuration WITHOUT a full stop/start cycle, if the service supports it — for example, Nginx can reload its config and pick up changes while continuing to serve existing connections, avoiding any downtime. Not all services support reload; it depends on how their unit file and the application itself are implemented.

**Q9: What's `vm.swappiness`, and why might a database server set it very low?**
> A: `vm.swappiness` is a kernel tunable (0-100, default often 60) controlling how aggressively the kernel prefers to swap memory pages to disk versus keeping them in RAM. Database servers often set it very low (even 0 or 1) because swapping database memory to disk introduces unpredictable, severe latency spikes — databases generally prefer relying on sufficient RAM and avoiding swap almost entirely, rather than degrading gracefully into disk-based swap.

**Q10: How would you check which services failed to start on a system, quickly?**
> A: `systemctl list-units --type=service --state=failed` — this filters directly to only services in a failed state, giving an immediate health overview without manually scanning through every service's status one by one.

**Q11: What's the difference between old-style runlevels and systemd targets, and which target do most servers boot into?**
> A: Runlevels were numbered states (0-6) used by the older SysV init system to represent things like shutdown, single-user mode, or graphical mode. systemd replaces this with named "targets" that serve a similar conceptual purpose but with more flexible dependency management, while keeping backward-compatible aliases (e.g., runlevel 5 maps to `graphical.target`). Most production servers boot into `multi-user.target`, which is full multi-user mode WITHOUT a GUI, since a graphical environment is unnecessary overhead on a headless server managed purely via SSH.

**Q12: High `wa` (I/O wait) percentage shows up in `vmstat`. What does that indicate?**
> A: High I/O wait means the CPU is frequently sitting idle, waiting for disk I/O operations to complete, rather than actually being CPU-bound. This points to a disk/storage bottleneck rather than a compute bottleneck — I'd investigate further with `iostat -x` to identify which specific disk device is under heavy load or performing poorly.
