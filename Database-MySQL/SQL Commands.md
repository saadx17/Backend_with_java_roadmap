SQL commands are *instructions used to interact* with and manage data in relational databases.

![[SQL_Commands.png]]

They are generally categorized into five types based on their specific purpose:

1. [[DDL - Data Definition Language]]
2. [[DML - Data Manipulation Language]]
3. [[DQL - Data Query Language]]
4. [[DCL -Data Control Language]]
5. [[TCL - Transaction Control Language]]

```
SQL COMMANDS
│
├── DDL (Auto-committed, affects structure)
│   ├── CREATE   → Make new DB/Table/View/Index
│   ├── ALTER    → Modify existing structure
│   ├── DROP     → Delete permanently
│   ├── TRUNCATE → Remove all rows (keep structure)
│   └── RENAME   → Rename table/database
│
├── DML (Can be rolled back, affects data)
│   ├── INSERT → Add new rows
│   ├── UPDATE → Modify existing rows
│   └── DELETE → Remove specific rows
│
├── DQL (Read only)
│   └── SELECT → Retrieve data
│       ├── WHERE      → Filter rows
│       ├── GROUP BY   → Group rows
│       ├── HAVING     → Filter groups
│       ├── ORDER BY   → Sort results
│       └── LIMIT      → Restrict count
│
├── DCL (Permissions)
│   ├── GRANT  → Give permissions
│   └── REVOKE → Remove permissions
│
└── TCL (Transaction management)
    ├── COMMIT           → Save permanently
    ├── ROLLBACK         → Undo changes
    ├── SAVEPOINT        → Create checkpoint
    └── SET TRANSACTION  → Configure transaction
```

**SQL Basics:**
```
CREATE DATABASE -> creates a new database

CREATE TABLE -> creates a new table

ALTER DATABASE -> modifies a database

ALTER TABLE -> modifies a table

DROP DATABASE -> deletes a database

DROP TABLE -> deletes a table

INSERT INTO -> inserts new data into a database

UPDATE -> updates data in a database

SELECT -> extracts data from a database (Query)
```
