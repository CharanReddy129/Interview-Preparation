# Linux Basics & Architecture

---

## What is Linux

Linux is a free and open-source operating system based on the Linux kernel. It manages the computer's hardware and provides an environment where users and applications can run.

An operating system acts as an interface between the user, applications, and the computer's hardware.

### Why was Linux created?

Before Linux, the Unix operating system was widely used, but it had some limitations:

- It was mostly proprietary and required expensive licences.
- The source code was not freely available.
- It was difficult for students and developers to modify or experiment with it.

In **1991, Linus Torvalds** created the Linux kernel to provide a free Unix-like operating system that anyone could use, modify, and distribute.

Today, thousands of developers around the world contribute to Linux.

### Why do we use Linux?

Linux is widely used because it is:

- Free and open source
- Stable and reliable
- Secure
- Fast and efficient
- Highly customizable
- Excellent for servers and cloud computing

Many modern technologies run on Linux, including:  ***Docker containers, Kubernetes clusters, Web servers, Databases, Android (uses the Linux kernel)***

### Features of Linux

- **Open Source**: The source code is publicly available.You can View the code, Modify it, Build your own distribution, Contribute improvements.
- **Multiuser**: Multiple users can use the same Linux system simultaneously. Each user has: Their own files, Their own permissions, Their own home directory.
- **Multitasking**: Linux can run multiple programs at the same time. All can run simultaneously because the kernel schedules CPU time efficiently.
- **Secure**: Linux provides strong security features like: User authentication, File permissions, Groups, Firewall support, SELinux/AppArmor.
- **Stable**: Linux systems can run continuously for months or even years without requiring a reboot. This is one reason it is widely used for servers.
- **Portable**: Linux runs on many different architectures: Desktop computers, Servers, Raspberry Pi, Mobile devices, Supercomputers, Embedded systems.

## Kernel vs Shell vs Userspace

Understanding these three layers is the foundation of literally everything else in Linux.

### The Kernel

- The kernel is the core part of the Linux operating system. It acts as a bridge between software **(applications)** and hardware **(CPU, RAM, disk, network, keyboard, etc.)**.
- It runs in a privileged mode ("kernel space") and directly manages hardware.

`The Linux kernel is the core component of the operating system. It manages system resources such as CPU, memory, processes, filesystems, devices, and networking, and acts as the interface between user applications and the hardware.`

Responsibilities:

- **Process management** — scheduling which process gets CPU time, when
- **Memory management** — allocating RAM to processes, virtual memory, swapping
- **Device management** — drivers that talk to hardware (disk, network card, USB, etc.)
- **File system management** — reading/writing to disk via filesystems (ext4, xfs, etc.)
- **System calls** — the only way userspace programs can request kernel services (e.g. `open()`, `read()`, `fork()`)

### The Shell

A shell is a **command-line interpreter**. It reads the commands you type, interprets them, and asks the Linux kernel to execute them. It acts as a bridge between the user and the operating system.

- **Common shells**: `bash`, `sh`, `zsh`, `ksh`, `fish`
- **Check your shell**: `echo $SHELL` or `cat /etc/shells` to see available ones
- The shell is NOT the kernel — it's just a program, like any other, that happens to be your interface to the system
- You can be dropped into a shell, and even if it crashes, the kernel keeps running

### Userspace

Everything that isn't the kernel — the shell, applications, system utilities (`ls`, `cp`, `systemd`, your Python script, a web server) — all runs in **userspace**, in an unprivileged mode. Userspace programs cannot directly touch hardware or memory outside their own allocation; they must ask the kernel via **system calls**.

### Putting it together — the flow

```text
You type: ls -l
    ↓
Shell (bash) parses the command
    ↓
Shell asks kernel (via system calls like fork(), exec()) to run the ls program
    ↓
Kernel schedules the process, allocates memory, loads the binary
    ↓
ls program runs in userspace, uses system calls (like readdir()) to ask kernel for directory info
    ↓
Kernel reads from disk (via filesystem driver), returns data
    ↓
ls formats and prints output to your terminal
```

Kernel = privileged, talks to hardware directly. Shell = a userspace program, just an interface. Userspace = everything else, restricted, must go through kernel via system calls to do anything privileged.

### What is Terminal?

A terminal is a program (or window) that lets you interact with the operating system by typing commands.
It is simply an interface for entering commands and displaying their output.
When you open a terminal, it usually starts a shell automatically.

### Terminal vs Shell

| **Terminal** | **Shell** |
| -------------- | ----------- |
| A program or window used to enter commands | A program that interprets commands |
| Provides the user interface | Processes and executes commands |
| Displays input and output | Communicates with the Linux kernel |
| Examples: GNOME Terminal, Windows Terminal, iTerm2 | Examples: Bash, Zsh, Fish, Sh |

---

## Popular Distributions & Their Differences

A "distro" (distribution) = Linux kernel + package manager + set of default tools/utilities + configuration conventions, bundled together.

| Distro | Family | Package Manager | Package Format | Common Use |
| --- | --- | --- | --- | --- |
| Ubuntu | Debian-based | `apt` / `apt-get` | `.deb` | General purpose, dev machines, cloud |
| Debian | Debian (original) | `apt` | `.deb` | Stability-focused servers |
| CentOS / RHEL | Red Hat-based | `yum` / `dnf` | `.rpm` | Enterprise servers, traditionally common in corporates |
| Amazon Linux | RHEL-based (Fedora-derived) | `yum` / `dnf` | `.rpm` | AWS EC2 default AMI, optimized for AWS |
| Fedora | Red Hat's community edition | `dnf` | `.rpm` | Cutting-edge features, RHEL testbed |
| Alpine | Independent | `apk` | `.apk` | Extremely lightweight, common base for Docker images |

### Why this matters

- **AWS EC2** default AMIs are often **Amazon Linux** (RPM-based, `yum`/`dnf`) or Ubuntu
- **RHEL/CentOS** is very common in traditional enterprise data centers
- Docker images frequently use **Alpine** for small image size, or **Debian-slim/Ubuntu** for compatibility
- Knowing the package manager and file locations differ between families is a real practical skill: e.g. Debian/Ubuntu uses `/etc/apt/sources.list`, RHEL-based uses `/etc/yum.repos.d/`

### Quick command comparison

```bash
# Debian/Ubuntu
apt update && apt upgrade
apt install nginx
apt remove nginx
dpkg -l                    # list installed packages

# RHEL/CentOS/Amazon Linux
yum update           # or dnf update (dnf is the newer replacement for yum)
yum install nginx
yum remove nginx
rpm -qa                    # list installed packages
```

### How to check which distro you're on

```bash
cat /etc/os-release
lsb_release -a
uname -a          # kernel version + architecture (not distro name, but related)
```

---

## Linux Filesystem Hierarchy Standard (FHS)

Everything in Linux lives under a single root `/`. All directories, disks, and devices are mounted somewhere under this one tree.

### Core Directories — Must Know

| Path | Purpose |
| --- | --- |
| `/` | Root of the entire filesystem tree |
| `/bin` | Essential user command binaries (ls, cp, cat) — needed even in single-user/recovery mode |
| `/sbin` | Essential **system administration** binaries (fdisk, reboot, iptables) — typically require root |
| `/etc` | System-wide configuration files (no binaries) — e.g. `/etc/passwd`, `/etc/hosts`, `/etc/ssh/sshd_config` |
| `/home` | Personal directories for each user (e.g. `/home/arjun`) |
| `/root` | Home directory for the root user specifically (separate from `/home`) |
| `/var` | "Variable" data that changes frequently — logs (`/var/log`), mail, spool, cache |
| `/tmp` | Temporary files, cleared on reboot |
| `/opt` | Optional/third-party software, often self-contained applications |
| `/usr` | User-installed programs and libraries (the bulk of installed software lives under `/usr/bin`, `/usr/lib`, etc. — despite the name, not literally "user files") |
| `/proc` | Virtual filesystem — a live, in-memory view into kernel and process information (not real files on disk) |
| `/sys` | Virtual filesystem exposing kernel/device/driver information (used for tuning hardware/kernel parameters) |
| `/dev` | Device files — represents hardware (e.g. `/dev/sda` = disk, `/dev/null`, `/dev/tty`) |
| `/mnt` | Conventional mount point for manually mounted temporary filesystems |
| `/media` | Auto-mount point for removable media (USB drives, CDs) |
| `/lib`, `/lib64` | Shared libraries needed by binaries in `/bin` and `/sbin` |
| `/boot` | Boot loader files, kernel image, initrd — needed to boot the system |

### `/bin` vs `/sbin` — commonly asked distinction

- `/bin`: commands any user might need, even for basic operation (`ls`, `cat`, `cp`, `bash`)
- `/sbin`: commands for system administration, typically run by root (`reboot`, `iptables`, `fdisk`, `ifconfig`)
- Note: on modern systems (Ubuntu, RHEL8+), `/bin` and `/sbin` are often just symlinks into `/usr/bin` and `/usr/sbin` as part of the "usr merge".

### `/proc` and `/sys` — Virtual Filesystems (frequently tested)

These are **not real files on disk** — they're generated live by the kernel in memory to expose runtime information.

```bash
cat /proc/cpuinfo        # live CPU info
cat /proc/meminfo        # live memory info
ls /proc/                # notice numbered folders — these are PIDs of running processes!
cat /proc/1234/status    # info about process with PID 1234
cat /proc/uptime         # system uptime in seconds

cat /sys/class/net/eth0/... # network interface info exposed by kernel
```

This is WHY tools like `ps`, `top`, and `free` work — they're literally just reading and parsing files under `/proc` under the hood.

### Config vs Data vs Logs

- **Config** → `/etc`
- **Variable/runtime data & logs** → `/var`
- **Temporary/disposable data** → `/tmp`
- **Actual installed software** → `/usr`, `/opt`

---

## Quick Reference Commands

```bash
uname -a                  # kernel version, architecture, hostname
cat /etc/os-release       # distro name and version
echo $SHELL               # current shell
cat /etc/shells           # available shells on system
lsb_release -a            # distro details (if installed)
ls /proc/                 # see all running process IDs
cat /proc/cpuinfo         # CPU info live from kernel
df -h                     # disk usage per mounted filesystem
mount                     # show all currently mounted filesystems
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: Explain the relationship between kernel, shell, and userspace in your own words.**
> A: The kernel is the core of the OS, running in privileged mode, directly managing hardware, memory, and processes. The shell is a userspace program that interprets the commands I type and translates them into system calls the kernel understands. Userspace is everything else — applications and utilities — that must go through the kernel via system calls to do anything privileged, since they can't touch hardware directly.

**Q2: What is a system call, and why can't userspace programs access hardware directly?**
> A: A system call is a controlled, defined interface that userspace programs use to request services from the kernel — like opening a file, reading from disk, or creating a process. Userspace programs are restricted from directly accessing hardware or kernel memory for stability and security reasons; if any random application could directly manipulate hardware, one buggy or malicious program could crash or compromise the entire system. The kernel acts as the gatekeeper.

**Q3: What's the difference between `/bin` and `/sbin`?**
> A: `/bin` contains essential commands any user might need for basic operation, like `ls` and `cp`. `/sbin` contains essential system administration binaries typically run by root, like `reboot`, `fdisk`, and `iptables`. On many modern distros both are now symlinked into `/usr/bin` and `/usr/sbin` as part of a filesystem consolidation, but the conceptual distinction still matters.

**Q4: Why would a cloud/DevOps engineer specifically need to know the differences between Debian-based and RHEL-based distros?**
> A: Because the package managers, config file locations, and even some default behaviors differ. For example, AWS EC2 commonly defaults to Amazon Linux (RPM-based, using yum/dnf), while many dev environments use Ubuntu (Debian-based, using apt). If I'm writing provisioning scripts or Ansible playbooks, using the wrong package manager command for the target OS will break automation — so knowing the target distro family upfront actually matters operationally, not just academically.

**Q5: What is `/proc` and why is it called a "virtual filesystem"?**
> A: `/proc` is generated live in memory by the kernel — it's not real data sitting on disk. It exposes real-time information about running processes and system/kernel state as if they were files, so you can `cat /proc/cpuinfo` or look at `/proc/<PID>/status` to inspect a running process. It's called virtual because the "files" are dynamically created representations of kernel data structures, not actual stored files.

**Q6: How do tools like `top` and `free` actually get their information?**
> A: They read and parse data from files under `/proc`, such as `/proc/meminfo` for memory stats and `/proc/[pid]/stat` for per-process CPU/memory info. These commands are essentially user-friendly wrappers around parsing the live kernel data exposed through `/proc`.

**Q7: If your shell crashes, does the kernel crash too?**
> A: No. The shell is just a userspace program, and it runs like any other application — if it crashes, the kernel is unaffected and keeps running. You could reconnect via a new shell/session and the system would still be operational, since the kernel operates at a completely separate, more privileged layer.

**Q8: What's the difference between `/var` and `/tmp`?**
> A: `/var` holds variable data that changes during system operation but is meant to persist across reboots — like logs in `/var/log`, mail, or spool data. `/tmp` holds temporary files meant to be disposable, and is typically cleared automatically on reboot. `/tmp` also has the sticky bit set by default so users can't delete each other's temp files.

**Q9: Where would you look to find out what Linux distribution and version a server is running?**
> A: `cat /etc/os-release` is the most reliable modern way — it gives distro name, version, and ID in a structured format. `lsb_release -a` is another common option if installed. `uname -a` gives kernel version and architecture but not the distro name specifically, which is a common mix-up.

**Q10: What's the practical difference between `/usr` and `/opt`?**
> A: `/usr` holds the bulk of installed software and libraries managed by the system's package manager — most standard applications live here. `/opt` is meant for optional, self-contained third-party software, often things installed outside the normal package manager (e.g., a vendor's proprietary application bundle that ships all its own dependencies in one folder).

---
