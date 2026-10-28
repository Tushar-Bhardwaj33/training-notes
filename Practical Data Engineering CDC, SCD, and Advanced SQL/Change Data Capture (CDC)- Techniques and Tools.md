# Change Data Capture (CDC): Techniques and Tools

## Fundamentals of Change Data Capture and Its Applications

### What is Change Data Capture (CDC)?

Change Data Capture (CDC) is a technique used to identify and capture changes made to data in a database. It allows the real-time or near-real-time tracking of inserts, updates, and deletes, enabling downstream systems to stay synchronized with the source data. CDC is essential in modern data architectures for maintaining data consistency across distributed systems.

- Real-time synchronization of transactional and analytical systems
- Efficient data replication across environments
- Facilitation of event-driven architectures
- Support for audit and compliance by tracking historical changes

### How CDC Works

CDC works by monitoring changes in the source database and delivering those changes to a target system. This process can be implemented using database logs, triggers, or query-based approaches. Modern CDC solutions often rely on transaction logs, which provide a low-latency and resource-efficient way to capture changes.

### Log-Based CDC

A connector reads the source database transaction log, such as PostgreSQL WAL or a MySQL binlog, and emits committed row changes. It usually has less per-change application overhead than triggers, but needs database-specific permissions, an initial snapshot, a durable log position, and sufficient source-log retention. Debezium is a CDC connector; Kafka transports and stores events rather than reading database logs by itself.

### Trigger-Based CDC

An insert, update, or delete trigger writes a change record into an audit or outbox table inside the source transaction. This can capture explicit before/after values, but each source write pays the trigger overhead. A separate consumer must deliver those records, track progress, and clean up only data that is safe to remove.

### Query-Based CDC

A polling job selects rows using a reliable change marker such as `updated_at`, or compares snapshots. Persist a successful checkpoint and use overlap plus deduplication so equal timestamps and delayed commits do not silently lose rows. Timestamp polling alone cannot observe hard-deleted rows; use tombstones, a delete log, or snapshot comparison when deletes matter.

#### Which CDC method is most efficient for low-latency and high-throughput systems?

- Log-Based CDC
- Trigger-Based CDC
- Query-Based CDC
- Manual Data Export and Import

**Answer:** **Log-Based CDC**, for supported transactional sources. It reads committed log changes without adding a trigger write for every changed row.

### Applications of CDC in Data Engineering

CDC plays a critical role in data engineering by enabling real-time data pipelines, supporting incremental data processing, and ensuring data consistency in distributed architectures. Some common use cases include ETL workflows, real-time analytics, and syncing data across microservices.

### Set up real-time data pipelines

Use CDC to push changes directly to data lakes or warehouses for real-time analytics.

#### What is the primary advantage of Log-Based CDC?

It provides a low-latency and resource-efficient way to capture changes.

#### Which CDC method uses queries to detect changes?

Query-Based CDC.

#### Name a common use case for CDC in data engineering.

Real-time data pipelines for analytics.

- **Step 1: Identify Source Tables**

  Determine which tables and columns need CDC for your use case (e.g., transactional tables).

- **Step 2: Choose a CDC Implementation Method**

  Compare log access, source write overhead, delete capture, latency, and operating complexity; choose log, trigger, or polling capture accordingly.

- **Step 3: Configure CDC Tools**

  Configure source permissions, the initial snapshot, the log or query checkpoint, source retention, and a durable event destination. Test capture before switching on downstream processing.

- **Step 4: Apply Changes to Target Systems**

  Apply events by stable key and source ordering, handle deletes, and commit a consumer checkpoint only after a successful target write. Make replays idempotent.

## SQL-Based CDC: Triggers, Log-Based Methods, and Queries

### Introduction to SQL-Based CDC Methods

SQL-based Change Data Capture (CDC) methods enable efficient tracking and processing of data changes in relational databases. This lesson focuses on three main SQL-based approaches: Triggers, Log-Based Methods, and Query-Based Techniques. Each method has distinct use cases, benefits, and trade-offs, which we will explore in this lesson.

- Trigger-Based CDC: Using database triggers to capture changes.
- Log-Based CDC: Leveraging transaction logs to track changes.
- Query-Based CDC: Identifying changes through SQL queries.

### How Trigger-Based CDC Works

This **PostgreSQL** example is a self-contained local capture pattern. Run it in a fresh training schema. The audit write commits or rolls back with the source change. `event_id` identifies records; sequence allocation alone does not guarantee transaction commit order.

```sql
CREATE TABLE cdc_customers (customer_id BIGINT PRIMARY KEY, region TEXT);
CREATE TABLE cdc_customer_events (
  event_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  operation TEXT NOT NULL,
  customer_id BIGINT NOT NULL,
  before_row JSONB,
  after_row JSONB,
  captured_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

CREATE FUNCTION capture_customer_change() RETURNS TRIGGER
LANGUAGE plpgsql AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO cdc_customer_events(operation, customer_id, after_row)
    VALUES (TG_OP, NEW.customer_id, to_jsonb(NEW));
    RETURN NEW;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO cdc_customer_events
      (operation, customer_id, before_row, after_row)
    VALUES (TG_OP, NEW.customer_id, to_jsonb(OLD), to_jsonb(NEW));
    RETURN NEW;
  ELSE
    INSERT INTO cdc_customer_events(operation, customer_id, before_row)
    VALUES (TG_OP, OLD.customer_id, to_jsonb(OLD));
    RETURN OLD;
  END IF;
END;
$$;

CREATE TRIGGER customer_change
AFTER INSERT OR UPDATE OR DELETE ON cdc_customers
FOR EACH ROW EXECUTE FUNCTION capture_customer_change();

INSERT INTO cdc_customers VALUES (1, 'North');
UPDATE cdc_customers SET region = 'South' WHERE customer_id = 1;
DELETE FROM cdc_customers WHERE customer_id = 1;
SELECT operation, customer_id, before_row, after_row
FROM cdc_customer_events ORDER BY event_id;
```

Expected operations are INSERT, UPDATE, and DELETE. A production consumer needs a delivery/acknowledgment protocol; simply reading `MAX(event_id)` can skip a lower-numbered event committed later by a concurrent transaction.

Trigger-based CDC relies on database triggers, which are procedural code executed automatically in response to specific events on a table or view. Triggers can capture INSERT, UPDATE, and DELETE operations, storing the changes in a separate audit or history table for further processing. While effective, this approach can impact database performance due to the additional overhead of managing triggers.

### Advantages of Trigger-Based CDC

Triggers can capture changes without a separate transaction-log reader and can write a business-specific audit payload in the same transaction as the change. They are useful when log access is unavailable and the database supports the required trigger behavior.

### Limitations of Trigger-Based CDC

Triggers add work and possible contention to source transactions. Trigger code must handle all relevant operations, schema changes, failures, and bulk-load behavior. A trigger captures records locally; it does not by itself guarantee downstream delivery, replay safety, or event ordering.

### Log-Based CDC: The Efficient Alternative

Log-based CDC methods utilize the transaction logs maintained by databases to identify and capture data changes. These methods are highly efficient as they bypass the application layer, reducing the performance impact on the database. Connectors such as Debezium read supported database logs; Apache Kafka commonly carries the emitted events to downstream consumers.

#### What is the primary advantage of log-based CDC?

Log-based CDC is highly efficient as it directly reads from transaction logs, minimizing performance impact.

#### What tools can be used for log-based CDC?

Debezium and AWS Database Migration Service (DMS) provide log-based CDC capabilities for supported sources. Apache Kafka is commonly the event transport, not the database-log reader.

#### What is a transaction log in databases?

A transaction log is a record of all changes made to a database, including INSERT, UPDATE, and DELETE operations.

### Query-Based CDC for Simple Implementations

Query-based CDC identifies changes by comparing the current state of data with a previously captured state. This method is often used when other CDC options are not available or when working with smaller datasets. While straightforward, it can be resource-intensive for large datasets or high-frequency changes.

For timestamp polling, the extraction query must have a bounded upper position and a replay-safe overlap. This Snowflake SQL pattern assumes an existing `poll_customers(customer_id, region, updated_at, is_deleted)` table, UTC timestamps, and a source contract that relevant commit delay is at most five minutes:

```sql
SET last_successful_upper = '2026-02-01 00:00:00'::TIMESTAMP_NTZ;
SET batch_upper = '2026-02-01 01:00:00'::TIMESTAMP_NTZ;

SELECT customer_id, region, updated_at, is_deleted
FROM poll_customers
WHERE updated_at >= DATEADD(minute, -5, $last_successful_upper)
  AND updated_at < $batch_upper
ORDER BY updated_at, customer_id;
```

Apply these rows with a deduplicating key-based write, then persist `batch_upper` only after the target write succeeds. Re-read the overlap on the next run. A timestamp can be assigned before a transaction commits, so the five-minute bound must be guaranteed by the source or replaced with a commit-ordered CDC position; no fixed overlap protects against arbitrarily late commits. The `is_deleted` field is a source-maintained tombstone. Without it or another deletion record, polling cannot see a row that has already been hard-deleted.

#### Which of the following is a disadvantage of query-based CDC?

- It requires additional tools like Debezium.
- It is not suitable for databases without triggers.
- It can be resource-intensive for large datasets.
- It cannot capture DELETE operations.

**Answer:** **It can be resource-intensive for large datasets.** Hard-delete capture is also a limitation of timestamp-only polling, but full snapshot comparison can detect missing rows, so the blanket claim that query-based CDC can never capture deletes is incorrect.

- **Step 1: Identify the Source Tables**

  Determine which tables contain the data changes you need to track. This is the first step for any CDC implementation.

- **Step 2: Choose the CDC Method**

  Select a trigger if an audit/outbox table is acceptable, a supported log connector for higher-throughput capture, or polling when only read access is available.

- **Step 3: Implement the Chosen Method**

  Install the selected trigger, connector, or polling query; specify initial loading, checkpoint semantics, delete handling, and the consumer retry contract.

- **Step 4: Test and Validate**

  Exercise insert, update, delete, duplicate replay, delayed commit, and schema-change cases. Compare the final target state with a consistent source snapshot.

### Understand the advantages and disadvantages of each CDC method.

Ensure you can evaluate the trade-offs between trigger-based, log-based, and query-based CDC.

## Implementing CDC with Snowflake Streams and Tasks

### Introduction to Snowflake Streams and Tasks

The following **Snowflake SQL** example requires a current database and schema, a role allowed to create these objects and run tasks, and an existing virtual warehouse named `CDC_WH`. It uses an initially empty source and a standard stream, then maintains a latest-state target. The source contract is one current row per immutable `customer_id`; Snowflake standard-table key declarations alone do not enforce that contract. If a business identifier must change, model it as a delete and insert with explicit reconciliation rather than updating the key through this example.

```sql
CREATE TABLE cdc_demo_source (customer_id NUMBER, region VARCHAR);
CREATE TABLE cdc_demo_target (customer_id NUMBER, region VARCHAR);
CREATE STREAM cdc_demo_stream ON TABLE cdc_demo_source;

CREATE TASK cdc_demo_apply
  WAREHOUSE = CDC_WH
  SCHEDULE = '1 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('cdc_demo_stream')
AS
MERGE INTO cdc_demo_target AS t
USING (
  SELECT customer_id, region, METADATA$ACTION AS action
  FROM cdc_demo_stream
  WHERE NOT (METADATA$ACTION = 'DELETE' AND METADATA$ISUPDATE)
) AS s ON t.customer_id = s.customer_id
WHEN MATCHED AND s.action = 'DELETE' THEN DELETE
WHEN MATCHED AND s.action = 'INSERT' THEN UPDATE SET region = s.region
WHEN NOT MATCHED AND s.action = 'INSERT' THEN
  INSERT (customer_id, region) VALUES (s.customer_id, s.region);

ALTER TASK cdc_demo_apply RESUME;
INSERT INTO cdc_demo_source VALUES (1, 'North');
```

After the task succeeds, the target contains `(1, 'North')`. Update that source row, wait for a successful task run, and verify the target region changes; delete it, wait again, and verify the target row disappears. Suspend the task after the exercise with `ALTER TASK cdc_demo_apply SUSPEND;`. The task's MERGE consumes the stream atomically. Independent deletes and reinserts of the same key in one unconsumed interval can produce ambiguous source matches; prevent that pattern or first reconcile those keys to the source's current state. This is a current-state example, not complete business-event history.

Snowflake Streams and Tasks are essential tools for implementing Change Data Capture (CDC) in modern data pipelines. Streams allow you to track changes in a table at a granular level, while Tasks enable the automation of processes, such as applying these changes to a target table. Together, they provide an efficient and scalable way to handle incremental data changes in a Snowflake environment.

- Streams: A Snowflake object that tracks changes (inserts, updates, deletes) made to a table.
- Tasks: A scheduled or event-driven execution framework to automate SQL statements.
- Together, they enable the implementation of CDC pipelines by capturing and applying data changes.

- **Step 1: Create a Source Table**

  Start by creating or identifying the table where data changes will occur. This is the source table for your CDC implementation.

- **Step 2: Set Up a Stream**

  Create a standard stream before new source changes, or separately coordinate the initial target load and stream offset. Choose append-only or insert-only only when its semantics fit.

- **Step 3: Create a Target Table**

  Create the current-state target with compatible column types and an explicitly tested unique business key.

- **Step 4: Define a Task**

  Use the MERGE task example with the named warehouse and required task privileges, then resume it. Ensure the source produces no ambiguous multiple matches per target key.

- **Step 5: Validate the CDC Pipeline**

  Apply separate insert, update, and delete test transactions, wait for successful task runs, and inspect task history, target rows, and stream staleness. Replay safely after a failure.

#### What is the purpose of a Snowflake Stream?

A Snowflake Stream tracks data changes (inserts, updates, deletes) in a table for CDC purposes.

#### What is a Snowflake Task used for?

A Snowflake Task automates the execution of SQL statements, such as applying changes from a stream to a target table.

#### What are the two core components for implementing CDC in Snowflake?

Streams and Tasks.

#### Which of the following best describes a Snowflake Stream?

- A mechanism to schedule SQL queries.
- A feature that tracks changes in a table over time.
- A way to load data into Snowflake from external sources.
- A tool for creating dashboards in Snowflake.

**Answer:** **A feature that tracks changes in a table over time.** More precisely, a standard stream exposes net changes since its offset and is not an unlimited historical event log.

### Benefits of Using Snowflake Streams and Tasks for CDC

By leveraging Streams and Tasks, data engineers can implement a highly efficient CDC process without the need for external tools. This approach minimizes latency, simplifies pipeline management, and ensures data consistency across systems.

### What types of changes can a Snowflake Stream track?

A standard stream on a supported table exposes net inserts, deletes, and updates between its offset and the current table version. Updates are represented by DELETE and INSERT rows with `METADATA$ISUPDATE = TRUE`. It is not a permanent event log of every intermediate state. Append-only streams track inserted rows; insert-only streams for external tables have more limited semantics.

### How does a Task execute in Snowflake?

A task executes SQL or a stored procedure using either a specified virtual warehouse or supported serverless compute. It can run on a schedule, as part of a task graph, or as a triggered task when its condition is satisfied. Newly created tasks are suspended until resumed. Reading a stream with SELECT does not advance its offset; a successfully committed consuming DML transaction does.

### Can Streams be used with external tables?

Yes, Snowflake supports insert-only streams on supported external tables, with `INSERT_ONLY = TRUE`. They track new rows associated with external-file changes after metadata refresh and do not provide the full update/delete behavior of a standard native-table stream. Overwritten files can produce replacement rows without matching delete events. Check external-table refresh, event, and retention limitations before using this as a CDC source.

## Using Databricks and Delta Lake for Real-Time CDC Pipelines

### Introduction to Databricks and Delta Lake

Databricks is a unified analytics platform that simplifies big data processing and machine learning. Delta Lake is an open-source storage layer that brings ACID transactions and scalable metadata management to data lakes. Together, they enable reliable, real-time data pipelines for change data capture (CDC).

- Understand the architecture of Databricks and its integration with Delta Lake.
- Learn how Delta Lake handles CDC with ACID transactions.
- Explore the key features of Delta Lake, such as schema enforcement and versioning.

### Setting Up Your Environment

To start building CDC pipelines, you need access to a Databricks workspace and a storage layer compatible with Delta Lake. This section covers the prerequisites and steps to configure your environment for seamless integration.

### Create a Databricks Workspace

Sign up for Databricks and create a workspace in your preferred cloud provider (AWS, Azure, or GCP).

- **Step 1: Ingest Data into Delta Lake**

  Load initial data into Delta Lake tables using Databricks notebooks or jobs.

- **Step 2: Enable Change Data Capture**

  Enable conventional CDF on the source Delta table before the changes to be captured, or use a supported automatic-CDF configuration. CDF does not extract changes from an external database by itself.

- **Step 3: Automate CDC with Databricks Jobs**

  Schedule a Lakeflow job or a streaming pipeline to consume CDF and apply ordered changes. Keep a durable checkpoint and choose separate current-state versus Type 2 handling.

- **Step 4: Monitor Pipeline Performance**

  Monitor consumer lag, failed batches, duplicate keys, checkpoint health, and available history. Alert before the consumer falls behind source retention.

### Delta Lake Schema Enforcement

A Delta write validates the incoming data against the table schema and supported type rules. Intentional evolution requires the relevant schema-evolution setting or DDL and compatible readers. A missing nullable field can be written as NULL; schema enforcement alone does not prove that every required business attribute is present. Add NOT NULL constraints or explicit quality checks where required.

### Delta Lake Time Travel

Use `VERSION AS OF` or `TIMESTAMP AS OF` to query a retained table snapshot. Time travel is useful for investigation and comparison, but it is not itself a row-change feed. Use Change Data Feed (CDF) for row-level insert, update, and delete records, and retain or archive changes before they age out.

### Delta Lake Upserts with MERGE INTO

For conventional, explicitly enabled Delta CDF, this Databricks SQL example creates the source with CDF active from its first version:

```sql
CREATE TABLE cdf_demo_customers (customer_id BIGINT, region STRING)
USING DELTA
TBLPROPERTIES (delta.enableChangeDataFeed = true);

INSERT INTO cdf_demo_customers VALUES (1, 'North');
UPDATE cdf_demo_customers SET region = 'South' WHERE customer_id = 1;

SELECT customer_id, region, _change_type, _commit_version
FROM table_changes('cdf_demo_customers', 0)
ORDER BY _commit_version, _change_type;
```

The feed includes an insert, an update preimage, and an update postimage. Persist `_commit_version` only after the downstream write succeeds. Capture future feed records into durable storage if you need history beyond CDF retention. Databricks also documents automatic CDF for supported newer tables and runtimes; this example deliberately uses the conventional explicitly enabled mechanism.

MERGE matches incoming rows to target business keys and applies insert, update, or delete clauses. Reduce input to a deterministic event per key for latest-state replication and handle deletes explicitly. For CDF, discard update preimages when applying current state and retain postimages. If intermediate states are required for Type 2 history, process them in order rather than collapsing them.

#### Which feature of Delta Lake lets you query prior retained table snapshots?

- Schema Enforcement
- Time Travel
- ACID Transactions
- Partitioning

**Answer:** **Time Travel.** It reads a retained table version. Use Change Data Feed when the requirement is consuming row-level change records rather than reading a historical snapshot.

#### What is the primary benefit of using Delta Lake for CDC?

Delta Lake provides ACID transactions, ensuring reliable and consistent change data capture.

#### How does Delta Lake handle schema changes during CDC?

Delta Lake validates compatible writes against its schema. Schema evolution is explicit, and missing nullable columns may become NULL; required-field checks need constraints or data-quality rules.

#### What SQL command is used for upserts in Delta Lake?

The MERGE INTO command is used for upserts in Delta Lake.

#### Delta Lake's ______ feature enables querying historical data snapshots.

**Answer:** **Time travel**, while the needed transaction-log entries and data files remain available.

### References

- [PostgreSQL trigger functions](https://www.postgresql.org/docs/current/plpgsql-trigger.html)
- [Snowflake streams and offsets](https://docs.snowflake.com/en/user-guide/streams-intro)
- [Snowflake tasks](https://docs.snowflake.com/en/user-guide/tasks-intro)
- [Databricks Change Data Feed](https://docs.databricks.com/aws/en/delta/delta-change-data-feed)
