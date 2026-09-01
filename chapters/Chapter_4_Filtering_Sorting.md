# Filtering, Sorting, and Limiting Data
#### WHERE, Comparison Operators, Logical Operators, BETWEEN, IN, LIKE, IS NULL, ORDER BY, LIMIT, and OFFSET in PostgreSQL

---

## Introduction: Why Filtering Matters

Real databases contain far more information than we usually need for a single analysis.

In biomedical and omics settings, a database may contain:

- Thousands or millions of patients
- Large numbers of biological samples
- Tens of thousands of genes
- Millions of variants
- Billions of expression measurements

Returning every row every time would be inefficient and analytically meaningless.

SQL therefore provides tools to:

- Filter rows
- Combine conditions
- Search ranges
- Match patterns
- Handle missing values
- Sort results
- Restrict output size

These concepts are foundational because almost every realistic SQL query uses some form of filtering or sorting.


## 3.1 Dataset Used in This Module

We will use a simplified biomedical schema.

### `patients`

| patient_id | sex | age | diagnosis | enrollment_date |
|---:|---|---:|---|---|
| 101 | F | 52 | Type 2 Diabetes | 2025-01-15 |
| 102 | M | 45 | Healthy | 2025-02-03 |
| 103 | F | 61 | Type 2 Diabetes | 2025-03-11 |
| 104 | M | 38 | Hypertension | 2025-03-20 |
| 105 | F | 47 | Healthy | 2025-04-08 |
| 106 | M | 55 | Type 2 Diabetes | 2025-04-19 |

### `samples`

| sample_id | patient_id | tissue | collection_date |
|---|---:|---|---|
| S001 | 101 | Blood | 2025-01-20 |
| S002 | 101 | Liver | 2025-01-21 |
| S003 | 102 | Blood | 2025-02-05 |
| S004 | 103 | Brain | 2025-03-15 |
| S005 | 104 | Blood | 2025-03-25 |
| S006 | 105 | Liver | 2025-04-10 |
| S007 | 106 | Blood | 2025-04-21 |
| S008 | 106 | Liver | 2025-04-22 |

### `genes`

| gene_id | gene_symbol | chromosome |
|---|---|---|
| ENSG001 | BRCA1 | 17 |
| ENSG002 | TP53 | 17 |
| ENSG003 | APOE | 19 |
| ENSG004 | MYC | 8 |
| ENSG005 | BRCA2 | 13 |

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


## 3.2 WHERE — Filtering Rows

The `WHERE` clause filters rows according to a condition.

Conceptually:

```text
Table
  ↓
Evaluate condition for each row
  ↓
Keep rows where condition is TRUE
```

`WHERE` is one of the most frequently used clauses in SQL.

### Clinical Example

Question:

> Show patients diagnosed with Type 2 Diabetes.



```python
SELECT
    patient_id,
    sex,
    age,
    diagnosis
FROM patients
WHERE diagnosis = 'Type 2 Diabetes';
```

The expression:

```sql
diagnosis = 'Type 2 Diabetes'
```

is called a **predicate** or **Boolean condition**.

Rows for which the predicate evaluates to `TRUE` are returned.


### Omics Example

Question:

> Show all blood samples.



```python
SELECT
    sample_id,
    patient_id,
    tissue,
    collection_date
FROM samples
WHERE tissue = 'Blood';
```

Filtering is critical in biological analysis because many questions are defined by subsets such as:

- A specific tissue
- A specific disease
- A chromosome
- A population
- A sequencing platform
- A quality-control status


## 3.3 Comparison Operators

Comparison operators compare values.

Common PostgreSQL operators include:

| Operator | Meaning |
|---|---|
| `=` | Equal to |
| `<>` | Not equal to |
| `!=` | Not equal to |
| `>` | Greater than |
| `<` | Less than |
| `>=` | Greater than or equal to |
| `<=` | Less than or equal to |

These operators produce Boolean results.


### Example: Patients Older Than 50



```python
SELECT
    patient_id,
    age
FROM patients
WHERE age > 50;
```

### Example: Patients Not Classified as Healthy



```python
SELECT
    patient_id,
    diagnosis
FROM patients
WHERE diagnosis <> 'Healthy';
```

### Omics Example: Expression Greater Than a Threshold



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value > 20;
```

In omics work, comparison operators are commonly used for:

- Expression thresholds
- Allele-frequency thresholds
- p-value thresholds
- Read-depth thresholds
- Sample call-rate thresholds
- Quality-control metrics


## 3.4 Logical Operators

Logical operators combine or modify Boolean conditions.

The main operators are:

- `AND`
- `OR`
- `NOT`

These make it possible to define precise cohorts and analytical subsets.


## 3.5 AND — All Conditions Must Be True

`AND` returns rows only when every connected condition is true.

### Clinical Example

Question:

> Find female patients older than 50.



```python
SELECT
    patient_id,
    sex,
    age
FROM patients
WHERE sex = 'F'
  AND age > 50;
```

A row survives only if:

```text
sex = F
AND
age > 50
```

are both true.


### Omics Example

Question:

> Find expression values above 15 for gene ENSG002.



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE gene_id = 'ENSG002'
  AND expression_value > 15;
```

## 3.6 OR — At Least One Condition Must Be True

`OR` returns a row if at least one condition is true.

### Clinical Example



```python
SELECT
    patient_id,
    diagnosis
FROM patients
WHERE diagnosis = 'Type 2 Diabetes'
   OR diagnosis = 'Hypertension';
```

This returns patients belonging to either diagnostic group.


### Omics Example



```python
SELECT
    sample_id,
    tissue
FROM samples
WHERE tissue = 'Blood'
   OR tissue = 'Brain';
```

## 3.7 NOT — Negating a Condition

`NOT` reverses the truth value of a condition.

Example:



```python
SELECT
    patient_id,
    diagnosis
FROM patients
WHERE NOT diagnosis = 'Healthy';
```

A clearer equivalent is often:



```python
SELECT
    patient_id,
    diagnosis
FROM patients
WHERE diagnosis <> 'Healthy';
```

`NOT` becomes especially useful with constructs such as:

```sql
NOT IN
NOT BETWEEN
NOT LIKE
```


## 3.8 Operator Precedence

When multiple logical operators appear in one condition, evaluation order matters.

A simplified precedence rule is:

```text
NOT
AND
OR
```

Consider:



```python
SELECT *
FROM patients
WHERE sex = 'F'
   OR sex = 'M'
  AND age > 50;
```

Because `AND` is evaluated before `OR`, this does **not** necessarily mean:

> Male or female patients older than 50.

Use parentheses to make intent explicit:



```python
SELECT *
FROM patients
WHERE (sex = 'F' OR sex = 'M')
  AND age > 50;
```

### Best Practice

When mixing `AND` and `OR`, use parentheses even when you understand precedence.

This improves readability and reduces logical errors.


## 3.9 BETWEEN — Filtering Inclusive Ranges

`BETWEEN` tests whether a value lies within an inclusive range.

General syntax:

```sql
column BETWEEN lower_bound AND upper_bound
```

This includes both boundaries.

### Clinical Example



```python
SELECT
    patient_id,
    age
FROM patients
WHERE age BETWEEN 40 AND 60;
```

This is equivalent to:



```python
SELECT
    patient_id,
    age
FROM patients
WHERE age >= 40
  AND age <= 60;
```

### Omics Example



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value BETWEEN 10 AND 20;
```

Typical uses include:

- Age ranges
- Genomic coordinate windows
- Dates
- Expression intervals
- Laboratory measurement ranges


## 3.10 NOT BETWEEN



```python
SELECT
    patient_id,
    age
FROM patients
WHERE age NOT BETWEEN 40 AND 60;
```

This returns patients outside the inclusive range.


## 3.11 IN — Matching a List of Values

`IN` checks whether a value belongs to a specified list.

Instead of writing:



```python
SELECT *
FROM samples
WHERE tissue = 'Blood'
   OR tissue = 'Liver'
   OR tissue = 'Brain';
```

we can write:



```python
SELECT *
FROM samples
WHERE tissue IN ('Blood', 'Liver', 'Brain');
```

This is shorter and easier to maintain.

### Omics Use Cases

`IN` is especially useful for:

- Selected chromosomes
- Selected tissues
- Gene panels
- Sample IDs
- Variant IDs
- Disease categories


### Gene Example



```python
SELECT
    gene_id,
    gene_symbol
FROM genes
WHERE gene_symbol IN ('BRCA1', 'BRCA2', 'TP53');
```

## 3.12 NOT IN



```python
SELECT
    gene_id,
    gene_symbol
FROM genes
WHERE gene_symbol NOT IN ('BRCA1', 'BRCA2');
```

This excludes specified genes.

Later, when `IN` or `NOT IN` contains a subquery, `NULL` behavior becomes especially important.


## 3.13 LIKE — Pattern Matching

`LIKE` performs pattern matching on text.

Two wildcard characters are important:

| Wildcard | Meaning |
|---|---|
| `%` | Zero or more characters |
| `_` | Exactly one character |

### Example: Gene Symbols Beginning with BRCA



```python
SELECT
    gene_id,
    gene_symbol
FROM genes
WHERE gene_symbol LIKE 'BRCA%';
```

This can match:

```text
BRCA1
BRCA2
BRCA...
```


### Example: Ends With a Pattern



```python
SELECT
    gene_symbol
FROM genes
WHERE gene_symbol LIKE '%1';
```

This returns symbols ending in `1`.


### Example: Single-Character Wildcard

```text
BRCA_
```

would match strings such as:

```text
BRCA1
BRCA2
```

where exactly one character follows `BRCA`.



```python
SELECT
    gene_symbol
FROM genes
WHERE gene_symbol LIKE 'BRCA_';
```

## 3.14 ILIKE — PostgreSQL Case-Insensitive Pattern Matching

PostgreSQL provides `ILIKE`, which performs case-insensitive pattern matching.

Example:



```python
SELECT
    gene_symbol
FROM genes
WHERE gene_symbol ILIKE 'brca%';
```

This can match values such as:

```text
BRCA1
BRCA2
brca1
Brca2
```

depending on the stored data.

`ILIKE` is PostgreSQL-specific and is not part of portable standard SQL.


## 3.15 NOT LIKE and NOT ILIKE



```python
SELECT
    gene_symbol
FROM genes
WHERE gene_symbol NOT LIKE 'BRCA%';
```

or:



```python
SELECT
    gene_symbol
FROM genes
WHERE gene_symbol NOT ILIKE 'brca%';
```

These exclude matching patterns.


## 3.16 NULL and Missing Data

`NULL` represents the absence of a known value.

It is not the same as:

- `0`
- an empty string
- `FALSE`

This is especially important in clinical and omics data, where missingness may have scientific meaning.


## 3.17 Why `= NULL` Is Wrong

This is incorrect:



```python
SELECT *
FROM patients
WHERE diagnosis = NULL;
```

SQL does not treat `NULL` as an ordinary value.

Use:

```sql
IS NULL
```

instead.



```python
SELECT *
FROM patients
WHERE diagnosis IS NULL;
```

To identify non-missing values:



```python
SELECT *
FROM patients
WHERE diagnosis IS NOT NULL;
```

### Omics Example



```python
SELECT
    sample_id,
    gene_id
FROM expression
WHERE expression_value IS NULL;
```

This can identify missing measurements rather than true biological zeros.


## 3.18 Three-Valued Logic

SQL conditions can evaluate to:

```text
TRUE
FALSE
UNKNOWN
```

`UNKNOWN` usually arises when `NULL` participates in comparisons.

For example:

```sql
NULL = 5
```

does not evaluate to `TRUE` or `FALSE`; it evaluates to `UNKNOWN`.

Rows are returned by `WHERE` only when the condition is `TRUE`.

This explains many apparent surprises involving missing data.


## 3.19 Filtering Dates

PostgreSQL supports proper date comparisons.

### Example: Patients Enrolled After March 1, 2025



```python
SELECT
    patient_id,
    enrollment_date
FROM patients
WHERE enrollment_date > DATE '2025-03-01';
```

Using explicit date literals makes the intended type clear.


### Date Range Example



```python
SELECT
    sample_id,
    collection_date
FROM samples
WHERE collection_date BETWEEN DATE '2025-03-01'
                          AND DATE '2025-03-31';
```

This is useful for:

- Recruitment windows
- Sample collection periods
- Follow-up intervals
- Experimental batches


## 3.20 ORDER BY — Sorting Results

SQL tables have no guaranteed natural presentation order.

If order matters, specify it using `ORDER BY`.

General form:

```sql
ORDER BY column ASC
```

or:

```sql
ORDER BY column DESC
```

`ASC` means ascending.

`DESC` means descending.


### Clinical Example: Youngest to Oldest



```python
SELECT
    patient_id,
    age
FROM patients
ORDER BY age ASC;
```

Because `ASC` is the default, this is equivalent to:



```python
SELECT
    patient_id,
    age
FROM patients
ORDER BY age;
```

### Oldest to Youngest



```python
SELECT
    patient_id,
    age
FROM patients
ORDER BY age DESC;
```

## 3.21 Omics Example: Highest Expression First



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
ORDER BY expression_value DESC;
```

This is useful for ranking biological signals.


## 3.22 Sorting by Multiple Columns

PostgreSQL can sort by more than one column.

Example:



```python
SELECT
    patient_id,
    diagnosis,
    age
FROM patients
ORDER BY diagnosis ASC, age DESC;
```

Conceptually:

1. Sort rows by diagnosis.
2. Within each diagnosis, sort by age descending.

This is common in reports.


### Omics Example



```python
SELECT
    gene_id,
    sample_id,
    expression_value
FROM expression
ORDER BY gene_id ASC, expression_value DESC;
```

This groups output visually by gene and ranks observations within each gene.


## 3.23 Sorting by an Alias

Aliases defined in `SELECT` can often be used in `ORDER BY`.

Example:



```python
SELECT
    patient_id,
    age,
    age * 12 AS age_in_months
FROM patients
ORDER BY age_in_months DESC;
```

This improves readability when sorting calculated expressions.


## 3.24 NULL Sorting in PostgreSQL

PostgreSQL provides explicit control over where nulls appear:

```sql
NULLS FIRST
NULLS LAST
```

Example:



```python
SELECT
    patient_id,
    diagnosis
FROM patients
ORDER BY diagnosis ASC NULLS LAST;
```

This is particularly useful in biomedical tables containing missing annotations.


## 3.25 LIMIT — Restricting the Number of Rows

`LIMIT` restricts how many rows PostgreSQL returns.

Example:



```python
SELECT *
FROM patients
LIMIT 5;
```

This is useful for:

- Previewing a table
- Debugging
- Top-N analyses
- Reducing output during development

However, without `ORDER BY`, the selected rows are not guaranteed to represent any meaningful ordering.


## 3.26 Top-N Queries

To find the oldest three patients:



```python
SELECT
    patient_id,
    age
FROM patients
ORDER BY age DESC
LIMIT 3;
```

### Omics Example: Top Five Expression Measurements



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
ORDER BY expression_value DESC
LIMIT 5;
```

The combination:

```text
ORDER BY + LIMIT
```

is the standard pattern for top-N queries.


## 3.27 OFFSET — Skipping Rows

`OFFSET` tells PostgreSQL to skip a number of rows before returning results.

Example:



```python
SELECT
    patient_id,
    age
FROM patients
ORDER BY age DESC
LIMIT 3
OFFSET 3;
```

Conceptually:

```text
Sort rows
↓
Skip first 3
↓
Return next 3
```

This is commonly used in pagination.


## 3.28 LIMIT and OFFSET for Pagination

A user interface might display 10 rows at a time.

Page 1:



```python
SELECT *
FROM patients
ORDER BY patient_id
LIMIT 10
OFFSET 0;
```

Page 2:



```python
SELECT *
FROM patients
ORDER BY patient_id
LIMIT 10
OFFSET 10;
```

Page 3:



```python
SELECT *
FROM patients
ORDER BY patient_id
LIMIT 10
OFFSET 20;
```

For very large datasets, high-offset pagination can become inefficient. More advanced approaches such as keyset pagination may be preferable.


## 3.29 DISTINCT — Removing Duplicate Result Rows

Although `DISTINCT` is conceptually part of `SELECT`, it is highly relevant when filtering and presenting results.

Example:



```python
SELECT DISTINCT tissue
FROM samples;
```

Possible result:

```text
Blood
Liver
Brain
```

Without `DISTINCT`, repeated tissue values would appear multiple times.


### Omics Example



```python
SELECT DISTINCT gene_id
FROM expression;
```

This returns unique genes represented in the expression table.


## 3.30 DISTINCT with Multiple Columns



```python
SELECT DISTINCT
    patient_id,
    tissue
FROM samples;
```

PostgreSQL considers the complete combination.

These rows are different:

```text
101 | Blood
101 | Liver
```

even though `patient_id` is the same.


## 3.31 Combining WHERE, ORDER BY, and LIMIT

### Clinical Example

Question:

> Show the three oldest diabetic patients.



```python
SELECT
    patient_id,
    age,
    diagnosis
FROM patients
WHERE diagnosis = 'Type 2 Diabetes'
ORDER BY age DESC
LIMIT 3;
```

Conceptually:

```text
patients
   ↓
WHERE diagnosis = Diabetes
   ↓
eligible rows
   ↓
ORDER BY age DESC
   ↓
LIMIT 3
```


## 3.32 Omics Example: Top Expressed Genes in Blood

Using joins from the next module conceptually:



```python
SELECT
    g.gene_symbol,
    e.expression_value
FROM expression AS e
JOIN genes AS g
    ON e.gene_id = g.gene_id
JOIN samples AS s
    ON e.sample_id = s.sample_id
WHERE s.tissue = 'Blood'
ORDER BY e.expression_value DESC
LIMIT 10;
```

This query combines:

- Filtering
- Joins
- Sorting
- Limiting

It asks:

> Among blood samples, what are the ten highest expression measurements?


## 3.33 PostgreSQL Regular Expressions: A Preview

For more advanced string matching, PostgreSQL supports regular-expression operators such as:

```text
~
~*
!~
!~*
```

Example:



```python
SELECT
    gene_symbol
FROM genes
WHERE gene_symbol ~ '^BRCA[0-9]+$';
```

This pattern means approximately:

> Starts with `BRCA`, followed by one or more digits, and then ends.

Regular expressions are more powerful than `LIKE`, but they are also more complex.


## 3.34 Case-Sensitive vs Case-Insensitive Matching

In PostgreSQL:

```sql
LIKE
```

is normally case-sensitive.

```sql
ILIKE
```

is case-insensitive.

Example:



```python
SELECT gene_symbol
FROM genes
WHERE gene_symbol ILIKE 'tp%';
```

This can match values such as `TP53`.


## 3.35 Filtering Using Expressions

`WHERE` conditions can contain calculations.

Example:



```python
SELECT
    patient_id,
    age
FROM patients
WHERE age * 12 > 600;
```

Since 600 months is 50 years, this effectively filters patients older than 50.

Although valid, simpler conditions such as `age > 50` are preferable when available.


## 3.36 Filtering with Functions

PostgreSQL functions can appear inside filtering conditions.

Example:



```python
SELECT
    gene_symbol
FROM genes
WHERE LENGTH(gene_symbol) > 4;
```

Another example:



```python
SELECT
    gene_symbol
FROM genes
WHERE LOWER(gene_symbol) = 'tp53';
```

Function-based filtering is useful, although applying functions to indexed columns can affect index usage depending on query and index design.


## 3.37 Query Processing Order

A simplified logical processing order is:

```text
FROM
  ↓
JOIN / ON
  ↓
WHERE
  ↓
GROUP BY
  ↓
HAVING
  ↓
SELECT
  ↓
DISTINCT
  ↓
ORDER BY
  ↓
LIMIT / OFFSET
```

This is a logical model, not necessarily PostgreSQL's exact physical execution plan.

Understanding it explains why some aliases are available in `ORDER BY` but not necessarily in earlier clauses such as `WHERE`.


## 3.38 Common Beginner Mistake: Using an Alias in WHERE

Problematic:



```python
SELECT
    patient_id,
    age * 12 AS age_in_months
FROM patients
WHERE age_in_months > 600;
```

`WHERE` is logically evaluated before the `SELECT` alias is created.

Use:



```python
SELECT
    patient_id,
    age * 12 AS age_in_months
FROM patients
WHERE age * 12 > 600;
```

or restructure the query later using a subquery or CTE when appropriate.


## 3.39 Common Beginner Mistakes

### Mistake 1 — Forgetting Quotes Around Text

Incorrect:

```sql
WHERE diagnosis = Diabetes
```

Correct:

```sql
WHERE diagnosis = 'Diabetes'
```

### Mistake 2 — Using `= NULL`

Incorrect:

```sql
WHERE tissue = NULL
```

Correct:

```sql
WHERE tissue IS NULL
```

### Mistake 3 — Misunderstanding BETWEEN

`BETWEEN 40 AND 60` includes both 40 and 60.

### Mistake 4 — Mixing AND and OR Without Parentheses

Use explicit parentheses to make logic clear.

### Mistake 5 — Using LIMIT Without ORDER BY for Top-N Analysis

This:

```sql
LIMIT 10
```

does not mean “top 10” unless an ordering criterion is defined.

### Mistake 6 — Confusing LIKE Wildcards

```text
% → any number of characters
_ → exactly one character
```

### Mistake 7 — Treating NULL as Zero

Missing expression is not automatically expression equal to zero.


## 3.40 Filtering in Biomedical Research

Filtering logic directly corresponds to cohort definition.

For example:

```text
Female
AND
Age > 50
AND
Type 2 Diabetes
```

is a clinical inclusion rule.

SQL:



```python
SELECT *
FROM patients
WHERE sex = 'F'
  AND age > 50
  AND diagnosis = 'Type 2 Diabetes';
```

This is essentially a reproducible **cohort-selection specification**.


## 3.41 Omics Cohort Example

Suppose we want blood samples collected after March 1, 2025:



```python
SELECT
    sample_id,
    patient_id,
    tissue,
    collection_date
FROM samples
WHERE tissue = 'Blood'
  AND collection_date >= DATE '2025-03-01'
ORDER BY collection_date;
```

This type of filtering can define samples entering downstream:

- RNA-seq analysis
- Genotyping
- Proteomics
- QC workflows


## 3.42 Gene-Panel Example



```python
SELECT
    gene_id,
    gene_symbol,
    chromosome
FROM genes
WHERE gene_symbol IN ('BRCA1', 'BRCA2', 'TP53', 'APOE')
ORDER BY chromosome, gene_symbol;
```

This resembles selecting a targeted set of genes for reporting or validation.


## 3.43 Expression Threshold Example



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value >= 10
  AND expression_value <= 25
ORDER BY expression_value DESC;
```

Equivalent:



```python
SELECT
    sample_id,
    gene_id,
    expression_value
FROM expression
WHERE expression_value BETWEEN 10 AND 25
ORDER BY expression_value DESC;
```

Both represent a bounded analytical interval.


## 3.44 Choosing the Right Filtering Construct

Use:

- `=` for exact equality
- `<>` or `!=` for inequality
- `>`, `<`, `>=`, `<=` for numeric/date comparisons
- `AND` when all conditions must hold
- `OR` when at least one condition may hold
- `NOT` to negate a condition
- `BETWEEN` for inclusive ranges
- `IN` for membership in a short list
- `LIKE` for simple text patterns
- `ILIKE` for case-insensitive PostgreSQL pattern matching
- `IS NULL` for missing values
- `ORDER BY` when output order matters
- `LIMIT` to restrict result count
- `OFFSET` to skip rows


# Module 3 Summary

In this module, we learned how PostgreSQL narrows and organizes query results.

Key concepts include:

- `WHERE` filters rows
- Comparison operators create Boolean predicates
- `AND`, `OR`, and `NOT` combine logical conditions
- Parentheses help control and clarify logical evaluation
- `BETWEEN` handles inclusive ranges
- `IN` handles membership in a list
- `LIKE` and `ILIKE` support pattern matching
- `%` matches zero or more characters
- `_` matches exactly one character
- `IS NULL` and `IS NOT NULL` handle missing values correctly
- SQL uses three-valued logic involving `TRUE`, `FALSE`, and `UNKNOWN`
- `ORDER BY` controls result ordering
- Multiple columns can be used for hierarchical sorting
- PostgreSQL supports `NULLS FIRST` and `NULLS LAST`
- `LIMIT` restricts the number of returned rows
- `OFFSET` skips rows and supports pagination
- `DISTINCT` removes duplicate result rows
- Filtering logic is directly applicable to clinical cohort selection and omics data extraction


# Key Takeaway

> **Filtering determines which rows enter an analysis, sorting determines how they are presented, and limiting determines how much of the result is returned.**

For biomedical and omics data, these are not merely formatting operations.

They define analytical populations such as:

```text
Patients with a specific diagnosis
Samples from a specific tissue
Genes belonging to a selected panel
Measurements above a QC threshold
Records collected within a defined time window
```

Mastering these operations is essential before moving to aggregation, joins, subqueries, and advanced SQL analytics.


# References

1. PostgreSQL Global Development Group. **PostgreSQL Documentation — SELECT**  
   https://www.postgresql.org/docs/current/sql-select.html

2. PostgreSQL Global Development Group. **PostgreSQL Documentation — Value Expressions**  
   https://www.postgresql.org/docs/current/sql-expressions.html

3. PostgreSQL Global Development Group. **PostgreSQL Documentation — Pattern Matching**  
   https://www.postgresql.org/docs/current/functions-matching.html

4. PostgreSQL Global Development Group. **PostgreSQL Documentation — Comparison Functions and Operators**  
   https://www.postgresql.org/docs/current/functions-comparison.html

5. PostgreSQL Global Development Group. **PostgreSQL Documentation — Date/Time Types**  
   https://www.postgresql.org/docs/current/datatype-datetime.html

6. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

7. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

8. Date, C. J. *An Introduction to Database Systems*. Addison-Wesley.

9. Ensembl Documentation  
   https://www.ensembl.org/info/docs/index.html

10. GTEx Portal  
    https://gtexportal.org/

