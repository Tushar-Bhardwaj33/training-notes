# Data Warehouse & Lake Engineering

A Complete Learning Course

Original course material: Srinath Goud.

Missing explanations, worked examples, step details, and assessment answers have been added to complete these notes. SQL examples identify their database dialect and setup assumptions; platform-specific limitations are linked to official documentation.

## [Foundations of Modern Data Architectures](Foundations%20of%20Modern%20Data%20Architectures.md)

1. [Introduction to Data Warehouses, Data Lakes, and Lakehouses](Foundations%20of%20Modern%20Data%20Architectures.md#introduction-to-data-warehouses-data-lakes-and-lakehouses)
2. [Comparing Use Cases: When to Use Each Architecture](Foundations%20of%20Modern%20Data%20Architectures.md#comparing-use-cases-when-to-use-each-architecture)
3. [Core Principles of Data Governance and Scalability](Foundations%20of%20Modern%20Data%20Architectures.md#core-principles-of-data-governance-and-scalability)
4. [Key Technologies: Snowflake, Databricks, and Delta Lake Overview](Foundations%20of%20Modern%20Data%20Architectures.md#key-technologies-snowflake-databricks-and-delta-lake-overview)

## [Dimensional Modeling for Analytics](Dimensional%20Modeling%20for%20Analytics.md)

1. [Understanding Star and Snowflake Schemas](Dimensional%20Modeling%20for%20Analytics.md#understanding-star-and-snowflake-schemas)
2. [Designing Fact Tables and Dimension Tables](Dimensional%20Modeling%20for%20Analytics.md#designing-fact-tables-and-dimension-tables)
3. [Best Practices for Handling High Cardinality Dimensions](Dimensional%20Modeling%20for%20Analytics.md#best-practices-for-handling-high-cardinality-dimensions)
4. [Optimizing Performance with Surrogate Keys and Indexing](Dimensional%20Modeling%20for%20Analytics.md#optimizing-performance-with-surrogate-keys-and-indexing)

## [Mastering Slowly Changing Dimensions (SCD)](Mastering%20Slowly%20Changing%20Dimensions%20%28SCD%29.md)

1. [Introduction to Slowly Changing Dimensions: Types and Use Cases](Mastering%20Slowly%20Changing%20Dimensions%20%28SCD%29.md#introduction-to-slowly-changing-dimensions-types-and-use-cases)
2. [Implementing SCD Type 1, Type 2, and Type 3 with SQL](Mastering%20Slowly%20Changing%20Dimensions%20%28SCD%29.md#implementing-scd-type-1-type-2-and-type-3-with-sql)
3. [Advanced Techniques for SCD in Snowflake and Databricks](Mastering%20Slowly%20Changing%20Dimensions%20%28SCD%29.md#advanced-techniques-for-scd-in-snowflake-and-databricks)
4. [Automating SCD Processes Using dbt](Mastering%20Slowly%20Changing%20Dimensions%20%28SCD%29.md#automating-scd-processes-using-dbt)

## [Change Data Capture (CDC): Techniques and Tools](Change%20Data%20Capture%20%28CDC%29-%20Techniques%20and%20Tools.md)

1. [Fundamentals of Change Data Capture and Its Applications](Change%20Data%20Capture%20%28CDC%29-%20Techniques%20and%20Tools.md#fundamentals-of-change-data-capture-and-its-applications)
2. [SQL-Based CDC: Triggers, Log-Based Methods, and Queries](Change%20Data%20Capture%20%28CDC%29-%20Techniques%20and%20Tools.md#sql-based-cdc-triggers-log-based-methods-and-queries)
3. [Implementing CDC with Snowflake Streams and Tasks](Change%20Data%20Capture%20%28CDC%29-%20Techniques%20and%20Tools.md#implementing-cdc-with-snowflake-streams-and-tasks)
4. [Using Databricks and Delta Lake for Real-Time CDC Pipelines](Change%20Data%20Capture%20%28CDC%29-%20Techniques%20and%20Tools.md#using-databricks-and-delta-lake-for-real-time-cdc-pipelines)

## [Leveraging Snowflake for Performance and Governance](Leveraging%20Snowflake%20for%20Performance%20and%20Governance.md)

1. [Understanding Snowflake's Micro-Partitioning and Clustering](Leveraging%20Snowflake%20for%20Performance%20and%20Governance.md#understanding-snowflakes-micro-partitioning-and-clustering)
2. [Optimizing Query Performance with Materialized Views](Leveraging%20Snowflake%20for%20Performance%20and%20Governance.md#optimizing-query-performance-with-materialized-views)
3. [Implementing Role-Based Access Control and Data Sharing](Leveraging%20Snowflake%20for%20Performance%20and%20Governance.md#implementing-role-based-access-control-and-data-sharing)
4. [Best Practices for Cost Management and Resource Optimization](Leveraging%20Snowflake%20for%20Performance%20and%20Governance.md#best-practices-for-cost-management-and-resource-optimization)

## [Scalable Data Processing with Databricks and Delta Lake](Scalable%20Data%20Processing%20with%20Databricks%20and%20Delta%20Lake.md)

1. [Overview of Databricks and Delta Lake Features](Scalable%20Data%20Processing%20with%20Databricks%20and%20Delta%20Lake.md#overview-of-databricks-and-delta-lake-features)
2. [Implementing ACID Transactions with Delta Lake](Scalable%20Data%20Processing%20with%20Databricks%20and%20Delta%20Lake.md#implementing-acid-transactions-with-delta-lake)
3. [Building Scalable ETL Pipelines in Databricks](Scalable%20Data%20Processing%20with%20Databricks%20and%20Delta%20Lake.md#building-scalable-etl-pipelines-in-databricks)
4. [Using Delta Live Tables for Real-Time Data Processing](Scalable%20Data%20Processing%20with%20Databricks%20and%20Delta%20Lake.md#using-delta-live-tables-for-real-time-data-processing)

## [Building Data Workflows and Models with dbt](Building%20Data%20Workflows%20and%20Models%20with%20dbt.md)

1. [Introduction to dbt: Key Features and Benefits](Building%20Data%20Workflows%20and%20Models%20with%20dbt.md#introduction-to-dbt-key-features-and-benefits)
2. [Creating dbt Models with Tests and Documentation](Building%20Data%20Workflows%20and%20Models%20with%20dbt.md#creating-dbt-models-with-tests-and-documentation)
3. [Managing Data Lineage and Dependencies in dbt](Building%20Data%20Workflows%20and%20Models%20with%20dbt.md#managing-data-lineage-and-dependencies-in-dbt)
4. [Implementing Incremental Models for Large Datasets](Building%20Data%20Workflows%20and%20Models%20with%20dbt.md#implementing-incremental-models-for-large-datasets)

## [Capstone: End-to-End Data Engineering Project](Capstone-%20End-to-End%20Data%20Engineering%20Project.md)

1. [Project Overview and Requirements Gathering](Capstone-%20End-to-End%20Data%20Engineering%20Project.md#project-overview-and-requirements-gathering)
2. [Designing the Data Architecture and Dimensional Models](Capstone-%20End-to-End%20Data%20Engineering%20Project.md#designing-the-data-architecture-and-dimensional-models)
3. [Implementing CDC and SCD Processes with Modern Tools](Capstone-%20End-to-End%20Data%20Engineering%20Project.md#implementing-cdc-and-scd-processes-with-modern-tools)
4. [Building Advanced SQL Transformations and dbt Workflows](Capstone-%20End-to-End%20Data%20Engineering%20Project.md#building-advanced-sql-transformations-and-dbt-workflows)
5. [Performance Tuning and Governance for Production Readiness](Capstone-%20End-to-End%20Data%20Engineering%20Project.md#performance-tuning-and-governance-for-production-readiness)
