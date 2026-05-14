**DCL** stands for **Data Control Language**. It is a subset of SQL commands used to control access to data stored in a database.
DCL manages **permissions** and **access control**.

### The `GRANT`

###### User Management (needed before GRANT)
```
-- Create users
CREATE USER 'john'@'localhost'    IDENTIFIED BY 'password123';
CREATE USER 'jane'@'%'           IDENTIFIED BY 'securepass';    -- any host
CREATE USER 'admin'@'192.168.1.%' IDENTIFIED BY 'adminpass';   -- IP range

-- Check users
SELECT user, host FROM mysql.user;
```

###### Syntax
```
GRANT privilege(s)
ON database.table
TO 'user'@'host'
[WITH GRANT OPTION];
```

###### Grant Level - `Database`
```
-- Grant ALL on specific database
GRANT ALL PRIVILEGES
ON company_db.*
TO 'john'@'localhost';

-- Grant specific privileges
GRANT SELECT, INSERT, UPDATE
ON company_db.*
TO 'jane'@'%';

-- Grant only SELECT
GRANT SELECT
ON company_db.*
TO 'readonly_user'@'localhost';
```

###### Grant Level - `Table`
```
-- Grant on specific table
GRANT SELECT, INSERT
ON company_db.employees
TO 'jane'@'%';

-- Grant SELECT on specific columns
GRANT SELECT (emp_id, first_name, last_name, dept_id)
ON company_db.employees
TO 'limited_user'@'localhost';

-- Grant UPDATE on specific columns
GRANT UPDATE (salary, dept_id)
ON company_db.employees
TO 'hr_user'@'localhost';
```

###### Grant Level - `Global`
```
-- Grant on ALL databases (*)
GRANT SELECT ON *.* TO 'analyst'@'localhost';

-- Grant SUPER admin
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
```

###### Privilege Types Reference
```
┌──────────────────┬──────────────────────────────────────────┐
│ Privilege        │ Description                              │
├──────────────────┼──────────────────────────────────────────┤
│ SELECT           │ Read data                                │
│ INSERT           │ Add new rows                             │
│ UPDATE           │ Modify rows                              │
│ DELETE           │ Remove rows                              │
│ CREATE           │ Create databases/tables                  │
│ DROP             │ Delete databases/tables                  │
│ ALTER            │ Modify table structure                   │
│ INDEX            │ Create/drop indexes                      │
│ REFERENCES       │ Create foreign keys                      │
│ EXECUTE          │ Run stored procedures                    │
│ CREATE VIEW      │ Create views                             │
│ SHOW VIEW        │ See view definitions                     │
│ TRIGGER          │ Create triggers                          │
│ ALL PRIVILEGES   │ Everything                               │
│ GRANT OPTION     │ Can grant their privileges to others     │
└──────────────────┴──────────────────────────────────────────┘
```

###### Apply changes immediately
```
FLUSH PRIVILEGES;
```

###### Check grants
```
SHOW GRANTS FOR 'john'@'localhost';
SHOW GRANTS FOR CURRENT_USER();
```

### The `REVOKE`

###### Syntax
```
REVOKE privilege(s)
ON database.table
FROM 'user'@'host';
```

###### All `REVOKE`
```
-- Revoke specific privilege
REVOKE DELETE
ON company_db.*
FROM 'jane'@'%';

-- Revoke multiple privileges
REVOKE INSERT, UPDATE
ON company_db.employees
FROM 'jane'@'%';

-- Revoke ALL privileges
REVOKE ALL PRIVILEGES
ON company_db.*
FROM 'john'@'localhost';

-- Revoke GRANT OPTION
REVOKE GRANT OPTION
ON company_db.*
FROM 'admin'@'localhost';

-- Revoke everything
REVOKE ALL PRIVILEGES, GRANT OPTION
FROM 'john'@'localhost';

-- Drop user (removes all their privileges too)
DROP USER 'john'@'localhost';

-- Refresh
FLUSH PRIVILEGES;

-- Verify
SHOW GRANTS FOR 'jane'@'%';
```
