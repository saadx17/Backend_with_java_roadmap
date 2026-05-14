DDL commands are used to **define, modify, and delete database structures** (schemas, tables, indexes, etc.). These commands ==auto-commit==, once executed, changes are permanent and cannot be rolled back.

### `CREATE`
**Definition:** In MySQL, `CREATE` is used to build entirely new objects in the database. You use this to create the database itself, or the tables within it.

#### 1. ==`CREATE DATABASE`==
**Definition:** To create a database in MySQL, use the **`CREATE DATABASE`** statement followed by the name you wish to give your database.

###### Syntax
```
-- Basic
CREATE DATABASE company_db;
```

```
-- Safe way (no error if exists)
CREATE DATABASE IF NOT EXISTS company_db;
```

```
-- With character set
CREATE DATABASE company_db
   CHARACTER SET utf8mb4
   COLLATE utf8mb4_unicode_ci;
```

```   
-- Use Database
USE company_db;
```

```   
-- Verify
SHOW DATABASES;
```

###### The Character Set: ==`CHARACTER SET utf8mb4`==
A Character Set is the specific dictionary of letters, numbers, and symbols the database is allowed to understand and store.

- **The MySQL `utf8` trap:** Historically, MySQL had a character set called `utf8`. But it was flawed, it only allowed a maximum of 3 bytes per character. That meant it could store standard English and most European languages, but if a user tried to save a mathematical symbol, a complex Mandarin character, or a modern **emoji** (like 🚀 or 😂), the database would literally crash and throw a fatal error.

- **The Solution (`utf8mb4`):** The `mb4` stands for **"Most Bytes 4"**. This is MySQL's true, fully compliant Unicode character set. By explicitly stating this, you are guaranteeing your database can safely store absolutely any character or emoji a user types into your application without breaking.

###### The Collation: ==`COLLATE utf8mb4_unicode_ci`==
While the _Character Set_ dictates what letters the database can store, the **Collation** dictates the rules for how the database compares and sorts those letters.

 1. **`unicode`:** This tells MySQL to use official international Unicode standards for sorting. For example, it ensures that in a German application, the letter "ß" is correctly sorted as if it were "ss".

2. **`_ci` (Case-Insensitive):** This is the most important part. Because it ends in `ci`, the database will ignore capital letters when comparing text.

- If a user searches for `"smith"`, the database will return `"Smith"`, `"SMITH"`, and `"smith"`.
- If you put a `UNIQUE` constraint on an email column, and someone registers `John@gmail.com`, the `ci` collation ensures no one else can register `john@gmail.com` because the database sees them as the exact same string.

#### 2. ==`CREATE SCHEMA`==
**Definition:** A schema is a structured framework, plan, or model that defines the organization, relationships, and properties of data or concepts. In technology, it dictates how data is stored (databases) or interpreted (SEO/metadata), while in psychology, it represents mental frameworks used to organize knowledge and guide behavior.
###### Syntax
```
CREATE SCHEMA hr_schema;
```

#### 3. ==`CREATE TABLE`==
**Definition:** In MySQL, the `CREATE TABLE` statement is a fundamental SQL command used to **define and create a new table** within a database. It essentially acts as a blueprint that specifies how your data will be structured and stored.
##### Syntax
```
-- Create
CREATE TABLE table_name (
    column_name datatype [constraints],
    column_name datatype [constraints],
    ...
);
```

```
-- Verify
SHOW CREATE TABLE table_name; -- Shows full CREATE statement
```

##### Basic Examples
```
-- departments table
CREATE TABLE departments (
  dept_id INT PRIMARY KEY AUTO_INCREMENT,
  dept_name VARCHAR(100) NOT NULL UNIQUE,
  location VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```
-- employees table (with all constraint types)
CREATE TABLE employees (
  emp_id INT PRIMARY KEY AUTO_INCREMENT,
  first_name VARCHAR(50) NOT NULL,
  last_name VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL UNIQUE,
  phone CHAR(10),
  salary DECIMAL(10,2) DEFAULT 0.00,
  hire_date DATE NOT NULL,
  dept_id INT,
  status ENUM('active','inactive','terminated') DEFAULT 'active',
  age TINYINT UNSIGNED,
  
  -- Table-level constraints
  CONSTRAINT fk_dept
  FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
    ON DELETE SET NULL
    ON UPDATE CASCADE,
    
  CONSTRAINT chk_salary CHECK (salary >= 0),
  CONSTRAINT chk_age CHECK (age >= 18 AND age <= 100)
);
```
**Note:** `NOT NULL`, `UNIQUE`, `PRIMARY KEY`, `FOREIGN KEY`, etc. will be talked in [[Constraints]] & [[Keys]].

```
-- products table
CREATE TABLE products (
  product_id INT PRIMARY KEY AUTO_INCREMENT,
  product_name VARCHAR(200) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  stock INT DEFAULT 0,
  category VARCHAR(50),
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
      ON UPDATE CURRENT_TIMESTAMP
);
```

```
-- orders table
CREATE TABLE orders (
  order_id INT PRIMARY KEY AUTO_INCREMENT,
  emp_id INT NOT NULL,
  product_id INT NOT NULL,
  quantity INT NOT NULL DEFAULT 1,
  order_date DATE DEFAULT (CURRENT_DATE),
  total DECIMAL(10,2),
  
  FOREIGN KEY (emp_id) REFERENCES employees(emp_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

```
-- Verify tables created
SHOW TABLES;
DESCRIBE employees;          -- Show structure (DESCRIBE)
DESC employees;              -- Short form (DESC)
SHOW COLUMNS FROM employees;
```
##### Advanced Examples
Don't look at these if you're just interested in the basics.
###### CREATE TABLE with `LIKE`
```
CREATE TABLE employees_backup LIKE employees;
```
**Note:** When you use `LIKE`, MySQL creates an empty table that is a **perfect carbon copy** of the original's metadata. This includes:

- **Primary Keys:** It keeps your unique identifiers.
- **Indexes:** All performance-improving indexes are recreated.
- **Column Attributes:** It preserves `AUTO_INCREMENT`, `NOT NULL` constraints, and default values.
- **Storage Engine:** It ensures the new table uses the same [[Engines|Engine]] (e.g., InnoDB).

###### CREATE TABLE AS `SELECT`
```
CREATE TABLE employees_2024 AS SELECT * FROM employees
WHERE hire_date >= '2024-01-01';
```
**Note:** This is the most common use case. Writing a schema from scratch is tedious. This command automatically inherits If you need a "frozen" version of your 2024 hires for an annual report or audit, this command creates a standalone table (`employees_2024`) that won't change even if the original `employees` table is updated later.

While this is a great shortcut, there are a few things MySQL **does not** copy over automatically when using `AS SELECT`:

- **Indexes:** The new table will not have the Primary Keys or Indexes of the old one. It’s just "raw" data.
- **Auto-Increments:** The `AUTO_INCREMENT` property usually doesn't carry over.
- **Foreign Keys:** Constraints that link the original table to others are not recreated.

> **Pro Tip:** If you just want to copy the **structure** without any data, you can add a "false" condition like `WHERE 1=0`. This creates an empty shell of the table instantly.

```
CREATE TABLE employees_2024 AS SELECT * FROM employees
WHERE 1=0;
```

### `ALTER`
**Definition:** In MySQL, the **`ALTER`** command is a **Data Definition Language (DDL)** statement used to modify the structure of an existing database object without deleting or recreating it. While most commonly used with tables, it can also be used to change the characteristics of databases, users, views, and stored routines.

#### 1. ==`ALTER DATABASE`==

**Change Character Set:**
```
ALTER DATABASE db_name
CHARACTER SET utf8mb4;
```

**Change Collation:**
```
ALTER DATABASE db_name
COLLATE utf8mb4_unicode_ci;
```

**Set Read-Only Status:**
```
ALTER DATABASE db_name
READ ONLY = 1;                  -- 1 for Read-Only, 0 for Read-Write
```

**[Encryption](https://dev.mysql.com/doc/refman/9.7/en/innodb-data-encryption.html):**
```
ALTER DATABASE db_name
ENCRYPTION = 'Y';               -- {'Y' | 'N'} For Yes & No
```

#### 2. ==`ALTER TABLE`== - Add Column

```
ALTER TABLE employees
    ADD COLUMN middle_name VARCHAR(50);            -- added at end
```

```
ALTER TABLE employees
    ADD COLUMN ssn VARCHAR(20) AFTER last_name;    -- specific position (After)
```

```
ALTER TABLE employees
    ADD COLUMN employee_code VARCHAR(10) FIRST;    -- add at beginning
```

```
-- Add multiple columns at once
ALTER TABLE employees
    ADD COLUMN linkedin_url VARCHAR(200),
    ADD COLUMN github_url   VARCHAR(200),
    ADD COLUMN rating       TINYINT DEFAULT 3;
```

#### 3. ==`ALTER TABLE`== - Modify Column
```
ALTER TABLE employees
    MODIFY COLUMN phone VARCHAR(15);               -- was CHAR(10)
```

```
-- Change name + type (CHANGE keyword)
ALTER TABLE employees
    CHANGE COLUMN phone mobile_number VARCHAR(15); -- rename + modify
```

```
-- Only rename column (MySQL 8.0+)
ALTER TABLE employees
    RENAME COLUMN middle_name TO mid_name;
```

#### 4. ==`ALTER TABLE`== - Drop Column
```
ALTER TABLE employees
    DROP COLUMN mid_name;
```

```
ALTER TABLE employees
    DROP COLUMN linkedin_url,
    DROP COLUMN github_url;
```
#### 5. ==`ALTER TABLE`== - Rename Column
```
-- Using ALTER TABLE
ALTER TABLE employees RENAME TO staff;

ALTER TABLE staff RENAME TO employees;
```
#### 6. ==`ALTER TABLE`== - Constraints

##### `ALTER` + `ADD`
```
-- Add PRIMARY KEY
ALTER TABLE employees
    ADD PRIMARY KEY (emp_id);
```

```
-- Add FOREIGN KEY
ALTER TABLE employees
    ADD CONSTRAINT fk_dept
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id);
```

```
-- Add UNIQUE constraint
ALTER TABLE employees
    ADD CONSTRAINT uq_email UNIQUE (email);
```

```
-- Add CHECK constraint
ALTER TABLE employees
    ADD CONSTRAINT chk_salary CHECK (salary >= 0);
```

##### `ALTER` + `DROP`
```
-- Drop PRIMARY KEY
ALTER TABLE employees
    DROP PRIMARY KEY;
```

```
-- Drop FOREIGN KEY
ALTER TABLE employees
    DROP FOREIGN KEY fk_dept;
```

```
-- Drop INDEX (Unique creates an index)
ALTER TABLE employees
    DROP INDEX uq_email;
```

```
-- Drop CHECK
ALTER TABLE employees
    DROP CHECK chk_salary;
```

#### 7. ==`ALTER TABLE`== - Misc
```
-- Change table engine
ALTER TABLE employees ENGINE = InnoDB;
```
**Note:** `ENGINE` was discussed in [[Engines]].

```
-- Change auto_increment start value
ALTER TABLE employees AUTO_INCREMENT = 1000;
```

```
-- Change character set
ALTER TABLE employees
  CONVERT TO CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;
```

### `DROP`
**Definition:** In MySQL, **`DROP`** is a Data Definition Language (DDL) statement used to permanently delete existing database objects, including their data and structure. Unlike the `DELETE` command, which only removes specific rows, or `TRUNCATE`, which clears all rows but keeps the table "shell," `DROP` completely removes the object from the system.

**Note:** **Permanently** deletes database objects (structure + data). ==CANNOT== be rolled back in MySQL.

```
-- DROP DATABASE
DROP DATABASE company_db;

-- DROP DATABASE (Ultra Safe)
DROP DATABASE IF EXISTS company_db;
```

```
-- DROP TABLE
DROP TABLE employees;
```

```
-- Safe DROP TABLE (no error if not exists)
DROP TABLE IF EXISTS employees;
```

```
-- Drop multiple TABLE
DROP TABLE IF EXISTS order_items, orders, employees, departments;
```

```
-- DROP INDEX
DROP INDEX idx_email ON employees;
```

```
-- DROP VIEW
DROP VIEW IF EXISTS emp_summary;
```

```
-- DROP PROCEDURE
DROP PROCEDURE IF EXISTS get_employee;
```

### `TRUNCATE`
**Definition:** In MySQL, the term **truncate** refers to two distinct operations: a statement for emptying tables and a function for shortening numbers.
Removes **all rows** from table but **keeps structure**, Faster than DELETE for clearing all data.

```
-- TRUNCATE TABLE
TRUNCATE TABLE employees;
TRUNCATE employees;                  -- shorthand
```

```
-- TRUNCATE resets AUTO_INCREMENT
INSERT INTO employees ...            -- next emp_id starts at 1 again
```

```
-- DELETE does NOT reset AUTO_INCREMENT
DELETE FROM employees;               -- next emp_id continues from last value
```

```
┌──────────────┬──────────────────┬──────────────────┐
│ Feature      │ TRUNCATE         │ DELETE           │
├──────────────┼──────────────────┼──────────────────┤
│ Speed        │ Fast             │ Slow             │
│ WHERE clause │ Not supported    │ Supported        │
│ Rollback     │ No (DDL)         │ Yes (DML)        │
│ Triggers     │ Not fired        │ Fired            │
│ AUTO_INC     │ Reset to 1       │ Not reset        │
│ Logging      │ Minimal          │ Row by row       │
└──────────────┴──────────────────┴──────────────────┘
```

### `RENAME`
**Definition:** In MySQL, **Rename** refers to the operation of changing the identifier (name) of a database object, such as a table, column, view, or user account, without modifying the underlying data or structure.

```
-- RENAME TABLE
RENAME TABLE employees TO staff;
RENAME TABLE staff TO employees;  -- rename back
```

```
-- Rename multiple tables at once
RENAME TABLE
    employees   TO staff,
    departments TO divisions,
    orders      TO purchase_orders;
```

```
-- RENAME DATABASE (MySQL doesn't support directly)
-- Workaround:
CREATE DATABASE new_company_db;
-- Then recreate all tables and copy data
```
