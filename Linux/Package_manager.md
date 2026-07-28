# Package Management

---

## 1. Why Package Managers Exist

Instead of manually downloading, compiling, and configuring every piece of software (and manually tracking dependencies), package managers automate:

- Downloading software from trusted repositories
- Resolving and installing **dependencies** automatically
- Tracking installed versions for updates/removal
- Verifying package integrity (checksums/signatures)

Two layers exist in most systems:

- **Low-level tool**: works with individual package files directly (`.deb` or `.rpm`) — `dpkg` (Debian) / `rpm` (RHEL)
- **High-level tool**: works with remote repositories, resolves dependencies automatically — `apt` (Debian) / `yum`/`dnf` (RHEL)

**Interview-relevant relationship:** `apt` is essentially a smarter wrapper around `dpkg` that also handles fetching from repos and dependency resolution; `dpkg` alone will install a package but won't automatically fetch/resolve missing dependencies for you. Same relationship exists between `yum`/`dnf` and `rpm`.

---

## 2. Debian-Based Systems (Ubuntu, Debian) — `apt` & `dpkg`

### `apt` — High-level package manager (modern, recommended)

```bash
sudo apt update                    # refresh the local package index/list from repositories (does NOT install/upgrade anything itself)
sudo apt upgrade                     # upgrade all installed packages to latest available versions
sudo apt install nginx                 # install a package (and its dependencies automatically)
sudo apt install nginx=1.18.0-0ubuntu1   # install a SPECIFIC version
sudo apt remove nginx                     # remove package, KEEP config files
sudo apt purge nginx                        # remove package AND its config files
sudo apt autoremove                           # remove packages that were installed as dependencies but are no longer needed by anything
sudo apt search nginx                            # search for a package by name/description
apt show nginx                                      # show detailed info about a package (version, dependencies, description)
apt list --installed                                   # list all installed packages
apt list --upgradable                                     # list packages with available updates
```

**`apt update` vs `apt upgrade` — critical, commonly confused:**
> "`apt update` refreshes the LOCAL package index — it tells your system what versions are currently available in the repositories, but installs nothing. `apt upgrade` actually installs newer versions of already-installed packages, based on that index. You typically run `apt update && apt upgrade` together — update first so upgrade knows what's actually available."

### `dpkg` — Low-level package tool (works directly with `.deb` files)

```bash
sudo dpkg -i package.deb          # install a LOCAL .deb file directly (no dependency resolution!)
dpkg -l                             # list installed packages
dpkg -l | grep nginx                  # check if a specific package is installed
dpkg -L nginx                           # list all FILES installed by a package
dpkg -S /usr/bin/nginx                    # find which package owns a specific file (reverse lookup)
sudo dpkg -r nginx                          # remove a package (no dependency handling)
sudo apt install -f                            # fix broken dependencies after a dpkg install failure (common combo)
```

### Repository config

```bash
cat /etc/apt/sources.list                    # main repository list
ls /etc/apt/sources.list.d/                     # additional repo files (PPAs, third-party repos)
sudo add-apt-repository ppa:something/ppa           # add a PPA (Personal Package Archive) - Ubuntu-specific
```

---

## 3. RHEL-Based Systems (CentOS, RHEL, Amazon Linux, Fedora) — `yum`/`dnf` & `rpm`

### `yum` / `dnf` — High-level package manager

`dnf` is the modern replacement for `yum` (used in RHEL 8+, Fedora, newer Amazon Linux) — same core commands work for both, `dnf` is just faster and has better dependency resolution.

```bash
sudo yum update                  # or: sudo dnf update  — update all packages (combines "refresh index" + "upgrade" in one, unlike apt's two-step)
sudo yum install nginx              # install a package
sudo yum remove nginx                  # remove a package
sudo yum search nginx                     # search for a package
yum info nginx                               # detailed package info
yum list installed                              # list installed packages
yum list installed | grep nginx                   # check if specific package installed
sudo yum autoremove                                  # remove unneeded dependency packages
yum check-update                                        # check what updates are available WITHOUT installing
sudo yum clean all                                         # clear cached package data
```

**Interview-relevant distinction from apt:** Unlike Debian's two-step `apt update && apt upgrade`, `yum update` handles refreshing the repo metadata AND upgrading in a single command by default.

### `rpm` — Low-level package tool (works directly with `.rpm` files)

```bash
sudo rpm -ivh package.rpm         # install (i), verbose (v), show hash progress (h)
rpm -qa                             # query ALL installed packages
rpm -qa | grep nginx                  # check if specific package installed
rpm -qi nginx                           # query detailed info about installed package
rpm -ql nginx                             # list files installed by a package
rpm -qf /usr/sbin/nginx                     # find which package owns a specific file
sudo rpm -e nginx                              # erase/remove a package (no dependency handling)
sudo rpm -Uvh package.rpm                         # upgrade a package from a local file
```

### Repository Config

```bash
ls /etc/yum.repos.d/           # repo config files (.repo files)
cat /etc/yum.repos.d/example.repo
```

---

## 4. Quick Comparison Table: Debian vs RHEL

| Task | Debian (apt/dpkg) | RHEL (yum/dnf/rpm) |
| --- | --- | --- |
| Refresh package index | `apt update` | (built into `yum update`) |
| Upgrade all packages | `apt upgrade` | `yum update` |
| Install package | `apt install nginx` | `yum install nginx` |
| Remove package (keep config) | `apt remove nginx` | `yum remove nginx` |
| Remove package + config | `apt purge nginx` | `rpm -e nginx` (config handling varies) |
| Install local package file | `dpkg -i pkg.deb` | `rpm -ivh pkg.rpm` |
| List installed packages | `dpkg -l` / `apt list --installed` | `rpm -qa` / `yum list installed` |
| Find which package owns a file | `dpkg -S /path/file` | `rpm -qf /path/file` |
| List files from a package | `dpkg -L pkgname` | `rpm -ql pkgname` |
| Package file extension | `.deb` | `.rpm` |
| Repo config location | `/etc/apt/sources.list` | `/etc/yum.repos.d/` |

---

## 5. Alpine Linux — `apk` (Quick Reference)

Alpine is extremely common as a **Docker base image** because of its tiny size (often ~5MB vs hundreds of MB for Ubuntu/Debian images), using `musl libc` instead of `glibc` and BusyBox instead of full GNU coreutils.

```bash
apk update                        # refresh package index
apk upgrade                          # upgrade all installed packages
apk add nginx                           # install a package
apk add --no-cache nginx                   # install WITHOUT caching the index locally — very common in Dockerfiles to keep image size minimal
apk del nginx                                 # remove/delete a package
apk search nginx                                 # search for a package
apk info nginx                                      # show package info
apk info -L nginx                                      # list files installed by a package
apk list --installed                                      # list installed packages
```

**Why `--no-cache` matters in Dockerfiles (genuinely interview-relevant for anyone doing containers):**

```dockerfile
# Bad practice - leaves cache bloat in the image layer
RUN apk update && apk add nginx

# Good practice - no separate update step, no cache left behind, smaller image
RUN apk add --no-cache nginx
```

This is a very common real Dockerfile optimization question, and knowing `--no-cache` shows genuine hands-on container awareness beyond just theory.

**One more Alpine-specific gotcha worth knowing:** Because Alpine uses `musl libc` instead of `glibc`, some compiled binaries built for standard Linux distros (which expect `glibc`) can fail to run on Alpine — this is a real, common source of "works on Ubuntu, breaks in the Alpine-based Docker container" bugs.

---

## 6. Compiling From Source — Basics

Sometimes software isn't available via any package manager, or you need a specific unavailable version/custom build flags — in which case you compile from source.

### The classic three-step pattern (C/C++ projects using Autotools)

```bash
./configure          # checks your system for required dependencies/libraries, generates a Makefile tailored to your system
make                    # actually compiles the source code into binaries, using the generated Makefile
sudo make install         # copies the compiled binaries/libraries into system directories (commonly /usr/local/bin, /usr/local/lib)
```

### What each step does (commonly asked to explain)

- **`./configure`**: A script that inspects your environment — checking for compilers, libraries, and dependencies — and generates a `Makefile` customized for your specific system. Fails here usually means a missing dependency/dev library.
- **`make`**: Reads the `Makefile` and actually compiles source code into executable binaries/object files, using the rules defined there. This is the actual "build" step.
- **`make install`**: Copies the freshly compiled binaries, libraries, and man pages into their proper system locations (typically under `/usr/local/`, which is the conventional location for manually-installed, non-package-manager software — this keeps it separate from package-manager-installed software in `/usr/bin`).

### Common supporting steps

```bash
sudo apt install build-essential          # Debian: installs gcc, make, and other core build tools
sudo yum groupinstall "Development Tools"    # RHEL: equivalent bundle of build tools
tar -xzvf source.tar.gz                        # extract the source tarball first (typically how source is distributed)
cd source-folder/
./configure && make && sudo make install         # the classic one-liner
```

### Uninstalling something compiled from source

```bash
sudo make uninstall     # only works IF the Makefile explicitly supports this target — not guaranteed to exist
```

**Important (worth mentioning in interviews):** Unlike `apt remove` or `yum remove`, which cleanly track and remove every file a package installed, compiling from source generally has NO reliable built-in uninstall unless the Makefile specifically provides a `make uninstall` target — this is why package managers are strongly preferred for anything long-term/production, and compiling from source is more of a last resort.

---

## Quick Reference Cheat Sheet

```bash
# Debian/Ubuntu
apt update && apt upgrade
apt install nginx
apt remove / apt purge nginx
dpkg -l
dpkg -S /path/to/file

# RHEL/CentOS/Amazon Linux
yum update  (or dnf update)
yum install nginx
yum remove nginx
rpm -qa
rpm -qf /path/to/file

# Alpine
apk update && apk upgrade
apk add --no-cache nginx
apk del nginx
apk info -L nginx

# Compile from source
tar -xzvf source.tar.gz
cd source/
./configure
make
sudo make install
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: What's the difference between `apt update` and `apt upgrade`?**
> A: `apt update` refreshes the local package index from the configured repositories — it tells your system what versions ARE available, but doesn't install anything. `apt upgrade` then actually installs newer versions of already-installed packages, based on that refreshed index. You need to run `update` before `upgrade` reflects the latest available versions, which is why they're typically chained together.

**Q2: What's the relationship between `apt` and `dpkg`?**
> A: `dpkg` is the low-level tool that directly installs/manages individual `.deb` package files, but it has no awareness of remote repositories and won't automatically resolve or fetch missing dependencies. `apt` is a higher-level tool built on top of `dpkg` that adds repository management and automatic dependency resolution — so `apt install` will pull in whatever dependencies are needed, while a raw `dpkg -i` on a package with missing dependencies will fail, requiring a manual `apt install -f` to fix.

**Q3: How would you find out which package a specific file on the system came from?**
> A: On Debian-based systems: `dpkg -S /path/to/file`. On RHEL-based systems: `rpm -qf /path/to/file`. Both do a reverse lookup, useful for troubleshooting — for example, figuring out which package owns a config file or binary you're investigating.

**Q4: Why is Alpine Linux so commonly used as a base image for Docker containers?**
> A: Alpine is extremely lightweight, often just a few megabytes compared to hundreds of megabytes for a full Ubuntu or Debian image, because it uses `musl libc` and BusyBox instead of the full GNU toolchain. Smaller base images mean faster image pulls, smaller storage footprint, and a reduced attack surface — all valuable in containerized environments where you're often running many container instances.

**Q5: Why would you use `apk add --no-cache` instead of `apk update && apk add` in a Dockerfile?**
> A: `--no-cache` fetches the package index directly for that install without saving a persistent local package cache, avoiding leftover cache data in that image layer. This keeps the resulting Docker image smaller, since Docker layers are immutable and any cache left behind from a separate `apk update` step would permanently bloat that layer even if you later delete it in a subsequent RUN command.

**Q6: What does `./configure && make && make install` actually do, step by step?**
> A: `./configure` inspects your system for the compilers, libraries, and dependencies needed, and generates a `Makefile` tailored to your specific environment. `make` reads that Makefile and actually compiles the source code into binaries. `make install` then copies those compiled binaries and supporting files into system directories, typically under `/usr/local/`, which is the conventional separate location for manually-built software, distinct from package-manager-installed software.

**Q7: Why is compiling from source generally discouraged for production servers compared to using a package manager?**
> A: Source-compiled software usually has no reliable, automatic uninstall or update mechanism — unless the project's Makefile specifically includes a `make uninstall` target, there's no clean way to remove all installed files later. Package managers track every installed file, handle dependency resolution, and support clean upgrades/removals. Compiling from source also means you're responsible for manually tracking security updates for that software, rather than getting them automatically through the package manager's update cycle.

**Q8: Why might a binary compiled on Ubuntu fail to run inside an Alpine-based Docker container?**
> A: Alpine uses `musl libc` instead of the `glibc` C library that most mainstream distros (including Ubuntu) use. Binaries compiled against `glibc` often have dependencies or expectations that `musl` doesn't satisfy, causing them to fail to run inside an Alpine container. This is a genuinely common real-world "works on my machine but breaks in the container" bug, especially with compiled language runtimes or native extensions.

**Q9: On a RHEL-based system, how would you check what updates are available without actually installing them?**
> A: `yum check-update` (or `dnf check-update`) lists available package updates without modifying anything on the system, letting you review what would change before committing to `yum update`.

**Q10: What's the equivalent of Debian's `apt purge` on an RHEL-based system?**
> A: There isn't a perfectly identical single-flag equivalent — `rpm -e` removes the package, but configuration file handling around removal differs based on how the RPM package itself was built (some RPMs explicitly mark config files to be preserved or removed on erase, via `%config` directives in the spec file), unlike apt's explicit distinct `remove` vs `purge` commands.
