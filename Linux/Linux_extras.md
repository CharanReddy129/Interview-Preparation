# Text Processing + Archiving + Job Scheduling + Environment Config

---

# Text Processing Tools

## 1. `grep` — Search text using patterns

```bash
grep "error" file.txt                # find lines containing "error"
grep -i "error" file.txt             # case-INSENSITIVE search
grep -v "error" file.txt             # INVERT — show lines that DON'T match
grep -r "TODO" /project/             # RECURSIVE search through all files in a directory
grep -c "error" file.txt             # COUNT of matching lines, not the lines themselves
grep -n "error" file.txt             # show LINE NUMBERS alongside matches
grep -l "error" *.log                # list only FILENAMES that contain a match (not the matching lines)
grep -w "cat" file.txt               # match WHOLE WORD only (won't match "category")
grep -A 3 "error" file.txt           # show 3 lines AFTER each match (After context)
grep -B 3 "error" file.txt           # show 3 lines BEFORE each match (Before context)
grep -E "error|warning" file.txt     # Extended regex — allows | (OR) without escaping
grep -oP '(?<=user=)\w+' file.txt    # -P enables Perl regex, -o prints ONLY the matched part (not full line) — powerful combo for extraction
```

### `grep` with regex basics

```bash
grep "^Error" file.txt          # ^ = start of line — lines STARTING with "Error"
grep "failed$" file.txt         # $ = end of line — lines ENDING with "failed"
grep "[0-9]" file.txt           # match any line containing at least one digit
grep "colou\?r" file.txt        # \? makes the preceding character optional — matches "color" or "colour" (basic regex escaping needed)
grep -E "colou?r" file.txt      # same thing but cleaner with Extended regex (-E), no backslash needed
```

---

## 2. `sed` — Stream Editor (find & replace, and more)

```bash
sed 's/old/new/' file.txt              # replace FIRST occurrence of "old" with "new" PER LINE
sed 's/old/new/g' file.txt             # replace ALL occurrences on EVERY line (g = global)
sed -i 's/old/new/g' file.txt          # -i = edit the file IN PLACE (careful — no undo!)
sed -i.bak 's/old/new/g' file.txt      # -i.bak makes a BACKUP first before editing in place — much safer habit
sed -n '5,10p' file.txt                # print ONLY lines 5 through 10 (-n suppresses default full print, p = print)
sed '3d' file.txt                      # delete line 3
sed '/pattern/d' file.txt              # delete any line matching "pattern"
sed 's/^/PREFIX: /' file.txt           # add a prefix to the start of every line
sed 's/$/  <-- END/' file.txt          # add a suffix to the end of every line
```

**Real-world use case:** Bulk find-and-replace across config files, log cleanup, or quick inline edits without opening an editor — extremely common in deployment/automation scripts.

**Important safety habit interviewers like hearing:** "I always test a `sed` command WITHOUT `-i` first to preview the output before actually modifying the file in place, since `-i` changes are immediate with no built-in undo — or I use `-i.bak` to keep an automatic backup."

---

## 3. `awk` — Pattern Scanning & Column-Based Processing

`awk` is more powerful than `grep`/`sed` for anything involving COLUMNS of data (like log files, CSVs, `ps`/`df` output).

```bash
awk '{print $1}' file.txt                # print the FIRST column/field of every line (default delimiter = whitespace)
awk '{print $1, $3}' file.txt            # print 1st and 3rd columns
awk -F: '{print $1}' /etc/passwd               # -F sets a CUSTOM delimiter (here, colon) — print just usernames from /etc/passwd
awk '{print NR, $0}' file.txt            # NR = current line number, $0 = the whole line
awk '{print NF}' file.txt                # NF = number of fields on that line
awk '$3 > 100 {print $0}' file.txt       # CONDITIONAL — print lines where 3rd column is greater than 100
awk '/error/ {print $0}' file.txt        # pattern match, like grep, but with awk's column power available too
awk '{sum += $2} END {print sum}' file.txt    # SUM a column across all lines, print total at the END
```

**Real-world example — a genuinely common one:**

```bash
ps aux | awk '{print $2, $3, $11}'      # extract PID, %CPU, and COMMAND columns from ps output
df -h | awk '$5+0 > 80 {print $0}'          # find filesystems over 80% full (note: $5+0 forces the percentage string to be treated as a number for comparison)
```

### `grep` vs `sed` vs `awk` — the classic distinguishing question

| Tool | Best for |
| --- | --- |
| `grep` | FINDING lines matching a pattern |
| `sed` | REPLACING/transforming text, line-based edits |
| `awk` | Working with COLUMNS/FIELDS of structured data, calculations, conditional logic per line |

---

## 4. `cut` — Extract Columns (simpler alternative to awk for basic cases)

```bash
cut -d: -f1 /etc/passwd            # -d sets delimiter (colon), -f1 = extract FIELD 1 (username)
cut -d: -f1,3 /etc/passwd          # extract fields 1 AND 3
cut -d, -f2- data.csv              # field 2 to the END
cut -c1-5 file.txt                 # extract by CHARACTER position (columns 1-5), not delimiter-based
```

`cut` is simpler than `awk` but less powerful — good for straightforward, single-delimiter column extraction; reach for `awk` when you need conditionals, calculations, or multiple operations.

---

## 5. `sort`, `uniq`, `wc` (quick recap — sort and wc covered in earlier notes, `uniq` new here)

### `uniq` — Remove/report duplicate ADJACENT lines

```bash
uniq file.txt                # removes CONSECUTIVE duplicate lines (important: only adjacent duplicates, not all duplicates in the file!)
sort file.txt | uniq             # THIS is why sort + uniq is such a common combo — sort first so all duplicates become adjacent, THEN uniq catches them all
uniq -c file.txt                 # COUNT how many times each line repeats (adjacent)
sort file.txt | uniq -c | sort -rn     # classic combo: count occurrences, then sort by count descending — "find the most frequent lines" pattern
uniq -d file.txt                 # show ONLY lines that appear as duplicates
uniq -u file.txt                 # show ONLY lines that are UNIQUE (never repeated)
```

**Critical  (frequently tested):** `uniq` only removes ADJACENT duplicate lines — if identical lines are scattered throughout a file, not next to each other, `uniq` alone won't catch them. This is exactly why `sort file.txt | uniq` is the standard combo: sorting brings all identical lines together first, so `uniq` can then correctly identify every duplicate.

---

## 6. `tr` — Translate/Transform Characters

```bash
echo "hello" | tr 'a-z' 'A-Z'         # lowercase to uppercase → "HELLO"
tr -d '0-9' < file.txt                # DELETE all digits
tr -s ' ' < file.txt                  # SQUEEZE repeated spaces into a single space
echo "hello world" | tr ' ' '_'       # replace spaces with underscores → "hello_world"
cat file.txt | tr -d '\r'             # remove Windows-style carriage returns (classic fix for files edited on Windows causing issues on Linux)
```

---

## 7. Piping & Redirection — The Glue That Connects Everything

### Redirection operators

```bash
command > file.txt             # redirect STDOUT, OVERWRITE the file
command >> file.txt            # redirect STDOUT, APPEND to the file
command 2> error.log           # redirect STDERR only (file descriptor 2) to a file
command > output.log 2>&1      # redirect BOTH stdout AND stderr to the same file (2>&1 means "send stderr to wherever stdout is currently going")
command &> output.log          # shorthand for the same thing (bash-specific, cleaner syntax)
command < input.txt            # redirect STDIN — feed a file's content AS input to a command
command 2>/dev/null            # discard stderr entirely — commonly used to suppress "permission denied" noise, e.g., with find
```

**Critical ordering (frequently tested):** `command > output.log 2>&1` works correctly (stdout goes to file, then stderr follows stdout to the same file). But `command 2>&1 > output.log` does NOT work as most people expect — the ORDER matters. At the point `2>&1` executes, stdout is still pointing to the terminal, so stderr gets redirected to the terminal, THEN stdout gets redirected to the file — leaving stderr on-screen and only stdout in the file. Always put `2>&1` AFTER the stdout redirection.

### Piping (`|`) — Chain commands together

```bash
ps aux | grep nginx                # feed ps output AS INPUT into grep
cat access.log | grep "ERROR" | wc -l    # chain multiple commands: filter, then count
```

The pipe connects one command's STDOUT directly to the next command's STDIN, without needing an intermediate file.

### `tee` — Write to a file AND still display output on screen (or pass it along a pipe)

```bash
command | tee output.log             # show output on screen AND save to file simultaneously
command | tee -a output.log             # APPEND instead of overwrite
command | tee output.log | grep "error"    # save full output to file, WHILE ALSO continuing to pipe it into grep — genuinely useful when you want both a permanent record and further processing
sudo command | tee /etc/config.conf        # useful trick: since redirection (>) happens in your shell BEFORE sudo takes effect, `sudo command > file` can fail on protected files; `sudo command | tee file` works because tee (not the shell redirect) does the actual privileged write
```

**Why the sudo+tee trick matters (a genuinely clever, commonly-asked practical question):** `sudo echo "text" > /etc/protected_file` FAILS even with sudo, because the `>` redirection is set up by YOUR shell BEFORE `sudo` runs — your shell (not running as root) is the one trying to open the file for writing. `echo "text" | sudo tee /etc/protected_file` works because `tee` itself runs with sudo privileges and performs the actual file write.

### `xargs` — Build and execute commands FROM piped input

```bash
find . -name "*.tmp" | xargs rm              # take each line of find's output and pass it AS AN ARGUMENT to rm
echo "file1.txt file2.txt" | xargs cat          # convert piped input into arguments for another command
find . -name "*.log" | xargs -I {} mv {} /backup/    # -I {} defines a placeholder for each input item, letting you use it multiple times/flexibly in the command
find . -name "*.txt" -print0 | xargs -0 rm          # -print0 / -0 combo handles filenames with SPACES safely (a very real, commonly-tested gotcha)
```

**Why `xargs` is needed at all:** Many commands (like `rm`, `cp`) expect arguments directly on their command line, NOT via stdin — piping directly into them doesn't work the way you'd hope (`find . -name "*.tmp" | rm` doesn't work, since `rm` doesn't read from stdin). `xargs` bridges this gap by taking piped input and converting it into actual command-line ARGUMENTS for the next command.

**The filename-with-spaces:** Plain `find . -name "*.txt" | xargs rm` can break/misbehave on filenames containing spaces, since xargs by default splits on whitespace. The fix is `find . -name "*.txt" -print0 | xargs -0 rm`, where `-print0` separates filenames with a NULL character instead of whitespace/newlines, and `xargs -0` reads that format correctly — this is considered the "correct, safe" way to combine find and xargs in real scripts.

---

## 8. Regular Expressions — Core Basics Recap

| Symbol | Meaning | Example |
| --- | --- | --- |
| `^` | Start of line | `^Error` — line starts with "Error" |
| `$` | End of line | `failed$` — line ends with "failed" |
| `.` | Any single character | `f.le` matches "file", "fale", "f9le" |
| `*` | Zero or more of the PRECEDING character | `ab*c` matches "ac", "abc", "abbbc" |
| `+` (extended regex) | One or more of the preceding character | `ab+c` matches "abc", "abbc" but NOT "ac" |
| `?` (extended regex) | Zero or one of the preceding (optional) | `colou?r` matches "color" or "colour" |
| `[]` | Character class — any ONE of these | `[abc]` matches a, b, or c |
| `[^]` | Negated character class | `[^0-9]` matches anything that's NOT a digit |
| `\|` (extended regex) | OR | `cat\|dog` matches "cat" or "dog" |
| `()` (extended regex) | Grouping | `(ab)+` matches "ab", "abab", "ababab" |
| `\d`, `\w`, `\s` (Perl regex, `-P`) | Digit, word character, whitespace | Only available with `-P` flag in grep |

**Basic regex vs Extended regex (BRE vs ERE) — commonly confused:** Plain `grep` uses Basic Regular Expressions by default, where special characters like `+`, `?`, `|`, `()` need to be ESCAPED with a backslash to have their special meaning (`\+`, `\?`). `grep -E` (or `egrep`) uses Extended Regular Expressions, where these characters work directly without escaping — this is why you'll often see `-E` used for anything beyond the simplest patterns.

---

# Archiving & Compression

## 9. `tar` — Tape Archive (bundle multiple files into one)

**Important distinction:** `tar` by itself just BUNDLES files together (like a container) — it doesn't compress by default. Compression is typically added via flags that pipe through gzip/bzip2/xz.

```bash
tar -cvf archive.tar file1 file2 folder/       # CREATE (c), Verbose (v), File (f) — bundle into archive.tar, NO compression
tar -xvf archive.tar                              # eXtract (x) an archive
tar -tvf archive.tar                                 # LIST contents WITHOUT extracting (t = table of contents)

tar -czvf archive.tar.gz file1 folder/                  # CREATE + gZip compression — the single most common combo
tar -xzvf archive.tar.gz                                   # eXtract a gzip-compressed tarball
tar -cjvf archive.tar.bz2 folder/                             # CREATE + bZip2 compression (better compression ratio, slower than gzip)
tar -xjvf archive.tar.bz2                                        # eXtract a bzip2 archive
tar -cJvf archive.tar.xz folder/                                    # CREATE + xz compression (even better ratio, slower still)

tar -xzvf archive.tar.gz -C /path/to/destination/       # extract to a SPECIFIC destination directory (-C changes directory first)
tar -czvf backup.tar.gz --exclude='*.log' folder/           # exclude specific file patterns from the archive
```

**Flag mnemonic that helps in interviews:** "Create, extract, or list Table of contents — then optionally add z (gzip), j (bzip2), or J (xz) for compression — always paired with v (verbose) and f (specify the filename)."

---

## 10. `gzip` / `gunzip` — Single-file Compression

```bash
gzip file.txt              # compresses file.txt INTO file.txt.gz, REPLACING the original (important — original is gone unless you keep a copy!)
gunzip file.txt.gz            # decompress, restoring file.txt, replacing the .gz
gzip -k file.txt                 # -k = KEEP the original file, don't delete it after compressing
gzip -9 file.txt                    # maximum compression level (slower); levels range 1 (fastest, least compression) to 9 (slowest, best compression)
zcat file.txt.gz                       # view contents of a gzipped file WITHOUT fully decompressing it to disk
```

**Important note:** `gzip` only compresses a SINGLE file — it has no concept of bundling multiple files/folders together, which is exactly why it's almost always paired with `tar` (`tar` bundles, `gzip` compresses the bundle).

---

## 11. `zip` / `unzip` — Cross-platform archive format

```bash
zip archive.zip file1.txt file2.txt        # create a zip with specific files
zip -r archive.zip folder/                    # RECURSIVE — zip an entire folder
unzip archive.zip                                # extract
unzip -l archive.zip                                # LIST contents without extracting
unzip archive.zip -d /path/to/destination/             # extract to a specific directory
```

**Why `zip` vs `tar.gz` matters:** `zip` is more universally recognized cross-platform (Windows, macOS handle it natively without extra tools), while `tar.gz`/`tar.xz` is the more standard, more efficient convention within the Linux/Unix world, and better preserves Unix-specific file permissions and metadata. If sending an archive to a non-technical Windows user, `zip` is friendlier; for Linux server backups/transfers, `tar.gz` is the norm.

---

## 12. Backup Strategies — Conceptual Overview

### Full vs Incremental vs Differential (frequently asked conceptually)

| Type | What it captures | Restore complexity | Storage use |
| --- | --- | --- | --- |
| **Full backup** | EVERYTHING, every time | Simplest — just restore the one full backup | Highest (repeats unchanged data every time) |
| **Incremental backup** | Only what changed SINCE THE LAST BACKUP (full or incremental) | More complex — need the last full backup PLUS every incremental since then, applied in order | Lowest (most storage-efficient) |
| **Differential backup** | Everything changed SINCE THE LAST FULL backup (regardless of any differentials in between) | Moderate — need the last full backup PLUS only the most recent differential | Middle ground |

**The actual trade-off interviewers want you to articulate:**
> "Full backups are simplest to restore from but use the most storage and take longest to create. Incremental backups are the most storage/time efficient to CREATE, but restoring requires replaying the full backup plus every single incremental in the correct order, which is slower and riskier if any one increment in the chain is corrupted or missing. Differential backups are a middle ground — restore only needs the last full plus the latest differential, at the cost of each differential growing larger over time until the next full backup resets it."

### The 3-2-1 backup rule

- **3** copies of your data total
- **2** different storage media/types (e.g., local disk + cloud storage)
- **1** copy stored **off-site** (protects against physical disasters like fire/theft affecting the primary location)

### Practical tools/approaches worth mentioning

```bash
rsync -avz --delete /source/ /backup/destination/    # efficient incremental-style sync; --delete removes files in destination that no longer exist in source (mirrors exactly)
tar -czvf backup_$(date +%Y%m%d).tar.gz /important/data/    # simple timestamped full backup via cron
```

Cloud-specific backup concepts worth knowing exist (good to mention briefly even without hands-on depth): AWS EBS snapshots, S3 lifecycle policies for archival tiers, automated snapshot scheduling.

---

# Job Scheduling

## 13. `cron` — Recurring Scheduled Tasks

```bash
crontab -e              # edit YOUR OWN crontab (opens in default editor)
crontab -l                 # list your current cron jobs
crontab -r                    # remove ALL of your cron jobs (careful — no confirmation, no undo!)
sudo crontab -u username -e     # edit ANOTHER user's crontab (needs root)
```

### Cron syntax — MUST memorize this cold

```fs
*  *  *  *  *  command_to_run
│  │  │  │  │
│  │  │  │  └── Day of week (0-7, both 0 and 7 = Sunday)
│  │  │  └───── Month (1-12)
│  │  └──────── Day of month (1-31)
│  └─────────── Hour (0-23)
└────────────── Minute (0-59)
```

### Common cron examples (very frequently asked to write from scratch)

```bash
0 2 * * *        /path/to/backup.sh          # every day at 2:00 AM
*/15 * * * *     /path/to/check.sh              # every 15 minutes
0 0 1 * *        /path/to/monthly.sh              # midnight, on the 1st of every month
0 9 * * 1-5      /path/to/weekday_report.sh          # 9 AM, Monday through Friday only
30 4 * * 0       /path/to/weekly_cleanup.sh              # 4:30 AM every Sunday
0 */6 * * *      /path/to/task.sh                           # every 6 hours (at minute 0)
@reboot          /path/to/startup_script.sh                     # special shortcut — run once at system boot/reboot
```

> **Critical practical:** Cron jobs run in a minimal environment WITHOUT your interactive shell's `$PATH`, environment variables, or working directory. This is exactly why scripts should use ABSOLUTE PATHS (as covered back in Topic 2) — a script that works fine when you run it manually can silently fail under cron because it can't find a command that was only reachable through your personal `$PATH`.

```bash
# Best practice pattern for cron entries:
0 2 * * * /usr/bin/bash/home/arjun/scripts/backup.sh >> /home/arjun/logs/backup.log 2>&1
```
This explicitly specifies the full interpreter path, the full script path, AND redirects both stdout and stderr to a log file — since cron output isn't visible on a screen anywhere, failing to redirect output means you'll never see errors unless you specifically go looking.

### Cron logs

```bash
grep CRON /var/log/syslog        # Debian/Ubuntu — see cron execution history
cat /var/log/cron                   # RHEL — same idea
```

---

## 14. `at` — One-Time Scheduled Jobs

Unlike cron (recurring), `at` schedules something to run ONCE at a specific future time.

```bash
at 2:00pm                    # enter interactive mode, then type commands, Ctrl+D to finish
at now + 1 hour                  # relative time
at 10:00am tomorrow              # specific future time
echo "backup.sh" | at 2:00am     # non-interactive, one-line version

atq                         # list pending 'at' jobs (queue)
atrm 3                      # remove a specific pending job by its job number
```

**When to use `at` instead of cron (the actual distinguishing question):** `cron` is for RECURRING tasks (daily, weekly, etc.); `at` is for a single, one-off future execution — like "run this cleanup script once, tonight at midnight" without needing to create and then remember to delete a recurring cron entry afterward.

---

## 15. systemd Timers — The Modern Alternative to cron

Since most systems now use systemd, it offers **timer units** as a more modern, more manageable alternative to cron, with better logging integration (via `journalctl`) and more flexible dependency handling.

### Structure — a timer + service pair

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup Script

[Service]
ExecStart=/home/arjun/scripts/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer     # enable and start the timer (NOT the service directly — you interact with the .timer unit)
systemctl list-timers                       # see all active timers and their NEXT scheduled run
journalctl -u backup.service                # view logs for what the timer actually triggered — genuinely nicer than digging through a cron log file
```

### Why systemd timers over cron (the interview-worthy comparison)

> "systemd timers integrate directly with journalctl for logging, so I don't need to manually redirect output to a log file the way I do with cron — it's automatically captured. Timers also support more flexible scheduling options like `Persistent=true`, which ensures a missed run (e.g., because the machine was powered off at the scheduled time) actually executes as soon as the system is back up, rather than just silently skipping that occurrence like cron would. That said, cron's syntax is simpler and still extremely widely used and recognized, so knowing both is genuinely valuable."

**`OnCalendar` examples (cron-equivalent syntax for timers):**

```bash
OnCalendar=daily                  # once a day (midnight)
OnCalendar=*-*-* 02:00:00            # every day at 2 AM
OnCalendar=Mon..Fri 09:00               # weekdays at 9 AM
OnCalendar=weekly                          # once a week
```

---

# Environment & Shell Configuration

## 16. Shell Startup Files — `.bashrc`, `.bash_profile`, `/etc/environment`

### The core distinction (frequently asked, genuinely confusing)

| File | When it runs | Typical use |
| --- | --- | --- |
| `~/.bashrc` | Every NEW interactive, NON-login shell (e.g., opening a new terminal tab) | Aliases, functions, prompt customization, PATH tweaks for interactive use |
| `~/.bash_profile` (or `~/.profile`) | LOGIN shells only (e.g., SSH-ing in, or a fresh terminal login) | Environment variables that should be set once per login session |
| `/etc/environment` | System-WIDE, applies to ALL users, ALL shells | System-wide environment variables (not shell-specific syntax — simpler `KEY=value` format, no shell scripting allowed here) |
| `/etc/profile` | System-WIDE, LOGIN shells for all users | System-wide login shell settings (like a global `.bash_profile` for everyone) |

**The classic confusion, and how to explain it well in an interview:** "Login shells (like SSH-ing into a server) run `.bash_profile` (or `/etc/profile` first, then user-specific files), while everyday interactive shells like opening a new terminal tab on your desktop typically only run `.bashrc`. A common pattern is for `.bash_profile` to explicitly `source ~/.bashrc` at the end, ensuring consistent behavior whether you're in a login or non-login shell — many default distro setups already do this for you."

```bash
source ~/.bashrc            # reload .bashrc in your CURRENT shell without closing/reopening the terminal — common after editing it
. ~/.bashrc                     # the dot (.) is a shorthand alias for 'source', does the exact same thing
```

---

## 17. `export` and the `PATH` Variable

### `export` — Make a variable available to CHILD processes, not just the current shell

```bash
MY_VAR="hello"              # regular shell variable — only visible in THIS shell, NOT passed to any programs/scripts it runs
export MY_VAR="hello"          # now MY_VAR is an ENVIRONMENT variable — passed down to any child process/script launched from this shell
echo $MY_VAR                 # view a variable's value
env                          # list ALL currently exported environment variables
printenv PATH                # view a SPECIFIC environment variable's value
```

**The actual conceptual distinction (commonly tested):** A regular shell variable exists only within your current shell session and is NOT inherited by any program or script you run from it. `export`-ing a variable promotes it to an ENVIRONMENT variable, which IS passed down to child processes — this is why a script might fail to see a variable you set, if you forgot to `export` it first.

### `PATH` — Where the shell looks for executable commands

```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

export PATH=$PATH:/home/arjun/scripts       # ADD a new directory to the END of existing PATH (append, don't overwrite!)
export PATH=/home/arjun/scripts:$PATH          # ADD to the BEGINNING — commands in this directory take priority if there's a naming conflict
```

**Critical gotcha (frequently tested):** `PATH` is a colon-separated LIST of directories the shell searches, IN ORDER, when you type a command name. If you accidentally do `export PATH=/home/arjun/scripts` (WITHOUT `$PATH:` included), you OVERWRITE the entire PATH, and suddenly basic commands like `ls` or `cd` stop working because the shell can no longer find them anywhere. Always append/prepend to the EXISTING `$PATH`, never replace it outright.

```bash
which python3         # shows WHICH directory in PATH actually provides the 'python3' command being used
```

---

## 18. Aliases

```bash
alias ll='ls -la'                   # create a shortcut — typing 'll' now runs 'ls -la'
alias gs='git status'                  # common developer shortcut
alias rm='rm -i'                          # make rm ALWAYS ask for confirmation — a popular safety habit
unalias ll                                   # remove an alias
alias                                           # list ALL currently defined aliases
```

**Important:** Aliases defined directly in the terminal only last for the CURRENT session — to make them PERMANENT, add the `alias` lines to `~/.bashrc` (since that file runs for every new interactive shell).

```bash
# Add to ~/.bashrc:
alias ll='ls -la'
alias update='sudo apt update && sudo apt upgrade -y'
```

---

## Quick Reference Cheat Sheet

```bash
# Text processing
grep -rn "TODO" .                       # recursive search with line numbers
sed -i 's/old/new/g' file.txt              # in-place replace all occurrences
awk -F: '{print $1}' /etc/passwd              # extract a column with custom delimiter
cut -d, -f2 data.csv                             # extract column 2 from CSV
sort file.txt | uniq -c | sort -rn                  # count and rank duplicate lines
tr 'a-z' 'A-Z' < file.txt                              # convert to uppercase

# Redirection/piping
command > out.log 2>&1                   # both stdout+stderr to one file (correct order!)
command | tee output.log                    # display AND save output
find . -print0 | xargs -0 rm                   # safely handle filenames with spaces

# Archiving
tar -czvf archive.tar.gz folder/          # create gzip-compressed tarball
tar -xzvf archive.tar.gz                     # extract it
zip -r archive.zip folder/                      # cross-platform zip

# Cron / scheduling
crontab -e                          # edit your cron jobs
0 2 * * *  /path/script.sh             # daily at 2 AM
at now + 1 hour                           # one-time future job
systemctl list-timers                        # view systemd timers

# Environment
source ~/.bashrc                    # reload shell config
export VAR=value                       # make available to child processes
export PATH=$PATH:/new/dir                # add to PATH (never overwrite!)
alias ll='ls -la'                            # shortcut
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What's the practical difference between `grep`, `sed`, and `awk`?**
> A: `grep` is for FINDING lines that match a pattern. `sed` is for transforming/replacing text, typically line by line. `awk` is the most powerful of the three for structured, column-based data — it can filter by column values, perform calculations across fields, and apply conditional logic per line, which grep and sed aren't designed for.

**Q2: Why does `sort file.txt | uniq` work correctly, but `uniq file.txt` alone often doesn't remove all duplicates?**
> A: `uniq` only removes ADJACENT duplicate lines — if identical lines are scattered non-consecutively throughout the file, plain `uniq` won't catch them. Sorting first brings all identical lines next to each other, so the subsequent `uniq` can then correctly identify and remove every duplicate, not just consecutive ones.

**Q3: What's wrong with `command 2>&1 > output.log`, and how should it be written correctly?**
> A: Redirection is processed left to right. At the point `2>&1` executes, stdout is still pointing at the terminal, so this makes stderr also point at the terminal — then `> output.log` redirects stdout to the file afterward, leaving stderr still going to the terminal instead of the file. The correct order is `command > output.log 2>&1`, so stdout is redirected to the file FIRST, and then stderr is told to follow stdout to that same destination.

**Q4: Why would `sudo command > /etc/protected_file` fail even when run with sudo, and how do you work around it?**
> A: The `>` redirection is set up by your CURRENT shell, which is not running as root, before `sudo` even executes the command — so your unprivileged shell is the one trying (and failing) to open the protected file for writing. The workaround is `command | sudo tee /etc/protected_file`, since `tee` itself runs with sudo privileges and performs the actual privileged write, rather than relying on shell-level redirection.

**Q5: Why is `xargs` needed at all — why not just pipe directly into a command like `rm`?**
> A: Many commands, including `rm`, expect their targets as command-line ARGUMENTS, not as piped standard input — `find . -name "*.tmp" | rm` doesn't work because `rm` doesn't read filenames from stdin at all. `xargs` converts piped input into actual command-line arguments for the next command, bridging that gap.

**Q6: What's the safe way to combine `find` and `xargs` when filenames might contain spaces?**
> A: `find . -name "*.txt" -print0 | xargs -0 rm` — `-print0` separates filenames using a NULL character instead of whitespace or newlines, and `xargs -0` reads that format correctly. Without this, a plain `find ... | xargs rm` can misinterpret a single filename containing a space as multiple separate arguments, causing incorrect or failed deletions.

**Q7: Why is a cron job that works fine when run manually sometimes failing silently when actually scheduled?**
> A: Cron jobs execute in a minimal environment that doesn't inherit your interactive shell's `$PATH` or other environment variables, and doesn't run from your usual working directory. A script relying on relative paths or commands only reachable via your personal PATH can fail under cron even though it works perfectly when you run it manually. The fix is using absolute paths for the interpreter, the script itself, and any commands/files it references, and explicitly redirecting output to a log file since cron failures aren't otherwise visible anywhere.

**Q8: What's the difference between `cron`, `at`, and systemd timers?**
> A: `cron` handles RECURRING scheduled tasks (daily, weekly, etc.). `at` schedules a single ONE-TIME future execution. systemd timers are the modern systemd-native alternative to both, offering tighter integration with `journalctl` for logging and features like `Persistent=true`, which ensures a missed scheduled run (e.g., because the machine was off) still executes once the system comes back up, rather than silently skipping it like cron would.

**Q9: What's the practical difference between `.bashrc` and `.bash_profile`, and why does it matter?**
> A: `.bashrc` runs for every new interactive, non-login shell — like opening a new terminal tab. `.bash_profile` runs specifically for LOGIN shells, like SSH-ing into a server. This matters because environment variables or settings placed only in `.bash_profile` won't automatically be available in a plain new terminal tab unless `.bash_profile` explicitly sources `.bashrc`, which is why many default setups chain the two together to keep behavior consistent.

**Q10: What's the difference between setting a regular shell variable and using `export`?**
> A: A regular variable (`MY_VAR=value`) exists only within the current shell and is NOT passed down to any child process or script launched from that shell. `export MY_VAR=value` promotes it to an actual environment variable, which IS inherited by child processes. This is a common source of confusion when a script unexpectedly can't see a variable — usually because it was set without `export`.

**Q11: What happens if you accidentally run `export PATH=/home/arjun/scripts` instead of appending to PATH?**
> A: This completely OVERWRITES the existing PATH rather than adding to it, meaning the shell can no longer find standard commands like `ls`, `cd`, or even `sudo` in their usual locations, since those directories are no longer part of PATH at all. The correct approach is always `export PATH=$PATH:/home/arjun/scripts` (or prepending with `$PATH` at the end), preserving the existing directories while adding the new one.

**Q12: Give a real example of the 3-2-1 backup rule in practice.**
> A: 3 total copies of your data: the original production data, a local backup copy, and an off-site/cloud copy. 2 different storage media: for example, local disk storage and a cloud object storage service like S3. 1 copy stored off-site: the cloud copy, ensuring that a physical disaster affecting the primary location (fire, theft, hardware failure) doesn't also destroy every backup copy simultaneously.
