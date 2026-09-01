# Stored Procedures, Functions, Control Flow, and Triggers
### PostgreSQL Stored Routines, Parameters, PL/pgSQL, IF/CASE, LOOP/WHILE/FOR, Exception Handling, and Omics Applications

---

## Introduction: Why Stored Programs Matter

Up to this point, most SQL statements have been issued directly by the user or application.

But many database tasks are repetitive:

- Validate a biological sample
- Calculate a patient's sample count
- Update QC status
- Log changes automatically
- Standardize analytical calculations
- Perform recurring multi-step operations
- Enforce database-side business or scientific logic

PostgreSQL allows logic to be stored and executed **inside the database**.

The major tools include:

- SQL functions
- PL/pgSQL functions
- Stored procedures
- Control-flow statements
- Exception handling
- Triggers

These tools move SQL from simple querying toward **database programming**.


# 11.1 What Is a Stored Program?

A stored program is executable logic saved inside the database.

Instead of repeatedly sending the same statements from Python, R, Java, or another application, the logic can be encapsulated once and reused.

Conceptually:

```text
Repeated SQL logic
      ↓
Store inside PostgreSQL
      ↓
Call by name
      ↓
Reusable database operation
```

This improves:

- Reusability
- Consistency
- Maintainability
- Centralized validation
- Encapsulation


# 11.2 Functions vs Procedures

PostgreSQL supports both **functions** and **procedures**.

They overlap in purpose but are not identical.

## Function

A function typically:

- Accepts zero or more arguments
- Returns a value, row, table, or set
- Can often be used inside SQL expressions
- Is called using `SELECT` or from another expression

## Procedure

A procedure:

- Is invoked using `CALL`
- Is intended for performing actions
- Does not behave like a scalar expression
- Can support transaction control in appropriate contexts

A useful mental model is:

```text
Function → compute and return something

Procedure → perform an operation
```


# 11.3 Why Use Database Functions?

Functions are useful when the same calculation or lookup occurs repeatedly.

Examples:

- Calculate BMI
- Normalize a score
- Convert quality metrics
- Count samples for a patient
- Return mean expression for a gene
- Categorize allele frequency
- Generate reusable derived values


# 11.4 Simple SQL Function

Suppose we want to calculate the square of a number.



```python
CREATE OR REPLACE FUNCTION square_value(x NUMERIC)
RETURNS NUMERIC
LANGUAGE SQL
AS $$
    SELECT x * x;
$$;
```

Call it with:



```python
SELECT square_value(5);
```

Result:

```text
25
```

This function accepts one parameter and returns one scalar value.


# 11.5 Omics Example: Allele Frequency Category

Suppose we repeatedly classify allele frequencies.

A simplified rule might be:

```text
AF < 0.01       → Rare
0.01–0.05       → Low-frequency
> 0.05          → Common
```



```python
CREATE OR REPLACE FUNCTION af_category(af NUMERIC)
RETURNS TEXT
LANGUAGE SQL
AS $$
    SELECT CASE
        WHEN af IS NULL THEN 'Unknown'
        WHEN af < 0.01 THEN 'Rare'
        WHEN af <= 0.05 THEN 'Low-frequency'
        ELSE 'Common'
    END;
$$;
```

Use it:



```python
SELECT
    variant_id,
    allele_frequency,
    af_category(allele_frequency) AS frequency_class
FROM variants;
```

The function centralizes the categorization logic.

If the classification rule later changes, the function can be updated once.


# 11.6 Function Parameters

Function parameters act like local inputs.

Example:



```python
CREATE OR REPLACE FUNCTION expression_above_threshold(
    expr NUMERIC,
    threshold NUMERIC
)
RETURNS BOOLEAN
LANGUAGE SQL
AS $$
    SELECT expr > threshold;
$$;
```

Call:



```python
SELECT expression_above_threshold(18.5, 15);
```

This returns a Boolean result.


# 11.7 Parameter Names Matter

Good parameter names improve readability.

Prefer:

```text
gene_id_input
expression_threshold
patient_id_input
```

rather than:

```text
x
y
z
```

for production biological functions.


# 11.8 PL/pgSQL

PostgreSQL provides a procedural language called **PL/pgSQL**.

It extends SQL with programming constructs such as:

- Variables
- `IF`
- `CASE`
- Loops
- Exception handling
- Multiple statements
- Control flow

This allows more complex database-side logic.


# 11.9 Basic PL/pgSQL Function Structure



```python
CREATE OR REPLACE FUNCTION function_name(parameter datatype)
RETURNS datatype
LANGUAGE plpgsql
AS $$
DECLARE
    -- local variables
BEGIN
    -- statements
    RETURN value;
END;
$$;
```

The major sections are:

```text
DECLARE
BEGIN
END
```

`DECLARE` is optional.

`BEGIN ... END` contains the executable logic.


# 11.10 Variables in PL/pgSQL

Variables store intermediate values.



```python
CREATE OR REPLACE FUNCTION double_expression(expr NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    result_value NUMERIC;
BEGIN
    result_value := expr * 2;
    RETURN result_value;
END;
$$;
```

PL/pgSQL uses:

```text
:=
```

for variable assignment.


# 11.11 Clinical Example: Age Category Function



```python
CREATE OR REPLACE FUNCTION age_group(age_input INTEGER)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF age_input IS NULL THEN
        RETURN 'Unknown';
    ELSIF age_input < 18 THEN
        RETURN 'Minor';
    ELSIF age_input < 65 THEN
        RETURN 'Adult';
    ELSE
        RETURN 'Older Adult';
    END IF;
END;
$$;
```

Use:



```python
SELECT
    patient_id,
    age,
    age_group(age) AS age_category
FROM patients;
```

This introduces conditional control flow.


# 11.12 IF, ELSIF, ELSE

PL/pgSQL supports:



```python
IF condition THEN
    statements;
ELSIF another_condition THEN
    statements;
ELSE
    statements;
END IF;
```

This is appropriate when different actions should occur depending on a condition.


# 11.13 Omics Example: QC Classification

Suppose sample QC is based on call rate.

A simplified educational rule:

```text
>= 0.99 → Excellent
>= 0.95 → Pass
otherwise → Fail
```



```python
CREATE OR REPLACE FUNCTION classify_call_rate(call_rate NUMERIC)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF call_rate IS NULL THEN
        RETURN 'Missing';
    ELSIF call_rate >= 0.99 THEN
        RETURN 'Excellent';
    ELSIF call_rate >= 0.95 THEN
        RETURN 'Pass';
    ELSE
        RETURN 'Fail';
    END IF;
END;
$$;
```

Call:



```python
SELECT
    sample_id,
    call_rate,
    classify_call_rate(call_rate) AS qc_class
FROM sample_qc;
```

Scientific thresholds should always reflect the real analysis protocol rather than arbitrary examples.


# 11.14 CASE in PL/pgSQL

PL/pgSQL also supports `CASE`.

Example:



```python
CREATE OR REPLACE FUNCTION chromosome_class(chr TEXT)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN CASE
        WHEN chr IN ('X', 'Y') THEN 'Sex chromosome'
        WHEN chr = 'MT' THEN 'Mitochondrial'
        ELSE 'Autosome'
    END;
END;
$$;
```

`CASE` is often cleaner than multiple `IF` branches for category mapping.


# 11.15 SQL CASE vs PL/pgSQL CASE

SQL `CASE` is an expression:



```python
SELECT
    gene_id,
    CASE
        WHEN chromosome = '6' THEN 'Chromosome 6'
        ELSE 'Other'
    END AS chr_group
FROM genes;
```

PL/pgSQL `CASE` can be used as procedural control flow.

Both are related but appear in different programming contexts.


# 11.16 Returning a Scalar Value

Example:



```python
CREATE OR REPLACE FUNCTION sample_count_for_patient(pid INTEGER)
RETURNS BIGINT
LANGUAGE SQL
AS $$
    SELECT COUNT(*)
    FROM samples
    WHERE patient_id = pid;
$$;
```

Call:



```python
SELECT sample_count_for_patient(101);
```

This returns one value.


# 11.17 Returning a Table

A PostgreSQL function can return multiple rows and columns.



```python
CREATE OR REPLACE FUNCTION samples_for_patient(pid INTEGER)
RETURNS TABLE (
    sample_id VARCHAR,
    tissue VARCHAR,
    qc_status VARCHAR
)
LANGUAGE SQL
AS $$
    SELECT
        s.sample_id,
        s.tissue,
        s.qc_status
    FROM samples AS s
    WHERE s.patient_id = pid;
$$;
```

Call:



```python
SELECT *
FROM samples_for_patient(101);
```

This behaves like a table-producing function.


# 11.18 Returning SETOF

PostgreSQL functions can also return `SETOF` rows of an existing table type.

Example:



```python
CREATE OR REPLACE FUNCTION blood_samples()
RETURNS SETOF samples
LANGUAGE SQL
AS $$
    SELECT *
    FROM samples
    WHERE tissue = 'Blood';
$$;
```

Call:



```python
SELECT *
FROM blood_samples();
```

This is useful when the function should return rows matching an existing table structure.


# 11.19 OUT Parameters

PostgreSQL functions can define output parameters.

Example:



```python
CREATE OR REPLACE FUNCTION expression_stats(
    input_gene_id VARCHAR,
    OUT mean_expression NUMERIC,
    OUT measurement_count BIGINT
)
LANGUAGE SQL
AS $$
    SELECT
        AVG(expression_value),
        COUNT(*)
    FROM expression
    WHERE gene_id = input_gene_id;
$$;
```

Call:



```python
SELECT *
FROM expression_stats('ENSG00000141510');
```

Output parameters can make multi-value results convenient.


# 11.20 IN, OUT, and INOUT Parameters

PostgreSQL routine parameters may conceptually serve as:

- `IN` → input
- `OUT` → output
- `INOUT` → input that is also returned/modified

Function inputs are `IN` by default.

Understanding these modes becomes more important in stored procedures and reusable APIs.


# 11.21 Stored Procedures

A PostgreSQL procedure is created with:



```python
CREATE PROCEDURE procedure_name(...)
LANGUAGE plpgsql
AS $$
BEGIN
    ...
END;
$$;
```

It is executed using:



```python
CALL procedure_name(...);
```

Procedures are appropriate when the goal is primarily to **perform actions** rather than return a query expression.


# 11.22 Simple Procedure Example



```python
CREATE OR REPLACE PROCEDURE mark_sample_passed(sample_input VARCHAR)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE samples
    SET qc_status = 'Pass'
    WHERE sample_id = sample_input;
END;
$$;
```

Call:



```python
CALL mark_sample_passed('S001');
```

This encapsulates an update operation.


# 11.23 Procedure with Multiple Statements



```python
CREATE OR REPLACE PROCEDURE finalize_sample(sample_input VARCHAR)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE samples
    SET qc_status = 'Pass'
    WHERE sample_id = sample_input;

    INSERT INTO sample_audit(sample_id, action, action_time)
    VALUES (
        sample_input,
        'QC finalized',
        CURRENT_TIMESTAMP
    );
END;
$$;
```

One procedure now performs two related actions:

```text
Update sample
+
Create audit record
```


# 11.24 Why Procedures Help

Procedures are useful when:

- Several SQL statements belong to one logical operation
- An administrative workflow should be standardized
- Repetitive database-side operations need centralization
- Audit/logging steps should accompany data changes


# 11.25 Procedures vs Functions

| Feature | Function | Procedure |
|---|---|---|
| Called with | `SELECT` / expression | `CALL` |
| Returns value | Commonly yes | Not as ordinary scalar expression |
| Can appear inside SELECT | Yes, depending on return type | No |
| Used for calculations | Very common | Less common |
| Used for multi-step actions | Yes | Very common |
| Transaction control | Restricted | Can be supported in proper context |

Choose based on semantics, not merely syntax.


# 11.26 Loops

PL/pgSQL supports several looping constructs:

- `LOOP`
- `WHILE`
- Integer `FOR`
- Query `FOR`
- `FOREACH`

Loops should be used carefully.

SQL is naturally **set-based**, so a single SQL statement is often preferable to processing rows one at a time.


# 11.27 Basic LOOP



```python
DO $$
DECLARE
    i INTEGER := 1;
BEGIN
    LOOP
        RAISE NOTICE 'Value: %', i;
        i := i + 1;

        EXIT WHEN i > 5;
    END LOOP;
END;
$$;
```

This demonstrates:

- Initialization
- Looping
- `EXIT WHEN`
- `RAISE NOTICE`


# 11.28 WHILE Loop



```python
DO $$
DECLARE
    i INTEGER := 1;
BEGIN
    WHILE i <= 5 LOOP
        RAISE NOTICE 'Value: %', i;
        i := i + 1;
    END LOOP;
END;
$$;
```

A `WHILE` loop continues while its condition remains true.


# 11.29 Integer FOR Loop



```python
DO $$
BEGIN
    FOR i IN 1..5 LOOP
        RAISE NOTICE 'Iteration: %', i;
    END LOOP;
END;
$$;
```

This is simpler for fixed numeric ranges.


# 11.30 Query FOR Loop

Rows returned by a query can be processed iteratively.



```python
DO $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN
        SELECT gene_id, gene_symbol
        FROM genes
        WHERE chromosome = '6'
    LOOP
        RAISE NOTICE 'Gene: % (%)',
            rec.gene_symbol,
            rec.gene_id;
    END LOOP;
END;
$$;
```

This is useful for procedural tasks, but many analytical operations should still use set-based SQL.


# 11.31 Set-Based SQL vs Loops

Suppose we want to mark all failed samples inactive.

Set-based:



```python
UPDATE samples
SET is_active = FALSE
WHERE qc_status = 'Fail';
```

A procedural loop over every failed sample would usually be unnecessary.

Rule of thumb:

> Prefer set-based SQL when one statement can express the operation clearly.


# 11.32 CONTINUE and EXIT

Inside loops:

```text
CONTINUE
```

skips to the next iteration.

```text
EXIT
```

leaves the loop.

Example:



```python
DO $$
DECLARE
    i INTEGER;
BEGIN
    FOR i IN 1..10 LOOP
        CONTINUE WHEN i % 2 = 0;
        RAISE NOTICE 'Odd value: %', i;
    END LOOP;
END;
$$;
```

# 11.33 Exception Handling

Database operations can fail.

Examples:

- Duplicate primary key
- Foreign-key violation
- Invalid value
- Division by zero
- Missing data
- Custom validation failure

PL/pgSQL supports exception handling.



```python
BEGIN
    -- statements
EXCEPTION
    WHEN some_exception THEN
        -- recovery logic
END;
```

# 11.34 Example: Division by Zero



```python
DO $$
DECLARE
    result_value NUMERIC;
BEGIN
    result_value := 10 / 0;

EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Division by zero was prevented.';
END;
$$;
```

This intercepts the error and performs alternative logic.


# 11.35 Custom Exceptions with RAISE EXCEPTION

You can deliberately reject invalid operations.



```python
CREATE OR REPLACE FUNCTION validate_expression(expr NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
BEGIN
    IF expr < 0 THEN
        RAISE EXCEPTION
            'Expression value cannot be negative: %',
            expr;
    END IF;

    RETURN expr;
END;
$$;
```

This can enforce custom validation inside program logic.

Where possible, true invariant rules should often also be expressed as database constraints.


# 11.36 RAISE Levels

PL/pgSQL supports message levels including:

- `DEBUG`
- `LOG`
- `INFO`
- `NOTICE`
- `WARNING`
- `EXCEPTION`

Example:



```python
RAISE NOTICE 'Sample % processed successfully', sample_input;
```

`RAISE EXCEPTION` aborts the current operation unless handled.


# 11.37 Transactions and Procedures

Procedures can support transaction control in appropriate execution contexts.

Conceptually:



```python
CREATE PROCEDURE example_procedure()
LANGUAGE plpgsql
AS $$
BEGIN
    -- statements
    COMMIT;
END;
$$;
```

Transaction behavior has restrictions depending on how the procedure is called and the surrounding context.

This topic connects directly to the upcoming transactions module.


# 11.38 What Is a Trigger?

A **trigger** is database logic that executes automatically when a specified event occurs.

Common trigger events include:

- `INSERT`
- `UPDATE`
- `DELETE`
- `TRUNCATE`

Triggers can run:

- `BEFORE`
- `AFTER`
- `INSTEAD OF` in supported contexts

A trigger is event-driven.


# 11.39 Why Triggers Matter

Triggers can automatically perform tasks such as:

- Audit changes
- Update timestamps
- Validate complex rules
- Synchronize related values
- Maintain history tables
- Enforce derived logic


# 11.40 Trigger Architecture

In PostgreSQL, a trigger commonly has two components:

```text
Trigger function
      +
CREATE TRIGGER
```

The trigger function contains the logic.

The trigger definition determines when that logic runs.


# 11.41 Basic Trigger Function



```python
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;
```

`NEW` represents the new row in row-level INSERT/UPDATE trigger contexts.


# 11.42 Create the Trigger



```python
CREATE TRIGGER trg_samples_updated_at
BEFORE UPDATE ON samples
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

Now whenever a sample row is updated, PostgreSQL automatically updates its timestamp.


# 11.43 OLD and NEW

Trigger functions may access:

```text
OLD
NEW
```

depending on the event.

For an `UPDATE`:

```text
OLD → row before update
NEW → row after update
```

For an `INSERT`:

```text
NEW → inserted row
```

For a `DELETE`:

```text
OLD → deleted row
```


# 11.44 Audit Trigger Example

Suppose we want to record sample QC changes.



```python
CREATE TABLE sample_qc_audit (
    audit_id BIGSERIAL PRIMARY KEY,
    sample_id VARCHAR(30),
    old_status VARCHAR(20),
    new_status VARCHAR(20),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Trigger function:



```python
CREATE OR REPLACE FUNCTION log_qc_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.qc_status IS DISTINCT FROM NEW.qc_status THEN
        INSERT INTO sample_qc_audit (
            sample_id,
            old_status,
            new_status
        )
        VALUES (
            NEW.sample_id,
            OLD.qc_status,
            NEW.qc_status
        );
    END IF;

    RETURN NEW;
END;
$$;
```

Trigger:



```python
CREATE TRIGGER trg_log_qc_change
AFTER UPDATE ON samples
FOR EACH ROW
EXECUTE FUNCTION log_qc_change();
```

Now QC status changes are logged automatically.


# 11.45 Why IS DISTINCT FROM Is Useful

Ordinary inequality:

```sql
OLD.qc_status <> NEW.qc_status
```

can behave awkwardly with `NULL`.

PostgreSQL provides:

```sql
IS DISTINCT FROM
```

which treats nulls as comparable for this purpose.

This makes it very useful in audit triggers.


# 11.46 BEFORE INSERT Trigger

A trigger can modify incoming values before insertion.

Example:



```python
CREATE OR REPLACE FUNCTION normalize_gene_symbol()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.gene_symbol := UPPER(NEW.gene_symbol);
    RETURN NEW;
END;
$$;
```


```python
CREATE TRIGGER trg_normalize_gene_symbol
BEFORE INSERT OR UPDATE ON genes
FOR EACH ROW
EXECUTE FUNCTION normalize_gene_symbol();
```

Now incoming gene symbols are standardized to uppercase.

Whether this is scientifically appropriate depends on the controlled naming convention being used.


# 11.47 BEFORE vs AFTER Triggers

Use `BEFORE` when you need to:

- Validate
- Modify the incoming row
- Reject the operation before storage

Use `AFTER` when you need to:

- Log the completed change
- Perform follow-up work
- React after the row change is complete

The exact choice depends on semantics.


# 11.48 Row-Level vs Statement-Level Triggers

A row-level trigger:



```python
FOR EACH ROW
```

runs once per affected row.

A statement-level trigger:



```python
FOR EACH STATEMENT
```

runs once for the SQL statement regardless of how many rows it affects.

This distinction can have major performance implications.


# 11.49 Trigger Performance

Triggers add hidden work to DML operations.

An update affecting one million rows may invoke a row-level trigger one million times.

Therefore, triggers should be:

- Simple
- Necessary
- Documented
- Performance tested


# 11.50 Trigger Example: Prevent Invalid QC Transition

Suppose a simplified rule says:

```text
Fail → Pass
```

cannot occur without manual review.

A trigger could reject it.



```python
CREATE OR REPLACE FUNCTION prevent_fail_to_pass()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.qc_status = 'Fail'
       AND NEW.qc_status = 'Pass' THEN
        RAISE EXCEPTION
            'Direct transition from Fail to Pass is not allowed.';
    END IF;

    RETURN NEW;
END;
$$;
```


```python
CREATE TRIGGER trg_prevent_fail_to_pass
BEFORE UPDATE OF qc_status ON samples
FOR EACH ROW
EXECUTE FUNCTION prevent_fail_to_pass();
```

In real systems, workflow rules should be carefully justified and documented.


# 11.51 Trigger vs Constraint

Use a **constraint** when a rule can be expressed declaratively.

Example:



```python
CHECK (expression_value >= 0)
```

Use a trigger when the rule depends on more complex event-driven logic that a normal constraint cannot express cleanly.

Prefer declarative constraints where possible because they are simpler and clearer.


# 11.52 Trigger vs Procedure

A procedure is called explicitly:



```python
CALL finalize_sample('S001');
```

A trigger runs automatically because an event occurs:



```python
UPDATE samples
SET qc_status = 'Pass'
WHERE sample_id = 'S001';
```

This distinction is fundamental:

```text
Procedure → explicit invocation
Trigger   → automatic invocation
```


# 11.53 Function Volatility Categories

PostgreSQL functions can be classified as:

- `IMMUTABLE`
- `STABLE`
- `VOLATILE`

These categories describe how function results may depend on database state and can influence optimization.


## IMMUTABLE

Same inputs always produce the same output.

Example:



```python
CREATE OR REPLACE FUNCTION square_num(x NUMERIC)
RETURNS NUMERIC
LANGUAGE SQL
IMMUTABLE
AS $$
    SELECT x * x;
$$;
```

## STABLE

Can read database state but is expected not to change results within one statement.

## VOLATILE

May change even within one statement or perform effects.

Functions are `VOLATILE` by default unless specified otherwise.


# 11.54 Why Volatility Matters

PostgreSQL's planner uses volatility metadata to decide whether function calls can be:

- Precomputed
- Reordered
- Reused
- Used safely in certain index expressions

Incorrect volatility declarations can produce incorrect behavior or bad planning assumptions.


# 11.55 Security Definer vs Security Invoker

Functions can execute under different privilege models.

Conceptually:

```text
SECURITY INVOKER
```

uses the calling user's privileges.

```text
SECURITY DEFINER
```

uses the function owner's privileges.

Security-definer functions require careful design because they can become privilege-escalation risks if written unsafely.


# 11.56 Stored Program Search Path Security

Security-sensitive PostgreSQL functions should be designed carefully with respect to:

- `search_path`
- Object qualification
- Privileges
- Input validation

This is especially important for `SECURITY DEFINER` routines.


# 11.57 Omics Function Example: Gene Mean Expression



```python
CREATE OR REPLACE FUNCTION mean_expression_for_gene(
    input_gene_id VARCHAR
)
RETURNS NUMERIC
LANGUAGE SQL
STABLE
AS $$
    SELECT AVG(expression_value)
    FROM expression
    WHERE gene_id = input_gene_id;
$$;
```

Call:



```python
SELECT
    gene_id,
    mean_expression_for_gene(gene_id)
FROM genes;
```

For very large tables, set-based grouped aggregation may be more efficient than invoking a scalar lookup function once per gene. Always consider the workload.


# 11.58 Set-Based Alternative



```python
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id;
```

This reinforces an important principle:

> Stored functions improve reuse, but they should not replace efficient set-based SQL when a direct relational query is more appropriate.


# 11.59 Omics Procedure Example: Mark Analysis Run Complete



```python
CREATE OR REPLACE PROCEDURE complete_analysis_run(
    run_id_input BIGINT
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE analysis_runs
    SET
        status = 'Complete',
        completed_at = CURRENT_TIMESTAMP
    WHERE run_id = run_id_input;

    INSERT INTO analysis_audit(
        run_id,
        event_type,
        event_time
    )
    VALUES (
        run_id_input,
        'Run completed',
        CURRENT_TIMESTAMP
    );
END;
$$;
```

Call:



```python
CALL complete_analysis_run(42);
```

This demonstrates a multi-step database workflow.


# 11.60 Common Beginner Mistakes

## Mistake 1 — Using Procedures for Simple Queries

Do not wrap every `SELECT` in a procedure.

Use normal SQL when normal SQL is sufficient.

## Mistake 2 — Using Loops Instead of Set-Based SQL

Prefer one `UPDATE` over row-by-row procedural processing when possible.

## Mistake 3 — Hiding Too Much Logic in Triggers

Triggers can make behavior difficult to discover because they execute automatically.

Document them carefully.

## Mistake 4 — Using Triggers Instead of Constraints

Use declarative constraints whenever a constraint can express the rule.

## Mistake 5 — Ignoring NULL Semantics

Use constructs such as:

```sql
IS DISTINCT FROM
```

when comparing possibly null values.


# 11.61 More Common Mistakes

## Mistake 6 — Incorrect Function Volatility

Do not mark a database-reading function `IMMUTABLE` unless that is truly valid.

## Mistake 7 — Excessive Exception Handling

Do not hide serious database errors with broad exception blocks.

## Mistake 8 — Trigger Recursion

A trigger that updates the same table may cause recursive behavior if not designed carefully.

## Mistake 9 — Heavy Logic in Row-Level Triggers

This can make bulk operations extremely slow.

## Mistake 10 — Security-Definer Misuse

Privilege-bearing functions must be designed defensively.


# 11.62 Routine Management

Functions can be dropped using:



```python
DROP FUNCTION IF EXISTS age_group(INTEGER);
```

Procedures:



```python
DROP PROCEDURE IF EXISTS mark_sample_passed(VARCHAR);
```

The argument signature matters because PostgreSQL supports **function overloading**.


# 11.63 Function Overloading

PostgreSQL can have multiple functions with the same name but different argument types.

Conceptually:



```python
CREATE FUNCTION example_value(x INTEGER)
RETURNS INTEGER
LANGUAGE SQL
AS $$
    SELECT x * 2;
$$;
```


```python
CREATE FUNCTION example_value(x NUMERIC)
RETURNS NUMERIC
LANGUAGE SQL
AS $$
    SELECT x * 2.0;
$$;
```

PostgreSQL resolves the appropriate function using the argument types.


# 11.64 Default Function Arguments



```python
CREATE OR REPLACE FUNCTION high_expression(
    expr NUMERIC,
    threshold NUMERIC DEFAULT 20
)
RETURNS BOOLEAN
LANGUAGE SQL
AS $$
    SELECT expr > threshold;
$$;
```

Calls:



```python
SELECT high_expression(25);
```


```python
SELECT high_expression(25, 30);
```

Default parameters make reusable functions more flexible.


# 11.65 Named Arguments

PostgreSQL can support calling functions with named notation in suitable contexts.

Example:



```python
SELECT high_expression(
    expr => 25,
    threshold => 15
);
```

This can improve readability when functions have several parameters.


# 11.66 Trigger Management

Triggers can be dropped:



```python
DROP TRIGGER IF EXISTS trg_log_qc_change
ON samples;
```

They can also be disabled and re-enabled administratively:



```python
ALTER TABLE samples
DISABLE TRIGGER trg_log_qc_change;
```


```python
ALTER TABLE samples
ENABLE TRIGGER trg_log_qc_change;
```

Disabling triggers can compromise integrity or auditing, so it should be done only with clear operational justification.


# 11.67 Database Programming Decision Framework

Ask:

### Do I only need a reusable calculation?

Use a function.

### Do I need to return rows?

Use a set-returning or table-returning function.

### Do I need to perform a multi-step operation?

Consider a procedure.

### Should logic execute automatically when data changes?

Consider a trigger.

### Can the rule be expressed as a constraint?

Prefer the constraint.

### Can set-based SQL solve the problem directly?

Prefer set-based SQL before introducing loops.


# 11.68 Realistic Biomedical Architecture

A mature analytical database may contain:

```text
Tables
  ↓
Constraints
  ↓
Views
  ↓
Functions
  ↓
Procedures
  ↓
Triggers
  ↓
Applications / analytical tools
```

Each layer has a different purpose.

Good database architecture avoids using one feature to solve every problem.


# 11.69 Complete Example: Sample QC Workflow

Suppose a sample undergoes QC.

A possible architecture:

```text
samples
sample_qc
sample_qc_audit
```

A function classifies the score.

A procedure finalizes the sample.

A trigger records status changes.

Conceptually:

```text
QC metric
   ↓
classification function
   ↓
procedure updates status
   ↓
trigger writes audit row
```

This demonstrates how stored database components can work together.


# Module 11 Summary

In this module, we learned:

- Stored programs place reusable logic inside PostgreSQL.
- Functions are commonly used to compute and return values or result sets.
- Procedures are invoked using `CALL` and are suited to multi-step actions.
- PostgreSQL supports SQL-language and PL/pgSQL functions.
- PL/pgSQL provides variables and procedural control flow.
- `IF`, `ELSIF`, `ELSE`, and `CASE` implement conditional logic.
- `LOOP`, `WHILE`, and `FOR` implement iteration.
- Set-based SQL should usually be preferred over row-by-row loops.
- Functions can return scalars, tables, or sets of rows.
- PostgreSQL supports `IN`, `OUT`, and `INOUT` parameter concepts.
- Exceptions can be caught and custom errors raised.
- `RAISE NOTICE`, `WARNING`, and `EXCEPTION` support messaging.
- Triggers execute automatically in response to database events.
- Trigger functions can access `OLD` and `NEW`.
- `BEFORE` triggers can validate or modify incoming rows.
- `AFTER` triggers are useful for auditing and follow-up logic.
- Triggers can operate per row or per statement.
- Constraints should be preferred over triggers when declarative rules are sufficient.
- PostgreSQL function volatility categories are `IMMUTABLE`, `STABLE`, and `VOLATILE`.
- Security-definer functions require careful privilege and search-path design.
- Stored programs are valuable for biomedical QC, auditing, workflow management, and reusable analytical logic.


# Key Takeaway

> **Stored database programs allow PostgreSQL to do more than store and retrieve data—they allow it to enforce and execute reusable workflows.**

For omics and biomedical systems, this can include:

```text
classify variant frequency
calculate gene-expression summaries
validate sample QC
finalize analysis runs
record audit history
normalize incoming identifiers
automatically track status changes
```

However, database programming should remain disciplined:

```text
Use constraints for invariants
Use SQL for set-based operations
Use functions for reusable calculations
Use procedures for explicit workflows
Use triggers for necessary automatic reactions
```

This separation keeps complex PostgreSQL systems understandable and maintainable.


# References

1. PostgreSQL Global Development Group. **PL/pgSQL — SQL Procedural Language**  
   https://www.postgresql.org/docs/current/plpgsql.html

2. PostgreSQL Global Development Group. **CREATE FUNCTION**  
   https://www.postgresql.org/docs/current/sql-createfunction.html

3. PostgreSQL Global Development Group. **CREATE PROCEDURE**  
   https://www.postgresql.org/docs/current/sql-createprocedure.html

4. PostgreSQL Global Development Group. **CALL**  
   https://www.postgresql.org/docs/current/sql-call.html

5. PostgreSQL Global Development Group. **PL/pgSQL Control Structures**  
   https://www.postgresql.org/docs/current/plpgsql-control-structures.html

6. PostgreSQL Global Development Group. **Errors and Messages in PL/pgSQL**  
   https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html

7. PostgreSQL Global Development Group. **Triggers**  
   https://www.postgresql.org/docs/current/triggers.html

8. PostgreSQL Global Development Group. **CREATE TRIGGER**  
   https://www.postgresql.org/docs/current/sql-createtrigger.html

9. PostgreSQL Global Development Group. **Function Volatility Categories**  
   https://www.postgresql.org/docs/current/xfunc-volatility.html

10. Silberschatz, A., Korth, H. F., & Sudarshan, S. *Database System Concepts*. McGraw-Hill.

11. Elmasri, R., & Navathe, S. B. *Fundamentals of Database Systems*. Pearson.

12. Ensembl Documentation  
    https://www.ensembl.org/info/docs/index.html

13. GTEx Portal  
    https://gtexportal.org/

