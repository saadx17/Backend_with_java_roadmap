# What is a Database Engine?
A **database engine** (also called a **storage engine**) is the underlying software component that a DBMS uses to perform **CRUD operations** (Create, Read, Update, Delete) on data.

**It Handles:**
```
┌─────────────────────────────────────────┐
│          DATABASE ENGINE                │
│                                         │
│  ✦ How data is STORED on disk           │
│  ✦ How data is READ from disk           │
│  ✦ How INDEXES work                     │
│  ✦ How TRANSACTIONS work                │
│  ✦ How LOCKING is managed               │
│  ✦ How CONCURRENCY is handled           │
│  ✦ How CRASH RECOVERY works             │
│  ✦ How data is COMPRESSED               │
│  ✦ How FOREIGN KEYS are enforced        │
│  ✦ Data FILE FORMAT on disk             │
└─────────────────────────────────────────┘
```

# What is CRUD Operations?
[CRUD operations](https://www.geeksforgeeks.org/sql/crud-operations-in-mysql/) (`Create, Read, Update, Delete`) are *the four fundamental actions used to manage persistent data in database-driven applications*. They enable users to add new records (Create), view stored information (Read), modify existing data (Update), and remove records (Delete). These operations form the backbone of most web apps, RESTful APIs, and database interactions.

#### CRUD Breakdown
- **==Create== (POST/INSERT):** Adds new data entries, such as registering a new user account.

- **==Read== (GET/SELECT):** Retrieves or fetches existing data to display, such as viewing a user profile.

- **==Update== (PUT/PATCH/UPDATE):** Modifies, edits, or updates existing records, like changing a username.

- **==Delete== (DELETE):** Removes existing data from the system, such as deleting a post.

#### CRUD in Technical Contexts
- **SQL Databases:** Mapped to INSERT, SELECT, UPDATE, and DELETE statements.

- **RESTful APIs:** Mapped to HTTP methods POST (Create), GET (Read), PUT/PATCH (Update), and DELETE (Delete).

- **NoSQL (e.g., MongoDB):** Uses methods like `insertOne`, `find`, `updateMany`, and `deleteOne`.

# Engine Architecture in MySQL
```
         ┌────────────────────────┐
         │      Client Layer      │
         │   (Connections, Auth)  │
         └──────────┬─────────────┘
                    │
         ┌──────────▼─────────────┐
         │      Server Layer      │
         │  (Parser, Optimizer,   │
         │   Cache, Query Exec)   │
         └──────────┬─────────────┘
                    │
    ┌───────────────▼───────────────────┐
    │        STORAGE ENGINE LAYER       │
    │  (Pluggable Architecture)         │
    ├──────┬──────┬──────┬──────┬───────┤
    │InnoDB│MyISAM│MEMORY│ CSV  │ etc.  │
    └──────┴──────┴──────┴──────┴───────┘
                    │
         ┌──────────▼─────────────┐
         │    File System / Disk  │
         └────────────────────────┘
```
**Key Concept:** MySQL has a **pluggable storage engine architecture**, different tables in the **same database** can use **different engines**.

# MySQL Storage Engines

**View All Available Engines:**
```
SHOW ENGINES;
```

**All Engines:**
1. [InnoDB](https://en.wikipedia.org/wiki/InnoDB)
2. [MyISAM](https://en.wikipedia.org/wiki/MyISAM)
3. [MEMORY](https://dev.mysql.com/doc/refman/9.7/en/memory-storage-engine.html)
4. [CSV](https://dev.mysql.com/doc/refman/8.4/en/csv-storage-engine.html)
5. [ARCHIVE](https://en.wikipedia.org/wiki/MySQL_Archive)
6. [BLACKHOLE](https://dev.mysql.com/doc/refman/9.7/en/blackhole-storage-engine.html)
7. [MRG_MYISAM](https://dev.mysql.com/doc/refman/9.7/en/merge-storage-engine.html)
8. [FEDERATED](https://dev.mysql.com/doc/refman/8.4/en/federated-storage-engine.html)
9. [PERFORMANCE_SCHEMA](https://dev.mysql.com/doc/refman/5.7/en/performance-schema.html)
