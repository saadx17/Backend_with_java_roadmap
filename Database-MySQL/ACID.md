In a live database, thousands of people might be reading, writing, and updating data at the exact same millisecond. To prevent chaos, database management systems bundle operations into single units of work called **transactions**.
**[ACID](https://www.geeksforgeeks.org/dbms/acid-properties-in-dbms/)** is an acronym representing the four mandatory properties a transaction must possess to guarantee that the database remains accurate and reliable, even in the event of a total system crash or power failure.

To explain how the syntax works, we must use the classic engineering scenario: **The Bank Transfer.**

- _Step 1:_ Deduct $100 from Alice's account.
- _Step 2:_ Add $100 to Bob's account.

- _The Danger:_ What if the database server crashes exactly between Step 1 and Step 2? Alice loses $100, Bob gets nothing, and the money vanishes into thin air.

Here is how ACID properties prevent that disaster, along with the syntax used to enforce them.

#### 1. Atomicity ("All or Nothing")
**The Concept:** A transaction is a single, indivisible unit of work. Either every single step succeeds, or the entire process aborts and the database acts like nothing ever happened. There is no such thing as "half-finished."

**The Syntax:** You enforce this using ==`START TRANSACTION`==, ==`COMMIT`==, and ==`ROLLBACK`==.
```
START TRANSACTION; -- Tells MySQL: "Group the following commands together."

UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';

-- If both updates succeed without any errors:
COMMIT; -- This makes the changes permanent.

-- If an error happens (e.g., the server loses connection before Step 2):
ROLLBACK; -- This instantly undoes Step 1, giving Alice her money back.
```

#### 2. Consistency ("The Rules are the Rules")
**The Concept:** A transaction can only bring the database from one valid state to another valid state. It cannot break the physical rules of the database (like Primary Keys, Foreign Keys, or Data Types).

**The Syntax:** You actually write this syntax when you _build_ the table, using Constraints.
```
-- Let's add a rule that a bank balance can physically NEVER go below zero.
ALTER TABLE accounts 
ADD CONSTRAINT check_balance CHECK (balance >= 0);

START TRANSACTION;
-- If Alice only has $50, this next line will violate the CHECK constraint.
UPDATE accounts SET balance = balance - 100 WHERE name = 'Alice';
-- Because Consistency is broken, MySQL will throw an error and refuse to COMMIT.
```

#### 3. Isolation ("Mind your own business")
**The Concept:** In a real-world app, 1,000 people might be hitting the database at the exact same millisecond. Isolation ensures that these concurrent transactions don't interfere with each other. If Transaction A is currently updating Alice's balance, Transaction B cannot see that "in-progress" math until Transaction A is completely finished (`COMMIT`).

**The Syntax:** You control this by setting the database's **Isolation Level**.
```
-- This is the strictest level. Transactions wait in line and execute one 
-- after the other (safest, but slows down performance on busy sites).
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- This is the default in MySQL (InnoDB). It strikes a balance, ensuring 
-- that if you read a row twice in one transaction, the data won't change 
-- halfway through, even if someone else updates it.
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

#### 4. Durability ("Written in Stone")
**The Concept:** Once you successfully execute a `COMMIT`, that data is permanently saved to the physical hard drive. Even if someone kicks the power cord out of the server one millisecond after the commit goes through, the data will survive and be exactly where you left it when the server reboots.

**The Syntax:** There is no specific SQL command you write for this!

Durability is guaranteed by the database engine itself (like InnoDB in MySQL). The moment you type `COMMIT;`, the database engine automatically writes a physical "transaction log" to your SSD/Hard Drive before it even tells your application that the save was successful.

#### Example:
```
-- Complete Bank Transfer with ACID
START TRANSACTION;

-- Check Atomicity: Both must succeed
  UPDATE accounts
  SET balance = balance - 1000
  WHERE account_id = 'A001' AND balance >= 1000; -- Consistency check
  
  -- Verify rows affected (Consistency)
  -- If 0 rows affected, balance was insufficient
  
  UPDATE accounts
  SET balance = balance + 1000
  WHERE account_id = 'A002';
  
  -- Log for Durability
  INSERT INTO transaction_log
  VALUES (NOW(), 'A001', 'A002', 1000, 'TRANSFER');
  
COMMIT;     -- Durable: persisted to disk
            -- Isolated: other transactions see consistent view
```
