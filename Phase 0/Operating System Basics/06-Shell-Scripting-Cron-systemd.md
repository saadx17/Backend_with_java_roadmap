# Shell Scripting, Cron Jobs & systemd

> **Phase 0 — Computer Science Fundamentals → 0.3 Operating System Basics**
> Goal: Learn the basics of bash scripting (automation), scheduling tasks with cron, and managing long-running services with systemd.

---

## 0. The Big Picture

The final OS skill set is **automation**: turning sequences of commands into reusable **scripts**, **scheduling** them with cron, and running your application as a managed **service** with systemd. These tie together everything from sections 0.3.

```
Scripts = automate tasks   |   Cron = run on a schedule   |   systemd = keep services running
```

---

## 1. Shell Scripting Basics

A **shell script** is a text file of commands the shell executes top to bottom — automating repetitive tasks (deployments, backups, log cleanup).

### 1.1 Anatomy of a script
```bash
#!/bin/bash               # 1. SHEBANG — which interpreter runs this file
# This is a comment       # 2. comments start with #

echo "Starting backup..." # 3. commands, one per line

DIR="/var/backups"        # 4. variable (no spaces around =)
mkdir -p "$DIR"           #    use "$DIR" (quoted) to reference it
echo "Done."
```
Run it:
```bash
chmod +x backup.sh        # make it executable (recall permissions note)
./backup.sh               # run it
bash backup.sh            # or run via bash explicitly
```
> The **shebang** (`#!/bin/bash`) tells the OS which interpreter to use (recall Phase 0.1: this is read by the loader). Without it, run with `bash script.sh`.

### 1.2 Variables & command substitution
```bash
NAME="Sam"                       # assign (NO spaces around =)
echo "Hello, $NAME"              # use with $
echo "Today is $(date)"          # $(...) runs a command, inserts its output
FILES=$(ls | wc -l)              # capture command output into a variable
```

### 1.3 Arguments & special variables
```bash
./deploy.sh prod v2.1
# Inside the script:
$0    # script name (deploy.sh)
$1    # first argument (prod)
$2    # second argument (v2.1)
$#    # number of arguments (2)
$@    # all arguments
$?    # exit code of the last command (0 = success)
$$    # PID of the script
```

### 1.4 Conditionals
```bash
if [ "$1" == "prod" ]; then
    echo "Deploying to production"
elif [ "$1" == "staging" ]; then
    echo "Deploying to staging"
else
    echo "Unknown environment"
    exit 1                        # exit with an error code
fi

# File/number tests:
if [ -f "config.yml" ]; then echo "file exists"; fi      # -f file exists
if [ -d "/var/log" ]; then echo "dir exists"; fi         # -d dir exists
if [ -z "$VAR" ]; then echo "VAR is empty"; fi           # -z empty string
if [ "$count" -gt 10 ]; then echo "big"; fi              # -gt -lt -eq -ge -le
```
| Test | Meaning |
|------|---------|
| `-f` / `-d` / `-e` | file / directory / exists |
| `-z` / `-n` | string empty / non-empty |
| `-eq -ne -gt -lt -ge -le` | numeric compare |
| `==` / `!=` | string compare |

### 1.5 Loops
```bash
for file in *.log; do            # iterate files
    echo "Processing $file"
done

for i in {1..5}; do echo "$i"; done    # range

while read line; do              # read a file line by line
    echo "$line"
done < input.txt
```

### 1.6 Exit codes & error handling
- Every command returns an **exit code**: `0` = success, non-zero = failure (check with `$?`).
- Robust scripts start with safety flags:
```bash
#!/bin/bash
set -e            # exit immediately if any command fails
set -u            # error on undefined variables
set -o pipefail   # fail if any command in a pipe fails
# common combined form:  set -euo pipefail
```
> `set -euo pipefail` is a best-practice header that makes scripts fail fast instead of silently continuing after errors (recall the fail-fast principle from Phase 1.5).

### 1.7 Functions
```bash
log() {
    echo "[$(date)] $1"          # $1 is the function's first argument
}
log "Backup started"
```

---

## 2. Cron Jobs (Scheduled Tasks)

**Cron** runs commands automatically **on a schedule** (backups, cleanups, reports). The **crontab** is the schedule table.

### 2.1 Managing the crontab
```bash
crontab -e      # edit your cron jobs
crontab -l      # list your cron jobs
crontab -r      # remove all (careful!)
```

### 2.2 Cron syntax (5 time fields + command)
```
* * * * * command-to-run
│ │ │ │ │
│ │ │ │ └── day of week (0-7, 0 & 7 = Sunday)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```

### 2.3 Examples
```bash
0 2 * * *      /opt/scripts/backup.sh         # every day at 2:00 AM
*/15 * * * *   /opt/scripts/health-check.sh   # every 15 minutes
0 0 * * 0      /opt/scripts/weekly-report.sh  # Sundays at midnight
0 9 1 * *      /opt/scripts/monthly.sh        # 9 AM on the 1st of each month
30 8 * * 1-5   /opt/scripts/workday.sh        # 8:30 AM Mon-Fri
```
| Field pattern | Meaning |
|---------------|---------|
| `*` | Every value |
| `*/n` | Every n units (e.g., `*/15` minutes) |
| `1-5` | Range |
| `1,15` | List |

### 2.4 Cron gotchas
- Cron runs with a **minimal environment** (limited `PATH`, no `~/.bashrc`) — use **absolute paths** and set needed env vars in the script.
- **Redirect output** or you won't see errors: `... >> /var/log/myjob.log 2>&1`.
- Test the script manually first; cron failures are silent unless logged.
> Tools like `crontab.guru` help you write/verify schedules.

---

## 3. systemd & Services

**systemd** is the modern Linux **init system** (PID 1 — recall the process note) and **service manager**. It starts, stops, monitors, and auto-restarts long-running programs (**services/daemons**) — the proper way to run your app on a server (vs `nohup &`).

### 3.1 Managing services with `systemctl`
```bash
sudo systemctl start nginx        # start a service
sudo systemctl stop nginx         # stop
sudo systemctl restart nginx      # restart
sudo systemctl reload nginx       # reload config without full restart
sudo systemctl status nginx       # check status (running? recent logs?) ★
sudo systemctl enable nginx       # start automatically on boot
sudo systemctl disable nginx      # don't start on boot
sudo systemctl is-active nginx    # quick active check
```
> `enable` = start on boot; `start` = start now. You usually want **both** for a server service.

### 3.2 A service unit file
Services are defined by **unit files** (in `/etc/systemd/system/`). Example for a Spring Boot app:
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Spring Boot App
After=network.target                  # start after networking is up

[Service]
User=appuser                          # run as a non-root user (security!)
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/java -jar /opt/myapp/app.jar
Environment=SPRING_PROFILES_ACTIVE=prod   # env vars (recall env note)
Restart=always                        # auto-restart if it crashes
RestartSec=10

[Install]
WantedBy=multi-user.target            # enable at normal multi-user boot
```
After creating/editing a unit:
```bash
sudo systemctl daemon-reload          # reload unit definitions
sudo systemctl enable --now myapp     # enable + start in one step
```

### 3.3 Why systemd over `nohup &`
| Feature | `nohup &` | systemd |
|---------|-----------|---------|
| Survives logout | Yes | Yes |
| **Auto-restart on crash** | No | **Yes** (`Restart=always`) |
| **Start on boot** | No | **Yes** (`enable`) |
| Centralized logs | Manual | **`journalctl`** |
| Resource limits | No | Yes |
| Proper signal handling | No | Yes (graceful SIGTERM — recall process note) |

> For anything that must "stay running," use **systemd** (or a container orchestrator like K8s — Phase 10), not `nohup`.

### 3.4 Viewing service logs — `journalctl`
systemd captures service output into the **journal**:
```bash
journalctl -u myapp               # logs for the myapp service
journalctl -u myapp -f            # follow live (like tail -f)
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp -n 100        # last 100 lines
```

---

## 4. Putting It Together (a typical server setup)
```
1. Write a script to run/deploy your app        (shell scripting)
2. Run the app as a managed service              (systemd unit, Restart=always, enable)
3. Schedule a nightly backup / log cleanup       (cron)
4. Monitor with status/logs                       (systemctl status, journalctl -f)
```

---

## 5. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| Spaces around `=` in bash assignments | `NAME="x"` (no spaces) |
| Unquoted variables (break on spaces) | Always quote `"$var"` |
| Scripts continuing after errors | Use `set -euo pipefail` |
| Cron using relative paths/env | Use absolute paths; set env in the script |
| Cron job "not running" with no clue | Redirect output to a log (`>> log 2>&1`) |
| Running services with `nohup &` | Use systemd (auto-restart, boot, logs) |
| `start` without `enable` (dies on reboot) | `enable` to start on boot |
| Running the service as root | Use a dedicated non-root `User=` |
| Forgetting `daemon-reload` after editing a unit | Run `systemctl daemon-reload` |

---

## 6. Connection to Backend / Spring (Why This Matters Later)

- **Running your app (Phase 10):** on a VM you'd run it via a **systemd unit** (`Restart=always`, non-root `User`, env vars, `journalctl`) — or, more commonly today, as a **container** (which is its own process supervisor).
- **Env vars + profiles:** systemd `Environment=` sets `SPRING_PROFILES_ACTIVE` and config (Phase 5.2).
- **Graceful restart/auto-restart** matters for uptime and zero-downtime deploys (Phase 10).
- **Cron / scheduled tasks:** mirrors Spring's `@Scheduled` and ShedLock (Phase 5.8) — and CI cron triggers (Phase 10.3); cleanup jobs (Project 4).
- **Scripting** powers CI/CD pipelines, Dockerfile `RUN` steps, and deployment automation (Phase 10).
- **`journalctl`/logs** are core to production debugging (Phase 9, 13).

---

## 7. Quick Self-Check Questions

1. What is a shebang, and why does a script need it?
2. Why must there be no spaces around `=` in a bash assignment, and why quote variables?
3. What do `$0`, `$1`, `$#`, `$?`, and `$@` mean?
4. What does `set -euo pipefail` do and why use it?
5. Write a cron entry that runs a script every day at 2 AM.
6. Why do cron jobs often fail due to environment/PATH, and how do you debug them?
7. What does systemd do, and why prefer it over `nohup &`?
8. What's the difference between `systemctl start` and `systemctl enable`?
9. How do you view and follow a service's logs?

---

## 8. Key Terms Glossary

- **Shell script:** a file of shell commands run sequentially.
- **Shebang (`#!`):** line specifying the interpreter.
- **Command substitution (`$(...)`):** insert a command's output.
- **Exit code (`$?`):** 0 = success, non-zero = failure.
- **`set -euo pipefail`:** safety flags for fail-fast scripts.
- **Cron / crontab:** scheduler / its schedule table.
- **Cron fields:** minute, hour, day-of-month, month, day-of-week.
- **systemd:** modern init system & service manager (PID 1).
- **`systemctl`:** control services (start/stop/enable/status).
- **Unit file:** a `.service` definition for a managed program.
- **`Restart=always` / `enable`:** auto-restart on crash / start on boot.
- **Daemon / service:** a long-running background program.
- **`journalctl`:** view systemd service logs.

---

*Previous topic: **Environment Variables, Networking Tools, Package Management & SSH**.*
*This completes **Section 0.3 — Operating System Basics**.*
*Next section in roadmap: **0.4 Version Control — Git (Mastery)**.*
