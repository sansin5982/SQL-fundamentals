# Indexes and Query Performance
### PostgreSQL Index Architecture, B-Tree, Hash, GIN, GiST, BRIN, Composite and Partial Indexes, EXPLAIN, EXPLAIN ANALYZE, Query Planning, and Omics Applications

---

## Introduction: Why Query Performance Matters

A SQL query that works correctly on 100 rows may become painfully slow on 100 million rows.

This is especially relevant in biomedical and omics databases, where tables may contain:

- Millions of variants
- Tens of thousands of genes
- Millions or billions of gene-expression measurements
- Large clinical cohorts
- Longitudinal patient records
- Sequencing QC metrics
- Annotation tables
- Genomic coordinates

PostgreSQL provides **indexes** and a sophisticated **query planner** to retrieve data efficiently.

This module explains not only how to create indexes, but also **why they work, when PostgreSQL uses them, when it does not, and how to inspect query performance correctly**.


## 10.1 What Is an Index?

An **index** is a separate database structure designed to help PostgreSQL locate rows efficiently.

Without an appropriate index, PostgreSQL may need to inspect many or all rows in a table.

This is called a:

```text
Sequential Scan
```

With an appropriate index, PostgreSQL may be able to navigate directly toward matching entries.

### Layman Analogy

A textbook may contain 1,000 pages.

If you want information about `TP53`, you could:

```text
Read page 1
Read page 2
Read page 3
...
```

or use the book's index:

```text
TP53 → page 742
```

A database index serves a similar purpose.


## 10.2 Indexes Improve Access, Not the Data Itself

An index does not normally change the logical rows in a table.

It changes how efficiently PostgreSQL may find those rows.

For example:



```python
SELECT *
FROM genes
WHERE gene_id = 'ENSG00000141510';
```

The query returns the same logical result whether or not an index exists.

The difference may be the **execution strategy and cost**.


## 10.3 Why Not Index Every Column?

Indexes are useful, but they are not free.

They require:

- Disk space
- Memory/cache resources
- Maintenance during `INSERT`
- Maintenance during `UPDATE`
- Maintenance during `DELETE`
- Vacuum and storage-management considerations

Therefore:

> More indexes do not automatically mean a faster database.

Index design is a trade-off between **read performance and write/storage overhead**.


# Part I — Basic PostgreSQL Indexes

## 10.4 CREATE INDEX

Basic syntax:



```python
CREATE INDEX index_name
ON table_name (column_name);
```

Example:



```python
CREATE INDEX idx_genes_gene_symbol
ON genes (gene_symbol);
```

This may help queries such as:



```python
SELECT *
FROM genes
WHERE gene_symbol = 'TP53';
```

PostgreSQL still decides whether using the index is actually cheaper than another plan.


## 10.5 PostgreSQL Automatically Creates Some Indexes

PostgreSQL automatically creates unique indexes to enforce:

- `PRIMARY KEY`
- `UNIQUE`

Example:



```python
CREATE TABLE genes (
    gene_id VARCHAR(30) PRIMARY KEY,
    gene_symbol VARCHAR(50) UNIQUE
);
```

You normally do **not** need to create another identical index on `gene_id` or `gene_symbol`.


## 10.6 B-Tree Index

**B-tree** is PostgreSQL's default index method.

If you write:



```python
CREATE INDEX idx_expression_value
ON expression (expression_value);
```

PostgreSQL creates a B-tree index unless another method is specified.

B-tree indexes are highly useful for:

- Equality comparisons
- Range comparisons
- Sorting
- Many `BETWEEN` queries
- `<`, `<=`, `=`, `>=`, `>`


### Omics Example: Expression Threshold



```python
CREATE INDEX idx_expression_value
ON expression (expression_value);
```


```python
SELECT *
FROM expression
WHERE expression_value > 20;
```

Whether PostgreSQL uses the index depends partly on how selective the condition is.


## 10.7 Selectivity

**Selectivity** describes how strongly a condition narrows the dataset.

Suppose a table contains 100 million variants.

Query A returns 10 variants:



```python
SELECT *
FROM variants
WHERE variant_id = 'rs12345';
```

This is highly selective.

Query B returns 80 million rows:



```python
SELECT *
FROM variants
WHERE chromosome IS NOT NULL;
```

This is poorly selective.

An index is generally more attractive when a query retrieves a relatively small fraction of the table.


## 10.8 Sequential Scan

A sequential scan reads table pages sequentially and evaluates rows.

Example plan terminology:

```text
Seq Scan on variants
```

Sequential scans are not inherently bad.

For queries that return a large fraction of a table, sequential access can be more efficient than repeatedly navigating through an index.


## 10.9 Index Scan

A plan may show:



```python
EXPLAIN
SELECT *
FROM genes
WHERE gene_symbol = 'TP53';
```

Possible terminology:

```text
Index Scan
```

This means PostgreSQL uses an index to locate matching table rows.


## 10.10 Index Only Scan

Sometimes PostgreSQL can satisfy a query largely from the index itself.

Possible plan:

```text
Index Only Scan
```

This can reduce heap access, although visibility information and PostgreSQL's MVCC behavior affect whether heap checks are still required.


## 10.11 Bitmap Index Scan and Bitmap Heap Scan

For some queries PostgreSQL may use a plan involving:

```text
Bitmap Index Scan
Bitmap Heap Scan
```

This can be useful when more rows match than would make a simple index scan ideal, but an index is still beneficial.

The planner chooses among these strategies based on estimated cost.


# Part II — Composite and Specialized Indexes

## 10.12 Composite Index

A **composite index** contains multiple columns.

Example:



```python
CREATE INDEX idx_expression_gene_sample
ON expression (gene_id, sample_id);
```

This can be useful when queries frequently filter using those columns in a compatible pattern.


## 10.13 Column Order Matters

Consider:



```python
CREATE INDEX idx_samples_tissue_qc
ON samples (tissue, qc_status);
```

This index is structured beginning with `tissue`.

Queries such as:



```python
SELECT *
FROM samples
WHERE tissue = 'Blood'
  AND qc_status = 'Pass';
```

may benefit strongly.

A query filtering only on `qc_status` may not benefit from the same index in the same way.

Composite-index column order should reflect actual query patterns.


## 10.14 Composite Omics Example

Suppose we frequently ask:

> Find expression measurements for a specific gene in a specific sample.

A composite index might be:



```python
CREATE INDEX idx_expression_sample_gene
ON expression (sample_id, gene_id);
```

However, if `(sample_id, gene_id)` is already the primary key, PostgreSQL already has a unique index supporting that key. Creating a duplicate index would usually be unnecessary.


## 10.15 Unique Index

A unique index prevents duplicate indexed values.



```python
CREATE UNIQUE INDEX idx_genes_ensembl_id
ON genes (ensembl_gene_id);
```

Often it is semantically clearer to define uniqueness as a table constraint:



```python
ALTER TABLE genes
ADD CONSTRAINT uq_genes_ensembl_id
UNIQUE (ensembl_gene_id);
```

Use constraints to express data-integrity rules; PostgreSQL uses indexes internally to enforce them.


## 10.16 Partial Index

A **partial index** indexes only rows satisfying a condition.

Example:



```python
CREATE INDEX idx_samples_qc_pass
ON samples (sample_id)
WHERE qc_status = 'Pass';
```

This may be useful when analyses frequently access only QC-passed samples.

Instead of indexing every row, the index covers the scientifically relevant subset.


## 10.17 Omics Partial Index Example



```python
CREATE INDEX idx_rare_variants
ON variants (gene_id, allele_frequency)
WHERE allele_frequency < 0.01;
```

This may help workloads that repeatedly query rare variants.

Partial indexes can reduce index size and maintenance cost when the indexed subset is well defined and frequently queried.


## 10.18 Expression Index

PostgreSQL can index the result of an expression.

Example:



```python
CREATE INDEX idx_genes_lower_symbol
ON genes (LOWER(gene_symbol));
```

This can support queries written compatibly:



```python
SELECT *
FROM genes
WHERE LOWER(gene_symbol) = 'tp53';
```

This is called an **expression index** or **functional index**.


## 10.19 Covering Indexes and INCLUDE

PostgreSQL supports `INCLUDE` columns.



```python
CREATE INDEX idx_expression_gene_include
ON expression (gene_id)
INCLUDE (expression_value);
```

The included column is stored in the index but is not part of the index search key.

This can sometimes help index-only access for frequently used queries.


# Part III — PostgreSQL Index Methods

## 10.20 Hash Index

A hash index is designed primarily for equality comparisons.

Example:



```python
CREATE INDEX idx_genes_symbol_hash
ON genes USING HASH (gene_symbol);
```

For most ordinary applications, B-tree remains the default and often sufficient choice. Use specialized index types because the workload justifies them, not merely because they exist.


## 10.21 GIN Index

**GIN** stands for **Generalized Inverted Index**.

GIN is useful for data containing multiple searchable components, including PostgreSQL features such as:

- Arrays
- `jsonb`
- Full-text search
- Certain extension-provided data types

Example with a `jsonb` annotation column:



```python
CREATE INDEX idx_variants_annotation_gin
ON variants
USING GIN (annotation);
```

A genomics database storing flexible annotation documents in `jsonb` may benefit from GIN for containment-style searches.


## 10.22 GiST Index

**GiST** stands for **Generalized Search Tree**.

GiST provides an extensible indexing framework used by many data types and operators, including:

- Range types
- Geometric data
- Specialized extensions

GiST becomes especially interesting when data represent intervals or multidimensional relationships.


## 10.23 Genomic Range Example

PostgreSQL range types can represent intervals.

For example, a genomic feature table might store a range column and index it using GiST:



```python
CREATE INDEX idx_genomic_features_region
ON genomic_features
USING GIST (genomic_range);
```

This can support interval-style operators when the column uses an appropriate PostgreSQL range type.

Such designs are useful for questions like:

> Which genomic features overlap this region?


## 10.24 BRIN Index

**BRIN** stands for **Block Range Index**.

BRIN stores summary information about ranges of physical table blocks rather than indexing every row individually.

It is especially useful when:

- Tables are extremely large
- Values correlate with physical row order
- A very small index is desirable

Examples may include large event tables ordered approximately by date or genomic tables physically organized by coordinate.



```python
CREATE INDEX idx_variants_position_brin
ON variants
USING BRIN (position);
```

A BRIN index is much smaller than a typical B-tree index, but it has different performance characteristics.


## 10.25 B-Tree vs BRIN for Genomic Coordinates

Suppose variants are physically stored approximately in chromosome-position order.

A BRIN index may efficiently eliminate large irrelevant block ranges.

A B-tree provides much more precise row-level indexing.

The best choice depends on:

- Table size
- Physical ordering
- Query selectivity
- Storage constraints
- Update patterns


# Part IV — Query Planning and EXPLAIN

## 10.26 PostgreSQL Is Cost Based

PostgreSQL uses a **cost-based query optimizer**.

It estimates the cost of possible execution plans and chooses one expected to be efficient.

The planner considers factors such as:

- Table statistics
- Number of rows
- Value distributions
- Selectivity
- Available indexes
- Join strategies
- Estimated I/O
- CPU cost
- Parallelism


## 10.27 EXPLAIN

`EXPLAIN` shows PostgreSQL's planned execution strategy without actually executing the query in the ordinary case.



```python
EXPLAIN
SELECT *
FROM variants
WHERE variant_id = 'rs12345';
```

The output may contain:

- Scan type
- Estimated cost
- Estimated rows
- Width
- Join nodes
- Sorts
- Aggregates


## 10.28 Understanding Cost

A plan may contain:



```python
cost=0.42..8.44
```

The two numbers are approximately:

```text
startup cost
total cost
```

These are PostgreSQL planner cost units, **not milliseconds**.


## 10.29 Estimated Rows

A plan may show:



```python
rows=1
```

This is the planner's estimate of how many rows the node will produce.

Accurate row estimates are extremely important because they influence plan selection.


## 10.30 EXPLAIN ANALYZE

`EXPLAIN ANALYZE` actually executes the query and reports runtime information.



```python
EXPLAIN ANALYZE
SELECT *
FROM genes
WHERE gene_symbol = 'TP53';
```

It can show information such as:

- Actual time
- Actual rows
- Number of loops
- Planning time
- Execution time

Because the query is executed, use caution with statements that modify data.


## 10.31 EXPLAIN with BUFFERS

For deeper performance analysis:



```python
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM expression
WHERE gene_id = 'ENSG00000141510';
```

Buffer information helps reveal whether data pages were obtained from shared buffers or required other access patterns.

This is useful for serious performance investigation.


## 10.32 Estimates vs Actual Rows

One of the most useful diagnostics is comparing:

```text
estimated rows
```

with:

```text
actual rows
```

Large discrepancies may indicate that PostgreSQL's statistical model does not adequately describe the data distribution.

This can lead to poor plan choices.


## 10.33 ANALYZE

`ANALYZE` collects statistics used by PostgreSQL's planner.



```python
ANALYZE variants;
```

PostgreSQL also performs statistics maintenance through autovacuum/autoanalyze under normal configurations.

Fresh statistics are important after substantial data changes.


## 10.34 VACUUM and VACUUM ANALYZE

PostgreSQL uses MVCC, which creates maintenance requirements different from some other database systems.

Commands include:



```python
VACUUM expression;
```


```python
VACUUM ANALYZE expression;
```

`VACUUM` and `ANALYZE` serve different purposes, though they can be invoked together.

Database maintenance is an important part of sustained PostgreSQL performance.


# Part V — Why PostgreSQL May Ignore an Index

## 10.35 An Index Does Not Guarantee an Index Scan

Suppose:



```python
CREATE INDEX idx_samples_tissue
ON samples (tissue);
```

and then:



```python
SELECT *
FROM samples
WHERE tissue = 'Blood';
```

PostgreSQL may still choose a sequential scan.

Why?

If most rows are blood samples, reading much of the table sequentially may be cheaper than following many index entries.


## 10.36 Small Tables

For very small tables, a sequential scan is often cheaper.

If a table contains only 50 rows, scanning all 50 rows may be simpler than navigating an index.

Therefore, do not judge index usefulness using tiny teaching datasets alone.


## 10.37 Functions Can Affect Index Usage

Suppose the index is:



```python
CREATE INDEX idx_genes_symbol
ON genes (gene_symbol);
```

but the query is:



```python
SELECT *
FROM genes
WHERE LOWER(gene_symbol) = 'tp53';
```

The ordinary index on `gene_symbol` may not support this expression in the desired way.

An expression index can:



```python
CREATE INDEX idx_genes_lower_symbol
ON genes (LOWER(gene_symbol));
```

Index design and query expression must align.


## 10.38 Data-Type Compatibility Matters

Poorly designed comparisons or implicit conversions can interfere with efficient access and may also indicate schema problems.

Use appropriate column types for:

- Numeric quantities
- Dates
- Identifiers
- Boolean values
- Genomic coordinates


## 10.39 Wildcard Search and Indexes

A query such as:



```python
SELECT *
FROM genes
WHERE gene_symbol LIKE 'BRCA%';
```

may be more index-friendly under suitable operator-class/collation conditions than:



```python
SELECT *
FROM genes
WHERE gene_symbol LIKE '%BRCA%';
```

A leading wildcard makes ordinary B-tree prefix navigation difficult.

PostgreSQL extensions and specialized indexes can support more advanced text-search workloads.


# Part VI — Indexing Joins and Relationships

## 10.40 Indexing Foreign-Key Columns

PostgreSQL automatically indexes primary and unique keys used as referenced keys, but it does **not automatically create an index on every referencing foreign-key column**.

Suppose:



```python
samples.patient_id REFERENCES patients(patient_id)
```

If `samples.patient_id` is frequently used for joins or referential actions, an index may be useful:



```python
CREATE INDEX idx_samples_patient_id
ON samples (patient_id);
```

This is an important PostgreSQL design consideration.


## 10.41 Join Example



```python
SELECT
    p.patient_id,
    p.diagnosis,
    s.sample_id
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
WHERE p.patient_id = 101;
```

Appropriate indexes can reduce the amount of data PostgreSQL must inspect during relationship traversal.


## 10.42 Expression Table Relationships

For a large expression table, common access paths may include:

```text
sample_id
gene_id
sample_id + gene_id
gene_id + expression_value
```

But indexes should be chosen from actual workloads, not created mechanically for every possible combination.


# Part VII — Omics Indexing Patterns

## 10.43 Variant Lookup by ID



```python
CREATE INDEX idx_variants_variant_id
ON variants (variant_id);
```


```python
SELECT *
FROM variants
WHERE variant_id = 'rs429358';
```

This supports direct variant lookup when `variant_id` is not already uniquely indexed.


## 10.44 Variant Lookup by Chromosome and Position



```python
CREATE INDEX idx_variants_chr_pos
ON variants (chromosome, position);
```

Query:



```python
SELECT *
FROM variants
WHERE chromosome = '6'
  AND position BETWEEN 25000000 AND 34000000
ORDER BY position;
```

This is directly relevant to regional genomic analyses such as xMHC exploration.


## 10.45 Gene-Based Variant Lookup



```python
CREATE INDEX idx_variants_gene
ON variants (gene_id);
```


```python
SELECT *
FROM variants
WHERE gene_id = 'ENSG00000196126';
```

This may support gene-centric variant exploration.


## 10.46 Rare Variant Workload



```python
CREATE INDEX idx_variants_gene_rare
ON variants (gene_id, allele_frequency)
WHERE allele_frequency < 0.01;
```

Query:



```python
SELECT *
FROM variants
WHERE gene_id = 'ENSG00000141510'
  AND allele_frequency < 0.01;
```

A partial index can align closely with the scientific workload.


## 10.47 Expression Lookup by Gene



```python
CREATE INDEX idx_expression_gene
ON expression (gene_id);
```


```python
SELECT *
FROM expression
WHERE gene_id = 'ENSG00000141510';
```

This can be useful when a large expression matrix is stored in long relational format.


## 10.48 Tissue-Based Query Pattern

If analyses frequently filter samples by tissue:



```python
CREATE INDEX idx_samples_tissue
ON samples (tissue);
```

But if there are only a few tissue categories and each represents many rows, selectivity may be low.

A composite or partial index aligned with the actual workload may be more useful.


## 10.49 QC-Passed Blood Samples



```python
CREATE INDEX idx_samples_passed_blood
ON samples (sample_id)
WHERE qc_status = 'Pass'
  AND tissue = 'Blood';
```

This is an example of designing an index around a recurring analysis-ready subset.


# Part VIII — Sorting, Grouping, and Indexes

## 10.50 Indexes and ORDER BY

A suitable B-tree index can sometimes provide rows in the required order and avoid a separate sort.

Example:



```python
CREATE INDEX idx_expression_value_desc
ON expression (expression_value DESC);
```


```python
SELECT *
FROM expression
ORDER BY expression_value DESC
LIMIT 10;
```

This can be valuable for top-N queries.


## 10.51 Top-N Omics Example



```python
SELECT
    gene_id,
    expression_value
FROM expression
ORDER BY expression_value DESC
LIMIT 100;
```

An appropriate index can make top-N retrieval much more efficient than sorting a huge table.


## 10.52 Indexes Do Not Automatically Make GROUP BY Fast

Aggregation may still require:

- Reading many rows
- Hash aggregation
- Sorting
- Parallel processing

Indexes may help some access patterns, but an index is not a universal solution for expensive aggregation.

Materialized views, partitioning, or pre-aggregation may sometimes be more appropriate.


# Part IX — Index Maintenance

## 10.53 INSERT Cost

Every relevant index may need updating when a row is inserted.

A table with many indexes therefore has more write overhead.


## 10.54 UPDATE Cost

If an indexed column changes, PostgreSQL must maintain the affected index structures.

This increases write cost.


## 10.55 DELETE and MVCC

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**.

Deleted or obsolete row versions are not always physically removed immediately.

Vacuuming is therefore important for reclaiming reusable space and maintaining healthy table/index behavior.


## 10.56 REINDEX

PostgreSQL provides:



```python
REINDEX INDEX idx_expression_gene;
```

and related forms for rebuilding indexes when necessary.

Routine reindexing should not be performed blindly; it is a maintenance tool for specific situations.


## 10.57 Index Size

Indexes consume storage.

PostgreSQL provides functions for inspecting sizes, for example:



```python
SELECT pg_size_pretty(
    pg_relation_size('idx_expression_gene')
);
```

Understanding index size is useful for very large genomic databases.


# Part X — Performance Anti-Patterns

## 10.58 SELECT * When You Need Few Columns

Instead of:



```python
SELECT *
FROM variants
WHERE gene_id = 'ENSG00000141510';
```

prefer only required columns when practical:



```python
SELECT
    variant_id,
    position,
    allele_frequency
FROM variants
WHERE gene_id = 'ENSG00000141510';
```

This can reduce data transfer and may enable more efficient access patterns.


## 10.59 Missing WHERE Clause

A forgotten filter on a huge table can return enormous datasets.

Always verify analytical scope before running expensive queries.


## 10.60 Unnecessary DISTINCT

`DISTINCT` may require sorting or hashing.

Do not use it merely to hide duplicate rows caused by an incorrect join.

First understand **why duplicates exist**.


## 10.61 Unnecessary ORDER BY

Sorting millions of rows can be expensive.

Use `ORDER BY` only when ordering is required.


## 10.62 Indexing Low-Value Columns Mechanically

A Boolean column such as:



```python
is_active BOOLEAN
```

may have poor selectivity if almost every row is `TRUE`.

A partial index may be more useful in some workloads:



```python
CREATE INDEX idx_inactive_samples
ON samples (sample_id)
WHERE is_active = FALSE;
```

if inactive rows are rare and frequently queried.


## 10.63 Over-Indexing

Creating indexes on every column can:

- Waste storage
- Slow writes
- Increase maintenance
- Complicate planning
- Duplicate existing indexes

Index only with a clear access pattern or integrity requirement.


# Part XI — A Practical Optimization Workflow

## 10.64 Step 1: Start with the Question

Example:

> Retrieve rare variants in the xMHC region on chromosome 6.

Query:



```python
SELECT
    variant_id,
    position,
    gene_id,
    allele_frequency
FROM variants
WHERE chromosome = '6'
  AND position BETWEEN 25000000 AND 34000000
  AND allele_frequency < 0.01;
```

Do not create indexes before understanding the workload.


## 10.65 Step 2: Inspect the Plan



```python
EXPLAIN
SELECT
    variant_id,
    position,
    gene_id,
    allele_frequency
FROM variants
WHERE chromosome = '6'
  AND position BETWEEN 25000000 AND 34000000
  AND allele_frequency < 0.01;
```

Look for:

- Scan type
- Estimated rows
- Costs
- Filter conditions


## 10.66 Step 3: Measure the Actual Query



```python
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    variant_id,
    position,
    gene_id,
    allele_frequency
FROM variants
WHERE chromosome = '6'
  AND position BETWEEN 25000000 AND 34000000
  AND allele_frequency < 0.01;
```

Now compare estimates with reality.


## 10.67 Step 4: Design an Index Around the Workload

A possible candidate:



```python
CREATE INDEX idx_variants_chr_pos_rare
ON variants (chromosome, position)
WHERE allele_frequency < 0.01;
```

This combines:

- Composite indexing
- Partial indexing
- A recurring rare-variant condition


## 10.68 Step 5: Re-Test

Run:



```python
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    variant_id,
    position,
    gene_id,
    allele_frequency
FROM variants
WHERE chromosome = '6'
  AND position BETWEEN 25000000 AND 34000000
  AND allele_frequency < 0.01;
```

Optimization should be **measured**, not assumed.


## 10.69 Step 6: Evaluate Trade-Offs

Ask:

- Did execution improve?
- How large is the index?
- How often is the query executed?
- How often is the table updated?
- Does another existing index already cover the workload?
- Is the index worth its maintenance cost?


# Part XII — Common Beginner Mistakes

## 10.70 Mistake: “Index = Faster”

Incorrect.

Better:

> An appropriate index can make a particular workload faster when the optimizer estimates that using it is beneficial.


## 10.71 Mistake: Duplicate Indexes

If a primary key already exists:



```python
PRIMARY KEY (sample_id)
```

do not mechanically create another identical B-tree index on `sample_id`.


## 10.72 Mistake: Ignoring Composite Column Order

These are different:



```python
CREATE INDEX idx_a
ON variants (chromosome, position);
```


```python
CREATE INDEX idx_b
ON variants (position, chromosome);
```

Their usefulness depends on query patterns.


## 10.73 Mistake: Using EXPLAIN ANALYZE Carelessly

`EXPLAIN ANALYZE` executes the statement.

For:



```python
DELETE
UPDATE
INSERT
```

this means the data-changing operation actually runs unless you deliberately protect the test using an appropriate transaction strategy.


## 10.74 Mistake: Optimizing Tiny Teaching Tables

A sequential scan on a 20-row table does not prove that an index is useless.

Performance testing should resemble realistic data volume and distribution.


## 10.75 Mistake: Optimizing Without Measurement

Do not guess.

Use:

```text
EXPLAIN
EXPLAIN ANALYZE
BUFFERS
statistics
realistic workloads
```

Performance engineering is empirical.


# Part XIII — Advanced Preview

## 10.76 Multicolumn Statistics

Columns can be correlated.

For example:

```text
chromosome
position
```

or:

```text
tissue
assay_type
```

PostgreSQL supports extended statistics that can help the planner estimate certain correlated conditions more accurately.

This becomes important in advanced performance tuning.


## 10.77 Partitioning Preview

For extremely large tables, partitioning may divide data into logical pieces.

Examples:

```text
variants by chromosome
events by year
samples by project
```

Partitioning is not the same as indexing, but the two strategies can complement each other.


## 10.78 Materialized Views and Indexes

From Module 9, materialized views physically store results.

They can also be indexed:



```python
CREATE INDEX idx_tissue_gene_mv_gene
ON tissue_gene_expression_mv (gene_id);
```

This combines:

```text
precomputation
+
physical storage
+
indexing
```

for repeated analytical workloads.


# 10.79 Index Selection Cheat Sheet

| Requirement | Possible Index Strategy |
|---|---|
| Equality/range lookup | B-tree |
| Composite filtering | Multicolumn B-tree |
| Small recurring subset | Partial index |
| Function/expression lookup | Expression index |
| Arrays / JSONB / full text | GIN |
| Range/geometric/special operators | GiST |
| Huge physically correlated table | BRIN |
| Enforce uniqueness | UNIQUE constraint/index |
| Top-N ordered retrieval | Suitable B-tree |
| Materialized summary lookup | Index materialized view |

These are starting points, not automatic prescriptions.


# 10.80 Omics Performance Mental Model

Suppose your database contains:

```text
2,000,000 variants
```

and you ask:

> Find rare variants in HLA-DQB1.

Without a suitable access path:

```text
Millions of rows
      ↓
Scan/filter
      ↓
small result
```

With a workload-aligned index:

```text
Index
  ↓
Relevant key range/subset
  ↓
small result
```

But if your query asks for almost every variant, scanning the table may still be the correct strategy.

The key concept is:

> Indexing is about reducing unnecessary work when the query pattern permits it.


# Module 10 Summary

In this module, we learned:

- An index is a database structure used to improve data access.
- PostgreSQL's default index type is B-tree.
- Indexes improve some reads but add storage and write-maintenance costs.
- Primary keys and unique constraints automatically create supporting unique indexes.
- PostgreSQL does not automatically index every referencing foreign-key column.
- Composite indexes contain multiple key columns, and column order matters.
- Partial indexes cover only rows satisfying a predicate.
- Expression indexes index calculated expressions.
- `INCLUDE` can add non-key columns to an index.
- PostgreSQL supports B-tree, Hash, GIN, GiST, and BRIN index methods.
- B-tree is useful for many equality, range, and ordering operations.
- GIN is important for structures such as arrays and `jsonb`.
- GiST supports extensible search strategies including range-oriented workloads.
- BRIN can be effective for very large, physically correlated tables.
- PostgreSQL uses a cost-based query planner.
- `EXPLAIN` shows the planned execution strategy.
- `EXPLAIN ANALYZE` executes the query and reports actual behavior.
- `BUFFERS` provides deeper I/O/cache insight.
- PostgreSQL may correctly choose a sequential scan even when an index exists.
- Selectivity strongly influences index usefulness.
- `ANALYZE` maintains planner statistics.
- PostgreSQL MVCC makes vacuum-related maintenance important.
- Index design should follow real query patterns.
- Optimization should always be measured before and after changes.


# Key Takeaway

> **Indexes are not decorations added to tables. They are workload-specific access structures.**

In omics databases, good indexing can transform questions such as:

```text
Find rs429358
Find all variants in chr6:25–34 Mb
Find rare variants in one gene
Retrieve expression for TP53
Find QC-passed blood samples
```

from expensive scans into efficient retrieval operations.

But advanced SQL performance requires more than `CREATE INDEX`.

You must understand:

```text
Query pattern
      ↓
Data distribution
      ↓
Selectivity
      ↓
Planner estimates
      ↓
Execution plan
      ↓
Measured runtime
```

That is the foundation of PostgreSQL query optimization.


# References

1. PostgreSQL Global Development Group. **PostgreSQL Documentation — Indexes**  
   https://www.postgresql.org/docs/current/indexes.html

2. PostgreSQL Global Development Group. **PostgreSQL Documentation — Index Types**  
   https://www.postgresql.org/docs/current/indexes-types.html

3. PostgreSQL Global Development Group. **PostgreSQL Documentation — Multicolumn Indexes**  
   https://www.postgresql.org/docs/current/indexes-multicolumn.html

4. PostgreSQL Global Development Group. **PostgreSQL Documentation — Partial Indexes**  
   https://www.postgresql.org/docs/current/indexes-partial.html

5. PostgreSQL Global Development Group. **PostgreSQL Documentation — Indexes on Expressions**  
   https://www.postgresql.org/docs/current/indexes-expressional.html

6. PostgreSQL Global Development Group. **PostgreSQL Documentation — Using EXPLAIN**  
   https://www.postgresql.org/docs/current/using-explain.html

7. PostgreSQL Global Development Group. **PostgreSQL Documentation — Planner Statistics**  
   https://www.postgresql.org/docs/current/planner-stats.html

8. PostgreSQL Global Development Group. **PostgreSQL Documentation — VACUUM**  
   https://www.postgresql.org/docs/current/sql-vacuum.html

9. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

10. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

11. Ensembl Documentation  
    https://www.ensembl.org/info/docs/index.html

12. GTEx Portal  
    https://gtexportal.org/

