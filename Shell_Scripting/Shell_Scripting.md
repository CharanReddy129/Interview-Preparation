# Shell Scripting — Complete Notes

---

## 1. Script Basics — Shebang, Permissions, Execution

### The Shebang line

```bash
#!/bin/bash
```

This MUST be the very first line of any script. It tells the OS which interpreter to use to run the script, regardless of what shell you're currently in.

```bash
#!/bin/bash              # most common — explicitly use bash
#!/bin/sh                   # POSIX-compliant shell — more portable, but fewer bash-specific features available
#!/usr/bin/env bash            # finds bash wherever it is in PATH — more portable across systems where bash might not be at /bin/bash
```

### Making a script executable and running it

```bash
chmod +x script.sh        # add execute permission (Topic 3 — connects directly back to permissions)
./script.sh                  # run it (the ./ is required unless the directory is in PATH — Topic 17)
bash script.sh                  # alternative: explicitly run with bash, ignoring the shebang and execute permission entirely
sh script.sh                       # run with sh interpreter specifically
```

**Interview-relevant nuance:** If you run `bash script.sh` directly, the shebang line is actually IGNORED — you're explicitly telling the shell which interpreter to use, overriding whatever the shebang says. The shebang only matters when you execute the script directly via `./script.sh`.

---

## 2. Variables

```bash
name="Arjun"                  # NO spaces around = (this is a very common beginner error — "name = Arjun" FAILS)
echo "Hello, $name"              # use $ to reference a variable's value
echo "Hello, ${name}"               # curly braces — safer, especially when concatenating: "${name}_backup" vs "$name_backup" (the latter tries to read a variable literally called name_backup!)

readonly PI=3.14              # create a CONSTANT — cannot be changed or unset after this
unset name                       # remove a variable entirely

# Command substitution — capture a command's OUTPUT into a variable
current_date=$(date +%Y-%m-%d)
echo "Today is $current_date"
files_count=$(ls | wc -l)          # modern syntax, preferred
files_count=`ls | wc -l`             # OLD backtick syntax — still works but considered legacy/harder to nest, avoid in new scripts
```

### Variable quoting — a genuinely critical, frequently tested concept

```bash
name="John Smith"
echo $name              # WITHOUT quotes — word-splits on spaces, can behave unexpectedly, especially in loops/conditions
echo "$name"                # WITH quotes — treated as a single value, preserves spaces correctly — ALWAYS quote variables unless you have a specific reason not to
```

**Why this matters practically (a real, common bug source):** `if [ $var == "" ]` can break/error if `$var` is empty or contains spaces, because without quotes the shell tries to word-split it before evaluating, potentially leaving an invalid empty comparison. `if [ "$var" == "" ]` is the safe version — always quote variables inside conditionals and most other contexts.

---

## 3. Special Variables & Command-Line Arguments

```bash
#!/bin/bash
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "All arguments as one string: $*"
echo "All arguments as separate items: $@"
echo "Number of arguments: $#"
echo "Exit status of last command: $?"
echo "Process ID of this script: $$"
echo "PID of last background command: $!"
```

Run as: `./script.sh hello world` → `$1` = "hello", `$2` = "world", `$#` = 2

### `$*` vs `$@` — commonly tested distinction

> "Both represent all arguments, but they behave differently when quoted. `"$*"` treats ALL arguments as a SINGLE combined string. `"$@"` treats each argument as a SEPARATE, distinct item — preserving individual arguments even if they contain spaces. This matters most when looping through arguments: `for arg in "$@"` correctly handles each argument separately (including ones with spaces), while `for arg in "$*"` would treat everything as one blob."

---

## 4. Reading User Input

```bash
read name                    # wait for user input, store it in $name
echo "Hello, $name"

read -p "Enter your name: " name    # -p shows a PROMPT on the same line
read -s -p "Enter password: " pass    # -s = SILENT (hides input — for passwords)
read -t 10 -p "Answer in 10 sec: " ans   # -t = TIMEOUT in seconds, moves on if no input given in time

read -a arr -p "Enter numbers: "    # -a reads input into an ARRAY (space-separated words become array elements)
```

---

## 5. Conditionals: `if`, `elif`, `else`

```bash
if [ "$age" -gt 18 ]; then
    echo "Adult"
elif [ "$age" -eq 18 ]; then
    echo "Exactly 18"
else
    echo "Minor"
fi
```

### Comparison operators — MUST memorize (different for numbers vs strings!)

| Numeric | Meaning | String | Meaning |
| --- | --- | --- | --- |
| `-eq` | equal | `==` or `=` | equal |
| `-ne` | not equal | `!=` | not equal |
| `-gt` | greater than | `<` | less than (needs `[[ ]]` or escaping in `[ ]`) |
| `-lt` | less than | `>` | greater than (same caveat) |
| `-ge` | greater or equal | `-z` | string is EMPTY |
| `-le` | less or equal | `-n` | string is NOT empty |

**Critical, extremely commonly tested mistake:** Using `-eq`/`-gt` for STRINGS or `==`/`>` for NUMBERS gives wrong or unexpected results. `[ "5" -gt "10" ]` correctly evaluates numerically (5 is NOT greater than 10 → false). But `[ "5" > "10" ]` compares them as STRINGS lexicographically, and "5" > "10" is actually TRUE as a string comparison (since '5' > '1' character-wise) — a classic bug if you use the wrong operator type.

### File test operators (extremely commonly used in real scripts)

```bash
if [ -f "/path/file.txt" ]; then echo "It's a regular file"; fi
if [ -d "/path/folder" ]; then echo "It's a directory"; fi
if [ -e "/path/thing" ]; then echo "Path exists (file OR directory)"; fi
if [ -r "/path/file" ]; then echo "File is readable"; fi
if [ -w "/path/file" ]; then echo "File is writable"; fi
if [ -x "/path/file" ]; then echo "File is executable"; fi
if [ -s "/path/file" ]; then echo "File exists AND is not empty (size > 0)"; fi
```

### `[ ]` vs `[[ ]]` — commonly asked

```bash
[ "$name" == "Arjun" ]        # POSIX test — works everywhere, but more fragile (word-splitting issues if unquoted, limited pattern matching)
[[ "$name" == "Arjun" ]]         # bash-specific EXTENDED test — safer (no word-splitting surprises even without quotes), supports pattern matching and && / || directly inside
[[ "$file" == *.txt ]]              # [[ ]] supports WILDCARD pattern matching directly — [ ] doesn't support this
```

**Interview-ready answer:** "`[[ ]]` is a bash-specific improvement over the POSIX `[ ]` test — it's generally safer because it doesn't word-split unquoted variables the same way, and it supports extra features like pattern matching and regex. I'd use `[[ ]]` in bash scripts by default, but `[ ]` if I specifically need POSIX portability across different shells like `sh` or `dash`."

### Logical operators

```bash
if [ "$age" -gt 18 ] && [ "$citizen" == "yes" ]; then echo "Eligible"; fi
if [ "$x" -eq 1 ] || [ "$y" -eq 1 ]; then echo "At least one is 1"; fi
if [[ "$age" -gt 18 && "$citizen" == "yes" ]]; then echo "Eligible"; fi   # && and || work directly INSIDE [[ ]], but not inside single [ ]
```

### `case` statement — cleaner alternative to long if/elif chains

```bash
case "$1" in
    start)
        echo "Starting service..."
        ;;
    stop)
        echo "Stopping service..."
        ;;
    restart)
        echo "Restarting service..."
        ;;
    *)
        echo "Usage: $0 {start|stop|restart}"
        exit 1
        ;;
esac
```

**Real-world use:** This is EXACTLY the pattern used in old-style init scripts (`/etc/init.d/servicename start/stop/restart`) — genuinely worth knowing since it directly mirrors real infrastructure scripts.

---

## 6. Loops: `for`, `while`, `until`

### `for` loop

```bash
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

for i in {1..10}; do echo "$i"; done            # range shorthand
for i in {1..10..2}; do echo "$i"; done            # range with STEP (1, 3, 5, 7, 9)

for file in *.txt; do                                 # loop through matching files (wildcard expansion)
    echo "Processing $file"
done

for ((i=0; i<5; i++)); do                                # C-style for loop
    echo "Index: $i"
done

for arg in "$@"; do                                         # loop through script arguments SAFELY (quoted!)
    echo "Argument: $arg"
done
```

### `while` loop — runs WHILE a condition is true

```bash
count=1
while [ $count -le 5 ]; do
    echo "Count: $count"
    ((count++))
done

while read -r line; do             # extremely common pattern — read a file LINE BY LINE
    echo "Line: $line"
done < input.txt
```

**Why `read -r` (with `-r`) matters:** `-r` prevents backslashes in the input from being interpreted as escape characters — without it, a line containing a backslash could get silently mangled. Using `read -r` is considered best practice by default when reading lines.

### `until` loop — runs UNTIL a condition becomes true (opposite of while)

```bash
count=1
until [ $count -gt 5 ]; do
    echo "Count: $count"
    ((count++))
done
```

### Loop control

```bash
break              # exit the loop entirely, immediately
continue              # skip to the NEXT iteration, skipping remaining code in this pass
```

---

## 7. Functions

```bash
greet() {
    echo "Hello, $1!"          # function arguments work just like script arguments — $1, $2, etc., LOCAL to the function call
}
greet "Arjun"                    # call it — prints "Hello, Arjun!"

# Function with a return value (via exit-code-style return, 0-255 only)
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0        # 0 = success/true, by shell convention
    else
        return 1           # any non-zero = failure/false, by shell convention
    fi
}
if is_even 4; then echo "Even"; else echo "Odd"; fi

# Function that "returns" an actual VALUE (not just an exit code) — via echo + command substitution
get_square() {
    echo $(( $1 * $1 ))
}
result=$(get_square 5)
echo "Square: $result"
```

**Critical concept (frequently misunderstood):** Bash functions can't directly "return" arbitrary values the way functions do in most programming languages — `return` in bash only returns an exit status CODE (0-255), meant to indicate success/failure, not an arbitrary value. To get an actual computed value out of a function, the standard pattern is to `echo` the value from inside the function and capture it via command substitution (`result=$(function_name)`) from the caller.

### `local` — scope variables to the function

```bash
my_func() {
    local temp="only visible inside this function"
    echo "$temp"
}
```

Without `local`, variables set inside a function are GLOBAL by default in bash — they leak out and could unexpectedly overwrite variables elsewhere in the script. Using `local` explicitly scopes them to the function.

---

## 8. Exit Codes & Error Handling

```bash
exit 0        # success (by convention)
exit 1           # generic failure
echo $?             # check the exit status of the PREVIOUS command

command1 && command2    # run command2 ONLY IF command1 SUCCEEDED (exit code 0)
command1 || command2       # run command2 ONLY IF command1 FAILED (non-zero exit code)

mkdir /some/folder && cd /some/folder && echo "Created and moved in"    # classic chained success pattern
ping -c 1 google.com || echo "Network appears down"                        # classic chained failure-handling pattern
```

### Checking command success explicitly

```bash
if command; then
    echo "Succeeded"
else
    echo "Failed"
fi

# Or explicitly checking $?
some_command
if [ $? -eq 0 ]; then
    echo "Succeeded"
else
    echo "Failed"
fi
```

### `set -e`, `set -u`, `set -o pipefail` — genuinely important production script safety flags

```bash
#!/bin/bash
set -e            # EXIT IMMEDIATELY if ANY command fails (non-zero exit code) — without this, bash by default just continues to the NEXT line even after a failure!
set -u               # treat UNSET/undefined variables as an ERROR instead of silently substituting an empty string — catches typos in variable names
set -o pipefail         # makes a PIPELINE's exit status reflect failure if ANY command in the pipe fails, not just the LAST one

set -euo pipefail          # the common combined shorthand for all three at once — genuinely a best-practice opening line for serious scripts
```

**Why this matters enormously in real production scripts (a genuinely high-value interview point):** "By default, bash scripts continue executing even after a command fails partway through, which can cause a script to keep running with bad/missing data and cause much worse damage than if it had just stopped. `set -e` makes the script halt immediately on any failure. `set -u` catches a very common bug — referencing a variable that was never set (often due to a typo), which by default just silently becomes an empty string rather than erroring, potentially causing dangerous behavior like an accidental `rm -rf /` if a path variable was empty. `set -o pipefail` fixes a subtle issue where a pipeline like `false | true` normally reports SUCCESS (since only the LAST command's exit code matters by default), even though an earlier command in the pipe actually failed."

---

## 9. Arrays

```bash
fruits=("apple" "banana" "cherry")     # declare an array
echo "${fruits[0]}"                       # access first element (apple)
echo "${fruits[@]}"                          # ALL elements
echo "${#fruits[@]}"                            # LENGTH of the array (number of elements)

fruits+=("mango")                     # append a new element
unset fruits[1]                          # remove a specific element by index

for fruit in "${fruits[@]}"; do        # loop through array elements SAFELY
    echo "$fruit"
done
```

---

## 10. Real-World Script Examples (Genuinely Interview-Worthy)

### Example 1: Disk space alert script

```bash
#!/bin/bash
set -euo pipefail

THRESHOLD=80
USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "WARNING: Disk usage is at ${USAGE}%, exceeding threshold of ${THRESHOLD}%"
    exit 1
else
    echo "Disk usage OK: ${USAGE}%"
    exit 0
fi
```

### Example 2: Simple backup script with timestamp and logging

```bash
#!/bin/bash
set -euo pipefail

SOURCE_DIR="/home/arjun/data"
BACKUP_DIR="/home/arjun/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/home/arjun/logs/backup.log"

mkdir -p "$BACKUP_DIR"

tar -czf "${BACKUP_DIR}/backup_${TIMESTAMP}.tar.gz" "$SOURCE_DIR" \
    && echo "$(date): Backup successful — backup_${TIMESTAMP}.tar.gz" >> "$LOG_FILE" \
    || { echo "$(date): Backup FAILED" >> "$LOG_FILE"; exit 1; }

# Delete backups older than 7 days
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +7 -delete
```

### Example 3: User creation script with validation

```bash
#!/bin/bash
set -euo pipefail

if [ "$#" -ne 1 ]; then
    echo "Usage: $0 <username>"
    exit 1
fi

USERNAME="$1"

if id "$USERNAME" &>/dev/null; then
    echo "User $USERNAME already exists"
    exit 1
fi

sudo useradd -m -s /bin/bash "$USERNAME"
echo "User $USERNAME created successfully"
```

### Example 4: Log monitoring / health check script

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="/var/log/app.log"
ERROR_COUNT=$(grep -c "ERROR" "$LOG_FILE" || true)   # || true prevents set -e from exiting if grep finds ZERO matches (grep returns exit code 1 when no match found, which set -e would otherwise treat as a script failure!)

if [ "$ERROR_COUNT" -gt 10 ]; then
    echo "ALERT: $ERROR_COUNT errors found in $LOG_FILE"
    exit 1
fi
echo "Log check OK — $ERROR_COUNT errors found"
```

**Worth explaining the `|| true` trick if asked:** `grep` returns exit code 1 (failure) when it finds ZERO matches — this is normal grep behavior, not an actual error, but under `set -e` it would incorrectly halt the whole script. Appending `|| true` ensures the overall command always reports success regardless of whether grep found matches, so the script continues to actually check the COUNT rather than dying prematurely on a clean log file.

---

## Quick Reference Cheat Sheet

```bash
#!/bin/bash
set -euo pipefail                # safety flags — put at the top of every serious script

var="value"                   # no spaces around =
echo "$var"                      # always quote variables

$1 $2 $#  $@ $*  $? $$ $!            # positional args, arg count, all args (safe/unsafe), exit code, PID, last bg PID

if [ "$a" -eq "$b" ]; then ... fi       # numeric comparison
if [ "$a" == "$b" ]; then ... fi           # string comparison
if [ -f file ]; then ... fi                   # file exists check
[[ "$a" == "$b" ]]                                # bash-safe test, supports patterns

for i in {1..5}; do ... done              # range for loop
while read -r line; do ... done < file       # read file line by line
until [ cond ]; do ... done                     # loop until true

func_name() { ... ; }                # function definition
local var="x"                           # scope variable to function
return 0 / return 1                        # exit code from function (0-255 only)
result=$(func_name)                           # capture an actual "returned" value via echo

command1 && command2               # run second only if first succeeds
command1 || command2                   # run second only if first fails
exit 0 / exit 1                           # script exit code
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What does the shebang line do, and does it matter if you run a script with `bash script.sh` instead of `./script.sh`?**
> A: The shebang (`#!/bin/bash`) tells the OS which interpreter to use when the script is executed directly. It only takes effect when you run the script directly (`./script.sh`) — if you instead run `bash script.sh`, you're explicitly specifying bash as the interpreter yourself, so the shebang line is effectively ignored in that case.

**Q2: Why should you always quote variables in bash, like `"$var"` instead of `$var`?**
> A: Without quotes, bash performs word-splitting and globbing on the variable's value before using it — a variable containing spaces could get split into multiple separate arguments unexpectedly, or an empty variable could disappear entirely from a comparison, causing a syntax error in something like `[ $var == "" ]`. Quoting (`"$var"`) preserves the value exactly as a single unit, avoiding these subtle, common bugs.

**Q3: What's the difference between `$*` and `$@`?**
> A: Both represent all script/function arguments, but `"$*"` combines them into a SINGLE string, while `"$@"` preserves each argument as a SEPARATE, distinct item. This matters most in loops — `for arg in "$@"` correctly processes each argument individually, even if one contains spaces, whereas `"$*"` would treat everything as one combined blob.

**Q4: What does `set -e` do, and why might a script behave dangerously WITHOUT it?**
> A: `set -e` causes the script to exit immediately if any command returns a non-zero (failure) exit code. Without it, bash's default behavior is to simply continue to the next line even after a command fails — which can mean a script keeps executing subsequent steps based on incomplete or failed prior steps, potentially causing much more damage than if it had just stopped immediately at the point of failure.

**Q5: What does `set -u` protect against?**
> A: It causes bash to treat any reference to an UNSET/undefined variable as an error, rather than silently substituting an empty string. This is valuable because a typo in a variable name would otherwise go completely unnoticed — the variable would just silently evaluate to nothing, which can be genuinely dangerous in something like a path used in `rm -rf "$SOME_TYPO'D_VAR"/*`, which could unintentionally become `rm -rf /*` if the variable resolves to empty.

**Q6: Can a bash function directly return a computed value, like a sum or a string?**
> A: Not directly in the way most programming languages work — bash's `return` statement only sets an exit status CODE, limited to 0-255, meant to represent success/failure rather than carry an arbitrary value. To get an actual computed value out of a function, the standard pattern is to `echo` the value inside the function and capture it in the caller using command substitution, like `result=$(my_function)`.

**Q7: What's the difference between `[ ]` and `[[ ]]` in bash conditionals?**
> A: `[ ]` is the POSIX-standard test command, available in any POSIX shell, but it's more fragile — unquoted variables can cause word-splitting issues, and it doesn't support pattern matching. `[[ ]]` is a bash-specific extended test that's generally safer (avoids those word-splitting surprises even without quotes) and supports additional features like wildcard pattern matching and using `&&`/`||` directly inside the brackets. I'd default to `[[ ]]` in bash scripts unless I specifically need POSIX portability across different shells.

**Q8: Why might `[ "5" -gt "10" ]` and `[ "5" > "10" ]` give different, seemingly contradictory results?**
> A: `-gt` performs a NUMERIC comparison, correctly determining that 5 is not greater than 10 (false). `>` inside `[ ]` performs a STRING/lexicographic comparison instead — and as strings, "5" is considered greater than "10" because the character '5' comes after '1' alphabetically/lexicographically. Using the wrong operator type for the kind of comparison you actually need is a classic, easy-to-miss scripting bug.

**Q9: In the log-checking script pattern, why would you append `|| true` after a `grep -c` command under `set -e`?**
> A: `grep` returns a non-zero exit code (specifically 1) when it finds ZERO matches — this is normal, expected grep behavior for "nothing found," not an actual error condition. But under `set -e`, that non-zero exit code would cause the entire script to halt prematurely, even on a perfectly healthy log file with no errors to report. Appending `|| true` ensures the command's overall reported exit status is always success, letting the script continue on to actually evaluate the resulting count.

**Q10: Why does using `local` inside a function matter, and what happens if you forget it?**
> A: By default, variables set inside a bash function are GLOBAL, not scoped to that function — without `local`, a variable set inside one function could unexpectedly overwrite a variable of the same name used elsewhere in the script, causing subtle, hard-to-trace bugs. `local` explicitly scopes the variable to only exist within that function call, preventing this kind of accidental interference.

**Q11: Walk me through what `command1 && command2` versus `command1 || command2` actually do.**
> A: `&&` runs the second command ONLY IF the first one succeeds (exit code 0) — used for chaining steps that should only proceed if the prior step worked, like `mkdir folder && cd folder`. `||` runs the second command ONLY IF the first one FAILS (non-zero exit code) — commonly used for fallback/error-handling, like `ping -c 1 host || echo "host unreachable"`.

**Q12: You need to read a file line by line in a script — what's the standard, safe pattern, and why the `-r` flag?**
> A: `while read -r line; do ... done < file.txt`. The `-r` flag tells `read` not to interpret backslashes in the input as escape characters — without it, a line containing a literal backslash could get silently altered/mangled during reading. Using `-r` by default is considered best practice for reliably reading raw file content line by line.
