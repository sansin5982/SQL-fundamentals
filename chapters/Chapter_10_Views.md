# Views and Materialized Views
### CREATE VIEW, Updatable Views, Security, Reusable Analytical Logic, Materialized Views, Refreshing, Performance, and Omics Applications in PostgreSQL

---

## Introduction: Why Views Matter

As databases grow, SQL queries can become long, repetitive, and difficult to maintain.

For example, an analyst may repeatedly need to:

- Join patients to samples
- Filter only QC-passed samples
- Hide sensitive identifiers
- Calculate mean gene expression
- Summarize expression by tissue
- Reuse the same analytical logic across many reports

Instead of rewriting the same query again and again, PostgreSQL allows us to store the **query definition** as a database object called a **view**.

A view behaves like a table when queried, but in most cases it does not store its own copy of the underlying data.

A **materialized view**, by contrast, stores the result physically and can therefore improve performance for expensive analytical queries.

Views are therefore important for:

- Reusability
- Abstraction
- Security
- Simplification
- Reporting
- Analytical consistency
- Performance optimization when materialized


## 9.1 What Is a View?

A **view** is a named SQL query stored in the database.

It presents the result of that query as though it were a table.

Conceptually:

```text
Underlying tables
       ↓
Stored SELECT query
       ↓
View
       ↓
Users query the view
```

A standard view is often described as a **virtual table** because its result is generated from the underlying tables when queried.


### Basic Syntax



```python
CREATE VIEW view_name AS
SELECT ...
FROM ...;
```

Once created, the view can be queried using ordinary SQL:



```python
SELECT *
FROM view_name;
```

The user does not need to rewrite the original query each time.


## 9.2 Why Do We Need Views?

Views solve several common database problems.

### Reusability

Store commonly used query logic once and reuse it.

### Simplification

Hide complicated joins or calculations behind a simple object name.

### Security

Expose only approved columns or rows.

### Abstraction

Users do not need to understand the full physical schema.

### Consistency

Different analysts can use the same standardized definition.

### Maintainability

Business or scientific logic can be updated centrally.


## 9.3 Layman Analogy

Imagine a large spreadsheet workbook with many hidden sheets.

A view is like creating a clean summary sheet that shows only the information a particular user needs.

The user sees:

```text
sample_id
tissue
qc_status
```

without needing to understand how those values were assembled from multiple underlying tables.


## 9.4 Example Biomedical Schema

We will use:

```text
patients
samples
genes
expression
```

Typical relationships:

```text
patients
   ↓
samples
   ↓
expression
   ↑
genes
```

Assume `samples` also contains:

```text
qc_status
```


## 9.5 Creating a Simple View

Suppose analysts frequently need only QC-passed samples.



```python
CREATE VIEW passed_samples AS
SELECT
    sample_id,
    patient_id,
    tissue
FROM samples
WHERE qc_status = 'Pass';
```

Now the same result can be retrieved with:



```python
SELECT *
FROM passed_samples;
```

The view stores the query definition, not a separate permanent copy of those rows.


## 9.6 Why This Helps

Without the view, every analyst must remember:



```python
SELECT
    sample_id,
    patient_id,
    tissue
FROM samples
WHERE qc_status = 'Pass';
```

With the view:



```python
SELECT *
FROM passed_samples;
```

This reduces duplicated SQL and the risk that different users apply inconsistent filters.


## 9.7 Views Are Dynamically Based on Underlying Data

Suppose a sample changes from:

```text
qc_status = 'Pending'
```

to:

```text
qc_status = 'Pass'
```

The next time the standard view is queried, the result reflects the new underlying data.

This is an important distinction between a regular view and a materialized view.


## 9.8 View Built from a JOIN

Views become especially useful when they hide complicated joins.

Suppose we repeatedly need patient and sample information together.



```python
CREATE VIEW patient_sample_metadata AS
SELECT
    p.patient_id,
    p.sex,
    p.age,
    p.diagnosis,
    s.sample_id,
    s.tissue,
    s.qc_status
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id;
```

Now:



```python
SELECT *
FROM patient_sample_metadata;
```

Users can work with one logical object rather than repeatedly rebuilding the join.


## 9.9 Clinical Filtering Through a View

Because a view behaves like a queryable relation, we can still filter it.



```python
SELECT
    patient_id,
    sample_id,
    tissue
FROM patient_sample_metadata
WHERE diagnosis = 'Type 2 Diabetes';
```

PostgreSQL can integrate the outer filter with the view definition during query planning.


## 9.10 Omics Example: Gene Expression View

Suppose we want gene symbols beside expression measurements.



```python
CREATE VIEW gene_expression_details AS
SELECT
    e.sample_id,
    e.gene_id,
    g.gene_symbol,
    g.chromosome,
    e.expression_value
FROM expression AS e
JOIN genes AS g
    ON e.gene_id = g.gene_id;
```

Then:



```python
SELECT *
FROM gene_expression_details
WHERE expression_value > 20;
```

This improves readability for users who care about biological interpretation rather than schema mechanics.


## 9.11 Views Built from Aggregation

A view can also contain aggregate calculations.

Example:



```python
CREATE VIEW gene_expression_summary AS
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression,
    COUNT(*) AS measurement_count,
    MIN(expression_value) AS minimum_expression,
    MAX(expression_value) AS maximum_expression
FROM expression
GROUP BY gene_id;
```

Now:



```python
SELECT *
FROM gene_expression_summary
ORDER BY mean_expression DESC;
```

This gives analysts a reusable gene-level summary.


## 9.12 Views for Tissue-Level Omics Analysis

Suppose samples contain tissue information.

We can create:



```python
CREATE VIEW tissue_gene_expression_summary AS
SELECT
    s.tissue,
    e.gene_id,
    AVG(e.expression_value) AS mean_expression,
    COUNT(*) AS n_measurements
FROM expression AS e
JOIN samples AS s
    ON e.sample_id = s.sample_id
GROUP BY
    s.tissue,
    e.gene_id;
```

This view can then answer:



```python
SELECT *
FROM tissue_gene_expression_summary
WHERE tissue = 'Blood'
ORDER BY mean_expression DESC;
```

This resembles a simplified GTEx-style analytical layer.


## 9.13 Column Aliases in Views

Good column names are important because the view becomes an interface for other users.

Example:



```python
CREATE VIEW patient_age_summary AS
SELECT
    diagnosis,
    AVG(age) AS mean_age,
    COUNT(*) AS patient_count
FROM patients
GROUP BY diagnosis;
```

Aliases such as:

```text
mean_age
patient_count
```

make the output self-documenting.


## 9.14 Replacing a View

PostgreSQL supports:

```sql
CREATE OR REPLACE VIEW
```

This lets us modify a view definition without necessarily dropping the view first.



```python
CREATE OR REPLACE VIEW passed_samples AS
SELECT
    sample_id,
    patient_id,
    tissue,
    qc_status
FROM samples
WHERE qc_status = 'Pass';
```

This is useful when the view logic evolves.

However, PostgreSQL imposes compatibility rules on the existing view columns, so structural changes may sometimes require dropping and recreating the view.


## 9.15 Renaming a View

A view can be renamed using `ALTER VIEW`.



```python
ALTER VIEW passed_samples
RENAME TO qc_passed_samples;
```

Renaming may improve naming conventions without redefining the logic.


## 9.16 Dropping a View

To remove a view:



```python
DROP VIEW qc_passed_samples;
```

To avoid an error if the view does not exist:



```python
DROP VIEW IF EXISTS qc_passed_samples;
```

Dropping the view does not delete rows from the underlying base tables.


## 9.17 Dependencies

Views depend on the database objects referenced by their definitions.

For example:

```text
patient_sample_metadata
```

depends on:

```text
patients
samples
```

PostgreSQL tracks these dependencies.

If an operation would break the dependent view, PostgreSQL may block the change unless the dependency is handled explicitly.


## 9.18 CASCADE When Dropping Objects

PostgreSQL can use:



```python
DROP TABLE samples CASCADE;
```

But this may also remove dependent objects such as views.

`CASCADE` is powerful and should be used carefully.

The safer question is:

> What depends on this object, and should those objects really be removed?


## 9.19 Simple vs Complex Views

A **simple view** may directly expose columns from one table.

Example:



```python
CREATE VIEW active_samples AS
SELECT
    sample_id,
    patient_id,
    tissue
FROM samples
WHERE is_active = TRUE;
```

A **complex view** may contain:

- Multiple joins
- Aggregation
- GROUP BY
- Subqueries
- Calculated columns
- Conditional logic

Example:



```python
CREATE VIEW disease_tissue_summary AS
SELECT
    p.diagnosis,
    s.tissue,
    COUNT(*) AS sample_count
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
GROUP BY
    p.diagnosis,
    s.tissue;
```

Complex views are usually more analytical and less likely to be directly updatable.


## 9.20 Updatable Views

Some PostgreSQL views are automatically updatable.

This means you may be able to run:



```python
UPDATE some_view
SET ...
WHERE ...;
```

and PostgreSQL propagates the change to the underlying base table.

Automatically updatable views generally need to be relatively simple, typically based on a single table without features such as grouping, aggregation, or set operations.


## 9.21 Example of an Updatable View

Suppose:



```python
CREATE VIEW patient_basic_info AS
SELECT
    patient_id,
    diagnosis
FROM patients;
```

Then this may be allowed:



```python
UPDATE patient_basic_info
SET diagnosis = 'Hypertension'
WHERE patient_id = 104;
```

The underlying `patients` table is updated.


## 9.22 Views That Are Not Automatically Updatable

A view containing aggregation such as:



```python
CREATE VIEW gene_expression_summary AS
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id;
```

cannot normally be updated directly in the obvious way because one summary row represents many underlying rows.

For example:

```text
mean_expression = 20
```

does not tell PostgreSQL which individual measurements should be changed.


## 9.23 WITH CHECK OPTION

A view can restrict which rows are visible.

`WITH CHECK OPTION` ensures that inserts or updates made through the view remain consistent with the view's filtering condition.

Example:



```python
CREATE VIEW blood_samples AS
SELECT
    sample_id,
    patient_id,
    tissue
FROM samples
WHERE tissue = 'Blood'
WITH CHECK OPTION;
```

Now an attempt to change a row through the view so that:



```python
tissue = 'Liver'
```

would be rejected because the row would no longer satisfy:

```sql
WHERE tissue = 'Blood'
```

This helps keep updates through filtered views logically consistent.


## 9.24 Views for Security

Views can expose only approved columns.

Suppose the `patients` table contains:

```text
patient_id
name
email
sex
age
diagnosis
```

Researchers may not need direct identifiers.

We can expose:



```python
CREATE VIEW research_patient_data AS
SELECT
    patient_id,
    sex,
    age,
    diagnosis
FROM patients;
```

Then appropriate permissions can be granted on the view rather than the full base table.

This creates a **controlled data-access layer**.

Important: a view alone is not a complete privacy solution. Proper permissions, de-identification, governance, and policy controls are also required.


## 9.25 Hiding Sensitive Columns

A view is often used to hide columns such as:

- Names
- Email addresses
- Direct identifiers
- Administrative notes
- Internal audit fields

while exposing only analysis-relevant variables.


## 9.26 Views as an Abstraction Layer

Suppose the physical database schema is complicated.

Researchers might need to join:

```text
patients
samples
sample_types
tissues
assays
genes
expression
```

A carefully designed view can hide that complexity and present a stable analytical interface.

Conceptually:

```text
Complex physical schema
        ↓
Curated view
        ↓
Simple researcher-facing table
```


## 9.27 Views and Reproducibility

A standardized view can encode a shared scientific definition.

For example:

```text
analysis_ready_samples
```

might define:

- QC passed
- consent valid
- required tissue present
- sequencing complete

If all analysts use the same view, cohort selection becomes more consistent and reproducible.


## 9.28 Example: Analysis-Ready Samples View



```python
CREATE VIEW analysis_ready_samples AS
SELECT
    sample_id,
    patient_id,
    tissue
FROM samples
WHERE qc_status = 'Pass'
  AND sequencing_status = 'Complete'
  AND consent_status = 'Active';
```

This view becomes a reusable cohort definition.


## 9.29 View vs Table

A **table** stores rows physically.

A regular **view** stores a query definition.

| Feature | Table | View |
|---|---|---|
| Stores data directly | Yes | Usually no |
| Stores query definition | No | Yes |
| Can be indexed directly | Yes | No, not like a table |
| Reflects base-table changes automatically | N/A | Yes |
| Useful for abstraction | Sometimes | Yes |
| Useful for security layers | Sometimes | Yes |


## 9.30 View vs CTE

A **CTE** created using `WITH` exists only for a single SQL statement.

A **view** persists as a database object until dropped.

Example CTE:



```python
WITH gene_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM gene_summary;
```

Equivalent reusable view:



```python
CREATE VIEW gene_summary AS
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id;
```

Then:



```python
SELECT *
FROM gene_summary;
```

### Rule of Thumb

Use a CTE when the logic is temporary and query-specific.

Use a view when the logic should become a reusable database interface.


# 9.31 Materialized Views

A **materialized view** stores the result of a query physically.

This is the major difference from a standard view.

Conceptually:

```text
Standard view
Query definition
      ↓
Run underlying query each time

Materialized view
Query definition
      ↓
Stored result
      ↓
Query stored rows
```

Materialized views can dramatically improve performance for expensive analytical queries.


## 9.32 Why Materialized Views Are Needed

Imagine a query that:

- Joins millions of expression records
- Groups by tissue and gene
- Calculates several aggregates

Running the full query repeatedly may be expensive.

A materialized view can calculate the result once and store it.


## 9.33 Creating a Materialized View



```python
CREATE MATERIALIZED VIEW tissue_gene_expression_mv AS
SELECT
    s.tissue,
    e.gene_id,
    AVG(e.expression_value) AS mean_expression,
    COUNT(*) AS n_measurements
FROM expression AS e
JOIN samples AS s
    ON e.sample_id = s.sample_id
GROUP BY
    s.tissue,
    e.gene_id;
```

Now PostgreSQL physically stores the result.


## 9.34 Querying a Materialized View



```python
SELECT *
FROM tissue_gene_expression_mv
WHERE tissue = 'Blood'
ORDER BY mean_expression DESC;
```

This may be much faster than recomputing the full join and aggregation every time.


## 9.35 The Major Trade-Off: Stale Data

A standard view reflects current base-table data when queried.

A materialized view does **not** automatically refresh every time underlying tables change.

Therefore, its data can become stale.


## 9.36 Refreshing a Materialized View

Use:



```python
REFRESH MATERIALIZED VIEW tissue_gene_expression_mv;
```

PostgreSQL reruns the underlying query and replaces the stored result with fresh data.


## 9.37 When Should You Refresh?

Possible strategies include:

- After every data load
- Nightly
- Weekly
- Before a report
- After a sequencing batch is finalized

The correct refresh strategy depends on:

- Data-update frequency
- Performance requirements
- Freshness requirements
- Reporting schedule


## 9.38 REFRESH MATERIALIZED VIEW CONCURRENTLY

PostgreSQL supports:



```python
REFRESH MATERIALIZED VIEW CONCURRENTLY tissue_gene_expression_mv;
```

A concurrent refresh allows reads to continue while the refresh runs, subject to PostgreSQL requirements.

A suitable unique index is required on the materialized view for `CONCURRENTLY`.


## 9.39 Creating an Index on a Materialized View

Unlike ordinary views, materialized views can have indexes.



```python
CREATE UNIQUE INDEX idx_tissue_gene_expression_mv
ON tissue_gene_expression_mv (tissue, gene_id);
```

This can support both:

- Concurrent refresh requirements
- Faster lookups


## 9.40 Additional Index Example



```python
CREATE INDEX idx_tissue_gene_expression_mean
ON tissue_gene_expression_mv (mean_expression);
```

Whether such an index is useful depends on real query patterns and should be evaluated with query plans.


## 9.41 Standard View vs Materialized View

| Feature | View | Materialized View |
|---|---|---|
| Stores query definition | Yes | Yes |
| Stores result physically | No | Yes |
| Reflects changes automatically | Yes | No |
| Requires refresh | No | Yes |
| Can have indexes | Not directly | Yes |
| Good for expensive summaries | Sometimes | Often |
| Risk of stale data | No | Yes |


## 9.42 Materialized View Omics Example

Suppose we have tens of millions of expression measurements.

Researchers repeatedly ask:

> What is mean expression per gene per tissue?

Instead of recalculating:



```python
SELECT
    s.tissue,
    e.gene_id,
    AVG(e.expression_value)
FROM expression AS e
JOIN samples AS s
    ON e.sample_id = s.sample_id
GROUP BY
    s.tissue,
    e.gene_id;
```

every time, a materialized view can cache the summary.


## 9.43 Materialized View for Variant Burden

A genomics database might create:



```python
CREATE MATERIALIZED VIEW gene_variant_burden_mv AS
SELECT
    gene_id,
    COUNT(*) AS variant_count,
    COUNT(*) FILTER (
        WHERE consequence = 'missense_variant'
    ) AS missense_count
FROM variants
GROUP BY gene_id;
```

This can support dashboards or reports that repeatedly query gene-level variant burden.


## 9.44 Views and Query Performance

A standard view is not automatically a performance optimization.

PostgreSQL usually expands the view definition into the surrounding query and plans the complete statement.

Therefore, a complicated view may still execute a complicated underlying query.


## 9.45 Materialized Views as a Performance Strategy

Materialized views can improve performance when:

- Underlying queries are expensive
- Data change less frequently than reads occur
- Summaries are reused often
- Some staleness is acceptable
- Refresh scheduling is manageable


## 9.46 When Materialized Views May Be a Poor Choice

Avoid or reconsider them when:

- Data must always be real-time
- Underlying data change constantly
- Refresh cost is extremely high
- The summary is rarely queried
- Storage duplication is undesirable


## 9.47 Views and Permissions

PostgreSQL permissions can be granted on views.

Conceptually:



```python
GRANT SELECT
ON research_patient_data
TO analyst_role;
```

The analyst can be given access to the curated view without necessarily receiving direct access to all underlying patient columns.

Security design requires careful role and privilege management.


## 9.48 Common Beginner Mistakes

### Mistake 1 — Thinking a View Stores a Copy of Data

A normal view stores query logic, not a separate result set.

### Mistake 2 — Assuming Views Are Always Faster

A standard view may execute the same expensive logic as the underlying query.

### Mistake 3 — Forgetting Materialized Views Become Stale

Refresh them when required.

### Mistake 4 — Trying to Update an Aggregate View

Aggregated views are generally not directly updatable.

### Mistake 5 — Using SELECT * in Long-Lived Views

Explicit columns are safer because schema changes are easier to control.

### Mistake 6 — Treating Views as a Complete Security Solution

Views must be combined with proper privileges and governance.

### Mistake 7 — Using CASCADE Carelessly

Dropping a table with `CASCADE` may remove dependent views.


## 9.49 Best Practice: Explicit Columns

Prefer:



```python
CREATE VIEW passed_samples AS
SELECT
    sample_id,
    patient_id,
    tissue,
    qc_status
FROM samples
WHERE qc_status = 'Pass';
```

rather than:



```python
CREATE VIEW passed_samples AS
SELECT *
FROM samples
WHERE qc_status = 'Pass';
```

Explicit columns make the interface more stable and intentional.


## 9.50 Best Practice: Descriptive Names

Good names communicate purpose.

Examples:

```text
qc_passed_samples
research_patient_data
gene_expression_summary
tissue_gene_expression_mv
analysis_ready_samples
```

Avoid vague names such as:

```text
view1
temp_view
data_view
```


## 9.51 Best Practice: Document the View Logic

A view can encode important scientific definitions.

Document:

- Purpose
- Source tables
- Inclusion criteria
- Exclusion criteria
- Refresh policy for materialized views
- Expected unit of observation

This is especially important in reproducible research.


## 9.52 Unit of Observation in Views

Always understand what one row represents.

Examples:

```text
one patient
one sample
one gene
one gene × tissue summary
one gene × sample measurement
```

Misunderstanding the unit of observation can lead to incorrect counts and averages.


## 9.53 Full Clinical View Example



```python
CREATE VIEW analysis_patient_samples AS
SELECT
    p.patient_id,
    p.sex,
    p.age,
    p.diagnosis,
    s.sample_id,
    s.tissue,
    s.qc_status
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
WHERE s.qc_status = 'Pass';
```

This creates a reusable analysis layer containing only valid samples.


## 9.54 Full Omics View Example



```python
CREATE VIEW analysis_expression AS
SELECT
    p.patient_id,
    p.diagnosis,
    s.sample_id,
    s.tissue,
    g.gene_id,
    g.gene_symbol,
    e.expression_value
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
JOIN expression AS e
    ON s.sample_id = e.sample_id
JOIN genes AS g
    ON e.gene_id = g.gene_id
WHERE s.qc_status = 'Pass';
```

Now analysts can ask:



```python
SELECT
    tissue,
    gene_symbol,
    AVG(expression_value) AS mean_expression
FROM analysis_expression
WHERE diagnosis = 'Type 2 Diabetes'
GROUP BY
    tissue,
    gene_symbol
ORDER BY
    tissue,
    mean_expression DESC;
```

The complicated relational logic is encapsulated in the view.


## 9.55 Full Materialized Omics Summary



```python
CREATE MATERIALIZED VIEW disease_tissue_gene_summary_mv AS
SELECT
    p.diagnosis,
    s.tissue,
    g.gene_id,
    g.gene_symbol,
    AVG(e.expression_value) AS mean_expression,
    COUNT(*) AS n_measurements
FROM patients AS p
JOIN samples AS s
    ON p.patient_id = s.patient_id
JOIN expression AS e
    ON s.sample_id = e.sample_id
JOIN genes AS g
    ON e.gene_id = g.gene_id
WHERE s.qc_status = 'Pass'
GROUP BY
    p.diagnosis,
    s.tissue,
    g.gene_id,
    g.gene_symbol;
```

This could support repeated analytical queries and dashboards.


## 9.56 Refreshing After New Data Load



```python
REFRESH MATERIALIZED VIEW disease_tissue_gene_summary_mv;
```

A production workflow may refresh materialized summaries after a successful ETL load.


## 9.57 View Design Decision Framework

Ask:

### Do I repeatedly use the same query logic?

Use a view.

### Do I want to hide schema complexity?

Use a view.

### Do I want to expose only approved columns or rows?

Use a view plus proper permissions.

### Is the query expensive and reused frequently?

Consider a materialized view.

### Must results always reflect the latest base-table data?

Prefer a standard view or direct query.

### Can slightly stale data be tolerated?

A materialized view may be appropriate.

### Is the logic needed only once?

Use a CTE or subquery rather than creating a permanent view.


# Module 9 Summary

In this module, we learned that views create reusable logical interfaces over relational data.

Key concepts include:

- A **view** is a stored query definition.
- A regular view behaves like a virtual table.
- Views simplify repeated SQL logic.
- Views can hide complex joins and calculations.
- Views can support security and abstraction.
- Views can expose only selected columns and rows.
- `CREATE VIEW` creates a standard view.
- `CREATE OR REPLACE VIEW` can update compatible view definitions.
- `ALTER VIEW` can rename a view.
- `DROP VIEW` removes the view, not the underlying data.
- PostgreSQL tracks dependencies between views and base objects.
- Some simple views are automatically updatable.
- Aggregate or complex views are usually not directly updatable.
- `WITH CHECK OPTION` helps enforce a filtered view's condition during writes through the view.
- A **materialized view** stores query results physically.
- Materialized views can improve repeated analytical query performance.
- Materialized views can become stale.
- `REFRESH MATERIALIZED VIEW` updates stored results.
- `REFRESH MATERIALIZED VIEW CONCURRENTLY` can allow continued reads when requirements are satisfied.
- Materialized views can have indexes.
- Standard views are not automatically faster than direct queries.
- View design should always consider security, maintainability, data freshness, and the unit of observation.


# Key Takeaway

> **A view is a reusable SQL interface; a materialized view is a reusable stored result.**

For biomedical and omics databases, views can transform a complicated schema such as:

```text
Patient
  ↓
Sample
  ↓
Expression
  ↓
Gene
```

into simple analysis-ready objects such as:

```text
analysis_expression
gene_expression_summary
qc_passed_samples
```

Materialized views go one step further by physically storing expensive summaries such as:

```text
gene × tissue mean expression
gene-level variant burden
disease × tissue × gene summaries
```

This makes views one of the most important tools for building clean, reusable, and scalable analytical SQL systems.


# References

1. PostgreSQL Global Development Group. **PostgreSQL Documentation — CREATE VIEW**  
   https://www.postgresql.org/docs/current/sql-createview.html

2. PostgreSQL Global Development Group. **PostgreSQL Documentation — CREATE MATERIALIZED VIEW**  
   https://www.postgresql.org/docs/current/sql-creatematerializedview.html

3. PostgreSQL Global Development Group. **PostgreSQL Documentation — REFRESH MATERIALIZED VIEW**  
   https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html

4. PostgreSQL Global Development Group. **PostgreSQL Documentation — ALTER VIEW**  
   https://www.postgresql.org/docs/current/sql-alterview.html

5. PostgreSQL Global Development Group. **PostgreSQL Documentation — DROP VIEW**  
   https://www.postgresql.org/docs/current/sql-dropview.html

6. PostgreSQL Global Development Group. **PostgreSQL Documentation — Privileges**  
   https://www.postgresql.org/docs/current/ddl-priv.html

7. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

8. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

9. Date, C. J. *An Introduction to Database Systems*. Addison-Wesley.

10. Ensembl Documentation  
    https://www.ensembl.org/info/docs/index.html

11. GTEx Portal  
    https://gtexportal.org/

