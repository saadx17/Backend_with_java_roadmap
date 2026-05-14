# What is a Database?
A **[Database](https://www.oracle.com/apac/database/what-is-database/)** is an organized collection of structured data stored electronically, designed for efficient *storage, retrieval, and manipulation.*

**Think of it like:**
- A filing cabinet with labeled folders
- An Excel workbook with multiple sheets
- A phone book with structured entries

**Example:**
A school has data about:
- Students (name, age, grade)
- Teachers (name, subject, salary)
- Classes (room number, timing)

##### **The Core Components:**

1. **The Data:** The actual information being stored, whether that's customer names, bank transaction records, or the inventory of a shoe store.

2. **The Database Management System (DBMS):** A Database Management System is software designed to efficiently create, store, manage, and retrieve data, serving as an interface between users and the database. It ensures data security, integrity, and consistency while reducing redundancy across applications. Popular systems include MySQL, PostgreSQL, Oracle, and MongoDB.

3. **The Applications:** The software, websites, or apps you use every day (like a banking app or an online store) that talk to the DBMS to get the information they need to show you.

#### Types of Databases
Databases are organized collections of data, primarily categorized into relational (==SQL==) for structured data and non-relational (==NoSQL==) for unstructured data.
###### 1. ==Relational== Database
A relational database is a structured system for storing and managing data in ==tables with rows and columns==, utilizing structured query language (SQL) for data retrieval and manipulation.
```
Structure: Tables with rows and columns
┌──────────┬──────────┬─────────┐
│ ID       │ Name     │ Age     │
├──────────┼──────────┼─────────┤
│ 1        │ Alice    │ 25      │
│ 2        │ Bob      │ 30.     │
└──────────┴──────────┴─────────┘ 

Examples: MySQL, PostgreSQL, Oracle, SQL Server Use Case: Banking, ERP, CRM systems
```

###### 2. ==NoSQL== Database
NoSQL databases ("==not only SQL==") are non-relational, distributed data management systems designed for high *scalability*, *flexibility*, and *handling* large volumes of unstructured or rapidly changing data.
Unlike SQL databases, they lack fixed, table-based schemas, enabling diverse data models like document, key-value, column-family, and graph, often prioritizing horizontal scaling and high performance over strict immediate consistency.
```
Types & Examples:
┌─────────────────┬──────────────────┬────────────────────┐
│ Type            │ Example          │ Use Case           │ ├─────────────────┼──────────────────┼────────────────────┤
│ Document        │ MongoDB          │ Content Management │
│ Key-Value       │ Redis            │ Caching, Sessions  │
│ Column-Family   │ Cassandra        │ Big Data           │
│ Graph           │ Neo4j            │ Social Networks    │ └─────────────────┴──────────────────┴────────────────────┘
```

```
MongoDB Document Example:
{
  "_id": "001",
  "name": "Alice",
  "orders": [
     {"item": "Laptop", "price": 999},
     {"item": "Mouse", "price": 29}
   ]
}
```

###### 3. ==In-Memory== Database
An in-memory database (IMDB) is a database management system that stores data entirely in main memory (==RAM==) rather than on disk, providing ultra-fast, low-latency data retrieval. By eliminating disk I/O, they power real-time ==analytics, gaming, IoT, and high-speed transactions==. Key options include Redis, SAP HANA, and [Oracle Database In-Memory](https://www.oracle.com/database/in-memory/).
```
Data stored in RAM (not disk)
┌─────────────────────────────────────┐
│ Disk DB: Read → 100ms               │
│ In-Memory: Read → 0.1ms             │
└─────────────────────────────────────┘
Examples: Redis, Memcached, VoltDB
Use Case: Real-time analytics, Gaming leaderboards
```

###### 4. ==Cloud== Database
A cloud database is a database designed, deployed, and accessed through a cloud environment. Cloud databases organize and store structured, unstructured, and semi-structured data just like ==traditional on-premises== databases. However, they also provide many of the same benefits of [cloud computing](https://cloud.google.com/learn/what-is-cloud-computing), including speed, scalability, agility, and reduced costs.
```
Hosted on cloud platforms
┌──────────────────────────────────────────┐
│ Provider     │ Service                   │
├──────────────┼───────────────────────────┤
│ AWS          │ RDS, DynamoDB, Aurora.    │
│ Google       │ Cloud SQL, BigQuery       │
│ Azure        │ Azure SQL, Cosmos DB      │ └──────────────┴───────────────────────────┘
Benefits: Scalable, Pay-as-you-go, Managed
```


# What is MySQL?
[MySQL](https://www.w3schools.com/mysql/mysql_intro.asp) is a widely-used, open-source relational database management system (==RDBMS==) that stores data in structured tables. It is commonly used for web development, powering popular platforms like WordPress, Drupal, and Facebook, and is a key component of the [LAMP](https://www.ibm.com/think/topics/lamp-stack) Stack.

##### What is RDBMS?
A Relational Database Management System ([RDBMS](https://www.w3schools.com/mysql/mysql_rdbms.asp)) is software that manages data organized into tables with rows and columns, allowing for secure, structured storage and complex querying.

**DBMS vs RDBMS:**
```
┌──────────────────────────────────────────────────────┐
│                         COMPARISON                   │
├─────────────────┬────────────────┬───────────────────┤
│ Feature         │ DBMS           │ RDBMS             │ ├─────────────────┼────────────────┼───────────────────┤
│ Data Storage    │ Files/Hierarch │ Tables (Relations)│
│ Relationships   │ Not enforced   │ Enforced via Keys │
│ Data Redundancy │ High           │ Low (Normalized)  │
│ SQL Support     │ No/Limited     │ Yes (Full SQL)    │
│ ACID            │ Not guaranteed │ Fully Supported   │
│ Normalization   │ Not Supported  │ Supported         │
│ Security        │ Low            │ High              │
│ Examples        │ XML, File Sys  │ MySQL, PostgreSQL │ └─────────────────┴────────────────┴───────────────────┘
```

# How MySQL Works?
MySQL is a (RDBMS) that ==functions through a **client-server model**==. It acts as a specialized manager that sits between an application (like a website) and the actual data files stored on a computer's disk.

**Works:** `You (Client)→ Write SQL Query → MySQL Server → Database → Result back to You (Client)`

**Flow:**
1. You type: "Show me all students with grade A"
2. MySQL reads your command
3. Goes to the database
4. Finds matching records
5. Returns the result to you

# Basic Terminology
##### Data vs. Information
In MySQL, the difference between **data** and **information** is defined by ==processing & context==, **data** is the raw material stored in the database, while **information** is the meaningful result obtained after querying or analyzing that data.

| Aspect              | Data                   | Information                |
| ------------------- | ---------------------- | -------------------------- |
| **Definition**      | Raw, unprocessed facts | Processed, meaningful data |
| **Form**            | Unorganized            | Organized                  |
| **Meaning**         | No context             | Has context                |
| **Example**         | `90, 45, 78, 23`       | `Average score = 59`       |
| **Dependency**      | Independent            | Depends on Data            |
| **Decision Making** | Cannot be used         | Can be used                |

##### Basics
These are the main basics:
- DATABASE      = A container that holds all your tables.
- TABLE              = A structure with rows and columns (like Excel sheet)
- ROW                = A single record (also called RECORD or TUPLE)
- COLUMN         = A field/attribute (like Name, Age, etc.)
- PRIMARY KEY = A unique identifier for each row.
- SQL                  = The language you use to talk to MySQL.

##### Visuals
```
DATABASE: company_db
│
├── SCHEMA: hr_schema
│ └── TABLE: employees
│ ┌──────────┬──────────┬────────────┬──────────┐
│ │ emp_id   │ name     │ department │ salary   │ ← COLUMNS
│ │ (INT)    │(VARCHAR) │ (VARCHAR)  │(DECIMAL) │ ← DATA TYPES
│ ├──────────┼──────────┼────────────┼──────────┤
│ │ 1        │ Alice    │ IT         │ 75000    │ ← ROW/RECORD
│ │ 2        │ Bob      │ HR         │ 65000    │ ← ROW/RECORD
│ │ 3        │ Carol    │ IT         │ 80000    │ ← ROW/RECORD
│ └──────────┴──────────┴────────────┴──────────┘
│       ↑ 
     INSTANCE = Current data at this moment
```