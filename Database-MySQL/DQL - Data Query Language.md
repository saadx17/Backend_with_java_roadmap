Data Query Language (DQL) is a component of SQL (Structured Query Language) focused solely on retrieving or fetching data from a database. It consists of only one core command, `SELECT`, which allows users to query specific data from tables without modifying the underlying database structure.
**Note:** DQL is all about **retrieving** data. `SELECT` is the most powerful and used SQL command.

### The `SELECT` Basic
```
SELECT   [DISTINCT] column(s)         -- What to show
FROM     table_name                    -- Where data is
JOIN     other_table ON condition      -- Combine tables
WHERE    condition                     -- Filter rows
GROUP BY column(s)                     -- Group rows
HAVING   condition                     -- Filter groups
ORDER BY column(s) [ASC|DESC]         -- Sort results
LIMIT    n OFFSET m;                  -- Limit results
```

### The `SELECT` Syntax
```
-- Select all columns
SELECT * FROM employees;
```

```
-- Select specific columns
SELECT first_name, last_name, salary FROM employees;
```

```
-- Select with alias
SELECT
    first_name              AS 'First Name',
    last_name               AS 'Last Name',
    salary                  AS 'Monthly Salary',
    salary * 12             AS 'Annual Salary',
    CONCAT(first_name,' ',last_name) AS full_name
FROM employees;
```

### The `SELECT DISTINCT`
```
SELECT DISTINCT dept_id    FROM employees;       -- Unique dept_ids
SELECT DISTINCT status     FROM employees;       -- Unique statuses
SELECT DISTINCT dept_id, status FROM employees;  -- Unique combinations
```

### The `WHERE` clause - Filtering
```
-- Comparison operators
SELECT * FROM employees WHERE salary  > 70000;
SELECT * FROM employees WHERE salary  < 70000;
SELECT * FROM employees WHERE salary >= 70000;
SELECT * FROM employees WHERE salary <= 70000;
SELECT * FROM employees WHERE salary  = 70000;
SELECT * FROM employees WHERE salary != 70000;
SELECT * FROM employees WHERE salary <> 70000;   -- same as !=
```

```
-- Logical operators
SELECT * FROM employees WHERE dept_id = 1 AND salary > 70000;
SELECT * FROM employees WHERE dept_id = 1 OR  dept_id = 2;
SELECT * FROM employees WHERE NOT status = 'inactive';
```

```
-- BETWEEN (inclusive)
SELECT * FROM employees WHERE salary  BETWEEN 60000 AND 80000;
SELECT * FROM employees WHERE hire_date BETWEEN '2022-01-01' AND '2023-12-31';
```

```
-- IN / NOT IN
SELECT * FROM employees WHERE dept_id IN (1, 2, 3);
SELECT * FROM employees WHERE status  NOT IN ('inactive', 'terminated');
```

```
-- LIKE (pattern matching)
SELECT * FROM employees WHERE first_name LIKE 'A%';   -- starts with A
SELECT * FROM employees WHERE first_name LIKE '%a';   -- ends with a
SELECT * FROM employees WHERE first_name LIKE '%li%'; -- contains 'li'
SELECT * FROM employees WHERE first_name LIKE 'A___'; -- A + 3 any chars
SELECT * FROM employees WHERE email LIKE '%@company.com';
```

```
-- IS NULL / IS NOT NULL
SELECT * FROM employees WHERE dept_id IS NULL;
SELECT * FROM employees WHERE dept_id IS NOT NULL;
SELECT * FROM employees WHERE phone   IS NULL;
```

### The `ORDER BY` - Sorting
```
SELECT * FROM employees
ORDER BY salary ASC;               -- ascending (default)

SELECT * FROM employees
ORDER BY salary DESC;              -- descending

SELECT * FROM employees
ORDER BY dept_id ASC, salary DESC; -- multiple columns

SELECT * FROM employees
ORDER BY first_name;               -- alphabetical

SELECT * FROM employees
ORDER BY 3;                        -- by column position
```

### The `LIMIT` & `OFFSET`
```
SELECT * FROM employees LIMIT 5;              -- First 5 rows
SELECT * FROM employees LIMIT 5 OFFSET 5;    -- Skip 5, get next 5
SELECT * FROM employees LIMIT 5, 5;          -- Same: OFFSET, LIMIT
```

```
-- Pagination example (page 3, 10 records per page)
SET @page = 3;
SET @per_page = 10;
SELECT * FROM employees
LIMIT 10 OFFSET 20;                          -- (page-1) * per_page = offset
```

```
-- Top N salary
SELECT * FROM employees
ORDER BY salary DESC
LIMIT 5;                                     -- Top 5 highest paid
```

### The Aggregate Function
```
SELECT COUNT(*) AS total_employees FROM employees;

SELECT COUNT(dept_id) AS has_dept FROM employees;  -- excludes NULL

SELECT COUNT(DISTINCT dept_id) AS unique_depts FROM employees;
```

```
SELECT SUM(salary) AS total_payroll FROM employees;
SELECT AVG(salary) AS avg_salary FROM employees;
SELECT MIN(salary) AS lowest_salary FROM employees;
SELECT MAX(salary) AS highest_salary FROM employees;
SELECT MIN(hire_date) AS first_hired FROM employees;
SELECT MAX(hire_date) AS latest_hired FROM employees;
```

```
-- Multiple aggregates
SELECT
    COUNT(*)        AS total,
    AVG(salary)     AS avg_sal,
    MIN(salary)     AS min_sal,
    MAX(salary)     AS max_sal,
    SUM(salary)     AS total_payroll
FROM employees
WHERE status = 'active';
```

### The `GROUP BY` - Grouping Rows

```
-- Count employees per department
SELECT dept_id, COUNT(*) AS emp_count
FROM employees
GROUP BY dept_id;
```

```
-- Salary stats per department
SELECT
    dept_id,
    COUNT(*)    AS emp_count,
    AVG(salary) AS avg_salary,
    MAX(salary) AS max_salary,
    SUM(salary) AS total_cost
FROM employees
GROUP BY dept_id
ORDER BY avg_salary DESC;
```

```
-- Group by multiple columns
SELECT dept_id, status, COUNT(*) AS count
FROM employees
GROUP BY dept_id, status;
```

```
-- GROUP BY with JOIN
SELECT
    d.dept_name,
    COUNT(e.emp_id) AS emp_count,
    ROUND(AVG(e.salary), 2) AS avg_salary
FROM employees e
JOIN departments d ON e.dept_id = d.dept_id
GROUP BY d.dept_id, d.dept_name
ORDER BY emp_count DESC;
```

### The `HAVING` - Filter Groups

###### Departments with more than 2 employees
```
SELECT dept_id, COUNT(*) AS emp_count
FROM employees
GROUP BY dept_id
HAVING COUNT(*) > 2;

```

###### Departments with average salary > 70000
```
SELECT dept_id, ROUND(AVG(salary),2) AS avg_sal
FROM employees
GROUP BY dept_id
HAVING AVG(salary) > 70000
ORDER BY avg_sal DESC;
```

###### Combined `WHERE` and `HAVING`

```
SELECT dept_id,
       COUNT(*)    AS emp_count,
       AVG(salary) AS avg_salary
FROM employees
WHERE status = 'active'           -- filter rows first
GROUP BY dept_id
HAVING COUNT(*) >= 2              -- then filter groups
ORDER BY avg_salary DESC;
```

###### `WHERE` vs. `HAVING`:

```
┌──────────┬────────────────────────┬─────────────────────────┐
│ Feature  │ WHERE                  │ HAVING                  │
├──────────┼────────────────────────┼─────────────────────────┤
│ Filters  │ Individual rows        │ Groups                  │
│ Used with│ No GROUP BY needed     │ Requires GROUP BY       │
│ Aggreg.  │ Cannot use directly    │ Can use aggregates      │
│ Order    │ Executes before GROUP  │ Executes after GROUP    │
└──────────┴────────────────────────┴─────────────────────────┘
```

### The `SELECT` with Functions

###### String Functions
```
SELECT
    UPPER(first_name)                        AS upper_name,
    LOWER(email)                             AS lower_email,
    LENGTH(first_name)                       AS name_length,
    CONCAT(first_name, ' ', last_name)       AS full_name,
    CONCAT_WS(', ', last_name, first_name)   AS formatted_name,
    SUBSTRING(first_name, 1, 3)              AS name_abbr,
    TRIM('  hello  ')                        AS trimmed,
    REPLACE(email, '@company.com', '@new.com') AS new_email,
    LEFT(first_name, 3)                      AS first3,
    RIGHT(last_name, 3)                      AS last3,
    LPAD(emp_id, 5, '0')                     AS emp_code   -- 00001
FROM employees;
```

###### Date Functions
```
SELECT
    hire_date,
    YEAR(hire_date)                          AS hire_year,
    MONTH(hire_date)                         AS hire_month,
    DAY(hire_date)                           AS hire_day,
    MONTHNAME(hire_date)                     AS month_name,
    DAYNAME(hire_date)                       AS day_name,
    DATEDIFF(CURDATE(), hire_date)           AS days_employed,
    TIMESTAMPDIFF(YEAR, hire_date, CURDATE()) AS years_employed,
    DATE_FORMAT(hire_date, '%d-%b-%Y')       AS formatted_date,
    DATE_ADD(hire_date, INTERVAL 1 YEAR)     AS next_anniversary,
    NOW()                                    AS current_datetime,
    CURDATE()                                AS today
FROM employees;
```

###### Numeric Functions
```
SELECT
    salary,
    ROUND(salary, 2)                         AS rounded,
    CEIL(salary / 1000)                      AS thousands_ceil,
    FLOOR(salary / 1000)                     AS thousands_floor,
    ABS(-salary)                             AS absolute,
    MOD(emp_id, 2)                           AS is_even,
    POWER(2, 10)                             AS two_to_ten,
    SQRT(salary)                             AS sqrt_salary
FROM employees;
```

###### Conditional Functions
```
SELECT first_name, salary,
IF(salary > 70000, 'High', 'Low') AS salary_band,

CASE
  WHEN salary >= 90000 THEN 'Senior'
  WHEN salary >= 70000 THEN 'Mid-Level'
  WHEN salary >= 50000 THEN 'Junior'
  ELSE 'Intern'
END AS level,

IFNULL(dept_id, 'Unassigned') AS department,
COALESCE(phone, email, 'No Contact') AS contact
FROM employees;
```
