# Introduction

### What is SQL

SQL (**Structured Query Language**) is the **declarative** language used to **define, query, and manipulate data** stored in a **relational database management system (RDBMS)**. “Declarative” means you specify **what result you want** (e.g., “give me all patients with BMI > 30”) rather than describing **how** the database should retrieve it internally (e.g., which index to use). This separation is powerful: the database’s **query optimizer** can choose an efficient **execution plan** (index scan vs. sequential scan, join algorithm selection, etc.) without you rewriting your logic.

From a layman’s perspective, SQL is like a standardized way to “ask questions” and “make changes” to large, organized collections of information—similar to how you might filter, sort, and summarize an Excel sheet, but at **much larger scale**, with **multiple linked tables**, strict **data integrity**, and **concurrent multi-user access**.

#### Purpose / Why we need it
* Provides a common language to interact with relational data across products and vendors.
* Enables **data definition (DDL)**, **data manipulation (DML)**, and **access control (DCL)** in a consistent framework.
* Supports complex operations: joins, aggregations, constraints, transactions, and analytical computations

---

### History of SQL

SQL emerged from IBM research in the 1970s as part of the **System R** project. Researchers **Donald Chamberlin** and **Raymond Boyce** developed an early language called SEQUEL (Structured English Query Language), grounded in **E. F. Codd’s relational model**—the theoretical foundation that formalized data as relations (tables) and promoted set-based querying. Over time, SEQUEL became SQL, and SQL became the dominant interface for relational databases in industry. 

* Explains why SQL is **set-based**, table-centric, and optimized for declarative querying.
* Clarifies why relational concepts (keys, joins, normalization) are central to serious data work.

---

### SQL Standards (ANSI, ISO)
SQL is standardized to improve **portability** and reduce vendor lock-in. The international standard is **ISO/IEC 9075 (Database languages — SQL)**, revised periodically; it defines the grammar, core features (“Core SQL”), and optional extensions. Vendors (PostgreSQL, MySQL, SQL Server, Oracle) implement varying subsets and add dialect-specific extensions, but the standard provides a shared baseline. 

#### Purpose / Why we need standards
* **Portability**: skills and many queries transfer between systems.
* **Interoperability**: tooling, drivers, and frameworks can target common SQL concepts.
* **Clarity**: defines formal behavior (data types, constraints, transaction semantics) that reduces ambiguity.

Standards are like “rules of grammar” for a language; every region may have accents (dialects), but the core grammar keeps communication possible.

---

## Types of Databases
### Relational Databases (RDBMS)

A **relational database** stores data as **relations (tables)** with rows and columns, guided by schema rules and integrity constraints. It excels when you need:

* strong **data consistency**
* well-defined relationships (e.g., patient → visits → lab tests)
* complex joins and reporting
* robust **ACID transactions** (Atomicity, Consistency, Isolation, Durability)

This design originates from Codd’s relational model and underpins most enterprise systems. 

### Non-relational Databases (NoSQL overview)

**NoSQL** is an umbrella term for databases that do not primarily use the classic relational table model. Common categories include:

* **Document stores** (JSON-like documents)
* **Key–value stores**
* **Wide-column stores**
* **Graph databases**

NoSQL systems often prioritize scalability and distributed operation. Many are designed around trade-offs described by the **CAP theorem** (Consistency, Availability, Partition tolerance) in distributed systems: you cannot guarantee all three simultaneously under network partitions. 

#### Purpose / Why NoSQL exists
* Handles massive scale, high write throughput, flexible evolving data structures, or graph-like relationships more naturally in some use cases.

---

### SQL vs NoSQL

A practical way to compare SQL vs NoSQL is to focus on **data model**, **consistency guarantees**, **schema rigidity**, and **query complexity**.

**SQL (Relational) tends to be best when you need**:

* **Structured schema** and strong data integrity
* **Joins** across many related entities
* **Complex analytics** and reporting
* Strong transactional behavior (ACID)

**NoSQL tends to be best when you need**:

* **Schema flexibility** (rapidly changing fields)
* Horizontal scaling for distributed workloads
* High-volume ingestion (events, logs, telemetry)
* Specialized models (graph traversals, document retrieval)

SQL is like a well-organized library with strict catalog rules (you always know where and how items are stored).

NoSQL is like a warehouse optimized for fast storage and retrieval of many kinds of packages, sometimes with fewer strict rules about uniform labeling. 

---

### Database vs Schema vs Table

These are levels of organization:

* **Database**: the top-level container (often an application or domain). It holds schemas, tables, views, functions, etc.
* **Schema**: a logical namespace inside a database (think “folder” or “module”) that groups objects and controls naming/permissions.
* **Table**: the core storage structure—like a spreadsheet grid—where rows represent records and columns represent attributes.

#### Purpose / Why we separate them
* **Security**: permissions can be applied at database/schema/table levels.
* **Organization**: separate domains (e.g., clinical, genomics, audit) cleanly.
* **Maintainability**: reduces naming collisions and improves governance.

---

### Row vs Column

A **table** is a two-dimensional structure:

* **Row (record/tuple)**: one instance of an entity (one patient, one sample, one transaction).
* **Column (field/attribute)**: one property measured for all rows (age, sex, sample_type, expression_value).

#### Purpose / How this helps
* Enables consistent data validation by column data types and constraints.
* Enables efficient column-based filtering and indexing (e.g., index on patient_id).

A row is one filled form; columns are the questions on the form.

---

### Primary Key Concept

A **Primary Key (PK)** is a constraint that enforces **entity integrity**: it uniquely identifies each row in a table.

Core properties:

* **Uniqueness**: no two rows share the same PK value.
* **Not NULL**: every row must have a PK value (in standard relational practice).

#### Purpose / Why we need it
* Prevents duplicate records (“two different rows both claiming to be patient #123”).
* Enables reliable joins and references from other tables.
* Improves performance when indexed (most systems implement PK with an index).

A primary key is like a government-issued ID number: one person, one unique ID.

---

### Foreign Key Concept

A **Foreign Key (FK)** enforces **referential integrity**: it ensures that values in one table correspond to valid values in another table’s primary (or unique) key.

Example idea:

* `visits.patient_id` (FK) must exist in `patients.patient_id` (PK)

#### Purpose / Why we need it
* Prevents “orphan” records (e.g., a visit referencing a patient that does not exist).
* Encodes real-world relationships into the database so the system can enforce correctness.
* Supports controlled cascading behaviors:
    * **ON DELETE CASCADE** (delete dependent rows automatically)
    * **ON DELETE RESTRICT/NO ACTION** (block deletion if dependents exist)
    * **ON UPDATE CASCADE** (propagate key changes)

A foreign key is like a receipt line item referencing a real invoice number—if the invoice doesn’t exist, the line item is invalid.

---

#### References

1. ISO. Database languages SQL — ISO/IEC 9075-1:2023 (overview page). 
2. ISO. ISO/IEC 9075 defines SQL; scope and minimum requirements (example: ISO/IEC 9075-11:2011 abstract).
3. IBM. What Is Structured Query Language (SQL)? (includes System R / SEQUEL origin and key figures).
4. Codd, E. F. (1970). A Relational Model of Data for Large Shared Data Banks (foundational relational theory).
5. IBM. Relational database history (context and impact of the relational model).
6. IBM. What is the CAP Theorem? (distributed systems trade-offs relevant to NoSQL).


```python

```
