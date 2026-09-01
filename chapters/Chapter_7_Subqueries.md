# Subqueries
### Scalar Subqueries, Single-Row and Multi-Row Subqueries, IN, EXISTS, NOT EXISTS, Correlated Subqueries, Nested Queries, and Omics Applications in PostgreSQL

---

## Introduction: Why Do We Need Subqueries?

Until now, most SQL questions have used information that can be obtained directly from tables.

For example:

> Which patients are older than 50?

```sql
SELECT patient_id, age
FROM patients
WHERE age > 50;
```

The threshold `50` is already known.

But real analytical questions are often more complex:

> Which patients are older than the **average age of all patients**?

Now we do not know the threshold beforehand.

PostgreSQL must first calculate:

```sql
SELECT AVG(age)
FROM patients;
```

Suppose this returns:

```text
49.67
```

Then PostgreSQL can conceptually execute:

```sql
SELECT patient_id, age
FROM patients
WHERE age > 49.67;
```

SQL allows us to combine these operations:

```sql
SELECT patient_id, age
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

The query inside parentheses is called a **subquery**.


## 6.1 What Is a Subquery?

A **subquery**, also called a **nested query** or **inner query**, is a SQL query embedded inside another SQL statement.

The surrounding query is commonly called the:

- **Outer query**
- **Main query**

Example:



```python
SELECT patient_id, age
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

Here:

```sql
SELECT AVG(age)
FROM patients;
```

is the **inner query**, while the surrounding `SELECT` statement is the **outer query**.

### Layman Analogy

Suppose someone asks:

> Which students scored above the class average?

You cannot answer immediately.

First:

> What is the class average?

Then:

> Which students scored above that value?

That is essentially how a subquery works.

```text
Question 1
   ↓
Calculate intermediate information
   ↓
Use that information
   ↓
Answer Question 2
```

Subqueries therefore allow SQL to solve a **question inside another question**.


## 6.2 Why Are Subqueries Important?

Subqueries are useful when one part of a query depends on the result of another query.

They help answer questions such as:

- Which patients are older than the cohort average?
- Which genes have expression above the global mean?
- Which patients have biological samples?
- Which genes occur in a particular pathway?
- Which samples have expression measurements?
- Which patients have no samples?
- Which genes have expression above their tissue-specific average?
- Which samples belong to patients meeting another clinical criterion?

Subqueries are particularly useful in **data science and omics** because biological questions are frequently hierarchical.

For example:

```text
Which genes
    ↓
have high expression
    ↓
in samples
    ↓
obtained from patients
    ↓
with a particular disease?
```


## 6.3 Dataset Used in This Chapter

We will continue with our simplified biomedical database.

### `patients`

| patient_id | sex | age | diagnosis |
|---:|---|---:|---|
| 101 | F | 52 | Type 2 Diabetes |
| 102 | M | 45 | Healthy |
| 103 | F | 61 | Type 2 Diabetes |
| 104 | M | 38 | Hypertension |
| 105 | F | 47 | Healthy |
| 106 | M | 55 | Type 2 Diabetes |

### `samples`

| sample_id | patient_id | tissue |
|---|---:|---|
| S001 | 101 | Blood |
| S002 | 101 | Liver |
| S003 | 102 | Blood |
| S004 | 103 | Brain |
| S005 | 104 | Blood |
| S006 | 105 | Liver |
| S007 | 106 | Blood |
| S008 | 106 | Liver |

### `genes`

| gene_id | gene_symbol | chromosome |
|---|---|---|
| ENSG001 | BRCA1 | 17 |
| ENSG002 | TP53 | 17 |
| ENSG003 | APOE | 19 |
| ENSG004 | MYC | 8 |

### `expression`

| sample_id | gene_id | expression_value |
|---|---|---:|
| S001 | ENSG001 | 12.5 |
| S001 | ENSG002 | 18.2 |
| S003 | ENSG001 | 10.4 |
| S003 | ENSG002 | 21.3 |
| S004 | ENSG003 | 35.8 |
| S006 | ENSG004 | 25.6 |
| S007 | ENSG001 | 14.1 |
| S007 | ENSG002 | 19.7 |


## 6.4 Basic Subquery Structure

The general pattern is:

```sql
SELECT column_name
FROM table_name
WHERE column_name operator (
    SELECT ...
);
```

For example:



```python
SELECT patient_id, age
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

Conceptually:

```text
Inner Query
     ↓
AVG(age)
     ↓
49.67
     ↓
Outer Query
     ↓
age > 49.67
```


## 6.5 Scalar Subqueries

A **scalar subquery** returns exactly:

- one row
- one column

In other words, it returns a **single value**.

Examples include:

```text
49.67
35.8
101
```

### Clinical Example: Patients Older Than Average



```python
SELECT
    patient_id,
    sex,
    age,
    diagnosis
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

The important point is that we did **not hard-code the average**.

If the patient population changes, PostgreSQL recalculates it automatically.

### Why Dynamic Thresholds Matter

Compare:

```sql
WHERE age > 50;
```

with:

```sql
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

The first uses a **literal constant**.

The second uses a **data-driven threshold**.

This makes the query:

- Dynamic
- Reproducible
- Adaptable to changing data


## 6.6 Omics Example: Expression Above the Global Mean



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value > (
    SELECT AVG(expression_value)
    FROM expression
);
```

PostgreSQL conceptually performs:

```text
All expression measurements
          ↓
    Calculate AVG()
          ↓
     Global average
          ↓
Compare each measurement
          ↓
Return measurements > average
```

This is a simple form of **data-driven biological filtering**.


## 6.7 Scalar Subquery in SELECT



```python
SELECT
    patient_id,
    age,
    (
        SELECT AVG(age)
        FROM patients
    ) AS cohort_average_age
FROM patients;
```

This gives every patient alongside the cohort average.

We can also calculate the difference from the mean:



```python
SELECT
    patient_id,
    age,
    (
        SELECT AVG(age)
        FROM patients
    ) AS cohort_average_age,
    age - (
        SELECT AVG(age)
        FROM patients
    ) AS difference_from_average
FROM patients;
```

Later, **window functions** will provide a more elegant way to perform this type of analysis.


## 6.8 Single-Row Subqueries

A **single-row subquery** returns one row.

It is commonly used with operators such as:

```text
=
>
<
>=
<=
<>
```

### Example: Find the Oldest Patient



```python
SELECT
    patient_id,
    age,
    diagnosis
FROM patients
WHERE age = (
    SELECT MAX(age)
    FROM patients
);
```

The inner query returns the maximum age. The outer query then returns every patient whose age equals that maximum.

This differs from:

```sql
ORDER BY age DESC
LIMIT 1
```

because the subquery approach returns **all ties** for the maximum value.


## 6.9 Multi-Row Subqueries

A **multi-row subquery** returns multiple rows.

For example:

```sql
SELECT patient_id
FROM samples;
```

may return several patient IDs.

Because the inner query returns multiple values, a scalar operator such as `=` is usually inappropriate.

For set membership, use operators such as:

```text
IN
EXISTS
ANY
ALL
```


## 6.10 IN with a Subquery

### Clinical Example: Which Patients Have at Least One Sample?



```python
SELECT
    patient_id,
    sex,
    age,
    diagnosis
FROM patients
WHERE patient_id IN (
    SELECT patient_id
    FROM samples
);
```

Conceptually:

```text
Subquery
   ↓
patient IDs represented in samples
   ↓
Outer query
   ↓
Return patients whose IDs belong to this set
```


### Omics Example: Genes with Expression Measurements



```python
SELECT
    gene_id,
    gene_symbol,
    chromosome
FROM genes
WHERE gene_id IN (
    SELECT gene_id
    FROM expression
);
```

This retrieves only genes represented in the expression dataset.


## 6.11 NOT IN

`NOT IN` asks whether a value does **not belong to a set**.



```python
SELECT
    gene_id,
    gene_symbol
FROM genes
WHERE gene_id NOT IN (
    SELECT gene_id
    FROM expression
);
```

This can identify genes absent from the expression table.

However, `NOT IN` has an important complication involving `NULL`, discussed later in this chapter.


## 6.12 Subqueries in WHERE

Subqueries are very commonly placed inside `WHERE`.

They can be combined with:

- Comparison operators
- `IN`
- `NOT IN`
- `EXISTS`
- `NOT EXISTS`
- `ANY`
- `ALL`

For example:



```python
SELECT *
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```


```python
SELECT *
FROM genes
WHERE gene_id IN (
    SELECT gene_id
    FROM expression
);
```

The inner query supplies information used to decide **which outer rows should remain**.


## 6.13 Subqueries in FROM: Derived Tables

A subquery can appear inside `FROM`.

When used this way, its result behaves like a temporary table for the duration of the query.

Example:



```python
SELECT
    gene_summary.gene_id,
    gene_summary.mean_expression
FROM (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
) AS gene_summary
WHERE gene_summary.mean_expression > 15;
```

The inner query creates an intermediate result:

```text
gene_id     mean_expression
-------     ---------------
ENSG001     ...
ENSG002     ...
ENSG003     ...
```

The outer query then filters that result.

A subquery in `FROM` is commonly called a **derived table**.

In PostgreSQL, such a subquery should be given an alias such as:

```sql
AS gene_summary
```


## 6.14 Joining to a Subquery



```python
SELECT
    g.gene_symbol,
    x.mean_expression
FROM genes AS g
JOIN (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
) AS x
    ON g.gene_id = x.gene_id;
```

Conceptually:

```text
expression
    ↓
AVG per gene
    ↓
Derived table
    ↓
JOIN genes
    ↓
Gene symbol + average expression
```


## 6.15 EXISTS — Testing Whether Related Data Exist

`EXISTS` asks:

> Does the subquery return **at least one row**?

The logical result is either `TRUE` or `FALSE`.

### Clinical Example



```python
SELECT
    p.patient_id,
    p.diagnosis
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

Why `SELECT 1`?

Because `EXISTS` does not care about the returned value. It only cares whether a qualifying row exists.

Conceptually:

```text
For each patient:
    Does at least one matching sample exist?
        YES → keep patient
        NO  → discard patient
```


## 6.16 Correlated Subqueries

The previous query contains:

```sql
WHERE s.patient_id = p.patient_id
```

The inner query refers to a value from the outer query.

This is called a **correlated subquery**.

### Definition

A correlated subquery depends on the current row of the outer query.

Conceptually:

```text
Outer row
   ↓
Evaluate inner relationship using that row
   ↓
Return TRUE/FALSE or a calculated result
```

This differs from an **uncorrelated subquery**, which can be evaluated independently of the outer row.

The PostgreSQL optimizer may internally transform the query, so this row-by-row view is a useful logical model rather than a guarantee of the physical execution plan.


### Uncorrelated Example



```python
SELECT *
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

### Correlated Example



```python
SELECT
    p.patient_id
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

The second inner query depends on `p.patient_id` from the outer query.


## 6.17 Omics Example Using EXISTS

### Question

Which genes have at least one expression value greater than 20?



```python
SELECT
    g.gene_id,
    g.gene_symbol
FROM genes AS g
WHERE EXISTS (
    SELECT 1
    FROM expression AS e
    WHERE e.gene_id = g.gene_id
      AND e.expression_value > 20
);
```

This asks:

> Does this gene have **at least one high-expression observation**?

It does not ask how many observations exist; it only tests existence.


## 6.18 NOT EXISTS

`NOT EXISTS` asks whether the subquery returns **no rows**.

### Clinical Example: Patients Without Samples



```python
SELECT
    p.patient_id,
    p.diagnosis
FROM patients AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

### Omics Example: Genes Without Expression Records



```python
SELECT
    g.gene_id,
    g.gene_symbol
FROM genes AS g
WHERE NOT EXISTS (
    SELECT 1
    FROM expression AS e
    WHERE e.gene_id = g.gene_id
);
```

This pattern is useful for identifying:

- Missing measurements
- Incomplete imports
- Genes absent from an assay
- Missing annotation relationships


## 6.19 NOT EXISTS vs LEFT JOIN ... IS NULL

The following patterns can express similar logic.

### LEFT JOIN approach



```python
SELECT
    p.patient_id
FROM patients AS p
LEFT JOIN samples AS s
    ON p.patient_id = s.patient_id
WHERE s.sample_id IS NULL;
```

### NOT EXISTS approach



```python
SELECT
    p.patient_id
FROM patients AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

Both can answer:

> Which patients have no samples?

`NOT EXISTS` often expresses the intent particularly clearly.


## 6.20 The NULL Problem with NOT IN

Suppose the subquery returns:

```text
101
102
103
NULL
```

Then a `NOT IN` condition can produce unexpected results because SQL uses **three-valued logic**:

```text
TRUE
FALSE
UNKNOWN
```

Comparisons involving `NULL` can become `UNKNOWN`.

For anti-membership tests, `NOT EXISTS` is often safer.



```python
SELECT
    p.patient_id
FROM patients AS p
WHERE NOT EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

Alternatively, if appropriate, explicitly remove nulls from the subquery:



```python
SELECT patient_id
FROM patients
WHERE patient_id NOT IN (
    SELECT patient_id
    FROM samples
    WHERE patient_id IS NOT NULL
);
```

## 6.21 ANY and SOME

PostgreSQL supports comparison against a subquery using `ANY` or its synonym `SOME`.

A condition such as:

```sql
expression_value > ANY (...)
```

means:

> Is this value greater than **at least one** value returned by the subquery?



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value > ANY (
    SELECT expression_value
    FROM expression
    WHERE gene_id = 'ENSG001'
);
```

## 6.22 ALL

`ALL` requires the comparison to hold against **every value** returned by the subquery.



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value > ALL (
    SELECT expression_value
    FROM expression
    WHERE gene_id = 'ENSG001'
);
```

Think of them this way:

```text
> ANY
```

means:

> Greater than at least one member of the set.

Whereas:

```text
> ALL
```

means:

> Greater than every member of the set.


## 6.23 Correlated Subquery with Aggregation

### Question

Which expression measurements are above the average expression for their own gene?



```python
SELECT
    e1.sample_id,
    e1.gene_id,
    e1.expression_value
FROM expression AS e1
WHERE e1.expression_value > (
    SELECT AVG(e2.expression_value)
    FROM expression AS e2
    WHERE e2.gene_id = e1.gene_id
);
```

This is a **correlated scalar subquery**.

Conceptually:

```text
Current expression row
        ↓
Which gene?
        ↓
Calculate average for that gene
        ↓
Compare current value with gene average
        ↓
Keep if above average
```

This is much more sophisticated than applying a fixed threshold.


## 6.24 Clinical Correlated Subquery Example

Suppose the patient table contains a `hospital` column.

Question:

> Which patients are older than the average patient age at their own hospital?



```python
SELECT
    p1.patient_id,
    p1.hospital,
    p1.age
FROM patients AS p1
WHERE p1.age > (
    SELECT AVG(p2.age)
    FROM patients AS p2
    WHERE p2.hospital = p1.hospital
);
```

This performs a **within-group comparison**.

Similar logic appears throughout biomedical research:

- Expression relative to tissue mean
- Biomarker relative to disease-group mean
- Coverage relative to sequencing-batch mean
- Allele frequency relative to population distribution


## 6.25 Nested Subqueries

A subquery can itself contain another subquery.

Example:

> Find genes expressed in samples belonging to patients with Type 2 Diabetes.



```python
SELECT
    gene_id,
    gene_symbol
FROM genes
WHERE gene_id IN (
    SELECT gene_id
    FROM expression
    WHERE sample_id IN (
        SELECT sample_id
        FROM samples
        WHERE patient_id IN (
            SELECT patient_id
            FROM patients
            WHERE diagnosis = 'Type 2 Diabetes'
        )
    )
);
```

Conceptually:

```text
Find diabetic patients
        ↓
Find their samples
        ↓
Find genes measured in those samples
        ↓
Return gene information
```

Deep nesting is possible, but excessive nesting can reduce readability.


## 6.26 JOIN vs Deeply Nested Subqueries

The previous question can also be expressed using joins:



```python
SELECT DISTINCT
    g.gene_id,
    g.gene_symbol
FROM genes AS g
JOIN expression AS e
    ON g.gene_id = e.gene_id
JOIN samples AS s
    ON e.sample_id = s.sample_id
JOIN patients AS p
    ON s.patient_id = p.patient_id
WHERE p.diagnosis = 'Type 2 Diabetes';
```

This does not mean JOIN is always better.

Choose the SQL structure that most clearly expresses the analytical intention.


## 6.27 JOIN vs Subquery: Existence Example

Question:

> Which patients have samples?

### JOIN approach



```python
SELECT DISTINCT
    p.patient_id,
    p.diagnosis
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

### EXISTS approach



```python
SELECT
    p.patient_id,
    p.diagnosis
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

The JOIN says:

> Combine patients with matching sample rows.

`EXISTS` says:

> Return the patient if at least one matching sample exists.

For pure existence tests, `EXISTS` often communicates the intent more directly.


## 6.28 JOINs Can Cause Row Multiplication

Suppose one patient has five samples.

A JOIN may return that patient five times.

If the question is only:

> Does this patient have at least one sample?

then `EXISTS` naturally avoids producing multiple logical matches in the output.

This is one reason `EXISTS` is valuable.


## 6.29 Subquery vs CTE

A derived-table subquery can often be rewritten using a **Common Table Expression (CTE)**.

Subquery form:



```python
SELECT *
FROM (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
) AS gene_summary
WHERE mean_expression > 15;
```

CTE form:



```python
WITH gene_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM gene_summary
WHERE mean_expression > 15;
```

CTEs can improve readability in complex analytical workflows and will be covered properly later.


## 6.30 Subqueries with Aggregation and HAVING

### Question

Which genes have an average expression greater than the overall average expression?



```python
SELECT
    gene_id,
    AVG(expression_value) AS gene_mean_expression
FROM expression
GROUP BY gene_id
HAVING AVG(expression_value) > (
    SELECT AVG(expression_value)
    FROM expression
);
```

This combines:

```text
GROUP BY
+
AVG()
+
HAVING
+
Subquery
```

The outer query calculates a mean per gene.

The inner query calculates the global mean.

The two are then compared.


## 6.31 Subquery with Multiple Conditions

### Question

Find patients older than the cohort average who also have a blood sample.



```python
SELECT
    p.patient_id,
    p.age,
    p.diagnosis
FROM patients AS p
WHERE p.age > (
    SELECT AVG(age)
    FROM patients
)
AND EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
      AND s.tissue = 'Blood'
);
```

The first subquery asks:

> Is the patient older than average?

The second asks:

> Does the patient have a blood sample?

Only patients satisfying both conditions are returned.


## 6.32 Realistic Omics Example

Question:

> Find genes whose mean expression in blood samples is greater than the overall mean expression across all blood-sample measurements.



```python
SELECT
    e.gene_id,
    AVG(e.expression_value) AS gene_mean_expression
FROM expression AS e
JOIN samples AS s
    ON e.sample_id = s.sample_id
WHERE s.tissue = 'Blood'
GROUP BY e.gene_id
HAVING AVG(e.expression_value) > (
    SELECT AVG(e2.expression_value)
    FROM expression AS e2
    JOIN samples AS s2
        ON e2.sample_id = s2.sample_id
    WHERE s2.tissue = 'Blood'
);
```

This performs two levels of analysis:

```text
All blood expression measurements
            ↓
         Global AVG

Blood expression grouped by gene
            ↓
       AVG per gene

Gene-specific mean > Global blood mean
```


## 6.33 Subquery Returning Multiple Columns

PostgreSQL supports row-wise comparisons.

For example:



```python
SELECT *
FROM expression
WHERE (gene_id, sample_id) IN (
    SELECT gene_id, sample_id
    FROM validated_expression
);
```

This asks:

> Does this exact gene–sample combination appear in the validated dataset?

This is useful when the logical key consists of multiple columns.


## 6.34 Performance Considerations

Subqueries are not inherently slow.

PostgreSQL has a sophisticated **query planner and optimizer**.

Performance depends on factors such as:

- Table size
- Indexes
- Cardinality
- Selectivity
- Data distribution
- Correlation
- Join strategy
- Query structure
- PostgreSQL statistics
- Available memory

Therefore:

> JOIN is always faster than subquery

is incorrect.

Similarly:

> Subqueries are always faster

is also incorrect.

Use:



```python
EXPLAIN
SELECT *
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```


```python
EXPLAIN ANALYZE
SELECT *
FROM patients
WHERE age > (
    SELECT AVG(age)
    FROM patients
);
```

These tools help inspect the actual execution plan and will be covered in the performance optimization module.


## 6.35 Common Beginner Mistakes

### Mistake 1 — Using `=` with a Multi-Row Subquery

Problematic:

```sql
WHERE patient_id = (
    SELECT patient_id
    FROM samples
);
```

If multiple rows are returned, PostgreSQL raises an error.

Use `IN`, `EXISTS`, or another appropriate set operator.

### Mistake 2 — Forgetting the Correlation Condition

Problematic:



```python
SELECT
    p.patient_id
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
);
```

If any sample exists anywhere in the table, the condition is true for every patient.

Correct:



```python
SELECT
    p.patient_id
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
);
```

### Mistake 3 — Using NOT IN Without Considering NULL

Potentially dangerous:

```sql
WHERE patient_id NOT IN (
    SELECT patient_id
    FROM samples
);
```

Prefer `NOT EXISTS` when appropriate.

### Mistake 4 — Unnecessary Deep Nesting

Very deeply nested queries can become difficult to:

- Read
- Debug
- Maintain

Consider JOINs, CTEs, or views when they improve clarity.

### Mistake 5 — Assuming Written Order Equals Physical Execution Order

SQL is **declarative**.

PostgreSQL's optimizer decides the physical execution plan.

The inner-query-first model is useful conceptually, but not always a literal description of execution.


## 6.36 Choosing Between IN, EXISTS, JOIN, and NOT EXISTS

| Question | Common Choice |
|---|---|
| Is a value in this set? | `IN` |
| Does at least one related row exist? | `EXISTS` |
| Does no related row exist? | `NOT EXISTS` |
| Do I need columns from both tables? | `JOIN` |
| Do I need to preserve unmatched rows? | Outer `JOIN` |
| Do I need a single calculated value? | Scalar subquery |
| Do I need an intermediate result table? | Derived table / CTE |

These are guidelines, not absolute rules.


## 6.37 Building the Subquery Mental Model

When facing a complex question, break it into smaller questions.

Suppose the biological question is:

> Which genes have above-average expression in blood samples from patients with Type 2 Diabetes?

Break it down:

1. Which patients have Type 2 Diabetes?
2. Which blood samples belong to those patients?
3. Which expression measurements belong to those samples?
4. What average should be calculated?
5. Which genes exceed that threshold?

This decomposition is one of the most useful problem-solving skills in SQL.


## 6.38 Subqueries in Real Omics Workflows

### Genomics

Find variants located in genes on chromosome 6:



```python
SELECT *
FROM variants
WHERE gene_id IN (
    SELECT gene_id
    FROM genes
    WHERE chromosome = '6'
);
```

### Transcriptomics

Find genes whose mean expression exceeds the overall expression mean:



```python
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id
HAVING AVG(expression_value) > (
    SELECT AVG(expression_value)
    FROM expression
);
```

### Clinical Genomics

Find patients who have at least one DNA sample:



```python
SELECT
    p.patient_id
FROM patients AS p
WHERE EXISTS (
    SELECT 1
    FROM samples AS s
    WHERE s.patient_id = p.patient_id
      AND s.sample_type = 'DNA'
);
```

### Quality Control

Find samples with no QC record:



```python
SELECT
    s.sample_id
FROM samples AS s
WHERE NOT EXISTS (
    SELECT 1
    FROM sample_qc AS q
    WHERE q.sample_id = s.sample_id
);
```

### Pathway Analysis

Find genes belonging to pathways related to DNA repair:



```python
SELECT
    g.gene_id,
    g.gene_symbol
FROM genes AS g
WHERE g.gene_id IN (
    SELECT gp.gene_id
    FROM gene_pathways AS gp
    WHERE gp.pathway_id IN (
        SELECT p.pathway_id
        FROM pathways AS p
        WHERE p.pathway_name ILIKE '%DNA repair%'
    )
);
```

PostgreSQL's `ILIKE` performs case-insensitive pattern matching.


## 6.39 From Basic SQL to Analytical SQL

### Basic



```python
SELECT *
FROM expression;
```

### Filtering



```python
SELECT *
FROM expression
WHERE expression_value > 20;
```

### Aggregation



```python
SELECT
    gene_id,
    AVG(expression_value)
FROM expression
GROUP BY gene_id;
```

### Subquery



```python
SELECT
    gene_id,
    AVG(expression_value)
FROM expression
GROUP BY gene_id
HAVING AVG(expression_value) > (
    SELECT AVG(expression_value)
    FROM expression
);
```

### Correlated Subquery



```python
SELECT
    e1.sample_id,
    e1.gene_id,
    e1.expression_value
FROM expression AS e1
WHERE e1.expression_value > (
    SELECT AVG(e2.expression_value)
    FROM expression AS e2
    WHERE e2.gene_id = e1.gene_id
);
```

This progression shows how SQL moves from simple retrieval toward **context-aware analytical reasoning**.


## 6.40 Subqueries vs Window Functions: Preview

Consider:

> Show expression observations above the average for their gene.

A correlated subquery can solve this:



```python
SELECT
    e1.sample_id,
    e1.gene_id,
    e1.expression_value
FROM expression AS e1
WHERE e1.expression_value > (
    SELECT AVG(e2.expression_value)
    FROM expression AS e2
    WHERE e2.gene_id = e1.gene_id
);
```

Later, window functions will allow:

```sql
AVG(expression_value)
OVER (PARTITION BY gene_id)
```

This can calculate group-level statistics while keeping the individual rows.

Window functions are not replacements for all subqueries, but they provide a powerful alternative for many analytical tasks.


# Module 6 Summary

In this module, we learned that a **subquery is a query embedded inside another SQL statement**.

The major concepts are:

- **Scalar subquery** — returns one value
- **Single-row subquery** — returns one row
- **Multi-row subquery** — returns multiple rows
- Subqueries can appear in `WHERE`, `SELECT`, and `FROM`
- A subquery in `FROM` creates a **derived table**
- `IN` tests membership in a result set
- `EXISTS` determines whether at least one qualifying row exists
- `NOT EXISTS` determines whether no qualifying row exists
- `ANY`/`SOME` compares against at least one returned value
- `ALL` compares against every returned value
- A **correlated subquery** references values from the outer query
- Nested subqueries allow complex questions to be decomposed
- `NOT IN` requires caution when `NULL` values are possible
- Subqueries can be combined with `JOIN`, `GROUP BY`, `HAVING`, and aggregate functions
- JOIN and subquery approaches may solve similar problems but express different intentions
- Subqueries are not inherently slower than joins
- Deeply nested queries should be rewritten with JOINs or CTEs when doing so improves clarity


# Key Takeaway

> **A subquery allows the answer to one SQL question to become part of another SQL question.**

For omics analysis, this allows us to move from simple questions such as:

```text
Which genes have expression > 20?
```

to context-dependent questions such as:

```text
Which genes have expression
above their own average
in blood samples
from patients
with a particular disease?
```

That ability to build **hierarchical, data-dependent questions** is what makes subqueries an essential bridge between basic SQL and advanced analytical SQL.


# References

1. PostgreSQL Global Development Group. **PostgreSQL Documentation — Subquery Expressions**  
   https://www.postgresql.org/docs/current/functions-subquery.html

2. PostgreSQL Global Development Group. **PostgreSQL Documentation — Table Expressions**  
   https://www.postgresql.org/docs/current/queries-table-expressions.html

3. PostgreSQL Global Development Group. **PostgreSQL Documentation — Scalar Subqueries**  
   https://www.postgresql.org/docs/current/sql-expressions.html#SQL-SYNTAX-SCALAR-SUBQUERIES

4. PostgreSQL Global Development Group. **PostgreSQL Documentation — SELECT**  
   https://www.postgresql.org/docs/current/sql-select.html

5. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

6. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

7. Date, C. J. *An Introduction to Database Systems*. Addison-Wesley.

8. Ensembl Documentation  
   https://www.ensembl.org/info/docs/index.html

9. GTEx Portal  
   https://gtexportal.org/

