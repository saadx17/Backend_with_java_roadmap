In MySQL, **TCL** stands for **Transaction Control Language**. It is a subset of SQL used to manage and control transactions, a sequence of operations performed as a single logical unit of work to ensure data integrity and consistency.
TCL manages **transactions** to ensure data consistency (ACID)

### The `COMMIT`

###### `AUTO COMMIT` (MySQL Default = ON)
```
SHOW VARIABLES LIKE 'autocommit';   -- Check status
SET autocommit = 0;                  -- Disable (manual commit needed)
SET autocommit = 1;                  -- Enable  (each statement auto commits)
```

###### Manual Transaction with `COMMIT`
```
START TRANSACTION;                  -- Begin transaction
-- or
BEGIN;                              -- Same as START TRANSACTION

    INSERT INTO departments (dept_name, location)
    VALUES ('Research & Development', 'Boston');

    INSERT INTO employees (first_name, last_name, email, salary, hire_date, dept_id)
    VALUES ('Zara', 'Ahmed', 'zara@company.com', 95000, CURDATE(), LAST_INSERT_ID());

    UPDATE employees
    SET salary = salary * 1.05
    WHERE dept_id = 1;

COMMIT;                             -- Save all changes permanently
```

###### Bank Transfer Example (Classic ACID demo)
```
CREATE TABLE bank_accounts (
    account_id  INT           PRIMARY KEY AUTO_INCREMENT,
    holder_name VARCHAR(100),
    balance     DECIMAL(10,2) DEFAULT 0.00
);

INSERT INTO bank_accounts (holder_name, balance) VALUES
    ('Alice', 5000.00),
    ('Bob',   3000.00);

-- Transfer ₹1000 from Alice to Bob
START TRANSACTION;

    -- Step 1: Deduct from Alice
    UPDATE bank_accounts
    SET balance = balance - 1000
    WHERE account_id = 1 AND balance >= 1000;

    -- Step 2: Check if deduction succeeded
    -- (In application, check ROW_COUNT() here)

    -- Step 3: Add to Bob
    UPDATE bank_accounts
    SET balance = balance + 1000
    WHERE account_id = 2;

COMMIT;   -- Both operations succeed together
```

###### Verify Syntax
```
-- Verify
SELECT * FROM bank_accounts;
```

### The `ROLLBACK`

###### Basic Syntax
```
START TRANSACTION;

    DELETE FROM employees WHERE dept_id = 1;  -- Oops! Wrong delete
    SELECT COUNT(*) FROM employees;            -- Check damage: 0 rows!

ROLLBACK;                                      -- Undo everything!

SELECT COUNT(*) FROM employees;                -- Back to normal ✅
```

###### `ROLLBACK` on Error (Practical Pattern)
```
START TRANSACTION;

    -- Attempt risky operations
    UPDATE bank_accounts SET balance = balance - 2000 WHERE account_id = 1;
    UPDATE bank_accounts SET balance = balance + 2000 WHERE account_id = 99;  -- Invalid ID!

    -- Check if both updates worked
    -- If NOT → ROLLBACK
ROLLBACK;  -- Revert because account_id=99 doesn't exist

-- Real-world: This logic goes in application code or stored procedure
```

### The `SAVEPOINT`

###### Basic Syntax
```
START TRANSACTION;

    INSERT INTO departments (dept_name, location)
    VALUES ('Legal', 'New York');
    SAVEPOINT after_dept_insert;         -- Checkpoint 1 💾

    INSERT INTO employees (first_name, last_name, email, salary, hire_date, dept_id)
    VALUES ('Sam', 'Lee', 'sam@company.com', 75000, CURDATE(), LAST_INSERT_ID());
    SAVEPOINT after_emp_insert;          -- Checkpoint 2 💾

    -- Oops! Wrong salary update
    UPDATE employees SET salary = 0 WHERE emp_id = LAST_INSERT_ID();
    SAVEPOINT after_update;              -- Checkpoint 3 💾

    -- Rollback only to after_emp_insert (undo the wrong update)
    ROLLBACK TO SAVEPOINT after_emp_insert;

    -- Now do correct update
    UPDATE employees SET salary = 85000 WHERE first_name = 'Sam';

COMMIT;   -- Save dept insert + emp insert + correct salary update
```

###### Multiple SAVEPOINTs - Full Example
```
START TRANSACTION;
    SAVEPOINT start_point;

    -- Batch 1: Insert departments
    INSERT INTO departments (dept_name, location) VALUES ('Batch Dept A', 'City A');
    SAVEPOINT batch1_done;

    -- Batch 2: Insert employees
    INSERT INTO employees (first_name, last_name, email, salary, hire_date)
    VALUES ('Test', 'User', 'test@test.com', 50000, CURDATE());
    SAVEPOINT batch2_done;

    -- Something goes wrong in Batch 3
    UPDATE employees SET salary = -9999 WHERE emp_id = LAST_INSERT_ID();

    -- Go back to before Batch 3, keep Batches 1 & 2
    ROLLBACK TO SAVEPOINT batch2_done;

    -- Release savepoint (optional cleanup)
    RELEASE SAVEPOINT batch1_done;

COMMIT;   -- Commit: Batch1 + Batch2 only
```

###### `RELEASE SAVEPOINT`
```
-- Removes the savepoint (cannot rollback to it anymore)
RELEASE SAVEPOINT after_dept_insert;
```

### The SET TRANSACTION

###### Isolation Levels
```
-- Set for next transaction only
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- MySQL default
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set for current session
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Set globally
SET GLOBAL  TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

###### Transaction Access Mode
```
SET TRANSACTION READ ONLY;      -- Cannot modify data
SET TRANSACTION READ WRITE;     -- Can read and write (default)
```

###### Isolation Level Effects
```
┌─────────────────────┬───────────┬───────────┬─────────┬─────────┐
│ Isolation Level     │ Dirty     │ Non-Rep   │ Phantom │ Speed   │
│                     │ Read      │ Read      │ Read    │         │
├─────────────────────┼───────────┼───────────┼─────────┼─────────┤
│ READ UNCOMMITTED    │    ✅     │    ✅      │   ✅    │ Fastest │
│ READ COMMITTED      │    ❌     │    ✅      │   ✅    │ Fast    │
│ REPEATABLE READ     │    ❌     │    ❌      │   ✅    │ Medium  │
│ SERIALIZABLE        │    ❌     │    ❌      │   ❌    │ Slowest │
└─────────────────────┴───────────┴───────────┴─────────┴─────────┘
✅ = Problem CAN occur  ❌ = Problem PREVENTED
```

```
-- Check current level
SELECT @@transaction_isolation;
SHOW VARIABLES LIKE 'transaction_isolation';

-- Practical example with isolation
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
    SELECT SUM(salary) FROM employees;   -- Gets a lock
    -- No other transaction can insert/update employees now
COMMIT;
```

###### Real Example
```
DELIMITER $$
CREATE PROCEDURE place_order(
    IN p_emp_id     INT,
    IN p_product_id INT,
    IN p_quantity   INT
)
BEGIN
    DECLARE v_stock     INT;
    DECLARE v_price     DECIMAL(10,2);
    DECLARE v_total     DECIMAL(10,2);
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT 'Transaction Failed - Order Rolled Back' AS message;
    END;

    START TRANSACTION;

        -- Check stock availability
        SELECT stock, price INTO v_stock, v_price
        FROM products
        WHERE product_id = p_product_id
        FOR UPDATE;                           -- Lock the row

        IF v_stock < p_quantity THEN
            SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Insufficient stock';
        END IF;

        -- Calculate total
        SET v_total = v_price * p_quantity;

        -- Create order
        INSERT INTO orders (emp_id, product_id, quantity, total, order_date)
        VALUES (p_emp_id, p_product_id, p_quantity, v_total, CURDATE());

        SAVEPOINT order_created;

        -- Reduce stock
        UPDATE products
        SET stock = stock - p_quantity
        WHERE product_id = p_product_id;

        SAVEPOINT stock_updated;

    COMMIT;

    SELECT 'Order Placed Successfully' AS message, v_total AS total_amount;
END$$
DELIMITER ;

-- Execute
CALL place_order(1, 1, 2);
```

