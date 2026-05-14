In MySQL, **keys** are attributes or sets of attributes used to identify records uniquely and establish relationships between tables. They serve as constraints to maintain data integrity and as indexes to improve search performance.
MySQL features several key types to ensure data integrity and define relationships, with the most common being,

1. Primary Key
2. Foreign Key
3. Unique Key
4. Candidate Key
5. Composite Key
6. Alternate Key
7. Super Key

# The Keys
### 1. Primary Key (PK)
This is the ultimate identifier. It guarantees that every single row in your table is 100% unique.

- **The Rules:** It cannot contain `NULL` (empty) values, and the value must never change. Every table should have one, and only one, Primary Key.

- **Example:** `User_ID`, `Employee_Number`, or a `Receipt_Number`.

**Syntax:**
```
CREATE TABLE Members (
    member_id INT AUTO_INCREMENT PRIMARY KEY, -- This is the primary key
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);
```

**Syntax after a table already exists:**
```
ALTER TABLE Members 
ADD PRIMARY KEY (member_id);
```

**Syntax for droping a primary key:**
```
ALTER TABLE table_name DROP PRIMARY KEY;
```

### 2. Foreign Key (FK)
This is the glue that builds the "Relational" part of the database. It is a column in one table that points directly to the Primary Key of another table, creating a strict link between the two.

- **The Rules:** It prevents you from inserting a record that points to a non-existent ID (e.g., you cannot assign an order to a `User_ID` that doesn't exist).

- **Example:** A `Customer_ID` column sitting inside an `Orders` table.

**Syntax:**
```
CREATE TABLE Class_Schedule (
    schedule_id INT AUTO_INCREMENT PRIMARY KEY,
    class_name VARCHAR(50),
    instructor_id INT,               -- This is just normal table
    
    -- The Foreign Key syntax
    CONSTRAINT fk_instructor 
    FOREIGN KEY (instructor_id) 
    REFERENCES Instructors(instructor_id)
);
```

**Syntax after a table already exists:**
```
ALTER TABLE Class_Schedule 
ADD CONSTRAINT fk_instructor 
FOREIGN KEY (instructor_id)
REFERENCES Instructors(instructor_id);
```

**Syntax for dropping a foreign key:**
```
ALTER TABLE table_name 
DROP FOREIGN KEY constraint_name;
```

### 3. Unique Key
This key ensures that all values in a specific column are different from one another. It is very similar to a Primary Key, but with a few distinct differences.

- **The Rules:** Unlike a Primary Key, a table can have _multiple_ Unique Keys, and Unique Keys _can_ accept a `NULL` value (though usually only one, depending on the specific SQL [[Engines]]).

- **Example:** An `Email_Address` column. You want to make sure no two users sign up with the exact same email, but the email address isn't the Primary Key (the `User_ID` is).

**Syntax:**
```
CREATE TABLE Members (
    member_id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(100) UNIQUE,           -- Inline definition
    phone_number VARCHAR(15),
    UNIQUE KEY (phone_number)            -- Bottom definition
);
```

**Syntax after a table already exists:**
```
ALTER TABLE Members 
ADD UNIQUE (email);
```

### 4. Composite Key
Sometimes, a single column isn't enough to uniquely identify a row. A Composite Key is a Primary Key (or Unique Key) made by combining two or more columns together.

- **The Rules:** Neither column is entirely unique on its own, but _together_ they form a unique fingerprint.

- **Example:** In a `Class_Roster` table, a `Student_ID` will appear many times (because a student takes many classes), and a `Class_ID` will appear many times (because a class has many students). But the combination of `Student_ID` + `Class_ID` will only ever appear exactly once.

**Syntax:**
```
CREATE TABLE Member_Signups (
    member_id INT,
    class_schedule_id INT,
    signup_date DATETIME,
    -- Define the composite key at the bottom
    PRIMARY KEY (member_id, class_schedule_id) 
);
```

### 5. Candidate Key
When you first design a table, you might have several columns that _could_ uniquely identify a row. All of these eligible columns are Candidate Keys.

- **Example:** In an `Employees` table, both `Social_Security_Number` and `Employee_ID` are perfectly unique. Both are Candidate Keys.

### 6. Alternate Key
Once you look at your Candidate Keys and officially select one to be your Primary Key, the ones that were left behind become Alternate Keys.

- **Example:** If you choose `Employee_ID` to be the Primary Key, then `Social_Security_Number` becomes the Alternate Key (and you would likely apply a Unique Key constraint to it).

### 7. Super Key
This is the broadest category. A Super Key is _any_ combination of columns that uniquely identifies a row.

- **Example:** If `Employee_ID` is unique, then the combination of `Employee_ID` + `First_Name` + `Last_Name` is also unique. That entire group is a Super Key. (A Candidate Key is simply a Super Key stripped down to its absolute minimum number of columns).

# SQL Implementation of all Keys
```
CREATE TABLE employees (
      emp_id INT PRIMARY KEY,                       -- Primary Key
      email VARCHAR(100) UNIQUE,                    -- Unique Key
      ssn VARCHAR(20) UNIQUE,                       -- Alternate Key
      dept_id INT,
      name VARCHAR(100) NOT NULL,
      FOREIGN KEY (dept_id) REFERENCES departments(dept_id) -- Foreign Key
);


-- Composite Primary Key
CREATE TABLE enrollment (
       student_id VARCHAR(10),
       course_id VARCHAR(10),
       grade CHAR(1),
       PRIMARY KEY (student_id, course_id)          -- Composite Key
);
```