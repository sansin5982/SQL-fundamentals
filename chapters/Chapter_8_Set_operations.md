# Set Operations
### UNION, UNION ALL, INTERSECT, and EXCEPT in PostgreSQL

---

## Introduction: Why Set Operations Matter

In relational databases, the result of a `SELECT` query can be treated as a **set of rows**.  
Very often, we need to compare or combine the results of two separate queries.

Examples include:

- Combining patients from two hospitals
- Combining genes detected in two tissues
- Finding genes shared between two experiments
- Finding variants unique to one cohort
- Comparing sample lists before and after quality control
- Identifying overlap between disease-associated gene sets

SQL provides **set operations** for exactly these tasks.

The major set operators in PostgreSQL are:

- `UNION`
- `UNION ALL`
- `INTERSECT`
- `EXCEPT`

These operations work vertically: they combine or compare **rows from query results** rather than columns from related tables.

That makes set operations fundamentally different from `JOIN`.

---

## 7.1 The Basic Idea

Suppose Query A returns:

| gene_symbol |
|---|
| BRCA1 |
| TP53 |
| APOE |

and Query B returns:

| gene_symbol |
|---|
| TP53 |
| APOE |
| MYC |

Using set operations, we can ask:

- Combine both lists
- Keep duplicates
- Find common genes
- Find genes unique to one list

Conceptually:

```text
A = {BRCA1, TP53, APOE}
B = {TP53, APOE, MYC}
```

Then:

```text
A UNION B
= {BRCA1, TP53, APOE, MYC}

A INTERSECT B
= {TP53, APOE}

A EXCEPT B
= {BRCA1}
```

This is very close to mathematical set theory.


# 7.2 Requirements for Set Operations

Before PostgreSQL can combine two query results, the queries must be **union-compatible**.

The major requirements are:

- The queries must return the **same number of columns**
- Corresponding columns must have **compatible data types**
- Column order must represent equivalent meanings

For example, this is logically valid:

```sql
SELECT gene_id, gene_symbol
FROM liver_genes

UNION

SELECT gene_id, gene_symbol
FROM brain_genes;
```

But this is logically incorrect:

```sql
SELECT gene_id, gene_symbol
FROM liver_genes

UNION

SELECT patient_id, age
FROM patients;
```

Even if PostgreSQL could coerce some data types, the biological meanings of the columns are unrelated.

> Set compatibility is not only a technical requirement; it is also a semantic requirement.


# 7.3 UNION — Combining Results Without Duplicates

`UNION` combines the results of two or more `SELECT` queries and removes duplicate rows.

Conceptually:

```text
Result A
   +
Result B
   ↓
Combine
   ↓
Remove duplicates
   ↓
Final result
```

### Real-Life Example

Suppose two hospitals maintain patient IDs for a collaborative study.

Hospital A:

```text
101
102
103
```

Hospital B:

```text
103
104
105
```

Using `UNION`, patient 103 appears only once.



```python
SELECT patient_id
FROM hospital_a_patients

UNION

SELECT patient_id
FROM hospital_b_patients;
```

Possible result:

```text
101
102
103
104
105
```

### Why Do We Need UNION?

`UNION` is useful when records come from multiple sources and we want a **deduplicated combined population**.

Typical use cases include:

- Combining cohorts
- Combining gene lists
- Combining sample registries
- Combining genomic annotation sources
- Merging results from multiple studies


# 7.4 Omics Example: Combining Genes from Two Tissues

Suppose we have two tables:

- `liver_genes`
- `brain_genes`

Each contains genes considered expressed in that tissue.

We want:

> Which genes are expressed in either liver or brain?



```python
SELECT gene_symbol
FROM liver_genes

UNION

SELECT gene_symbol
FROM brain_genes;
```

If `TP53` appears in both tissues, it will appear only once in the final result.

This answers the biological question:

> What is the **combined unique gene set** across both tissues?


# 7.5 UNION ALL — Combining Results While Preserving Duplicates

`UNION ALL` also combines query results, but unlike `UNION`, it does **not remove duplicates**.

Example:



```python
SELECT gene_symbol
FROM liver_genes

UNION ALL

SELECT gene_symbol
FROM brain_genes;
```

If `TP53` occurs in both tables, the output may contain:

```text
TP53
TP53
```

### Why Is UNION ALL Important?

Duplicates are not always errors.

Sometimes repeated rows represent meaningful observations.

For example:

- The same gene detected in multiple tissues
- The same variant observed in multiple cohorts
- The same patient appearing in multiple visits
- The same sample appearing in multiple batches

`UNION ALL` preserves the full row count and provenance implied by the source queries.


# 7.6 UNION vs UNION ALL

This distinction is extremely important.

| Feature | UNION | UNION ALL |
|---|---|---|
| Combines query results | Yes | Yes |
| Removes duplicates | Yes | No |
| Preserves repeated rows | No | Yes |
| Usually requires duplicate elimination work | Yes | No |
| Often faster | Usually no | Usually yes |

Because `UNION` must remove duplicates, PostgreSQL may need additional work such as sorting or hashing.

Therefore, if duplicates are acceptable or meaningful, `UNION ALL` is often preferable.


# 7.7 Clinical Example: Combining Diagnoses Across Cohorts

Suppose we have:

- `study_a_patients`
- `study_b_patients`

We want all diagnosis labels appearing in either study.



```python
SELECT diagnosis
FROM study_a_patients

UNION

SELECT diagnosis
FROM study_b_patients;
```

This produces a unique list of diagnosis categories across both studies.

If instead we want to preserve every diagnostic record:




```python
SELECT diagnosis
FROM study_a_patients

UNION ALL

SELECT diagnosis
FROM study_b_patients;
```

The second query may be useful before calculating total frequencies across both cohorts.


# 7.8 INTERSECT — Finding Common Rows

`INTERSECT` returns rows that appear in **both query results**.

Conceptually:

```text
A ∩ B
```

If:

```text
A = {BRCA1, TP53, APOE}
B = {TP53, APOE, MYC}
```

then:

```text
A INTERSECT B
= {TP53, APOE}
```

This makes `INTERSECT` extremely useful for overlap analyses.


# 7.9 Omics Example: Genes Shared Between Liver and Brain

### Biological Question

> Which genes are expressed in both liver and brain?



```python
SELECT gene_symbol
FROM liver_genes

INTERSECT

SELECT gene_symbol
FROM brain_genes;
```

This returns the **shared gene set**.

Typical omics applications include:

- Shared genes between tissues
- Shared variants between cohorts
- Common pathways between experiments
- Shared proteins across conditions
- Overlapping significant loci between GWAS analyses


# 7.10 Clinical Example: Patients Present in Two Studies

Suppose the same participant IDs may appear in two cohort tables.

We can identify overlapping participants:



```python
SELECT patient_id
FROM cohort_a

INTERSECT

SELECT patient_id
FROM cohort_b;
```

This can be useful for:

- Detecting duplicate recruitment
- Identifying longitudinal follow-up cohorts
- Cross-study validation
- Participant reconciliation


# 7.11 EXCEPT — Finding Rows Present in One Result but Not Another

`EXCEPT` returns rows from the first query that do not appear in the second query.

Conceptually:

```text
A - B
```

If:

```text
A = {BRCA1, TP53, APOE}
B = {TP53, APOE, MYC}
```

then:

```text
A EXCEPT B
= {BRCA1}
```


# 7.12 Omics Example: Liver-Specific Genes

### Question

> Which genes appear in the liver gene set but not in the brain gene set?



```python
SELECT gene_symbol
FROM liver_genes

EXCEPT

SELECT gene_symbol
FROM brain_genes;
```

The result represents genes unique to the first dataset under the exact criteria used to construct those two query results.

Important scientific caution:

> SQL can identify set differences, but biological interpretation requires careful thought.

A gene missing from the brain set may be:

- Truly tissue-specific
- Below the detection threshold
- Missing because of technical filtering
- Absent due to incomplete measurement


# 7.13 Direction Matters with EXCEPT

`EXCEPT` is not symmetric.

```text
A EXCEPT B
```

is generally different from:

```text
B EXCEPT A
```

For example:



```python
SELECT gene_symbol
FROM liver_genes

EXCEPT

SELECT gene_symbol
FROM brain_genes;
```

means:

> Genes found in liver but not brain.

Whereas:



```python
SELECT gene_symbol
FROM brain_genes

EXCEPT

SELECT gene_symbol
FROM liver_genes;
```

means:

> Genes found in brain but not liver.

Always pay attention to the order of the queries.


# 7.14 Variant Comparison Example

Suppose we have:

- `case_variants`
- `control_variants`

We want variants found among cases but not controls.



```python
SELECT variant_id
FROM case_variants

EXCEPT

SELECT variant_id
FROM control_variants;
```

This is a clean set-theoretic comparison.

However, such a result should not automatically be interpreted as a disease-associated variant. Differences may reflect:

- Sample size
- Sequencing depth
- Genotyping quality
- Allele frequency
- Missingness
- Population structure

SQL performs the comparison; scientific inference requires proper statistical analysis.


# 7.15 Set Operations with Multiple Columns

Set operators compare complete rows.

For example:



```python
SELECT gene_id, tissue
FROM experiment_a

INTERSECT

SELECT gene_id, tissue
FROM experiment_b;
```

A row is considered shared only if both corresponding values match.

For example:

```text
BRCA1 | Liver
```

is different from:

```text
BRCA1 | Brain
```

This is crucial when working with multidimensional omics results.


# 7.16 Column Names in Set Operations

The final result normally takes its output column names from the **first SELECT**.

Example:



```python
SELECT gene_symbol AS gene
FROM liver_genes

UNION

SELECT gene_symbol AS symbol
FROM brain_genes;
```

The final column will typically be named:

```text
gene
```

because that alias came from the first query.

Therefore, when building production queries, use clear aliases in the first `SELECT`.


# 7.17 Combining More Than Two Queries

Set operations are not limited to two queries.

For example:



```python
SELECT gene_symbol
FROM liver_genes

UNION

SELECT gene_symbol
FROM brain_genes

UNION

SELECT gene_symbol
FROM heart_genes;
```

This produces the unique set of genes found across all three tissues.

With `UNION ALL`, repeated genes would be preserved.


# 7.18 Set Operations with WHERE

Each individual query can contain its own filtering logic.

Example:



```python
SELECT gene_id
FROM liver_expression
WHERE expression_value > 10

INTERSECT

SELECT gene_id
FROM brain_expression
WHERE expression_value > 10;
```

This asks:

> Which genes pass the expression threshold in both liver and brain?

Notice that `WHERE` is applied separately within each query before the set comparison.


# 7.19 Set Operations with JOIN

Each query may itself contain joins.

For example:

> Which genes are highly expressed in blood samples from diabetic patients and also highly expressed in blood samples from hypertensive patients?



```python
SELECT DISTINCT e.gene_id
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
JOIN expression AS e
    ON s.sample_id = e.sample_id
WHERE p.diagnosis = 'Type 2 Diabetes'
  AND s.tissue = 'Blood'
  AND e.expression_value > 20

INTERSECT

SELECT DISTINCT e.gene_id
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
JOIN expression AS e
    ON s.sample_id = e.sample_id
WHERE p.diagnosis = 'Hypertension'
  AND s.tissue = 'Blood'
  AND e.expression_value > 20;
```

This demonstrates an important principle:

> Set operations combine complete query results, and those queries may themselves be complex.


# 7.20 Set Operations with Aggregation

We can also compare aggregated results.

Suppose we want tissues having more than 100 samples in either of two studies:



```python
SELECT tissue
FROM study_a_samples
GROUP BY tissue
HAVING COUNT(*) > 100

UNION

SELECT tissue
FROM study_b_samples
GROUP BY tissue
HAVING COUNT(*) > 100;
```

Each query performs:

```text
GROUP BY
↓
HAVING
↓
Set operation
```

The final result combines the qualifying tissue sets.


# 7.21 ORDER BY with Set Operations

When using set operations, `ORDER BY` is usually applied to the **final combined result**.

Example:



```python
SELECT gene_symbol
FROM liver_genes

UNION

SELECT gene_symbol
FROM brain_genes

ORDER BY gene_symbol;
```

The sorting occurs after the set operation has produced its final result.


# 7.22 LIMIT with Set Operations

Similarly, `LIMIT` can be applied to the final result:



```python
SELECT gene_symbol
FROM liver_genes

UNION

SELECT gene_symbol
FROM brain_genes

ORDER BY gene_symbol
LIMIT 10;
```

This returns only the first ten rows of the combined sorted gene set.


# 7.23 Parentheses and Query-Level ORDER BY

Sometimes we need to sort or limit one component before applying the set operator.

Parentheses help define the intended scope.

Example:



```python
(
    SELECT gene_id, expression_value
    FROM liver_expression
    ORDER BY expression_value DESC
    LIMIT 10
)

UNION

(
    SELECT gene_id, expression_value
    FROM brain_expression
    ORDER BY expression_value DESC
    LIMIT 10
);
```

This asks PostgreSQL to take the top ten rows from each query first, then combine them.

Without parentheses, the scope of `ORDER BY` and `LIMIT` may not represent what we intend.


# 7.24 UNION vs JOIN — A Critical Difference

Beginners often confuse `UNION` with `JOIN`.

They solve completely different problems.

### JOIN

Combines **columns horizontally** based on a relationship.

Example:

```text
patient_id | diagnosis
+
patient_id | tissue

↓

patient_id | diagnosis | tissue
```

### UNION

Combines **rows vertically** from compatible result sets.

Example:

```text
Liver genes
+
Brain genes

↓

Combined gene list
```

Think:

```text
JOIN  → add columns
UNION → add rows
```

This simple rule is very useful.


# 7.25 Example: JOIN vs UNION

### JOIN



```python
SELECT
    p.patient_id,
    p.diagnosis,
    s.tissue
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

This connects related information.

### UNION



```python
SELECT patient_id
FROM study_a_patients

UNION

SELECT patient_id
FROM study_b_patients;
```

This stacks compatible datasets.

The two operations are conceptually unrelated despite both involving multiple tables.


# 7.26 INTERSECT vs INNER JOIN

`INTERSECT` can sometimes answer a question similar to an inner join, but the semantics differ.

Example:



```python
SELECT gene_id
FROM liver_genes

INTERSECT

SELECT gene_id
FROM brain_genes;
```

This asks:

> Which gene IDs occur in both sets?

An equivalent-style join might be:



```python
SELECT DISTINCT l.gene_id
FROM liver_genes AS l
JOIN brain_genes AS b
    ON l.gene_id = b.gene_id;
```

Both may return similar gene IDs, but:

- `INTERSECT` expresses **set overlap**
- `JOIN` expresses a **relationship between rows**

Choose the construct that best describes the question.


# 7.27 EXCEPT vs NOT EXISTS

Suppose we want genes in liver but not brain.

Using `EXCEPT`:



```python
SELECT gene_id
FROM liver_genes

EXCEPT

SELECT gene_id
FROM brain_genes;
```

Using `NOT EXISTS`:



```python
SELECT l.gene_id
FROM liver_genes AS l
WHERE NOT EXISTS (
    SELECT 1
    FROM brain_genes AS b
    WHERE b.gene_id = l.gene_id
);
```

Both can express set difference.

`EXCEPT` is often very elegant when the query outputs are directly compatible.

`NOT EXISTS` is often more flexible when the matching rule is more complex.


# 7.28 NULL Behavior in Set Operations

Set operations have specific duplicate-handling semantics.

For duplicate elimination in operations such as `UNION`, PostgreSQL treats rows containing corresponding null values as duplicates for set-operation purposes when the rows are otherwise equal.

This differs from ordinary comparisons such as:

```sql
NULL = NULL
```

which do not evaluate to `TRUE`.

This is another reminder that SQL's treatment of `NULL` depends on context.


# 7.29 Duplicate Elimination and Performance

`UNION`, `INTERSECT`, and `EXCEPT` generally involve duplicate elimination unless their `ALL` variants are used where supported.

Duplicate elimination may require operations such as:

- Sorting
- Hashing
- Temporary memory
- Disk spill for large result sets

Therefore:

```sql
UNION ALL
```

can often be more efficient than:

```sql
UNION
```

when duplicate removal is unnecessary.

Performance should ultimately be assessed using:

```sql
EXPLAIN
```

and:

```sql
EXPLAIN ANALYZE
```

rather than assumptions alone.


# 7.30 PostgreSQL ALL Variants

PostgreSQL supports forms such as:

- `UNION ALL`
- `INTERSECT ALL`
- `EXCEPT ALL`

The `ALL` forms preserve duplicate multiplicities rather than reducing results to distinct rows.

For beginner-to-intermediate SQL, `UNION ALL` is the most commonly encountered of these.

The others become relevant when exact duplicate frequencies matter.


# 7.31 Omics Example: Shared Significant Genes

Suppose two independent differential-expression analyses produce tables:

- `disease_a_results`
- `disease_b_results`

Each contains:

```text
gene_id
adjusted_p_value
log2_fold_change
```

We can find genes significant in both analyses:



```python
SELECT gene_id
FROM disease_a_results
WHERE adjusted_p_value < 0.05

INTERSECT

SELECT gene_id
FROM disease_b_results
WHERE adjusted_p_value < 0.05;
```

This identifies the **overlapping significant gene set**.

It does not prove a common biological mechanism by itself, but it gives us a candidate overlap for further analysis.


# 7.32 Omics Example: Genes Unique to One Disease

Suppose we want genes significant in Disease A but not Disease B:



```python
SELECT gene_id
FROM disease_a_results
WHERE adjusted_p_value < 0.05

EXCEPT

SELECT gene_id
FROM disease_b_results
WHERE adjusted_p_value < 0.05;
```

This produces an A-specific result set under the specified significance criteria.


# 7.33 GWAS-Style Example

Suppose:

```text
schizophrenia_hits
bipolar_hits
```

contain genome-wide significant loci.

To find shared lead variants:



```python
SELECT variant_id
FROM schizophrenia_hits

INTERSECT

SELECT variant_id
FROM bipolar_hits;
```

To find schizophrenia-only lead variants:



```python
SELECT variant_id
FROM schizophrenia_hits

EXCEPT

SELECT variant_id
FROM bipolar_hits;
```

These examples demonstrate how set operations can naturally express **cross-trait comparisons**.


# 7.34 Sample QC Example

Suppose:

- `all_samples` contains every enrolled sample
- `passed_qc_samples` contains samples surviving quality control

To find excluded samples:



```python
SELECT sample_id
FROM all_samples

EXCEPT

SELECT sample_id
FROM passed_qc_samples;
```

This is one of the cleanest ways to answer:

> Which samples were removed during QC?


# 7.35 Combining Multiple Omics Cohorts

Suppose three studies contain variant IDs.

To create a unique combined catalogue:



```python
SELECT variant_id
FROM cohort_1_variants

UNION

SELECT variant_id
FROM cohort_2_variants

UNION

SELECT variant_id
FROM cohort_3_variants;
```

If instead we want every occurrence preserved:



```python
SELECT variant_id
FROM cohort_1_variants

UNION ALL

SELECT variant_id
FROM cohort_2_variants

UNION ALL

SELECT variant_id
FROM cohort_3_variants;
```

The second form can later be aggregated to determine how frequently each variant appears across source datasets.


# 7.36 UNION ALL Followed by GROUP BY

This is a powerful pattern.

Suppose we want to know in how many input records each variant appears.



```python
SELECT
    variant_id,
    COUNT(*) AS occurrence_count
FROM (
    SELECT variant_id
    FROM cohort_1_variants

    UNION ALL

    SELECT variant_id
    FROM cohort_2_variants

    UNION ALL

    SELECT variant_id
    FROM cohort_3_variants
) AS combined_variants
GROUP BY variant_id
ORDER BY occurrence_count DESC;
```

Now:

```text
UNION ALL
```

preserves all observations, while:

```text
GROUP BY
```

summarizes their frequency.

This is often more informative than immediately using `UNION`.


# 7.37 Adding Source Labels with UNION ALL

A particularly useful analytical technique is to preserve information about the source dataset.

Example:



```python
SELECT
    gene_id,
    'Liver' AS source_tissue
FROM liver_genes

UNION ALL

SELECT
    gene_id,
    'Brain' AS source_tissue
FROM brain_genes;
```

Now the result preserves both:

- the gene
- the source tissue

This allows downstream analysis such as:



```python
SELECT
    gene_id,
    COUNT(DISTINCT source_tissue) AS tissue_count
FROM (
    SELECT gene_id, 'Liver' AS source_tissue
    FROM liver_genes

    UNION ALL

    SELECT gene_id, 'Brain' AS source_tissue
    FROM brain_genes
) AS combined
GROUP BY gene_id;
```

This is a useful pattern for constructing integrated multi-source analytical datasets.


# 7.38 Common Beginner Mistakes

## Mistake 1 — Different Number of Columns

Incorrect:



```python
SELECT gene_id, gene_symbol
FROM genes

UNION

SELECT sample_id
FROM samples;
```

The two query results do not contain the same number of columns.


## Mistake 2 — Incompatible Meanings

Even when SQL data types are compatible, this is conceptually wrong:



```python
SELECT gene_id
FROM genes

UNION

SELECT patient_id
FROM patients;
```

Both may be text identifiers, but they represent different entities.


## Mistake 3 — Using UNION When Duplicates Matter

If repeated observations carry scientific meaning, `UNION` may silently remove information.

Use `UNION ALL` when duplicates need to be preserved.


## Mistake 4 — Assuming EXCEPT Is Symmetric

```text
A EXCEPT B
```

is not the same as:

```text
B EXCEPT A
```


## Mistake 5 — Confusing UNION with JOIN

Remember:

```text
UNION → vertical combination of rows

JOIN  → horizontal combination of related columns
```


## Mistake 6 — Ordering Individual Queries Incorrectly

`ORDER BY` generally belongs to the final combined result unless individual branches are explicitly wrapped and limited using parentheses.


# 7.39 A Practical Decision Framework

When facing a comparison problem, ask:

- Do I want to combine unique rows from two results? → `UNION`
- Do I want to combine everything, including duplicates? → `UNION ALL`
- Do I want only rows shared by both results? → `INTERSECT`
- Do I want rows in A but not B? → `EXCEPT`
- Do I need columns from related tables? → `JOIN`
- Do I want to test whether a related record exists? → `EXISTS`
- Do I want a flexible anti-match condition? → `NOT EXISTS`


# 7.40 Building the Set-Operation Mental Model

Consider two gene lists:

```text
Liver genes
Brain genes
```

Ask what biological relationship you want.

### Any tissue

```text
Liver OR Brain
```

Use:

```sql
UNION
```

### Both tissues

```text
Liver AND Brain
```

Use:

```sql
INTERSECT
```

### Liver but not brain

```text
Liver - Brain
```

Use:

```sql
EXCEPT
```

### Keep every observation from both sources

Use:

```sql
UNION ALL
```

This simple mathematical model makes set operations easy to remember.


# 7.41 Module 7 Summary

In this module, we learned that SQL set operations compare or combine **complete query result sets**.

The major concepts are:

- `UNION` combines compatible query results and removes duplicate rows.
- `UNION ALL` combines results while preserving duplicates.
- `INTERSECT` returns rows common to both query results.
- `EXCEPT` returns rows found in the first result but not the second.
- `EXCEPT` is directional.
- Set-operation queries must return the same number of columns with compatible data types and corresponding meanings.
- Set operations work vertically on rows, whereas joins generally combine columns horizontally.
- Individual branches can contain `WHERE`, `JOIN`, `GROUP BY`, `HAVING`, and other SQL logic.
- `ORDER BY` and `LIMIT` commonly operate on the final combined result.
- Parentheses are useful when ordering or limiting individual branches.
- `UNION ALL` is often preferable when duplicate preservation is required or duplicate elimination is unnecessary.
- `INTERSECT` is useful for overlap analysis.
- `EXCEPT` is useful for identifying unique or excluded records.
- Set operations are especially natural for comparing cohorts, tissues, experiments, gene sets, variant sets, and QC results.


# Key Takeaway

> Set operations allow us to treat query results as mathematical sets.

For omics analysis:

```text
UNION
```

can answer:

> Which genes appear in either tissue?

```text
INTERSECT
```

can answer:

> Which genes appear in both tissues?

```text
EXCEPT
```

can answer:

> Which genes appear in one tissue but not the other?

And:

```text
UNION ALL
```

can preserve the complete observational history needed for downstream counting and integration.

These operations provide a clean bridge between **relational SQL and set-based scientific reasoning**.


# References

1. PostgreSQL Global Development Group. **PostgreSQL Documentation — Combining Queries (`UNION`, `INTERSECT`, `EXCEPT`)**.  
   https://www.postgresql.org/docs/current/queries-union.html

2. PostgreSQL Global Development Group. **PostgreSQL Documentation — SELECT**.  
   https://www.postgresql.org/docs/current/sql-select.html

3. PostgreSQL Global Development Group. **PostgreSQL Documentation — Table Expressions**.  
   https://www.postgresql.org/docs/current/queries-table-expressions.html

4. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

5. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

6. Date, C. J. *An Introduction to Database Systems*. Addison-Wesley.

7. Ensembl Documentation.  
   https://www.ensembl.org/info/docs/index.html

8. GTEx Portal.  
   https://gtexportal.org/

