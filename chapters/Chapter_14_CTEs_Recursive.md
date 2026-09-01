# Chapter 14: Common Table Expressions (CTEs) and Recursive CTEs
### WITH, Multiple CTEs, Chained Queries, Recursive Queries, Materialization Behavior, Data-Modifying CTEs, and Omics Applications in PostgreSQL

---

## Introduction: Why CTEs Matter

As SQL queries become more advanced, they can become difficult to read.

A single query may contain:

- Multiple joins
- Subqueries
- Aggregations
- Filters
- Ranking logic
- Reused intermediate calculations
- Hierarchical relationships

A **Common Table Expression**, usually abbreviated as **CTE**, allows us to break a complex query into smaller, named logical steps.

A CTE is introduced using the:

```sql
WITH
```

clause.

Conceptually:

```text
Complex query
    ↓
Break into logical steps
    ↓
Name each step
    ↓
Use those steps in the final query
```

CTEs are therefore extremely important for:

- Readability
- Modularity
- Query organization
- Reusable intermediate results
- Hierarchical queries
- Recursive data traversal
- Advanced analytical SQL


# 14.1 What Is a CTE?

A **Common Table Expression** is a temporary named result set defined within a single SQL statement.

A CTE exists only for the duration of that statement.

Basic syntax:



```python
WITH cte_name AS (
    SELECT ...
)
SELECT *
FROM cte_name;
```

The query inside the parentheses creates the CTE.

The final query can then use the CTE as though it were a temporary table.

Important:

> A CTE is not a permanent table and not a permanent view.

It exists only while the statement is being executed.


# 14.2 Layman Analogy

Imagine solving a complex mathematical problem.

Instead of solving everything in one line, you write:

```text
Step 1 = calculate A
Step 2 = calculate B using A
Step 3 = calculate final answer using B
```

A CTE allows SQL to work in the same structured way.

Instead of one giant nested query:

```text
query inside query
inside another query
inside another query
```

we can create named stages.


# 14.3 Basic Clinical Example

Suppose we want to identify patients older than 50.

Without a CTE:



```python
SELECT
    patient_id,
    age,
    diagnosis
FROM patients
WHERE age > 50;
```

Using a CTE:



```python
WITH older_patients AS (
    SELECT
        patient_id,
        age,
        diagnosis
    FROM patients
    WHERE age > 50
)
SELECT *
FROM older_patients;
```

This simple example does not require a CTE, but it demonstrates the structure.

The real value appears when the intermediate step is more complicated.


# 14.4 Why Use a CTE Instead of a Subquery?

Consider this query:



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

The same logic using a CTE:



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

The CTE version separates the logic into:

```text
Step 1 → calculate gene-level mean expression
Step 2 → filter genes
```

This is easier to read and maintain.


# 14.5 CTE vs Derived Table

A subquery inside `FROM` is often called a **derived table**.

Example:



```python
SELECT *
FROM (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
) AS x;
```

CTE version:



```python
WITH x AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM x;
```

Both can represent an intermediate result.

The main difference is organization and scope.

A CTE gives the intermediate result a clear name before the main query begins.


# 14.6 CTE Scope

A CTE exists only inside the SQL statement in which it is defined.

For example:



```python
WITH diabetic_patients AS (
    SELECT *
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
)
SELECT *
FROM diabetic_patients;
```

After this statement finishes, you cannot run:



```python
SELECT *
FROM diabetic_patients;
```

unless the CTE is defined again.

This differs from a view, which remains in the database until dropped.


# 14.7 CTE vs View

| Feature | CTE | View |
|---|---|---|
| Exists temporarily | Yes | No |
| Persists in database | No | Yes |
| Used within one statement | Yes | No |
| Useful for complex query organization | Yes | Yes |
| Reusable across separate queries | No | Yes |
| Created using | `WITH` | `CREATE VIEW` |

Use a CTE for query-local logic.

Use a view when the logic should become a reusable database object.


# 14.8 Omics Example: Gene Expression Summary

Question:

> Calculate mean expression for each gene and return genes with mean expression greater than 20.



```python
WITH gene_expression_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression,
        COUNT(*) AS n_measurements
    FROM expression
    GROUP BY gene_id
)
SELECT
    gene_id,
    mean_expression,
    n_measurements
FROM gene_expression_summary
WHERE mean_expression > 20
ORDER BY mean_expression DESC;
```

Conceptually:

```text
expression
    ↓
GROUP BY gene
    ↓
CTE: gene_expression_summary
    ↓
Filter mean_expression > 20
    ↓
Sort
```


# 14.9 Multiple CTEs

A `WITH` clause can define more than one CTE.

Example:



```python
WITH diabetic_patients AS (
    SELECT patient_id
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
),
blood_samples AS (
    SELECT
        sample_id,
        patient_id
    FROM samples
    WHERE tissue = 'Blood'
)
SELECT
    b.sample_id,
    b.patient_id
FROM blood_samples AS b
JOIN diabetic_patients AS d
    ON b.patient_id = d.patient_id;
```

This query creates two logical datasets:

```text
diabetic_patients
blood_samples
```

and then joins them.


# 14.10 Why Multiple CTEs Are Powerful

Complex analytical workflows can be represented as stages.

For example:

```text
CTE 1 → eligible patients
CTE 2 → QC-passed samples
CTE 3 → filtered expression
CTE 4 → gene summary
Final query → ranked results
```

This is much easier to understand than one deeply nested statement.


# 14.11 Chained CTEs

A later CTE can reference an earlier CTE.

Example:



```python
WITH diabetic_patients AS (
    SELECT patient_id
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
),
diabetic_samples AS (
    SELECT
        s.sample_id,
        s.patient_id,
        s.tissue
    FROM samples AS s
    JOIN diabetic_patients AS d
        ON s.patient_id = d.patient_id
)
SELECT *
FROM diabetic_samples;
```

The second CTE uses the result produced by the first.

This is called **CTE chaining**.


# 14.12 Omics Chained CTE Example

Suppose we want mean expression for genes measured in blood samples from diabetic patients.

We can divide the problem:



```python
WITH diabetic_patients AS (
    SELECT patient_id
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
),
blood_samples AS (
    SELECT
        s.sample_id
    FROM samples AS s
    JOIN diabetic_patients AS d
        ON s.patient_id = d.patient_id
    WHERE s.tissue = 'Blood'
),
filtered_expression AS (
    SELECT
        e.gene_id,
        e.expression_value
    FROM expression AS e
    JOIN blood_samples AS b
        ON e.sample_id = b.sample_id
)
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM filtered_expression
GROUP BY gene_id
ORDER BY mean_expression DESC;
```

This query is logically organized as:

```text
Patients
   ↓
Blood samples
   ↓
Expression
   ↓
Gene-level summary
```

This is an ideal use of CTEs in biomedical SQL.


# 14.13 Reusing a CTE More Than Once

A CTE can be referenced multiple times inside the same statement.

Example:



```python
WITH gene_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT
    g1.gene_id,
    g1.mean_expression
FROM gene_summary AS g1
WHERE g1.mean_expression > (
    SELECT AVG(g2.mean_expression)
    FROM gene_summary AS g2
);
```

Here the same intermediate gene summary is used in:

- the outer query
- a subquery

This can improve readability and reduce duplicated SQL text.


# 14.14 CTE with JOIN

CTEs can contain joins:



```python
WITH patient_samples AS (
    SELECT
        p.patient_id,
        p.diagnosis,
        s.sample_id,
        s.tissue
    FROM patients AS p
    JOIN samples AS s
        ON p.patient_id = s.patient_id
)
SELECT *
FROM patient_samples
WHERE tissue = 'Blood';
```

The CTE hides the join complexity from the final query.


# 14.15 CTE with Aggregation

Example:



```python
WITH tissue_counts AS (
    SELECT
        tissue,
        COUNT(*) AS sample_count
    FROM samples
    GROUP BY tissue
)
SELECT *
FROM tissue_counts
WHERE sample_count >= 10;
```

This separates aggregation from post-aggregation filtering.


# 14.16 CTE with HAVING

We can also place the filtering inside the CTE:



```python
WITH tissue_counts AS (
    SELECT
        tissue,
        COUNT(*) AS sample_count
    FROM samples
    GROUP BY tissue
    HAVING COUNT(*) >= 10
)
SELECT *
FROM tissue_counts;
```

Both approaches may be appropriate depending on readability.


# 14.17 CTE with Subqueries

A CTE can contain subqueries.

Example:



```python
WITH above_average_patients AS (
    SELECT
        patient_id,
        age
    FROM patients
    WHERE age > (
        SELECT AVG(age)
        FROM patients
    )
)
SELECT *
FROM above_average_patients;
```

CTEs and subqueries are complementary SQL tools.


# 14.18 CTE with Set Operations

CTEs can contain `UNION`, `INTERSECT`, or `EXCEPT`.

Example:



```python
WITH shared_genes AS (
    SELECT gene_id
    FROM liver_genes

    INTERSECT

    SELECT gene_id
    FROM brain_genes
)
SELECT *
FROM shared_genes;
```

This provides a readable name for the set-operation result.


# 14.19 CTE with Window Functions: Preview

Later, window functions will be used heavily with CTEs.

Example:



```python
WITH ranked_expression AS (
    SELECT
        gene_id,
        sample_id,
        expression_value,
        ROW_NUMBER() OVER (
            PARTITION BY gene_id
            ORDER BY expression_value DESC
        ) AS rn
    FROM expression
)
SELECT *
FROM ranked_expression
WHERE rn = 1;
```

This is a classic pattern:

```text
Window function in CTE
        ↓
Filter window result outside
```

We will cover window functions comprehensively in Chapter 15.


# 14.20 Why Window Functions Often Need a CTE

SQL logical processing prevents us from directly using many window-function aliases inside `WHERE`.

A CTE solves this cleanly.

Conceptually:

```text
CTE
  ↓
Calculate rank
  ↓
Outer query
  ↓
Filter rank
```

This makes CTEs and window functions natural partners.


# 14.21 CTE Column Names

A CTE can explicitly define output column names.

Example:



```python
WITH patient_summary (
    diagnosis,
    patient_count
) AS (
    SELECT
        diagnosis,
        COUNT(*)
    FROM patients
    GROUP BY diagnosis
)
SELECT *
FROM patient_summary;
```

Explicit column naming can improve clarity when expressions lack useful aliases.


# 14.22 Nested WITH Queries

CTEs may appear in complex query structures, but unnecessary nesting should be avoided.

Readable SQL is the goal.

Prefer:

```text
one clear WITH clause
+
logical CTE names
+
simple final query
```

instead of deeply layered nested `WITH` statements unless truly necessary.


# 14.23 What Is a Recursive CTE?

A **recursive CTE** is a CTE that can refer to its own previous output.

It is introduced using:



```python
WITH RECURSIVE
```

Recursive CTEs are designed for hierarchical or iterative relationships.

Examples include:

- Employee → manager hierarchy
- Parent → child categories
- Gene ontology structures
- Biological pathways
- Pedigrees
- Folder trees
- Taxonomic hierarchies


# 14.24 Recursive CTE Structure

A recursive CTE generally contains two parts:

```text
Anchor query
+
Recursive query
```

General structure:



```python
WITH RECURSIVE cte_name AS (

    -- Anchor member
    SELECT ...

    UNION ALL

    -- Recursive member
    SELECT ...
    FROM table_name
    JOIN cte_name
        ON ...
)
SELECT *
FROM cte_name;
```

The anchor member starts the recursion.

The recursive member repeatedly expands the result.


# 14.25 Layman Analogy for Recursion

Suppose you want to find all descendants in a family tree.

Start with:

```text
Person A
```

Find A's children.

Then find the children's children.

Then their children.

Continue until no more descendants exist.

That is the basic logic of recursion.


# 14.26 Simple Number Recursive CTE

A classic educational example:



```python
WITH RECURSIVE numbers AS (
    SELECT 1 AS n

    UNION ALL

    SELECT n + 1
    FROM numbers
    WHERE n < 5
)
SELECT *
FROM numbers;
```

Result:

```text
1
2
3
4
5
```

How it works:

```text
Anchor → 1

Recursive:
1 → 2
2 → 3
3 → 4
4 → 5

Stop because n < 5 becomes false
```


# 14.27 Why the Termination Condition Matters

This condition:



```python
WHERE n < 5
```

stops the recursion.

Without a valid termination mechanism, a recursive query can continue indefinitely until PostgreSQL or system limits intervene.

Recursive SQL must be designed carefully.


# 14.28 Hierarchical Employee Example

Suppose:



```python
CREATE TABLE employees (
    employee_id INTEGER PRIMARY KEY,
    employee_name TEXT,
    manager_id INTEGER
);
```

Data:



```python
INSERT INTO employees (
    employee_id,
    employee_name,
    manager_id
)
VALUES
    (1, 'Director', NULL),
    (2, 'Manager A', 1),
    (3, 'Manager B', 1),
    (4, 'Analyst A', 2),
    (5, 'Analyst B', 2),
    (6, 'Analyst C', 3);
```

Hierarchy:

```text
Director
├── Manager A
│   ├── Analyst A
│   └── Analyst B
└── Manager B
    └── Analyst C
```


# 14.29 Recursive Employee Query



```python
WITH RECURSIVE employee_tree AS (

    SELECT
        employee_id,
        employee_name,
        manager_id,
        1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.employee_id,
        e.employee_name,
        e.manager_id,
        et.level + 1
    FROM employees AS e
    JOIN employee_tree AS et
        ON e.manager_id = et.employee_id
)
SELECT *
FROM employee_tree
ORDER BY level, employee_id;
```

The CTE walks downward through the hierarchy.


# 14.30 Depth / Level Tracking

The expression:



```python
et.level + 1
```

records hierarchy depth.

Conceptually:

```text
Director   → level 1
Managers   → level 2
Analysts   → level 3
```

Depth tracking is extremely useful in hierarchical queries.


# 14.31 Omics Example: Gene Ontology-Like Hierarchy

Suppose a simplified ontology table contains:



```python
CREATE TABLE ontology_terms (
    term_id VARCHAR(30) PRIMARY KEY,
    term_name TEXT,
    parent_term_id VARCHAR(30)
);
```

Conceptually:

```text
Biological Process
      ↓
Immune Process
      ↓
T-cell Activation
```

A recursive CTE can traverse parent-child relationships.


# 14.32 Recursive Ontology Query

Suppose we want all descendants of a particular ontology term.



```python
WITH RECURSIVE term_tree AS (

    SELECT
        term_id,
        term_name,
        parent_term_id,
        0 AS depth
    FROM ontology_terms
    WHERE term_id = 'GO_PARENT'

    UNION ALL

    SELECT
        t.term_id,
        t.term_name,
        t.parent_term_id,
        tt.depth + 1
    FROM ontology_terms AS t
    JOIN term_tree AS tt
        ON t.parent_term_id = tt.term_id
)
SELECT *
FROM term_tree
ORDER BY depth, term_id;
```

This can identify all descendant terms below an ontology node.


# 14.33 Pathway Hierarchy Example

A pathway database may contain:

```text
Pathway
  ↓
Sub-pathway
  ↓
Sub-sub-pathway
```

Recursive CTEs can traverse these structures when parent-child relationships are stored relationally.


# 14.34 Pedigree Example

Suppose:



```python
CREATE TABLE pedigree (
    individual_id VARCHAR(30) PRIMARY KEY,
    father_id VARCHAR(30),
    mother_id VARCHAR(30)
);
```

Recursive queries can be used to traverse ancestry or descendancy.

Pedigree recursion must be designed carefully because each individual can have multiple parent relationships.


# 14.35 Recursive CTE with a Path

We can build a textual path showing hierarchy traversal.

Example:



```python
WITH RECURSIVE employee_tree AS (

    SELECT
        employee_id,
        employee_name,
        manager_id,
        employee_name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.employee_id,
        e.employee_name,
        e.manager_id,
        et.path || ' > ' || e.employee_name
    FROM employees AS e
    JOIN employee_tree AS et
        ON e.manager_id = et.employee_id
)
SELECT *
FROM employee_tree;
```

Possible path:

```text
Director > Manager A > Analyst A
```

Path construction helps interpret recursive traversal.


# 14.36 Detecting Cycles

Hierarchical data can contain accidental cycles.

Example:

```text
A → B
B → C
C → A
```

Without protection, recursion may loop repeatedly.

PostgreSQL provides recursive-query features and SQL constructs that can help with cycle detection, and manual path-based logic can also be used when appropriate.


# 14.37 Manual Cycle Avoidance with Arrays

A simplified PostgreSQL pattern can track visited IDs.



```python
WITH RECURSIVE hierarchy AS (

    SELECT
        employee_id,
        employee_name,
        manager_id,
        ARRAY[employee_id] AS visited
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.employee_id,
        e.employee_name,
        e.manager_id,
        h.visited || e.employee_id
    FROM employees AS e
    JOIN hierarchy AS h
        ON e.manager_id = h.employee_id
    WHERE NOT e.employee_id = ANY(h.visited)
)
SELECT *
FROM hierarchy;
```

The array stores previously visited nodes.

This helps avoid revisiting the same node in a cycle.


# 14.38 PostgreSQL SEARCH and CYCLE

Modern PostgreSQL supports SQL-standard-style `SEARCH` and `CYCLE` clauses for recursive queries.

These features can help manage:

- Traversal order
- Cycle detection

They are advanced recursive-query tools and are useful when hierarchical logic becomes complex.


# 14.39 Depth-First and Breadth-First Traversal

Two important traversal concepts are:

```text
Depth-first
Breadth-first
```

Depth-first explores one branch deeply before moving to another.

Breadth-first explores all nodes at one level before moving deeper.

PostgreSQL recursive query output ordering can be constructed to represent these traversal strategies.


# 14.40 Recursive CTE Performance

Recursive queries can become expensive.

Performance depends on:

- Number of hierarchy levels
- Number of child nodes
- Join indexes
- Cycle handling
- Duplicate elimination
- Termination logic

Indexes on relationship columns are often important.


# 14.41 Indexing Recursive Relationships

For employee hierarchy:



```python
CREATE INDEX idx_employees_manager_id
ON employees (manager_id);
```

For ontology:



```python
CREATE INDEX idx_ontology_parent
ON ontology_terms (parent_term_id);
```

These indexes can help recursive steps find child rows efficiently.


# 14.42 UNION vs UNION ALL in Recursive CTEs

Recursive CTEs commonly use:



```python
UNION ALL
```

because it avoids duplicate elimination overhead.

`UNION` removes duplicates and can sometimes help prevent repeated states, but it also changes performance and semantics.

Choose deliberately.


# 14.43 CTE Materialization

Historically, PostgreSQL treated CTEs as optimization fences more often.

Modern PostgreSQL can inline non-recursive, side-effect-free CTEs in suitable cases.

This means PostgreSQL may optimize a CTE together with the surrounding query rather than always materializing it.


# 14.44 MATERIALIZED

PostgreSQL allows explicit:



```python
WITH gene_summary AS MATERIALIZED (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM gene_summary
WHERE mean_expression > 20;
```

`MATERIALIZED` tells PostgreSQL to compute and store the CTE result as an intermediate relation for the statement.

This can sometimes be useful when:

- The CTE is reused
- The calculation is expensive
- Recalculation should be avoided


# 14.45 NOT MATERIALIZED

PostgreSQL also supports:



```python
WITH gene_summary AS NOT MATERIALIZED (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM gene_summary
WHERE mean_expression > 20;
```

This gives PostgreSQL more freedom to fold the CTE into the parent query where allowed.

Use these options only when you understand the execution trade-offs.


# 14.46 Do Not Assume CTEs Are Always Faster

A CTE is primarily a **query-organization tool**.

It is not automatically a performance optimization.

A CTE can be:

- Faster
- Slower
- Equivalent

depending on:

- Query structure
- PostgreSQL version
- Materialization
- Statistics
- Indexes
- Number of references


# 14.47 Use EXPLAIN

Always measure.



```python
EXPLAIN
WITH gene_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM gene_summary
WHERE mean_expression > 20;
```

For real performance measurement:



```python
EXPLAIN (ANALYZE, BUFFERS)
WITH gene_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression
    FROM expression
    GROUP BY gene_id
)
SELECT *
FROM gene_summary
WHERE mean_expression > 20;
```

Performance conclusions should be evidence-based.


# 14.48 Data-Modifying CTEs

PostgreSQL allows data-modifying statements inside `WITH`.

These include:

- `INSERT`
- `UPDATE`
- `DELETE`
- `MERGE` in appropriate contexts

This makes it possible to combine data changes and result handling.


# 14.49 UPDATE with RETURNING in a CTE

Example:



```python
WITH updated_samples AS (
    UPDATE samples
    SET qc_status = 'Pass'
    WHERE qc_status = 'Pending'
    RETURNING
        sample_id,
        patient_id,
        qc_status
)
SELECT *
FROM updated_samples;
```

The CTE captures rows changed by the update.


# 14.50 Why RETURNING Is Powerful

PostgreSQL supports:



```python
RETURNING
```

with data-modifying statements.

This lets SQL immediately return:

- Inserted rows
- Updated rows
- Deleted rows

without running another separate query.


# 14.51 INSERT with CTE



```python
WITH new_sample AS (
    INSERT INTO samples (
        sample_id,
        patient_id,
        tissue,
        qc_status
    )
    VALUES (
        'S999',
        101,
        'Blood',
        'Pending'
    )
    RETURNING *
)
SELECT *
FROM new_sample;
```

This inserts the row and returns it through the CTE.


# 14.52 DELETE with CTE



```python
WITH deleted_samples AS (
    DELETE FROM samples
    WHERE qc_status = 'Fail'
    RETURNING
        sample_id,
        patient_id,
        tissue
)
SELECT *
FROM deleted_samples;
```

This can be useful for auditing or downstream operations.

Use destructive CTEs carefully and preferably inside explicit transactions when testing.


# 14.53 Data-Modifying CTE for Auditing

A pattern can capture updated rows and write audit information.

Example:



```python
WITH updated AS (
    UPDATE samples
    SET qc_status = 'Pass'
    WHERE sample_id = 'S001'
    RETURNING sample_id, qc_status
)
INSERT INTO sample_audit (
    sample_id,
    action,
    action_time
)
SELECT
    sample_id,
    'QC changed to ' || qc_status,
    CURRENT_TIMESTAMP
FROM updated;
```

This links data modification and audit insertion in one SQL statement.


# 14.54 CTE with INSERT ... SELECT

A CTE can prepare rows before insertion.

Example:



```python
WITH eligible_samples AS (
    SELECT
        sample_id,
        patient_id
    FROM samples
    WHERE qc_status = 'Pass'
)
INSERT INTO analysis_samples (
    sample_id,
    patient_id
)
SELECT
    sample_id,
    patient_id
FROM eligible_samples;
```

This provides a readable ETL-style pattern.


# 14.55 Omics ETL Example

Suppose expression measurements need to be summarized before being inserted into a results table.



```python
WITH expression_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression,
        COUNT(*) AS n_measurements
    FROM expression
    GROUP BY gene_id
)
INSERT INTO gene_expression_results (
    gene_id,
    mean_expression,
    n_measurements
)
SELECT
    gene_id,
    mean_expression,
    n_measurements
FROM expression_summary;
```

This is a common database transformation pattern.


# 14.56 CTEs for Debugging

CTEs are useful when developing complicated SQL.

Build the query step by step.

Start with:



```python
WITH step1 AS (
    SELECT *
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
)
SELECT *
FROM step1;
```

Then add:



```python
WITH step1 AS (
    SELECT *
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
),
step2 AS (
    SELECT
        s.*
    FROM samples AS s
    JOIN step1 AS p
        ON s.patient_id = p.patient_id
)
SELECT *
FROM step2;
```

Continue adding stages only after each one is verified.

This makes complex SQL much easier to debug.


# 14.57 Good CTE Naming

Prefer descriptive names:

```text
eligible_patients
qc_passed_samples
blood_expression
gene_summary
ranked_genes
```

Avoid:

```text
cte1
cte2
temp1
x
y
```

unless the query is extremely short.


# 14.58 CTEs and Unit of Observation

Every CTE should have a clear **unit of observation**.

Examples:

```text
one row per patient
one row per sample
one row per gene
one row per gene × tissue
one row per sample × gene
```

This prevents accidental overcounting and join multiplication.


# 14.59 Omics Example: Multiple Analytical Stages

Suppose the goal is:

> Among QC-passed blood samples from diabetic patients, identify genes with at least 10 measurements and mean expression above 20.

We can write:



```python
WITH diabetic_patients AS (
    SELECT patient_id
    FROM patients
    WHERE diagnosis = 'Type 2 Diabetes'
),
eligible_samples AS (
    SELECT
        s.sample_id
    FROM samples AS s
    JOIN diabetic_patients AS p
        ON s.patient_id = p.patient_id
    WHERE s.tissue = 'Blood'
      AND s.qc_status = 'Pass'
),
eligible_expression AS (
    SELECT
        e.gene_id,
        e.expression_value
    FROM expression AS e
    JOIN eligible_samples AS s
        ON e.sample_id = s.sample_id
),
gene_summary AS (
    SELECT
        gene_id,
        AVG(expression_value) AS mean_expression,
        COUNT(*) AS n_measurements
    FROM eligible_expression
    GROUP BY gene_id
)
SELECT
    gene_id,
    mean_expression,
    n_measurements
FROM gene_summary
WHERE n_measurements >= 10
  AND mean_expression > 20
ORDER BY mean_expression DESC;
```

This is a highly readable analytical pipeline.


# 14.60 The Same Logic Without CTEs

The same question could be written as one large nested query.

But readability would usually be lower.

The value of CTEs is not that they create new analytical capabilities.

The value is that they allow complex logic to be expressed in **named, structured stages**.


# 14.61 Common Beginner Mistake: Thinking a CTE Is Permanent

A CTE disappears after the statement finishes.

If you need persistent reusable logic, consider:

- View
- Materialized view
- Table


# 14.62 Common Beginner Mistake: Overusing CTEs

Not every simple query needs a CTE.

This:



```python
WITH x AS (
    SELECT *
    FROM patients
)
SELECT *
FROM x;
```

adds no real value.

Use CTEs when they improve clarity, reuse, recursion, or statement structure.


# 14.63 Common Beginner Mistake: Assuming Written Order Means Physical Execution

CTEs make the query look sequential:

```text
CTE 1
CTE 2
CTE 3
Final query
```

But PostgreSQL is a declarative database.

The optimizer may rewrite or inline parts of the query where allowed.

The written order is a **logical organization**, not always the exact physical execution order.


# 14.64 Common Beginner Mistake: Confusing Recursive CTE with Ordinary Recursion

Recursive SQL is iterative set processing.

It is not identical to recursion in Python or other procedural languages.

The database repeatedly evaluates relational results until no new qualifying rows remain.


# 14.65 Common Beginner Mistake: Missing Termination Logic

A recursive query without proper termination or cycle handling can become extremely expensive or fail.

Always understand:

```text
Starting rows
Recursive relationship
Termination
Cycle risk
```


# 14.66 Common Beginner Mistake: Using UNION Instead of UNION ALL Without Reason

Recursive CTEs often use `UNION ALL`.

Using `UNION` introduces duplicate elimination.

That may be necessary in some problems, but it should be deliberate.


# 14.67 Common Beginner Mistake: Ignoring Indexes in Recursive Queries

Recursive joins repeatedly search relationship columns.

For example:

```text
manager_id
parent_term_id
parent_category_id
```

Indexes on these columns may be crucial.


# 14.68 CTE Decision Framework

Use a CTE when:

### The query contains several logical stages

Use named CTEs.

### The same intermediate result is referenced multiple times

A CTE may improve readability.

### A hierarchical relationship must be traversed

Use `WITH RECURSIVE`.

### A data-modifying statement should feed another operation

Use a data-modifying CTE with `RETURNING`.

### The logic is used only once

A CTE is often more appropriate than creating a permanent view.

### The query is trivial

A CTE may be unnecessary.


# 14.69 CTE vs Subquery vs View vs Temporary Table

| Feature | CTE | Subquery | View | Temporary Table |
|---|---|---|---|---|
| Query-local | Yes | Yes | No | No |
| Named | Yes | Usually alias only | Yes | Yes |
| Persistent definition | No | No | Yes | Session-level |
| Stores rows physically | Usually no | No | No | Yes |
| Supports recursion | Yes | Not directly | No | No |
| Good for staged query logic | Excellent | Moderate | Good for reuse | Good for multi-step workflows |


# 14.70 Advanced Preview: CTE + Window Function

The next chapter will use CTEs extensively.

Example:



```python
WITH ranked_genes AS (
    SELECT
        tissue,
        gene_id,
        expression_value,
        RANK() OVER (
            PARTITION BY tissue
            ORDER BY expression_value DESC
        ) AS expression_rank
    FROM expression_by_tissue
)
SELECT *
FROM ranked_genes
WHERE expression_rank <= 5;
```

This pattern finds the top five genes per tissue.

It combines:

```text
CTE
+
Window function
+
Partitioning
+
Ranking
```

This is one of the most important advanced analytical SQL patterns.


# Module 14 Summary

In this chapter, we learned:

- A Common Table Expression is a named result set defined using `WITH`.
- A CTE exists only for the duration of one SQL statement.
- CTEs improve readability and modularity.
- CTEs can replace difficult-to-read derived-table nesting.
- Multiple CTEs can be defined in one `WITH` clause.
- Later CTEs can reference earlier CTEs.
- CTEs can contain joins, aggregations, subqueries, and set operations.
- CTEs are especially useful for staged biomedical analytical workflows.
- CTEs and views are different: views persist, CTEs do not.
- Recursive CTEs use `WITH RECURSIVE`.
- Recursive CTEs contain an anchor member and a recursive member.
- Recursive CTEs are useful for hierarchies, ontologies, pedigrees, and pathway trees.
- Recursive queries require proper termination logic.
- Cycle detection is important for graph-like data.
- Relationship columns should often be indexed for recursive traversal.
- Modern PostgreSQL may inline eligible non-recursive CTEs.
- `MATERIALIZED` and `NOT MATERIALIZED` can influence CTE planning behavior.
- CTEs are not automatically performance optimizations.
- `EXPLAIN` and `EXPLAIN ANALYZE` should be used to evaluate performance.
- PostgreSQL supports data-modifying CTEs.
- `RETURNING` can feed modified rows into subsequent query logic.
- CTEs are powerful for ETL-style transformations and auditing.
- A clear unit of observation should be maintained at every CTE stage.


# Key Takeaway

> **A CTE turns a difficult SQL query into a sequence of named logical steps.**

For omics analysis, this allows a complex question such as:

```text
Select diabetic patients
        ↓
Find QC-passed blood samples
        ↓
Retrieve expression measurements
        ↓
Summarize by gene
        ↓
Filter high-expression genes
```

to be represented clearly and reproducibly.

Recursive CTEs extend the idea further by allowing PostgreSQL to traverse structures such as:

```text
Ontology parent → child
Pathway → sub-pathway
Manager → employee
Ancestor → descendant
```

CTEs are therefore one of the key bridges between intermediate SQL and advanced analytical SQL.


# References

1. PostgreSQL Global Development Group. **Queries With Common Table Expressions**  
   https://www.postgresql.org/docs/current/queries-with.html

2. PostgreSQL Global Development Group. **WITH Queries (Common Table Expressions)**  
   https://www.postgresql.org/docs/current/queries-with.html

3. PostgreSQL Global Development Group. **SELECT**  
   https://www.postgresql.org/docs/current/sql-select.html

4. PostgreSQL Global Development Group. **INSERT**  
   https://www.postgresql.org/docs/current/sql-insert.html

5. PostgreSQL Global Development Group. **UPDATE**  
   https://www.postgresql.org/docs/current/sql-update.html

6. PostgreSQL Global Development Group. **DELETE**  
   https://www.postgresql.org/docs/current/sql-delete.html

7. PostgreSQL Global Development Group. **Data Manipulation — RETURNING Data from Modified Rows**  
   https://www.postgresql.org/docs/current/dml-returning.html

8. PostgreSQL Global Development Group. **Using EXPLAIN**  
   https://www.postgresql.org/docs/current/using-explain.html

9. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

10. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

11. Date, C. J. *An Introduction to Database Systems*. Addison-Wesley.

12. Ensembl Documentation  
    https://www.ensembl.org/info/docs/index.html

13. Gene Ontology Resource  
    https://geneontology.org/

14. Reactome Pathway Database  
    https://reactome.org/

