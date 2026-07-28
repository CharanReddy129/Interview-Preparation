# Process Management

---

## 1. What is a Process?

A process is a running instance of a program. Every process has:

- A unique **PID** (Process ID)
- A **PPID** (Parent Process ID) — the process that created it
- An owner (UID/GID)
- A state (running, sleeping, stopped, zombie — covered below)
- Its own memory space, open file descriptors, and CPU time allocation

Every process (except the very first one) is created by another process via `fork()` — the parent duplicates itself, then typically `exec()`s to replace itself with the new program. This is why every process has a parent, forming a tree structure, all the way up to PID 1.

```bash
ps -ef --forest       # visualize the process tree, showing parent-child relationships
pstree                  # dedicated tree view (if installed)
```

**PID 1** is special — it's the first process started by the kernel at boot (`systemd` on most modern distros, or `init` on older ones). If a process's parent dies, PID 1 (or a systemd equivalent) "adopts" the orphaned process — covered more below.

---

## 2. Viewing Processes: `ps`, `top`, `htop`

### `ps` — Snapshot of processes at this moment (not live-updating)

```bash
ps                     # shows processes for YOUR current terminal session only
ps -e                  # ALL processes on the system
ps -ef                 # all processes, full format (shows PPID, UID, etc.)
ps -eF                 # like -ef but with more columns (CPU/memory)
ps aux                         # BSD-style syntax — extremely common combo, shows user, PID, %CPU, %MEM, command
ps -ef | grep nginx           # find a specific process by name
ps -o pid,ppid,cmd -p 1234     # show specific columns for a specific PID
```

**`ps aux` column breakdown:**

```bash
USER   PID  %CPU  %MEM   VSZ   RSS  TTY  STAT  START  TIME  COMMAND
```

- `%CPU` / `%MEM` — resource usage
- `VSZ` — virtual memory size
- `RSS` — resident set size (actual physical RAM currently used)
- `STAT` — process state (see states table below)
- `TIME` — total CPU time consumed (not wall-clock/elapsed time — a common confusion)

### `top` — Live, continuously updating view

```bash
top                  # interactive live view, refreshes every few seconds by default
# Inside top:
#   P = sort by CPU usage
#   M = sort by memory usage
#   k = kill a process (prompts for PID)
#   q = quit
#   1 = toggle per-core CPU display
```

Shows system-wide summary at the top (load average, memory, swap usage) plus a live process table below.

### `htop` — Improved, more user-friendly version of `top`

```bash
htop
```

Advantages over `top`: color-coded output, mouse support, easier process tree view, scrollable list, and simpler kill/renice via function keys — generally preferred when available, but `top` is guaranteed to exist on virtually every system, which matters when working on minimal/production servers where `htop` may not be installed.

---

## 3. Process States

| State (STAT code) | Meaning |
| --- | --- |
| `R` | Running or runnable (in queue to run) |
| `S` | Sleeping (interruptible) — waiting for an event/resource, can be woken by a signal |
| `D` | Uninterruptible sleep — usually waiting on I/O (disk/network); notably CANNOT be killed with normal signals while in this state |
| `T` | Stopped — paused, usually via a signal like `SIGSTOP` or `Ctrl+Z` |
| `Z` | Zombie — process has finished executing but its exit status hasn't been collected yet (see below) |
| `<` | High priority process (appended as a modifier) |
| `+` | Foreground process group (appended as a modifier) |

**Interview-relevant:** A process in `D` state (uninterruptible sleep, typically stuck waiting on disk I/O) is notoriously hard to kill — even `kill -9` won't work while it's in that state, since the process hasn't returned control to the kernel enough to process the signal. This is a real production headache, often caused by disk/NFS issues.

---

## 4. Foreground vs Background Processes

By default, when you run a command, it runs in the **foreground** — it occupies your terminal, and you can't type new commands until it finishes.

### Running in background

```bash
long_running_script.sh &         # & runs the command in the background immediately, frees up terminal
```

### Managing background/stopped jobs

```bash
jobs                    # list background/stopped jobs in the CURRENT shell session
jobs -l                    # include PIDs in the listing

Ctrl+Z                       # SUSPEND (pause) the current foreground process, sends SIGTSTP
bg                              # resume a suspended job, but keep it running in the BACKGROUND
bg %1                              # resume specific job number 1 in background

fg                                    # bring a background job to the FOREGROUND
fg %2                                    # bring specific job number 2 to foreground
```

### `nohup` — Keep a process running even after you log out / close the terminal

```bash
nohup long_script.sh &                    # runs in background AND ignores SIGHUP (the signal sent when terminal closes)
nohup long_script.sh > output.log 2>&1 &     # also redirect output, since nohup output otherwise goes to nohup.out by default
```

**Why this matters practically:** Without `nohup`, closing your SSH session/terminal sends `SIGHUP` to child processes, which by default terminates them. `nohup` is essential for kicking off long-running deployment scripts, builds, or migrations over SSH that need to survive you disconnecting.

**Modern alternative worth mentioning in interviews:** `tmux` or `screen` — these create persistent terminal sessions you can detach from and reattach to later, which is often preferred over `nohup` for genuinely long/interactive background work, since you can reconnect and see live output, not just a log file.

---

## 5. Zombie vs Orphan Processes

### Zombie Process

A process that has **finished executing** (called `exit()`) but its entry still exists in the process table because its **parent hasn't yet read its exit status** via `wait()`.

- Shows as state `Z` or "defunct" in `ps`
- Consumes almost NO resources (no memory, no CPU) — just a small process table entry holding the exit code
- **Cannot be killed** with `kill` — since it's already dead/not running, there's nothing to signal
- **Fix:** The only real fix is for the PARENT process to call `wait()` and collect the exit status, which removes the zombie. If the parent is buggy and never does this, you may need to kill or restart the PARENT process, at which point the zombie gets reassigned to PID 1, which reaps it immediately.

```bash
ps aux | grep 'Z'          # find zombie processes
```

### Orphan Process

A process whose **parent has terminated** BEFORE the child finished — the child is still actively running, just without its original parent.

- Immediately gets "adopted" (re-parented) by **PID 1** (`systemd`/`init`), or in some systems a subreaper process
- Continues running completely normally — not a problem state, unlike a zombie
- `systemd`/init will properly `wait()` for it when it eventually finishes, so orphans don't become zombies under a properly functioning init system

### Key distinction

| | Zombie | Orphan |
| --- | --- | --- |
| Process itself | Dead (finished execution) | Alive (still running) |
| Problem is | Parent hasn't collected exit status yet | Parent died, but child is fine, just re-parented |
| Resource usage | Negligible (just a process table entry) | Normal (still actively consuming resources like any process) |
| Fix | Parent needs to call wait(), or kill the parent | Not actually a problem — PID 1 adopts and manages it |

**One-line answer for interviews:** "A zombie is a dead process waiting for its parent to acknowledge it; an orphan is a live process whose parent died and got reassigned to init. Zombies are essentially harmless bookkeeping leftovers unless they accumulate in huge numbers from a buggy parent that never reaps its children, at which point they can exhaust the process table."

---

## 6. Signals & `kill`

Signals are how Unix/Linux processes communicate asynchronous events — including how you tell a process to stop, reload, or terminate.

### Most important signals

| Signal | Number | Meaning | Can be caught/ignored? |
| --- | --- | --- | --- |
| `SIGHUP` | 1 | Hang up — originally meant terminal disconnected; commonly repurposed by daemons to mean "reload config" | Yes |
| `SIGINT` | 2 | Interrupt — sent by Ctrl+C | Yes |
| `SIGKILL` | 9 | Force kill — immediate, unconditional termination | **NO — cannot be caught, blocked, or ignored** |
| `SIGTERM` | 15 | Terminate gracefully — default signal sent by plain `kill` | Yes (process can clean up before exiting) |
| `SIGSTOP` | 19 | Pause/suspend the process | **NO — cannot be caught or ignored** |
| `SIGCONT` | 18 | Resume a stopped process | Yes |
| `SIGTSTP` | 20 | Sent by Ctrl+Z — like SIGSTOP but CAN be caught/handled | Yes |

### `kill` command syntax

```bash
kill 1234              # sends SIGTERM (15) by default — graceful shutdown request
kill -9 1234              # sends SIGKILL — immediate force kill, no cleanup
kill -SIGTERM 1234           # explicit, same as plain kill
kill -HUP 1234                 # send SIGHUP — commonly used to make a daemon reload its config without restarting
kill -l                          # list ALL available signal names/numbers
```

### `killall` — Kill by process NAME instead of PID

```bash
killall nginx           # kill ALL processes named "nginx"
killall -9 firefox         # force kill all matching processes
```

### `pkill` — Kill by pattern matching (more flexible than killall)

```bash
pkill -f "python.*script.py"       # kill based on a pattern matched against the full command line
pkill -u arjun                        # kill all processes owned by user arjun
```

### **Critical interview point: `SIGTERM` vs `SIGKILL`**

> "`SIGTERM` (15) is a polite request — it asks the process to terminate gracefully, giving it a chance to save state, close file handles, and clean up before exiting; well-written applications catch this signal and shut down properly. `SIGKILL` (9) is not a request — it's the kernel forcibly terminating the process immediately, with NO opportunity for cleanup, which can lead to issues like corrupted files or orphaned locks if the process was mid-write. Best practice is always to try `SIGTERM` first and only escalate to `SIGKILL` if the process doesn't respond after a reasonable wait — this is exactly what Docker and Kubernetes do by default when stopping containers (SIGTERM, then SIGKILL after a grace period, typically 10-30 seconds)."

This last point is a great one to mention — it directly connects Linux fundamentals to real container orchestration behavior, which interviewers love hearing from entry-level candidates.

---

## 7. Process Priority: `nice` and `renice`

Linux uses a **niceness** value to influence CPU scheduling priority, ranging from **-20 (highest priority) to +19 (lowest priority)**. Default niceness for a new process is **0**.

**Confusing but important:** LOWER niceness number = HIGHER priority (gets more CPU time). The name comes from the idea that a "nicer" process (higher number) is more considerate/yields more CPU to others.

```bash
nice -n 10 long_task.sh          # start a NEW process with LOWER priority (niceness +10)
nice -n -5 important_task.sh        # start with HIGHER priority (niceness -5) — requires root for negative values

renice -n 5 -p 1234                    # change priority of an ALREADY RUNNING process (PID 1234) to niceness 5
renice -n -10 -p 1234                     # requires root, since negative niceness needs elevated privileges

ps -o pid,ni,comm -p 1234                   # view niceness ('ni' column) of a specific process
top                                            # niceness also shown in the NI column
```

**Real-world use case:** If a backup script is running and slowing down a production application sharing the same server, you could `renice` the backup script to a higher niceness value (lower priority) so the kernel scheduler favors the production app for CPU time.

---

## Quick Reference Cheat Sheet

```bash
ps aux                          # all processes, detailed
ps -ef --forest                 # process tree view
top / htop                      # live process monitor
jobs                            # background jobs in current shell
Ctrl+Z                          # suspend foreground process
bg / fg                         # resume in background / bring to foreground
nohup cmd &                     # survive terminal close
kill PID                        # graceful terminate (SIGTERM)
kill -9 PID                     # force kill (SIGKILL)
killall processname             # kill by name
pkill -f pattern                # kill by command-line pattern
kill -l                         # list all signals
nice -n 10 cmd                  # start with lower priority
renice -n 5 -p PID              # change priority of running process
ps aux | grep 'Z'               # find zombie processes
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What's the difference between a zombie process and an orphan process?**
> A: A zombie process has already finished executing, but its entry remains in the process table because its parent hasn't yet called `wait()` to collect its exit status — it's dead, just not fully cleaned up. An orphan process is the opposite: it's still actively running, but its PARENT has died, so it gets re-parented to PID 1 (init/systemd), which will properly manage and eventually reap it. Zombies are dead-but-not-cleaned-up; orphans are alive-but-reassigned.

**Q2: Can you kill a zombie process with `kill -9`? Why or why not?**
> A: No — a zombie process is already dead/terminated; it has no running code left to receive or respond to any signal. The only way to clear it is for its parent process to call `wait()` and read its exit status, or if the parent itself is killed/restarted, at which point the zombie gets reassigned to PID 1, which reaps it immediately.

**Q3: What's the practical difference between `SIGTERM` and `SIGKILL`?**
> A: `SIGTERM` (signal 15) is the default signal sent by `kill`, and it asks a process to terminate gracefully — the process can catch this signal and perform cleanup like closing files or saving state before exiting. `SIGKILL` (signal 9) forcibly and immediately terminates the process at the kernel level with zero opportunity for cleanup — it cannot be caught, blocked, or ignored. Best practice is to try SIGTERM first and only use SIGKILL if the process becomes unresponsive, since SIGKILL can leave behind corrupted files or orphaned resources if the process was mid-operation.

**Q4: Why can't a process in "D" state be killed even with `kill -9`?**
> A: "D" is uninterruptible sleep, typically while the process is waiting on I/O like a disk or network operation at the kernel level. In this state, the process hasn't returned control back to a point where it can process signals — the kernel itself is blocking it. This is a genuine production pain point, often caused by disk or NFS issues, and sometimes the only real fix is resolving the underlying I/O problem or, in stubborn cases, rebooting.

**Q5: What does `nohup` actually do, and when would you use it?**
> A: `nohup` makes a process ignore the `SIGHUP` signal, which is normally sent to child processes when the parent terminal/session closes (like disconnecting from SSH). Without `nohup`, closing your SSH session would terminate any foreground/background processes you started in that session. It's essential for kicking off long-running tasks like deployments, backups, or builds over SSH that need to keep running after you disconnect.

**Q6: Explain what `nice` and `renice` do, and why does a LOWER niceness number mean HIGHER priority?**
> A: Both control CPU scheduling priority, ranging from -20 (highest priority) to +19 (lowest priority), with 0 as default. `nice` sets priority when STARTING a new process; `renice` changes priority of an ALREADY RUNNING process. The naming is somewhat counterintuitive — a higher "niceness" value means the process is being more considerate/"nicer" to other processes by yielding CPU time, so it gets LESS priority, while a lower or negative value means it's less yielding and gets scheduled MORE aggressively.

**Q7: What's the difference between `kill`, `killall`, and `pkill`?**
> A: `kill` targets a process by its specific PID. `killall` targets processes by exact NAME, killing all matching processes at once. `pkill` is the most flexible — it can match against patterns in the full command line (`-f`), or filter by owning user (`-u`), letting you target processes more precisely without needing exact names or PIDs.

**Q8: How does Docker/Kubernetes actually stop a container under the hood, and how does that relate to signals?**
> A: When stopping a container, Docker/Kubernetes first sends `SIGTERM` to the main process, giving it a grace period (commonly 10-30 seconds by default) to shut down cleanly — closing connections, flushing buffers, finishing in-flight requests. If the process hasn't exited by the end of that grace period, it sends `SIGKILL` to force-terminate it. This is exactly the same SIGTERM-then-SIGKILL escalation pattern used at the OS level, just automated by the container runtime.

**Q9: What's the difference between `ps` and `top`?**
> A: `ps` gives a one-time snapshot of processes at the moment you run it — it doesn't update automatically. `top` is a live, continuously refreshing interactive view showing real-time CPU/memory usage and letting you sort, filter, or even kill processes directly from within it. I'd use `ps` for scripting or quick checks, and `top`/`htop` for actively monitoring system behavior over time.

**Q10: If you press Ctrl+Z on a running command, what actually happens, and how do you resume it?**
> A: Ctrl+Z sends `SIGTSTP`, which suspends (pauses) the foreground process and returns control of the terminal to you, without terminating the process. You can resume it in the background with `bg`, or bring it back to the foreground with `fg`. It stays paused, consuming no CPU, until resumed with one of those, or explicitly killed.

**Q11: What column in `ps aux` tells you actual physical RAM usage versus virtual memory, and why does that distinction matter?**
> A: `RSS` (Resident Set Size) shows actual physical RAM currently in use by the process, while `VSZ` (Virtual Size) shows the total virtual memory address space allocated, which can be much larger than what's actually resident in physical RAM (since it can include memory-mapped files, shared libraries, and unused allocated address space). RSS is generally the more meaningful number when diagnosing actual memory pressure on a system.

**Q12: A process seems stuck consuming high CPU — walk me through how you'd investigate it.**
> A: I'd start with `top` or `ps aux --sort=-%cpu` to identify the exact PID and process name consuming the most CPU. Then I'd check what it's doing — `ps -o pid,ppid,cmd -p PID` for context, potentially `strace -p PID` to see what system calls it's making if it seems stuck in a loop, and check logs related to that application. If it's genuinely unresponsive and safe to stop, I'd try `kill PID` (SIGTERM) first to allow graceful shutdown, and only escalate to `kill -9` if it doesn't respond within a reasonable time.
