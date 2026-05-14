[Relationships](https://www.geeksforgeeks.org/sql/relationships-in-sql-one-to-one-one-to-many-many-to-many/) in SQL define how tables in a relational database are connected and interact through foreign keys, ensuring data integrity and enabling efficient data retrieval by allowing data to be linked across multiple tables.

# 1. One-to-Many
This is the most common relationship in database design. One row in T==able A can be linked to many rows in Table B==, but a row in Table B can only belong to one row in Table A.
**How to build it in MySQL:** You simply place a Foreign Key inside the "Many" table that points to the Primary Key of the "One" table.

**Example:** A Customer can have many Orders.
```
-- The "One" Table
CREATE TABLE Customers (
    customer_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100)
);

-- The "Many" Table
CREATE TABLE Orders (
    order_id INT PRIMARY KEY AUTO_INCREMENT,
    order_date DATE,
    customer_id INT, -- This column will hold the Foreign Key
    FOREIGN KEY (customer_id) REFERENCES Customers(customer_id)
);
```
Because `customer_id` in the `Orders` table is just a standard column, you can insert the number `5` (representing Jane Doe) into a hundred different order rows without breaking any rules.

# 2. One-to-One
This happens when one row in ==Table A is linked to exactly one row in Table B==, and vice versa. It is usually used to split up a massive table or to isolate highly sensitive data (like putting passwords or credit cards in a separate table).
**How to build it in MySQL:** You set it up exactly like a One-to-Many relationship, but you add a `UNIQUE` constraint to the Foreign Key column.

**Example:** A User has exactly one User Profile.
```
CREATE TABLE Users (
    user_id INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50)
);

CREATE TABLE Profiles (
    profile_id INT PRIMARY KEY AUTO_INCREMENT,
    bio TEXT,
    user_id INT UNIQUE, -- The UNIQUE constraint stops it from being 1:Many
    FOREIGN KEY (user_id) REFERENCES Users(user_id)
);
```
By making `user_id` unique in the `Profiles` table, MySQL will throw an error if you try to assign a second profile to the same user.

# 3. Many-to-Many
This is where things get interesting. Relational databases like MySQL ==cannot directly handle Many-to-Many relationships==. If a Student can take many Courses, and a Course can have many Students, placing a Foreign Key in either table creates a logical paradox.

**How to build it in MySQL:** You have to create a third table, called a **Junction Table** (or Pivot Table, or Mapping Table) to sit right between them. This table breaks the Many-to-Many relationship down into two manageable One-to-Many relationships.

**Example:** Students and Courses. First, you create your two main tables without any Foreign Keys in them at all.
```
CREATE TABLE Students (
    student_id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100)
);

CREATE TABLE Courses (
    course_id INT PRIMARY KEY AUTO_INCREMENT,
    course_name VARCHAR(100)
);
```

Then, you create the Junction Table. This table will hold Foreign Keys pointing to _both_ tables, and together, those Foreign Keys act as a Composite Primary Key.

```
CREATE TABLE Student_Course_Enrollments (
    student_id INT,
    course_id INT,
-- Make the combination of the two IDs the Primary Key
    PRIMARY KEY (student_id, course_id), 
-- Link them to the main tables
    FOREIGN KEY (student_id) REFERENCES Students(student_id),
    FOREIGN KEY (course_id) REFERENCES Courses(course_id)
);
```
Now, if Student #1 enrolls in Course #101 and Course #102, the junction table will have two rows:

- Row 1: `student_id = 1`, `course_id = 101`
- Row 2: `student_id = 1`, `course_id = 102`

If you try to enroll Student #1 in Course #101 a second time, MySQL will block it because the combination of `(1, 101)` already exists as the Primary Key for that row.