# Data Manipulation Language (DML)
#### INSERT, SELECT, UPDATE, DELETE with PostgreSQL and Omics Examples

### Introduction: Why DML Matters in Real Systems

Once tables are created, the real value of a database begins when we **manipulate data**.

In clinical, genomics, and omics databases, data is **continuously arriving, queried, corrected, and sometimes removed**.

In SQL, this is handled using **Data Manipulation Language (DML):**

* `INSERT` → add new biological or clinical facts
* `SELECT` → retrieve and analyze those facts
* `UPDATE` → correct or refine existing facts
* `DELETE` → remove invalid or obsolete facts

PostgreSQL executes all DML inside a **transactional engine**, ensuring **data integrity and consistency**.

---

#### Assumed Tables

We will conceptually use these tables:
* `patients(patient_id, sex, age, diagnosis, enrollment_date)`
* `samples(sample_id, patient_id, tissue, collection_date)`
* `genes(gene_id, gene_symbol, chromosome)`
* `expression(sample_id, gene_id, expression_value)`

We do **not** need to create these now—they are used to explain logic.

---

## 1. INSERT — Adding Data

---

### 1.1 INSERT: Patient Registration
**Concept**

`INSERT` adds **new rows** into a table. Each row represents a **new real-world event or entity**.

**Real-Life Scenario**

A new patient is enrolled in a diabetes study.

**PostgreSQL Query**

```bash
INSERT INTO patients (patient_id, sex, age, diagnosis, enrollment_date)
VALUES (101, 'F', 52, 'Type 2 Diabetes', '2024-06-15');
```

#### How It Works (Step-by-Step)

* PostgreSQL checks:
    * Data types (age must be numeric)
    * Constraints (patient_id must be unique)

* If valid:
    * A new row is written to disk
    * The transaction is committed

**Why This Matters**
* Without INSERT, no new patients can exist
* Every clinical and omics pipeline starts with data insertion

#### Analogy:
Writing **a new patient file** and placing it into the hospital archive.

---

### 1.2 INSERT: Biological Sample Collection

**Scenario**

A blood sample is collected from patient 101.

**PostgreSQL Query**

```bash
INSERT INTO samples (sample_id, patient_id, tissue, collection_date)
VALUES ('SAMP_001', 101, 'Blood', '2024-06-20');
```

#### What PostgreSQL Enforces
* `patient_id = 101` must already exist (foreign key)
Prevents orphan samples

#### Why This Is Critical
* Maintains patient → **sample traceability**
* Essential for reproducible omics research

---


## 2. SELECT — Retrieving Data

---

### 2.1 SELECT: Viewing Patient Data
**Concept**

`SELECT` retrieves data **without modifying it**.

**Clinical Question**

“Show all patients with diabetes.”

**PostgreSQL Query**

```bash
SELECT patient_id, sex, age
FROM patients
WHERE diagnosis = 'Type 2 Diabetes';
```

#### How It Works
* PostgreSQL scans the table (or index)
* Applies the WHERE filter
* Returns matching rows as a **result set**

#### Why SELECT Is the Most Important SQL Command
Used in:

* Dashboards
* Reports
* Research analysis
* Machine learning pipelines

**Analogy:**

Filtering patient records in a register to see only relevant cases.

---

### 2.2 SELECT: Query (Gene Expression)

**Question**

“Which genes are highly expressed in blood samples?”

#### PostgreSQL Query

```bash
SELECT g.gene_symbol, e.expression_value
FROM expression e
JOIN genes g ON e.gene_id = g.gene_id
JOIN samples s ON e.sample_id = s.sample_id
WHERE s.tissue = 'Blood'
AND e.expression_value > 10;
```

#### What Is Happening Technically
* PostgreSQL performs **JOIN operations**
* Matches genes → expression → samples
* Filters biologically relevant rows

#### Why This Is Powerful
* Enables **biological insight using pure SQL**
* No Python or R needed at this stage

---

## 3. UPDATE — Modifying Existing Data

---

### 3.1 UPDATE: Correcting Clinical Information

**Scenario**
A patient was initially marked as “Pre-diabetes” but later confirmed as “Type 2 Diabetes”.

**PostgreSQL Query**

```bash
UPDATE patients
SET diagnosis = 'Type 2 Diabetes'
WHERE patient_id = 101;
```

#### How It Works
* PostgreSQL locates the matching row
* Updates only the specified column
* Preserves the row’s identity

#### Why UPDATE Is Essential
* Medical data evolves
* Corrections must not create duplicates

**Analogy**:
Correcting a diagnosis written incorrectly in a medical file.

---

### 3.2 UPDATE: Correcting Sample Annotation
**Scenario**

A tissue label was incorrect.

**PostgreSQL Query**

```bash
UPDATE samples
SET tissue = 'Plasma'
WHERE sample_id = 'SAMP_001';
```

#### Why This Matters in Omics
* Annotation errors can invalidate downstream GWAS or RNA-seq results
* UPDATE preserves sample lineage while fixing metadata

---

## 4. DELETE — Removing Data

---

### 4.1 DELETE: Patient Withdraws Consent

**Scenario**

A patient withdraws consent from the study.

**PostgreSQL Query**

```bash
DELETE FROM patients
WHERE patient_id = 101;
```

#### What PostgreSQL Checks
* Foreign key constraints
* Cascading rules (e.g., delete samples automatically or block deletion)

#### Why DELETE Is Critical
* Ethical compliance (GDPR, IRB rules)
* Prevents unauthorized data usage

**Analogy**:
Removing a patient’s file completely from the system.

---

### 4.2 DELETE: Failed Omics Experiment
**Scenario**
A sequencing run failed QC and must be excluded.

**PostgreSQL Query**

```bash
DELETE FROM samples
WHERE sample_id = 'SAMP_001';
```

#### Why This Is Important
* Prevents bad data from polluting analysis
* Maintains scientific validity

---

## 5. INSERT vs UPDATE vs DELETE 

| Operation | Purpose                | Example            |
| --------- | ---------------------- | ------------------ |
| INSERT    | Add new reality        | New patient        |
| SELECT    | Understand reality     | Analyze expression |
| UPDATE    | Correct reality        | Fix diagnosis      |
| DELETE    | Remove invalid reality | Withdraw consent   |

#### Why PostgreSQL Is Ideal for This

PostgreSQL provides:

* Strong ACID transactions
* Excellent JOIN optimization
* Full window function support
* Robust data integrity enforcement

This makes it ideal for:
* Clinical databases
* Omics metadata
* Large-scale analytics

---

#### Key Beginner Takeaway

If you remember one thing:

> **SQL DML is not about syntax — it is about managing real-world facts safely and correctly.**

---

#### Reference
* Date, C. J. An Introduction to Database Systems. Addison-Wesley.
* Silberschatz, A., Korth, H. F., Sudarshan, S. Database System Concepts. McGraw-Hill.
* ISO/IEC 9075:2023 – Database languages — SQL.
* PostgreSQL Official Documentation – Data Manipulation.
* Ensembl Documentation – Relational Schema Design.
* GTEx Portal – Metadata Structure and Best Practices.
* NCBI – Clinical and Genomic Data Modeling Guidelines.


```python

```
