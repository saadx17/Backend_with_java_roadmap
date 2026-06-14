# Environment Variables, Networking Tools, Package Management & SSH

> **Phase 0 — Computer Science Fundamentals → 0.3 Operating System Basics**
> Goal: Master environment variables & PATH, the network/disk diagnostic commands, package managers (apt/yum), and remote access with SSH/SCP.

---

## 0. The Big Picture

This note covers the "operating a server" toolkit: configuring the environment, diagnosing network/disk issues, installing software, and connecting to remote machines — the daily reality of running backend systems on Linux.

---

## 1. Environment Variables & PATH

### 1.1 What environment variables are
**Environment variables** are named values available to processes — used for configuration (credentials, settings, paths) without hardcoding them.
```bash
echo $HOME              # /home/alice
echo $USER              # alice
echo $PATH              # directories searched for commands
printenv                # list all environment variables
env                     # same
```

### 1.2 Setting variables
```bash
NAME="value"            # shell variable (current shell only)
export NAME="value"     # ENVIRONMENT variable (inherited by child processes)
export DB_URL="jdbc:postgresql://localhost:5432/mydb"
unset NAME              # remove it
```
> **`export` is the key difference:** without `export`, the variable exists only in the current shell; with `export`, **child processes inherit it** (e.g., your Java app reads it). Recall Phase 0.1: child processes inherit the parent's environment via `fork`/`exec`.

### 1.3 Common environment variables
| Variable | Meaning |
|----------|---------|
| `PATH` | Colon-separated dirs searched for executables |
| `HOME` | Current user's home directory |
| `USER` | Current username |
| `PWD` | Current working directory |
| `SHELL` | The login shell |
| `LANG` | Locale/encoding (affects character handling — recall Phase 1.4) |
| `JAVA_HOME` | Where the JDK is installed (Java tooling needs this) |

### 1.4 PATH — how the shell finds commands
When you type `java`, the shell searches each directory in **`PATH`**, in order, for an executable named `java`:
```bash
echo $PATH      # /usr/local/bin:/usr/bin:/bin:...
which java      # shows which PATH entry was matched
```
Add a directory to PATH:
```bash
export PATH="$PATH:/opt/myapp/bin"     # append (note: include the old $PATH!)
```
> A classic error: "command not found" usually means the executable isn't in `PATH`. And `JAVA_HOME` + `PATH` setup is the first step of installing the JDK (Phase 1.1).

### 1.5 Making variables persistent
Variables set in a shell vanish when it closes. To persist them, add `export` lines to a startup file:
| File | When it loads |
|------|---------------|
| `~/.bashrc` | Each interactive non-login shell (most common for aliases/exports) |
| `~/.bash_profile` / `~/.profile` | Login shells |
| `/etc/environment` | System-wide variables |
```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-21' >> ~/.bashrc
source ~/.bashrc        # reload without re-login
```

> ⚠️ **Security:** environment variables are a common way to pass secrets to apps (12-factor config), but they can leak via `ps`/`/proc` and logs — for high-security needs use a secrets manager (Phase 15). Never hardcode secrets in scripts.

---

## 2. Networking Diagnostic Commands

These connect Phase 0.2 concepts (DNS, ports, HTTP) to practical troubleshooting.

### 2.1 HTTP clients
| Command | Use |
|---------|-----|
| `curl <url>` | Make HTTP requests (inspect APIs) ★ |
| `wget <url>` | Download files over HTTP |

```bash
curl https://api.example.com/users          # GET, print body
curl -i https://api.example.com             # include response headers
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"Sam"}' https://api.example.com/users   # POST JSON
curl -v https://example.com                 # verbose (see the whole request/response, TLS)
curl -o file.zip https://example.com/file   # save to a file
wget https://example.com/archive.tar.gz     # download
```
> `curl` is *the* tool for testing/debugging HTTP APIs from the command line (recall the HTTP note) — you'll use it constantly.

### 2.2 Connectivity & DNS
| Command | Use |
|---------|-----|
| `ping <host>` | Test reachability (ICMP) |
| `traceroute <host>` | Show the network path (hops) to a host |
| `dig <domain>` | Detailed DNS lookup (recall DNS note) |
| `nslookup <domain>` | Simpler DNS lookup |
| `host <domain>` | Quick DNS lookup |

```bash
ping google.com                  # is it reachable? round-trip time
traceroute google.com            # where does traffic go / where does it stall
dig example.com +short           # just the IP
nslookup example.com
```

### 2.3 Ports & connections
| Command | Use |
|---------|-----|
| `ss -tulpn` | Show listening ports & sockets (modern) ★ |
| `netstat -tulpn` | Same (older tool) |
| `lsof -i :8080` | Which process uses a port (recall process note) |

```bash
ss -tulpn                # t=tcp u=udp l=listening p=process n=numeric
ss -tulpn | grep 8080    # is my app listening on 8080?
```
> "Is my server actually listening on the port?" → `ss -tulpn`. "Can I reach another service?" → `curl`/`ping`/`telnet host port`.

---

## 3. Disk & Storage Commands

| Command | Use |
|---------|-----|
| `df -h` | Disk **f**ree space per filesystem (human-readable) ★ |
| `du -sh <dir>` | Disk **u**sage of a directory (summary) |
| `du -sh *` | Size of each item in the current dir |
| `mount` | Show mounted filesystems |
| `lsblk` | List block devices (disks/partitions) |

```bash
df -h                    # "is the disk full?" — common production issue
du -sh /var/log          # how big is the log directory?
du -sh * | sort -rh | head   # biggest items here
```
> ⚠️ **"Disk full" is a classic outage cause** (logs filling `/var/log`, etc.). `df -h` is the first check; `du` finds the culprit.

---

## 4. Package Management

Installing/updating software is done via your distro's **package manager**.

### 4.1 Debian/Ubuntu — `apt`
```bash
sudo apt update                 # refresh the package index (do first)
sudo apt upgrade                # upgrade installed packages
sudo apt install nginx          # install a package
sudo apt remove nginx           # remove
sudo apt search openjdk         # search
dpkg -l                         # list installed packages
```

### 4.2 Red Hat/CentOS/Fedora — `yum` / `dnf`
```bash
sudo yum install nginx          # (or dnf on newer systems)
sudo yum update
sudo yum remove nginx
```

| Distro family | Package manager | Package format |
|---------------|-----------------|----------------|
| Debian/Ubuntu | `apt` (`dpkg`) | `.deb` |
| Red Hat/CentOS/Fedora | `yum` / `dnf` (`rpm`) | `.rpm` |
| Alpine (Docker) | `apk` | `.apk` |

> In **Dockerfiles** (Phase 10) you'll run these to install dependencies — e.g., `RUN apt-get update && apt-get install -y curl`. Alpine's `apk` is common for small images.

---

## 5. SSH — Secure Remote Access

**SSH (Secure Shell)** lets you securely log into and run commands on a remote machine over an encrypted connection (port 22). It's how you access servers.

### 5.1 Connecting
```bash
ssh user@server.com                 # connect (prompts for password/key)
ssh user@203.0.113.5                # by IP
ssh -p 2222 user@host               # custom port
ssh -i ~/.ssh/mykey.pem user@host   # with a specific private key (e.g., AWS)
```

### 5.2 SSH key-based authentication (preferred over passwords)
```bash
ssh-keygen -t ed25519               # generate a key pair (private + public)
ssh-copy-id user@host               # install your PUBLIC key on the server
# now you can ssh in without a password (private key proves identity)
```
- **Key pair:** a **private key** (kept secret, `chmod 600` — recall permissions note) and a **public key** (placed on servers in `~/.ssh/authorized_keys`).
- The server challenges you; only your private key can answer → you're authenticated **without sending a password** (asymmetric crypto — recall the TLS note's public-key concept).
- ⚠️ SSH **refuses** private keys with loose permissions — keep them `600`.

### 5.3 SCP — copy files over SSH
```bash
scp file.txt user@host:/remote/path/      # local -> remote
scp user@host:/remote/file.txt ./         # remote -> local
scp -r mydir/ user@host:/remote/          # recursive (a directory)
```
- `rsync` is a faster, resumable alternative for large/repeated transfers: `rsync -avz mydir/ user@host:/remote/`.

### 5.4 SSH config (convenience)
Put host shortcuts in `~/.ssh/config`:
```
Host myserver
    HostName 203.0.113.5
    User alice
    Port 22
    IdentityFile ~/.ssh/mykey.pem
```
Then just `ssh myserver`.

---

## 6. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Setting a var without `export` (child can't see it) | `export` it for child processes |
| Overwriting PATH (`export PATH=/new`) | Always include `$PATH`: `export PATH="$PATH:/new"` |
| "command not found" | The binary isn't in PATH; check `which`/install it |
| SSH key with loose permissions | `chmod 600` the private key |
| Committing secrets/keys to git | Never; use env vars / secret managers, `.gitignore` |
| Ignoring `df -h` until disk is full | Monitor disk; rotate logs |
| Hardcoding secrets in scripts | Use environment variables / vault |

---

## 7. Connection to Backend / Spring (Why This Matters Later)

- **12-Factor config (Phase 5.2):** Spring Boot reads config from **environment variables** (`SPRING_DATASOURCE_URL`, etc.) — the standard way to configure apps across environments.
- **`JAVA_HOME`/`PATH`** are step one of setting up the JDK (Phase 1.1).
- **`curl`** is your daily API-testing/debugging tool (Phase 5, 7).
- **`ss`/`lsof`/`df`** are essential for production debugging — port conflicts, disk-full outages (Phase 13).
- **Package managers** appear in every **Dockerfile** (Phase 10).
- **SSH/SCP** are how you access servers and (with keys) how CI/CD deploys (Phase 10.3).
- **Secrets handling** ties to security (Phase 15) — env vars vs vaults.

---

## 8. Quick Self-Check Questions

1. What's the difference between a shell variable and an exported environment variable?
2. What is `PATH`, and what does "command not found" usually mean?
3. How do you persist an environment variable across sessions?
4. Write a `curl` command that POSTs JSON to an API.
5. How do you check which process is listening on port 8080? Which DNS tool gives just the IP?
6. What command checks free disk space, and what finds a directory's size?
7. What's the difference between `apt` and `yum`, and where do you use `apk`?
8. How does SSH key-based auth work, and why is it preferred over passwords?
9. How do you copy a directory to a remote server?

---

## 9. Key Terms Glossary

- **Environment variable:** named value available to processes.
- **`export`:** mark a variable for inheritance by child processes.
- **`PATH`:** directories searched for executables.
- **`JAVA_HOME`:** location of the JDK.
- **`~/.bashrc` / `/etc/environment`:** where to persist variables.
- **`curl` / `wget`:** HTTP client / file downloader.
- **`ping` / `traceroute` / `dig`:** reachability / path / DNS tools.
- **`ss` / `netstat` / `lsof`:** socket and port inspection.
- **`df` / `du`:** disk free / disk usage.
- **`apt` / `yum` / `apk`:** package managers (Debian / RHEL / Alpine).
- **SSH:** encrypted remote shell access (port 22).
- **SSH key pair:** private (secret) + public key for passwordless auth.
- **SCP / rsync:** copy files over SSH.

---

*Previous topic: **Process Management & Signals**.*
*Next topic: **Shell Scripting, Cron Jobs & systemd**.*
