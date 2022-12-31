# Apache Iceberg 101

## What is Iceberg?

Apache Iceberg is an `open table format` designed for PB scale tables that helps us manage, organise and track all of our files (E.g. `ORC` or `Parquet`) that comprise a table with the cloud in mind (i.e. Object storage like `S3`).

### Table vs File Formats

File formats help us modify and skip data within a single file. Table formats are the same (i.e. `HIVE`) just applied to the table level (multiple files).

## Why Iceberg?

Data engineers at Netflix did not want spend time to:

1. Rewrite SQL queries
2. Reorder joins by hand to improve performance
3. Tune parquet files based on block size / dictionary page size suitable for their respective datasets

in order to optimize their pipeline performance given PB scale data and wanted to focus on delivering value to their clients. So performance tuning etc. should be more `automatic` without heavy intervention by DEs. However, there are a few stumbling blocks:

1. Unsafe operations everywhere
   1. Writing to multiple partitions
   2. Renaming a column (E.g. We cannot rename a Parquet column, if we do we need to backfill)
2. Interaction with object stores (like `S3`) cause issues like
   1. Consistency problems
   2. Performance problems
3. Endless scale challenges

These challenges and bottlenecks can be addressed by a table format (i.e. Iceberg)

### Current State of Table Formats - Hive & Hive Metastore

In HIVE tables, a given table's files are organised such:

```txt
date=20221225/
|-── hour=00/
│   |-  file01.parquet
│   |-  file02.parquet
|   |-  ...
|
|─── hour=01/
│   |-  file01.parquet
│   |-  file02.parquet
|   |-  ...
```

Since `Hive Metastore` (database) tracks data at the `directory` (`partition`) level, it thus needs to perform file list operations when working with data in that table.

```txt
date=20221225/hour=00 -> hdfs:/.../date=20221225/hour00
```

This leads to a couple for problems:

- State is being kept in both the **metastore** and in a **file system**
- Changes are not atomic without locking since we need to make a thousand files go live at the exact same moment
- Consequently, this requires too many directory listing calls
  - O(n) listing calls, n = Number of matching partitions thus `Performance problems`
  - `Eventual Consistency` breaks correctness

Data may also appear missing during such operations on eventually consistency object stores like `S3`.

### Iceberg's Goals

- Serializable Isolation - Reads will be isolated from concurrent writes and will always use a committed snapshot of a table's data + Writes will support removing and adding files in a single operation and are never partially visible. Readers will not acquire locks
- Speed - Ops will use O(1) remote calls to plan the files for scanning and not O(N)
- Scale - Job planning will be handled primarily by clients and not bottleneck on a central metadata store + Metadata will include information needed for cost-based optimisation
- Evolution - Tables support full schema and partition spec evolution
- Dependable Types
- Storage Separation - Partitioning will be a table configuration with Reads planned using predicates on data values and not partition values
- Formats - Underlying data file formats will support identical schema evolution rules and types

## Iceberg's Design

### Key Ideas

1. Track all files in a table over time using a persistent tree structure
   - A snapshot is a complete set of files in a table
   - Each write produces and commits a new snapshot
   - Thus expensive list operations are no longer needed when querying for instance. (i.e. O(N) file listing becomes O(1) RPC to read a snapshot, thus Hive Metastore is no longer a bottleneck + Pruning is also available to speed up query planning)

   <br>

    ```mermaid
    graph LR;
        id1[[S1]] --> id2[[S2]] --> id3[[S3]] -.-> id4[[...]]
    ```

2. Gain atomicity from tacking snapshots
   - Readers will use the current snapshot or historical ones when time travelling
   - Writers optimistically create new snapshots then commit
   - Thus isolated Reads and Writes without locking the table

    ```mermaid
    graph LR;
        id5((Reader)) --> id2[[s2]]
        id6((Writer)) --> id3[[s3]]
        id1[[S1]] --> id2[[S2]] --> id3[[S3]] -.-> id4[[...]]
    ```

*NOTE: Actual implementation is much more complicated since if we were to do the above it would be incredibly slow on Writes since we are re-writing the entire table state every single time. But TLDR, there is some reuse across each snapshot.*

### Iceberg Format Overview

![apache-iceberg-table-format](./images/apache-iceberg/iceberg-metadata.png)

- ```Snapshot```: State of a table at some point in time including the set of all data files that make up the contents of the table. Data files are stored across multiple manifest files and manifests for a snapshot are listed in a single manifest list file. Every write / delete produces a new snapshot that reuses as much of the previous snapshot's metadata tree as possible to `avoid high write volumes`
- ```Snapshot Metadata File```: Metadata about the table (E.g. Schema, Partition Specification, Path to the Manifest List)
- ```Manifest List```: Metadata file that contains an entry for each manifest file associated with the snapshot. Each entry also includes a path to the manifest file & metadata (i.e. Partitions value ranges and Data file counts) allowing us to avoid reading manifests that are not required for a given operation. One per snapshot.
- ```Manifest File```: Contains a list of paths to related data or delete files with each entry for a data file including some metadata about the file (i.e. column statistics) such as per-column upper and lower bounds useful for pruning files during scan planning. A subset of a snapshot.
- ```Data File```: Physical data files (E.g. file01.parquet or file02.orc or file03.avro). Contains rows of a table.
- ```Delete File```: Encodes rows of a table that are deleted by position or data values.

<br>

## Key Features

Summary of the most essential bits that Iceberg brings to the table

### 1. Evolution

In-place table evolution is supported where a table schema can evolve even in nested structures and partition layouts can be changed when the data volume changes without any costly migration operations. (E.g. In Hive we need to rewrite to a new table if we change the names of the tables or change the granularity from daily to an hourly partition layout)

#### Schema Evolution

Iceberg support the following schema evolution changes by updating metadata (`data files` are nout updated and thus do not need to be rewritten):

- `Add`
- `Drop`
- `Rename`
- `Update`
- `Reorder`

`Correctness` through schema evolution is guaranteed to be `independent` and `free of side-effects` without file rewrites:

1. Added columns never read existing values from another column
2. Dropping a column or field does not change the values in any other column
3. Updating a column or field does not change values in any other column
4. Changing the order of columns or fields in a strcut does not change the values associated with a column or field name

Each column in a table is tracked using a `unique id` so that it never violates #1 and #2 above.

#### Partition Evolution

`Partitioning` is crucial to making queries faster by `grouping similar rows together when writing`. For example a query for logs between 10 and 12 AM as such

```sql
SELECT  level, message
FROM    logs
WHERE   event_time BETWEEN '2022-12-29 10:00:00' AND '2022-12-29 12:00:00'
```

where logs is partitioned by the date of the event_time will group our log events into files with the same event date. This allows Iceberg to skip files with other dates that do not contain useful data.

Partitioning can be updated in an existing table as queries `do not reference partition values directly`. When a partition spec evolves, the `old data` written with an `earlier spec` remains `unchanged`. `New data` is written using the `new spec` in a new layout.

Metadata for each partition version is kept separately and thus gives us `split planning` when querying where each partition layout plans files separately using the filter it derives for that specific partition layout. Allowing for `partition layouts to coexist` in the same table as such:

![partition-spec-evolution](./images/apache-iceberg/partition-spec-evolution.png)

***Note: Each data file is sotred with a partition tuple (tuple or struct of partition data) where all values in a partition tuple are the same for all rows stored in a datafile. These are produced by transforming values from row data using a partition spec which are stored unmodified unlike in Hive***

#### Hidden Partitioning in Iceberg vs Hive Partitions

In HIVE, we often have to explicitly specify a partition (which shows up as a column) for instance when writing data:

```sql
INSERT OVERWRITE INTO logs PARTITION(event_date)
SELECT  level,
        message,
        event_time,
        format_time(event_time, 'YYYY-MM-dd') AS event_date
FROM    unstructured_log_source
```

consequently our queries performed on the `logs` table must use an `event_date` filter on top of the `event_time` filter. If `event_date` were not included, Hive would scan through every file belonging to the `logs` table.

```sql
SELECT  level,
        count(1) AS count
FROM    logs
WHERE   event_time BETWEEN '2018-12-01 10:00:00' AND '2018-12-01 12:00:00'
        AND event_date = '2018-12-01'
```

Thus, this leads to a number of problems:

1. Hive cannot validate partition values
2. User has to write the queries correctly (E.g. specify the correct date format)
3. Working queries are tied to the table's partitioning scheme and cannot be changed without breaking production code

In Iceberg, we use `hidden partitioning` where partiton values are produced by taking a column value and optionally transforming it. That is, Iceberg is responsible for converting `event_time` into `event_date` and keeping track of the relationship.

Thus, there is no need for user-maintained partition columns to be specified, thus we just need to query the data we need and `Iceberg will automatically prune out files` that `do not contain matching data`.

#### Sort Order Evolution

Same as partition spec. Old data written maintains old sort order.

### 2. Time Travel & Version Rollback

Iceberg keeps a log of previous snapshots of a table as shown in the `Design` section above. This enables us to query from past snapshots or rollback tables.

### 3. Performance

Iceberg enables high performance with multi-PB tables being able to be read from a single node without the need for a distributed SQL engine to sift through table metadata.

#### Scan Planning

Fast scan planning in Iceberg enables it to fit on a single node since its metadata can be used to prune unnecessary metadata files that do not contain matching data. Thus enabling 1) Lower latency queries (Eliminating distributed scans to plan a distributed scan) and 2) Access from any client (Standalone processes can read data directly from Iceberg tables). This is achieved using:

1. `Metadata Filtering`
   - This occurs using 1) `Manifest files` (stores list of data files + partition data + column-level stats) and 2) `Manifest lists` (stores a snapshot's list of manifests + range of values for each partition field)
   - Fast scan planning occurs by first filtering manifests using the partiton value ranges in the `manifest list`
   - Then it reads each manifest to obtain the data files with the `manifest list` acting as an index over manifests thus not needing to read `all manifests`.
2. `Data Filtering`
    - `Manifest files` include a tuple of partition data + column-level stats for each data file
    - During planning query predicates are converted to predicates on the partition data and applied to filter the data files
    - Column-level stats are used to eliminate files that cannot match the query predicate

### 4. Reliability

To overcome the reliability issues (lack of atomicity) in Hive tables which tracks data files using both a central metastore for partitions and a file system for individual files making atomic changes impossible (i.e. S3 may return incorrect results due to the use of listing files to reconstruct the state of the table - O(Number of Partitions))

Reliability guarantees in Iceberg:

- `Serializable Isolation`: All changes to the table occur in a linear history of atomic table updates
- `Reliable Reads`: Always read using a consistent snapshot of the table without holding a lock
- `Version history and Rollback`: Snapshots are kept to rollback if latest job produces bad data
- `Safe file-level Operations`: With atomic changes we can safely compact small files + append late data to tables

Performance benefits:

- `O(1) RPCs to Plan`: No need to list O(n) directions in a table to plan jobs since we can read in O(1) time the latest snapshot.
- `Distributed Planning`: File pruning and predicate push-down are distributed to jobs, removing the metastore as a bottleneck
- `File Granularity Partitioning`: Distributed planning + O(1) RPC calls remove current barriers to finer-grained partitioning

#### Concurrent Writes

Iceberg uses `optimistic concurrency` where each writer assumes no other writers are operating and writes to a new table metadata before committing by swapping the new table metadata file for the existing metadata file. If it failes a retry will occur using the new current table state (Iceberg reduces cost here by maximising the work that can be reused across retries + Validates after a conflict to ensure that assumptions are met by the current table state and thus is safe to re-apply commit and its actions)

### Java API Basics

#### Creating a Schema

```java
import org.apache.iceberg.Schema;
import org.apache.iceberg.types.Types;

// Create a schema
Schema schema = new Schema(
    Types.NestedField.required(1, "level", Types.StringType.get()),
    Types.NestedField.required(2, "event_time", Types.TimestampType.withZone()),
    Types.NestedField.required(3, "message", Types.StringType.get()),
    Types.NestedField.optional(4, "call_stack", Types.ListType.ofRequired(5, Types.StringType.get()))
);

// Convert from Spark
import org.apache.iceberg.spark.SparkSchemaUtil;

Schema schema = SparkSchemaUtil.schemaForTable(sparkSession, table_name);
```

#### Creating a Iceberg Table

1. Using the `Hive Catalog`

    Use `Hive catalog` to connect to a `Hive MetaStore` to keep track of Iceberg Table.

    ```java
    import org.apache.iceberg.hive.HiveCatalog;
    import org.apache.iceberg.Table;
    import org.apache.iceberg.catalog.TableIdentifier;

    HiveCatalog catalog = new HiveCatalog();
    catalog.setConf(
        spark.sparkContext()
        .hadoopConfiguration()
    ); // Using Spark's Hadoop Config (This behaviour will change in the future)

    Map<String, String> properties = new HashMap<String, String>();
    properties.put("warehouse", "...");
    properties.put("uri", "...")

    catalog.initialize("hive", properties);

    // Creating a table
    TableIdentifier name = TableIdentifier.of("logging", "logs");
    Table table = catalog.createTable(name, schema, spec)

    // Load an existing table
    Table table = catalog.loadTable(name)

    // Load from spark
    spark = SparkSession.builder().master("local[2]").getOrCreate();
    spark.read.format("iceberg").load("bookings.test_table")
    ```

2. Using the Hadoop Catalog

   Does not need to connect to the Hive MetaStore but can only be used with HDFS or file systems which support atomic renaming. Concurrent writes are not safe with a local FS or S3

    ```java
    import org.apache.hadoop.conf.Configuration;
    import org.apache.iceberg.hadoop.HadoopCatalog;

    // Configuration
    Configuration conf = new Configuration();
    String warehousePath = "hdfs://host:8020/warehouse_path";
    HadoopCatalog catalog = new HadoopCatalog(conf, warehousePath);

    // Create Table
    TableIdentifier name = TableIdentifier.of("logging", "logs");
    Table table = catalog.createTable(name, schema, spec);

    // Load from Spark
    spark.read.format("iceberg").load("hdfs://abc:8020/path/to/table")
    ```

3. Using Hadoop Tables

    Supports tables that are stored in a directory in HDFS but once again concurrent writes with a Hadoop table aer not safe when stored in the local FS or S3.

    ```java
    import org.apache.iceberg.hadoop.HadoopTables;

    // Configuration
    Configuration conf = new Configuration();
    HadoopTables tables = new HadoopTables(conf);

    // Create Table
    Table table = tables.create(schema, spec, table_location);
    ```

#### Partition Spec & Evolution

`Partition specs` describes how Iceberg should group records into their respective data files and can be created for a table's schema using a builder (i.e. How partition values are derived from data fields). A `partition spec` has a list of fields that consist of:

- `Source Column Id` from the table's `schema` of primitive type (no Maps or Lists)
- `Partition Field Id` used to identify a partition field and is unique within a partition spec (or ALL partition specs for v2 table metadata)
- `Transform` applied to the source column to produce a partition value
- `Partition Name`

Example for a `logs` table partitioned by the `hour` of the event's timestamp and by log level

```java
import org.apache.iceberg.PartitionSpec;

PartitionSpec spec = PartitionSpec.builderFor(schema)
    .hour("event_time")
    .identity("level")
    .build();
```

To update a partition spec we can do the following

```java
table.updateSpec()
    .addField(bucket("id", 8))
    .removeField("category")
    .commit();
```

OR in `sql`

```sql
ALTER TABLE prod.db.table ADD PARTITION FIELD bucket(8, id)
ALTER TABLE prod.db.table DROP PARTITION FIELD category
```

#### Sort Order Spec & Evolution

A `sort field` comprises:

- `Source Column id`
- `Transform` (Same as `Partition Transforms`)
- `Sort Direction` - `Asc` or `Desc` only
- `Null Order` - Describes the order of null values when sorted. Can only be `nulls-first` or `nulls-last`

```java
table.replaceSortOrder()
    .asc("id", NullOrder.NULLS_LAST)
    .dec("category", NullOrder.NULL_FIRST)
    .commit();
```

#### Time Travel

To view all snapshots with spark:

```java
// Using a specific timestamp
spark.read
    .option("as-of-timestamp", "499162860000")
    .format("iceberg")
    .load("path/to/table")

// Using a specific snapshot id
spark.read
    .option("snapshot-id", 10963874102873L)
    .format("iceberg")
    .load("path/to/table")
```

OR in `sql`

```sql
-- time travel to October 26, 1986 at 01:21:00
SELECT * FROM prod.db.table TIMESTAMP AS OF '1986-10-26 01:21:00';

-- time travel to snapshot with id 10963874102873L
SELECT * FROM prod.db.table VERSION AS OF 10963874102873;
```

#### Rollbacks

```java
// To a specific time
table.rollback()
    .toSnapshotAtTime(499162860000)
    .commit();

// To a specific snapshot id
table.rollback()
    .toSnapshotId(10963874102873L)
    .commit();
```

## Miscellaneous

### Maintenance Best Practices

#### 1. Expiring Snapshots

Each write to an Iceberg table generates a new `snapshot` of the table for `time-travelling` and `rollbacks`. These `snapshots` accumulate until they are expired to remove data files that are no longer needed + keep the size of table metadata small. For example:

```java
long tsToExpire = System.currentTimeMillis() - (1000 * 60 * 60 * 24); // 1 Day
table.expireSnapshots()
    .expireOlderThan(tsToExpire)
    .commit();

// Or running in parallel using a SparkAction
SparkActions.get()
    .expireSnapshots(table)
    .expireOlderThan(tsToExpire)
    .execute();
```

#### 2. Removing Old Metadata Files

Iceberg tracks table metadata using `JSON` files which are produced upon each change to the table to ensure `atomicity`. However, this can pile up when there are frequent commits like in streaming jobs. We can autmatically clean them up by configuring write.`metadata.delete-after-commit.enabled=true` which retains metadata up till `write.metadata.previous-versions-max`

#### 3. Deletion of Orphaned Files

In distributed processing task and job failures may leave files that are not referenced by table metadata. Thus we can clean these up based on a tables location as such

```java
SparkActions
    .get()
    .deleteOrphanFiles(table)
    .execute();
```

***Note: Do not remove orphan files with a retention interval < time expected for any write to complete.***

#### 4. Compaction of Data Files

Similar to the small files issue we have in Hive, streaming queries may similarly produce small data files that can and should be `compacted into larger files` + Some tables can benefit from `rewriting manifest files` to make locating data much faster. Compaction can be done as follows:

```java
SparkActions
    .get()
    .rewriteDataFiles(table)
    .filter(Expressions.equal("date", "2020-08-18"))
    .option("target-file-size-bytes", Long.toString(500 * 1024 * 1024)) // 500 MB
    .execute();

```

#### 5. Rewriting Manifests

Manifest list metadata is used to speed up query planning and are automatically compacted in the order in which they are added. This speeds up queries when the write pattern aligns with the read filters. However this may not always be the case. To rewrite manifests for speed ups we can do the following which rewrites small manifests and groups data files by the first partition field.

```java
SparkActions
    .get()
    .rewriteManifests(table)
    .rewriteIf(file -> file.length() < 10 * 1024 * 1024) // 10 MB
    .execute();
```

### Type Conversion

#### Hive to Iceberg

|       **Hive**       |         **Iceberg**         |    **Notes**     |
|:-------------------: |:--------------------------: |:---------------: |
| boolean              | boolean                     |                  |
| short                | integer                     | auto-conversion  |
| byte                 | integer                     | auto-conversion  |
| integer              | integer                     |                  |
| long                 | long                        |                  |
| float                | float                       |                  |
| double               | double                      |                  |
| date                 | date                        |                  |
| timestamp            | timestamp without timezone  |                  |
| timestamplocaltz     | timestamp with timezone     | Hive 3 only      |
| interval_year_month  |                             | not supported    |
| interval_day_time    |                             | not supported    |
| char                 | string                      | auto-conversion  |
| varchar              | string                      | auto-conversion  |
| string               | string                      |                  |
| binary               | binary                      |                  |
| decimal              | decimal                     |                  |
| struct               | struct                      |                  |
| list                 | list                        |                  |
| map                  | map                         |                  |
| union                |                             | not supported    |

## References

- <https://medium.com/expedia-group-tech/a-short-introduction-to-apache-iceberg-d34f628b6799>
- <https://iceberg.apache.org/docs/latest/java-api-quickstart/>
