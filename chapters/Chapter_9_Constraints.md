# Constraints and Data Integrity
### PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, CHECK, DEFAULT, Composite Constraints, Referential Actions, and Omics Applications in PostgreSQL

---

## Introduction: Why Constraints Matter

A database is useful only when the data stored inside it is reliable.

Without rules, a database could contain:

- Duplicate patient IDs
- Samples linked to patients who do not exist
- Missing mandatory tissue labels
- Invalid disease-status values
- Impossible ages
- Duplicate gene–sample measurements
- Negative values where the measurement domain does not allow them
- Broken relationships between biological entities

SQL **constraints** are rules enforced by the database management system to prevent these problems.

A constraint tells PostgreSQL:

> What values are allowed, what relationships must remain valid, and what operations should be rejected.

Constraints are therefore central to **data integrity**, **data quality**, **referential consistency**, and professional relational database design.


## 8.1 What Is Data Integrity?

**Data integrity** refers to the accuracy, consistency, validity, and reliability of data throughout its lifecycle.

Important forms include:

### Entity Integrity

Every row representing an entity should be uniquely identifiable.

Typical identifiers include:

```text
patient_id
sample_id
gene_id
variant_id
```

This is usually enforced using a `PRIMARY KEY`.

### Referential Integrity

Relationships between tables should remain valid.

For example:

```text
samples.patient_id
```

should refer to an existing:

```text
patients.patient_id
```

This is enforced using a `FOREIGN KEY`.

### Domain Integrity

Column values should belong to an acceptable domain.

Examples:

```text
age >= 0
expression_value >= 0
qc_status IN ('Pass', 'Fail', 'Pending')
```

Domain integrity is enforced using:

- Data types
- `CHECK`
- `NOT NULL`
- `DEFAULT`
- Other column or table constraints


## 8.2 Why Do We Need Constraints?

Imagine a biomedical database where someone inserts:

```text
patient_id = 101
patient_id = 101
```

twice.

Or a sample:

```text
sample_id = S100
patient_id = 999
```

even though patient 999 does not exist.

Or:

```text
age = -12
```

Or:

```text
qc_status = 'Maybe'
```

Without constraints, PostgreSQL may accept logically invalid data.

Constraints move validation into the **database layer**, rather than relying entirely on:

- Python code
- R scripts
- Application logic
- Manual checks
- User discipline

This centralizes the rules and ensures that every application using the database must obey them.


## 8.3 Example Biomedical Schema

We will use four simplified tables:

```text
patients
samples
genes
expression
```

Conceptually:

```text
patients
   |
   | 1-to-many
   v
samples
   |
   | 1-to-many
   v
expression
   ^
   |
   | many-to-1
genes
```

The schema lets us demonstrate both entity-level and relationship-level constraints.


## 8.4 PRIMARY KEY — Uniquely Identifying Rows

A `PRIMARY KEY` uniquely identifies each row in a table.

It enforces two major rules:

- The value must be unique
- The value cannot be `NULL`

### Clinical Example



```python
CREATE TABLE patients (
    patient_id INTEGER PRIMARY KEY,
    sex VARCHAR(10),
    age INTEGER,
    diagnosis VARCHAR(100)
);
```

Here:

```sql
patient_id INTEGER PRIMARY KEY
```

means PostgreSQL will reject:

- duplicate patient IDs
- missing patient IDs

### Why Primary Keys Matter

They support:

- Entity integrity
- Reliable joins
- Foreign-key references
- Index-based lookup
- Stable record identity


## 8.5 Duplicate Primary Key Example

Suppose patient 101 already exists:



```python
INSERT INTO patients (patient_id, sex, age, diagnosis)
VALUES (101, 'F', 52, 'Type 2 Diabetes');
```

Trying to insert the same `patient_id` again:



```python
INSERT INTO patients (patient_id, sex, age, diagnosis)
VALUES (101, 'M', 45, 'Healthy');
```

PostgreSQL rejects the second row because the primary-key constraint has been violated.

This is exactly what we want.

The database prevents two different patients from sharing the same official identifier.


## 8.6 Primary Key vs Natural Key vs Surrogate Key

A **natural key** has real-world meaning.

Examples:

```text
ENSG00000141510
rs12345
official_sample_code
```

A **surrogate key** is an artificial identifier created for database use.

Examples:

```text
1
2
3
4
```

or generated UUIDs.

### Example



```python
CREATE TABLE genes (
    id BIGSERIAL PRIMARY KEY,
    ensembl_gene_id VARCHAR(30) UNIQUE,
    gene_symbol VARCHAR(50)
);
```

Here:

```text
id
```

is a surrogate primary key.

```text
ensembl_gene_id
```

is a biologically meaningful unique identifier.

Both approaches have legitimate uses.


## 8.7 Composite Primary Key

A primary key can contain more than one column.

This is called a **composite primary key**.

Consider an expression table:



```python
CREATE TABLE expression (
    sample_id VARCHAR(30),
    gene_id VARCHAR(30),
    expression_value NUMERIC,
    PRIMARY KEY (sample_id, gene_id)
);
```

The combination:

```text
sample_id + gene_id
```

must be unique.

This means:

```text
S001 + BRCA1
```

can occur only once.

But:

```text
S001 + TP53
```

is different and valid.

Likewise:

```text
S002 + BRCA1
```

is also different and valid.


## 8.8 Why Composite Keys Are Useful in Omics

Many biological observations are naturally identified by combinations.

Examples:

```text
sample_id + gene_id
sample_id + variant_id
subject_id + visit_date
gene_id + tissue_id
variant_id + population_id
```

A composite key allows PostgreSQL to enforce uniqueness at the correct analytical unit.


## 8.9 FOREIGN KEY — Enforcing Relationships

A `FOREIGN KEY` ensures that a value in one table refers to a valid row in another table.

Suppose:

```text
patients.patient_id
```

is the parent key.

Then:

```text
samples.patient_id
```

can reference it.



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER,
    tissue VARCHAR(100),
    FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
);
```

Now PostgreSQL enforces:

> Every non-null patient ID stored in `samples.patient_id` must already exist in `patients.patient_id`.

This is **referential integrity**.


## 8.10 Parent and Child Tables

In a foreign-key relationship:

```text
patients
```

is the **parent table**.

```text
samples
```

is the **child table**.

Conceptually:

```text
patients.patient_id
        |
        v
samples.patient_id
```

The child depends on the parent for referential validity.


## 8.11 Foreign-Key Violation Example

Suppose patient 999 does not exist.

This insert:



```python
INSERT INTO samples (sample_id, patient_id, tissue)
VALUES ('S999', 999, 'Blood');
```

will fail if the foreign-key constraint is active.

Why?

Because:

```text
patient_id = 999
```

does not exist in the parent table.

Without the foreign key, we would create an **orphan record**.


## 8.12 Why Foreign Keys Matter in Omics

Foreign keys are especially useful for relationships such as:

```text
patient → sample
sample → expression
gene → expression
variant → annotation
subject → visit
experiment → batch
```

They prevent disconnected biological records from entering the database.


## 8.13 FOREIGN KEY with Multiple Columns

Foreign keys can also reference composite keys.

Example:



```python
CREATE TABLE expression_qc (
    sample_id VARCHAR(30),
    gene_id VARCHAR(30),
    qc_status VARCHAR(20),
    FOREIGN KEY (sample_id, gene_id)
        REFERENCES expression(sample_id, gene_id)
);
```

This means a QC record is valid only if the corresponding sample–gene measurement already exists.


## 8.14 UNIQUE Constraint

A `UNIQUE` constraint requires all non-null values in the constrained column or column combination to be distinct according to PostgreSQL's uniqueness rules.

Example:



```python
CREATE TABLE genes (
    gene_id BIGSERIAL PRIMARY KEY,
    ensembl_gene_id VARCHAR(30) UNIQUE,
    gene_symbol VARCHAR(50)
);
```

Now two rows cannot share the same:

```text
ensembl_gene_id
```

### Why Use UNIQUE?

Use it when a column:

- Must not contain duplicate known values
- Is not necessarily the table's primary key
- Represents an alternative identifier


## 8.15 PRIMARY KEY vs UNIQUE

| Feature | PRIMARY KEY | UNIQUE |
|---|---|---|
| Enforces uniqueness | Yes | Yes |
| Allows NULL | No | Generally yes, depending on null semantics |
| Number per table | One primary key | Multiple unique constraints possible |
| Main row identity | Yes | Usually alternative uniqueness |
| Commonly indexed | Yes | Yes |

A table can therefore have:

```text
one PRIMARY KEY
multiple UNIQUE constraints
```


## 8.16 Composite UNIQUE Constraint

Suppose the same gene can appear in multiple databases, but the combination of source database and external identifier must be unique.



```python
CREATE TABLE gene_crossrefs (
    id BIGSERIAL PRIMARY KEY,
    source_database VARCHAR(50),
    external_id VARCHAR(100),
    UNIQUE (source_database, external_id)
);
```

Now:

```text
Ensembl + ENSG001
```

cannot occur twice.

But:

```text
NCBI + ENSG001
```

would be treated as a different combination.


## 8.17 NOT NULL — Mandatory Values

A `NOT NULL` constraint prevents a column from containing `NULL`.

Example:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER NOT NULL,
    tissue VARCHAR(100) NOT NULL
);
```

Now every sample must have:

- a patient ID
- a tissue label

### Why Do We Need NOT NULL?

Use it when missing values are not acceptable.

Typical biomedical examples:

```text
sample_id
patient_id
gene_id
collection_date
assay_type
```

depending on the data model.


## 8.18 NOT NULL Does Not Mean Non-Empty Text

Consider:

```text
tissue = ''
```

An empty string is not the same as `NULL`.

Therefore:

```sql
tissue VARCHAR(100) NOT NULL
```

still allows:

```text
''
```

unless additional validation is added.

If empty strings should be prohibited, use a `CHECK` constraint.



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    tissue VARCHAR(100) NOT NULL,
    CHECK (tissue <> '')
);
```

## 8.19 CHECK Constraint

A `CHECK` constraint enforces a Boolean condition.

A row is accepted only when the check condition is satisfied according to SQL constraint semantics.

Example:



```python
CREATE TABLE patients (
    patient_id INTEGER PRIMARY KEY,
    age INTEGER CHECK (age >= 0)
);
```

Now PostgreSQL rejects:

```text
age = -5
```

This protects domain integrity.


## 8.20 CHECK with a Range

Suppose age must be biologically plausible:



```python
CREATE TABLE patients (
    patient_id INTEGER PRIMARY KEY,
    age INTEGER CHECK (age BETWEEN 0 AND 120)
);
```

This is an example of **domain validation**.


## 8.21 CHECK with Categorical Values

Suppose QC status must belong to a controlled vocabulary:



```python
CREATE TABLE sample_qc (
    sample_id VARCHAR(30) PRIMARY KEY,
    qc_status VARCHAR(20)
        CHECK (qc_status IN ('Pass', 'Fail', 'Pending'))
);
```

This prevents values such as:

```text
Maybe
UnknownStatus
Pss
```

from silently entering the database.


## 8.22 Omics Example: Expression Constraints

If an expression measurement system produces only non-negative values:



```python
CREATE TABLE expression (
    sample_id VARCHAR(30),
    gene_id VARCHAR(30),
    expression_value NUMERIC
        CHECK (expression_value >= 0),
    PRIMARY KEY (sample_id, gene_id)
);
```

Important scientific caution:

Do not create a check constraint unless the rule is actually valid for the measurement.

Some omics transformations may produce negative values, such as:

```text
log fold changes
z-scores
normalized residuals
```

Database constraints must reflect the scientific meaning of the variable.


## 8.23 CHECK with Multiple Conditions

A check can contain multiple logical conditions.



```python
CREATE TABLE patients (
    patient_id INTEGER PRIMARY KEY,
    age INTEGER,
    sex VARCHAR(20),
    CHECK (age >= 0 AND age <= 120),
    CHECK (sex IN ('F', 'M', 'Other', 'Unknown'))
);
```

Separate constraints are often easier to maintain than one enormous condition.


## 8.24 DEFAULT Constraint

A `DEFAULT` supplies a value automatically when an `INSERT` statement does not provide one.

Example:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    qc_status VARCHAR(20) DEFAULT 'Pending'
);
```

Now:



```python
INSERT INTO samples (sample_id)
VALUES ('S001');
```

automatically results conceptually in:

```text
sample_id = S001
qc_status = Pending
```


## 8.25 Why Defaults Help

Defaults:

- Reduce repetitive data entry
- Standardize initial values
- Prevent unnecessary nulls
- Improve consistency

Typical examples include:

```text
status = 'Pending'
active = TRUE
created_at = current timestamp
version = 1
```


## 8.26 PostgreSQL Default Timestamp Example



```python
CREATE TABLE experiments (
    experiment_id BIGSERIAL PRIMARY KEY,
    experiment_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Now PostgreSQL automatically records the insert time unless another value is provided.


## 8.27 Boolean Defaults

Example:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE
);
```

This is useful when new records should begin in a known state.


## 8.28 DEFAULT Is Not the Same as NOT NULL

Consider:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    qc_status VARCHAR(20) DEFAULT 'Pending'
);
```

A user may still explicitly insert:



```python
INSERT INTO samples (sample_id, qc_status)
VALUES ('S002', NULL);
```

unless `NOT NULL` is also specified.

To enforce both:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    qc_status VARCHAR(20)
        DEFAULT 'Pending'
        NOT NULL
);
```

Now:

- omitted value → `Pending`
- explicit `NULL` → rejected


## 8.29 Combining Multiple Constraints

Real table definitions commonly use several constraints together.



```python
CREATE TABLE patients (
    patient_id BIGSERIAL PRIMARY KEY,
    external_patient_id VARCHAR(50) UNIQUE NOT NULL,
    sex VARCHAR(20)
        CHECK (sex IN ('F', 'M', 'Other', 'Unknown')),
    age INTEGER
        CHECK (age BETWEEN 0 AND 120),
    diagnosis VARCHAR(100),
    enrollment_date DATE DEFAULT CURRENT_DATE
);
```

This single schema combines:

- Primary key
- Unique constraint
- Not-null constraint
- Check constraints
- Default value


## 8.30 Full Omics Table Example



```python
CREATE TABLE genes (
    gene_id VARCHAR(30) PRIMARY KEY,
    gene_symbol VARCHAR(50) NOT NULL,
    chromosome VARCHAR(10) NOT NULL,
    start_position BIGINT CHECK (start_position > 0),
    end_position BIGINT CHECK (end_position > 0),
    CHECK (end_position >= start_position)
);
```

This table enforces genomic coordinate logic:

```text
start > 0
end > 0
end >= start
```

Again, constraints should reflect the coordinate system and biological schema being used.


## 8.31 Naming Constraints

PostgreSQL can automatically generate constraint names.

However, explicitly naming constraints can improve maintenance.

Example:



```python
CREATE TABLE patients (
    patient_id INTEGER,
    age INTEGER,
    CONSTRAINT pk_patients
        PRIMARY KEY (patient_id),
    CONSTRAINT chk_patient_age
        CHECK (age BETWEEN 0 AND 120)
);
```

Named constraints are helpful when:

- Reading error messages
- Dropping constraints
- Modifying schemas
- Maintaining large databases


## 8.32 Column-Level vs Table-Level Constraints

A constraint may be written directly beside a column:



```python
CREATE TABLE patients (
    patient_id INTEGER PRIMARY KEY,
    age INTEGER CHECK (age >= 0)
);
```

or at table level:



```python
CREATE TABLE patients (
    patient_id INTEGER,
    age INTEGER,
    PRIMARY KEY (patient_id),
    CHECK (age >= 0)
);
```

Table-level syntax becomes necessary for constraints involving multiple columns.


## 8.33 Adding Constraints with ALTER TABLE

Constraints can be added after a table already exists.

Example:



```python
ALTER TABLE patients
ADD CONSTRAINT pk_patients
PRIMARY KEY (patient_id);
```

Add a check:



```python
ALTER TABLE patients
ADD CONSTRAINT chk_patient_age
CHECK (age BETWEEN 0 AND 120);
```

Add a foreign key:



```python
ALTER TABLE samples
ADD CONSTRAINT fk_samples_patient
FOREIGN KEY (patient_id)
REFERENCES patients(patient_id);
```

This is common when evolving an existing schema.


## 8.34 What Happens If Existing Data Violate a New Constraint?

Suppose a table already contains:

```text
age = -3
```

and we attempt:



```python
ALTER TABLE patients
ADD CONSTRAINT chk_patient_age
CHECK (age >= 0);
```

PostgreSQL will normally validate existing rows and reject the constraint if violations exist.

Therefore, schema hardening often requires:

1. Audit existing data
2. Correct invalid records
3. Add the constraint
4. Verify application behavior


## 8.35 Dropping Constraints

A constraint can be removed using `ALTER TABLE`.

Example:



```python
ALTER TABLE patients
DROP CONSTRAINT chk_patient_age;
```

This removes the rule, not necessarily the existing data.

After the constraint is removed, future invalid values may become possible unless another validation mechanism exists.


## 8.36 Inspecting Constraint Names

In real databases, constraint names matter.

PostgreSQL tools and catalog queries can be used to inspect table definitions.

In `psql`, one useful command is:

```text
\d table_name
```

For example:

```text
\d patients
```

This displays columns, indexes, and constraints.


## 8.37 Referential Actions

Foreign keys can define what should happen when a referenced parent row is updated or deleted.

Important actions include:

- `NO ACTION`
- `RESTRICT`
- `CASCADE`
- `SET NULL`
- `SET DEFAULT`


## 8.38 ON DELETE NO ACTION

Example:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER,
    FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
        ON DELETE NO ACTION
);
```

Conceptually:

> Do not allow the parent deletion if it would violate referential integrity.

`NO ACTION` is the default behavior when no explicit referential action is specified.


## 8.39 ON DELETE RESTRICT

Example:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER,
    FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
        ON DELETE RESTRICT
);
```

This prevents deleting a patient while dependent sample rows exist.

Conceptually:

```text
Patient has samples
      ↓
Attempt DELETE patient
      ↓
Rejected
```

This can be desirable when child records must never become orphaned.


## 8.40 ON DELETE CASCADE

`CASCADE` automatically propagates the parent deletion to dependent child rows.



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER,
    FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
        ON DELETE CASCADE
);
```

Now:

```text
Delete patient 101
```

may also delete all sample rows referencing patient 101.

### Important Warning

`CASCADE` is powerful and potentially destructive.

Use it only when the child record has no independent meaning without the parent.


## 8.41 Omics Example of CASCADE

Suppose an analysis project contains:

```text
analysis_run
   ↓
temporary_results
```

If temporary results should always disappear when the analysis run is deleted, `ON DELETE CASCADE` may be appropriate.

But if samples represent physical biospecimens that must remain recorded even when a study enrollment changes, automatic cascading may be inappropriate.

Database design must reflect the real scientific lifecycle.


## 8.42 ON DELETE SET NULL

Example:



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER,
    FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
        ON DELETE SET NULL
);
```

If the patient row is deleted, the sample remains, but:

```text
patient_id
```

becomes `NULL`.

This is useful when the child entity may remain valid independently.


## 8.43 SET NULL Requires Nullable Columns

If:



```python
patient_id INTEGER NOT NULL
```

then:

```sql
ON DELETE SET NULL
```

is logically incompatible because PostgreSQL cannot set the column to `NULL`.

Constraint design must therefore be internally consistent.


## 8.44 ON DELETE SET DEFAULT

A foreign key can also use:



```python
ON DELETE SET DEFAULT
```

This replaces the foreign-key value with its column default after deletion.

This is less common in biomedical schemas and should be used only when the default represents a valid referenced value.


## 8.45 ON UPDATE CASCADE

Foreign-key relationships can also define behavior when the referenced key changes.



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id INTEGER,
    FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
        ON UPDATE CASCADE
);
```

If the parent's key changes, PostgreSQL automatically updates matching child keys.

In well-designed systems, primary keys are often stable and rarely updated, but the option is available.


## 8.46 Referential Action Decision Framework

Ask:

### Should child rows prevent parent deletion?

Use:

```text
RESTRICT / NO ACTION
```

### Should child rows disappear with the parent?

Use:

```text
CASCADE
```

### Should child rows remain but lose the relationship?

Use:

```text
SET NULL
```

### Should child rows point to a predefined fallback?

Potentially:

```text
SET DEFAULT
```

The correct choice depends on business and scientific semantics.


## 8.47 Foreign Key vs JOIN

A common beginner confusion is:

> Is a foreign key the same as a join?

No.

A `FOREIGN KEY` is a **constraint**.

It enforces relationship validity.

A `JOIN` is a **query operation**.

It retrieves related data.

For example:

```text
FOREIGN KEY
```

says:

> This sample must reference a valid patient.

Whereas:



```python
SELECT
    p.patient_id,
    s.sample_id
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

says:

> Retrieve patients together with their samples.

Related concept, different purpose.


## 8.48 Constraint vs Index

Constraints and indexes are also related but not identical.

A constraint defines a **data rule**.

An index is primarily a **data-access structure** used to improve lookup and enforce certain uniqueness requirements.

For example:

```text
PRIMARY KEY
UNIQUE
```

typically result in supporting unique indexes in PostgreSQL.

But not every index is a constraint.


## 8.49 Example: Non-Unique Index

This:



```python
CREATE INDEX idx_samples_tissue
ON samples(tissue);
```

may improve queries filtering by tissue, but it does not prevent duplicate tissue values.

Many samples are expected to have:

```text
Blood
Blood
Blood
```

That is perfectly valid.


## 8.50 Constraint Violations Are Useful

Beginners sometimes think a constraint error means the database is broken.

In fact, a constraint violation often means:

> The database successfully prevented invalid data.

Examples include:

- Duplicate key violation
- Foreign-key violation
- Not-null violation
- Check-constraint violation

These errors are protective mechanisms.


## 8.51 Clinical Example: Validating Patient Data



```python
CREATE TABLE patients (
    patient_id BIGSERIAL PRIMARY KEY,
    external_patient_id VARCHAR(50) UNIQUE NOT NULL,
    sex VARCHAR(20)
        CHECK (sex IN ('F', 'M', 'Other', 'Unknown')),
    age INTEGER
        CHECK (age BETWEEN 0 AND 120),
    diagnosis VARCHAR(100),
    enrolled BOOLEAN DEFAULT TRUE NOT NULL
);
```

This schema prevents several common data-quality problems before they ever reach downstream analytics.


## 8.52 Sample Table with Referential Integrity



```python
CREATE TABLE samples (
    sample_id VARCHAR(30) PRIMARY KEY,
    patient_id BIGINT NOT NULL,
    tissue VARCHAR(100) NOT NULL,
    collection_date DATE,
    qc_status VARCHAR(20)
        DEFAULT 'Pending'
        NOT NULL,
    CONSTRAINT fk_samples_patient
        FOREIGN KEY (patient_id)
        REFERENCES patients(patient_id)
        ON DELETE RESTRICT,
    CONSTRAINT chk_sample_qc
        CHECK (qc_status IN ('Pass', 'Fail', 'Pending')),
    CONSTRAINT chk_tissue_nonempty
        CHECK (tissue <> '')
);
```

This table combines:

- Primary key
- Foreign key
- Not null
- Default
- Check constraints
- Referential action


## 8.53 Genes Table



```python
CREATE TABLE genes (
    gene_id VARCHAR(30) PRIMARY KEY,
    gene_symbol VARCHAR(50) NOT NULL,
    chromosome VARCHAR(10) NOT NULL,
    start_position BIGINT NOT NULL,
    end_position BIGINT NOT NULL,
    CONSTRAINT chk_gene_start
        CHECK (start_position > 0),
    CONSTRAINT chk_gene_end
        CHECK (end_position >= start_position)
);
```

This enforces basic coordinate consistency.


## 8.54 Expression Table with Composite Key and Foreign Keys



```python
CREATE TABLE expression (
    sample_id VARCHAR(30) NOT NULL,
    gene_id VARCHAR(30) NOT NULL,
    expression_value NUMERIC NOT NULL,
    CONSTRAINT pk_expression
        PRIMARY KEY (sample_id, gene_id),
    CONSTRAINT fk_expression_sample
        FOREIGN KEY (sample_id)
        REFERENCES samples(sample_id)
        ON DELETE CASCADE,
    CONSTRAINT fk_expression_gene
        FOREIGN KEY (gene_id)
        REFERENCES genes(gene_id)
        ON DELETE RESTRICT,
    CONSTRAINT chk_expression_nonnegative
        CHECK (expression_value >= 0)
);
```

This is a realistic example of relational integrity in an omics-style schema.


## 8.55 Why the Expression Primary Key Is Composite

The table may contain:

```text
S001 | BRCA1
S001 | TP53
S002 | BRCA1
```

These are all valid.

But:

```text
S001 | BRCA1
S001 | BRCA1
```

should normally not appear twice if one row represents one sample–gene expression measurement.

The composite primary key enforces that rule.


## 8.56 Constraints and Normalization

Constraints work together with normalization.

Normalization separates entities such as:

```text
patients
samples
genes
expression
```

Constraints ensure those separated tables remain valid.

Without foreign keys, normalization alone does not guarantee valid relationships.


## 8.57 Constraints and ETL Pipelines

In ETL workflows:

```text
Extract
Transform
Load
```

constraints act as a final validation barrier during the load stage.

If incoming data violate rules, PostgreSQL rejects the problematic rows or transaction depending on how the load is structured.

This can expose:

- Mapping errors
- Missing identifiers
- Invalid category values
- Duplicate records
- Broken foreign-key relationships


## 8.58 Constraints and Data Cleaning

Constraints do not replace data cleaning.

Instead, they complement it.

A strong workflow may be:

```text
Raw data
   ↓
Cleaning / transformation
   ↓
Validation
   ↓
Load into constrained tables
   ↓
Database rejects invalid residual records
```

This is much safer than loading everything into unconstrained production tables.


## 8.59 Staging Tables vs Production Tables

In real pipelines, staging tables may intentionally have fewer constraints.

Why?

Because staging tables are used to inspect incoming raw data.

Then cleaned data are inserted into strongly constrained production tables.

Conceptually:

```text
raw/staging
    ↓
cleaning
    ↓
validated production schema
```

This is common in data engineering and omics ingestion pipelines.


## 8.60 Constraint Design Must Reflect Scientific Meaning

Do not over-constrain biological data.

For example, this may be wrong:



```python
CHECK (expression_value >= 0)
```

if the column stores:

```text
log2 fold change
z-score
residual
```

because negative values are valid.

Likewise:

```sql
CHECK (sex IN ('M', 'F'))
```

may be inappropriate if the database needs broader categories or unknown values.

Constraints should encode **true invariants**, not simplistic assumptions.


## 8.61 Common Beginner Mistakes

### Mistake 1 — Assuming PRIMARY KEY Only Means Unique

A primary key also implies non-nullability.

### Mistake 2 — Using UNIQUE Instead of a Proper Primary Key

A table should normally have a clear primary row identifier.

### Mistake 3 — Forgetting Foreign Keys

Without them, orphan records can enter the database.

### Mistake 4 — Using CASCADE Without Understanding Consequences

Deleting one parent row may delete large amounts of child data.

### Mistake 5 — Confusing DEFAULT with Mandatory Data

A default does not automatically imply `NOT NULL`.

### Mistake 6 — Creating Scientifically Incorrect CHECK Rules

The database rule must match the actual measurement domain.

### Mistake 7 — Assuming NOT NULL Rejects Empty Strings

It does not.


## 8.62 Practical Constraint Decision Framework

Ask these questions:

### Must every row have a unique identity?

Use:

```text
PRIMARY KEY
```

### Must another identifier also be unique?

Use:

```text
UNIQUE
```

### Must a value always be present?

Use:

```text
NOT NULL
```

### Must a value satisfy a logical rule?

Use:

```text
CHECK
```

### Should a missing insert value automatically receive something?

Use:

```text
DEFAULT
```

### Must a row reference an existing row elsewhere?

Use:

```text
FOREIGN KEY
```

### Is uniqueness defined by several columns together?

Use:

```text
Composite PRIMARY KEY
or
Composite UNIQUE
```


## 8.63 Module 8 Summary

In this module, we learned that constraints are database-enforced rules that protect data integrity.

The major concepts are:

- **Entity integrity** ensures each entity can be uniquely identified.
- **Referential integrity** ensures relationships between tables remain valid.
- **Domain integrity** ensures values belong to valid ranges or categories.
- `PRIMARY KEY` uniquely identifies rows and disallows nulls.
- Composite primary keys can identify rows using multiple columns.
- `FOREIGN KEY` protects parent–child relationships.
- `UNIQUE` enforces alternate uniqueness.
- `NOT NULL` makes values mandatory.
- `CHECK` enforces Boolean validation rules.
- `DEFAULT` supplies automatic values when columns are omitted.
- Constraints can be defined at column level or table level.
- Constraints can be added or removed with `ALTER TABLE`.
- Named constraints improve maintainability.
- Referential actions include `NO ACTION`, `RESTRICT`, `CASCADE`, `SET NULL`, and `SET DEFAULT`.
- Foreign keys and joins are related but serve different purposes.
- Constraints and indexes are related but not identical.
- Constraint violations are protective, not inherently problematic.
- Strong constraints are especially valuable in clinical and omics databases where invalid relationships can compromise downstream analysis.


# Key Takeaway

> **Constraints are the rules that make a relational database trustworthy.**

A database should not merely store data.

It should actively prevent invalid states.

In an omics system, constraints can ensure that:

```text
Every sample belongs to a valid patient
Every expression measurement belongs to a valid sample and gene
Every sample–gene pair is unique
Mandatory metadata are present
Controlled categories remain valid
Impossible values are rejected
```

This is the foundation upon which reliable analytics, reproducible research, and advanced SQL are built.


# References

1. PostgreSQL Global Development Group. **PostgreSQL Documentation — Constraints**  
   https://www.postgresql.org/docs/current/ddl-constraints.html

2. PostgreSQL Global Development Group. **PostgreSQL Documentation — CREATE TABLE**  
   https://www.postgresql.org/docs/current/sql-createtable.html

3. PostgreSQL Global Development Group. **PostgreSQL Documentation — ALTER TABLE**  
   https://www.postgresql.org/docs/current/sql-altertable.html

4. PostgreSQL Global Development Group. **PostgreSQL Documentation — Referential Constraints and Foreign Keys**  
   https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK

5. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

6. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

7. Date, C. J. *An Introduction to Database Systems*. Addison-Wesley.

8. Ensembl Documentation  
   https://www.ensembl.org/info/docs/index.html

9. GTEx Portal  
   https://gtexportal.org/

