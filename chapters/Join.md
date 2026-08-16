# Joins and Relationships
_**JOIN, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN, CROSS JOIN, SELF JOIN, Multi-Table Joins, and Biological Relationships**_

## Introduction: Why Joins Are One of the Most Important Concepts in SQL

A well-designed relational database usually does **not** keep all information in one giant table.

Instead, information is divided into multiple related tables.

For example, in a biomedical database we may have:

* a `patients table` containing patient information,
* a `samples table` containing biological samples,
* a `genes table` containing gene annotations,
* an `expression table` containing gene-expression measurements.

The information is deliberately separated because this reduces redundancy, improves data integrity, and supports **normalization**.

But eventually, we need to combine these tables to answer meaningful questions.

That is the purpose of a **JOIN**.

A JOIN allows SQL to combine rows from two or more tables using a **relationship between columns**.

In SQL, join conditions are commonly specified using `ON` or `USING`. The `ON` condition is a Boolean expression that determines which pairs of rows from the two tables are considered matches.

----

## 5.1 Why Do We Need Joins?

Imagine a hospital database.

The `patients` table contains:

| patient_id | sex | age | diagnosis |
| ---------: | --- | --: | --------- |
|        101 | F   |  52 | Diabetes  |
|        102 | M   |  45 | Healthy   |
|        103 | F   |  61 | Diabetes  |

The `samples` table contains:

| sample_id | patient_id | tissue |
| --------- | ---------: | ------ |
| S001      |        101 | Blood  |
| S002      |        101 | Liver  |
| S003      |        102 | Blood  |
| S004      |        103 | Brain  |

Suppose we ask:

> Which tissue samples belong to diabetic patients?

The diagnosis exists in:

```bash
patients
```

while the tissue information exists in:

```bash
samples
```

Neither table alone can answer the question.

We must connect them using:

```bash
patient_id
```

Conceptually:

```bash
patients.patient_id
        │
        │
        ▼
samples.patient_id
```

That relationship allows SQL to reconstruct the complete information.

---

## 5.2 The Relational Principle Behind Joins

The concept of joins comes directly from the **relational model**.

Tables represent different kinds of entities:

```bash
Patient
Sample
Gene
Transcript
Variant
Tissue
Experiment
```

Relationships connect those entities.

For example:

```bash
Patient
   │
   └────< Sample
            │
            └────< Expression
                       >──── Gene
```

The symbols can be interpreted as:

```bash
one patient → many samples


one sample → many expression measurements


one gene → many expression measurements
```

These are examples of **one-to-many relationships**.

A many-to-many relationship can be represented through an intermediate or **junction/association table**.

The `expression` table effectively creates a relationship between:

```bash
Sample ↔ Gene
```

because each sample contains measurements for many genes, and each gene is measured in many samples.

---

## 5.3 Primary Keys and Foreign Keys Revisited

Joins frequently use:

* **Primary Keys**
* **Foreign Keys**

Consider:

```bash
patients
--------
patient_id   PRIMARY KEY
```

and:

```bash
samples
-------
sample_id    PRIMARY KEY
patient_id   FOREIGN KEY
```

The foreign key in `samples` points to the primary key in `patients`.

Conceptually:

patients

```bash
patient_id
-----------
101
102
103
   ▲
   │
   │ Foreign Key reference
   │


samples


sample_id   patient_id
---------   ----------
S001        101
S002        101
S003        102
S004        103
```

This is known as **referential integrity**.

---

## 5.4 Our Example Biomedical Schema

We will use four simplified tables throughout this chapter.

`patients`

| patient_id | sex | age | diagnosis       |
| ---------: | --- | --: | --------------- |
|        101 | F   |  52 | Type 2 Diabetes |
|        102 | M   |  45 | Healthy         |
|        103 | F   |  61 | Type 2 Diabetes |
|        104 | M   |  38 | Hypertension    |


`samples`

| sample_id | patient_id | tissue |
| --------- | ---------: | ------ |
| S001      |        101 | Blood  |
| S002      |        101 | Liver  |
| S003      |        102 | Blood  |
| S004      |        103 | Brain  |
| S005      |        105 | Blood  |

Notice that `patient_id = 105` does not exist in the simplified patient table. In a properly enforced production schema, a foreign-key constraint would normally prevent this orphaned reference.

`genes`

| gene_id | gene_symbol | chromosome |
| ------- | ----------- | ---------- |
| ENSG001 | BRCA1       | 17         |
| ENSG002 | TP53        | 17         |
| ENSG003 | APOE        | 19         |
| ENSG004 | MYC         | 8          |

`expression`

| sample_id | gene_id | expression_value |
| --------- | ------- | ---------------: |
| S001      | ENSG001 |             12.5 |
| S001      | ENSG002 |             18.2 |
| S003      | ENSG001 |             10.4 |
| S003      | ENSG002 |             21.3 |
| S004      | ENSG003 |             35.8 |

----

## 5.5 Basic JOIN Syntax

The general PostgreSQL syntax is:

```bash
SELECT columns
FROM table1
JOIN table2
    ON table1.column = table2.column;
```

For example:

```bash
SELECT
    p.patient_id,
    p.diagnosis,
    s.sample_id,
    s.tissue
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Here:

```bash
p
```

and:

```bash
s
```
are **table aliases**.

----

Aliases make queries shorter and easier to read.

## 5.6 INNER JOIN
**Concept**

An `INNER JOIN` returns only rows where a valid match exists in **both tables**.

SQL treats `JOIN` without an explicit modifier as an inner join.

Therefore:

```bash
JOIN
```

and:

```bash
INNER JOIN
```

have the same meaning in this context.

----
Analogy

Imagine two invitation lists.

One contains people who registered for a conference.

The other contains people who paid.

An INNER JOIN asks:

> Show me only people who appear in both lists.

---

## 5.7 INNER JOIN: Patient and Sample Example
**Question**

> Show every patient together with their biological samples.

```bash
SELECT
    p.patient_id,
    p.sex,
    p.diagnosis,
    s.sample_id,
    s.tissue
FROM patients AS p
INNER JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Possible result:

| patient_id | sex | diagnosis       | sample_id | tissue |
| ---------: | --- | --------------- | --------- | ------ |
|        101 | F   | Type 2 Diabetes | S001      | Blood  |
|        101 | F   | Type 2 Diabetes | S002      | Liver  |
|        102 | M   | Healthy         | S003      | Blood  |
|        103 | F   | Type 2 Diabetes | S004      | Brain  |

Notice that patient 104 is missing.

Why?

Because patient **104 has no matching sample**.

Likewise, sample S005 does not appear because its patient ID has no matching patient row in our simplified example.

That is the central behavior of an INNER JOIN:

> Only matching rows survive.

----

## 5.8 Understanding the Join Condition

The important part is:

```bash
ON p.patient_id = s.patient_id
```

This tells SQL:

> Match a patient row with a sample row whenever their patient_id values are equal.

Conceptually:

```bash
patients                         samples


101   Diabetes      ───────────  S001   101   Blood
                    └─────────── S002   101   Liver


102   Healthy       ───────────  S003   102   Blood


103   Diabetes      ───────────  S004   103   Brain
```

Because patient 101 has two samples, the patient information appears twice in the result.

This is expected.

----

## 5.9 One-to-Many Relationships and Row Multiplication

This behavior often surprises beginners.

Suppose:

```bash
Patient 101
```

has three samples:

```bash
S001
S002
S003
```

```bash
Joining patient and sample tables produces three rows:

101 → S001
101 → S002
101 → S003
```

The database has not duplicated the patient record physically.

The **result set** contains multiple joined combinations because one patient relates to multiple samples.

This phenomenon is sometimes informally called **row multiplication** or **fan-out**.

Understanding it becomes essential when calculating counts and averages.

----

## 5.10 Omics Example: Sample to Gene Expression

Suppose we ask:

Show gene-expression measurements together with gene symbols.

```bash
SELECT
    e.sample_id,
    g.gene_symbol,
    e.expression_value
FROM expression AS e
INNER JOIN genes AS g
    ON e.gene_id = g.gene_id;
```

Possible result:

| sample_id | gene_symbol | expression_value |
| --------- | ----------- | ---------------: |
| S001      | BRCA1       |             12.5 |
| S001      | TP53        |             18.2 |
| S003      | BRCA1       |             10.4 |
| S003      | TP53        |             21.3 |
| S004      | APOE        |             35.8 |

The `expression` table stores IDs.

The `genes` table converts those IDs into biologically meaningful annotations.

This pattern is extremely common in genomics databases.

---

## 5.11 Multi-Table JOIN

Real analyses commonly require three or more tables.

Suppose we ask:

> Which genes are expressed in samples from patients with Type 2 Diabetes?

We need information from:

```bash
patients
samples
expression
genes
```

The relationships are:

```bash
patients.patient_id
        │
        ▼
samples.patient_id


samples.sample_id
        │
        ▼
expression.sample_id


expression.gene_id
        │
        ▼
genes.gene_id
```

The SQL query is:

```bash
SELECT
    p.patient_id,
    p.diagnosis,
    s.sample_id,
    s.tissue,
    g.gene_symbol,
    e.expression_value
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
JOIN expression AS e
    ON s.sample_id = e.sample_id
JOIN genes AS g
    ON e.gene_id = g.gene_id
WHERE p.diagnosis = 'Type 2 Diabetes';
```

This query creates a biological chain:

```bash
Patient
   ↓
Sample
   ↓
Expression measurement
   ↓
Gene
```

This is precisely why relational databases are valuable for multi-layer biological data.

---

## 5.12 INNER JOIN with Aggregation

Now we can combine Module 4 with Module 5.

**Question**

> How many samples does each diagnosis have?

```bash
SELECT
    p.diagnosis,
    COUNT(s.sample_id) AS sample_count
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
GROUP BY p.diagnosis;
```

This combines:

`bash
JOIN
+
GROUP BY
+
COUNT()
```

A very common analytical pattern.

---

## 5.13 LEFT JOIN
Concept

A LEFT JOIN, formally a LEFT OUTER JOIN, keeps:

every row from the left table,
matching rows from the right table,
and fills unmatched right-side columns with NULL.

PostgreSQL documents this behavior explicitly: unmatched rows from the left-hand table are retained, while the missing right-hand fields are represented by null values.

**Analogy**

Suppose you have:

```bash
All enrolled patients
```

and:

```bash
Patients who supplied samples
```

You want to see **every enrolled patient**, including those who never supplied a sample.

That is a LEFT JOIN.

----


## 5.14 LEFT JOIN Example

```bash
SELECT
    p.patient_id,
    p.diagnosis,
    s.sample_id,
    s.tissue
FROM patients AS p
LEFT JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Possible result:

| patient_id | diagnosis       | sample_id | tissue |
| ---------: | --------------- | --------- | ------ |
|        101 | Type 2 Diabetes | S001      | Blood  |
|        101 | Type 2 Diabetes | S002      | Liver  |
|        102 | Healthy         | S003      | Blood  |
|        103 | Type 2 Diabetes | S004      | Brain  |
|        104 | Hypertension    | NULL      | NULL   |

Patient 104 remains in the result.

Because there is no sample, PostgreSQL returns:

```bash
NULL
```

for sample-related columns.

---


## 5.15 Why LEFT JOIN Is Extremely Important

LEFT JOIN is essential when absence itself is meaningful.

Examples include:

* Patients without samples
* Genes without detected expression
* Variants without annotation
* Samples without QC records
* Subjects without follow-up
* Genes without pathway assignments
* Variants without known clinical significance

INNER JOIN would silently remove these records.

`LEFT JOIN` allows us to preserve them.

---

## 5.16 Finding Missing Relationships with LEFT JOIN

A very useful SQL pattern is:

```bash
LEFT JOIN
+
IS NULL
```

Suppose we want:

> Which patients have no biological samples?

```bash
SELECT
    p.patient_id,
    p.diagnosis
FROM patients AS p
LEFT JOIN samples AS s
    ON p.patient_id = s.patient_id
WHERE s.sample_id IS NULL;
```


Result:

```bash
104   Hypertension
```

```bash
Conceptually:

Keep all patients
      ↓
Attach matching samples
      ↓
Unmatched patients receive NULL
      ↓
WHERE sample_id IS NULL
      ↓
Patients with no sample
```

This pattern is often called an **anti-join pattern**.

---

## 5.17 Omics Anti-Join Example

Suppose you want:

> Find genes for which no expression measurements exist.

```bash
SELECT
    g.gene_id,
    g.gene_symbol
FROM genes AS g
LEFT JOIN expression AS e
    ON g.gene_id = e.gene_id
WHERE e.gene_id IS NULL;
```

This can help identify:

* missing measurements,
* unexpressed or unmeasured genes,
* annotation mismatches,
* incomplete imports

----

## 5.18 RIGHT JOIN
**Concept**

A `RIGHT JOIN` is the mirror image of a LEFT JOIN.

It preserves:

* every row from the right table,
* plus matching rows from the left table.

Example:

```bash
SELECT
    p.patient_id,
    s.sample_id,
    s.tissue
FROM patients AS p
RIGHT JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Here, every sample is retained.

If a sample has no matching patient, patient columns become `NULL`.

----

## 5.19 Do We Really Need RIGHT JOIN?

Technically, yes.

Practically, many SQL developers use LEFT JOIN more often because a RIGHT JOIN can usually be rewritten by switching table order.

For example:

```bash
FROM patients p
RIGHT JOIN samples s
```

can generally be written as:

```bash
FROM samples s
LEFT JOIN patients p
```

The latter is often easier to reason about because the **preserved table appears first**.

Still, understanding RIGHT JOIN is important.

----

## 5.20 FULL OUTER JOIN
**Concept**

A `FULL OUTER` JOIN preserves:

* matched rows,
* unmatched rows from the left table,
* unmatched rows from the right table.

Think of it as:

```bash
LEFT JOIN
+
RIGHT JOIN
```

conceptually combined.

**Example**

```bash
SELECT
    p.patient_id,
    s.sample_id,
    s.tissue
FROM patients AS p
FULL OUTER JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Possible result:

| patient_id | sample_id | tissue |
| ---------: | --------- | ------ |
|        101 | S001      | Blood  |
|        101 | S002      | Liver  |
|        102 | S003      | Blood  |
|        103 | S004      | Brain  |
|        104 | NULL      | NULL   |
|       NULL | S005      | Blood  |

This immediately reveals two types of mismatch:

```bash
Patient 104 → no sample


Sample S005 → no valid patient
```

----

## 5.21 Why FULL OUTER JOIN Is Useful

It is particularly valuable for:

* Data reconciliation
* ETL validation
* Comparing database releases
* Finding unmatched identifiers
* Auditing pipeline outputs
* Comparing annotation sources

For example, suppose you have variant lists from two annotation pipelines.

A FULL OUTER JOIN can identify:

```bash
variants present in both
variants only in pipeline A
variants only in pipeline B
```

That is extremely useful in bioinformatics data validation.

----

## 5.22 Visual Model of the Main Join Types

Suppose:

```bash
A = patients
B = samples
```

Conceptually:

```bash
INNER JOIN
A ∩ B


Only matches
```

```bash
LEFT JOIN
All A + matching B
```

```bash
RIGHT JOIN
All B + matching A
```

```bash
FULL OUTER JOIN
All A + All B
```

The critical question is:

> Which unmatched rows do I want to preserve?

That usually determines the appropriate join type.

----

## 5.23 CROSS JOIN
Concept

A `CROSS JOIN` generates the **Cartesian product** of two tables.

Every row from table A is paired with every row from table B.

Suppose:

```bash
patients = 4 rows
tissues  = 3 rows
```

Then:

```bash
4 × 3 = 12 combinations
```

SQL Query

```bash
SELECT
    p.patient_id,
    t.tissue_name
FROM patients AS p
CROSS JOIN tissues AS t;
```

If tissues are:

```bash
Blood
Liver
Brain
```

then every patient is paired with each tissue.

----

## 5.24 Why Would Anyone Use CROSS JOIN?

At first, CROSS JOIN may seem strange.

But it is useful when we intentionally need **all possible combinations.**

Examples:

* Every gene × every tissue
* Every patient × every scheduled visit
* Every treatment × every dose level
* Every chromosome × every population
* Every experimental condition × every time point

**Omics Example**

Suppose we want to create a matrix of:

```bash
Gene × Tissue
```

before attaching expression values.

```bash
SELECT
    g.gene_symbol,
    t.tissue_name
FROM genes AS g
CROSS JOIN tissues AS t;
```

This creates every theoretical gene-tissue combination.

It might later be LEFT JOINed to observed expression data.

This creates every theoretical gene-tissue combination.

----

## 5.25 The Danger of CROSS JOIN

CROSS JOIN can produce enormous result sets.

Suppose:

```bash
20,000 genes
×
50 tissues
```

That produces:

```bash
1,000,000 rows
```

Now suppose:

```bash
2,000,000 variants
×
20,000 genes
```

That would theoretically produce:

```bash
40 billion combinations
```
Therefore, Cartesian products must be used intentionally.

An accidental missing join condition can cause enormous intermediate results and severe performance problems.

----

## 5.26 SELF JOIN
**Concept**

A **self join** occurs when a table is joined to itself.

The same table appears twice with different aliases.

This is useful when rows within the same table have relationships with other rows.

Real-Life Example: Employee and Manager

Suppose:

```bash
employees
```
| employee_id | employee_name | manager_id |
| ----------: | ------------- | ---------: |
|           1 | Asha          |       NULL |
|           2 | Ravi          |          1 |
|           3 | Meera         |          1 |
|           4 | Arjun         |          2 |

Both employees and managers are stored in the same table.

We can write:

```bash
SELECT
    e.employee_name AS employee,
    m.employee_name AS manager
FROM employees AS e
LEFT JOIN employees AS m
    ON e.manager_id = m.employee_id;
```

The table is treated as two logical roles:

```bash
e → employee


m → manager
```

## 5.27 Omics Example of SELF JOIN

A self join can also be useful for biological relationships.

Suppose a table stores genes and genomic coordinates:

```bash
genes
-----
gene_id
gene_symbol
chromosome
start_position
end_position
```

We may ask:

> Which genes occur on the same chromosome?

A simplified query could be:

```bash
SELECT
    g1.gene_symbol AS gene_1,
    g2.gene_symbol AS gene_2,
    g1.chromosome
FROM genes AS g1
JOIN genes AS g2
    ON g1.chromosome = g2.chromosome
WHERE g1.gene_id < g2.gene_id;
```

The condition:

```bash
g1.gene_id < g2.gene_id
```

helps avoid:

```bash
BRCA1 ↔ BRCA1
```

and duplicate reverse pairs such as:

```bash
BRCA1 ↔ TP53
TP53  ↔ BRCA1
```

----

## 5.28 JOIN with USING

If two tables contain a join column with the same name, PostgreSQL supports USING.

Instead of:

```bash
SELECT *
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

we can write:

```bash
SELECT *
FROM patients AS p
JOIN samples AS s
    USING (patient_id);
```

USING tells SQL:

> Join these tables using the identically named patient_id column.

----

## 5.28 JOIN with USING

If two tables contain a join column with the same name, PostgreSQL supports `USING`.

Instead of:

```bash
SELECT *
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

we can write:

```bash
SELECT *
FROM patients AS p
JOIN samples AS s
    USING (patient_id);
```

`USING` tells PostgreSQL:

> Join these tables using the identically named patient_id column.

---

## 5.29 ON vs USING
`ON`

```bash
ON p.patient_id = s.patient_id
```

Advantages:

* Explicit
* Flexible
* Columns may have different names
* Supports complex Boolean conditions

`USING`

```bash
USING (patient_id)
```
Advantages:

* Shorter
* Cleaner when both columns have the same name
* Avoids duplicate output columns for the join key

For educational purposes, using `ON` initially is often preferable because it makes the relationship completely visible.

----

## 5.30 Joining on Multiple Columns

Sometimes one column alone does not uniquely define a relationship.

Suppose gene expression is identified by:

```bash
sample_id
gene_id
```

A related QC table may contain:

```bash
sample_id
gene_id
qc_status
```

We could join using both columns:

```bash
SELECT
    e.sample_id,
    e.gene_id,
    e.expression_value,
    q.qc_status
FROM expression AS e
JOIN expression_qc AS q
    ON e.sample_id = q.sample_id
   AND e.gene_id = q.gene_id;
```

This is a **composite join condition**.

----

## 5.31 Many-to-Many Relationships

Consider:

```bash
genes
```

and:

```bash
pathways
```

A gene can belong to many pathways.

A pathway contains many genes.

Therefore:

```bash
Gene ↔ Pathway
```

is a **many-to-many relationship**.

Relational databases generally represent this with an intermediate table:

```bash
genes
    │
    │
gene_pathways
    │
    │
pathways
```

Example:

```bash
gene_pathways
-------------
gene_id
pathway_id
```

**Query**

```bash
SELECT
    g.gene_symbol,
    p.pathway_name
FROM genes AS g
JOIN gene_pathways AS gp
    ON g.gene_id = gp.gene_id
JOIN pathways AS p
    ON gp.pathway_id = p.pathway_id;
```

This type of pattern is common in:

* Gene ontology
* Pathway databases
* Protein interaction networks
* Disease-gene relationships
* Drug-gene relationships

----

## 5.32 Ensembl-Like Relationships

Ensembl provides a useful real-world example of hierarchical genomic relationships. Ensembl annotation structures connect entities such as genes, transcripts, and exons; transcript-level entities are linked to genes, while exons are linked to transcripts.

Conceptually:

```bash
Gene
  │
  ├── Transcript 1
  │      ├── Exon 1
  │      ├── Exon 2
  │      └── Exon 3
  │
  └── Transcript 2
         ├── Exon 1
         └── Exon 4
```

This hierarchy is naturally suited to relational joins.

A simplified query could resemble:

```bash
SELECT
    g.gene_id,
    g.gene_symbol,
    t.transcript_id,
    e.exon_id
FROM genes AS g
JOIN transcripts AS t
    ON g.gene_id = t.gene_id
JOIN transcript_exons AS te
    ON t.transcript_id = te.transcript_id
JOIN exons AS e
    ON te.exon_id = e.exon_id;
```

This traverses:

```bash
Gene
→ Transcript
→ Exon
```

---

## 5.33 GTEx-Style Relationships

GTEx is designed around tissue-specific gene expression and regulation, and its public resources expose gene-expression information at tissue and sample levels.

A simplified GTEx-style relational model might look like:

```bash
Donor
  │
  └──── Sample
          │
          ├──── Tissue
          │
          └──── Expression
                   │
                   └──── Gene
```

We might create tables such as:

```bash
donors
samples
tissues
genes
expression
```

Then ask:

> What is the mean expression of each gene in each tissue?


```bash
SELECT
    g.gene_symbol,
    t.tissue_name,
    AVG(e.expression_value) AS mean_expression
FROM expression AS e
JOIN genes AS g
    ON e.gene_id = g.gene_id
JOIN samples AS s
    ON e.sample_id = s.sample_id
JOIN tissues AS t
    ON s.tissue_id = t.tissue_id
GROUP BY
    g.gene_symbol,
    t.tissue_name;
```

Now several modules work together:

```bash
JOIN
+
GROUP BY
+
AVG()
```

This is approaching genuine omics analytics.

---

## 5.34 Joining Tables and Filtering

The join determines **how tables are related**.

WHERE determines **which resulting rows remain**.

Example:

```bash
SELECT
    p.patient_id,
    s.sample_id,
    s.tissue
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
WHERE p.diagnosis = 'Type 2 Diabetes';
```

Conceptually:

```bash
patients
   +
samples
   ↓
JOIN
   ↓
combined rows
   ↓
WHERE diagnosis = Diabetes
   ↓
final rows
```

## 5.35 ON vs WHERE in Outer Joins

This becomes particularly important with outer joins.

Consider:

```bash
SELECT
    p.patient_id,
    s.sample_id
FROM patients AS p
LEFT JOIN samples AS s
    ON p.patient_id = s.patient_id
WHERE s.tissue = 'Blood';
```

Although this uses a LEFT JOIN, the WHERE condition removes rows where:

```bash
s.tissue = NULL
```

Therefore, patients with no samples disappear.

In many situations this effectively behaves like an INNER JOIN for that condition.

Compare:

```bash
SELECT
    p.patient_id,
    s.sample_id
FROM patients AS p
LEFT JOIN samples AS s
    ON p.patient_id = s.patient_id
   AND s.tissue = 'Blood';
```

Now every patient remains, but only blood samples are matched.

This distinction is technically important because PostgreSQL evaluates the join condition when deciding which rows match, while filtering conditions applied later can remove the null-extended rows produced by an outer join.

----

## 5.36 Example: All Patients and Their Blood Samples

Correct approach when we want to preserve every patient:

```bash
SELECT
    p.patient_id,
    p.diagnosis,
    s.sample_id
FROM patients AS p
LEFT JOIN samples AS s
    ON p.patient_id = s.patient_id
   AND s.tissue = 'Blood';
```

Possible output:

| patient_id | diagnosis    | sample_id |
| ---------: | ------------ | --------- |
|        101 | Diabetes     | S001      |
|        102 | Healthy      | S003      |
|        103 | Diabetes     | NULL      |
|        104 | Hypertension | NULL      |

This query answers:

> Show all patients, and attach a blood sample when one exists.

That is different from:

> Show only patients who have blood samples.


The second question would normally use an INNER JOIN.

----

## 5.37 Joining and COUNT(): A Common Trap

Suppose one patient has three samples.

If we join:

```bash
patients
JOIN samples
```

that patient appears three times.

Now consider:

```bash
SELECT COUNT(*)
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

This counts **joined rows**, which are samples in this relationship, not unique patients.

If we want the number of unique patients represented:

```bash
SELECT COUNT(DISTINCT p.patient_id)
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

This distinction is crucial in real analytics.

----

## 5.38 Omics Example: Avoiding Inflated Counts

Suppose:

```bash
one gene
```

has:

```bash
50 expression observations
```

After joining genes to expression:

```bash
gene row → 50 joined rows
```

If you simply run:

```bash
COUNT(*)
```

you are counting measurements, not genes.

To count unique genes:

```bash
COUNT(DISTINCT g.gene_id)
```

This prevents **join-induced overcounting**.

----

## 5.39 Join Cardinality

A useful technical concept is join cardinality.

Cardinality describes how rows in one table relate to rows in another.

Common relationship types include:

* One-to-one
* One-to-many
* Many-to-one
* Many-to-many

Examples:

```bash
Patient → Sample
1        many
```

```bash
Gene → Transcript
1      many
```

```bash
Gene ↔ Pathway
many   many
```

Understanding cardinality helps predict:

* how many rows a join will generate,
* whether aggregates may be inflated,
* whether duplicate-looking rows are expected.

----

## 5.40 Join Selectivity

Another important technical concept is **join selectivity**.

Join selectivity describes how restrictive a join condition is.

A highly selective join may match relatively few rows.

A poorly selective condition may match many combinations.

For example:

```bash
ON p.patient_id = s.patient_id
```

is usually highly meaningful.

But:

```bash
ON p.sex = s.sample_type
```
would likely be nonsensical and could generate incorrect combinations.

The database optimizer uses statistical information about column distributions to estimate join cardinality and select an appropriate execution strategy.

----

## 5.41 Join Algorithms: What PostgreSQL Does Internally

When you write:

```bash
JOIN
```

you describe the logical relationship.

You usually do not tell PostgreSQL exactly how to perform the join physically.

The query planner / optimizer decides.

Common physical join algorithms include:

#### Nested Loop Join

Conceptually:

```bash
For each row in table A
    search matching rows in table B
```

Can be efficient when:
* one input is small,
* an appropriate index exists.

#### Hash Join

SQL can create a **hash table** from one input and probe it using the other.

Often effective for:

* equality joins,
* larger unsorted datasets.

#### Merge Join

Both inputs are processed in join-key order and merged.

Can be efficient when:

* inputs are already sorted,
* indexes provide useful ordering,
* large datasets are being joined.

These algorithms are part of **physical query execution**, whereas `INNER JOIN`, `LEFT JOIN`, etc. describe the **logical semantics**.

We will study this more deeply in the query optimization module.

----

## 5.42 Indexes and Joins

Indexes can substantially improve join performance.

Suppose:

```bash
samples.patient_id
```

is frequently joined to:

```bash
patients.patient_id
```

An index on the foreign-key side may help PostgreSQL locate related rows efficiently.

Conceptually:

Without index:

```bash
Search many sample rows
```

With appropriate index:

```bash
Jump efficiently to matching patient_id entries
```

However, whether PostgreSQL actually uses an index depends on:

* table size,
* selectivity,
* statistics,
* expected number of rows,
* estimated I/O cost,
* query plan.

An index is therefore **not automatically faster for every query**.

---

## 5.43 JOIN vs Subquery

Sometimes the same logical question can be expressed using:

* a JOIN,
* a subquery,
* `EXISTS`.

For example:

> Find patients who have samples.

Using JOIN:

```bash
SELECT DISTINCT
    p.patient_id
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Later, we will learn an alternative:

```bash
SELECT
    p.patient_id
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

These queries express related logic but have different semantics and practical uses.

We will examine this properly **Subqueries**.

----

## 5.44 Common Join Mistakes
#### Mistake 1: Forgetting the Join Condition

Incorrect or dangerous:

```bash
SELECT *
FROM patients AS p
JOIN samples AS s
    ON TRUE;
```

This effectively produces a Cartesian product.

#### Mistake 2: Joining on the Wrong Columns

Wrong:

```bash
ON p.age = s.patient_id
```

A query can be syntactically valid but biologically meaningless.

SQL cannot determine whether your scientific relationship makes sense.

#### Mistake 3: Ignoring One-to-Many Multiplication

A patient with five samples generates five joined rows.

This can inflate:

```bash
COUNT
SUM
AVG
```

if the analyst misunderstands the unit of observation.

#### Mistake 4: Using INNER JOIN When Missing Data Matter

INNER JOIN removes unmatched rows.

If you are studying missingness, use a suitable outer join.

#### Mistake 5: Filtering a LEFT JOIN Incorrectly

This:

```bash
LEFT JOIN ...
WHERE right_table.column = ...
```

may eliminate the unmatched rows you intended to preserve.

#### Mistake 6: Using SELECT * in Large Multi-Table Joins

For example:

```bash
SELECT *
FROM patients
JOIN samples ...
JOIN expression ...
JOIN genes ...;
```

may return:

* duplicate-named columns,
* unnecessary fields,
* very wide datasets,
* excessive memory/network usage.

Prefer explicitly selected columns.

---

## 5.45 A Practical Join Decision Framework

When deciding which JOIN to use, ask:

#### Question 1
**Do I want only records that match on both sides?**

Use:

```bash
INNER JOIN
```

#### Question 2
**Do I want every record from my main table, even if no match exists?**

Use:

```bash
LEFT JOIN
```

#### Question 3
Do I specifically want every record from the right-side table?

Use:

```bash
RIGHT JOIN
```

although reordering tables and using LEFT JOIN is often clearer.

#### Question 4

**Do I need matched and unmatched records from both sides?**

Use:

```bash
FULL OUTER JOIN
```

#### Question 5

**Do I intentionally need every possible pair?**

Use:

```bash
CROSS JOIN
```

#### Question 6

**Am I comparing records inside the same table?**

Use:

```bash
SELF JOIN
```

---

## 5.46 Complete Omics Example

Suppose our goal is:

> Among patients with Type 2 Diabetes, calculate the mean expression of each gene in each tissue.

We need:

```bash
patients
samples
expression
genes
```

Query:

```bash
SELECT
    s.tissue,
    g.gene_symbol,
    AVG(e.expression_value) AS mean_expression
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
JOIN expression AS e
    ON s.sample_id = e.sample_id
JOIN genes AS g
    ON e.gene_id = g.gene_id
WHERE p.diagnosis = 'Type 2 Diabetes'
GROUP BY
    s.tissue,
    g.gene_symbol
ORDER BY
    s.tissue,
    mean_expression DESC;
```

This one query combines concepts from several modules:

```bash
FROM
JOIN
ON
WHERE
GROUP BY
AVG()
ORDER BY
```

Biologically, it answers:

> Within the selected disease cohort, what is the average expression of each gene within each tissue?

This is already structurally similar to the types of questions we could explore using GTEx-like data, which is designed to support research into tissue-specific human gene expression and regulation.

----

## 5.47 The Bigger Biological Picture

Relational joins are particularly well suited to omics because modern biological information is naturally interconnected.

For example:

```bash
Participant
     ↓
Sample
     ↓
Tissue
     ↓
Expression
     ↓
Gene
     ↓
Transcript
     ↓
Protein
     ↓
Pathway
     ↓
Disease
```

No single table should necessarily contain all of this information.

Instead, SQL lets us construct the relevant biological view when needed.

That is one of the core strengths of the relational model.

----

## 5.48 Module 5 Summary

In this module, we learned that:

* A **JOIN** combines related rows from multiple tables.
* `INNER JOIN` returns matching records from both sides.
* `LEFT JOIN` preserves all rows from the left table.
* `RIGHT JOIN` preserves all rows from the right table.
* `FULL OUTER JOIN` preserves matched and unmatched rows from both tables.
* `CROSS JOIN` creates the Cartesian product.
* A `SELF JOIN` joins a table to itself.
* `ON` specifies the join predicate.
* `USING` provides concise syntax when join columns have identical names.
* Multiple joins can reconstruct complex relationships such as **Patient → Sample → Expression → Gene**.
* One-to-many and many-to-many relationships can cause **row multiplication**.
* `COUNT(DISTINCT ...)` is often needed to prevent analytical overcounting after joins.
* `LEFT JOIN` combined with `IS NULL` is useful for identifying missing relationships.
* Filtering in `ON` versus `WHERE` can change the semantics of an outer join.
* **SQL** may physically implement joins using **nested-loop, hash, or merge join algorithms**.
* Correct join logic requires understanding not only SQL syntax, but also the **cardinality and biological meaning of the underlying relationships**.

----

### Key Takeaway

> A relational database deliberately separates information to maintain structure and integrity. JOIN is the mechanism that reconstructs those relationships when we need to answer a question.

In an omics context, this means SQL can connect a biological chain such as:

```bash
Patient
   ↓
Sample
   ↓
Tissue
   ↓
Expression
   ↓
Gene
```

without forcing all of that information into one enormous, redundant table.

----

#### References
* PostgreSQL Global Development Group. PostgreSQL Documentation — SELECT and Joined Tables. PostgreSQL defines the semantics of inner and outer joins, including how unmatched rows in outer joins are represented using null values.
* PostgreSQL Global Development Group. PostgreSQL Documentation — Table Expressions and Join Conditions. Join conditions can be expressed using ON, USING, and related constructs.
* Ensembl. Ensembl Gene Annotation and Transcript Documentation. Ensembl genomic annotations organize biological entities such as genes, transcripts, and exons in linked structures.
* GTEx Consortium. GTEx Portal. GTEx provides a public resource for studying tissue-specific human gene expression and regulation.
* GTEx Portal API Documentation. Gene-expression resources include normalized expression information that can be queried at tissue and sample levels.
* Silberschatz, A., Korth, H. F., & Sudarshan, S. Database System Concepts. McGraw-Hill.
* Elmasri, R., & Navathe, S. B. Fundamentals of Database Systems. Pearson.
* Date, C. J. An Introduction to Database Systems. Addison-Wesley.
