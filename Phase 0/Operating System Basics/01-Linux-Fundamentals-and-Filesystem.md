# Linux Fundamentals & Filesystem Hierarchy

> **Phase 0 — Computer Science Fundamentals → 0.3 Operating System Basics**
> Goal: Understand why Linux matters for backend engineers, how the shell works, and the Linux filesystem hierarchy (where everything lives).

---

## 0. The Big Picture

As a backend engineer, **you will deploy on Linux.** Servers, Docker containers, Kubernetes nodes, and most cloud instances run Linux. Even if you develop on Windows/macOS, your code runs in production on Linux — so being comfortable at the Linux command line is non-negotiable.

```
Your laptop (dev)  -->  Docker image (Linux)  -->  Server / K8s node (Linux)
```

> This section is practical: the goal is fluency with the shell, the filesystem, permissions, processes, and scripting — the daily tools of operating and debugging real systems.

---

## 1. What Linux Is

- **Linux** is an open-source, Unix-like **operating system kernel** (by Linus Torvalds), plus the surrounding GNU tools = a full OS.
- **Distributions (distros)** package the kernel + tools: **Ubuntu/Debian**, **Red Hat/CentOS/Fedora**, **Alpine** (tiny, common in Docker), Amazon Linux, etc.
- The **kernel** manages hardware, processes, memory, and I/O (recall Phase 0.1: processes, virtual memory, scheduling — the kernel does all this).

### 1.1 Why backend engineers use Linux
| Reason | Why it matters |
|--------|----------------|
| **Servers run Linux** | The vast majority of production servers |
| **Containers are Linux** | Docker images are typically Linux-based (Phase 10) |
| **Free & open-source** | No licensing; runs everywhere |
| **Powerful CLI** | Automate, script, and debug efficiently |
| **Stability & security** | Designed for long-running, multi-user servers |

---

## 2. The Shell (Bash)

The **shell** is a program that interprets your typed commands and runs them. **Bash** (Bourne Again SHell) is the most common; `zsh` and others exist.

```
$ command [options] [arguments]
$ ls -l /home          # command=ls, option=-l, argument=/home
```
- `$` = normal user prompt; `#` = root (superuser) prompt.
- A **terminal** is the window; the **shell** is the program running inside it interpreting commands.

### 2.1 How a command runs (recap from Phase 0.1)
When you type a command, the shell **`fork()`s** a child process and **`exec()`s** the program (recall the "how programs are loaded" note). It then waits for it to finish (unless you background it with `&`).

### 2.2 Anatomy of commands
| Part | Example | Meaning |
|------|---------|---------|
| Command | `ls` | The program to run |
| Options/flags | `-l`, `-a`, `--help` | Modify behavior (short `-l` / long `--all`) |
| Arguments | `/home` | What to operate on |
| `man` / `--help` | `man ls`, `ls --help` | Read documentation for a command |

---

## 3. The Linux Filesystem Hierarchy

Linux organizes everything under a **single root `/`** (no drive letters like `C:`). Key directories you must know:

```
/                  <- root of everything
├── bin            <- essential user commands (ls, cp, cat)
├── sbin           <- system admin commands
├── boot           <- bootloader & kernel
├── dev            <- device files (disks, terminals)
├── etc            <- system-wide CONFIGURATION files  ★
├── home           <- user home directories (/home/alice)  ★
├── lib            <- shared libraries
├── media / mnt    <- mount points for removable/temporary media
├── opt            <- optional / third-party software
├── proc           <- virtual: live process & kernel info (/proc/<pid>)
├── root           <- the root user's home directory
├── tmp            <- TEMPORARY files (cleared on reboot)  ★
├── usr            <- user programs & data (/usr/bin, /usr/local)  ★
└── var            <- VARIABLE data: logs, caches, spool (/var/log)  ★
```

### 3.1 The directories you'll use most (★)
| Directory | Contains | Why you care |
|-----------|----------|--------------|
| **`/etc`** | System & app **configuration** | Edit configs (nginx, systemd services, hosts) |
| **`/var`** | **Logs** (`/var/log`), caches, spools | `/var/log/...` is where you read app/system logs |
| **`/home`** | Per-user files (`/home/<user>`) | Your working files; `~` is shorthand for your home |
| **`/tmp`** | **Temporary** files (wiped on reboot) | Scratch space — don't store anything persistent |
| **`/usr`** | Installed programs/libraries | `/usr/local/bin` for locally installed tools |
| **`/opt`** | Optional 3rd-party apps | Some installs (e.g., a JDK) land here |
| **`/proc`** | Virtual files exposing kernel/process state | `/proc/<pid>` shows live process info |

### 3.2 Absolute vs relative paths
- **Absolute path:** starts from root — `/home/alice/app/config.yml`.
- **Relative path:** relative to the current directory — `app/config.yml`, `../logs`.
- Special shortcuts:
  | Symbol | Means |
  |--------|-------|
  | `/` | Root directory |
  | `~` | Current user's home (`/home/alice`) |
  | `.` | Current directory |
  | `..` | Parent directory |
  | `-` | Previous directory (with `cd -`) |

### 3.3 Everything is a file
A Unix philosophy: **almost everything is represented as a file** — regular files, directories, devices (`/dev/sda`), and even process info (`/proc`). This uniform interface is why the same tools (`cat`, `grep`) work on so many things.

### 3.4 Hidden files
Files starting with `.` are **hidden** (dotfiles) — config files like `.bashrc`, `.gitignore`, `.env`. Show them with `ls -a`.

---

## 4. Getting Help

| Command | Use |
|---------|-----|
| `man <cmd>` | Full manual page (`man grep`) |
| `<cmd> --help` | Quick usage summary |
| `tldr <cmd>` | Community examples (if installed) |
| `which <cmd>` | Show the path of a command's executable |
| `type <cmd>` | Show how a name resolves (alias/builtin/file) |
| `apropos <word>` | Search man pages by keyword |

---

## 5. Common Pitfalls / Misconceptions

| Misconception | Reality |
|---------------|---------|
| "Linux is like Windows with folders" | Single root `/`, no drive letters; case-sensitive paths |
| Editing files in `/tmp` expecting them to persist | `/tmp` is wiped on reboot |
| Looking for logs in random places | App/system logs live under `/var/log` |
| Confusing `/` (root dir) with `/root` (root user's home) | They're different |
| Case-insensitive filenames | Linux is **case-sensitive** (`File.txt` ≠ `file.txt`) |
| Spaces in filenames without quoting | Quote or escape (`"my file"` or `my\ file`) |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Deployment (Phase 10):** Docker images are Linux; you'll write Dockerfiles that copy to `/app`, set configs, and run your JAR on Linux.
- **Logs (Phase 9):** reading `/var/log` and container logs is core to debugging production.
- **Config (`/etc`):** systemd service files, nginx configs, environment files.
- **`/proc` & process tools** (next notes) underpin diagnosing CPU/memory issues (Phase 1.11, 13).
- **Filesystem knowledge** is needed for volumes, mounts, and persistent storage in Docker/K8s (Phase 10).

---

## 7. Quick Self-Check Questions

1. Why must backend engineers be comfortable with Linux?
2. What's the difference between a terminal and a shell?
3. What happens (process-wise) when you run a command? (Hint: fork/exec.)
4. What lives in `/etc`, `/var/log`, `/home`, `/tmp`, and `/proc`?
5. What's the difference between an absolute and a relative path?
6. What do `~`, `.`, `..`, and `-` refer to?
7. What does "everything is a file" mean in Unix?
8. How do you view hidden files, and what's an example dotfile?

---

## 8. Key Terms Glossary

- **Linux:** open-source Unix-like OS kernel + GNU tools.
- **Distribution (distro):** a packaged Linux OS (Ubuntu, Alpine, etc.).
- **Kernel:** core managing hardware, processes, memory, I/O.
- **Shell / Bash:** command interpreter.
- **Terminal:** the interface window running a shell.
- **Filesystem hierarchy:** the `/`-rooted directory tree.
- **`/etc` / `/var` / `/home` / `/tmp` / `/usr` / `/proc`:** config / variable data / users / temp / programs / kernel-process info.
- **Absolute vs relative path:** from root vs from current dir.
- **Dotfile:** hidden file starting with `.`.
- **`man` / `--help`:** documentation commands.

---

*This is the first note of **Section 0.3 — Operating System Basics**.*
*Next topic: **Essential Commands (navigation, files, text processing)**.*
