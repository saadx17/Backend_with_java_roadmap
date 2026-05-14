A data model is ==a structured blueprint or abstract framework== that defines how data is organized, stored, and accessed within a database. It defines data elements, their attributes, and relationships to ensure data integrity and consistency. Common types include conceptual, logical, and physical models, often visualized as diagrams to guide database design.

# ER Model
The Entity-Relationship Model ([ER Model](https://www.geeksforgeeks.org/dbms/introduction-of-er-model/)) is a conceptual model for ==designing a database==. This model represents the logical structure of a database, including entities, their attributes, and relationships between them.
##### Building an ER Model (The Blueprint)
An ER Model is essentially a flowchart that maps out how your data connects before you write any code. It is made of three key building blocks:

1. ==Entities== (The Nouns)
- **What they are:** The real-world objects or concepts you want to store data about. If you were building a school database, your entities would be things like `Student`, `Course`, or `Professor`.
- **How to draw them:** Represented by **rectangles**.

2. ==Attributes== (The Adjectives)
- **What they are:** The specific details or properties that describe an entity. For a `Student` entity, the attributes would be `StudentID`, `FirstName`, `LastName`, and `DateOfBirth`.
- **How to draw them:** Represented by **ovals** connected to the entity rectangle. (The primary unique identifier, like `StudentID`, gets its name underlined).

3. ==Relationships== (The Verbs)
- **What they are:** How different entities interact with one another. For example, a Student _Enrolls_ in a Course.
- **How to draw them:** Represented by **diamonds** sitting on a line that connects two entities.
- **Cardinality:** You also define the rules of the relationship. Is it one-to-one (One student has one ID card)? One-to-many (One professor teaches many courses)? Or many-to-many (Many students enroll in many courses)?

##### Let's Study an ER Diagram
ER diagrams use very specific visual shorthand to explain how a database should be built. Let's decode this diagram step-by-step, starting with the shapes and then moving into those specific underlines and colors.

![[ER_Diagram.png]]

###### 1. The Core Shapes

1. **Rectangles = ==Entities== :** These are the main objects or concepts in the game. `Account`, `Region`, `Item`, and `Creep` (monsters) are standard entities.
   - _Notice the Double Rectangles?_ Look at `Character` or `Item Instantiation`. A double rectangle means it is a **Weak Entity**. It cannot exist on its own. A `Character` cannot exist without an `Account` tied to it.

2. **Diamonds = ==Relationships== :** These describe how entities interact. An Account `Has` a Character. A Character `RanInto` a Creep.
   - _Notice the Double Diamonds?_ Look at the `Has` diamond. A double diamond is an **Identifying Relationship**. It connects a standard (strong) entity to a weak entity.

3.  **Ovals = ==Attributes== :** These are the specific details stored about an entity. An `Account` has a `Password`, an `Item` has an `ItemType`, etc.

###### 2. Decoding the Lines and Underlines
1. **Full Underline** (e.g., `AcctName`, `RegionName`, `ItemName`): This denotes a ==Primary Key==. A primary key is a 100% unique identifier. This means the database will mathematically reject any attempt to create two accounts with the exact same `AcctName`.

   - **The Red Circles:** In standard database notation, the solid underline is all that is needed to denote a Primary Key. The red glowing circles here are just a stylistic choice by whoever drew the diagram to make those critical primary keys pop out visually.

2. **Dotted Underline** (e.g., `CharName`, `IDNum`): This denotes a ==Partial Key== (sometimes called a discriminator). This is used specifically for those *Weak Entities*. It means the name `CharName` only needs to be unique _within its parent account_. 

   - Note: While many modern games force globally unique character names, this specific diagram is designed so that Account A and Account B could technically both have a character named "Gandalf", but Account A cannot have _two_ "Gandalfs").

3. **Double Ovals** (e.g., `Foliage` on Region): This is a ==Multivalued Attribute==. It tells the database that a single region can hold multiple different values for this detail. A forest region's foliage might be Pine, Oak, Fern.

4. **Dashed/Dotted Oval Outline** (e.g., `PlayersIn` on Region): This is a ==Derived Attribute==. This means the database does _not_ actually save a hard number for this. Instead, it calculates the number on the fly by counting how many Characters currently have a `Contains` relationship with that Region.

###### 3. Special Cases in this Diagram

1. **Attributes Attached to Diamonds (Relationships)** Look at the `Has` relationship between Account and Character. It has its own ovals: `LastPlayed` and `CreatedOn`. This means those details don't belong to the Account alone, nor the Character alone, they belong to the _specific link_ between them.

2. **The "Instantiation" Concept** You might wonder why there is an `Item` and an `Item Instantiation`. This is a classic database design pattern for games and commerce.

- `Item` (The Blueprint): This holds the core stats for an "Iron Sword". You only need to store the blueprint once.

- `Item Instantiation` (The Physical Object): This is the specific Iron Sword _your_ character is currently holding. It links back to the blueprint, but it has its own unique `IDNum` and maybe a special `Modifier` (like "+1 Fire Damage" that only your specific sword has).

###### 4. Cardinality (The 1s, ns, and ms)
The little letters and numbers on the lines tell you how many of one thing can connect to another, you will learn about these in [[Relationships]] :

- **1 to n (One-to-Many):** Look at Account to Character. `1` Account `Has` `n` (many) Characters.

- **m to n (Many-to-Many):** Look at Character to Creep (`RanInto`). `m` (many) Characters can run into `n` (many) Creeps.

- **Double Lines vs Single Lines:** Look at the lines connecting `Character` to `Has`. It is a double line. This represents **Total Participation**. It enforces a strict rule: A character _must_ belong to an account to exist. The single line on the Account side means an Account _can_ exist even if it hasn't created a Character yet.

# Relational Model
If the ER Diagram is the rough sketch of a house, this [Relational Model](https://www.geeksforgeeks.org/dbms/relational-model-in-dbms/) (often called a Database Schema) is the actual architect's blueprint. This is exactly what a database engineer looks at right before they start typing out `CREATE TABLE` SQL commands.
**Relational Model** is the actual, structural framework you build inside the computer. It is the most common way data has been organized and stored since the 1980s. When you hear people talk about "SQL databases" (like MySQL, PostgreSQL, or SQLite), they are talking about databases built on the Relational Model.

##### Let's decode the visual language of this Model
![[relational-model.png]]

###### 1. The Boxes (Tables / Relations)
In the ER diagram, these were your "Entities" (the rectangles). Here, they have been formalized into actual database ==Tables==.

   - Instead of just abstract concepts, we now have concrete storage bins: `Members`, `Classes`, `Instructors`, etc.

###### 2. The Text Inside (Columns & Data Types)
Inside each box is a list of the columns that will make up that table.

- **The Big Upgrade:** Notice that the attributes now have specific data types next to them! You can see `(string)`, `(integer)`, `(datetime)`, and `(boolean)`. This is where your previous knowledge comes in, when writing the SQL, you would translate `(string)` to `VARCHAR`, `(integer)` to `INT`, and `(boolean)` to `TINYINT(1)`.

###### 3. The Keys (The Anchors and The Bridges)
This is the most important part of a relational model. It's how the database prevents duplicate data and connects information together.

1. **Primary Key (The Anchor):** Every single box here has an `id (primary key)` at the very top. This is the unique identifier for that specific row. It guarantees that even if you have two members named "John Smith," the database knows exactly which one is `id: 1` and which is `id: 2`.

2. **Foreign Key (The Bridge):** Look at the colored rows. A Foreign Key is simply a Primary Key from _another_ table that you drop into your current table to create a link.

   - _The Color Coding:_ This diagram uses colors perfectly to show these bridges. The yellow `id` in `Classes` is linked directly to the yellow `class_id` in the `Class Schedule` table. This tells the database, "This specific schedule block belongs to this specific class type."

###### 4. The Lines (Crow's Foot Notation)
In the ER diagram, relationships were diamonds with `1` and `n`. In a relational model, industry professionals use **Crow's Foot Notation** (the shape of the lines connecting the boxes).

- **The Plain Line (The "One"):** Look at the line touching the `Instructors` table. It's just a straight, plain line. This represents "One".

- **The Three-Pronged Fork (The "Many"):** Look at the other end of that same line touching the `Class Schedule` table. It splits into three little lines (like a bird's footprint). This represents "Many".

- **How to read it:** Start at one box and read toward the other. "One Instructor can teach Many Class Schedules." And reading backward: "Many Class Schedules are taught by One Instructor."

###### 5. The "Many-to-Many" Secret (Junction Tables)
In the real world, a Member can sign up for Many classes, and a Class Schedule can contain Many members.
Databases **hate** direct Many-to-Many relationships. They mathematically cannot handle them without breaking. To solve this, database engineers create what is called a **Junction Table** (or linking table).

- Look at the **`Member Signups`** table. This table exists for one single reason: to act as a middleman.

- It grabs the `member_id` (Blue) and the `class_schedule_id` (Green) and puts them together in one row. It turns one impossible Many-to-Many relationship into two perfect One-to-Many relationships.

# Hierarchical Model
While the Relational Model is the modern standard for general-purpose databases, the **Hierarchical Model** is one of the oldest ways to organize data. It was actually developed by IBM in the 1960s to manage the massive amounts of manufacturing data needed for the Apollo space program!
If the relational model is a collection of connected spreadsheets, the hierarchical model is an **upside-down tree**.

##### Here is how to read the blueprint of the Hierarchical Model
![[hierarchical_model.jpg]]

###### 1. The Structure: The Upside-Down Tree
A hierarchical database organizes data in a tree-like structure, starting from a single point at the top and branching downward.

- **The Root Node (`Shoes`):** This is the very top of the tree. Every single piece of data in this database eventually traces back to this single origin point. It has no parent.

###### 2. The Golden Rule: One Parent Only
The defining characteristic of a hierarchical model is strict ownership.

- **Parent Nodes (`Women's Shoes`, `Men's Shoes`):** These nodes sit directly below the root. They "own" the categories below them.

- **Child / Leaf Nodes (`High Heels`, `Bellies`, `Sports Shoes`, `Sneakers`):** These sit at the bottom.

- **The Rule:** A parent can have many children (a 1-to-N relationship), but **a child can only ever have ONE parent.** Notice how `Sneakers` only connects to `Men's Shoes`. It is mathematically impossible in this model for a child to connect to two parents.

###### 3. How Data is Accessed (Navigation)
In a relational database (like MySQL), you can jump straight to a specific row of data using a SQL `SELECT` query.
In a hierarchical database, you cannot teleport. You must **navigate** the tree from the top down.

- If the database wants to look up information about `Sneakers`, it cannot just search for "Sneakers".

- It must walk the exact path: `Shoes` ➔ `Men's Shoes` ➔ `Sneakers`.

##### The Industry Perspective: Why did we move away from this?
Hierarchical databases are incredibly fast if you know exactly what path you are walking down, which is why they are still used today in specific scenarios like Windows Registry or XML/HTML documents (the DOM tree).

**But it has a fatal flaw for complex data:** It is too rigid. Imagine a shoe company creates a line of "Unisex Sneakers." Where does it go?

- You can't connect it to _both_ `Men's Shoes` and `Women's Shoes` because that breaks the "One Parent Only" rule.

- You are forced to either duplicate the `Sneakers` data (putting a copy under both parents, which wastes space and creates update errors) or create a completely new, awkward parent category.

Because the real world is messy and heavily interconnected (many-to-many relationships), the industry largely abandoned the Hierarchical Model in favor of the flexible Relational Model (SQL) you were looking at previously!