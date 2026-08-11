# Database & Table Basics

### 1. Database Creation
**Concept** 
A **database** is the highest-level logical container in a relational database management system (RDBMS). It encapsulates:

* Tables
* Schemas
* Views
* Indexes
* Functions
* Permissions

**Database creation** refers to allocating a new, isolated logical space where data objects can be stored and managed.

#### Purpose: Why Do We Need It?
* To **separate applications or domains** (e.g., clinical data vs genomics data)
* To enforce **administrative boundaries**
* To enable **access control**, backups, and maintenance independently

#### How It Helps
* Prevents accidental data mixing
* Enables independent lifecycle management (backup, restore, migration)
* Supports multi-tenant systems

#### Analogy:
A database is like a **building**, while tables are the **rooms inside it**.

---

### 2. Database Deletion
**Concept**

Database deletion removes the **entire logical container**, including all tables and data within it.

#### Purpose
* Cleanup of unused or obsolete systems
* Removing test or staging environments
* Enforcing data retention policies

#### How It Helps
* Frees disk space
* Reduces administrative overhead
* Prevents confusion from legacy databases

#### Important Note
* Database deletion is **irreversible**
* Requires high-level privileges

#### Analogy:
Demolishing a building — everything inside is gone permanently.

---

### 3. Database Selection
**Concept**

Before issuing queries, the database engine must know **which database context** to operate in.

#### Purpose
* Ensures queries run against the correct dataset
* Avoids ambiguity when multiple databases exist

#### How It Helps
* Prevents accidental operations on wrong databases
* Improves clarity and safety

#### Analogy:
Choosing which **notebook** you are writing in before adding notes.

---

### 4. Table Creation
**Concept**

A **table** is the fundamental structure used to store data in rows and columns.

Each table definition includes:

* Column names
* Data types
* Constraints (PRIMARY KEY, NOT NULL, etc.)

#### Purpose
* To model real-world entities (patients, genes, orders, samples)
* To enforce structure and validation at the database level

#### How It Helps
* Ensures consistent data format
* Enables relational operations (joins, aggregations)
* Improves data quality and integrity

#### Analogy:
A table is like a **spreadsheet template** where each column has strict rules.

---

### 5. Table Deletion
**Concept**

Table deletion removes the **structure and data** of a table from the database.

#### Purpose
* Removing obsolete data models
* Refactoring schema design
* Cleaning development environments

#### How It Helps
* Prevents stale or misleading data usage
* Simplifies schema maintenance

#### Technical Consideration
* Foreign key dependencies may block deletion
* Deletion is permanent

---

### 6. Table Truncation
**Concept**

**Truncation** removes **all rows from a table** while preserving its structure.

#### Key Technical Properties
* Faster than DELETE
* Minimal logging
* Resets auto-increment counters
* Cannot be rolled back in many systems

#### Purpose
* Bulk data removal
* Resetting staging or temporary tables

#### How It Helps
* High-performance cleanup
* Maintains table schema for reuse

#### Analogy:
Emptying all pages from a notebook without tearing out the notebook itself.

---

### 7. Table Renaming
**Concept**

Table renaming changes the **identifier (name)** of a table without altering its data.

#### Purpose
* Improve naming clarity
* Reflect evolving business logic
* Standardize naming conventions

#### How It Helps
* Improves schema readability
* Avoids data duplication during refactoring

---

### 8. Data Types (Overview)
**Concept**

A **data type** defines:

* What kind of data a column can store
* How much storage it uses
* What operations are allowed

#### Purpose
* Enforces data correctness
* Optimizes storage and performance
* Enables query optimization

#### How It Helps
* Prevents invalid data (e.g., text in numeric columns)
* Improves indexing efficiency

---

### 9. Numeric Data Types
**Concept**

Numeric types store **quantitative values**.

Examples:

* Integers (IDs, counts)
* Floating-point numbers (measurements)
* Fixed-precision decimals (financial data)

#### Purpose
* Enable arithmetic operations
* Support aggregations and analytics

#### How It Helps
* Accurate mathematical computations
* Efficient indexing and sorting

#### Analogy:
Used for anything you can **count** or **measure**.

---

### 10. String (Character) Data Types
**Concept**

String types store **textual information**.

Examples:
* Names
* Identifiers
* Descriptions
* Sequences (in bioinformatics metadata)

#### Purpose
* Represent human-readable information
* Support pattern matching and searching

#### How It Helps
* Enables filtering using LIKE, regex, etc.
* Stores flexible descriptive data

---

### 11. Date & Time Data Types
**Concept**

These types represent **temporal information**.

Examples:

* Dates (birth date)
* Timestamps (event time)
* Intervals (duration)

#### Purpose
* Capture time-based events
* Enable time-series analysis

#### How It Helps
* Supports ordering and filtering by time
* Enables trend and cohort analysis

Layman analogy:
A built-in calendar and clock for your data.

---

### 12. Boolean Data Type
**Concept**

Boolean types store **logical values**:

* TRUE
* FALSE
* Sometimes NULL (unknown)

#### Purpose
* Represent binary states (yes/no, active/inactive)

#### How It Helps
* Simplifies conditional logic
* Improves query readability

---

### 13. NULL Values
**Concept**

NULL represents **absence of a value**, not zero, not empty string.

#### Purpose
* Model unknown, missing, or inapplicable data

#### How It Helps
* Avoids forcing incorrect placeholder values
* Enables accurate representation of real-world uncertainty

#### Important Technical Notes
* NULL ≠ NULL (three-valued logic)
* Requires special handling (IS NULL)

#### Analogy:
Leaving a question unanswered rather than guessing.

---

### 14. Default Values
**Concept**

A **default value** is automatically assigned to a column when no explicit value is provided.

#### Purpose
* Ensure data completeness
* Reduce repetitive input
* Enforce consistent assumptions

#### How It Helps
* Prevents accidental NULLs
* Simplifies inserts
* Improves data consistency

#### Analogy:
Pre-filled form values that apply unless you change them.

---

#### References

* Date, C. J. An Introduction to Database Systems. Addison-Wesley.
* Silberschatz, A., Korth, H. F., Sudarshan, S. Database System Concepts. McGraw-Hill.
* ISO/IEC 9075:2023 – Database languages — SQL.
* PostgreSQL Documentation – Data Types and Schema Management.
* MySQL Reference Manual – Data Definition Language (DDL).
* IBM Knowledge Center – Relational Database Concepts.
