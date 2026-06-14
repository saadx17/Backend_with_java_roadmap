# File Permissions & Ownership

> **Phase 0 — Computer Science Fundamentals → 0.3 Operating System Basics**
> Goal: Master Linux file permissions — read/write/execute, the user/group/other model, octal notation, and the commands `chmod`, `chown`, and `umask`.

---

## 0. The Big Picture

Linux is a **multi-user** OS. Every file and directory has **permissions** controlling **who** can do **what**. This is fundamental to security: it's how the OS prevents one user (or compromised process) from reading/modifying another's files.

```
-rwxr-xr--  1  alice  devs  4096  Jun 12 10:00  script.sh
^                ^      ^
permissions     owner  group
```

---

## 1. The Permission Model: Who × What

### 1.1 Three permission types (What)
| Permission | Symbol | On a **file** | On a **directory** |
|------------|--------|---------------|---------------------|
| **Read** | `r` | View contents | List its entries (`ls`) |
| **Write** | `w` | Modify contents | Create/delete/rename entries inside |
| **Execute** | `x` | Run it as a program | Enter/traverse it (`cd` into) |

> ⚠️ Note the **directory** meanings differ: `x` on a directory means you can *enter* it; `r` means you can *list* it. You often need `x` to access files inside even if you have `r`.

### 1.2 Three permission classes (Who)
| Class | Symbol | Means |
|-------|--------|-------|
| **User (owner)** | `u` | The file's owner |
| **Group** | `g` | Members of the file's group |
| **Other** | `o` | Everyone else |
| (All) | `a` | u + g + o together |

---

## 2. Reading `ls -l` Output

```
-rwxr-xr--
│└┬┘└┬┘└┬┘
│ │  │  └── other:  r-- (read only)
│ │  └───── group:  r-x (read + execute)
│ └──────── user:   rwx (read + write + execute)
└────────── file type
```

### 2.1 The first character: file type
| Char | Type |
|------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` / `b` | Character / block device |
| `p` / `s` | Named pipe / socket |

### 2.2 The next 9 characters: 3 groups of rwx
`rwx r-x r--` → user=`rwx`, group=`r-x`, other=`r--`. A `-` means that permission is absent.

Full example:
```
-rw-r--r--   a regular file: owner can read/write, group & others read-only (common for data files)
drwxr-xr-x   a directory: owner full, group & others can enter & list (common for dirs)
-rwxr-xr-x   an executable script everyone can run
```

---

## 3. Octal (Numeric) Notation

Permissions are often written as **3 octal digits** — a compact, ubiquitous form. Each permission has a value:
```
r = 4    w = 2    x = 1    (- = 0)
```
Add them per class:
| Octal | Binary | Permissions | Meaning |
|-------|--------|-------------|---------|
| 7 | 111 | `rwx` | read + write + execute |
| 6 | 110 | `rw-` | read + write |
| 5 | 101 | `r-x` | read + execute |
| 4 | 100 | `r--` | read only |
| 0 | 000 | `---` | none |

Three digits = user, group, other:
```
chmod 755 file    ->  rwx r-x r-x   (owner full; others read+execute)  -- common for scripts/dirs
chmod 644 file    ->  rw- r-- r--   (owner read/write; others read)    -- common for data files
chmod 600 file    ->  rw- --- ---   (owner only)                       -- secrets/private keys
chmod 777 file    ->  rwx rwx rwx   (everyone everything — AVOID!)     -- security risk
```
> **Memorize the common ones:** `755` (executables/dirs), `644` (regular files), `600` (private/secret files). Recall Phase 0: octal = base-8 (groups of 3 bits).

---

## 4. `chmod` — Change Permissions

### 4.1 Octal mode (absolute)
```bash
chmod 755 script.sh        # set exactly rwxr-xr-x
chmod 644 data.txt
chmod 600 ~/.ssh/id_rsa    # private key — owner only
```

### 4.2 Symbolic mode (relative changes)
```bash
chmod +x script.sh         # add execute for all
chmod u+x script.sh        # add execute for the owner only
chmod g-w file             # remove write from group
chmod o-rwx secret         # remove all perms from others
chmod u=rw,go=r file       # set owner=rw, group&other=r (== 644)
chmod -R 755 mydir/        # recursive (apply to dir and contents)
```
| Symbol | Meaning |
|--------|---------|
| `+` / `-` / `=` | add / remove / set exactly |
| `u` `g` `o` `a` | user / group / other / all |

> Making a script runnable is the most common use: `chmod +x deploy.sh`.

---

## 5. `chown` — Change Ownership

Changes the **owner** and/or **group** of a file (usually needs `sudo`):
```bash
chown alice file.txt            # change owner to alice
chown alice:devs file.txt       # change owner to alice AND group to devs
chown :devs file.txt            # change only the group
chown -R alice:devs /app        # recursive (whole tree)
chgrp devs file.txt             # change only group (alternative)
```
> Common in deployment: ensuring the app's user owns its files/directories (e.g., `chown -R appuser:appuser /opt/app`).

---

## 6. `umask` — Default Permission Mask

When a new file/directory is created, its permissions are determined by a base value **minus** the `umask`. The `umask` *removes* permissions from the default.

```
Default base:   files = 666 (rw-rw-rw-),  directories = 777 (rwxrwxrwx)
umask:          typically 022
Result:         files = 666 - 022 = 644,  dirs = 777 - 022 = 755
```
```bash
umask              # show current mask (e.g., 0022)
umask 077          # new files: 600, new dirs: 700 (private by default)
```
- `umask 022` (common default) → new files `644`, new dirs `755`.
- `umask 077` → owner-only (more secure for sensitive environments).
- New files **never get execute by default** (base is 666, not 777) — you must add `x` explicitly.

> Think of `umask` as "which permission bits to *turn off*" for newly created files.

---

## 7. The Root User & `sudo`

- **root** (UID 0) is the **superuser** — bypasses all permission checks. Can do anything.
- **`sudo`** runs a single command as root (or another user): `sudo apt update`.
- Best practice: **don't log in as root**; use a normal user and `sudo` only when needed (principle of least privilege — recall Phase 1.3 access modifiers; same idea at the OS level).

```bash
sudo systemctl restart nginx     # run one command as root
sudo -i                          # start a root shell (use sparingly)
```

---

## 8. Special Permission Bits (awareness)

Beyond rwx, three special bits exist (a 4th leading octal digit):
| Bit | Octal | Effect |
|-----|-------|--------|
| **setuid** | 4000 | Run a file as its **owner** (e.g., `passwd`) |
| **setgid** | 2000 | Run as the **group** / new files inherit the dir's group |
| **sticky bit** | 1000 | In a shared dir (e.g., `/tmp`), only the owner can delete their files |

```bash
ls -l /tmp     # drwxrwxrwt  <- the 't' is the sticky bit
```
> You'll rarely set these, but recognizing `s`/`t` in `ls -l` output and knowing the sticky bit on `/tmp` is useful.

---

## 9. Common Pitfalls / Misconceptions

| Pitfall | Fix |
|---------|-----|
| `chmod 777` to "fix" a permission problem | Huge security risk — grant the *minimum* needed |
| Forgetting directories need `x` to enter | Add execute on directories (`755`) |
| Private keys not `600` | SSH refuses keys with loose perms — `chmod 600` |
| Editing files owned by root without `sudo` | Use `sudo` or fix ownership with `chown` |
| Expecting new files to be executable | Default removes `x`; add it explicitly |
| Recursive `chmod` on the wrong tree | Double-check the path with `-R` |
| Running everything as root | Use a normal user + `sudo` (least privilege) |

---

## 10. Connection to Backend / Spring (Why This Matters Later)

- **Docker security (Phase 10):** run containers as a **non-root user**; set correct file ownership/permissions — a key best practice and security requirement.
- **Secrets (Phase 15):** key files, `.env`, and config with credentials should be `600` (owner-only).
- **Deployment scripts:** `chmod +x` on scripts, `chown` so the app user owns its directories.
- **Volume mounts (Phase 10):** permission mismatches between host and container are a common, confusing bug.
- **Least privilege** at the OS level mirrors least privilege in code (Phase 1.3) and auth (Phase 15) — the same security principle throughout.

---

## 11. Quick Self-Check Questions

1. What do read, write, and execute mean for a **file** vs a **directory**?
2. What are the three permission classes (who)?
3. Decode `-rwxr-xr--` into its classes and permissions.
4. What are the octal values of r, w, x? What does `755` mean? `644`? `600`?
5. How do you make a script executable with `chmod` (two ways)?
6. What does `chown alice:devs file` do?
7. What does `umask 022` produce for new files and directories, and why no execute?
8. Why should you avoid `chmod 777` and running as root?

---

## 12. Key Terms Glossary

- **Permissions (rwx):** read / write / execute rights.
- **User / group / other:** the owner / the file's group / everyone else.
- **Octal notation:** numeric permissions (r=4, w=2, x=1).
- **`chmod`:** change permissions (octal or symbolic).
- **`chown` / `chgrp`:** change owner / group.
- **`umask`:** mask removing bits from default permissions on creation.
- **root / superuser:** UID 0; bypasses permission checks.
- **`sudo`:** run a command with elevated privileges.
- **setuid / setgid / sticky bit:** special permission bits.
- **Least privilege:** grant only the minimum access needed.

---

*Previous topic: **Essential Commands**.*
*Next topic: **Process Management & Signals**.*
