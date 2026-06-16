# Essential Commands (Navigation, Files & Text Processing)

> **Phase 0 — Computer Science Fundamentals → 0.3 Operating System Basics**
> Goal: Build fluency with the everyday Linux commands — navigation, file manipulation, and the text-processing tools (`grep`, `sed`, `awk`, `cut`) — plus pipes and redirection.

---

## 0. The Big Picture

The Linux command line is a toolbox of small, focused programs that you **combine** to do powerful things. The Unix philosophy: *"do one thing well, and connect tools with pipes."* This note covers the commands you'll type dozens of times a day.

---

## 1. Navigation

| Command | Purpose | Example |
|---------|---------|---------|
| `pwd` | Print working directory (where am I?) | `pwd` → `/home/alice` |
| `ls` | List directory contents | `ls -la` |
| `cd` | Change directory | `cd /var/log`, `cd ..`, `cd ~`, `cd -` |
| `find` | Search for files (live, powerful) | `find . -name "*.log"` |
| `locate` | Search files via a prebuilt index (fast) | `locate nginx.conf` |
| `tree` | Show directory tree (if installed) | `tree -L 2` |

### 1.1 `ls` options you'll use constantly
```bash
ls -l      # long format (permissions, owner, size, date)
ls -a      # include hidden (dot) files
ls -la     # both
ls -lh     # human-readable sizes (KB/MB)
ls -lt     # sort by modification time (newest first)
ls -lS     # sort by size
```

### 1.2 `find` — the workhorse search
```bash
find /var/log -name "*.log"            # by name
find . -type f -size +100M             # files larger than 100MB
find . -type d -name "target"          # directories named target
find . -name "*.tmp" -mtime +7         # modified more than 7 days ago
find . -name "*.log" -delete           # find and delete (careful!)
find . -name "*.java" -exec grep -l "TODO" {} \;   # run a command per result
```
> `find` searches **live** (slower, always current). `locate` uses a **prebuilt database** (instant, but may be stale — update with `updatedb`).

---

## 2. Viewing File Contents

| Command | Purpose |
|---------|---------|
| `cat` | Print whole file (also concatenate) |
| `less` | Page through a file (scroll, search with `/`) |
| `head` | First N lines (`head -n 20 file`) |
| `tail` | Last N lines (`tail -n 50 file`) |
| `tail -f` | **Follow** a file live (essential for logs!) |
| `wc` | Count lines/words/bytes (`wc -l`) |

```bash
cat config.yml
less /var/log/syslog          # q to quit, /searchterm to search
head -n 20 data.csv           # first 20 lines
tail -n 100 app.log           # last 100 lines
tail -f /var/log/app.log      # watch logs in real time  ★ (most-used for debugging)
tail -f app.log | grep ERROR  # live-filter for errors
wc -l data.csv                # how many lines?
```
> `tail -f` (follow) is *the* command for watching application logs as requests come in.

---

## 3. File & Directory Manipulation

| Command | Purpose | Example |
|---------|---------|---------|
| `touch` | Create empty file / update timestamp | `touch new.txt` |
| `mkdir` | Make directory (`-p` for nested) | `mkdir -p a/b/c` |
| `cp` | Copy (`-r` for directories) | `cp -r src/ backup/` |
| `mv` | Move or rename | `mv old.txt new.txt` |
| `rm` | Remove (`-r` recursive, `-f` force) | `rm -rf temp/` ⚠️ |
| `rmdir` | Remove empty directory | `rmdir emptydir` |
| `ln -s` | Create a symbolic link | `ln -s /path target` |

```bash
mkdir -p project/src/main      # create nested dirs in one go
cp -r config/ config-backup/   # copy a directory recursively
mv report.txt reports/2026/    # move into a folder
rm file.txt                    # delete a file
rm -rf old-project/            # delete a directory tree (DANGEROUS)
```
> ⚠️ **`rm -rf` is irreversible and has no recycle bin.** Double-check the path. Never run `rm -rf /` or `rm -rf $VAR/` where `$VAR` might be empty.

---

## 4. Pipes & Redirection (the glue)

This is what makes the CLI powerful: connect commands and redirect their input/output.

### 4.1 Pipes (`|`) — chain commands
The output of one command becomes the input of the next:
```bash
cat access.log | grep "404" | wc -l        # count 404s in a log
ps aux | grep java                          # find java processes
ls -l | sort -k5 -n                         # list sorted by size
history | grep git                          # past git commands
```

### 4.2 Redirection — send output to/from files
| Operator | Effect |
|----------|--------|
| `>` | Redirect stdout to a file (**overwrite**) |
| `>>` | Redirect stdout to a file (**append**) |
| `<` | Read stdin from a file |
| `2>` | Redirect stderr |
| `2>&1` | Redirect stderr to wherever stdout goes |
| `&>` | Redirect both stdout and stderr |

```bash
echo "hello" > file.txt          # write (overwrites)
echo "more" >> file.txt          # append
java -jar app.jar > out.log 2>&1 # capture both output and errors to a log
command 2> errors.txt            # only errors to a file
sort < unsorted.txt              # feed a file as input
command > /dev/null 2>&1         # discard all output
```

### 4.3 The three standard streams (recap from Phase 0.1 I/O)
| Stream | Number | Purpose |
|--------|--------|---------|
| **stdin** | 0 | Standard input |
| **stdout** | 1 | Standard output (normal results) |
| **stderr** | 2 | Standard error (error messages) |

---

## 5. Text Processing — grep, sed, awk, cut

These four are the heart of log analysis and data wrangling on the command line.

### 5.1 `grep` — search text by pattern (regex!)
```bash
grep "ERROR" app.log              # lines containing ERROR
grep -i "error" app.log           # case-insensitive
grep -r "TODO" src/               # recursive search in a directory
grep -n "main" Main.java          # show line numbers
grep -v "DEBUG" app.log           # INVERT: lines NOT containing DEBUG
grep -c "404" access.log          # count matching lines
grep -E "error|warn" app.log      # extended regex (OR)
grep -A 3 -B 2 "Exception" app.log # 3 lines After, 2 Before each match
```
> `grep` uses **regular expressions** (recall Phase 1.4 regex). It's your #1 tool for finding things in logs and code.

### 5.2 `cut` — extract columns/fields
```bash
cut -d',' -f1,3 data.csv          # fields 1 and 3, comma-delimited (CSV)
cut -d':' -f1 /etc/passwd         # usernames (first colon-field)
cut -c1-10 file.txt               # characters 1-10 of each line
```
- `-d` = delimiter, `-f` = field number(s), `-c` = character range.

### 5.3 `sed` — stream editor (find/replace, transform)
```bash
sed 's/foo/bar/' file.txt         # replace first 'foo' per line with 'bar'
sed 's/foo/bar/g' file.txt        # replace ALL (global)
sed 's/foo/bar/gi' file.txt       # global + case-insensitive
sed -i 's/old/new/g' file.txt     # edit the file IN PLACE (-i)
sed -n '10,20p' file.txt          # print only lines 10-20
sed '/^#/d' config.txt            # delete comment lines (starting with #)
```
> `sed` is a non-interactive editor — great for scripted find/replace across files.

### 5.4 `awk` — column-aware processing & reports
`awk` treats each line as fields (`$1`, `$2`, ...; `$0` = whole line):
```bash
awk '{print $1}' access.log              # first whitespace field of each line
awk -F',' '{print $2}' data.csv          # 2nd field, comma-delimited
awk '{sum += $3} END {print sum}' data   # sum the 3rd column
awk '$3 > 100 {print $1}' data           # print col1 where col3 > 100
awk -F':' '{print $1, $3}' /etc/passwd   # multiple fields
ps aux | awk '{print $2, $11}'           # PID and command from ps
```
> `awk` shines for **column-based** data and quick aggregations (sums, averages, filtering) — like a mini spreadsheet/SQL on text.

### 5.5 Choosing the right text tool
| Tool | Best for |
|------|----------|
| `grep` | Finding lines matching a pattern |
| `cut` | Extracting fixed columns/fields |
| `sed` | Find/replace and line edits |
| `awk` | Field-aware processing, math, reports |
| `sort` / `uniq` | Ordering and deduplicating |

### 5.6 A real-world combo
```bash
# Top 10 IPs hitting your server, from an access log:
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
#   extract IP    | sort | count dups | sort by count desc | top 10
```

---

## 6. Other Handy Commands

| Command | Purpose |
|---------|---------|
| `sort` | Sort lines (`-n` numeric, `-r` reverse, `-k` by column) |
| `uniq` | Remove/count adjacent duplicates (`-c` count) |
| `tr` | Translate/delete characters (`tr 'a-z' 'A-Z'`) |
| `tee` | Write to a file AND pass through (`... | tee out.log`) |
| `xargs` | Build command args from input (`... | xargs rm`) |
| `diff` | Compare two files |
| `echo` / `printf` | Print text |
| `clear` | Clear the screen |

---

## 7. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `>` overwriting a file you meant to append | Use `>>` to append |
| `rm -rf` on the wrong path | Double-check; no undo exists |
| Forgetting errors go to stderr | Use `2>&1` to capture them too |
| `grep` pattern with special chars | Quote it; remember it's regex |
| `sed -i` without a backup | Use `sed -i.bak` to keep a backup |
| `locate` results stale | Run `updatedb`, or use `find` |
| Unquoted variables/filenames with spaces | Always quote `"$var"` |

---

## 8. Connection to Backend / Spring (Why This Matters Later)

- **Log analysis (Phase 9):** `tail -f`, `grep`, `awk` are how you debug production logs in real time.
- **Data processing (Project 2):** `cut`/`awk`/`sort` mirror what you'll do in code for CSVs.
- **Docker/CI scripts (Phase 10):** pipes and redirection are everywhere in Dockerfiles and CI pipelines.
- **Debugging deployments:** `find`, `cat`, `grep` to inspect configs and logs inside containers (`docker exec`).
- **Shell scripting (later note)** composes these commands into automation.

---

## 9. Quick Self-Check Questions

1. What's the difference between `find` and `locate`?
2. How do you watch a log file update in real time?
3. What do `>`, `>>`, `2>`, and `2>&1` do?
4. What are stdin, stdout, and stderr (and their numbers)?
5. How do you count the number of "ERROR" lines in a log?
6. What does `grep -v` do? `grep -r`?
7. When would you use `cut` vs `awk`?
8. Write a pipeline to find the top 10 most frequent values in column 1 of a file.

---

## 10. Key Terms Glossary

- **Pipe (`|`):** send one command's output to another's input.
- **Redirection (`>`/`>>`/`<`):** route output to/from files.
- **stdin/stdout/stderr (0/1/2):** standard input/output/error streams.
- **`grep`:** pattern (regex) line search.
- **`sed`:** stream editor for find/replace and line edits.
- **`awk`:** field-aware text processing and reporting.
- **`cut`:** extract columns/fields/characters.
- **`sort`/`uniq`:** order / dedupe lines.
- **`find`/`locate`:** live search / indexed search.
- **`tail -f`:** follow a file as it grows.
- **`xargs`:** turn input into command arguments.

---

*Previous topic: **Linux Fundamentals & Filesystem**.*
*Next topic: **File Permissions & Ownership**.*
