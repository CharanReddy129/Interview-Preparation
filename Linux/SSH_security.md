# SSH and Security Basics

---

# SSH

## 1. What SSH Actually Is

SSH (Secure Shell) is an encrypted protocol for securely accessing and managing remote systems over a network — it's THE primary way DevOps engineers interact with remote servers (cloud VMs, on-prem servers, containers), replacing insecure legacy tools like Telnet and rlogin which transmitted everything, including passwords, in plaintext.

```bash
ssh username@remote_host              # basic connection
ssh username@remote_host -p 2222         # connect on a NON-default port (default SSH port is 22)
ssh -v username@remote_host                 # verbose mode — shows the full connection/handshake process, invaluable for debugging connection issues
```

---

## 2. SSH Key Pairs — How They Actually Work

SSH supports two authentication methods: **password-based** (simpler, but weaker — vulnerable to brute-force, and passwords can be intercepted/guessed) and **key-based** (far more secure, and the standard for any real production environment).

### The core concept: asymmetric cryptography
- You generate a **key PAIR**: a **private key** (stays on YOUR machine, NEVER shared) and a **public key** (safe to share, gets placed on servers you want to access).
- The server, holding your public key, can verify that whoever is connecting possesses the matching private key — WITHOUT the private key ever being transmitted over the network.
- This is fundamentally more secure than passwords: even if the public key is intercepted, it's useless without the private key, and there's no password to brute-force or phish.

### `ssh-keygen` — Generate a key pair
```bash
ssh-keygen                              # interactive — prompts for save location and passphrase
ssh-keygen -t rsa -b 4096                  # explicitly generate RSA, 4096-bit (strong, widely compatible)
ssh-keygen -t ed25519                         # generate using Ed25519 — modern, faster, and considered more secure than RSA at equivalent strength; increasingly the recommended default
ssh-keygen -t ed25519 -C "arjun@laptop"          # -C adds a COMMENT (typically email/identifier) to help identify the key later, especially useful when managing multiple keys
```

This creates two files, by default in `~/.ssh/`:

- `id_ed25519` (or `id_rsa`) — the **PRIVATE** key — must be kept secret, never shared, never committed to Git
- `id_ed25519.pub` (or `id_rsa.pub`) — the **PUBLIC** key — safe to share/copy to servers.

**Passphrase (frequently asked about):** During generation, you're prompted for an optional passphrase that encrypts the private key file itself on disk. Without one, anyone who steals the private key file can use it directly. With a passphrase, they'd also need to know it to decrypt and use the key — an important extra layer of protection, especially on laptops that could be lost or stolen.

### `ssh-copy-id` — Easily copy your public key to a remote server

```bash
ssh-copy-id username@remote_host          # copies your PUBLIC key to the server's ~/.ssh/authorized_keys, setting correct permissions automatically
ssh-copy-id -i ~/.ssh/id_ed25519.pub username@remote_host   # specify which key explicitly, if you have multiple
```

This is the fast, reliable way to set up key-based auth — it handles creating `~/.ssh/` on the remote server if needed and appends your public key to `authorized_keys` with the correct permissions, avoiding common manual mistakes.

**Manual equivalent (good to understand what's happening under the hood):**

```bash
cat ~/.ssh/id_ed25519.pub | ssh username@remote_host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### `~/.ssh/authorized_keys` (on the SERVER side)

This file lists every public key permitted to log in as that user. Each line is one public key.

```bash
chmod 700 ~/.ssh                        # SSH will often REFUSE to work if permissions are too open
chmod 600 ~/.ssh/authorized_keys
```

**Important real-world gotcha:** SSH is strict about permissions on `~/.ssh` and its contents — if they're too permissive (e.g., group/world-writable), SSH may silently refuse to use key-based auth for security reasons, and the error messages aren't always obvious. This trips people up constantly when setting up new servers.

---

## 3. `~/.ssh/config` — Simplify Connections

Instead of typing long commands repeatedly, you can define connection shortcuts:

```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.50
    User arjun
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

Host prod-db
    HostName db.example.com
    User deploy
    IdentityFile ~/.ssh/prod_key
    ProxyJump bastion-host
```

Now instead of `ssh -p 2222 -i ~/.ssh/id_ed25519 arjun@192.168.1.50`, you just run:

```bash
ssh myserver
```

**`ProxyJump` (worth knowing exists):** Lets you route through an intermediate "jump host"/bastion server to reach a server that isn't directly reachable from your machine — very common in cloud environments where production servers sit in a private subnet, only reachable via a bastion host with a public IP.

---

## 4. `known_hosts` — Server Identity Verification

The FIRST time you connect to a new host, SSH shows a fingerprint and asks you to confirm you trust it:

```bash
The authenticity of host 'example.com (1.2.3.4)' can't be established.
ED25519 key fingerprint is SHA256:abc123...
Are you sure you want to continue connecting (yes/no)?
```

Once accepted, that server's public key fingerprint gets stored in `~/.ssh/known_hosts` on your machine. On future connections, SSH automatically compares the server's presented key against this stored record.

**Why this matters (frequently asked security concept):** This protects against **man-in-the-middle (MITM) attacks** — if someone intercepts your connection and tries to impersonate the server, their key won't match what's stored in `known_hosts`, and you'll get a scary warning: `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` This is SSH refusing to connect because the server's identity doesn't match what you previously verified and trusted.

```bash
cat ~/.ssh/known_hosts                    # view stored host keys
ssh-keygen -R hostname                       # remove a specific host's entry (needed if a server's key legitimately changed, e.g., after a reinstall)
```
**Important nuance:** A "host key changed" warning is NOT always an attack — it can legitimately happen after a server is rebuilt/reinstalled, or its SSH host keys are regenerated. But you should always verify through another trusted channel before blindly accepting a changed key, rather than assuming it's automatically fine.

---

## 5. SCP and SFTP — Transferring Files Over SSH

### `scp` — Secure Copy (simple file transfer over SSH)
```bash
scp file.txt username@remote_host:/path/to/destination/     # copy a LOCAL file TO a remote server
scp username@remote_host:/path/to/file.txt ./                  # copy FROM a remote server TO local
scp -r folder/ username@remote_host:/path/to/destination/         # recursive, for directories
scp -P 2222 file.txt username@remote_host:/path/                     # specify a non-default port (note: CAPITAL -P for scp, lowercase -p for ssh — a classic gotcha)
scp file.txt user@host1:/path/ user@host2:/path/                        # not directly supported this way — scp is point-to-point; for multi-host copying you'd script it or use rsync
```

### `sftp` — SSH File Transfer Protocol (interactive file browser over SSH)
```bash
sftp username@remote_host
# Inside sftp session:
#   ls / lls        = list remote / list local
#   cd / lcd          = change remote dir / change local dir
#   get file.txt         = download a file
#   put file.txt            = upload a file
#   exit                       = quit
```

### `scp` vs `sftp` (occasionally asked)
| | `scp` | `sftp` |
|---|---|---|
| Style | One-shot command-line copy | Interactive session, browse remote filesystem |
| Best for | Quick, scriptable single transfers | Browsing, multiple operations in one session, more control |
| Resume support | No (older scp implementations) | Yes, generally more robust for interrupted transfers |

**Modern note worth mentioning:** `rsync` (over SSH) is often preferred over both for anything beyond simple one-off transfers, since it only transfers the DIFFERENCES between source and destination (efficient for repeated syncs/backups) and supports resuming interrupted transfers cleanly:
```bash
rsync -avz -e ssh /local/folder/ username@remote_host:/remote/folder/
```

---

# Security Basics

## 6. File Permission Hardening

Revisiting permissions specifically through a security lens:
- **Never use `chmod 777`** on anything in a real environment — it means ANY user on the system can read, write, AND execute the file, which is a textbook security misconfiguration flagged in any audit.
- **Audit for unexpected SUID binaries regularly**: `find / -perm -4000 -type f 2>/dev/null` — an unexpected or unnecessary SUID binary is a common privilege escalation vector if it has a vulnerability.
- **Principle of least privilege**: give users and processes only the minimum permissions needed to do their job — nothing more. This applies at every level: file permissions, sudo access, group memberships, and service accounts (this connects directly to why services should run as dedicated non-root users, from Topic 4).
- **Secure sensitive files tightly**: SSH private keys (`600`), `/etc/shadow` (`600`/`640`, root-only), application secrets/config files containing credentials.

---

## 7. SELinux / AppArmor — Conceptual Awareness

These are **Mandatory Access Control (MAC)** systems — an ADDITIONAL security layer on top of standard Linux permissions (which are technically called **Discretionary Access Control, DAC**).

### The core difference from normal permissions (the actual insight interviewers want)
> "Standard Linux permissions (chmod/chown) are discretionary — the file OWNER decides who can access it. SELinux and AppArmor add mandatory, system-wide policies that the ADMINISTRATOR defines and that even the file owner can't override. Even if a file has permissive standard permissions, SELinux/AppArmor can still block access if it violates the defined security policy — this is why a process can sometimes be denied access to a file it technically has normal Unix permission to read."

### SELinux (common on RHEL/CentOS/Fedora)
- Uses **labels/contexts** attached to every file and process, and policies define exactly which labeled processes can interact with which labeled resources
- Has distinct modes:
  - `Enforcing` — actively blocks policy violations
  - `Permissive` — logs violations but doesn't block them (useful for testing/debugging policy issues)
  - `Disabled` — SELinux completely off
```bash
sestatus                    # check current SELinux status/mode
getenforce                     # quick check: just shows current mode
sudo setenforce 0                 # temporarily switch to Permissive (0) — useful for troubleshooting "is SELinux blocking this?"
sudo setenforce 1                    # switch back to Enforcing (1)
ls -Z file.txt                          # view a file's SELinux context/label
```
**Real-world troubleshooting insight (genuinely useful to mention):** "If something works fine when SELinux is set to Permissive but fails in Enforcing mode, that confirms SELinux policy is the cause of the block — not a standard permissions issue. From there, you'd examine `/var/log/audit/audit.log` for the specific denial and either adjust the policy or use tools like `audit2allow` to generate an appropriate policy exception, rather than just disabling SELinux entirely, which is generally considered bad practice in production."

### AppArmor (common on Ubuntu/Debian)
- Simpler, path-based (rather than label-based) MAC system — profiles are tied to specific application paths rather than requiring the more complex labeling system SELinux uses
- Generally considered easier to configure than SELinux, at some cost of granularity
```bash
sudo aa-status              # check AppArmor status and loaded profiles
sudo aa-complain /path/to/profile   # set a profile to "complain" mode (log only, like SELinux's Permissive)
sudo aa-enforce /path/to/profile      # set a profile to enforce mode
```

**Interview-safe summary if this comes up and you haven't used it hands-on:** "I understand SELinux and AppArmor conceptually as Mandatory Access Control systems that add a policy-based security layer beyond standard file permissions — SELinux is label-based and common on RHEL, AppArmor is path-based and common on Ubuntu. I haven't deeply configured policies myself yet, but I know that if something is unexpectedly denied despite correct standard permissions, checking whether SELinux/AppArmor is enforcing a policy is one of the first things to investigate, often via the audit logs or by temporarily testing in permissive/complain mode."

---

## 8. Basic Server Hardening Checklist

### Disable root login over SSH
Editing `/etc/ssh/sshd_config`:
```
PermitRootLogin no
```
**Why this matters:** Root is a known, guessable username that exists on virtually every Linux system, making it a constant target for brute-force attacks. Disabling direct root SSH login forces attackers (and legitimate admins) to log in as a regular user first, then `sudo` for elevated actions — this also creates a clearer audit trail of WHO performed privileged actions, rather than everything just showing as "root."

### Enforce key-based authentication only (disable password auth)
```
PasswordAuthentication no
PubkeyAuthentication yes
```
**Why this matters:** Removes the entire attack surface of password brute-forcing/guessing over SSH entirely — if there's no password to try, that attack vector simply doesn't exist. This is considered close to a baseline requirement for any real production server.

### Change the default SSH port (minor "security through obscurity" measure)
```
Port 2222
```
**Honest interview-worthy:** This doesn't provide REAL security on its own — it mainly reduces noise from automated bots scanning the default port 22, rather than stopping a targeted attacker who would simply port-scan to find it. Worth mentioning this nuance rather than overselling it, since interviewers may specifically probe whether you understand "security through obscurity" isn't real security by itself.

### Other common hardening steps worth knowing
```bash
sudo systemctl restart sshd          # apply sshd_config changes (always restart the SSH service after editing this file)
```

- **Fail2ban**: monitors logs (like `auth.log`) for repeated failed login attempts and automatically bans offending IPs temporarily via firewall rules — a very commonly deployed tool for reducing brute-force attempts
- **Keep the system updated**: regularly applying security patches (`apt upgrade`/`yum update`) closes known vulnerabilities
- **Firewall rules**: only open the ports actually needed (connects directly to Topic 10 — firewall basics)
- **Regular SUID/permission audits**: as covered above

**Important safety note before actually applying these settings on a real server (genuinely worth saying in an interview to show good judgment):** "Before disabling password authentication or root login on a real server, I'd make absolutely sure key-based auth is already working correctly and tested in a separate session — locking yourself out of a remote server with no other access method is a classic, painful mistake. I'd keep an existing SSH session open while testing changes in a new one, so I have a fallback if something's misconfigured."

---

## Quick Reference Cheat Sheet

```bash
ssh-keygen -t ed25519 -C "you@email"      # generate a modern key pair
ssh-copy-id user@host                        # copy public key to a server
ssh user@host                                   # connect
ssh -v user@host                                  # verbose, for debugging connection issues
cat ~/.ssh/config                                    # connection shortcuts
cat ~/.ssh/known_hosts                                  # trusted host fingerprints
ssh-keygen -R hostname                                     # remove a stale host entry

scp file.txt user@host:/path/          # copy file to remote
scp -r folder/ user@host:/path/           # copy directory to remote
sftp user@host                               # interactive file transfer session
rsync -avz -e ssh src/ user@host:/dst/          # efficient sync (only transfers diffs)

chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys   # correct SSH permission hardening
find / -perm -4000 -type f 2>/dev/null                    # audit SUID binaries

getenforce / sestatus              # check SELinux mode
sudo setenforce 0                     # temporarily set Permissive, for troubleshooting
sudo aa-status                           # check AppArmor status

# In /etc/ssh/sshd_config, then: sudo systemctl restart sshd
PermitRootLogin no
PasswordAuthentication no
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: Explain how SSH key-based authentication actually works, at a conceptual level.**
> A: It relies on asymmetric cryptography — a key pair consisting of a private key that stays only on my machine, and a public key that I place on servers I want to access. The server uses the public key to verify that whoever is connecting possesses the matching private key, without that private key ever being transmitted over the network. This is more secure than passwords because there's nothing transmittable to intercept or brute-force — possession of the actual private key file is required.

**Q2: What's the purpose of a passphrase on an SSH key, and what does it actually protect against?**
> A: The passphrase encrypts the private key file itself while it's stored on disk. Without one, anyone who gains access to that private key file (through a stolen laptop, a misconfigured backup, or a compromised machine) could use it directly to authenticate as you. With a passphrase, they'd also need to know it to decrypt and actually use the key, adding a meaningful extra layer of protection.

**Q3: What is `known_hosts`, and what real security problem does it protect against?**
> A: `known_hosts` stores the fingerprints of servers you've previously connected to and verified. On future connections, SSH compares the server's presented key against this record — if they don't match, you get a strong warning. This protects against man-in-the-middle attacks, where someone might try to intercept your connection and impersonate the real server; their key wouldn't match what's already trusted and stored.

**Q4: If you get a "REMOTE HOST IDENTIFICATION HAS CHANGED" warning, does that always mean you're under attack?**
> A: Not necessarily — it can legitimately happen if a server was rebuilt, reinstalled, or had its SSH host keys regenerated for a valid reason. However, you shouldn't just blindly dismiss it and reconnect either; the safe approach is to verify through another trusted channel (like checking with whoever manages that server) that the change was expected, before removing the old entry with `ssh-keygen -R` and reconnecting.

**Q5: Why is disabling root SSH login and enforcing key-based-only authentication considered baseline server hardening?**
> A: "root" is a universally known, guessable username present on nearly every Linux system, making it a constant brute-force target — disabling direct root login forces a two-step process (login as a regular user, then sudo), which also creates a clearer audit trail of who performed privileged actions. Enforcing key-based auth entirely removes password brute-forcing as a viable attack vector, since there's no password to guess at all — only possession of the correct private key grants access.

**Q6: What's the fundamental difference between standard Linux file permissions and SELinux/AppArmor?**
> A: Standard permissions (chmod/chown) are Discretionary Access Control — the file's OWNER decides who can access it. SELinux and AppArmor are Mandatory Access Control systems that add an additional, policy-based layer defined by the administrator, which even the file owner can't override. This means SELinux/AppArmor can still deny access to a file even if standard Unix permissions would normally allow it, if the action violates the defined security policy.

**Q7: A file has correct read permissions, but a process still can't access it on an RHEL server. What might be going on, and how would you investigate?**
> A: This is a classic SELinux scenario — even with correct standard permissions, SELinux's mandatory access control policy could still be blocking the access based on security context/labels. I'd check `getenforce` to confirm SELinux is in Enforcing mode, temporarily switch to `setenforce 0` (Permissive) to confirm whether that resolves the issue, and if so, check `/var/log/audit/audit.log` for the specific denial to understand and properly address the underlying policy issue rather than leaving SELinux disabled long-term.

**Q8: Why is changing the default SSH port (from 22 to something else) not considered real security?**
> A: It's often described as "security through obscurity" — it mainly reduces noise from automated bots that scan the default port 22 specifically, but it does nothing to stop a targeted attacker who simply runs a full port scan to discover whatever port SSH is actually listening on. It can reduce log noise and low-effort automated attempts, but it shouldn't be relied upon as an actual security control on its own — proper measures like key-based auth and firewalls matter far more.

**Q9: What's the difference between `scp` and `rsync`, and why might you prefer `rsync` for regular backups?**
> A: `scp` performs a full, one-shot copy every time you run it, regardless of what's already at the destination. `rsync` compares source and destination and only transfers the actual DIFFERENCES, making repeated syncs much more efficient — especially valuable for regular backups of large datasets where most of the data hasn't changed since the last run. `rsync` also supports resuming interrupted transfers more robustly.

**Q10: Before disabling password authentication on a production SSH server, what precaution should you take?**
> A: Make absolutely sure key-based authentication is already fully working and tested first — ideally by successfully logging in with a key in a NEW separate session while keeping your CURRENT session open as a fallback. If you disable password auth and something's misconfigured with the keys, you could lock yourself (and everyone else) out of the server entirely, with no remaining way back in without out-of-band console access (like a cloud provider's web-based console).
