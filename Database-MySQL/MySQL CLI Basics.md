The **MySQL CLI** (Command-Line Interface), often just called the `mysql` client, is a text-based tool used to connect to, manage, and interact with a MySQL database server.
Instead of using a graphical user interface (GUI) with buttons and visual layouts like MySQL Workbench or phpMyAdmin, you type commands and SQL queries directly into your computer's terminal or command prompt.
### What it is used for?

- **Executing SQL Queries:** You can run `SELECT`, `INSERT`, `UPDATE`, and `DELETE` statements and see the results output as text-based tables directly in your console.

- **Database Structure Management:** Creating, modifying, or deleting databases and tables (`CREATE DATABASE`, `ALTER TABLE`, etc.).

- **User Administration:** Managing database users, passwords, and security privileges.

- **Running SQL Scripts:** You can feed entire `.sql` files into the CLI to execute hundreds of commands at once, which is the standard way to import databases or set up initial schemas.

### Why use the CLI instead of a GUI?

- **Server Environments:** When you connect to a remote web server (like a Linux VPS) via SSH, you usually don't have a graphical desktop. The CLI is the fastest and most reliable way to manage the database on that server.

- **Speed:** For experienced developers and database administrators, typing commands is often much faster than clicking through menus and waiting for heavy GUI applications to load.

- **Automation:** Because it runs from the command line, you can easily write scripts to automate tasks like daily database backups or data migrations.

### Installation Check
```
-- Check MySQL version
mysql --version
mysql -V
```

**Check if MySQL service is running:**
```
-- Linux/Mac:
sudo systemctl status mysql      -- Ubuntu/Debian
sudo systemctl status mysqld     -- CentOS/RHEL
brew services list               -- macOS with Homebrew

-- Windows:
sc query MySQL80                 -- Check service status
net start MySQL80                -- Start MySQL service
net stop  MySQL80                -- Stop  MySQL service
```

**Start / Stop MySQL Service:**
```
-- Linux
sudo systemctl start   mysql     -- Start
sudo systemctl stop    mysql     -- Stop
sudo systemctl restart mysql     -- Restart
sudo systemctl enable  mysql     -- Auto-start on boot

-- macOS (Homebrew)
brew services start  mysql
brew services stop   mysql
brew services restart mysql

-- Windows (Command Prompt as Admin)
net start MySQL80
net stop MySQL80
```

## Basic Connection

**Syntax:**
```
-- mysql [options] [database]
```

```
-- Connect with username + password prompt
mysql -u root -p
      ↑        ↑
--   username   prompt for password (secure)
```

```
-- Connect with password inline (NOT recommended - shows in history)
mysql -u root -pYourPassword
               ↑
         no space between -p and password
```

```
-- Connect to specific database directly
mysql -u root -p company_db
```

##### Connection Option
```
-- Local connection (most common)
mysql -u root -p
```

```
-- Connect directly to a database
mysql -u root -p company_db
```

```
-- Connect to specific host
mysql -u root -p -h localhost         -- explicit localhost
mysql -u root -p -h 127.0.0.1         -- using IP
```

```
-- Connect to remote server
mysql -u admin -p -h 192.168.1.100    -- remote IP
mysql -u admin -p -h db.mysite.com    -- remote hostname
```

```
-- Connect with specific port
mysql -u root -p -P 3306              -- default port
mysql -u root -p -P 3307              -- custom port
```

```
-- Full connection command (all options)
mysql -u username -p -h hostname -P port database_name
```

```
-- Example: remote connection
mysql -u admin -p -h 192.168.1.100 -P 3306 company_db
```

**Password options (security trade-off):**
```
mysql -u root -p                      -- SAFE   - prompts for password
mysql -u root -pYourPassword          -- UNSAFE - visible in terminal history
               ↑ no space!
```

**Connection Flags Reference:**
```
┌────────┬─────────┬──────────────────────────────────────┐
│ Short  │ Long    │ Description                          │
├────────┼─────────┼──────────────────────────────────────┤
│ -u     │ --user  │ MySQL username                       │
│ -p     │ --pass  │ Password (prompt if no value given)  │
│ -h     │ --host  │ Server hostname (default: localhost) │
│ -P     │ --port  │ Port number    (default: 3306)       │
│ -D     │ --db    │ Database to select on connect        │
│ -e     │ --exec  │ Execute SQL statement and exit       │
│ -V     │ --vers  │ Print version and exit               │
│ --help │ --help  │ Show help                            │
└────────┴─────────┴──────────────────────────────────────┘
```

##### **What the prompt tells you:**
```
mysql>          -- Ready for new command
    ->          -- Waiting (multi-line, no ; yet)
    '>          -- Inside a single quote  '
    ">          -- Inside a double quote  "
    `>          -- Inside a backtick      `
   /*>          -- Inside a /* comment */
```

**Stuck in wrong prompt? Use \c to cancel:**
```
mysql> SELECT * FROM               -- forgot to finish
    -> \c                          -- cancel it!
mysql>                             -- back to fresh prompt

mysql> SELECT 'hello               -- forgot closing quote
    '>  \c                         -- stuck in '> prompt
mysql>                             -- rescued!
```

## Navigation
Once you successfully log in, your prompt will change to `mysql>`.
**Crucial Rule:** Every SQL command inside the CLI **must end with a semicolon (`;`)** before you press Enter. If you forget it, the CLI will just move to a new line (`->`) waiting for you to finish.

##### Prompts For Database
```
SHOW DATABASES;                -- lists all database
```

```
-- Filter databases by name
SHOW DATABASES LIKE 'comp%';           -- Starts with 'comp'
SHOW DATABASES LIKE '%db%';           -- Contains 'db'
```

```
USE company_db;                -- uses database
```

```
SELECT DATABASE();             -- Check which database is currently selected
```

##### Prompts for Tables
```
SHOW TABLES;                   -- lists all tables
```

```
SHOW TABLES FROM company_db;   -- List tables from specific database(without USE)
SHOW TABLES IN   company_db;   -- Same thing
```

```
SHOW FULL TABLES;              -- Show tables with type (BASE TABLE or VIEW)
```

```
-- Filter tables by name pattern
SHOW TABLES LIKE 'emp%';               -- Tables starting with emp
SHOW TABLES LIKE '%order%';           -- Tables containing order
```

```
-- Show columns and types (multiple ways - all same result)
DESCRIBE  employees;
DESC      employees;                   -- Short form (most used)
EXPLAIN   employees;                   -- Same as DESCRIBE

SHOW COLUMNS FROM employees;
SHOW FULL COLUMNS FROM employees;      -- Extra detail
```

```
-- Show the exact CREATE TABLE statement
SHOW CREATE TABLE employees;
SHOW CREATE TABLE employees\G         -- Easier to read (vertical)
```

```
-- Show indexes on a table
SHOW INDEXES FROM employees;
SHOW KEYS   FROM employees;           -- Same thing
```

##### Status & Info Commands
```
-- Full status (most useful command!)
STATUS;
\s                                     -- Shortcut
```

##### BACKSLASH COMMANDS (no semicolon)
```
\h              -- Help
\s              -- Status
\u db_name      -- Switch database
\c              -- Cancel current input
\G              -- Execute vertically
\g              -- Execute (same as ;)
\W              -- Show warnings
\r              -- Reconnect
\q              -- Quit
\! cmd          -- Run shell command
source file.sql -- Run SQL file
```

