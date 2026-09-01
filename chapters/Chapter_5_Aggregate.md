# Aggregate Functions, GROUP BY, and HAVING
#### _**COUNT, SUM, AVG, MIN, MAX, GROUP BY, HAVING, and WHERE vs HAVING in SQL**_

#### Introduction: From Retrieving Records to Analyzing Data

Until now, most of our SQL queries have answered questions such as:

* Which patients have diabetes?0
* Which samples were obtained from blood?
* Which genes have expression values above a threshold?
* What are the ten highest expression measurements?

These questions retrieve **individual records**.

* But real-world data analysis frequently requires a different kind of question:
* How many patients have diabetes?
* What is the average age of patients?
* How many samples were collected from each tissue?
* What is the average expression of each gene?
* Which genes have the highest average expression?
* Which tissues contain more than 100 samples?

To answer such questions, SQL provides **aggregate functions**.

An **aggregate function** takes multiple rows of data and produces a **summary value**.

For example, imagine these ages:

```bash
45
52
38
61
54
```

Instead of returning all five values, SQL can calculate:

```bash
COUNT = 5
AVG   = 50
MIN   = 38
MAX   = 61
SUM   = 250
```
This transition—from retrieving individual observations to **summarizing groups of observations**—is one of the most important steps in learning SQL for **data science, analytics, epidemiology, genomics, transcriptomics, and other omics disciplines**.

---

## 4.1 What Is an Aggregate Function?

An **aggregate function** performs a calculation over a collection of rows and returns a summarized result.

Common SQL aggregate functions include:

| Function  | Purpose                   |
| --------- | ------------------------- |
| `COUNT()` | Count observations        |
| `SUM()`   | Add numerical values      |
| `AVG()`   | Calculate arithmetic mean |
| `MIN()`   | Find smallest value       |
| `MAX()`   | Find largest value        |


These functions are often combined with:

* `WHERE`
* `GROUP BY`
* `HAVING`
* `ORDER BY`

Together, these constructs form the foundation of **analytical SQL**.

## 4.2 The Dataset Used in This Chapter

We will continue using our biomedical database.

`patients`

| patient_id | sex | age | diagnosis       |
| ---------- | --- | --: | --------------- |
| 101        | F   |  52 | Type 2 Diabetes |
| 102        | M   |  45 | Healthy         |
| 103        | F   |  61 | Type 2 Diabetes |
| 104        | M   |  38 | Hypertension    |
| 105        | F   |  47 | Healthy         |
| 106        | M   |  55 | Type 2 Diabetes |

`samples`

| sample_id | patient_id | tissue |
| --------- | ---------: | ------ |
| S001      |        101 | Blood  |
| S002      |        101 | Liver  |
| S003      |        102 | Blood  |
| S004      |        103 | Brain  |
| S005      |        104 | Blood  |
| S006      |        105 | Liver  |
| S007      |        106 | Blood  |
| S008      |        106 | Liver  |

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
| S006      | ENSG004 |             25.6 |
| S007      | ENSG001 |             14.1 |
| S007      | ENSG002 |             19.7 |

We will use these simplified tables to understand the concepts before eventually moving to larger datasets such as GTEx and Ensembl.

----

## 4.3 COUNT() — Counting Records
**Concept**

`COUNT()` determines the **number of rows or non-NULL values** represented by a query.

It is one of the most frequently used functions in SQL.

#### Why do we need COUNT?

In biomedical research, counting is fundamental.

We frequently need to determine:

* Number of patients
* Number of cases and controls
* Number of samples
* Number of genes
* Number of variants
* Number of samples per tissue
* Number of patients per diagnosis
* Number of observations passing QC

---

#### Example 1: Count All Patients
**Question**

> How many patients are present in our database?

```bash
SELECT COUNT(*)
FROM patients;
```

Result

```bash
count
.....
6
```

#### What does COUNT(*) mean?

The asterisk tells SQL:

> Count the rows returned by the query.

Therefore:

```bash
COUNT(*)
```

counts rows regardless of whether individual columns contain `NULL`.

----

## 4.4 Giving Aggregates Meaningful Names with Aliases

The previous result appears as:

```bash
count
......
6
```

We can make it more descriptive using a **column alias**.

```bash
SELECT COUNT(*) AS total_patients
FROM patients;
```

`AS` creates an alias for the output column.

This does **not rename the actual database column**. It only changes the label displayed in the query result.

Aliases become especially important when constructing reports and analytical queries.

---

## 4.5 COUNT(*) vs COUNT(column)

This distinction is extremely important.

Consider:

```bash
SELECT COUNT(*)
FROM patients;
```

versus

```bash
SELECT COUNT(diagnosis)
FROM patients;
```

They are not necessarily equivalent.

`COUNT(*)`

Counts **all rows**.

`COUNT(diagnosis)`

Counts only rows where `diagnosis` is **not NULL**.

Suppose we have:

| patient_id | diagnosis    |
| ---------- | ------------ |
| 101        | Diabetes     |
| 102        | Healthy      |
| 103        | NULL         |
| 104        | Hypertension |

Then:

```bash
SELECT COUNT(*)
FROM patients;
```

returns

```bash
4
```

But:

```bash
SELECT COUNT(diagnosis)
FROM patients;
```

returns:

```bash
3
```

because SQL excludes the `NULL` diagnosis.

#### Important principle

> Most SQL aggregate functions ignore NULL values.

We will revisit this behavior because it has major implications for data analysis.

----

## 4.6 COUNT(DISTINCT) — Counting Unique Values

Suppose our samples come from three tissues:

```bash
Blood
Liver
Blood
Brain
Blood
Liver
Blood
Liver
```

If we write:

```bash
SELECT COUNT(tissue)
FROM samples;
```

SQL counts all non-NULL tissue observations.

Result:

```bash
8
```

But perhaps our biological question is:

> How many different tissue types are represented?

We need `DISTINCT`.

```bash
SELECT COUNT(DISTINCT tissue) AS tissue_types
FROM samples;
```

Result:

```bash
3
```

because the unique tissues are:

```bash
Blood
Liver
Brain
```

#### Omics applications

`COUNT(DISTINCT ...)` is extremely useful for questions such as:

* Number of unique genes
* Number of unique subjects
* Number of tissues represented
* Number of chromosomes
* Number of experimental platforms
* Number of variants associated with at least one gene

----

## 4.7 COUNT with WHERE

Aggregate functions can be combined with filtering.

**Clinical question**

> How many patients have Type 2 Diabetes?

```bash
SELECT COUNT(*) AS diabetes_patients
FROM patients
WHERE diagnosis = 'Type 2 Diabetes';
```

SQL first filters the table using WHERE.

Conceptually:

```bash
All patients
      ↓
WHERE diagnosis = 'Type 2 Diabetes'
      ↓
Matching patients
      ↓
COUNT(*)
      ↓
3
```

This illustrates an important SQL principle:

> **WHERE filters rows before aggregation takes place.**

---


## 4.8 SUM() — Adding Numerical Values
**Concept**

`SUM()` calculates the total of a numerical column.

General syntax:

```bash
SELECT SUM(column_name)
FROM table_name;
```

**Why do we need SUM?**

It is useful for:

* Total sales
* Total expenditure
* Total read counts
* Total sequencing depth
* Total number of mutations
* Total expression counts
* Total hospital visits

Real-Life Example

Suppose a table contains healthcare expenditure:

```bash
patient_id    treatment_cost
101           5000
102           7500
103           4000
```

We could calculate total expenditure:

```bash
SELECT SUM(treatment_cost) AS total_cost
FROM treatments;
```

Result:

```bash
16500
```

#### Omics Example

Suppose read_count contains raw RNA-seq counts.

```bash
SELECT SUM(read_count) AS total_reads
FROM gene_counts;
```

This might calculate the total number of reads represented in a particular query.

---

## 4.9 AVG() — Calculating the Mean
**Concept**

`AVG()` calculates the **arithmetic mean** of non-NULL numerical values.

Mathematically:

$$
Mean = \sum {x_i}/n
$$

SQL performs this calculation automatically.

#### Clinical Example
**Question**

> What is the average age of patients?

```bash
SELECT AVG(age) AS average_age
FROM patients;
```

Using our sample data:

```bash
52
45
61
38
47
55
```

the result is approximately:

```bash
49.67
```

#### Why AVG Is Important

Average values are frequently used in:

* Descriptive statistics
* Clinical cohort summaries
* Quality-control reports
* Expression summaries
* Experimental measurements

---

## 4.10 Omics Example: Average Gene Expression

Suppose we want the average expression across all measurements.

```bash
SELECT AVG(expression_value) AS mean_expression
FROM expression;
```

SQL collects all non-NULL `expression_value` values and calculates their arithmetic mean.

But this query asks:

> What is the average expression across **everything**?

Biologically, that may not be particularly useful.

A more interesting question is:

> What is the average expression **for each gene**?

That requires `GROUP BY`.

This is where aggregate functions become significantly more powerful.

---

## 4.11 MIN() — Finding the Minimum

`MIN()` returns the smallest value.

**Clinical Example**

> What is the age of the youngest patient?

```bash
SELECT MIN(age) AS youngest_age
FROM patients;
```

Result:

```bash
38
```

#### Omics Example

> What is the lowest observed expression value?

```bash
SELECT MIN(expression_value) AS minimum_expression
FROM expression;
```

`MIN()` can also work with several non-numeric data types, including dates.

For example:

```bash
SELECT MIN(collection_date) AS earliest_collection
FROM samples;
```

This returns the earliest sample collection date.

---

## 4.12 MAX() — Finding the Maximum

`MAX()` performs the opposite operation.

**Clinical Example**

> What is the age of the oldest patient?

```bash
SELECT MAX(age) AS oldest_age
FROM patients;
```

**Omics Example**

> What is the highest expression value in the dataset?

```bash
SELECT MAX(expression_value) AS maximum_expression
FROM expression;
```

---

## 4.13 Using Multiple Aggregate Functions Together

SQL allows several aggregate calculations in one query.

**Clinical Example**

```bash
SELECT
    COUNT(*) AS total_patients,
    AVG(age) AS average_age,
    MIN(age) AS youngest_age,
    MAX(age) AS oldest_age
FROM patients;
```
The result might look like:

| total_patients | average_age | youngest_age | oldest_age |
| -------------: | ----------: | -----------: | ---------: |
|              6 |       49.67 |           38 |         61 |

This single query provides a basic descriptive statistical summary of the patient cohort.

---

## 4.14 GROUP BY — The Foundation of Analytical SQL

`GROUP BY` is one of the most important concepts in SQL.

Without `GROUP BY`, an aggregate generally summarizes the **entire result set**.

For example:

```bash
SELECT AVG(age)
FROM patients;
```

asks:

> What is the average age of **all patients**?

But what if we want:

> What is the average age for **males and females separately**?

Now SQL needs to divide the rows into groups.

That is the purpose of `GROUP BY`.

---

## 4.15 Understanding GROUP BY in Layman Terms

Imagine six patients:

| Patient | Sex | Age |
| ------- | --- | --: |
| P1      | F   |  52 |
| P2      | M   |  45 |
| P3      | F   |  61 |
| P4      | M   |  38 |
| P5      | F   |  47 |
| P6      | M   |  55 |

We ask SQL:

```bash
SELECT sex, AVG(age)
FROM patients
GROUP BY sex;
```

Conceptually, SQL separates the rows:

```bash
                PATIENTS
                   |
         ---------------------
         |                   |
       Female               Male
         |                   |
      52,61,47            45,38,55
         |                   |
       AVG()               AVG()
         |                   |
       53.33               46.00
```

The result is:

| sex |   avg |
| --- | ----: |
| F   | 53.33 |
| M   | 46.00 |

**The key idea**

> GROUP BY creates groups of rows that share the same value, and aggregate functions summarize each group independently.

---

## 4.16 Why GROUP BY Is So Important

`GROUP BY` transforms SQL from a data-retrieval language into a powerful **analytical language**.

It allows questions such as:

* Patients per disease
* Average age per diagnosis
* Samples per tissue
* Genes per chromosome
* Variants per chromosome
* Average expression per gene
* Average expression per tissue
* Samples per sequencing platform
* Cases per population
* Variants per consequence category

These are fundamental questions in biomedical data analysis.

---

## 4.17 Clinical Example: Patients per Diagnosis
Question

> How many patients belong to each diagnostic category?

```bash
SELECT
    diagnosis,
    COUNT(*) AS patient_count
FROM patients
GROUP BY diagnosis;
```

Possible result:

| diagnosis       | patient_count |
| --------------- | ------------: |
| Type 2 Diabetes |             3 |
| Healthy         |             2 |
| Hypertension    |             1 |

#### What SQL Is Doing

Conceptually:

```bash
patients
   ↓
GROUP BY diagnosis
   ↓
--------------------------------
Diabetes    Healthy    Hypertension
   ↓           ↓             ↓
COUNT(*)    COUNT(*)       COUNT(*)
   ↓           ↓             ↓
   3           2             1
```

#### 4.18 Omics Example: Number of Samples per Tissue
**Biological question**

> How many biological samples were collected from each tissue?

```bash
SELECT
    tissue,
    COUNT(*) AS sample_count
FROM samples
GROUP BY tissue;
```

Possible result:

| tissue | sample_count |
| ------ | -----------: |
| Blood  |            4 |
| Liver  |            3 |
| Brain  |            1 |

This type of query is extremely useful for examining **sample distribution** before downstream analysis.

For example, a researcher may discover that:

```bash
Blood = 1,000 samples
Brain = 40 samples
Liver = 600 samples
```

That imbalance could affect statistical analyses and should be understood before modeling.

---


## 4.19 Omics Example: Genes per Chromosome

Suppose our `genes` table contains thousands of genes.

We can ask:

> How many genes are represented on each chromosome?

```bash
SELECT
    chromosome,
    COUNT(*) AS gene_count
FROM genes
GROUP BY chromosome;
```

For a larger Ensembl-derived dataset, the result might conceptually resemble:

```bash
chromosome    gene_count
----------    ----------
1             ...
2             ...
3             ...
...
X             ...
Y             ...
```

This is a simple example of **genome-level summarization** using SQL.

----

## 4.20 Average Expression per Gene

Now consider a much more important transcriptomics example.

Suppose each gene has expression measurements from many samples.

The question is:

> What is the average expression of each gene across the samples?

```bash
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id;
```

Conceptually:

```bash
Gene A → 12.5, 10.4, 14.1 → AVG → 12.33
Gene B → 18.2, 21.3, 19.7 → AVG → 19.73
Gne C → 35.8             → AVG → 35.80
```

SQL performs this operation for every distinct `gene_id`.

--

## 4.21 GROUP BY with JOIN

In real databases, identifiers often exist in one table while descriptive information exists in another.

For example:

`expression` may contain:

```bash
gene_id
ENSG001
```

while genes contains:

```bash
ENSG001 → BRCA1
```

We can combine them.

```bash
SELECT
    g.gene_symbol,
    AVG(e.expression_value) AS mean_expression
FROM expression AS e
JOIN genes AS g
    ON e.gene_id = g.gene_id
GROUP BY g.gene_symbol;
```

Now the result is easier to interpret:

| gene_symbol | mean_expression |
| ----------- | --------------: |
| BRCA1       |           12.33 |
| TP53        |           19.73 |
| APOE        |           35.80 |
| MYC         |           25.60 |

This demonstrates the power of combining:

```bash
JOIN
+
GROUP BY
+
Aggregate Function
```

These three concepts appear constantly in real-world SQL.

----

## 4.22 GROUP BY Multiple Columns

SQL can group by more than one variable.

Suppose we want:

> Number of patients by **sex and diagnosis**.

```bash
SELECT
    sex,
    diagnosis,
    COUNT(*) AS patient_count
FROM patients
GROUP BY sex, diagnosis;
```

Now SQL creates combinations such as:

```bash
Female + Diabetes
Male + Diabetes
Female + Healthy
Male + Healthy
Male + Hypertension
```

This is similar to **stratification** in statistics.

----

#### Omics Interpretation

The same principle can be used for:

```bash
tissue + sex
tissue + disease
gene + tissue
chromosome + consequence
population + allele
cancer_type + mutation_type
```

For example:

```bash
SELECT
    tissue,
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY tissue, gene_id;
```

assuming `tissue` exists in that analytical table.

Now PostgreSQL calculates gene expression separately for every:

```bash
Gene × Tissue
```
combination.

This begins to resemble real **GTEx-style analytical querying**.

----

## 4.23 An Important GROUP BY Rule

Consider:

```bash
SELECT
    diagnosis,
    sex,
    COUNT(*)
FROM patients
GROUP BY diagnosis;
```

PostgreSQL will generally reject this query because sex is neither:

* included in GROUP BY, nor
* used inside an aggregate function.

Why?

Imagine:

```bash
diagnosis          sex
Diabetes           F
Diabetes           M
Diabetes           F
```

After grouping all Diabetes patients together, PostgreSQL has one group.

But what value should it display for `sex`?

```bash
F?
M?
```

There is no single correct answer.

Therefore, the general rule is:

> A selected column in an aggregate query normally needs to be either **aggregated** or **part of the grouping definition**.

A valid query is:

```bash
SELECT
    diagnosis,
    sex,
    COUNT(*) AS patient_count
FROM patients
GROUP BY diagnosis, sex;
```

----

## 4.24 HAVING — Filtering Groups

This is another fundamental concept.

We already know:

```bash
WHERE
```

filters **rows**.

But after `GROUP BY`, we sometimes need to filter **groups** based on aggregate results.

That is the purpose of:

```bash
HAVING
```

----

## 4.25 Why WHERE Cannot Replace HAVING

Suppose we ask:

> Show only tissues having more than two samples.

First, SQL needs to calculate:

```bash
Blood → 4
Liver → 3
Brain → 1
```

Then we want:

```bash
Blood → 4
Liver → 3
```

but not:

```bash
Brain → 1
```

We cannot determine this until the groups have been created and counted.

Therefore:

```bash
SELECT
    tissue,
    COUNT(*) AS sample_count
FROM samples
GROUP BY tissue
HAVING COUNT(*) > 2;
```
Result:

| tissue | sample_count |
| ------ | -----------: |
| Blood  |            4 |
| Liver  |            3 |

----

## 4.26 WHERE vs HAVING

This distinction must be understood thoroughly.

WHERE

Filters **individual rows before grouping**.

HAVING

Filters **groups after aggregation**.

Consider:

```bash
Raw rows
   ↓
WHERE
   ↓
Filtered rows
   ↓
GROUP BY
   ↓
Groups
   ↓
Aggregate functions
   ↓
HAVING
   ↓
Filtered groups
```

This mental model is extremely useful.

---

## 4.27 Clinical Example of HAVING
**Question**

> Which diagnoses occur in at least two patients?

```bash
SELECT
    diagnosis,
    COUNT(*) AS patient_count
FROM patients
GROUP BY diagnosis
HAVING COUNT(*) >= 2;
```

Possible result:

| diagnosis       | patient_count |
| --------------- | ------------: |
| Type 2 Diabetes |             3 |
| Healthy         |             2 |


Hypertension is excluded because only one patient has that diagnosis.

----

## 4.28 Omics Example of HAVING
**Question**

> Which genes have an average expression greater than 15?

```bash
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id
HAVING AVG(expression_value) > 15;
```

This is different from:

```bash
WHERE expression_value > 15
```

The distinction is scientifically important.

---

## 4.29 WHERE vs HAVING in an Expression Study

Suppose gene A has expression values:

```bash
5
10
20
25
```

**Query A**
```bash
SELECT
    gene_id,
    AVG(expression_value)
FROM expression
WHERE expression_value > 10
GROUP BY gene_id;
```

`WHERE` removes:

```bash
5
10
```

before calculating the mean.

The mean is therefore calculated using:

```bash
20
25
```

**Query B**

```bash
SELECT
    gene_id,
    AVG(expression_value)
FROM expression
GROUP BY gene_id
HAVING AVG(expression_value) > 10;
```

Now SQL first calculates the average using:

```bash
5
10
20
25
```

and **then** determines whether that gene's average exceeds 10.

These queries answer **different biological questions**.

This distinction is extremely important because incorrect placement of filtering conditions can change the scientific interpretation of the results.

---

## 4.30 Combining WHERE, GROUP BY, and HAVING

Now we can construct more realistic analytical queries.

**Clinical Question**

> Among patients older than 40, which diagnoses have at least two patients?

```bash
SELECT
    diagnosis,
    COUNT(*) AS patient_count
FROM patients
WHERE age > 40
GROUP BY diagnosis
HAVING COUNT(*) >= 2;
```

Conceptually:

```bash
patients
    ↓
WHERE age > 40
    ↓
eligible patients
    ↓
GROUP BY diagnosis
    ↓
diagnostic groups
    ↓
COUNT(*)
    ↓
HAVING COUNT(*) >= 2
```

Notice that `WHERE` and `HAVING` have completely different responsibilities.

----

## 4.31 Combining Aggregation with ORDER BY

Suppose we want:

> Rank tissues according to the number of samples.

```bash
SELECT
    tissue,
    COUNT(*) AS sample_count
FROM samples
GROUP BY tissue
ORDER BY sample_count DESC;
```

Result:

| tissue | sample_count |
| ------ | -----------: |
| Blood  |            4 |
| Liver  |            3 |
| Brain  |            1 |

This pattern is extremely common:

```bash
SELECT
    category,
    COUNT(*) AS count
FROM table
GROUP BY category
ORDER BY count DESC;
```

It can be applied to:

* Diagnoses
* Tissues
* Genes
* Chromosomes
* Variants
* Cancer types
* Mutation consequences
* Populations
* Sequencing platforms

----

## 4.32 Top Genes by Average Expression

We can combine several concepts from Modules 2–4.

**Question**

> Which five genes have the highest average expression?

```bash
SELECT
    g.gene_symbol,
    AVG(e.expression_value) AS mean_expression
FROM expression AS e
JOIN genes AS g
    ON e.gene_id = g.gene_id
GROUP BY g.gene_symbol
ORDER BY mean_expression DESC
LIMIT 5;
```

This query contains:

* `SELECT`
* `JOIN`
* `AVG()`
* `GROUP BY`
* `ORDER BY`
* `LIMIT`

Although the query is still relatively simple, it already represents a realistic **analytical SQL workflow**.

----

## 4.33 PostgreSQL FILTER — Conditional Aggregation

SQL provides a particularly useful syntax called the aggregate FILTER clause.

Suppose we want to calculate total patients, female patients, and male patients in one query.

```bash
SELECT
    COUNT(*) AS total_patients,
    COUNT(*) FILTER (WHERE sex = 'F') AS female_patients,
    COUNT(*) FILTER (WHERE sex = 'M') AS male_patients
FROM patients;
```

Possible result:

| total_patients | female_patients | male_patients |
| -------------: | --------------: | ------------: |
|              6 |               3 |             3 |

This is called **conditional aggregation**.

It is extremely useful in analytical reporting.

**Omics Example**

Suppose a variants table contains:

```bash
variant_id
chromosome
consequence
```

We could calculate multiple consequence categories simultaneously:

```bash
SELECT
    chromosome,
    COUNT(*) AS total_variants,
    COUNT(*) FILTER (
        WHERE consequence = 'missense_variant'
    ) AS missense_variants,
    COUNT(*) FILTER (
        WHERE consequence = 'synonymous_variant'
    ) AS synonymous_variants
FROM variants
GROUP BY chromosome;
```

This could produce:

| chromosome | total_variants | missense_variants | synonymous_variants |
| ---------- | -------------: | ----------------: | ------------------: |
| 1          |         152340 |              8214 |                6132 |
| 2          |         164291 |              7912 |                5943 |
| 3          |         138920 |              7104 |                5201 |

This is a highly useful pattern for **genomic summary tables**.

---

## 4.34 NULL Values and Aggregate Functions

NULL handling deserves special attention.

Suppose ages are:

```bash
52
45
NULL
61
```

Then:

```bash
SELECT COUNT(*)
FROM patients;
```

returns:

```bash
4
```

while:

```bash
SELECT COUNT(age)
FROM patients;
```

returns:

```bash
3
```

Similarly:

```bash
AVG(age)
```

normally ignores the NULL value.

It does **not automatically treat NULL as zero**.

This distinction is important.

If missing age values were treated as zero, the average age would be artificially lowered.

----

## 4.35 NULL Is Not Zero

In biological data:

```bash
expression_value = 0
```

and:

```bash
expression_value = NULL
```

can have completely different meanings.

`0` may mean:

> The gene was measured and the observed value was zero.

`NULL` may mean:

> No value is available.

Possible reasons include:

* Measurement not performed
* Assay failure
* Missing annotation
* Data unavailable
* Value not applicable

Confusing **missingness** with a true biological zero can seriously bias downstream analyses.

----

## 4.36 Aggregation vs Window Functions: An Important Preview

Suppose we write:

```bash
SELECT
    tissue,
    AVG(expression_value)
FROM expression
GROUP BY tissue;
```

`GROUP BY` **collapses multiple rows into summary rows.**

For example:

```bash
Blood:
Sample1  10
Sample2  15
Sample3  20
```

becomes:

```bash
Blood  15
```

The individual observations disappear from the result.

Later, we will learn **window functions**, which allow us to calculate summaries **without collapsing the individual rows**.

Conceptually:

```bash
Sample1   Blood   10   Average Blood = 15
Sample2   Blood   15   Average Blood = 15
Sample3   Blood   20   Average Blood = 15
```

This distinction between:

**aggregation**

and

**windowed analytics**

will become central when we reach advanced SQL.

----

## 4.37 Logical Query Processing Order

We can now extend the processing-order model introduced in Module 3.

A simplified conceptual order is:

```bash
FROM
  ↓
JOIN / ON
  ↓
WHERE
  ↓
GROUP BY
  ↓
Aggregate calculations
  ↓
HAVING
  ↓
SELECT
  ↓
ORDER BY
  ↓
LIMIT
```

Consider:

```bash
SELECT
    tissue,
    AVG(expression_value) AS mean_expression
FROM expression
WHERE expression_value IS NOT NULL
GROUP BY tissue
HAVING COUNT(*) >= 10
ORDER BY mean_expression DESC
LIMIT 5;
```

Conceptually PostgreSQL:

1. Determines the source table.
2. Removes rows with missing expression values.
3. Groups remaining observations by tissue.
4. Calculates aggregate statistics.
5 .Keeps groups with at least 10 observations.
6. Produces the requested output columns.
7. Sorts tissues by mean expression.
8. Returns the top five.

Understanding **logical query** processing becomes increasingly important as queries become more complex.

---

## 4.38 Aggregation in Real Omics Research

The concepts in this chapter directly translate into real research questions.

**Genomics**

How many variants occur on each chromosome?

```bash
SELECT
    chromosome,
    COUNT(*) AS variant_count
FROM variants
GROUP BY chromosome;
```

**Transcriptomics**

What is the mean expression of each gene?

```bash
SELECT
    gene_id,
    AVG(expression_value) AS mean_expression
FROM expression
GROUP BY gene_id
```

**Clinical genomics**

How many samples exist for each disease?

```bash
SELECT
    diagnosis,
    COUNT(*) AS sample_count
FROM samples
GROUP BY diagnosis;
```

**Variant annotation**

How many variants belong to each consequence category?

```bash
SELECT
    consequence,
    COUNT(*) AS variant_count
FROM variants
GROUP BY consequence
ORDER BY variant_count DESC;
```

**Population genetics**

What is the average allele frequency by population?

```bash
SELECT
    population,
    AVG(allele_frequency) AS mean_allele_frequency
FROM population_frequencies
GROUP BY population;
```

The same SQL concepts can therefore be applied across very different omics domains.

----

## 4.39 Common Beginner Mistakes
**Mistake 1 — Confusing COUNT(*) and COUNT(column)**

```bash
COUNT(*)
```

counts rows.

```bash
COUNT(column)
```

counts non-NULL values in that column.

**Mistake 2 — Using WHERE for aggregate filtering**

Incorrect:

```bash
SELECT tissue, COUNT(*)
FROM samples
WHERE COUNT(*) > 10
GROUP BY tissue;
```

Correct:

```bash
SELECT tissue, COUNT(*)
FROM samples
GROUP BY tissue
HAVING COUNT(*) > 10;
```

**Mistake 3 — Selecting columns that are not grouped**

Problematic:

```bash
SELECT tissue, sample_id, COUNT(*)
FROM samples
GROUP BY tissue;
```

`sample_id` does not represent a single value for the tissue group.

**Mistake 4 — Assuming NULL means zero**

```bash
NULL ≠ 0
```

This is particularly important in scientific databases.

**Mistake 5 — Using GROUP BY when it is unnecessary**

If you simply need:

```bash
SELECT COUNT(*)
FROM patients;
```

there is no reason to use `GROUP BY`.

Use grouping when you want **separate summaries for categories or combinations of categories**.

## 4.40 Building the Analytical Mental Model

At this stage, think about SQL questions in three steps.

**Step 1 — Which observations do I want?**

Use:

```bash
WHERE
```

Example:

> Only female patients older than 50.

**Step 2 — How should those observations be divided?**

Use:

```bash
GROUP BY
```

Example:

> Divide them according to diagnosis.

**Step 3 — What should I calculate for each group?**

Use:

```bash
COUNT()
AVG()
SUM()
MIN()
MAX()
```

Example:

> Calculate the number of patients and their average age.

The complete query becomes:

```bash
SELECT
    diagnosis,
    COUNT(*) AS patient_count,
    AVG(age) AS average_age
FROM patients
WHERE sex = 'F'
  AND age > 50
GROUP BY diagnosis;
```

Then, if necessary:

**Step 4 — Which summarized groups should remain?**

Use:

```bash
HAVING
```

For example:

```bash
SELECT
    diagnosis,
    COUNT(*) AS patient_count,
    AVG(age) AS average_age
FROM patients
WHERE sex = 'F'
  AND age > 50
GROUP BY diagnosis
HAVING COUNT(*) >= 2;
```

This four-step mental framework will remain useful throughout advanced SQL.

----

## 4.41 Module 4 Summary

The core concepts introduced in this chapter are:

* **Aggregate functions** summarize multiple observations.
* `COUNT()` measures frequency.
* `SUM()` calculates totals.
* `AVG()` calculates arithmetic means.
* `MIN()` and `MAX()` identify extreme values.
* `COUNT(DISTINCT ...)` counts unique non-NULL values.
* `GROUP BY` divides rows into groups before aggregation.
* `WHERE` filters **rows before aggregation**.
* `HAVING` filters **groups after aggregation**.
* `ORDER BY` can rank aggregate results.
* SQL's `FILTER` clause supports elegant **conditional aggregation**.
* Aggregate functions generally ignore `NULL` values.
* Aggregation collapses rows, whereas window functions, which we will study later, can calculate group-level statistics while retaining individual observations.

#### Key Takeaway

> `WHERE` **decides which observations enter the analysis**. `GROUP BY` **decides how those observations are stratified. Aggregate functions decide what is calculated, and** `HAVING` **decides which summarized groups remain**.

For data science and omics, this is much more than SQL syntax. It is effectively a way of expressing an analytical question as a database query.

----

#### References
* PostgreSQL Documentation — Aggregate Functions
* PostgreSQL Documentation — SELECT
* PostgreSQL Documentation — Aggregate FILTER syntax
* Silberschatz, A., Korth, H. F., & Sudarshan, S. Database System Concepts. McGraw-Hill.
* Elmasri, R., & Navathe, S. B. Fundamentals of Database Systems. Pearson.
* Date, C. J. An Introduction to Database Systems. Addison-Wesley.
* ISO/IEC 9075 — Information technology — Database languages SQL.
* Ensembl Documentation — reference for genomic data organization and biological database concepts.
* GTEx Portal — reference resource for tissue-specific gene-expression data and metadata.


```python

```
