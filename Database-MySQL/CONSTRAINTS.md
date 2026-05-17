# What are Constraints?
Constraints are **rules enforced on data columns** in a table.

They are used to:
- **Limit** the type of data that can go into a table
- **Ensure accuracy** and reliability of the data
- **Maintain data integrity**

Constraints can be defined at:
- **Column level** - applies to a single column
- **Table level** - applies to the whole table (multiple columns)

# Types of Constraints

| Constraint       | Description                                                |
| ---------------- | ---------------------------------------------------------- |
| `NOT NULL`       | Column cannot have NULL value                              |
| `UNIQUE`         | All values in column must be different                     |
| `PRIMARY KEY`    | Combination of NOT NULL + UNIQUE, uniquely identifies rows |
| `FOREIGN KEY`    | Links two tables together                                  |
| `CHECK`          | Ensures values satisfy a specific condition                |
| `DEFAULT`        | Sets a default value for a column                          |
| `AUTO_INCREMENT` | Automatically generates a unique number                    |
| `INDEX`          | Used to create and retrieve data quickly                   |

### 1. NOT NULL
Ensures that a column **cannot store NULL values**.

**Syntax at Table Creation:** 
```
CREATE TABLE employees (
  id INT NOT NULL,
  first_name VARCHAR(50) NOT NULL,
  last_name VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  phone VARCHAR(20)                -- This CAN be NULL
);
```

**Add NOT NULL to Existing Column:**
```
ALTER TABLE employees
MODIFY phone VARCHAR(20) NOT NULL;
```

**Remove NOT NULL:**
```
ALTER TABLE employees
MODIFY phone VARCHAR(20) NULL;
```

**Key Points:**
- By default, columns accept NULL values
- NOT NULL means you MUST provide a value during INSERT
- Cannot use NOT NULL with `DEFAULT NULL`

### 2. UNIQUE
Ensures all values in a column are **distinct** (no duplicates). It can also work as a [[Keys#3. Unique Key|Key]].

**Column-Level Syntax:**
```
CREATE TABLE users (
  id INT,
  email VARCHAR(100) UNIQUE,
  username VARCHAR(50) UNIQUE
);
```

**Table-Level Syntax - Named Constraint:**
```
CREATE TABLE users (
    id INT,
    email VARCHAR(100),
    username VARCHAR(50),
    CONSTRAINT uc_email UNIQUE (email),
    CONSTRAINT uc_username UNIQUE (username)
);
```

**Composite UNIQUE Constraint:**
```
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    CONSTRAINT uc_enrollment UNIQUE (student_id, course_id)
);
```

**Add UNIQUE to Existing Table:**
```
-- Method 1
ALTER TABLE users
ADD UNIQUE (email);

-- Method 2 (Named)
ALTER TABLE users
ADD CONSTRAINT uc_email UNIQUE (email);

-- Method 3 (Using Index)
CREATE UNIQUE INDEX idx_email ON users(email);
```

**Drop UNIQUE Constraint:**
```
-- Using Index Name
ALTER TABLE users
DROP INDEX uc_email;

-- OR
DROP INDEX uc_email ON users;
```

**Key Points**
- `UNIQUE` allows **multiple NULL** values (NULL ≠ NULL)
- A table can have **multiple** UNIQUE constraints
- MySQL automatically creates an **index** for UNIQUE columns
- UNIQUE constraint = UNIQUE INDEX internally

### 3. PRIMARY KEY
**Uniquely identifies each row** in a table. It's a combination of `NOT NULL` + `UNIQUE`. It is also a [[Keys#1. Primary Key (PK)|Key]].

**Column-level Syntax:**
```
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

**Table-level Syntax:**
```
CREATE TABLE students (
    student_id INT,
    name VARCHAR(100),
    email VARCHAR(100),
    PRIMARY KEY (student_id)
);
```

**Named Primary Key:**
```
CREATE TABLE students (
    student_id INT,
    name VARCHAR(100),
    CONSTRAINT pk_student PRIMARY KEY (student_id)
);
```

**Composite Primary Key:**
```
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Add PRIMARY KEY to Existing Table:**
```
ALTER TABLE students
ADD PRIMARY KEY (student_id);

-- Named
ALTER TABLE students
ADD CONSTRAINT pk_student PRIMARY KEY (student_id);
```

**Drop PRIMARY KEY:**
```
ALTER TABLE students
DROP PRIMARY KEY;
```

**Note:** *If the primary key column is `AUTO_INCREMENT`, you must first remove `AUTO_INCREMENT` before dropping the primary key.*
```
ALTER TABLE students
MODIFY student_id INT;  -- Remove AUTO_INCREMENT first

ALTER TABLE students
DROP PRIMARY KEY;
```

**Key Points:**
- A table can have **only ONE** primary key
- Primary key **cannot be NULL**
- Composite primary key can consist of **multiple columns**
- MySQL creates a **clustered index** on the primary key (InnoDB)
- Primary key determines the **physical storage order** of rows in InnoDB

### 4. FOREIGN KEY
Establishes a **link between two tables**. It refers to the [[#3. PRIMARY KEY|Primary Key]] or [[#2. UNIQUE|Unique Key]] of another table. It is also a [[Keys#2. Foreign Key (FK)|Key]].

**Terminology:**
- Parent Table (Referenced Table) ← has the PRIMARY KEY
- Child Table (Referencing Table) ← has the FOREIGN KEY

**Syntax at Table Creation:**
```
-- Parent Table
CREATE TABLE departments (
    dept_id INT PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL
);

-- Child Table
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id INT,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
```

**Named Foreign Key:**
```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id INT,
    CONSTRAINT fk_dept 
        FOREIGN KEY (dept_id) 
        REFERENCES departments(dept_id)
);
```

**Composite Foreign Key:**
```
CREATE TABLE order_item_details (
    detail_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    CONSTRAINT fk_order_item
        FOREIGN KEY (order_id, product_id)
        REFERENCES order_items(order_id, product_id)
);
```

**Referential Actions (ON DELETE / ON UPDATE):**
```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    dept_id INT,
    CONSTRAINT fk_dept 
        FOREIGN KEY (dept_id) 
        REFERENCES departments(dept_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

**All Referential Actions:**

|Action|ON DELETE Behavior|ON UPDATE Behavior|
|---|---|---|
|`RESTRICT`|Prevents deletion of parent row|Prevents update of parent key|
|`CASCADE`|Deletes child rows automatically|Updates child key automatically|
|`SET NULL`|Sets child FK to NULL|Sets child FK to NULL|
|`NO ACTION`|Same as RESTRICT (in MySQL)|Same as RESTRICT (in MySQL)|
|`SET DEFAULT`|Not supported in InnoDB|Not supported in InnoDB|

**Examples of Each Action:**
```
-- CASCADE: Delete department → all employees in that dept are deleted
FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
ON DELETE CASCADE ON UPDATE CASCADE;

-- SET NULL: Delete department → employees' dept_id becomes NULL
FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
ON DELETE SET NULL ON UPDATE SET NULL;

-- RESTRICT (default): Cannot delete department if employees exist
FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
ON DELETE RESTRICT ON UPDATE RESTRICT;
```

**Add Foreign Key to Existing Table:**
```
ALTER TABLE employees
ADD FOREIGN KEY (dept_id) REFERENCES departments(dept_id);

-- Named
ALTER TABLE employees
ADD CONSTRAINT fk_dept 
    FOREIGN KEY (dept_id) 
    REFERENCES departments(dept_id)
    ON DELETE CASCADE;
```

**Drop Foreign Key:**
```
ALTER TABLE employees
DROP FOREIGN KEY fk_dept;
```

**Dropping the foreign key does NOT drop the associated index. To drop the index too:**
```
ALTER TABLE employees
DROP FOREIGN KEY fk_dept,
DROP INDEX fk_dept;
```

**Check Existing Foreign Keys:**
```
-- Method 1
SHOW CREATE TABLE employees;

-- Method 2
SELECT * FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_NAME = 'employees' 
AND REFERENCED_TABLE_NAME IS NOT NULL;

-- Method 3
SELECT * FROM INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS
WHERE TABLE_NAME = 'employees';
```

**Temporarily Disable Foreign Key Checks:**
```
-- Disable
SET FOREIGN_KEY_CHECKS = 0;

-- Do your operations (bulk load, truncate, etc.)
TRUNCATE TABLE employees;
LOAD DATA INFILE 'data.csv' INTO TABLE employees;

-- Re-enable
SET FOREIGN_KEY_CHECKS = 1;
```

**Self-Referencing Foreign Key:**
```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    manager_id INT,
    CONSTRAINT fk_manager 
        FOREIGN KEY (manager_id) 
        REFERENCES employees(emp_id)
        ON DELETE SET NULL
);
```

**Key Points:**
- Foreign key column must have the **same data type** as the referenced column
- Referenced column must have an **index** (PRIMARY KEY or UNIQUE)
- Foreign keys are only supported in **InnoDB** storage engine
- **MyISAM** does NOT enforce foreign keys
- Foreign key column **can be NULL** (unless NOT NULL is specified)
- Default action is `RESTRICT`

### 5. CHECK
Ensures that values in a column satisfy a **specific condition**.

**Column-Level Syntax:**
```
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT CHECK (age >= 18),
    salary DECIMAL(10,2) CHECK (salary > 0)
);
```

**Table-Level Syntax (Named):**
```
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT,
    salary DECIMAL(10,2),
    hire_date DATE,
    end_date DATE,
    CONSTRAINT chk_age CHECK (age >= 18 AND age <= 65),
    CONSTRAINT chk_salary CHECK (salary > 0),
    CONSTRAINT chk_dates CHECK (end_date > hire_date)
);
```

**Complex CHECK Constraints:**
```
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    category ENUM('Electronics', 'Clothing', 'Food'),
    price DECIMAL(10,2),
    discount DECIMAL(5,2),
    status VARCHAR(20),
    
    CONSTRAINT chk_price CHECK (price > 0),
    CONSTRAINT chk_discount CHECK (discount >= 0 AND discount <= 100),
    CONSTRAINT chk_status CHECK (status IN ('active', 'inactive', 'discontinued')),
    CONSTRAINT chk_price_discount CHECK (price * (1 - discount/100) > 0)
);
```

**Add CHECK to Existing Table:**
```
ALTER TABLE employees
ADD CONSTRAINT chk_age CHECK (age >= 18);

ALTER TABLE employees
ADD CHECK (salary > 0);
```

**Drop CHECK Constraint:**
```
ALTER TABLE employees
DROP CHECK chk_age;

-- OR
ALTER TABLE employees
DROP CONSTRAINT chk_age;
```

**View CHECK Constraints:**
```
SELECT * FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS 
WHERE TABLE_NAME = 'employees' 
AND CONSTRAINT_TYPE = 'CHECK';

SELECT * FROM INFORMATION_SCHEMA.CHECK_CONSTRAINTS
WHERE CONSTRAINT_SCHEMA = 'your_database';
```

**Key Points:**
- **MySQL 8.0.16+** enforces CHECK constraints
- Earlier versions **silently ignore** them
- CHECK can reference **multiple columns** (table-level)
- Cannot reference **other tables**
- Cannot contain **subqueries**
- Cannot use **stored functions**, **user variables**, or **non-deterministic functions**
- NULL values pass CHECK constraints (NULL does not violate the condition)

### 6. DEFAULT
Sets a **default value** for a column when no value is specified during INSERT.

**Syntax at Table Creation:**
```
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'pending',
    quantity INT DEFAULT 1,
    is_active BOOLEAN DEFAULT TRUE,
    notes TEXT DEFAULT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**Expression as Default (MySQL 8.0.13+):**
```
CREATE TABLE products (
    id INT PRIMARY KEY,
    price DECIMAL(10,2),
    tax_rate DECIMAL(5,2) DEFAULT 0.10,
    uuid BINARY(16) DEFAULT (UUID_TO_BIN(UUID())),
    created_json JSON DEFAULT ('{}'),
    random_val FLOAT DEFAULT (RAND())
);
```

**Add/Change DEFAULT on Existing Column:**
```
-- Add default
ALTER TABLE orders
ALTER status SET DEFAULT 'processing';

-- OR using MODIFY
ALTER TABLE orders
MODIFY status VARCHAR(20) DEFAULT 'processing';
```

**Remove DEFAULT:**
```
ALTER TABLE orders
ALTER status DROP DEFAULT;
```

**Key Points:**
- DEFAULT value must match the column's **data type**
- `BLOB`, `TEXT`, `GEOMETRY`, and `JSON` columns **cannot have literal defaults** (use expressions in MySQL 8.0.13+)
- `CURRENT_TIMESTAMP` is allowed for `DATETIME`/`TIMESTAMP` columns
- `ON UPDATE CURRENT_TIMESTAMP` auto-updates the value on row modification
- If you INSERT with an explicit NULL, the DEFAULT is **NOT** used (you get NULL)

```
-- These use the DEFAULT:
INSERT INTO orders (id) VALUES (1);
INSERT INTO orders (id, status) VALUES (2, DEFAULT);

-- This does NOT use DEFAULT (gets NULL if allowed):
INSERT INTO orders (id, status) VALUES (3, NULL);
```

### 7. AUTO_INCREMENT
Automatically generates a **unique sequential number** for a column.

**Syntax:**
```
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100)
);
```

**With Custom Starting Value:**
```
CREATE TABLE invoices (
    invoice_id INT AUTO_INCREMENT PRIMARY KEY,
    amount DECIMAL(10,2)
) AUTO_INCREMENT = 1000;
```

**Change AUTO_INCREMENT Value:**
```
ALTER TABLE invoices AUTO_INCREMENT = 5000;
```

**Check Current AUTO_INCREMENT Value:**
```
-- Method 1
SHOW TABLE STATUS WHERE Name = 'users';

-- Method 2
SELECT AUTO_INCREMENT 
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_NAME = 'users'
AND TABLE_SCHEMA = 'your_database';

-- Method 3: Last inserted auto_increment value
SELECT LAST_INSERT_ID();
```

**AUTO_INCREMENT with BIGINT:**
```
CREATE TABLE big_table (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    data VARCHAR(255)
);
```

**Key Points:**
- Only **one** AUTO_INCREMENT column per table
- Must be **indexed** (usually PRIMARY KEY)
- Must be **numeric** type (INT, BIGINT, etc.)
- Values are **NOT reused** after deletion (by default)
- `TRUNCATE TABLE` **resets** AUTO_INCREMENT to starting value
- `DELETE FROM table` does **NOT reset** AUTO_INCREMENT
- Default increment is 1; configurable with:
```
-- Change increment step (globally)
SET @@auto_increment_increment = 2;

-- Change starting offset (for multi-master replication)
SET @@auto_increment_offset = 1;
```

**InnoDB Behavior (MySQL 8.0+):**
- AUTO_INCREMENT counter is **persistent** across server restarts
- Before 8.0, it was recalculated on restart as `MAX(id) + 1`

### 8. INDEX (As a Constraint/Optimization)
While not a traditional "constraint," [[Indexes]] are closely related and often discussed together.

**Types of Indexes:**
```
-- Regular Index
CREATE INDEX idx_name ON employees(last_name);

-- Unique Index (acts like UNIQUE constraint)
CREATE UNIQUE INDEX idx_email ON employees(email);

-- Composite Index
CREATE INDEX idx_name_dept ON employees(last_name, dept_id);

-- Full-Text Index
CREATE FULLTEXT INDEX idx_description ON products(description);

-- Spatial Index
CREATE SPATIAL INDEX idx_location ON stores(location);

-- Prefix Index (for long strings)
CREATE INDEX idx_name ON employees(last_name(10));
```

**Index in CREATE TABLE:**
```
CREATE TABLE employees (
    id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    dept_id INT,
    
    INDEX idx_last_name (last_name),
    UNIQUE INDEX idx_email (email),
    INDEX idx_dept (dept_id),
    INDEX idx_full_name (first_name, last_name)
);
```

**Drop Index:**
```
DROP INDEX idx_name ON employees;

ALTER TABLE employees DROP INDEX idx_name;
```

**View Indexes:**
```
SHOW INDEX FROM employees;

SHOW INDEXES FROM employees;

SHOW KEYS FROM employees
```


# Managing Constraints

### View All Constraints on a Table
```
-- Method 1: SHOW CREATE TABLE
SHOW CREATE TABLE employees\G

-- Method 2: INFORMATION_SCHEMA
SELECT 
    CONSTRAINT_NAME,
    CONSTRAINT_TYPE
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS
WHERE TABLE_SCHEMA = 'your_database'
AND TABLE_NAME = 'employees';

-- Method 3: All constraint details
SELECT 
    tc.CONSTRAINT_NAME,
    tc.CONSTRAINT_TYPE,
    kcu.COLUMN_NAME,
    kcu.REFERENCED_TABLE_NAME,
    kcu.REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.TABLE_CONSTRAINTS tc
JOIN INFORMATION_SCHEMA.KEY_COLUMN_USAGE kcu
    ON tc.CONSTRAINT_NAME = kcu.CONSTRAINT_NAME
    AND tc.TABLE_SCHEMA = kcu.TABLE_SCHEMA
WHERE tc.TABLE_SCHEMA = 'your_database'
AND tc.TABLE_NAME = 'employees';
```

### Summary of Adding Constraints
```
-- NOT NULL
ALTER TABLE t MODIFY col datatype NOT NULL;

-- UNIQUE
ALTER TABLE t ADD CONSTRAINT uc_name UNIQUE (col);

-- PRIMARY KEY
ALTER TABLE t ADD CONSTRAINT pk_name PRIMARY KEY (col);

-- FOREIGN KEY
ALTER TABLE t ADD CONSTRAINT fk_name FOREIGN KEY (col) REFERENCES parent(col);

-- CHECK
ALTER TABLE t ADD CONSTRAINT chk_name CHECK (condition);

-- DEFAULT
ALTER TABLE t ALTER col SET DEFAULT value;
ALTER TABLE t MODIFY col datatype DEFAULT value;
```

### Summary of Dropping Constraints
```
-- NOT NULL
ALTER TABLE t MODIFY col datatype NULL;

-- UNIQUE
ALTER TABLE t DROP INDEX constraint_name;

-- PRIMARY KEY
ALTER TABLE t DROP PRIMARY KEY;

-- FOREIGN KEY
ALTER TABLE t DROP FOREIGN KEY constraint_name;

-- CHECK
ALTER TABLE t DROP CHECK constraint_name;

-- DEFAULT
ALTER TABLE t ALTER col DROP DEFAULT;
```

### Full Schema with All Constraints
```
-- ================================================
-- COMPLETE EXAMPLE DATABASE WITH ALL CONSTRAINTS
-- ================================================

CREATE DATABASE IF NOT EXISTS company;
USE company;

-- Parent Table 1: Departments
CREATE TABLE departments (
    dept_id INT AUTO_INCREMENT PRIMARY KEY,
    dept_name VARCHAR(100) NOT NULL UNIQUE,
    budget DECIMAL(15,2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_budget CHECK (budget >= 0)
);

-- Parent Table 2: Job Titles
CREATE TABLE job_titles (
    job_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(100) NOT NULL UNIQUE,
    min_salary DECIMAL(10,2) NOT NULL,
    max_salary DECIMAL(10,2) NOT NULL,
    
    CONSTRAINT chk_salary_range CHECK (max_salary >= min_salary),
    CONSTRAINT chk_min_salary CHECK (min_salary > 0)
);

-- Child Table: Employees
CREATE TABLE employees (
    emp_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    hire_date DATE NOT NULL DEFAULT (CURRENT_DATE),
    salary DECIMAL(10,2) NOT NULL,
    dept_id INT,
    job_id INT NOT NULL,
    manager_id INT,
    status ENUM('active', 'inactive', 'terminated') DEFAULT 'active',
    
    -- UNIQUE Constraints
    CONSTRAINT uc_email UNIQUE (email),
    
    -- FOREIGN KEY Constraints
    CONSTRAINT fk_emp_dept 
        FOREIGN KEY (dept_id) 
        REFERENCES departments(dept_id)
        ON DELETE SET NULL
        ON UPDATE CASCADE,
    
    CONSTRAINT fk_emp_job
        FOREIGN KEY (job_id)
        REFERENCES job_titles(job_id)
        ON DELETE RESTRICT
        ON UPDATE CASCADE,
    
    CONSTRAINT fk_emp_manager
        FOREIGN KEY (manager_id)
        REFERENCES employees(emp_id)
        ON DELETE SET NULL
        ON UPDATE CASCADE,
    
    -- CHECK Constraints
    CONSTRAINT chk_emp_salary CHECK (salary > 0),
    CONSTRAINT chk_emp_email CHECK (email LIKE '%@%.%'),
    
    -- Indexes
    INDEX idx_last_name (last_name),
    INDEX idx_dept (dept_id),
    INDEX idx_full_name (first_name, last_name)
);

-- Junction Table: Projects
CREATE TABLE projects (
    project_id INT AUTO_INCREMENT PRIMARY KEY,
    project_name VARCHAR(200) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE,
    budget DECIMAL(15,2) DEFAULT 0,
    
    CONSTRAINT chk_project_dates CHECK (end_date IS NULL OR end_date >= start_date),
    CONSTRAINT chk_project_budget CHECK (budget >= 0)
);

-- Junction/Bridge Table: Employee-Project Assignment
CREATE TABLE project_assignments (
    emp_id INT,
    project_id INT,
    assigned_date DATE DEFAULT (CURRENT_DATE),
    role VARCHAR(50) DEFAULT 'member',
    hours_allocated DECIMAL(5,1) DEFAULT 40.0,
    
    -- Composite Primary Key
    PRIMARY KEY (emp_id, project_id),
    
    -- Foreign Keys
    CONSTRAINT fk_pa_emp
        FOREIGN KEY (emp_id)
        REFERENCES employees(emp_id)
        ON DELETE CASCADE,
    
    CONSTRAINT fk_pa_project
        FOREIGN KEY (project_id)
        REFERENCES projects(project_id)
        ON DELETE CASCADE,
    
    -- Check Constraints
    CONSTRAINT chk_hours CHECK (hours_allocated > 0 AND hours_allocated <= 168),
    CONSTRAINT chk_role CHECK (role IN ('lead', 'member', 'reviewer', 'consultant'))
);

-- ================================================
-- INSERT SAMPLE DATA
-- ================================================

INSERT INTO departments (dept_name, budget) VALUES
('Engineering', 500000.00),
('Marketing', 300000.00),
('Human Resources', 200000.00);

INSERT INTO job_titles (title, min_salary, max_salary) VALUES
('Software Engineer', 60000, 150000),
('Marketing Manager', 50000, 120000),
('HR Specialist', 40000, 90000);

INSERT INTO employees (first_name, last_name, email, salary, dept_id, job_id) VALUES
('John', 'Doe', 'john.doe@company.com', 85000, 1, 1),
('Jane', 'Smith', 'jane.smith@company.com', 95000, 1, 1),
('Bob', 'Johnson', 'bob.j@company.com', 75000, 2, 2);

-- Set manager
UPDATE employees SET manager_id = 2 WHERE emp_id = 1;
```

