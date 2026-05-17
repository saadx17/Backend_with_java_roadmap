Data querying is t*he process of requesting, retrieving, and manipulating specific information* from a database. It acts as a "question" posed to a database management system (DBMS) to find and display data.
**Purpose:** To extract, analyze, and manage structured data stored in tables, such as calculating totals, filtering specific records, or combining data from multiple tables.

# `SELECT` - The Show

#### 1. `SELECT` + `FROM`

**Syntax:**
```
SELECT column1, column2, ...
FROM   table_name;
```

```
-- Select ALL columns (* = means ALL)
SELECT * FROM employees;
SELECT * FROM products;
SELECT * FROM departments;
```

```
-- Select SPECIFIC columns
SELECT first_name, last_name, salary FROM employees;
SELECT product_name, price, stock    FROM products;
SELECT dept_name, location, budget   FROM departments;
```

```
-- Select with EXPRESSIONS
SELECT
    first_name,
    last_name,
    salary,
    salary * 12 AS annual_salary,
    salary * 12 * 0.10 AS yearly_bonus,
    salary / 160 AS hourly_rate
FROM employees;
```

```
-- Select CONSTANTS and LITERALS
SELECT
    'Hello World' AS greeting,
    42 AS the_answer,
    3.14 AS pi,
    CURDATE() AS today,
    NOW() AS current_time;
```

```
-- Select WITHOUT a table (dual table concept)
SELECT 100 + 200 AS addition;
SELECT CONCAT('MySQL', ' ', 'Rocks') AS message;
SELECT VERSION() AS mysql_version;
SELECT DATABASE() AS current_db;
SELECT USER() AS current_user;
```

#### 2. `AS` (Alias)

###### Column Aliases
```
-- Using AS keyword (recommended)
SELECT
    first_name  AS 'First Name',
    last_name   AS 'Last Name',
    salary      AS 'Base Salary'
FROM employees;
```

```
-- Without AS keyword (also works)
SELECT
    first_name  'First Name',
    last_name   'Last Name',
    salary      'Base Salary'
FROM employees;
```

```
-- No quotes for single-word aliases
SELECT
    first_name  AS fname,
    last_name   AS lname,
    salary      AS pay
FROM employees;
```

```
-- Aliases with expressions
SELECT
    CONCAT(first_name, ' ', last_name)      AS full_name,
    UPPER(email)                            AS email_upper,
    salary * 12                             AS annual_salary,
    DATEDIFF(CURDATE(), hire_date) / 365    AS years_worked,
    ROUND(salary / 160, 2)                  AS hourly_rate
FROM employees;
```

###### Table Aliases (very useful with JOINs)

```
-- Table alias
SELECT e.first_name, e.last_name, e.salary
FROM employees AS e;
```

```
-- Without AS
SELECT e.first_name, e.salary
FROM employees e;
```

```
-- Table alias avoids ambiguity
SELECT
    e.first_name,
    e.dept_id,
    d.dept_name
FROM employees  AS e,
     departments AS d
WHERE e.dept_id = d.dept_id;
```

```
-- ⚠️ Cannot use column alias in WHERE clause
-- This FAILS:
SELECT salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 900000;   -- ❌ Error!
```

```
-- This WORKS (use expression again):
SELECT salary * 12 AS annual_salary
FROM employees
WHERE salary * 12 > 900000;     -- ✅
```

```
-- Or use subquery

SELECT * FROM (
  SELECT salary * 12 AS annual_salary,first_name
  FROM employees
) AS sub
WHERE annual_salary > 900000; -- ✅
```

#### 3. `WHERE` - Filtering

###### Syntax
```
-- Comparison Operators
SELECT * FROM employees WHERE salary  = 95000;    -- equal
SELECT * FROM employees WHERE salary != 95000;    -- not equal
SELECT * FROM employees WHERE salary <> 95000;    -- not equal (same)
SELECT * FROM employees WHERE salary  > 80000;    -- greater than
SELECT * FROM employees WHERE salary  < 60000;    -- less than
SELECT * FROM employees WHERE salary >= 90000;    -- greater or equal
SELECT * FROM employees WHERE salary <= 50000;    -- less or equal
```

```
-- String comparison
SELECT * FROM employees WHERE first_name = 'Alice';
SELECT * FROM employees WHERE status = 'active';
SELECT * FROM employees WHERE job_title = 'Developer';
```

#### 4. `AND`, `OR`, `NOT` - Filtering

##### `AND` - Both conditions must be TRUE
```
-- Active IT employees
SELECT first_name, last_name, salary, status
FROM employees
WHERE dept_id = 1
AND   status  = 'active';
```

```
-- High salary active employees
SELECT first_name, salary, dept_id, status
FROM employees
WHERE salary > 80000
AND   status = 'active'
AND   dept_id IN (1, 6);
```

##### `OR` - At least ONE condition must be TRUE
```
-- Employees from IT or HR
SELECT first_name, last_name, dept_id
FROM employees
WHERE dept_id = 1
OR    dept_id = 2;
```

```
-- Low or high earners
SELECT first_name, salary
FROM employees
WHERE salary < 50000
OR    salary > 95000;
```

```
-- Multiple OR conditions
SELECT first_name, status
FROM employees
WHERE status = 'inactive'
OR    status = 'terminated';
```

##### `NOT` - Negates a condition
```
SELECT * FROM employees WHERE NOT status  = 'active';
SELECT * FROM employees WHERE NOT dept_id = 1;
SELECT * FROM employees WHERE NOT salary > 70000;
```

#### 5. `IN` and `NOT IN` - Filtering

##### `IN` - Matches any value in a list
```
-- Replaces multiple OR conditions
SELECT first_name, dept_id FROM employees
WHERE dept_id IN (1, 2, 6);      -- IT, HR, R&D
```

```
-- Same as:
SELECT first_name, dept_id FROM employees
WHERE dept_id = 1 OR dept_id = 2 OR dept_id = 6;
```

```
-- IN with strings
SELECT first_name, status FROM employees
WHERE status IN ('inactive', 'terminated');
```

```
-- IN with dates
SELECT first_name, hire_date FROM employees
WHERE hire_date IN ('2019-03-15', '2020-07-22', '2021-01-05');
```

```
-- IN with subquery (powerful!)
SELECT first_name, dept_id FROM employees
WHERE dept_id IN (
    SELECT dept_id FROM departments
    WHERE location IN ('New York', 'Boston')
);
```

##### `NOT IN` - Excludes values in a list
```
SELECT first_name, dept_id FROM employees
WHERE dept_id NOT IN (1, 2, 3);

SELECT first_name, status FROM employees
WHERE status NOT IN ('inactive', 'terminated');
```

```
-- NOT IN with NULL WARNING

If the list contains NULL, NOT IN returns NO rows!

SELECT * FROM employees WHERE dept_id NOT IN (1, NULL);
→ Returns 0 rows! (Because NULL comparison is unknown)
```

```
-- Use IS NOT NULL to be safe

SELECT * FROM employees
WHERE dept_id NOT IN (1, 2)
AND dept_id IS NOT NULL;
```

#### 6. `BETWEEN` - Filtering

###### value `BETWEEN` low AND high
```
-- Numeric range
SELECT first_name, salary FROM employees
WHERE salary BETWEEN 60000 AND 85000;   -- includes 60000 and 85000
```

```
-- Same as:
SELECT first_name, salary FROM employees
WHERE salary >= 60000 AND salary <= 85000;
```

```
-- Date range
SELECT first_name, hire_date FROM employees
WHERE hire_date BETWEEN '2020-01-01' AND '2021-12-31';
```

```
-- Price range
SELECT product_name, price FROM products
WHERE price BETWEEN 50.00 AND 200.00;
```

```
-- NOT BETWEEN
SELECT first_name, salary FROM employees
WHERE salary NOT BETWEEN 60000 AND 90000;
```

```
-- BETWEEN with DATETIME
SELECT first_name, hire_date FROM employees
WHERE hire_date BETWEEN '2019-01-01 00:00:00'
                    AND '2020-12-31 23:59:59';
```

###### `BETWEEN` with strings (alphabetical range)
```
SELECT first_name FROM employees
WHERE first_name BETWEEN 'A' AND 'F';   -- Names A, B, C, D, E, F...
```

###### `BETWEEN` Tips:
- Both endpoints are INCLUSIVE.
- Works with numbers, dates, strings.
- low value must come before high value.
- BETWEEN 'A' AND 'M' includes strings A through M. 

#### 7. `IS NULL` and `IS NOT NULL` - Filtering

###### `IS NULL`

```
SELECT first_name, phone FROM employees
WHERE phone IS NULL;

SELECT first_name, manager_id FROM employees
WHERE manager_id IS NULL;       -- Employees with no manager = Top level

SELECT first_name, email FROM employees
WHERE email IS NULL;
```

###### `IS NOT NULL`

```
SELECT first_name, phone FROM employees
WHERE phone IS NOT NULL;

SELECT first_name, manager_id FROM employees
WHERE manager_id IS NOT NULL;   -- Employees who HAVE a manager
```

```
-- Combining with other conditions
SELECT first_name, phone, dept_id
FROM employees
WHERE phone IS NOT NULL
AND   dept_id = 1;
```

```
-- NULL in calculations
SELECT
   first_name, salary,
   NULL + 100 AS null_math,     -- Results in NULL
   IFNULL(phone, 'N/A') AS safe_phone
FROM employees;
```

#### 8. `ORDER BY` - Sorting

###### `ASC` & `DESC`
```
-- Single column - Ascending (default)
SELECT first_name, salary FROM employees
ORDER BY salary;              -- lowest first

SELECT first_name, salary FROM employees
ORDER BY salary ASC;          -- same as above, explicit
```

```
-- Single column - Descending
SELECT first_name, salary FROM employees
ORDER BY salary DESC;         -- highest first
```

```
-- Sort by string (alphabetical)
SELECT first_name, last_name FROM employees
ORDER BY first_name ASC;      -- A to Z

SELECT first_name, last_name FROM employees
ORDER BY first_name DESC;     -- Z to A
```

```
-- Sort by date
SELECT first_name, hire_date FROM employees
ORDER BY hire_date ASC;       -- oldest hire first

SELECT first_name, hire_date FROM employees
ORDER BY hire_date DESC;      -- newest hire first
```

###### Multiple Column Sorting
```
-- Sort by dept_id ASC, then salary DESC within each dept
SELECT dept_id, first_name, salary
FROM employees
ORDER BY dept_id ASC, salary DESC;
```

```
-- Sort by status then name
SELECT first_name, status, salary
FROM employees
ORDER BY status ASC, first_name ASC;
```

```
-- 3 columns sort
SELECT dept_id, status, first_name, salary
FROM employees
ORDER BY dept_id ASC,
         status ASC,
         salary DESC;
```

###### `ORDER BY` with column position (number)
```
SELECT first_name, last_name, salary
FROM employees
ORDER BY 3 DESC;            -- Sort by 3rd column (salary)

SELECT first_name, last_name, salary
FROM employees
ORDER BY 1, 2;              -- Sort by first_name, then last_name
```

###### `ORDER BY` with expression or function
```
-- Sort by calculated value
SELECT first_name, salary, salary * 12 AS annual
FROM employees
ORDER BY salary * 12 DESC;
```

```
-- Sort by string length
SELECT first_name FROM employees
ORDER BY LENGTH(first_name) ASC;
```

```
-- Sort by year of hire, then name
SELECT first_name, hire_date
FROM employees
ORDER BY YEAR(hire_date) DESC, first_name ASC;
```

```
-- Sort by month (useful for birthday/anniversary reports)
SELECT first_name, hire_date
FROM employees
ORDER BY MONTH(hire_date), DAY(hire_date);
```

###### `ORDER BY` with CASE (Custom Sort Order)
```
-- Custom priority order for status
SELECT first_name, status
FROM employees
ORDER BY
    CASE status
        WHEN 'active'     THEN 1    -- show active first
        WHEN 'inactive'   THEN 2    -- then inactive
        WHEN 'terminated' THEN 3    -- terminated last
    END,
    first_name ASC;
```

###### `NULL` handling in `ORDER BY`
```
-- NULLs come FIRST in ASC order
SELECT first_name, phone FROM employees
ORDER BY phone ASC;      -- NULL rows appear first
```

```
-- NULLs come LAST in DESC order
SELECT first_name, phone FROM employees
ORDER BY phone DESC;     -- NULL rows appear last
```

```
-- Force NULLs to LAST in ASC (using CASE trick)
SELECT first_name, phone FROM employees
ORDER BY
  CASE WHEN phone IS NULL THEN 1 ELSE 0 END,
  phone ASC;
```

#### 9. `LIMIT` and `OFFSET` - Limiting

###### `LIMIT` - Restrict number of rows returned
```
-- Get first 5 rows
SELECT * FROM employees LIMIT 5;
```

```
-- Get first 10 products
SELECT product_name, price FROM products LIMIT 10;
```

```
-- Top N pattern
SELECT first_name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;                        -- Top 3 highest paid
```

```
-- Bottom N pattern
SELECT first_name, salary
FROM employees
ORDER BY salary ASC
LIMIT 3;                        -- 3 lowest paid
```

###### `OFFSET` - Skip N rows before returning results
```
-- Skip first 5 rows, return next 5
SELECT first_name, salary FROM employees
ORDER BY emp_id
LIMIT 5 OFFSET 5;
```

```
-- Two syntax options (both identical):
LIMIT 5 OFFSET 10    -- skip 10, return 5
LIMIT 10, 5          -- same: LIMIT offset, count
```

###### Pagination Pattern (Most Important Use Case)

**Formula:**
```
Formula: OFFSET = (page_number - 1) × rows_per_page
```

**Result:**
```
Page 1: LIMIT 5 OFFSET 0   → rows 1-5
Page 2: LIMIT 5 OFFSET 5   → rows 6-10
Page 3: LIMIT 5 OFFSET 10  → rows 11-15
Page 4: LIMIT 5 OFFSET 15  → rows 16-20
```

**Syntax:**
```
-- Page 1 (rows 1-5)
SELECT emp_id, first_name, salary FROM employees
ORDER BY emp_id
LIMIT 5 OFFSET 0;

-- Page 2 (rows 6-10)
SELECT emp_id, first_name, salary FROM employees
ORDER BY emp_id
LIMIT 5 OFFSET 5;

-- Page 3 (rows 11-15)
SELECT emp_id, first_name, salary FROM employees
ORDER BY emp_id
LIMIT 5 OFFSET 10;

-- Page 4 (rows 16-20)
SELECT emp_id, first_name, salary FROM employees
ORDER BY emp_id
LIMIT 5 OFFSET 15;
```

```
-- Dynamic pagination using variables
SET @page     = 2;     -- which page
SET @per_page = 5;     -- rows per page

SELECT emp_id, first_name, salary
FROM employees
ORDER BY emp_id
LIMIT 5 OFFSET 5;      -- (@page - 1) * @per_page = 5
```

###### `LIMIT` with `WHERE` and `ORDER BY`

```
-- Second highest salary
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;       -- Skip the highest, return next
```

```
-- Nth highest salary pattern (N=3, third highest)
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;       -- OFFSET = N - 1
```

```
-- Latest 5 hired employees
SELECT first_name, hire_date
FROM employees
ORDER BY hire_date DESC
LIMIT 5;
```

```
-- Oldest 3 employees by hire date
SELECT first_name, hire_date
FROM employees
ORDER BY hire_date ASC
LIMIT 3;
```

```
-- Cheapest 5 products in Electronics
SELECT product_name, price
FROM products
WHERE category = 'Electronics'
ORDER BY price ASC
LIMIT 5;
```

#### 10. `SELECT DISTINCT` - Distinct

###### `DISTINCT` - Returns unique/non-duplicate rows
```
-- Single column distinct
SELECT DISTINCT dept_id   FROM employees;
SELECT DISTINCT status    FROM employees;
SELECT DISTINCT category  FROM products;
SELECT DISTINCT job_title FROM employees;
```

###### `DISTINCT` on multiple column - Unique COMBINATION of all selected columns
```
SELECT DISTINCT dept_id, status
FROM employees
ORDER BY dept_id, status;
-- Returns unique (dept_id, status) pairs

SELECT DISTINCT category, rating
FROM products
ORDER BY category, rating;
```

###### `DISTINCT` vs `GROUP BY`
```
-- Both give same result for basic deduplication
SELECT DISTINCT dept_id FROM employees ORDER BY dept_id;

SELECT dept_id FROM employees
GROUP BY dept_id
ORDER BY dept_id;

-- GROUP BY is more powerful (can use aggregates)
SELECT dept_id, COUNT(*) AS count
FROM employees
GROUP BY dept_id;
```

###### `DISTINCT` with Aggregate Functions
```
-- Count unique departments that have employees
SELECT COUNT(DISTINCT dept_id)   AS unique_depts    FROM employees;
SELECT COUNT(DISTINCT status)    AS unique_statuses FROM employees;
SELECT COUNT(DISTINCT job_title) AS unique_titles   FROM employees;

-- Sum of distinct salary values
SELECT SUM(DISTINCT salary) AS sum_unique_salaries FROM employees;

-- Average of distinct prices
SELECT AVG(DISTINCT price) AS avg_unique_price FROM products;
```

###### `DISTINCT` with `WHERE`
```
SELECT DISTINCT dept_id
FROM employees
WHERE status = 'active';           -- Unique depts with active employees

SELECT DISTINCT category
FROM products
WHERE price > 100;                 -- Categories with products > $100
```

###### `DISTINCT` with `ORDER BY`
```
SELECT DISTINCT job_title
FROM employees
WHERE status = 'active'
ORDER BY job_title ASC;
```

###### `NULL` with `DISTINCT`
```
SELECT DISTINCT phone FROM employees;  -- Shows one NULL + unique phones
```

###### Notes
- NULL is treated as a single unique value. SELECT DISTINCT phone → one NULL in result.
- DISTINCT applies to ALL selected columns combined, not just the first column.
- Can slow performance on large tables.
- Consider GROUP BY for better performance.

#### 11. `LIKE` (%, _ ) - Pattern Matching

###### Wildcards
```
-- LIKE Wildcards:
-- % = Any number of characters (including zero)
-- _ = Exactly ONE character
```

**Wildcard Examples:**
```
-- Starts with 'A'
SELECT first_name FROM employees
WHERE first_name LIKE 'A%';            -- Alice

-- Ends with 'n'
SELECT first_name FROM employees
WHERE first_name LIKE '%n';            -- Nathan

-- Contains 'ar'
SELECT first_name FROM employees
WHERE first_name LIKE '%ar%';          -- Karen, Rachel

-- Starts with 'J' and ends with 'k'
SELECT first_name FROM employees
WHERE first_name LIKE 'J%k';          -- Jack

-- Email from specific domain
SELECT first_name, email FROM employees
WHERE email LIKE '%@company.com';

-- Job titles containing 'Senior'
SELECT first_name, job_title FROM employees
WHERE job_title LIKE '%Senior%';
```

**Wildcard Examples (exact character count):**

```
-- Exactly 3 characters
SELECT first_name FROM employees
WHERE first_name LIKE '___';           -- Mia, Leo (3 chars)

-- Exactly 4 characters
SELECT first_name FROM employees
WHERE first_name LIKE '____';          -- Paul, Tina, Emma (4 chars)

-- Starts with any char + 'li'
SELECT first_name FROM employees
WHERE first_name LIKE '_li%';          -- Alice (A+li+ce)

-- Second character is 'a'
SELECT first_name FROM employees
WHERE first_name LIKE '_a%';           -- Rachel, Nathan, Karen, Jack

-- Exactly 5 chars starting with 'G'
SELECT first_name FROM employees
WHERE first_name LIKE 'G____';         -- Grace (G+race = 5)
```

###### Combining % and _
```
-- 5+ character names starting with 'A'
SELECT first_name FROM employees
WHERE first_name LIKE 'A_____%';       -- At least 5 chars after A

-- Phone numbers starting with '987650'
SELECT first_name, phone FROM employees
WHERE phone LIKE '987650%';

-- Product names with exactly 3 words (has 2 spaces)
SELECT product_name FROM products
WHERE product_name LIKE '% % %';
```

###### `NOT LIKE`
```
-- Names NOT starting with A, B, C
SELECT first_name FROM employees
WHERE first_name NOT LIKE 'A%'
AND   first_name NOT LIKE 'B%'
AND   first_name NOT LIKE 'C%';

-- Emails not from company domain
SELECT first_name, email FROM employees
WHERE email NOT LIKE '%@company.com'
AND   email IS NOT NULL;
```

###### `LIKE` is Case-INSENSITIVE in MySQL (default)
```
SELECT * FROM employees WHERE first_name LIKE 'alice';   -- Finds Alice ✅
SELECT * FROM employees WHERE first_name LIKE 'ALICE';   -- Finds Alice ✅
SELECT * FROM employees WHERE first_name LIKE 'aLiCe';   -- Finds Alice ✅

-- Make Case-SENSITIVE using BINARY
SELECT * FROM employees
WHERE first_name LIKE BINARY 'alice';   -- Finds alice only (not Alice) ❌
```

###### `Escape` Special Characters in `LIKE`
```
-- Search for literal % or _ using ESCAPE
SELECT * FROM products
WHERE product_name LIKE '%50\%%';            -- Contains "50%"

-- OR
SELECT * FROM products
WHERE product_name LIKE '%50!%%' ESCAPE '!'; -- Custom escape char
```

#### 12. `REGEXP` - Pattern Matching

###### REGEXP Metacharacters
```
┌──────────┬───────────────────────────────────────────┐
│ Pattern  │ Meaning                                   │
├──────────┼───────────────────────────────────────────┤
│ .        │ Any single character                      │
│ ^        │ Start of string                           │
│ $        │ End of string                             │
│ *        │ 0 or more of previous                     │
│ +        │ 1 or more of previous                     │
│ ?        │ 0 or 1 of previous                        │
│ [abc]    │ Any one of a, b, c                        │
│ [^abc]   │ Any character NOT a, b, c                 │
│ [a-z]    │ Any character in range a to z             │
│ {n}      │ Exactly n repetitions                     │
│ {n,m}    │ Between n and m repetitions               │
│ a|b      │ a or b                                    │
│ (abc)    │ Group                                     │
└──────────┴───────────────────────────────────────────┘
```

###### ^ (Start of String)
```
-- Names starting with 'A', 'B', or 'C'
SELECT first_name FROM employees
WHERE first_name REGEXP '^[ABC]';

-- Names starting with a vowel
SELECT first_name FROM employees
WHERE first_name REGEXP '^[aeiouAEIOU]';

-- Names starting with 'Al'
SELECT first_name FROM employees
WHERE first_name REGEXP '^Al';
```

###### $ (End of String)
```
-- Names ending with 'n'
SELECT first_name FROM employees
WHERE first_name REGEXP 'n$';

-- Names ending with 'a', 'e', 'i', 'o', 'u'
SELECT first_name FROM employees
WHERE first_name REGEXP '[aeiou]$';

-- Emails ending with '.com'
SELECT email FROM employees
WHERE email REGEXP '\\.com$';
```

###### | (OR)
```
-- Names starting with A, E, or I
SELECT first_name FROM employees
WHERE first_name REGEXP '^A|^E|^I';

-- Or using character class
SELECT first_name FROM employees
WHERE first_name REGEXP '^[AEI]';

-- Job titles containing 'Developer' or 'Analyst'
SELECT first_name, job_title FROM employees
WHERE job_title REGEXP 'Developer|Analyst';

-- Multiple word search
SELECT product_name FROM products
WHERE product_name REGEXP 'Pro|Deluxe|HD';
```

###### Character classes
```
-- Names with only alphabets (no numbers/special chars)
SELECT first_name FROM employees
WHERE first_name REGEXP '^[A-Za-z]+$';

-- Names with exactly 4 characters
SELECT first_name FROM employees
WHERE first_name REGEXP '^[A-Za-z]{4}$';

-- Names with 4 to 6 characters
SELECT first_name FROM employees
WHERE first_name REGEXP '^[A-Za-z]{4,6}$';

-- Phone starting with 9876
SELECT first_name, phone FROM employees
WHERE phone REGEXP '^9876';
```

###### (Any character)
```
-- Any 4-character string starting with G
SELECT first_name FROM employees
WHERE first_name REGEXP '^G...';      -- G + any 3 chars
```

###### NOT REGEXP
```
-- Names NOT starting with vowels
SELECT first_name FROM employees
WHERE first_name NOT REGEXP '^[aeiouAEIOU]';

-- Emails not from 'company' domain
SELECT first_name, email FROM employees
WHERE email NOT REGEXP '@company'
AND   email IS NOT NULL;
```

###### `LIKE` vs. `REGEXP`
```
┌──────────────────┬────────────────────┬────────────────────┐
│ Goal             │ LIKE               │ REGEXP             │
├──────────────────┼────────────────────┼────────────────────┤
│ Starts with 'A'  │ LIKE 'A%'          │ REGEXP '^A'        │
│ Ends with 'n'    │ LIKE '%n'          │ REGEXP 'n$'        │
│ Contains 'ar'    │ LIKE '%ar%'        │ REGEXP 'ar'        │
│ A or B or C      │ IN or multiple OR  │ REGEXP '^[ABC]'    │
│ Exactly 4 chars  │ LIKE '____'        │ REGEXP '^.{4}$'    │
│ Digit in name    │ Complex            │ REGEXP '[0-9]'     │
│ Valid email      │ Not easy           │ REGEXP '@.+\..+'   │
└──────────────────┴────────────────────┴────────────────────┘
```

REGEXP is more powerful but:
- Slower than LIKE
- Cannot use indexes efficiently
- Use LIKE when possible, REGEXP for complex patterns

###### `REGEXP_LIKE()` function (MySQL 8.0+)
```
SELECT first_name FROM employees
WHERE REGEXP_LIKE(first_name, '^[A-M]');  -- Names A through M

-- Case sensitive (c flag)
SELECT first_name FROM employees
WHERE REGEXP_LIKE(first_name, '^alice', 'c');  -- Case sensitive
```

###### `REGEXP_REPLACE()` - Replace using pattern
```
-- Mask phone numbers (keep last 4 digits)
SELECT
    first_name,
    phone,
    REGEXP_REPLACE(phone, '^[0-9]{6}', '******') AS masked_phone
FROM employees
WHERE phone IS NOT NULL;

-- Remove all spaces
SELECT REGEXP_REPLACE('Hello   World', ' +', ' ') AS cleaned;
```

###### `REGEXP_SUBSTR()` - Extract matching part
```
-- Extract domain from email
SELECT email, REGEXP_SUBSTR(email, '@[^@]+$') AS domain
FROM employees
WHERE email IS NOT NULL;
```

#### 13. `CASE` WHEN...THEN...ELSE...END - Conditional

###### `CASE` Syntax - Two Forms
```
-- Form 1: Simple CASE (equality checks)
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ELSE default_result
END

-- Form 2: Searched CASE (complex conditions)
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

###### Form 1: Simple `CASE`
```
-- dept_id to department name
SELECT
    first_name,
    dept_id,
    CASE dept_id
        WHEN 1 THEN 'IT'
        WHEN 2 THEN 'Human Resources'
        WHEN 3 THEN 'Finance'
        WHEN 4 THEN 'Marketing'
        WHEN 5 THEN 'Operations'
        WHEN 6 THEN 'R&D'
        ELSE        'Unknown'
    END AS department_name
FROM employees;

-- Status display
SELECT
    first_name,
    CASE status
        WHEN 'active'     THEN '✅ Active'
        WHEN 'inactive'   THEN '⏸️  Inactive'
        WHEN 'terminated' THEN '❌ Terminated'
        ELSE                   '❓ Unknown'
    END AS employee_status
FROM employees;
```

###### Form 2: Searched `CASE` (more flexible)
```
-- Salary bands
SELECT
    first_name,
    salary,
    CASE
        WHEN salary >= 90000 THEN 'Band A - Senior'
        WHEN salary >= 70000 THEN 'Band B - Mid'
        WHEN salary >= 50000 THEN 'Band C - Junior'
        ELSE                      'Band D - Entry'
    END AS salary_band
FROM employees
ORDER BY salary DESC;
```

```
-- Performance rating based on multiple conditions
SELECT
    first_name,
    salary,
    dept_id,
    CASE
        WHEN salary > 90000 AND dept_id IN (1,6) THEN 'Star Performer'
        WHEN salary > 80000                       THEN 'High Performer'
        WHEN salary > 60000                       THEN 'Good Performer'
        WHEN salary > 40000                       THEN 'Average Performer'
        ELSE                                           'Needs Improvement'
    END AS performance
FROM employees;
```

```
-- Product pricing tier
SELECT
    product_name,
    price,
    CASE
        WHEN price >= 500  THEN 'Premium'
        WHEN price >= 100  THEN 'Standard'
        WHEN price >= 30   THEN 'Budget'
        ELSE                    'Economy'
    END AS price_tier
FROM products;
```

###### `CASE` in `ORDER BY` (Custom Sort)
```
SELECT first_name, status
FROM employees
ORDER BY
    CASE status
        WHEN 'active'     THEN 1
        WHEN 'inactive'   THEN 2
        WHEN 'terminated' THEN 3
    END,
    first_name;
```

###### `CASE` in `WHERE`
```
-- Different salary threshold by department
SELECT first_name, dept_id, salary
FROM employees
WHERE salary > CASE dept_id
                    WHEN 1 THEN 80000   -- IT: high threshold
                    WHEN 6 THEN 85000   -- R&D: highest
                    ELSE       60000    -- Others: lower
               END;
```

###### `CASE` in `GROUP BY` / Aggregates
```
-- Count employees in each salary band
SELECT
    CASE
        WHEN salary >= 90000 THEN 'Senior (90k+)'
        WHEN salary >= 70000 THEN 'Mid (70-90k)'
        WHEN salary >= 50000 THEN 'Junior (50-70k)'
        ELSE                      'Entry (<50k)'
    END AS salary_band,
    COUNT(*) AS employee_count,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY
    CASE
        WHEN salary >= 90000 THEN 'Senior (90k+)'
        WHEN salary >= 70000 THEN 'Mid (70-90k)'
        WHEN salary >= 50000 THEN 'Junior (50-70k)'
        ELSE                      'Entry (<50k)'
    END
ORDER BY avg_salary DESC;
```

```
-- Pivot-like report: Count by status per department
SELECT dept_id,
       SUM(CASE WHEN status = 'active' THEN 1 ELSE 0 END) AS active_count,
       SUM(CASE WHEN status = 'inactive' THEN 1 ELSE 0 END) AS inactive_count,
       SUM(CASE WHEN status = 'terminated' THEN 1 ELSE 0 END) AS terminated_count,
       COUNT(*) AS total
FROM employees
GROUP BY dept_id ORDER BY dept_id;
```

#### 14. `IF()` - Conditional

###### `IF` (condition, value_if_true, value_if_false)
```
-- Basic IF
SELECT
    first_name,
    salary,
    IF(salary > 75000, 'High', 'Low') AS salary_category
FROM employees;

-- Active/Inactive label
SELECT
    first_name,
    IF(status = 'active', 'Yes', 'No') AS is_active
FROM employees;

-- Calculate bonus with IF
SELECT
    first_name,
    salary,
    IF(dept_id = 1, salary * 0.20, salary * 0.10) AS bonus,
    salary + IF(dept_id = 1, salary * 0.20, salary * 0.10) AS total_comp
FROM employees;
```

###### Nested `IF` (use CASE for clarity instead)
```
SELECT
    first_name,
    salary,
    IF(salary > 90000, 'Senior',
        IF(salary > 70000, 'Mid',
            IF(salary > 50000, 'Junior',
                'Entry'
            )
        )
    ) AS level
FROM employees;
```

###### `IF` with `IS NULL`
```
SELECT
    first_name,
    phone,
    IF(phone IS NULL, 'No Phone', phone) AS contact_phone
FROM employees;

SELECT
    first_name,
    manager_id,
    IF(manager_id IS NULL, 'Top Level', 'Has Manager') AS position
FROM employees;
```

###### `IF` in aggregate (conditional counting)
```
SELECT dept_id,
       COUNT(*) AS total,
       SUM(IF(status = 'active', 1, 0)) AS active,
       SUM(IF(salary > 70000, 1, 0)) AS high_earners,
       AVG(IF(status = 'active', salary, NULL)) AS avg_active_salary
FROM employees
GROUP BY dept_id;
```

#### 15. `IFNULL()`, `NULLIF()`, `COALESCE()` - Conditional

###### `IFNULL`(expression, replacement) - Returns replacement if expression `IS NULL`
```
-- Replace NULL phone with default text
SELECT
    first_name,
    IFNULL(phone, 'Not Provided')     AS phone,
    IFNULL(email, 'No Email')         AS email,
    IFNULL(manager_id, 0)             AS manager_id
FROM employees;
```

```
-- Replace NULL in calculations (NULL math = NULL)
SELECT
    first_name,
    dept_id,
    IFNULL(dept_id, 0) + 100          AS dept_code
FROM employees;
```

```
-- Use default salary if NULL
SELECT
    first_name,
    IFNULL(salary, 30000)             AS salary
FROM employees;
```

###### `NULLIF`(expr1, expr2)
→ Returns NULL if expr1 = expr2
→ Returns expr1 if they are different

```
NULLIF(x, y) is equivalent to: IF(x = y, NULL, x)
```

```
-- Avoid division by zero!
SELECT
    dept_id,
    salary,
    100000 / NULLIF(salary, 0)        AS ratio   -- safe division
FROM employees;
```

```
-- Without NULLIF (dangerous!):
SELECT 100000 / 0;                               -- Error: Division by zero!
```

```
-- Mark specific values as NULL
SELECT
    first_name,
    NULLIF(status, 'terminated')     AS clean_status
FROM employees;
-- Returns NULL for terminated employees, status value for others
```

```
-- Compare two values
SELECT
    first_name,
    salary,
    NULLIF(salary, 95000)            AS flagged_salary
FROM employees;
-- Returns NULL if salary = 95000, else returns salary
```

```
-- Practical: Calculate percentage
SELECT
    dept_id,
    COUNT(*) AS emp_count,
    ROUND(COUNT(*) * 100.0 / NULLIF(
        (SELECT COUNT(*) FROM employees), 0
    ), 2) AS percentage
FROM employees
GROUP BY dept_id;
```

###### `COALESCE`(val1, val2, val3, ...)
→ Returns FIRST non-NULL value in the list
→ Accepts multiple arguments (unlike IFNULL)

```
-- Return first available contact info
SELECT
    first_name,
    phone,
    email,
    COALESCE(phone, email, 'No Contact Info') AS best_contact
FROM employees;
```

Explanation:
- COALESCE checks left to right:
- If phone is NOT NULL → return phone
- Else if email is NOT NULL → return email
- Else return 'No Contact Info'

```
-- Default values for multiple nullable columns
SELECT
    first_name,
    COALESCE(manager_id, 0)               AS mgr_id,
    COALESCE(phone, 'N/A')                AS phone,
    COALESCE(email, 'N/A')                AS email,
    COALESCE(salary, 35000)               AS salary
FROM employees;
```

```
-- Address fallback example
CREATE TABLE contacts (
    id          INT PRIMARY KEY AUTO_INCREMENT,
    name        VARCHAR(100),
    mobile      VARCHAR(15),
    home_phone  VARCHAR(15),
    work_phone  VARCHAR(15),
    email       VARCHAR(100)
);

INSERT INTO contacts VALUES
(1, 'Alice', '9999999999', NULL,         NULL,         'a@x.com'),
(2, 'Bob',   NULL,         '8888888888', NULL,         'b@x.com'),
(3, 'Carol', NULL,         NULL,         '7777777777', 'c@x.com'),
(4, 'David', NULL,         NULL,         NULL,         'd@x.com'),
(5, 'Eve',   NULL,         NULL,         NULL,         NULL);

SELECT
    name,
    COALESCE(mobile, home_phone, work_phone, email, 'No contact') AS reach_at
FROM contacts;
```

```
Result:
Alice → 9999999999  (mobile)
Bob   → 8888888888  (home_phone)
Carol → 7777777777  (work_phone)
David → d@x.com     (email)
Eve   → No contact  (all NULL)
```

###### Comparison: `IFNULL` vs `COALESCE` vs `NULLIF`

```
┌────────────────┬──────────────────────────────────────────────┐
│ Function       │ Usage & Purpose                              │
├────────────────┼──────────────────────────────────────────────┤
│ IFNULL(a, b)   │ Return b if a is NULL (2 args only)          │
│ COALESCE(a...) │ Return first non-NULL (multiple args)        │
│ NULLIF(a, b)   │ Return NULL if a = b, else return a          │
│ IF(c, a, b)    │ Ternary: return a if c is true, else b       │
└────────────────┴──────────────────────────────────────────────┘
```

```
-- IFNULL vs COALESCE:
IFNULL(a, b)          ≡  COALESCE(a, b)   -- identical for 2 args
COALESCE(a, b, c, d)  -- No IFNULL equivalent for 4 args
```