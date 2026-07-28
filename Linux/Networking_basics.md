# Networking Basics

---

## 1. Viewing Network Interfaces & IP Addresses

### `ip addr` (modern) — the current standard tool

```bash
ip addr                     # show all network interfaces and their IPs
ip addr show eth0              # details for a specific interface
ip a                              # shorthand, same as 'ip addr'
ip link                              # show interfaces WITHOUT IP details (just link/MAC layer info)
ip link set eth0 up                     # bring an interface up
ip link set eth0 down                      # bring an interface down
ip route                                      # show the routing table
ip route add default via 192.168.1.1              # add a default gateway route
```

### `ifconfig` — legacy tool (deprecated but still widely recognized/asked about)

```bash
ifconfig                # older equivalent of 'ip addr' — largely replaced by 'ip' on modern distros, may need net-tools package installed
```

**Interview-relevant fact:** `ifconfig` comes from the older `net-tools` package, which is deprecated on most modern distros in favor of `iproute2` (`ip` command). Knowing this transition happened, and that `ip` is now the standard, is itself a good signal of current knowledge in an interview.

---

## 2. Checking Connections & Ports: `netstat` / `ss`

### `ss` — modern, faster replacement for `netstat`

```bash
ss -tuln              # t=TCP, u=UDP, l=listening sockets only, n=numeric (don't resolve hostnames/service names) — THE most common combo
ss -tulnp                # add 'p' to also show the PROCESS/PID using each port — extremely useful for "what's using this port" troubleshooting
ss -a                       # show ALL sockets (listening + established connections)
ss -s                          # summary statistics of socket usage
```

### `netstat` — older tool, still very commonly referenced in interviews/legacy docs

```bash
netstat -tuln            # same idea as ss -tuln
netstat -tulnp               # same idea, with process info (needs sudo to see all processes)
netstat -r                      # show routing table (older equivalent of 'ip route')
```

**Interview-relevant fact:** Like `ifconfig`, `netstat` is part of the deprecated `net-tools` package, replaced by `ss` (from `iproute2`) on modern systems — `ss` is generally faster since it reads directly from kernel data structures rather than parsing `/proc` files the way `netstat` does. Still, `netstat` remains extremely commonly referenced, so know both.

### The classic "what's using port 8080" scenario

```bash
sudo ss -tulnp | grep :8080
# or
sudo lsof -i :8080
```
This exact scenario — "a port is already in use, find out what's using it" — is one of the most common practical troubleshooting questions asked in interviews.

---

## 3. Connectivity Testing: `ping`, `traceroute`

### `ping` — Test basic reachability using ICMP
```bash
ping google.com              # sends continuous ICMP echo requests until you Ctrl+C
ping -c 4 google.com            # send exactly 4 packets, then stop — better for scripting/quick checks
```
**What ping actually tells you:** whether a host is reachable and responding, and round-trip latency. It does NOT confirm a specific service/port is working — a host can respond to ping while the actual application (like a web server on port 80) is down, since ping operates at the ICMP/network layer, not the application layer.

### `traceroute` (or `tracepath`) — Show the network path/hops to a destination

```bash
traceroute google.com        # shows each router/hop the packet passes through on its way to the destination
```
Useful for diagnosing WHERE in the network path a connectivity problem or high latency is occurring — e.g., is the delay happening at your local network, your ISP, or somewhere further along the route.

**`ping` vs `traceroute` — quick distinction:** `ping` tells you IF you can reach a destination and how fast; `traceroute` tells you the PATH taken to get there, hop by hop, which helps pinpoint WHERE a failure or slowdown is occurring along the route.

---

## 4. Fetching Data: `curl`, `wget`

### `curl` — Versatile tool for making HTTP(S) requests (and other protocols), extremely common in DevOps for testing APIs

```bash
curl https://example.com                    # fetch and print the response body
curl -I https://example.com                    # HEAD request — headers ONLY, no body (quick way to check status/response headers)
curl -o output.html https://example.com           # save output to a file
curl -X POST https://api.example.com/data            # specify HTTP method
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" https://api.example.com/data   # POST with JSON body and headers
curl -v https://example.com                             # VERBOSE — shows the full request/response including headers, extremely useful for debugging
curl -s -o /dev/null -w "%{http_code}" https://example.com  # print ONLY the HTTP status code, useful in health-check scripts
```

### `wget` — Primarily for downloading files

```bash
wget https://example.com/file.zip           # download a file
wget -O newname.zip https://example.com/file.zip   # download and save with a specific filename
wget -c https://example.com/largefile.zip              # resume a partially downloaded file
```

### `curl` vs `wget` — commonly asked

| | `curl` | `wget` |
| --- | --- | --- |
| Primary strength | Flexible HTTP requests (GET/POST/PUT/DELETE), API testing, headers, auth | Simple, robust file downloading, including resuming interrupted downloads |
| Output by default | Prints to stdout | Saves to a file |
| Common DevOps use | Testing APIs, health checks in scripts | Downloading installers/packages/artifacts |

---

## 5. DNS Lookups: `dig`, `nslookup`

### `dig` — Detailed DNS lookup tool (preferred, more information)

```bash
dig google.com                  # full DNS query details — includes query time, DNS server used, TTL, etc.
dig google.com +short              # just the resolved IP, no extra details — quick and clean
dig google.com MX                     # query a SPECIFIC record type (MX = mail server records)
dig -x 8.8.8.8                           # REVERSE lookup — IP to hostname
```

### `nslookup` — Older, simpler alternative

```bash
nslookup google.com          # basic DNS lookup
```

**`dig` vs `nslookup`:** `dig` is generally preferred by professionals for its more detailed, scriptable output (great for troubleshooting DNS propagation, TTLs, specific record types); `nslookup` is simpler and more commonly what people learn first, but considered somewhat legacy in comparison.

---

## 6. Key Config Files: `/etc/hosts`, `/etc/resolv.conf`

### `/etc/hosts` — Manual, LOCAL hostname-to-IP mapping (checked BEFORE DNS)

```bash
cat /etc/hosts
# 127.0.0.1   localhost
# 192.168.1.50   myserver.local
```

Adding an entry here lets you manually resolve a hostname without needing DNS at all — commonly used for local development (pointing a domain to `127.0.0.1` or a local IP), testing, or overriding DNS temporarily for a specific host without touching actual DNS records.

**Resolution order (frequently asked):** By default, Linux checks `/etc/hosts` FIRST, and only falls back to actual DNS servers (as configured in `/etc/resolv.conf`) if no match is found there — this order is technically controlled by `/etc/nsswitch.conf`, but `/etc/hosts`-before-DNS is the default behavior almost everywhere.

### `/etc/resolv.conf` — DNS server configuration

```bash
cat /etc/resolv.conf
# nameserver 8.8.8.8
# nameserver 1.1.1.1
```
Lists the DNS servers the system queries when a hostname isn't found in `/etc/hosts`. Note: on many modern systems this file is auto-generated/managed by NetworkManager or systemd-resolved, so manually editing it may get overwritten.

---

## 7. Ports & Protocols

### TCP vs UDP (fundamental, always asked in some form)

| | TCP | UDP |
| --- | --- | --- |
| Connection | Connection-oriented (handshake required before data flows) | Connectionless (just sends, no handshake) |
| Reliability | Reliable — guarantees delivery, retransmits lost packets, ensures order | Unreliable — no guarantee of delivery or order |
| Speed | Slower (overhead from reliability guarantees) | Faster (minimal overhead) |
| Use cases | Web (HTTP/HTTPS), SSH, databases, email — anywhere data integrity matters | DNS queries, video streaming, VoIP, gaming — anywhere speed matters more than perfect reliability |

### Common ports to know cold (frequently tested)

| Port | Service |
| --- | --- |
| 20/21 | FTP (data/control) |
| 22 | SSH |
| 23 | Telnet (insecure, legacy) |
| 25 | SMTP (email sending) |
| 53 | DNS |
| 80 | HTTP |
| 443 | HTTPS |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 6379 | Redis |
| 8080 | Common alternate HTTP port (often used for dev servers, proxies) |
| 27017 | MongoDB |

```bash
cat /etc/services | grep ssh          # look up the standard port for a known service name
```

---

## 8. Firewall Basics

### `iptables` — Traditional, low-level Linux firewall tool (rules-based)

```bash
sudo iptables -L                       # list current rules
sudo iptables -L -v -n                    # verbose, numeric (no DNS lookups, faster/clearer)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT     # ALLOW incoming SSH (port 22)
sudo iptables -A INPUT -p tcp --dport 80 -j DROP          # BLOCK incoming HTTP (port 80)
sudo iptables -A INPUT -j DROP                                # DROP all other unmatched incoming traffic (must go LAST, after allow rules — order matters!)
```

**Critical concept — rule order matters:** `iptables` processes rules top to bottom, and the FIRST matching rule wins. A common real mistake is placing a broad DROP/REJECT rule before more specific ALLOW rules, which would block traffic that should have been allowed, since the DROP rule matches first and iptables stops evaluating further rules for that packet.

### `ufw` (Uncomplicated Firewall) — Simplified frontend for iptables, common on Ubuntu

```bash
sudo ufw status                    # check if enabled and current rules
sudo ufw enable                       # turn on the firewall
sudo ufw allow 22                        # allow SSH
sudo ufw allow 80/tcp                       # allow HTTP specifically over TCP
sudo ufw deny 23                               # deny telnet
sudo ufw delete allow 80                          # remove a specific rule
```

### `firewalld` — Common on RHEL/CentOS, uses the concept of "zones"
```bash
sudo firewall-cmd --state                     # check if running
sudo firewall-cmd --list-all                     # show current zone's rules
sudo firewall-cmd --add-port=8080/tcp --permanent   # open a port PERMANENTLY (survives reboot)
sudo firewall-cmd --reload                             # apply permanent changes (needed after --permanent changes)
```

**Important:** Forgetting `--permanent` means the rule only applies until the next reboot/reload; forgetting to `--reload` after adding a `--permanent` rule means it won't take effect immediately even though it's saved.

---

## Quick Reference Cheat Sheet

```bash
ip addr                            # view IPs/interfaces (modern)
ip route                              # routing table
ss -tulnp                                # listening ports + owning process (modern)
netstat -tulnp                              # same idea (legacy tool)
ping -c 4 host                                 # test reachability
traceroute host                                   # trace network path
curl -I url                                          # headers only
curl -v url                                             # verbose, full request/response
wget url                                                   # download a file
dig domain +short                                             # quick DNS lookup
cat /etc/hosts                                                    # local hostname overrides
cat /etc/resolv.conf                                                 # DNS servers used
sudo iptables -L -v -n                                                  # list firewall rules
sudo ufw allow 22                                                          # allow SSH via ufw
sudo firewall-cmd --add-port=8080/tcp --permanent                            # open port permanently (RHEL)
```

---

## INTERVIEW QUESTIONS & ANSWERS

**Q1: How would you find out what process is using a specific port, like 8080?**
> A: `sudo ss -tulnp | grep :8080` — this shows listening sockets along with the owning process name and PID. `sudo lsof -i :8080` is an equally common alternative that gives similar information. This is a very frequent real troubleshooting scenario, like when a service fails to start because "address already in use."

**Q2: What's the difference between TCP and UDP, and why would you choose one over the other?**
> A: TCP is connection-oriented and reliable — it establishes a handshake, guarantees delivery, and preserves order, at the cost of more overhead. UDP is connectionless and doesn't guarantee delivery or order, but is faster with minimal overhead. TCP is used where data integrity matters, like web traffic or database connections; UDP is used where speed matters more than perfect reliability, like DNS queries or video streaming, where a dropped packet is preferable to a delay.

**Q3: Does a successful `ping` guarantee that a web server on that host is working?**
> A: No — `ping` only confirms the host is reachable at the network/ICMP level and measures round-trip latency. It says nothing about whether a specific application or service, like a web server on port 80, is actually running and responding. A host can respond to ping while its web server is completely down, so you'd need something like `curl` against the actual service/port to confirm application-level availability.

**Q4: What's the difference between `curl` and `wget`, and when would you use each?**
> A: `curl` is more flexible for making various types of HTTP requests — GET, POST, PUT, custom headers, authentication — making it ideal for testing APIs or scripting health checks. `wget` is more focused on simple, robust file downloading, including the ability to resume interrupted downloads. In practice, I'd use `curl` for interacting with APIs or checking service status, and `wget` for pulling down files like installers or build artifacts.

**Q5: In what order does Linux resolve a hostname — `/etc/hosts` first, or DNS first?**
> A: By default, `/etc/hosts` is checked FIRST, and DNS (configured via `/etc/resolv.conf`) is only queried if no match is found there. This lookup order is technically controlled by `/etc/nsswitch.conf`, but hosts-before-DNS is the standard default behavior. This is why adding an entry to `/etc/hosts` can override or bypass actual DNS for testing purposes.

**Q6: Why does rule ORDER matter in `iptables`?**
> A: `iptables` evaluates rules top-to-bottom and stops at the FIRST matching rule for a given packet. If a broad DROP or REJECT rule is placed before more specific ALLOW rules, it will match and block traffic before the system ever reaches the intended ALLOW rule further down. This is why allow rules for specific, needed traffic should come first, with a catch-all deny/drop rule placed last.

**Q7: What's the practical difference between `ping` and `traceroute`?**
> A: `ping` tells you WHETHER you can reach a destination and how fast (round-trip latency), but nothing about the path taken. `traceroute` shows the actual PATH — each router/hop the packet passes through en route to the destination — which is useful for pinpointing exactly WHERE in the network a delay or failure is occurring, rather than just knowing that something is wrong.

**Q8: Why are `ifconfig` and `netstat` considered legacy tools, and what replaced them?**
> A: Both come from the older `net-tools` package, which has been largely deprecated on modern Linux distributions in favor of the `iproute2` package — `ip` replaces `ifconfig`, and `ss` replaces `netstat`. `ss` in particular is generally faster since it reads directly from kernel data structures, while `netstat` parses information from `/proc`. Both legacy tools are still widely recognized and often still installed, but modern documentation and scripts increasingly favor `ip`/`ss`.

**Q9: On RHEL using `firewalld`, you add a rule with `--permanent` but it doesn't seem to take effect immediately. Why?**
> A: The `--permanent` flag saves the rule to persist across reboots, but it doesn't apply it to the currently running firewall configuration immediately — you need to run `firewall-cmd --reload` afterward to actually activate the permanent rule in the live running configuration. Forgetting the reload step is a common reason a newly added rule appears saved but isn't actually working yet.

**Q10: What's the difference between port 80 and port 443, and what do they represent?**
> A: Port 80 is the standard port for HTTP (unencrypted web traffic), while port 443 is the standard port for HTTPS (HTTP over TLS/SSL, encrypted). Modern web traffic strongly favors 443/HTTPS for security, and many setups configure port 80 to simply redirect to 443 rather than serve unencrypted content directly.
