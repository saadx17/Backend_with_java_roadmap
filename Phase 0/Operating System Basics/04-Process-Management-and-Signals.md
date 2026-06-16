# Process Management & Signals

> **Phase 0 — Computer Science Fundamentals → 0.3 Operating System Basics**
> Goal: Master managing processes on Linux — `ps`, `top`/`htop`, `kill`, background jobs (`&`, `nohup`), PIDs, signals, and zombie/orphan processes.

---

## 0. The Big Picture

A **process** is a running program (recall Phase 0.1: process vs thread, virtual memory). As a backend engineer you constantly **inspect, control, and troubleshoot** processes — your app, the database, stuck jobs, runaway CPU. This note covers the tools and concepts to do that.

```
Every running program = a process with a unique PID, an owner, CPU/memory usage, and a state.
```

---

## 1. Process Identifiers (PID, PPID)

| Term | Meaning |
|------|---------|
| **PID** | Process ID — a unique number per running process |
| **PPID** | Parent Process ID — the process that started this one |
| **PID 1** | The **init** process (`systemd`) — the ancestor of all processes |

- Processes form a **tree**: every process (except PID 1) has a parent that started it (via `fork()`/`exec()` — recall Phase 0.1).
- Your own shell's PID is in `$$`; the last background job's PID is in `$!`.

---

## 2. Viewing Processes — `ps`

`ps` shows a **snapshot** of current processes.
```bash
ps                    # processes in the current shell
ps aux                # ALL processes, detailed (most common)
ps -ef                # all processes, full format (alternative style)
ps aux | grep java    # find your Java app's process  ★
ps -p 1234            # info about a specific PID
```

### 2.1 Reading `ps aux` columns
```
USER  PID  %CPU  %MEM   VSZ    RSS   STAT  START  TIME  COMMAND
alice 1234 12.0  3.5  987654  72000  Sl    10:00  0:42  java -jar app.jar
```
| Column | Meaning |
|--------|---------|
| `USER` | Owner of the process |
| `PID` | Process ID |
| `%CPU` / `%MEM` | CPU / memory usage |
| `VSZ` / `RSS` | Virtual size / Resident (physical) memory (recall virtual memory note) |
| `STAT` | Process state (see §3) |
| `COMMAND` | The command that started it |

---

## 3. Process States

| State | Symbol | Meaning |
|-------|--------|---------|
| **Running/Runnable** | `R` | Executing or ready to execute |
| **Sleeping** | `S` | Waiting for an event (interruptible) |
| **Uninterruptible sleep** | `D` | Waiting on I/O (can't be interrupted) |
| **Stopped** | `T` | Suspended (e.g., Ctrl+Z) |
| **Zombie** | `Z` | Finished but not yet reaped by its parent (§7) |

(Recall Phase 0.1 thread lifecycle states — process states are analogous at the OS level.)

---

## 4. Live Monitoring — `top` / `htop`

`top` shows a **live, updating** view of processes sorted by resource usage:
```bash
top         # live process monitor (q to quit)
htop        # nicer, colored, interactive version (if installed)
```
- See **CPU%**, **memory**, **load average**, per-process usage in real time.
- In `top`: press `M` (sort by memory), `P` (sort by CPU), `k` (kill a process).
- **Load average** (e.g., `0.50, 0.75, 1.00`) = average number of processes wanting CPU over 1/5/15 minutes. Roughly, load ≈ number of CPU cores means fully utilized; much higher = overloaded (recall context-switching/thrashing, Phase 0.1).

> `top`/`htop` are your first stop when "the server is slow" — instantly shows what's eating CPU or memory.

---

## 5. Background Jobs — `&`, `jobs`, `fg`, `bg`, `nohup`

### 5.1 Running in the background with `&`
```bash
java -jar app.jar &      # start in the background; shell stays free
                         # prints the PID, e.g., [1] 1234
```

### 5.2 Managing jobs
| Command | Effect |
|---------|--------|
| `jobs` | List background jobs of this shell |
| `fg %1` | Bring job 1 to the **foreground** |
| `bg %1` | Resume a stopped job in the **background** |
| `Ctrl+Z` | Suspend the foreground job (state `T`) |
| `Ctrl+C` | Send SIGINT to terminate the foreground job |

### 5.3 `nohup` — survive logout
A background job started with `&` normally **dies when you close the terminal** (it receives SIGHUP). `nohup` detaches it so it keeps running:
```bash
nohup java -jar app.jar > app.log 2>&1 &    # keep running after logout, log to file
```
> `nohup ... &` is a quick way to keep a process alive — but for real services, use **systemd** (next note) or a container, not `nohup`.

---

## 6. Signals & `kill`

A **signal** is an asynchronous notification sent to a process to tell it something happened or to request an action (recall Phase 0.1: signals are an IPC mechanism). `kill` sends signals (despite the name, not only to "kill").

### 6.1 Common signals
| Signal | Number | Meaning | Catchable? |
|--------|--------|---------|------------|
| **SIGTERM** | 15 | **Polite** request to terminate (default) — lets the app clean up | ✅ Yes |
| **SIGKILL** | 9 | **Force** kill immediately — cannot be ignored | ❌ No |
| **SIGINT** | 2 | Interrupt (Ctrl+C) | ✅ Yes |
| **SIGHUP** | 1 | Terminal hangup / reload config | ✅ Yes |
| **SIGSTOP** | 19 | Suspend (Ctrl+Z is SIGTSTP) | ❌ No |
| **SIGCONT** | 18 | Resume a stopped process | ✅ |

### 6.2 Using `kill`
```bash
kill 1234              # send SIGTERM (15) — graceful, the default
kill -15 1234          # explicit SIGTERM
kill -9 1234           # SIGKILL — force kill (last resort)
kill -SIGTERM 1234     # by name
killall java           # kill all processes named "java"
pkill -f "app.jar"     # kill by matching the command line
```

### 6.3 SIGTERM vs SIGKILL (important)
- **SIGTERM (15):** asks the process to shut down **gracefully** — it can finish requests, flush data, close connections, then exit. **Always try this first.**
- **SIGKILL (9):** the OS **forcibly** terminates it immediately — no cleanup, possible data loss/corruption. **Last resort** for stuck processes.

> **Graceful shutdown matters:** your Spring app should catch SIGTERM to finish in-flight requests and close DB connections before exiting (Spring Boot does this). Docker/K8s send SIGTERM first, then SIGKILL after a grace period — handle SIGTERM properly! (Phase 10.)

---

## 7. Zombie & Orphan Processes

These two often confuse people — they're about the parent/child relationship.

### 7.1 Zombie processes (`Z`)
- A **zombie** is a process that has **finished**, but its parent hasn't **reaped** it (read its exit status via `wait()`).
- It occupies only a slot in the process table (an entry, not real resources).
- Normally harmless and brief; **many** zombies indicate a buggy parent not reaping children.
- You can't kill a zombie (it's already dead) — fix or restart the **parent**.

### 7.2 Orphan processes
- An **orphan** is a process whose **parent died first** while the child is still running.
- Orphans are **re-parented to PID 1 (`systemd`/init)**, which reaps them when they finish.
- Orphans are generally fine (init adopts them); zombies are the symptom of a bug.

| | Zombie | Orphan |
|---|--------|--------|
| State | Dead, awaiting reaping | Still running |
| Cause | Parent didn't `wait()` | Parent died first |
| Fixed by | Parent reaping (or restart) | Adopted by init (PID 1) |

> ⚠️ In Docker, your app often runs as **PID 1** — which normally doesn't reap orphans, leading to zombie buildup. Use a proper init (`--init`, `tini`) or handle signals. (Phase 10 "PID 1 problem".)

---

## 8. Resource & System Info Commands

| Command | Shows |
|---------|-------|
| `free -h` | Memory usage (total/used/free) |
| `uptime` | Load average + how long the system's been up |
| `nproc` | Number of CPU cores |
| `vmstat 1` | System stats incl. context switches (recall Phase 0.1) |
| `pidstat` | Per-process resource stats over time |
| `lsof -p <pid>` | Files/sockets a process has open |
| `lsof -i :8080` | What process is using port 8080 ★ |

```bash
lsof -i :8080            # "what's already using my port?" — super common
free -h                  # quick memory check
```

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Reaching for `kill -9` first | Try SIGTERM (15) first for graceful shutdown |
| Not handling SIGTERM in your app | Implement graceful shutdown (in-flight requests, connections) |
| Using `nohup &` for production services | Use systemd / containers instead |
| Trying to kill a zombie | It's already dead — fix the parent |
| "Port already in use" confusion | `lsof -i :PORT` to find the culprit |
| Ignoring load average | High sustained load = overloaded (scale or optimize) |
| App as PID 1 in Docker not reaping | Use `--init`/tini |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Graceful shutdown (Phase 10):** Spring Boot handles SIGTERM to drain requests; Docker/K8s rely on this for safe rolling deploys.
- **Debugging production (Phase 13):** `top`/`htop`, `ps`, `pidstat`, `free` to find CPU/memory hogs; `lsof` for port/file issues.
- **JVM diagnostics (Phase 1.11):** you find your app's PID (`jps`/`ps`) before running `jstack`/`jmap`.
- **Docker PID 1 problem (Phase 10):** zombie reaping and signal handling in containers.
- **Process model** ties back to Phase 0.1 (process vs thread, signals as IPC, context switching/load).

---

## 11. Quick Self-Check Questions

1. What are PID, PPID, and PID 1?
2. How do you find your running Java app's process?
3. What do the `STAT`, `RSS`, and `%CPU` columns in `ps aux` mean?
4. What is load average, and roughly when is the system "fully utilized"?
5. How do you run a process in the background and keep it alive after logout?
6. What's the difference between SIGTERM and SIGKILL? Which should you use first?
7. What is a zombie process vs an orphan process, and how is each resolved?
8. How do you find which process is using port 8080?
9. Why does graceful SIGTERM handling matter for a Spring app in Docker/K8s?

---

## 12. Key Terms Glossary

- **Process / PID / PPID:** running program / its ID / its parent's ID.
- **PID 1 (init/systemd):** ancestor of all processes; adopts orphans.
- **`ps` / `top` / `htop`:** process snapshot / live monitors.
- **Process state (R/S/D/T/Z):** running/sleeping/IO-wait/stopped/zombie.
- **Load average:** average processes wanting CPU (1/5/15 min).
- **Background job (`&`):** a process detached from the foreground.
- **`nohup`:** keep a process running after terminal close.
- **Signal:** async notification to a process.
- **SIGTERM (15) / SIGKILL (9):** graceful vs forced termination.
- **`kill` / `killall` / `pkill`:** send signals to processes.
- **Zombie:** finished process awaiting reaping by its parent.
- **Orphan:** running process whose parent died (re-parented to init).
- **`lsof`:** list open files/sockets (e.g., which process uses a port).

---

*Previous topic: **File Permissions & Ownership**.*
*Next topic: **Environment Variables, PATH & SSH/SCP**.*
