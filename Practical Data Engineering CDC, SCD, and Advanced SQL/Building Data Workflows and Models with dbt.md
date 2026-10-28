# Building Data Workflows and Models with dbt

## Introduction to dbt: Key Features and Benefits

### What is dbt?

dbt, short for 'data build tool,' is an open-source framework designed to help data teams transform, test, and document their data workflows. It allows users to define data transformations as SQL-based models, enabling version control, modularity, and collaboration. dbt is widely adopted in modern data engineering workflows for its ability to streamline data transformation pipelines.

- SQL-focused approach for data transformations.
- Supports incremental loading to optimize performance.
- Automated testing to ensure data quality.
- Lineage tracking for full visibility into dependencies.
- Seamless integration with modern cloud data platforms.

#### What is the primary programming language used in dbt?

SQL is the core language used in dbt for defining data transformations.

#### What feature in dbt helps track dependencies between models?

Lineage tracking provides visibility into model dependencies and their relationships.

#### How does dbt ensure data quality during transformations?

dbt supports automated testing, allowing users to define and execute tests for their data models.

### Incremental Loading in dbt

An incremental materialization creates the target on its first run, then processes a selected subset on later runs. The SQL author defines that subset. A supported merge strategy and a correct `unique_key` can update existing rows instead of always appending; neither a unique key configuration nor an incremental materialization automatically validates the source or implements SCD history.

### Automated Testing

Built-in data tests include not-null, uniqueness, accepted-values, and relationships checks. A singular SQL test returns violating rows and fails when violations exist. Run tests with `dbt test`, or use `dbt build` to build selected resources and run their tests. Add explicit tests for business rules; built-in tests do not prove overall correctness.

### Lineage Tracking

Use `ref()` to connect models and `source()` to connect declared raw tables. dbt derives dependency edges from these references and displays them in generated documentation. Hard-coded table names can hide dependencies, and dbt lineage does not automatically capture transformations that occur outside the declared project.

- **Step 1: Define Models**

  Write SQL-based models that define how raw data should be transformed. Models can be organized into folders for modularity.

- **Step 2: Apply Tests**

  Declare tests in model properties YAML and add SQL tests for business invariants. Identify the build failures that should stop downstream publication.

- **Step 3: Run dbt**

  Run dbt debug to verify connection, then dbt build for selected resources and their tests, or run and test as separate commands.

- **Step 4: Review Lineage**

  Generate documentation and inspect upstream/downstream references. Replace hard-coded project relations with ref() or declared source() references where appropriate.

#### Which of the following is NOT a feature of dbt?

- Automated Testing
- Lineage Tracking
- Real-time Streaming
- Incremental Loading

**Answer:** **Real-time Streaming**, as a built-in continuously running stream processor in the Core workflow described here. dbt builds transformations on its supported data platform; ingestion and continuous stream consumption require other mechanisms.

### Install dbt

For these examples, use a configured **dbt Core 1.9 or later** project with the Snowflake adapter and a writable training database/schema. Install a compatible adapter and Core version together, initialize the project, configure the connection outside version control, and run `dbt debug`. Adapter compatibility and authentication depend on the environment; no credentials belong in these notes.

Ensure dbt is installed via pip or another package manager compatible with your environment.

## Creating dbt Models with Tests and Documentation

### What Are dbt Models?

In dbt (data build tool), models are SQL files that define transformations on your raw data. These models are at the core of dbt and are used to convert raw data into meaningful tables, views, or datasets. Models can be simple queries or complex transformations depending on your data requirements.

- Models are stored in the `models/` directory of your dbt project.
- Each model corresponds to a single SQL file.
- Models can be materialized as tables, views, or incremental tables.
- They can reference other models to build a dependency graph.

### Adding Tests to Models

The following small project example uses a seed instead of an unspecified external source. Add this file as `seeds/customers_raw.csv`:

```csv
customer_id,customer_name,region,is_active,updated_at
1,Asha,North,true,2026-01-01 00:00:00
2,Ravi,West,true,2026-01-02 00:00:00
3,Meera,South,false,2026-01-03 00:00:00
```

Create `models/stg_customers.sql` using Snowflake casts:

```sql
select
    cast(customer_id as number) as customer_id,
    customer_name,
    region,
    cast(is_active as boolean) as is_active,
    cast(updated_at as timestamp_ntz) as updated_at
from {{ ref('customers_raw') }}
```

Create `models/active_customers.sql`:

```sql
select customer_id, customer_name, region, updated_at
from {{ ref('stg_customers') }}
where is_active = true
```

Create `models/customers.yml`:

```yaml
version: 2
models:
  - name: stg_customers
    description: One current source row per customer, including inactive customers.
    columns:
      - name: customer_id
        description: Stable customer business identifier.
        data_tests:
          - not_null
          - unique
      - name: updated_at
        description: Source change timestamp in UTC, stored without a timezone suffix.
        data_tests:
          - not_null
  - name: active_customers
    description: One row per currently active customer; not a historical dimension.
    columns:
      - name: customer_id
        description: Business key inherited from stg_customers.
        data_tests:
          - not_null
          - unique
```

Run `dbt seed --select customers_raw`, then `dbt run --select stg_customers active_customers`, followed by `dbt test --select stg_customers active_customers`. The active model should contain customers `1` and `2`. A schema properties file describes models and tests; it is different from `profiles.yml`, which configures the connection.

dbt allows you to add tests to your models to ensure data quality and accuracy. Tests can be applied at the model level or at the column level, and they help identify issues in your data pipelines early on.

- **Step 1: Define Your Model**

  Create a new SQL file in the `models/` directory. Write a query to transform your data. For example, if you want to create a table of active customers, write a query that filters out inactive customers.

- **Step 2: Add a Schema File**

  Add a version: 2 properties YAML file under models/ that names the model exactly and documents its columns and tests.

- **Step 3: Apply Tests**

  Apply not_null and unique to the intended key and add relationship or custom business tests where appropriate. A configured unique_key on an incremental model is not a uniqueness test.

- **Step 4: Run Tests**

  Build the example model, run dbt test for it, and inspect failing rows before publishing. Generate documentation only after resolving compilation errors.

### Documenting Your Models

Documentation in dbt is essential for understanding your data pipeline and for team collaboration. dbt automatically generates documentation for your models based on the schema file, which can be viewed in a web-based interface.

### Model-Level Documentation

In a model properties YAML file, provide a description of the dataset, its grain, business purpose, update cadence, and important limitations. Store the documentation with the code and review it whenever behavior changes. The example below documents a customer model alongside its tests.

### Column-Level Documentation

Describe each column's business meaning, data type expectations, null behavior, units, and key role. For historical columns, distinguish business effective time from processing time. Add column tests where the intended rule can be checked automatically.

### Auto-Generated Lineage

dbt documentation builds a graph from parsed model and source references. The graph shows build dependencies and downstream impact; it does not prove that a transformation has the correct business meaning. Generate it after successful parsing and compilation of the project.

#### What is the purpose of a dbt model?

A dbt model is a SQL file that defines transformations on raw data to create meaningful tables, views, or datasets.

#### Where are dbt models stored?

Models are stored in the `models/` directory of the dbt project.

#### How does dbt ensure data quality in models?

dbt uses tests defined in schema YAML files to validate data quality at the model and column levels.

#### Which command is used to execute tests in dbt?

- dbt run
- dbt test
- dbt docs
- dbt compile

**Answer:** **`dbt test`.** `dbt build` also runs tests as part of building selected resources; it is not one of the listed options.

## Managing Data Lineage and Dependencies in dbt

Data lineage and dependency management are critical aspects of modern data engineering. In this lesson, we will explore how dbt enables efficient tracking of data transformations and dependencies, ensuring transparency and reliability in your data pipeline.

### What is Data Lineage?

Data lineage refers to the life cycle of data, including its origins, transformations, and ultimate destination. It provides visibility into how data flows through your system, helping identify potential issues and maintain data quality.

- Understand the source of your data.
- Track transformations applied to the data.
- Identify dependencies between datasets.
- Ensure compliance and auditability.

### How dbt Handles Data Lineage

dbt automatically tracks data lineage by analyzing relationships between models, sources, and tests. This information is visualized in the dbt documentation site, providing a clear picture of your data pipeline.

#### What does data lineage track?

Data lineage tracks the origins, transformations, and destinations of data.

#### How does dbt visualize data lineage?

dbt visualizes data lineage through its auto-generated documentation site.

#### Why is data lineage important?

It ensures transparency, helps debug issues, and maintains data quality.

### Defining Dependencies in dbt

Write `SELECT ... FROM {{ ref("stg_customers") }}` to reference another project model, and declare raw sources in YAML before using `source()`. `ref()` resolves the configured relation and contributes a dependency edge. Avoid cycles, since dbt requires a directed acyclic dependency graph for model execution.

### Visualizing Lineage in dbt

For dbt Core, run `dbt docs generate`, then `dbt docs serve` to view the generated documentation locally. Use the graph to trace a model back to declared inputs and inspect downstream dependents before changing it. Hosted dbt products may expose their own catalog interface.

### Maintaining Documentation

Update descriptions, key assumptions, tests, ownership, and source definitions in the same change as the model SQL. Regenerate documentation after successful builds and review whether renamed or removed columns affect downstream models. Keep credentials in the connection environment, not in documentation or committed profiles.

- **Step 1: Define Your Models**

  Create dbt models using SQL files. Use the 'ref' function to define dependencies between models.

- **Step 2: Add Tests and Documentation**

  Document source and model grain, define required column tests, and record meaningful descriptions in version-controlled YAML.

- **Step 3: Generate Documentation**

  Run dbt docs generate after validating the project so the catalog and manifest reflect the intended environment.

- **Step 4: Review Lineage Graphs**

  Trace changed models to their upstream sources and downstream consumers, checking for hidden hard-coded relations and unexpected dependency cycles.

#### Which dbt command is used to generate the documentation site?

- dbt run
- dbt test
- dbt docs generate
- dbt seed

**Answer:** **`dbt docs generate`.** For dbt Core, `dbt docs serve` can then serve the generated site locally.

#### The 'ref' function in dbt is used to define ______ between models.

**Answer:** **Dependencies.** It also resolves the relation name for the target environment.

## Implementing Incremental Models for Large Datasets

### Understanding Incremental Models

Incremental models are designed to process only the new or changed data in a dataset, rather than reprocessing the entire dataset. This approach significantly improves performance and reduces computational costs, especially for large datasets. In this lesson, we will explore how to implement incremental models using modern tools and SQL patterns.

- Minimizing data processing time for large datasets.
- Reducing resource usage by avoiding full reprocessing.
- Ensuring consistent data updates while maintaining accuracy.

### Core Components of Incremental Models

Using the seed and staging model defined earlier, create `models/customer_current.sql`. This example uses **dbt-snowflake** with a merge strategy and retains the latest state, including an `is_active` flag; it is not Type 2 history.

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='customer_id',
    on_schema_change='fail'
) }}

select customer_id, customer_name, region, is_active, updated_at
from {{ ref('stg_customers') }}
{% if is_incremental() %}
where updated_at >= dateadd(
    day, -7,
    coalesce(
        (select max(updated_at) from {{ this }}),
        '1900-01-01 00:00:00'::timestamp_ntz
    )
)
{% endif %}
```

Run `dbt run --select customer_current` after the staging model exists. Then edit customer `1` in the seed to region `East` and a later `updated_at`, reload the seed, rebuild `stg_customers`, and rerun `customer_current`. Customer `1` should be updated without an extra target row. Repeating that run should leave the same state.

The seven-day lookback is an explicit example policy, not a universal safe interval. It assumes relevant late changes fall within that source-time window and that the stage is unique per customer. A change older than the window needs a backfill or CDC-based strategy. Hard deletes are not propagated by this model; use a soft-delete flag or explicit delete reconciliation. The source timestamps in this example are UTC. Use `dbt run --full-refresh --select customer_current` only when intentionally rebuilding this latest-state target.

To build effective incremental models, you need to understand the core components: identifying new or changed data, maintaining state between runs, and ensuring data integrity. Each of these components plays a vital role in ensuring efficiency and accuracy.

- **Step 1: Define the Incremental Logic**

  Determine the criteria for identifying new or updated data, such as a timestamp column or a unique identifier.

- **Step 2: Configure State Management**

  Choose an authoritative source change marker. For the example, derive a watermark from target updated_at with a seven-day overlap; for CDC, persist the source position only after successful processing.

- **Step 3: Implement the Model**

  Use the merge example with one staged row per customer, a correct unique_key, and an explicit schema-change policy. Include first-run and full-refresh behavior.

- **Step 4: Test and Validate**

  Test initial build, a new key, an updated key, unchanged replay, equal timestamps, a late update outside the lookback, and a delete. Document which cases require backfill or extra logic.

#### What is the primary purpose of incremental models?

To process only new or updated data, improving performance and reducing costs.

#### Which component tracks the last processed record in an incremental model?

State management tracks the last processed record or timestamp.

#### What tool can be used to implement incremental models with SQL?

dbt is a popular tool for implementing incremental models using SQL.

#### Which of the following is NOT a benefit of incremental models?

- Minimizing data processing time.
- Reducing computational costs.
- Automatically handling schema changes.
- Ensuring data consistency.

**Answer:** **Automatically handling schema changes.** Schema behavior must be configured, and compatible downstream changes and historical backfills may still require work. Incremental processing also needs correct logic to maintain consistency.

### Why use a timestamp column for incremental logic?

A trustworthy change timestamp allows a model to select records updated since a previous run. It must change for every relevant mutation and must have a defined timezone. Use inclusive boundaries and deduplication for equal timestamps. A bounded lookback can handle limited lateness, but a source commit sequence or CDC offset is safer when arbitrarily late changes must be captured.

### How does dbt handle incremental models?

On the first run, dbt creates the complete target from the model query. On later qualifying runs, `is_incremental()` becomes true and the author's filter selects a subset; the configured adapter strategy applies it. A merge strategy can use `unique_key` to update matches and insert new rows. Deletes and Type 2 versioning require explicit logic or another mechanism.

### What happens if state tracking fails?

An advanced checkpoint can skip source changes; a lost checkpoint can cause unnecessary reprocessing or duplicates in an append-only design. Recover from a known successful state, replay with idempotent keys, and reconcile against the source. Use a controlled full refresh or bounded backfill only after assessing downstream references and history requirements.

### Define Incremental Logic

Ensure you have a clear criterion for identifying new or updated records.

#### The function _______ in dbt is used to identify incremental runs.

**Answer:** **`is_incremental()`**, a Jinja macro. It is true when the model is incremental, the target already exists as a table, and the run is not a full refresh.

### References

- [dbt incremental models and is_incremental()](https://docs.getdbt.com/docs/build/incremental-models)
- [dbt data tests](https://docs.getdbt.com/docs/build/data-tests)
- [dbt ref() dependencies](https://docs.getdbt.com/reference/dbt-jinja-functions/ref)
- [dbt documentation](https://docs.getdbt.com/docs/build/documentation)
- [dbt snapshots](https://docs.getdbt.com/docs/build/snapshots)
