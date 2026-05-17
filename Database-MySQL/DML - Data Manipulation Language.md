DML manipulates **data** inside tables, Changes CAN be rolled back (within a transaction).

### The `INSERT`
**Definition:** In MySQL, "insert" typically refers to the **`INSERT` statement**, which is a Data Manipulation Language (DML) command used to add new rows of data into a database table.

##### Basic Syntax
```
-- Method 1: Specify column names (RECOMMENDED)
INSERT INTO employees (first_name, last_name, email, salary, hire_date, dept_id)
VALUES ('Alice', 'Johnson', 'alice@company.com', 75000.00, '2022-01-15', 1);
```

```
-- Method 2: Without column names (must match ALL columns in order)
INSERT INTO employees
VALUES (NULL, 'Bob', 'Smith', 'bob@company.com', '9876543210',
        65000.00, '2021-06-20', 1, 'active', 28, NULL);
```

```
-- Method 3: SET syntax (Slow & Takes Time)
INSERT INTO employees
SET first_name = 'Carol',
    last_name  = 'Williams',
    email      = 'carol@company.com',
    salary     = 80000.00,
    hire_date  = '2023-03-10',
    dept_id    = 2;
```

##### `INSERT` multiple rows (Batch Insert)
```
INSERT INTO employees (first_name, last_name, email, salary, hire_date, dept_id)
VALUES
    ('David',  'Brown',   'david@company.com',  70000, '2020-09-01', 3),
    ('Emma',   'Davis',   'emma@company.com',   85000, '2019-04-15', 1),
    ('Frank',  'Miller',  'frank@company.com',  60000, '2023-07-22', 4),
    ('Grace',  'Wilson',  'grace@company.com',  90000, '2018-11-30', 1),
    ('Henry',  'Moore',   'henry@company.com',  55000, '2024-01-10', 5),
    ('Ivy',    'Taylor',  'ivy@company.com',    72000, '2022-08-05', 2),
    ('Jack',   'Anderson','jack@company.com',   68000, '2021-12-01', 3);
```

##### `INSERT` with `SELECT` (Copy Data) - Advanced
```
-- Copy all data
INSERT INTO employees_backup
SELECT * FROM employees;
```

```
-- Copy filtered data
INSERT INTO employees_2024
SELECT * FROM employees
WHERE hire_date >= '2024-01-01';
```

```
-- Copy with transformation
INSERT INTO employees (first_name, last_name, email, salary, hire_date, dept_id)
SELECT first_name, last_name,
       CONCAT(first_name, '.new@company.com'),   -- modified email
       salary * 1.10,                              -- 10% raise
       CURDATE(),
       dept_id
FROM employees_backup
WHERE dept_id = 1;
```

##### `INSERT` or `IGNORE` / `ON DUPLICATE KEY` - Advanced
```
-- Ignore error if duplicate key
INSERT IGNORE INTO employees (emp_id, first_name, email)
VALUES (1, 'Alice', 'alice@company.com');       -- won't error if emp_id=1 exists
```

```
-- Update if duplicate key exists (UPSERT)
INSERT INTO employees (emp_id, first_name, email, salary)
VALUES (1, 'Alice', 'alice@company.com', 80000)
ON DUPLICATE KEY UPDATE
    salary     = VALUES(salary),                 -- update salary
    first_name = VALUES(first_name);             -- update name
```

```
-- Modern syntax (MySQL 8.0.19+)
INSERT INTO employees (emp_id, first_name, salary)
VALUES (1, 'Alice', 80000)
AS new_vals
ON DUPLICATE KEY UPDATE
    salary = new_vals.salary;
```

##### Check inserted data
```
SELECT LAST_INSERT_ID();        -- ID of last inserted row SELECT
ROW_COUNT();                    -- Rows affected by last query
```

### The `UPDATE`
**Definition:** In MySQL, **`UPDATE`** is a Data Manipulation Language (DML) statement used to modify existing records in a table. Unlike the `ALTER` command, which changes the structure of a table (like adding a column), `UPDATE` only changes the data values within those columns.

##### Basic Syntax
```
-- Update single row
UPDATE employees
SET salary = 80000
WHERE emp_id = 1;

-- Update multiple columns
UPDATE employees
SET salary     = 85000,
    dept_id    = 2,
    status     = 'active'
WHERE emp_id = 1;
```

##### `UPDATE` with Expressions
```
-- Give 10% raise to all IT employees
UPDATE employees
SET salary = salary * 1.10
WHERE dept_id = 1;
```

```
-- Give 5% raise to employees hired before 2022
UPDATE employees
SET salary = salary * 1.05
WHERE hire_date < '2022-01-01';

-- Update using CASE (conditional update)
UPDATE employees
SET salary = CASE
    WHEN dept_id = 1 THEN salary * 1.15   -- IT gets 15%
    WHEN dept_id = 2 THEN salary * 1.10   -- HR gets 10%
    WHEN dept_id = 3 THEN salary * 1.08   -- Finance gets 8%
    ELSE salary * 1.05                     -- Others get 5%
END;
```

```
-- Update string values
UPDATE employees
SET email = LOWER(email);             -- make all emails lowercase
```

```
UPDATE employees
SET first_name = CONCAT(UPPER(LEFT(first_name,1)),
                        LOWER(SUBSTRING(first_name,2)));  -- Capitalize
```
**Note:** `CONCAT`, `LOWER` etc. will be discussed in [[Functions]].

##### `UPDATE` multiple rows
```
UPDATE products
SET price = price * 1.05;             -- 5% price increase for ALL products

-- Update with IN clause
UPDATE employees
SET dept_id = 3
WHERE emp_id IN (1, 2, 3);

-- Update with BETWEEN
UPDATE employees
SET status = 'inactive'
WHERE hire_date BETWEEN '2018-01-01' AND '2019-12-31';
```

##### `UPDATE` with `Join` (Update from another table)
```
-- Update employees salary based on department budget
UPDATE employees e
JOIN departments d ON e.dept_id = d.dept_id
SET e.salary = e.salary * 1.10
WHERE d.dept_name = 'Information Technology';

-- Update using subquery
UPDATE employees
SET salary = salary * 1.20
WHERE dept_id = (
    SELECT dept_id FROM departments
    WHERE dept_name = 'Finance'
);
```

##### `UPDATE` with `LIMIT` (Update only N rows)
```
UPDATE employees
SET status = 'inactive'
WHERE status = 'active'
LIMIT 3;                          -- Only update first 3 matching rows
```

##### Safe `UPDATE` mode
```
-- MySQL safe mode prevents UPDATE without WHERE
SET SQL_SAFE_UPDATES = 0;                    -- Disable safe mode
UPDATE employees SET salary = 50000;         -- Update ALL rows
SET SQL_SAFE_UPDATES = 1;                    -- Re-enable safe mode
```

### The `DELETE`
**Definition:** In MySQL, **`DELETE`** is a Data Manipulation Language (DML) statement used to remove existing records (rows) from a table. It allows you to target specific rows using a condition or remove all rows while keeping the table's structure, constraints, and indexes intact.

##### Basic Syntax
```
-- Delete single row
DELETE FROM employees
WHERE emp_id = 1;

-- Delete with multiple conditions
DELETE FROM employees
WHERE dept_id = 5
AND   status  = 'inactive';
```

##### `DELETE` with various `WHERE` condition
```
-- Delete using IN
DELETE FROM employees
WHERE emp_id IN (10, 11, 12);

-- Delete using BETWEEN
DELETE FROM employees
WHERE hire_date BETWEEN '2018-01-01' AND '2019-12-31';

-- Delete using LIKE
DELETE FROM employees
WHERE email LIKE '%@oldcompany.com';

-- Delete using comparison
DELETE FROM products
WHERE stock = 0;               -- Delete out-of-stock products

DELETE FROM employees
WHERE salary < 30000;          -- Delete low salary records
```

##### `DELETE` with `JOIN`
```
-- Delete employees from specific department
DELETE e
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Operations';

-- Delete with multiple table join
DELETE e
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
WHERE d.location = 'Phoenix'
AND   e.status   = 'inactive';
```

##### `DELETE` with Subquery
```
-- Delete employees from a specific department (using subquery)
DELETE FROM employees
WHERE dept_id = (
    SELECT dept_id FROM departments
    WHERE dept_name = 'Operations'
);

-- Delete employees with no orders
DELETE FROM employees
WHERE emp_id NOT IN (
    SELECT DISTINCT emp_id FROM orders
);
```

##### `DELETE` with `LIMIT`
```
-- Delete only first 5 matching rows
DELETE FROM employees
WHERE status = 'inactive'
ORDER BY hire_date ASC
LIMIT 5;
```

##### `Delete` ALL rows (3 ways)
```
DELETE FROM employees;         -- DML: slow, logged, rollback possible
TRUNCATE TABLE employees;      -- DDL: fast, resets AUTO_INCREMENT
DROP TABLE employees;          -- Removes table entirely
```

##### Soft `DELETE`
```
-- Instead of actually deleting, mark as deleted
ALTER TABLE employees
    ADD COLUMN is_deleted  BOOLEAN   DEFAULT FALSE,
    ADD COLUMN deleted_at  TIMESTAMP NULL;

-- "Delete" (soft)
UPDATE employees
SET is_deleted = TRUE,
    deleted_at = CURRENT_TIMESTAMP
WHERE emp_id = 5;

-- Query active records only
SELECT * FROM employees WHERE is_deleted = FALSE;
```
