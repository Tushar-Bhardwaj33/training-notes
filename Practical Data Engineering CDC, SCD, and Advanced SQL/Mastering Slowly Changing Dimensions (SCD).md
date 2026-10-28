# Mastering Slowly Changing Dimensions (SCD)

## Introduction to Slowly Changing Dimensions: Types and Use Cases

### What Are Slowly Changing Dimensions (SCD)?

Slowly Changing Dimensions (SCD) refer to a data management and integration concept in data warehousing. It involves managing changes in dimension tables over time. Dimension tables store descriptive attributes about the facts in your data model, such as customer details, product information, or location data. Since these attributes can change over time, SCD techniques help maintain historical accuracy while enabling analysis of both current and past states of the data.

- Preserve historical data for accurate reporting and trend analysis.
- Support analytics that require comparing current vs. historical states.
- Enable organizations to track changes in business entities over time.

### Why Are SCDs Important in Data Warehousing?

SCDs play a critical role in building robust data warehouses. They provide the ability to maintain historical records for dimensions, which is essential for trend analysis and compliance. For example, in a retail scenario, understanding how a customer’s profile has evolved over time can help in designing targeted marketing campaigns and measuring their impact.

#### What is the primary purpose of Slowly Changing Dimensions?

To manage changes in dimension data over time while preserving historical data for analysis.

#### Why are SCDs critical in data warehousing?

They enable accurate historical analysis and trend reporting by maintaining prior states of data.

#### What type of data do dimension tables typically store?

Descriptive attributes about facts, such as customer details, product information, or geographic locations.

### Type 0: Fixed Dimensions

Retain the original value and reject or separately audit later changes to that attribute. Use this for attributes whose original meaning must remain fixed, such as an original acquisition date. Type 0 does not store a sequence of later values.

### Type 1: Overwrite Dimensions

Replace the stored value when the business attribute changes. Use this for corrections or attributes for which only the latest value matters. Earlier values cannot be reconstructed from the dimension, and historical reports joined to it will use the replacement value.

### Type 2: Historical Row Versioning

Insert a new dimension row for each accepted change and close the previous row. Keep a business key, a unique surrogate key per version, `valid_from`, `valid_to`, and an optional current-row flag. Use half-open intervals: a version applies when `event_time >= valid_from AND (event_time < valid_to OR valid_to IS NULL)`. Exactly one current version is allowed per active business key.

### Type 3: Add New Column

Store limited history in extra columns, such as `current_region` and `previous_region`, on the same entity row. This supports comparing current and previous values without a historical join. Another change replaces the previous value; Type 3 cannot reconstruct an unlimited sequence of states.

### Type 4: Historical Table

Keep current attributes in one dimension table and historical versions in a separate history table. Latest-state queries read the current table; historical queries read the history table using version dates. This course uses the separate-history-table definition of Type 4, since naming conventions vary across modeling references.

### Type 6: Hybrid SCD

Combine Type 1, Type 2, and Type 3 behavior. For example, retain versioned rows, store the prior region on each version, and maintain a current-region attribute across versions. Document which columns describe the historical state and which intentionally show the latest state so analysts do not accidentally mix meanings.

> Type 2 SCD Example: Tracking Historical Changes

| customer_key | customer_id | region | valid_from | valid_to | is_current |
| --- | --- | --- | --- | --- | --- |
| 101 | 1 | North | 2026-01-01 | 2026-02-01 | false |
| 102 | 1 | South | 2026-02-01 | NULL | true |

A January sale references key `101`; a sale on or after February 1 references key `102`. The end date is exclusive.

### Understand the purpose of SCDs.

Recognize why tracking changes in dimension tables is essential for analytical workloads.

#### Which type of Slowly Changing Dimension creates a new row for each change, allowing for historical tracking?

- Type 0
- Type 1
- Type 2
- Type 3

**Answer:** **Type 2.** Each accepted change receives a new version row while earlier versions remain queryable.

- **Step 1: Identify the Dimension Attributes**

  Determine the attributes in your dimension table that are subject to change over time.

- **Step 2: Choose the Appropriate SCD Type**

  For each attribute, decide whether it is immutable, should overwrite, needs full history, or needs only a prior value. Record the analytical reason for that decision.

- **Step 3: Implement the SCD Logic**

  Implement the selected overwrite or versioning rules, with a stable business key, deterministic change detection, and explicit effective-time boundaries.

- **Step 4: Validate and Test**

  Test a new entity, a changed entity, an unchanged replay, a deletion, and a late change. Verify that each active business key has exactly one current row.

#### In a Type ___ Slowly Changing Dimension, changes are tracked by creating a new row for each version of the record.

**Answer:** **2.** The business key repeats across versions, while each version has its own surrogate key.

## Implementing SCD Type 1, Type 2, and Type 3 with SQL

### Introduction to Implementing SCD Types with SQL

Slowly Changing Dimensions (SCDs) are a crucial concept in data warehousing, allowing you to manage changes in dimension data over time. In this lesson, we will explore the implementation of SCD Type 1, Type 2, and Type 3 using SQL. Each type serves a distinct purpose in managing updates and historical data in dimension tables.

### SCD Type 1: Overwrite Dimensions

The following independent examples use **Snowflake SQL** and temporary tables in the current database and schema. Run each block in order within its own example. Type 1 overwrites an existing customer and inserts a new customer; `customer_id` must be unique in the staged input.

```sql
CREATE TEMP TABLE dim_customer_t1 (customer_id NUMBER, region VARCHAR);
INSERT INTO dim_customer_t1 VALUES (1, 'North');
CREATE TEMP TABLE stage_customer_t1 (customer_id NUMBER, region VARCHAR);
INSERT INTO stage_customer_t1 VALUES (1, 'South'), (2, 'West');

MERGE INTO dim_customer_t1 AS t
USING stage_customer_t1 AS s ON t.customer_id = s.customer_id
WHEN MATCHED AND t.region IS DISTINCT FROM s.region
  THEN UPDATE SET region = s.region
WHEN NOT MATCHED THEN INSERT (customer_id, region)
  VALUES (s.customer_id, s.region);

SELECT * FROM dim_customer_t1 ORDER BY customer_id;
```

Expected state: customer `1` has region `South`; customer `2` has region `West`. The original `North` value is not retained.

### SCD Type 2: Historical Row Versioning

This example accepts at most one change per customer per batch, in strictly increasing effective-time order for an existing customer. Deduplicate the stage and reject equal-time conflicts before the transaction. If every intermediate change must be preserved, process those changes in event order instead of reducing them to the latest row.

```sql
CREATE TEMP TABLE dim_customer_t2 (
  customer_key NUMBER AUTOINCREMENT,
  customer_id NUMBER,
  region VARCHAR,
  valid_from TIMESTAMP_NTZ,
  valid_to TIMESTAMP_NTZ,
  is_current BOOLEAN
);
INSERT INTO dim_customer_t2
  (customer_id, region, valid_from, valid_to, is_current)
VALUES (1, 'North', '2026-01-01 00:00:00', NULL, TRUE);

CREATE TEMP TABLE stage_customer_t2 (
  customer_id NUMBER, region VARCHAR, effective_at TIMESTAMP_NTZ
);
INSERT INTO stage_customer_t2 VALUES
  (1, 'South', '2026-02-01 00:00:00'),
  (2, 'West', '2026-02-01 00:00:00');

BEGIN TRANSACTION;

UPDATE dim_customer_t2 AS t
SET valid_to = s.effective_at, is_current = FALSE
FROM stage_customer_t2 AS s
WHERE t.customer_id = s.customer_id
  AND t.is_current
  AND s.effective_at > t.valid_from
  AND t.region IS DISTINCT FROM s.region;

INSERT INTO dim_customer_t2
  (customer_id, region, valid_from, valid_to, is_current)
SELECT s.customer_id, s.region, s.effective_at, NULL, TRUE
FROM stage_customer_t2 AS s
WHERE NOT EXISTS (
  SELECT 1 FROM dim_customer_t2 AS t
  WHERE t.customer_id = s.customer_id AND t.is_current
);

COMMIT;

SELECT customer_id, region, valid_from, valid_to, is_current
FROM dim_customer_t2 ORDER BY customer_id, valid_from;
```

Customer `1` has a closed `North` version ending on February 1 and a current `South` version beginning then. Customer `2` has one current `West` version. Replaying this unchanged batch creates no extra versions. This example does not implement deletes or out-of-order corrections; those require an explicit policy and interval repair. Serialize writers for the same dimension, and roll back both statements if either fails.

### SCD Type 3: Add New Column

Keep only the latest region and one prior region. In this Snowflake example, a repeated unchanged batch does not shift the previous value again.

```sql
CREATE TEMP TABLE dim_customer_t3 (
  customer_id NUMBER, current_region VARCHAR, previous_region VARCHAR
);
INSERT INTO dim_customer_t3 VALUES (1, 'North', NULL);
CREATE TEMP TABLE stage_customer_t3 (customer_id NUMBER, region VARCHAR);
INSERT INTO stage_customer_t3 VALUES (1, 'South'), (2, 'West');

MERGE INTO dim_customer_t3 AS t
USING stage_customer_t3 AS s ON t.customer_id = s.customer_id
WHEN MATCHED AND t.current_region IS DISTINCT FROM s.region
  THEN UPDATE SET previous_region = t.current_region,
                  current_region = s.region
WHEN NOT MATCHED THEN INSERT (customer_id, current_region, previous_region)
  VALUES (s.customer_id, s.region, NULL);

SELECT * FROM dim_customer_t3 ORDER BY customer_id;
```

Customer `1` has current region `South` and previous region `North`. A subsequent change to `East` would retain only `East` and `South`.

- **Step 1: Analyze Change Requirements**

  Determine the type of changes that need to be tracked and whether historical data is required for your dimension table.

- **Step 2: Design the Schema**

  Create version keys and validity columns for Type 2, current/prior attribute columns for Type 3, and a unique staged business key for all types.

- **Step 3: Implement SQL Logic**

  Run the corresponding SQL example. For Type 2, expire and insert as one atomic unit using a transaction on Snowflake; do not commit an expired row without its replacement.

- **Step 4: Validate and Test**

  Check expected row counts and attributes, ensure intervals do not overlap, and replay the same input to verify that it produces no duplicate versions.

> SQL Implementation of SCD Type 2: Managing Historical Rows

The Type 2 SQL block above first closes changed current rows, then inserts their replacements and new entities in the same Snowflake transaction. It compares attributes rather than generating a new version for every incoming row.

#### Which operation belongs in a MERGE statement's WHEN MATCHED branch to overwrite existing attributes for SCD Type 1?

- INSERT
- UPDATE
- DELETE
- MERGE

**Answer:** **UPDATE.** A MERGE statement uses its matched UPDATE branch to overwrite existing attribute values; its unmatched INSERT branch can add new business keys.

### Understand the Requirements

Identify the type of changes and historical tracking needed for your dimension table.

#### What is SCD Type 1 used for?

SCD Type 1 is used for overwriting data in the dimension table, without preserving historical data.

#### How does SCD Type 2 track historical data?

SCD Type 2 tracks historical data by creating a new row for each change, with columns like Start Date, End Date, and Current Flag.

#### What is the primary method for implementing SCD Type 3?

SCD Type 3 is implemented by adding new columns to store previous and current values of dimension attributes.

## Advanced Techniques for SCD in Snowflake and Databricks

### Optimizing Slowly Changing Dimensions in Snowflake

Snowflake provides micro-partitions and clustering keys that can improve pruning for suitable SCD query patterns. Evaluate the improvement against reclustering compute and retained-storage costs; neither faster queries nor storage savings are automatic. This section will explore how to design SCD schemas and implement them efficiently in Snowflake.

- Use clustering keys to optimize query access patterns for SCD Type 2 tables.
- Leverage Snowflake streams and tasks for incremental updates.
- Choose clustering keys based on frequently queried attributes; Snowflake manages native-table micro-partitions automatically.

> SCD Type 2 Optimization in Snowflake

| Concern | Action |
| --- | --- |
| Excessive scanned partitions | Evaluate clustering for frequent selective predicates |
| Reprocessing unchanged data | Consume bounded change batches with streams and tasks |
| Multiple current rows | Serialize writers and test current-row uniqueness |
| Incorrect historical joins | Join on business key and half-open validity interval |
| High maintenance cost | Compare pruning benefits with clustering and storage charges |

### Advanced SCD Techniques in Databricks

Databricks, integrated with Delta Lake, offers advanced features for handling Slowly Changing Dimensions. Features like ACID transactions, versioning, and merge operations make it an ideal platform for implementing complex SCD logic. This section covers the use of Delta Lake for implementing SCD and ensuring data integrity.

- **Step 1: Define Delta Tables**

  Create Delta tables with appropriate schema design, including versioning columns like Start Date, End Date, and Current Flag.

- **Step 2: Implement Merge Logic**

  In Delta, use a staged single-table MERGE that creates one matched row to expire the old version and a separate unmatched staged row to insert its replacement. Deduplicate input and serialize conflicting updates. Two independent Delta statements are separate transactions unless an explicitly supported multi-statement transaction feature is used.

- **Step 3: Validate Data Integrity**

  Test one current row per business key, nonoverlapping validity intervals, deterministic event ordering, and joins from facts to the version effective at the fact time.

- **Step 4: Optimize Table Performance**

  Inspect query plans and file sizes before choosing supported clustering or compaction. Avoid partitioning by unique customer IDs; plan transaction-log and historical-data retention.

#### What Snowflake feature can be used to optimize SCD Type 2 tables?

Clustering keys can be used to optimize query performance and access patterns for SCD Type 2 tables.

#### What command in Delta Lake is used for handling updates and inserts in SCD?

The MERGE INTO command is used for updates, inserts, and deletions in Delta Lake.

#### How does Delta Lake ensure data integrity during SCD implementation?

Delta Lake uses ACID transactions and versioning to ensure data integrity and consistency.

#### Which Snowflake feature can improve pruning for a large SCD Type 2 table with suitable filter patterns?

- Streams
- Tasks
- Clustering Keys
- Partitions

**Answer:** **Clustering Keys.** They can improve pruning for matching access patterns, with additional maintenance cost; streams and tasks address change processing instead.

### Micro-partitions in Snowflake

Snowflake automatically manages micro-partitions; users do not create directory partitions for a native analytical table. Metadata such as column ranges helps prune irrelevant data. For a sufficiently large Type 2 table, evaluate clustering against actual business-key and date predicates. Clustering consumes maintenance credits and does not guarantee a storage-cost reduction.

### Delta Lake Versioning

Each committed change produces a new table version in the Delta transaction log. Time travel reads a retained version of the whole table; Type 2 rows preserve business-attribute history within the table. These are different mechanisms. Log and data-file retention, including `VACUUM`, limit time travel; Type 2 history remains only if its rows are retained by the model.

## Automating SCD Processes Using dbt

### Introduction to Automating SCD Processes with dbt

In this lesson, we'll explore how dbt (data build tool) can be used to automate Slowly Changing Dimension (SCD) processes. By leveraging dbt's modular architecture, you can create maintainable, scalable, and testable data transformation pipelines that handle SCD complexities efficiently.

- Understand the role of dbt in modern data workflows.
- Leverage dbt's incremental models for SCD processes.
- Define schema-change handling explicitly, reuse transformation logic with dbt macros, and validate results with tests.

### Setting Up dbt for SCD Automation

Choose the mechanism according to the required behavior: use a merge-style incremental model for latest-state Type 1 updates, a dbt snapshot for observed Type 2 history, or custom SQL for Type 3. An incremental materialization does not itself implement Type 2 expiration and insertion.

For dbt Core 1.9 or later with snapshot YAML support, first define a model named `stg_customers` with one row per `customer_id` and a trustworthy `updated_at`. Add this snapshot definition under `snapshots/` and run `dbt snapshot --select customer_history`:

```yaml
snapshots:
  - name: customer_history
    relation: ref('stg_customers')
    config:
      schema: snapshots
      unique_key: customer_id
      strategy: timestamp
      updated_at: updated_at
```

dbt adds validity metadata such as `dbt_valid_from` and `dbt_valid_to`. The timestamp strategy assumes `updated_at` changes for every relevant source update. Configure hard-delete handling separately when source rows can disappear. A snapshot sees states present when it runs, so it cannot recover changes that occurred and disappeared between runs.

Before diving into implementation, ensure that you have a dbt project set up and connected to your data warehouse. This includes configuring your dbt profiles.yml file with the correct credentials and creating a project structure that supports modular model development.

- **Step 1: Define SCD Requirements**

  Identify the SCD type to implement (Type 1, Type 2, or Type 3). Understand how changes in source data will impact your dimension tables and analytics.

- **Step 2: Configure dbt Incremental Models**

  Use incremental merge for Type 1 or a custom incremental history model; select a snapshot when observed Type 2 history is the goal. Configure a real unique key and an explicit schema-change policy.

- **Step 3: Write dbt SQL Models**

  Stage and deduplicate the source, detect relevant attribute changes, and implement the selected overwrite or historical-version behavior. Use ref() to express dependencies.

- **Step 4: Implement dbt Macros for Reusability**

  Extract shared comparison and normalization expressions into macros; keep business-specific validity and delete policies visible in the model configuration.

- **Step 5: Test and Validate**

  Run uniqueness, not-null, relationship, and custom interval tests. Verify that a repeated snapshot or model run on unchanged input creates no extra history.

#### What is the purpose of dbt incremental models?

To process only new or updated data instead of reprocessing the entire dataset, improving efficiency.

#### Which dbt feature allows for dynamic SQL generation?

dbt's Jinja templating system.

#### How does dbt help ensure data quality in SCD implementation?

By using built-in tests and custom tests to validate data integrity and identify issues like duplicate keys or missing values.

### Verify dbt Project Setup

Ensure your dbt project is connected to the correct data warehouse and has a valid profiles.yml configuration.

#### Which dbt feature reduces reprocessing by transforming only new or updated rows?

- Snapshots
- Incremental Models
- Materialized Views
- Macros

**Answer:** **Incremental Models**, for reducing the amount of data processed. **Snapshots** are the built-in mechanism specifically for observed Type 2 history. These features solve different parts of the problem.

### Benefits of Automating SCD with dbt

Version-controlled SQL, reusable macros, dependency tracking, tests, and scheduled builds make dimension changes repeatable and reviewable. Incremental models can reduce processing volume, while snapshots provide built-in Type 2 history for mutable source tables. Automation still needs explicit rules for keys, deletions, late changes, and schema evolution.

### Common Challenges in SCD Automation

Common failures include duplicate business keys, an `updated_at` field that does not change on every update, missed source deletions, out-of-order events, and concurrent writers creating overlapping versions. Snapshot schedules can miss intermediate states between runs. Test current-row uniqueness and interval overlap, monitor run failures, and choose a recovery strategy that can safely replay data.

### References

- [dbt snapshots and Type 2 history](https://docs.getdbt.com/docs/build/snapshots)
- [dbt incremental models](https://docs.getdbt.com/docs/build/incremental-models)
- [Delta Lake merge patterns](https://docs.delta.io/delta-update/)
- [Snowflake clustering keys](https://docs.snowflake.com/en/user-guide/tables-clustering-keys)
