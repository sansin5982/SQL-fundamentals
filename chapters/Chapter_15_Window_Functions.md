# Chapter 15: Window Functions 
### OVER(), PARTITION BY, ORDER BY, Ranking, Offset Functions, Window Aggregates, Frames, and Omics Applications

## Introduction

Window functions are among the most important tools in advanced analytical SQL. Unlike `GROUP BY`, which collapses rows, a window function calculates across related rows **while preserving the original rows**.

They answer questions such as:

- Which genes rank highest within each tissue?
- How does each sample compare with its sequencing-batch average?
- What was a patient's previous biomarker measurement?
- What is the change from baseline?
- What is the cumulative number of variants along a chromosome?
- Which are the top variants within every gene?

The central model is:

```text
Rows
 ↓
PARTITION BY → define groups
 ↓
ORDER BY → define sequence
 ↓
Window frame → define relevant neighboring rows
 ↓
Window function → calculate
 ↓
Original rows remain
```


## 15.1 GROUP BY vs Window Functions

`GROUP BY` reduces many rows to fewer summary rows. A window function keeps the row-level observations and adds group-level information.

With `GROUP BY`:


```python
SELECT tissue, AVG(expression_value) AS mean_expression
FROM expression
GROUP BY tissue;
```

The result has one row per tissue.

With a window function:


```python
SELECT
    sample_id,
    tissue,
    gene_id,
    expression_value,
    AVG(expression_value) OVER (
        PARTITION BY tissue
    ) AS tissue_mean
FROM expression;
```

Every original row remains. This distinction is fundamental:

```text
GROUP BY       → many rows become fewer rows
Window function → same rows + analytical information
```

## 15.2 Basic Syntax

The general structure is:

```sql
function_name(...) OVER (
    PARTITION BY ...
    ORDER BY ...
    ROWS / RANGE / GROUPS ...
)
```

`OVER()` tells PostgreSQL that the function should operate as a window calculation.

The three major components are:

- `PARTITION BY` — analytical groups
- `ORDER BY` — analytical sequence
- window frame — which rows within the ordered partition participate

Not every function needs all three.

## 15.3 OVER()

An empty `OVER()` treats all rows visible to the query as one window:


```python
SELECT
    sample_id,
    expression_value,
    AVG(expression_value) OVER () AS overall_mean
FROM expression;
```

Every row receives the overall mean.

## 15.4 PARTITION BY

`PARTITION BY` divides rows into independent analytical groups:


```python
SELECT
    sample_id,
    tissue,
    expression_value,
    AVG(expression_value) OVER (
        PARTITION BY tissue
    ) AS tissue_mean
FROM expression;
```

Unlike `GROUP BY`, individual rows are not collapsed.

## 15.5 ORDER BY Inside OVER()

Window `ORDER BY` defines the sequence used by the analytical function:


```python
SELECT
    gene_id,
    expression_value,
    ROW_NUMBER() OVER (
        ORDER BY expression_value DESC
    ) AS expression_position
FROM expression_by_tissue;
```

Window `ORDER BY` and the final query `ORDER BY` have different purposes. The first controls the calculation; the second controls presentation.

# Ranking Functions

## 15.6 ROW_NUMBER()

`ROW_NUMBER()` assigns a unique sequential integer:


```python
SELECT
    tissue,
    gene_id,
    expression_value,
    ROW_NUMBER() OVER (
        PARTITION BY tissue
        ORDER BY expression_value DESC, gene_id
    ) AS position_within_tissue
FROM expression_by_tissue;
```

Numbering restarts for each tissue. Equal expression values still receive different row numbers.

A tie-breaker such as `gene_id` makes the result deterministic.

## 15.7 RANK()

`RANK()` gives tied values the same rank and leaves gaps afterward:


```python
SELECT
    gene_id,
    expression_value,
    RANK() OVER (
        ORDER BY expression_value DESC
    ) AS expression_rank
FROM expression_by_tissue;
```

Example:

```text
100 → 1
 90 → 2
 90 → 2
 80 → 4
```

## 15.8 DENSE_RANK()

`DENSE_RANK()` gives ties the same rank without leaving gaps:


```python
SELECT
    gene_id,
    expression_value,
    DENSE_RANK() OVER (
        ORDER BY expression_value DESC
    ) AS expression_rank
FROM expression_by_tissue;
```

Example:

```text
100 → 1
 90 → 2
 90 → 2
 80 → 3
```

### ROW_NUMBER vs RANK vs DENSE_RANK

| Function | Ties share rank? | Gaps? | Unique sequence? |
|---|---:|---:|---:|
| `ROW_NUMBER()` | No | No | Yes |
| `RANK()` | Yes | Yes | No |
| `DENSE_RANK()` | Yes | No | No |

The choice is semantic. If biologically equal values should share rank, do not use `ROW_NUMBER()` merely for convenience.

## 15.9 Top-N Per Group

A classic problem is finding the top three genes in every tissue:


```python
WITH ranked_genes AS (
    SELECT
        tissue,
        gene_id,
        expression_value,
        ROW_NUMBER() OVER (
            PARTITION BY tissue
            ORDER BY expression_value DESC, gene_id
        ) AS rn
    FROM expression_by_tissue
)
SELECT *
FROM ranked_genes
WHERE rn <= 3
ORDER BY tissue, rn;
```

This is a major reason CTEs were taught before window functions.

## 15.10 NTILE()

`NTILE(n)` divides ordered rows into approximately equal groups:


```python
SELECT
    tissue,
    gene_id,
    expression_value,
    NTILE(4) OVER (
        PARTITION BY tissue
        ORDER BY expression_value
    ) AS tissue_expression_quartile
FROM expression_by_tissue;
```

Common applications include quartiles, deciles, score strata, and risk groups.

# Offset Functions

## 15.11 LAG()

`LAG()` accesses a preceding row without a self-join:


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    LAG(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
    ) AS previous_value
FROM longitudinal_biomarkers;
```

### Change from Previous Visit


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    biomarker_value -
        LAG(biomarker_value) OVER (
            PARTITION BY patient_id
            ORDER BY visit_date
        ) AS change_from_previous
FROM longitudinal_biomarkers;
```

## 15.12 Genomic LAG Example

Order variants by chromosome and genomic position:


```python
SELECT
    chromosome,
    position,
    variant_id,
    position - LAG(position) OVER (
        PARTITION BY chromosome
        ORDER BY position
    ) AS distance_from_previous_variant
FROM variants;
```

This calculates distance from the previous variant on the same chromosome.

## 15.13 LEAD()

`LEAD()` looks forward rather than backward:


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    LEAD(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
    ) AS next_value
FROM longitudinal_biomarkers;
```

## 15.14 Offsets and Defaults

The second argument specifies the offset; the third can specify a default:


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    LAG(biomarker_value, 2) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
    ) AS value_two_visits_ago
FROM longitudinal_biomarkers;
```

Do not use an arbitrary default such as zero for missing biological measurements unless zero has the correct scientific meaning.

# Value Functions

## 15.15 FIRST_VALUE()

`FIRST_VALUE()` can retrieve a baseline measurement:


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    FIRST_VALUE(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
    ) AS baseline_value
FROM longitudinal_biomarkers;
```

### Change from Baseline


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    biomarker_value -
        FIRST_VALUE(biomarker_value) OVER (
            PARTITION BY patient_id
            ORDER BY visit_date
        ) AS change_from_baseline
FROM longitudinal_biomarkers;
```

## 15.16 LAST_VALUE() and the Frame Trap

`LAST_VALUE()` returns the last value of the **current frame**, not automatically the final row of the partition.

To explicitly retrieve the final measurement:


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    LAST_VALUE(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND UNBOUNDED FOLLOWING
    ) AS final_measurement
FROM longitudinal_biomarkers;
```

This is one of the most important subtleties in window functions.

## 15.17 NTH_VALUE()

`NTH_VALUE()` retrieves the nth value in the applicable frame:


```python
SELECT
    tissue,
    gene_id,
    expression_value,
    NTH_VALUE(expression_value, 2) OVER (
        PARTITION BY tissue
        ORDER BY expression_value DESC
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND UNBOUNDED FOLLOWING
    ) AS second_ordered_value
FROM expression_by_tissue;
```

# Aggregate Window Functions

Ordinary aggregates such as `SUM`, `AVG`, `COUNT`, `MIN`, and `MAX` become window functions when used with `OVER()`.

## 15.18 Compare a Row with Its Group Mean


```python
SELECT
    tissue,
    gene_id,
    expression_value,
    AVG(expression_value) OVER (
        PARTITION BY tissue
    ) AS tissue_mean,
    expression_value -
        AVG(expression_value) OVER (
            PARTITION BY tissue
        ) AS difference_from_tissue_mean
FROM expression_by_tissue;
```

## 15.19 Windowed COUNT()


```python
SELECT
    sample_id,
    tissue,
    COUNT(*) OVER (
        PARTITION BY tissue
    ) AS samples_in_tissue
FROM samples;
```

## 15.20 Percentage of Total

Window functions can operate on already grouped results:


```python
SELECT
    tissue,
    COUNT(*) AS tissue_count,
    100.0 * COUNT(*) /
        SUM(COUNT(*)) OVER () AS percent_of_samples
FROM samples
GROUP BY tissue;
```

## 15.21 Running Total


```python
SELECT
    analysis_date,
    variants_processed,
    SUM(variants_processed) OVER (
        ORDER BY analysis_date
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS cumulative_variants
FROM analysis_progress;
```

## 15.22 Cumulative Variant Count


```python
SELECT
    chromosome,
    position,
    variant_id,
    COUNT(*) OVER (
        PARTITION BY chromosome
        ORDER BY position
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS cumulative_variant_count
FROM variants;
```

# Window Frames

A partition defines the broad analytical group. A **frame** defines the subset of rows within that partition used for the current calculation.

Important boundaries:

- `UNBOUNDED PRECEDING`
- `n PRECEDING`
- `CURRENT ROW`
- `n FOLLOWING`
- `UNBOUNDED FOLLOWING`

## 15.23 ROWS Frame: Moving Average


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    AVG(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_average_3_rows
FROM longitudinal_biomarkers;
```

This means current row plus up to two preceding rows.

## 15.24 Symmetric Local Genomic Window


```python
SELECT
    chromosome,
    position,
    signal_value,
    AVG(signal_value) OVER (
        PARTITION BY chromosome
        ORDER BY position
        ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
    ) AS local_average
FROM genomic_signal;
```

Row-based windows are only scientifically meaningful when row adjacency represents the intended biological neighborhood.

## 15.25 ROWS vs RANGE vs GROUPS

- `ROWS` uses row offsets.
- `RANGE` uses values of the ordering expression.
- `GROUPS` advances by peer groups.

They are not interchangeable.

### RANGE Example: Seven-Day Rolling Mean


```python
SELECT
    measurement_date,
    biomarker_value,
    AVG(biomarker_value) OVER (
        ORDER BY measurement_date
        RANGE BETWEEN INTERVAL '7 days' PRECEDING
                  AND CURRENT ROW
    ) AS rolling_7_day_mean
FROM daily_biomarkers;
```

Seven rows and seven days are not the same thing when observations are irregularly spaced.

## 15.26 Peer Rows

Rows equal according to the window `ORDER BY` expressions are **peers**.

Peer behavior affects:

- `RANK()`
- `DENSE_RANK()`
- `RANGE`
- `GROUPS`
- default frame behavior

## 15.27 Named Windows

Repeated window definitions can be named:


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    LAG(biomarker_value) OVER w AS previous_value,
    LEAD(biomarker_value) OVER w AS next_value,
    AVG(biomarker_value) OVER w AS running_mean
FROM longitudinal_biomarkers
WINDOW w AS (
    PARTITION BY patient_id
    ORDER BY visit_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
);
```

# Distribution Functions

## 15.28 PERCENT_RANK()

`PERCENT_RANK()` expresses relative rank from 0 to 1:


```python
SELECT
    gene_id,
    expression_value,
    PERCENT_RANK() OVER (
        ORDER BY expression_value
    ) AS relative_rank
FROM expression_by_tissue;
```

## 15.29 CUME_DIST()

`CUME_DIST()` gives the cumulative distribution position:


```python
SELECT
    gene_id,
    expression_value,
    CUME_DIST() OVER (
        ORDER BY expression_value
    ) AS cumulative_distribution
FROM expression_by_tissue;
```

# Combining Windows with Other SQL

## 15.30 Window Functions After GROUP BY


```python
SELECT
    tissue,
    AVG(expression_value) AS mean_expression,
    RANK() OVER (
        ORDER BY AVG(expression_value) DESC
    ) AS tissue_rank
FROM expression
GROUP BY tissue;
```

Aggregation occurs first; the resulting tissue rows are then ranked.

## 15.31 Ranking Mean Gene Expression Within Tissue


```python
WITH gene_tissue_summary AS (
    SELECT
        tissue,
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY tissue, gene_id
),
ranked AS (
    SELECT
        tissue,
        gene_id,
        mean_expression,
        DENSE_RANK() OVER (
            PARTITION BY tissue
            ORDER BY mean_expression DESC
        ) AS expression_rank
    FROM gene_tissue_summary
)
SELECT *
FROM ranked
WHERE expression_rank <= 10
ORDER BY tissue, expression_rank, gene_id;
```

## 15.32 Why Window Aliases Cannot Normally Be Filtered in WHERE

`WHERE` is logically evaluated before window calculations. Therefore calculate the window result in a CTE or subquery and filter it outside.

Simplified logical order:

```text
FROM / JOIN
↓
WHERE
↓
GROUP BY
↓
HAVING
↓
Window functions
↓
SELECT
↓
ORDER BY
↓
LIMIT
```

# Omics Applications

## 15.33 Variant Ranking Within Gene


```python
SELECT
    gene_symbol,
    variant_id,
    pathogenicity_score,
    RANK() OVER (
        PARTITION BY gene_symbol
        ORDER BY pathogenicity_score DESC NULLS LAST
    ) AS variant_rank
FROM variant_annotations;
```

## 15.34 Top Variant Per Gene


```python
WITH ranked_variants AS (
    SELECT
        gene_symbol,
        variant_id,
        pathogenicity_score,
        ROW_NUMBER() OVER (
            PARTITION BY gene_symbol
            ORDER BY pathogenicity_score DESC NULLS LAST,
                     variant_id
        ) AS rn
    FROM variant_annotations
)
SELECT *
FROM ranked_variants
WHERE rn = 1;
```

## 15.35 Sample QC Ranking Within Sequencing Batch


```python
SELECT
    sequencing_batch,
    sample_id,
    call_rate,
    RANK() OVER (
        PARTITION BY sequencing_batch
        ORDER BY call_rate DESC
    ) AS qc_rank_in_batch,
    AVG(call_rate) OVER (
        PARTITION BY sequencing_batch
    ) AS batch_mean_call_rate
FROM sample_qc;
```

## 15.36 Longitudinal Patient Analysis


```python
SELECT
    patient_id,
    visit_date,
    biomarker_value,
    FIRST_VALUE(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
    ) AS baseline,
    LAG(biomarker_value) OVER (
        PARTITION BY patient_id
        ORDER BY visit_date
    ) AS previous_value,
    biomarker_value -
        LAG(biomarker_value) OVER (
            PARTITION BY patient_id
            ORDER BY visit_date
        ) AS change_from_previous
FROM longitudinal_biomarkers;
```

## 15.37 Genomic Rolling Signal


```python
SELECT
    chromosome,
    position,
    signal_value,
    AVG(signal_value) OVER (
        PARTITION BY chromosome
        ORDER BY position
        ROWS BETWEEN 5 PRECEDING AND 5 FOLLOWING
    ) AS local_smoothed_signal
FROM genomic_signal;
```

# Performance Considerations

Window functions frequently require sorting. PostgreSQL execution plans may contain a `WindowAgg` node and one or more sorts.

Always measure rather than guessing:


```python
EXPLAIN (ANALYZE, BUFFERS)
SELECT
    tissue,
    gene_id,
    expression_value,
    RANK() OVER (
        PARTITION BY tissue
        ORDER BY expression_value DESC
    ) AS expression_rank
FROM expression_by_tissue;
```

An index whose leading columns align with partitioning and ordering may sometimes help:


```python
CREATE INDEX idx_expression_tissue_value
ON expression_by_tissue (
    tissue,
    expression_value DESC
);
```

Do not create an index merely because a window function exists. Evaluate actual query plans, table size, selectivity, sort behavior, and workload.

# Common Mistakes

## 15.38 Missing PARTITION BY

If you intend tissue-specific ranking but omit `PARTITION BY tissue`, you calculate a global rank.

## 15.39 Confusing Analytical ORDER BY with Display ORDER BY

Window `ORDER BY` defines the calculation. Final `ORDER BY` defines result presentation.

## 15.40 Using ROW_NUMBER When Ties Matter

Use `RANK()` or `DENSE_RANK()` when tied observations should share a scientific rank.

## 15.41 LAST_VALUE Without Understanding Frames

Always ask: **last value of which frame?**

## 15.42 Using ROWS for a Time or Genomic-Distance Question

Seven rows are not necessarily seven days. Five neighboring variants are not necessarily a fixed genomic distance.

## 15.43 Non-Deterministic Ties

Use stable tie-breakers when reproducibility matters.

## 15.44 Filtering Too Early

Filtering changes the rows available to the window and can therefore change ranks, means, distributions, and cumulative calculations.

## 15.45 Ignoring the Unit of Observation

Always establish whether each row represents one patient, visit, sample, gene, gene × tissue, variant, or genomic position before defining a window.

# Decision Framework

- Need one summary row per group? → `GROUP BY`
- Need group context while retaining rows? → window aggregate
- Need ranking? → `ROW_NUMBER`, `RANK`, `DENSE_RANK`
- Need quantile-like bins? → `NTILE`
- Need previous/next observations? → `LAG`, `LEAD`
- Need baseline/final observation? → `FIRST_VALUE`, `LAST_VALUE`
- Need cumulative statistics? → ordered aggregate window
- Need rolling statistics? → explicit `ROWS`, `RANGE`, or `GROUPS` frame
- Need to filter calculated rank? → CTE/subquery then filter

# 15.46 Integrated Omics Example

For every tissue:

1. calculate mean expression per gene;
2. calculate the tissue-wide mean of those gene means;
3. rank genes;
4. return the top five.



```python
WITH gene_tissue_summary AS (
    SELECT
        tissue,
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY tissue, gene_id
),
analytical_results AS (
    SELECT
        tissue,
        gene_id,
        mean_expression,
        AVG(mean_expression) OVER (
            PARTITION BY tissue
        ) AS tissue_mean_of_gene_means,
        DENSE_RANK() OVER (
            PARTITION BY tissue
            ORDER BY mean_expression DESC
        ) AS expression_rank
    FROM gene_tissue_summary
)
SELECT
    tissue,
    gene_id,
    mean_expression,
    tissue_mean_of_gene_means,
    mean_expression - tissue_mean_of_gene_means
        AS difference_from_tissue_mean,
    expression_rank
FROM analytical_results
WHERE expression_rank <= 5
ORDER BY tissue, expression_rank, gene_id;
```

# Chapter 15 Summary

In this chapter, we learned:

- Window functions retain row-level detail while adding analytical calculations.
- `OVER()` creates a window calculation.
- `PARTITION BY` defines analytical groups.
- Window `ORDER BY` defines analytical sequence.
- `ROW_NUMBER`, `RANK`, and `DENSE_RANK` provide different ranking semantics.
- `NTILE` creates approximately equal ordered groups.
- `LAG` and `LEAD` access preceding and following rows.
- `FIRST_VALUE`, `LAST_VALUE`, and `NTH_VALUE` retrieve ordered values.
- Aggregate functions can operate as window functions.
- Window frames support cumulative and rolling calculations.
- `ROWS`, `RANGE`, and `GROUPS` are different frame models.
- `PERCENT_RANK` and `CUME_DIST` describe relative distribution.
- CTEs are natural partners for window functions.
- Window functions are highly applicable to gene ranking, variant prioritization, QC, genomic positions, and longitudinal data.
- Performance should be evaluated with `EXPLAIN (ANALYZE, BUFFERS)`.

# Key Takeaway

> **GROUP BY summarizes groups; window functions analyze groups without losing the rows.**

Mastering windows is a major transition from intermediate SQL to advanced analytical SQL.

# References

1. PostgreSQL Global Development Group. **Window Functions**  
   https://www.postgresql.org/docs/current/functions-window.html

2. PostgreSQL Global Development Group. **Window Function Calls**  
   https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS

3. PostgreSQL Global Development Group. **Tutorial: Window Functions**  
   https://www.postgresql.org/docs/current/tutorial-window.html

4. PostgreSQL Global Development Group. **SELECT**  
   https://www.postgresql.org/docs/current/sql-select.html

5. PostgreSQL Global Development Group. **Aggregate Functions**  
   https://www.postgresql.org/docs/current/functions-aggregate.html

6. PostgreSQL Global Development Group. **WITH Queries**  
   https://www.postgresql.org/docs/current/queries-with.html

7. PostgreSQL Global Development Group. **Using EXPLAIN**  
   https://www.postgresql.org/docs/current/using-explain.html

8. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

9. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

10. Ensembl Documentation  
    https://www.ensembl.org/info/docs/index.html

11. GTEx Portal  
    https://gtexportal.org/

12. gnomAD  
    https://gnomad.broadinstitute.org/

