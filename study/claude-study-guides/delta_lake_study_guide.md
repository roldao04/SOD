# Delta Lake: High-Performance ACID Table Storage Study Guide

**Paper:** Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores
**Authors:** Michael Armbrust, Tathagata Das, Liwen Sun, Burak Yavuz, et al. (Databricks)
**Conference:** PVLDB 13(12): 3411-3424, 2020
**DOI:** https://doi.org/10.14778/3415478.3415560

---

## Executive Summary

Delta Lake is an open-source ACID table storage layer designed for cloud object stores (Amazon S3, Azure Blob Storage, Google Cloud Storage). It addresses fundamental challenges of using cloud object stores for data warehousing:

**Core Problem:** Cloud object stores are key-value stores with limited consistency guarantees and expensive metadata operations, making it difficult to achieve ACID transactions and high performance for tabular datasets.

**Solution:** Delta Lake maintains a transaction log stored in the cloud object store itself using Apache Parquet format. This log tracks which objects are part of each table version, provides ACID properties, and enables fast metadata operations.

**Key Innovation:** All metadata lives in the object store with no separate servers required. Transactions use optimistic concurrency protocols directly against object stores.

**Impact:** Deployed at thousands of Databricks customers processing exabytes of data per day. Reduces support issues from ~50% to nearly zero and provides speedups up to 100× in selective queries.

**Architectural Philosophy:** Creates a "lakehouse" paradigm combining data lake low costs with data warehouse management features.

---

## 1. Introduction and Motivation

### 1.1 Cloud Object Store Characteristics

**Why Cloud Object Stores?**
- Largest and most cost-effective storage systems (exabytes of data)
- Separate scaling of compute and storage resources
- Pay-as-you-go billing, economies of scale
- Industry trend: Store petabytes but run clusters only when needed

**Key Challenge:** Object stores are key-value stores, not filesystems
- No cross-key consistency guarantees
- Performance characteristics very different from HDFS
- High latency for metadata operations

### 1.2 Object Store APIs

**Basic Operations:**
- PUT/GET for individual objects (up to 5TB on S3)
- LIST operations for discovering objects (lexicographic order)
- Byte-range requests supported (efficient partial reads)
- No cheap renames or directory operations

**LIST Operation Limitations:**
- Returns only up to 1000 keys per call
- Takes tens to hundreds of milliseconds per call
- Listing millions of objects can take minutes

**Update Semantics:**
- Typically requires rewriting entire object
- Updates can be atomic per object
- Some systems support appends (Azure)

### 1.3 Consistency Properties

**Amazon S3 Example (as of paper publication):**
- Read-after-write consistency for new objects
- EXCEPTION: If client issued GET to nonexistent key before PUT, subsequent GETs might not read object (negative caching)
- LIST operations are always eventually consistent
- No atomic operations across multiple keys

**Other Providers:**
- Some offer stronger guarantees
- Still lack atomic multi-key operations

### 1.4 Performance Characteristics

**Read Operations:**
- Base latency: 5-10 ms minimum
- Throughput: 50-100 MB/s per operation
- Need several hundred KB reads to achieve half peak throughput
- Need multiple MB reads to approach peak throughput
- Parallelism required: 8-10 concurrent reads to saturate 10 Gbps network

**Write Operations:**
- Must replace whole object (or append)
- Creates tradeoff: small objects for low latency vs large objects for read performance

**Implications for Table Storage:**
1. Keep frequently accessed data sequentially close (columnar formats)
2. Make objects large but not too large
3. Avoid LIST operations when possible

### 1.5 Existing Approaches

**1. Directories of Files (Apache Hive Style)**
- Store table as collection of Parquet/ORC objects
- Optional partitioning into directories by field values
- Problems:
  - No atomicity across multiple objects
  - Eventual consistency issues
  - Poor performance (expensive LIST operations)
  - No management functionality (versioning, audit logs)

**2. Custom Storage Engines (Snowflake)**
- Separate metadata service manages state
- Problems:
  - All I/O contacts metadata service (cost, performance, availability)
  - More engineering for connectors
  - Vendor lock-in
- Apache Hive ACID uses similar approach with Hive Metastore
  - Can become bottleneck with millions of objects

**3. Metadata in Object Stores (Delta Lake's Approach)**
- Store transaction log directly in object store
- Use protocols for serializability over object store operations
- Data in Parquet format (easy connectors)
- Concurrent systems: Apache Hudi, Apache Iceberg

---

## 2. Delta Lake Storage Format

### 2.1 Overall Structure

**Directory Layout:**
```
mytable/
├── date=2020-01-01/
│   ├── 1b8a32d2ad.parquet
│   └── a2dc5244f7.parquet
├── date=2020-01-02/
│   ├── f52312dfae.parquet
│   └── ba68f6bd4f.parquet
└── _delta_log/
    ├── 000001.json
    ├── 000002.json
    ├── 000003.json
    ├── 000003.parquet
    ├── 000004.json
    ├── 000005.json
    └── _last_checkpoint
```

**Components:**
- Data objects in Apache Parquet format
- Optional Hive-style partitioning (directories by field values)
- Transaction log in `_delta_log/` subdirectory

### 2.2 Data Objects

**Format:** Apache Parquet
- Column-oriented for analytical workloads
- Diverse compression options
- Supports nested data types (semi-structured)
- Performant implementations in many engines

**Naming:**
- Unique names generated using GUIDs
- Which objects belong to each version determined by transaction log
- Objects are immutable once written

### 2.3 Transaction Log

**Log Records:**
- Stored as JSON files: `000001.json`, `000002.json`, etc.
- Zero-padded numerical IDs (increasing order)
- Each contains array of actions for that transaction

**Checkpoints:**
- Parquet files summarizing log up to specific point
- Example: `000003.parquet` combines log records 1-3
- Created periodically (default: every 10 transactions)
- Removes redundant actions, keeps only:
  - Active `add` records (files still in table)
  - Recent `remove` tombstones (within retention period)
  - Latest metadata and protocol actions
  - Latest `txn` actions per appId

**_last_checkpoint File:**
- Contains ID of most recent checkpoint
- Helps clients efficiently find starting point
- OK if out of date due to eventual consistency

### 2.4 Log Actions

**Change Metadata (metaData):**
- Changes table metadata (schema, partition columns, format, config)
- First version must contain metaData action
- Subsequent actions completely overwrite metadata

**Add or Remove Files (add, remove):**
- `add`: Adds data object to table
  - Includes statistics: record count, per-column min/max, null counts
  - Can upgrade statistics in new versions
- `remove`: Removes data object from table
  - Includes timestamp for retention policy
  - Remains as tombstone until physical deletion
- `dataChange` flag: Set to false if only rearranging data or adding stats

**Protocol Evolution (protocol):**
- Increases version of Delta protocol required
- Enables adding new features while maintaining compatibility

**Provenance Information (commitInfo):**
- Logs which user/operation performed transaction
- Can include application-specific information

**Application Transaction IDs (txn):**
- Custom application state tracking
- Fields: `appId` and `version`
- Use case: Streaming "exactly-once" semantics
- Application tracks which writes committed atomically with data

---

## 3. Access Protocols

### 3.1 Reading from Tables (5 Steps)

**Step 1:** Read `_last_checkpoint` to get recent checkpoint ID (if exists)

**Step 2:** LIST from checkpoint ID (or 0) to find `.json` and `.parquet` files
- May return non-contiguous set due to eventual consistency
- Use largest ID returned as target version

**Step 3:** Reconstruct table state
- Load checkpoint Parquet file (if present)
- Apply subsequent log records
- Result: Set of data objects with add but no remove
- Can parallelize this step (Spark jobs)

**Step 4:** Use statistics to identify relevant data objects
- Filter based on min/max values
- Skip objects that can't contain query results

**Step 5:** Read relevant data objects in parallel
- Handle eventual consistency by retrying if objects not visible

**Key Property:** Tolerates eventual consistency at every step

### 3.2 Write Transactions (5 Steps)

**Step 1:** Identify recent log record ID `r`
- Use read protocol steps 1-2

**Step 2:** Read data at version `r` (if needed)
- Use same steps as read protocol

**Step 3:** Write new data objects
- Generate names using GUIDs
- Write to correct data directories
- Can happen in parallel

**Step 4:** Atomically write log record `r+1`
- CRITICAL: Only one client can succeed
- Implementation varies by cloud provider (see below)
- If fails, can retry (reuse data objects if applicable)

**Step 5:** Optionally write checkpoint and update `_last_checkpoint`
- Only affects performance, not correctness
- Failure here doesn't corrupt data

### 3.3 Atomic Log Record Creation

**Google Cloud Storage & Azure Blob Store:**
- Use atomic put-if-absent operations

**HDFS & Azure Data Lake Storage (ADLS Gen2):**
- Use atomic rename operations
- Rename temporary file to target name or fail if exists

**Amazon S3:**
- No atomic put-if-absent or rename
- Databricks: Uses lightweight coordination service
  - Only needed for log writes (not reads or data)
  - Low load since only coordinates log record IDs
- Open source: In-memory state in Spark driver prevents conflicts within same cluster
- Pluggable LogStore API for custom coordination

---

## 4. Transaction Isolation Levels

### 4.1 Serializability

**Write Transactions:**
- All write transactions are serializable
- Serial schedule in increasing order of log record IDs
- Follows from atomic log record creation protocol

**Read Transactions:**
- Snapshot isolation by default (read protocol reads snapshot)
- Serializability possible: Execute dummy write to block concurrent writes
- Clients cache latest log IDs seen (read own writes)

### 4.2 Limitations

**Single Table Transactions:**
- Current version only supports transactions within one table
- Each table has own transaction log
- Could extend to multi-table transactions in future

### 4.3 Transaction Rates

**Limitations:**
- Write rate limited by put-if-absent latency (tens to hundreds of ms)
- High write rates cause commit failures (optimistic concurrency)
- Practical rate: Several transactions per second

**Why Sufficient:**
- Streaming jobs batch many objects per transaction
- OLAP workloads have moderate write rates
- Custom LogStore could improve commit times

**Read Transactions:**
- No contention at snapshot isolation level
- Unlimited concurrent reads

---

## 5. Higher-Level Features

### 5.1 Time Travel and Rollbacks

**Capability:**
- Query past snapshots by reading older log record IDs
- Data objects and log are immutable (MVCC)

**SQL Syntax:**
```sql
SELECT * FROM mytable TIMESTAMP AS OF '2020-01-01'
SELECT * FROM mytable VERSION AS OF 42
```

**Configuration:**
- Per-table data retention interval
- Physical deletion delayed until after retention period

**Use Cases:**
- Fix data pipeline errors
- Compare ML training data versions
- Compliance and audit requirements

**Rollback Example:**
```sql
MERGE INTO mytable target
USING mytable TIMESTAMP AS OF <old_date> source
ON source.userId = target.userId
WHEN MATCHED THEN UPDATE SET *
```

**Integration:**
- MLflow automatically records table versions used in training

**Future Feature:**
- CLONE command for copy-on-write table copies

### 5.2 UPSERT, DELETE, and MERGE

**Motivation:**
- GDPR compliance (delete user data)
- Fix errors in old records
- Late-arriving data
- Update aggregate tables

**Implementation:**
- Transactionally replace updated objects
- New add and remove records in Delta log
- Standard SQL syntax supported

**Traditional Approach Problems:**
- Directory of Parquet files: Hard to update without stopping readers
- Failures leave partial updates

**Delta Lake Solution:**
- Atomic updates via transaction log
- Concurrent readers see consistent snapshots

### 5.3 Streaming Ingest and Consumption

**Goal:** Use Delta tables as message queues (replace Kafka/Kinesis)

**Feature 1: Write Compaction**
- Writers create small objects for low latency
- Background process compacts to large objects
- Set `dataChange=false` on compaction transactions
- Streaming consumers skip compaction operations

**Feature 2: Exactly-Once Streaming Writes**
- Use `txn` action to track (appId, version)
- Atomically commit data + version information
- After failure, check which versions already committed
- Spark Structured Streaming uses this

**Feature 3: Efficient Log Tailing**
- LIST from last seen log record ID to find new records
- Lexicographically increasing IDs enable efficient discovery
- Skip records with `dataChange=false`
- Remember last record ID for restarts

**Result:** Streaming pipelines with seconds latency on object stores

### 5.4 Data Layout Optimization

**Transactional Updates Enable:**
- Compact data objects
- Change record order within objects
- Update statistics and indexes
- All without affecting concurrent operations

**OPTIMIZE Command:**
- Manually compact small objects
- Default target: 1 GB per object (customizable)
- Computes missing statistics

**Z-Ordering by Multiple Attributes:**
- Problem: Selective queries on multiple dimensions
- Hive partitioning: Exponential number of partitions
- Solution: Z-order space-filling curve
  - Creates locality in all specified dimensions
  - Works with min/max statistics for data skipping

**Example Use Case:**
- Network security data: (sourceIP, sourcePort, destIP, destPort)
- Z-ordering enables 43-54% object skipping in any dimension
- Global sort only helps first dimension

**AUTO OPTIMIZE:**
- Databricks feature: Automatically compact new data
- Triggered on writes when property enabled

**Future Features:**
- Indexes
- Expensive statistics

### 5.5 Caching

**Motivation:**
- Long-lived clusters with local storage (e.g., AWS i3 instances with SSDs)
- 50% higher cost than general-purpose for 237 GB SSD per core
- Cache improves read performance

**Safety:**
- Delta Lake objects (data, log, checkpoints) are immutable
- Safe to cache aggressively

**Databricks Implementation:**
- Transparent SSD cache for Delta tables
- Caches both data and metadata

**Performance:**
- Significant speedups for repeated queries
- Section 6 shows experimental results

### 5.6 Audit Logging

**Mechanism:**
- Transaction log stores commitInfo records
- Immutable audit trail

**Databricks Locked-Down Mode:**
- User-defined functions can't access storage directly
- Only runtime engine writes commitInfo
- Ensures immutable audit log

**SQL Command:**
```sql
DESCRIBE HISTORY mytable
```

**Information Logged:**
- Version number
- Timestamp
- User ID
- Operation (MERGE, UPDATE, etc.)
- Notebook ID (in Databricks)

**Open Source:**
- Commit information logging available
- Up to applications to ensure immutability

### 5.7 Schema Evolution and Enforcement

**Schema Changes:**
- Transactional schema updates
- Can update underlying objects if needed
- Or keep history to read old Parquet files without rewriting

**Schema Enforcement:**
- Delta clients validate new data follows table schema
- Catches many user errors (wrong schema appended)

**Problem Before Delta Lake:**
- Jobs writing Parquet to same directory
- Hard to trace down wrong-schema errors

### 5.8 Connectors to Query Engines

**Full Connectors:**
- Apache Spark (SQL and Structured Streaming)
- Uses Spark data source API

**Read-Only Integrations:**
- Apache Hive
- Presto
- AWS Athena
- AWS Redshift
- Snowflake

**ETL/CDC Tools:**
- Fivetran
- Informatica
- Qlik
- Talend

**Symlink Manifest Files:**
- Mechanism for Hive-compatible systems
- Text file `_symlink_format_manifest` lists paths
- Generated with SQL command
- Provides consistent snapshot per partition
- Enables queries from Presto, Athena, Redshift, Snowflake

**Apache Hive:**
- Open source connector using plugin APIs

---

## 6. Use Cases

### 6.1 Data Engineering and ETL

**Traditional Architecture:**
- Data lake (object store) for raw data
- ETL processes
- Data warehouse for analytics

**Problems:**
- Multiple copies of data
- Complex maintenance
- Latency to make data queryable

**Delta Lake Solution:**
- Perform ETL directly on object store
- ACID transactions for reliability
- UPSERT/MERGE for updates
- Time travel and audit logs
- Single storage system (simpler architecture)

**Tools:**
- SQL queries (familiar to data engineers)
- Programmatic APIs (Apache Spark)
- Can expose data to other workloads on separate clusters

**Industries:**
- Financial services
- Healthcare
- Media

### 6.2 Data Warehousing and BI

**Requirements:**
- ETL/ELT functionality
- Efficient queries
- Interactive workloads

**Delta Lake Features:**
- Columnar format (Parquet)
- Data layout optimization
- Min/max statistics
- SSD caching
- All enabled by transactional design

**Advantage:**
- Analysts get fresh data (no separate ingest)
- Query directly with SQL or BI tools (Tableau)

**Databricks Optimizations:**
- Vectorized execution engine for BI
- Spark runtime optimizations

### 6.3 Compliance and Reproducibility

**GDPR and Privacy Requirements:**
- Efficient deletion of user data
- Traditional data lakes designed for immutable data

**Delta Lake Solution:**
- UPSERT, MERGE, DELETE operations
- Audit logging (Section 4.6)

**Machine Learning Reproducibility:**
- Time travel to replay with same data version
- Integration with MLflow
- Automatically record dataset version used for training

**Industries:**
- Multiple industries requiring data governance

### 6.4 Computer System Event Data (SIEM)

**Use Case:** Large technology company security platform

**Scale:**
- Petabytes of data
- Hundreds of TB per day ingestion
- Events: TCP/UDP flows, authentication, SSH logins, etc.

**Workloads:**
- Programmatic ETL, SQL, graph analytics, ML
- Streaming jobs for real-time detection
- 100+ analysts querying interactively

**Challenges:**
- Vast amounts of data collected automatically
- Long retention (forensic analysis months later)
- Queries along multiple dimensions (sourceIP, destIP, time, etc.)

**Delta Lake Features Used:**
- Z-ordering by multiple dimensions (sourceIP, sourcePort, destIP, destPort)
- Min/max statistics for data skipping
- AUTO OPTIMIZE
- Time travel
- ACID transactions

**Performance:**
- Selective queries: High locality in Z-ordered data
- 43-54% object skipping per dimension
- Maintains hundreds of developers collaborating

### 6.5 Bioinformatics

**Data Sources:**
- DNA sequencing, RNA sequencing
- Electronic medical records
- Time series from medical devices
- Public datasets (UK Biobank: 500,000 individuals)

**Traditional Formats:**
- SAM, BAM, VCF
- Now migrating to Parquet (Big Data Genomics project)

**Delta Lake Benefits:**
- Fast multi-dimensional queries (Z-ordering)
- ACID transactions
- Efficient UPSERT/MERGE

**Performance:**
- 100× speedups in some cases
- Network traffic dataset: 93% data skipping with multi-attribute queries

**Tools:**
- Glow (2019): Open source toolkit for genomics using Delta

### 6.6 Media Datasets for Machine Learning

**Problem:**
- Millions of small image files (few KB each)
- LIST operations take minutes
- Hard to parallelize reads for GPU inference

**Delta Lake Solution:**
- Store images as BINARY records in Delta table
- Faster inference queries
- Stream processing support
- ACID transactions

**Users:**
- E-commerce companies
- Travel companies
- Managing millions of user-uploaded images

---

## 7. Performance Experiments

### 7.1 Impact of Many Partitions

**Experiment Setup:**
- 16-node clusters, i3.2xlarge VMs (8 vCPUs, 61 GB RAM, 1.9 TB SSD)
- Small tables: 33M rows, 1K to 1M partitions
- Query: Simple sum over all records
- Systems: Databricks Runtime, third-party Hive/Presto

**Results:**
- **Hive:** >1 hour for 10K partitions
- **Presto:** >1 hour for 100K partitions
- **Databricks Parquet:** 450 seconds for 100K partitions (parallel LIST)
- **Delta Lake (no cache):** 108 seconds for 1M partitions
- **Delta Lake (cached):** 17 seconds for 1M partitions

**Conclusion:**
- Delta Lake log avoids expensive LIST operations
- Centralized metadata dramatically faster
- SSD caching provides further speedup

### 7.2 Impact of Z-Ordering

**Experiment Setup:**
- Network flow dataset: (sourceIP, sourcePort, destIP, destPort)
- 100 Parquet objects
- Queries: Filter on specific value in each dimension
- Compare: Global sort order vs Z-order

**Results (% Objects Skipped):**

| Dimension | Global Sort | Z-Order |
|-----------|-------------|---------|
| sourceIP | 99% | 67% |
| sourcePort | 0% | 60% |
| destIP | 0% | 47% |
| destPort | 0% | 44% |
| Average | 25% | 54% |

**Real-World Example:**
- 500 TB network traffic dataset
- Multi-attribute queries with Z-ordering: 93% data skipping

**Conclusion:**
- Z-ordering provides balanced performance across dimensions
- Single sort order only helps first dimension
- More objects → better skipping (each object covers smaller range)

### 7.3 TPC-DS Performance

**Experiment Setup:**
- TPC-DS power test
- 1 master + 8 workers, i3.2xlarge VMs
- 1 TB data in S3
- Systems: Databricks Delta, Databricks Parquet, third-party Spark/Presto

**Results (Power Test Duration):**
- **Databricks Delta:** 0.93 hours
- **Databricks Parquet:** 0.99 hours
- **Third-party Spark:** 1.44 hours
- **Third-party Presto:** 3.76 hours

**Analysis:**
- Delta Lake outperforms all configurations
- Delta speedup primarily on longer queries
- Databricks Runtime optimizations significant vs third-party Spark

### 7.4 Write Performance

**Experiment Setup:**
- Load 400 GB TPC-DS store_sales table
- From CSV to Delta or Parquet
- 1 master + 8 workers, i3.2xlarge

**Results:**
- Delta and Parquet: Similar load times
- Statistics collection doesn't add significant overhead

**Conclusion:**
- Delta Lake write overhead negligible

---

## Key Concepts and Definitions

### Core Concepts

**Cloud Object Store:**
- Key-value store for binary blobs (objects)
- Examples: Amazon S3, Azure Blob Storage, Google Cloud Storage
- Characteristics: Eventual consistency, expensive metadata operations, no atomic multi-key operations

**ACID Properties:**
- **Atomicity:** All operations in transaction succeed or all fail
- **Consistency:** Transactions maintain data integrity constraints
- **Isolation:** Concurrent transactions don't interfere
- **Durability:** Committed transactions persist

**Transaction Log:**
- Sequence of log records describing table changes
- Stored as JSON files with increasing numerical IDs
- Contains actions: add, remove, metaData, protocol, commitInfo, txn

**Checkpoint:**
- Parquet file summarizing transaction log up to specific point
- Removes redundant actions
- Enables fast table reconstruction

**Lakehouse:**
- Combines data lake and data warehouse features
- Low-cost object storage with DBMS management capabilities

### Storage Format

**Apache Parquet:**
- Column-oriented file format
- Compression support
- Nested data types
- Footer contains metadata and statistics

**Log Record Actions:**
- **add:** Add data file to table (includes statistics)
- **remove:** Remove data file (tombstone with timestamp)
- **metaData:** Change table schema/configuration
- **protocol:** Update Delta protocol version
- **commitInfo:** Provenance information
- **txn:** Application transaction ID tracking
- **dataChange flag:** Indicates if action changes data content

**Statistics:**
- Stored in add actions
- Record count, per-column min/max, null counts
- Enable data skipping

### Transaction Protocols

**Optimistic Concurrency Control:**
- Multiple transactions proceed without locks
- At commit time, check for conflicts
- Retry on conflict

**Put-If-Absent:**
- Atomic operation: Write only if key doesn't exist
- Used to create log records atomically
- Available on GCS, Azure Blob Store

**Atomic Rename:**
- Rename file atomically, fail if target exists
- Used on HDFS, ADLS Gen2

**Snapshot Isolation:**
- Transactions read consistent snapshot
- Don't see concurrent writes
- May have write-write conflicts

**Serializability:**
- Strictest isolation level
- Equivalent to serial execution
- All Delta writes are serializable

### Data Layout

**Partitioning:**
- Organize data by field values into directories
- Example: `/mytable/date=2020-01-01/`
- Hive-style naming convention

**Z-Order Curve:**
- Space-filling curve for multi-dimensional ordering
- Interleaves bits from multiple dimensions
- Creates locality along all dimensions

**Data Skipping:**
- Use statistics (min/max) to skip reading irrelevant files
- Effective with Z-ordering for multi-dimensional queries

**Compaction:**
- Combine small files into larger ones
- Improves read performance
- Background process, doesn't affect readers

### Features

**Time Travel:**
- Query table at past version
- Read older log record ID
- Requires data retention policy

**Exactly-Once Semantics:**
- Streaming guarantee: Each record processed exactly once
- Use txn actions to track progress atomically

**Log Tailing:**
- Efficiently discover new log records
- LIST from last seen ID
- Enables streaming consumption

**Audit Logging:**
- Immutable record of all changes
- Based on commitInfo actions
- DESCRIBE HISTORY command

---

## Comparison Tables

### Delta Lake vs Traditional Approaches

| Feature | Directories of Files | Custom Storage Engines | Delta Lake |
|---------|---------------------|----------------------|------------|
| **Metadata location** | None (LIST operations) | Separate service | Object store |
| **Atomicity** | Per-object only | Multi-object | Multi-object |
| **Consistency** | Eventual | Strong | Serializable writes, snapshot reads |
| **Server requirements** | None | Always running | None (optional for S3) |
| **LIST operations** | Required, expensive | Service manages | Avoided via log |
| **Time travel** | Not supported | Varies | Supported |
| **UPSERT/DELETE** | Difficult | Supported | Supported |
| **Streaming** | Not optimized | Varies | Optimized |
| **Connector complexity** | Low (just Parquet) | High (proprietary API) | Low (Parquet + log) |
| **Example systems** | Apache Hive | Snowflake, Hive ACID | Delta Lake, Hudi, Iceberg |

### Cloud Object Store Consistency Models

| Object Store | Read-After-Write | LIST Consistency | Atomic Multi-Key | Atomic Rename |
|--------------|-----------------|------------------|------------------|---------------|
| **Amazon S3** | Yes (with exceptions) | Eventual | No | No |
| **Azure Blob Store** | Yes | Eventual | Put-if-absent | No |
| **Google Cloud Storage** | Yes | Strong | Put-if-absent | No |
| **Azure ADLS Gen2** | Yes | Strong | Filesystem API | Yes |
| **HDFS** | Yes | Strong | Filesystem API | Yes |

### Delta Lake vs Similar Systems

| Feature | Delta Lake | Apache Hudi | Apache Iceberg |
|---------|-----------|-------------|----------------|
| **Development start** | 2016 | ~2016 | ~2017 |
| **Open sourced** | 2019 | 2019 | 2018 |
| **Z-ordering** | Yes | No | No |
| **Streaming source** | Yes | No | Limited |
| **Caching support** | Yes (Databricks) | No | No |
| **Auto optimization** | Yes (Databricks) | No | No |
| **Concurrent writers** | Multiple | Single | Multiple |
| **Connectors** | Spark, Hive, Presto, Athena, Redshift, Snowflake | Spark, Hive, Presto | Spark, Hive, Presto |
| **Data format** | Parquet | Parquet/ORC | Parquet/ORC/Avro |

### Storage System Comparison

| System | Storage Type | Transaction Support | Scale | Latency | Use Case |
|--------|-------------|-------------------|-------|---------|----------|
| **HDFS** | Distributed filesystem | File-level | Petabytes | Low-Medium | Batch processing |
| **Cloud Object Store** | Key-value store | Per-key | Exabytes | Medium-High | Cost-effective storage |
| **HBase/Kudu** | Distributed table store | Row-level | Petabytes | Low | Low-latency OLTP |
| **Delta Lake** | Table layer over object store | Table-level | Exabytes | Medium | Analytics, ETL, BI |
| **Data Warehouse** | Custom engine | Table-level | Petabytes | Low-Medium | BI, SQL analytics |
| **Kafka** | Message queue | Append-only | Petabytes | Very Low | Streaming |

---

## Exam Preparation Tips

### Critical Concepts to Master

1. **Why Delta Lake Exists**
   - Understand limitations of cloud object stores (eventual consistency, expensive LIST, no multi-key atomicity)
   - Know the problems with "directories of files" approach
   - Explain the tradeoff between custom storage engines and Delta's approach

2. **Transaction Log Design**
   - How log records provide ACID properties
   - Role of checkpoints in performance
   - Why objects are immutable (enables time travel, caching, MVCC)

3. **Access Protocols**
   - 5-step read protocol (tolerates eventual consistency)
   - 5-step write protocol (optimistic concurrency)
   - How atomic log record creation varies by cloud provider

4. **Z-Ordering**
   - Why single-attribute sorting insufficient
   - How Z-order curve creates multi-dimensional locality
   - Relationship between Z-ordering and data skipping

5. **Higher-Level Features**
   - How transactions enable each feature
   - Exactly-once streaming (txn actions)
   - Time travel (immutable objects + log)
   - Efficient UPSERT/DELETE (atomic object replacement)

### Common Exam Question Types

**Compare and Contrast:**
- Delta Lake vs directories of Parquet files
- Delta Lake vs Snowflake/custom storage engines
- Z-ordering vs partitioning vs single-dimension sorting
- Snapshot isolation vs serializability

**Architecture and Design:**
- Explain how Delta Lake achieves ACID without separate servers
- Describe the trade-offs in Delta Lake's design
- How would you modify Delta Lake for multi-table transactions?
- Why use Parquet as the data format?

**Performance Analysis:**
- Why are LIST operations expensive on cloud object stores?
- How does Delta Lake avoid these costs?
- Explain the performance impact of small files
- Why does Z-ordering improve query performance?

**Practical Application:**
- Design a data pipeline using Delta Lake
- How to implement exactly-once streaming ingestion
- Optimize a table for multi-dimensional queries
- Rollback a table to fix a data error

### Key Numbers to Remember

- **Cloud object store characteristics:**
  - Base latency: 5-10 ms
  - Throughput: 50-100 MB/s per operation
  - S3 LIST: Returns up to 1000 keys per call
  - Need 8-10 concurrent reads to saturate 10 Gbps network

- **Delta Lake deployment:**
  - Thousands of customers
  - Exabytes of data per day
  - Support issues reduced from 50% to nearly zero
  - Up to 100× speedups in selective queries

- **Default configurations:**
  - Checkpoints every 10 transactions
  - Default object size: 1 GB (OPTIMIZE command)

- **Performance experiments:**
  - Delta Lake: 17 seconds cached, 108 seconds uncached for 1M partitions
  - Hive: >1 hour for 10K partitions
  - Z-ordering: 54% average object skipping vs 25% for single sort

### Red Flags (Common Misconceptions)

❌ Delta Lake requires a separate metadata service
✓ All metadata is in the object store; no servers required (except optional S3 coordinator)

❌ Delta Lake is slower than Parquet for writes
✓ Write performance is similar (statistics collection has negligible overhead)

❌ Cloud object stores provide strong consistency
✓ Most have eventual consistency (though improving over time)

❌ Z-ordering solves all performance problems
✓ It helps multi-dimensional queries but isn't a complete solution; statistics and data skipping also critical

❌ Delta Lake supports multi-table transactions
✓ Currently only single-table transactions (acknowledged limitation)

❌ Time travel keeps all data forever
✓ Configurable retention policy; data physically deleted after retention period

### How to Approach Design Questions

1. **Identify Requirements:**
   - Transaction guarantees needed?
   - Query patterns (dimensions, selectivity)?
   - Write patterns (batch, streaming, update frequency)?
   - Latency requirements?

2. **Consider Trade-offs:**
   - Cost vs performance (object storage vs custom engine)
   - Write latency vs read performance (object size)
   - Metadata overhead vs query speed
   - Complexity vs capabilities

3. **Apply Delta Lake Features:**
   - Partitioning for coarse filtering
   - Z-ordering for multi-dimensional selectivity
   - Streaming features for low-latency ingestion
   - Time travel for reproducibility
   - OPTIMIZE for compaction

4. **Evaluate Limitations:**
   - Write transaction rate limits
   - Single-table transaction scope
   - Object store latency floor
   - Statistics maintenance overhead

---

## Practice Questions

### Conceptual Questions

1. **Q:** Why is it challenging to implement ACID transactions over cloud object stores?

   **A:** Cloud object stores are key-value stores with no atomic operations across multiple keys and eventual consistency per key. Traditional table operations need to update multiple files atomically, which isn't directly supported. LIST operations are expensive, making metadata management slow.

2. **Q:** Explain how Delta Lake achieves serializability for write transactions without a centralized coordinator.

   **A:** Delta Lake uses optimistic concurrency control with atomic creation of log record files. Each transaction writes a log record with a specific ID (e.g., 000005.json). The atomic put-if-absent (or atomic rename) ensures only one transaction can create each log ID. This creates a total order of transactions, providing serializability.

3. **Q:** What are the trade-offs between Delta Lake's approach and a custom storage engine like Snowflake?

   **A:** Delta Lake stores all metadata in object store, requires no running servers, provides direct high-bandwidth access, and enables easy connectors (Parquet-based). Snowflake requires frontend servers always running, adds latency/cost for streaming through frontend, but may provide better optimization and performance for some workloads. Delta Lake prioritizes operational simplicity and openness.

4. **Q:** How does Delta Lake's checkpoint mechanism improve performance?

   **A:** Checkpoints compress transaction log into Parquet format, removing redundant actions (e.g., add followed by remove). This reduces the number of files to read when reconstructing table state. Parquet format is columnar and efficient for querying metadata. Clients can start from last checkpoint instead of reading entire log.

5. **Q:** Why doesn't Delta Lake support multi-table transactions?

   **A:** Each Delta table has its own separate transaction log. Coordinating across multiple logs would require additional synchronization mechanism and increase contention. Single-table transactions are sufficient for most analytics workloads. Could extend to multi-table in future with shared log.

### Technical Deep-Dive Questions

6. **Q:** Describe the 5-step read protocol and explain how it tolerates eventual consistency.

   **A:**
   1. Read _last_checkpoint (OK if stale)
   2. LIST from checkpoint to find new .json/.parquet files (gaps OK, use largest ID)
   3. Reconstruct table state from checkpoint + logs (parallel)
   4. Use statistics to find relevant objects
   5. Read objects (retry if not visible)

   Tolerates consistency: Can read stale checkpoint, find newer logs via LIST; handles gaps in log IDs by targeting largest; retries if objects not visible.

7. **Q:** How do txn actions enable exactly-once semantics for streaming applications?

   **A:** Streaming application writes (appId, version) pair in txn action within same log record as add/remove actions. This atomically commits both data changes and version information. After failure, application checks log for its appId to find last committed version, then replays from next offset in input stream. Prevents duplicate writes.

8. **Q:** Compare the effectiveness of Z-ordering vs single-attribute sorting for the network security use case (sourceIP, sourcePort, destIP, destPort).

   **A:** Single sort order (e.g., sourceIP, sourcePort, destIP, destPort): 99% object skipping for sourceIP queries, 0% for other dimensions (each file covers full range). Average: 25%. Z-ordering by all four: 67% sourceIP, 60% sourcePort, 47% destIP, 44% destPort. Average: 54%. Z-ordering provides balanced performance across dimensions.

9. **Q:** How does Delta Lake enable streaming applications to use tables as message queues?

   **A:** Three features: (1) Write compaction - Writers create small objects for low latency, background process compacts, dataChange=false lets consumers skip compaction. (2) Exactly-once writes - txn actions track progress. (3) Efficient log tailing - LIST from last log ID to discover new records using lexicographic ordering. Together enable seconds-latency streaming on object stores.

10. **Q:** What happens if a client fails while writing a checkpoint in step 5 of the write protocol?

    **A:** No data corruption. Checkpoint writing only affects performance, not correctness. If checkpoint Parquet file is written but _last_checkpoint not updated, other clients still use previous checkpoint. If _last_checkpoint updated but checkpoint incomplete, clients may get error but can fall back to earlier checkpoint. Log records remain source of truth.

### Design and Application Questions

11. **Q:** Design a data pipeline that ingests streaming clickstream data, performs ETL, and serves analysts. How would you use Delta Lake?

    **A:**
    - **Raw data table:** Streaming writes of small objects (low latency), Z-order by (timestamp, userId, pageId)
    - **Background compaction:** AUTO OPTIMIZE or scheduled OPTIMIZE to combine small files
    - **ETL:** Spark Structured Streaming reads raw table, transforms, writes to aggregate tables using txn actions for exactly-once
    - **Serving:** Analysts query aggregate tables with SQL, benefit from statistics and Z-ordering
    - **Features:** Time travel for reproducing analyses, audit logging for governance

12. **Q:** A table has billions of objects and queries are slow. What Delta Lake features would you use to improve performance?

    **A:**
    - **Checkpoints:** Ensure recent checkpoints exist (every 10 transactions)
    - **Compaction:** Run OPTIMIZE to reduce number of small objects (target 1 GB)
    - **Z-ordering:** Identify query dimensions, run OPTIMIZE ZORDER BY (dim1, dim2, ...)
    - **Statistics:** Ensure add actions include min/max statistics for data skipping
    - **Caching:** Enable SSD caching if on Databricks with long-lived clusters
    - **Partitioning:** Consider Hive-style partitioning for coarse filtering (but avoid too many partitions)

13. **Q:** How would you implement GDPR "right to be forgotten" (delete all data about a user) in Delta Lake?

    **A:**
    ```sql
    DELETE FROM events WHERE userId = '<user_id>'
    ```

    Delta Lake transactionally identifies all Parquet files containing the user's data, rewrites them without those records, creates new add actions for new files and remove actions for old files, atomically commits via log record. Concurrent readers unaffected (see old version). Old files physically deleted after retention period.

14. **Q:** Compare using Delta Lake vs Kafka for a streaming pipeline. When would you choose each?

    **A:**
    - **Kafka:** Sub-second latency critical, high message rate (millions/sec), temporary data (hours-days retention), complex stream processing topologies
    - **Delta Lake:** Seconds-minute latency acceptable, fewer but larger messages, long-term storage needed, integration with batch analytics, SQL queries on stream data
    - **Hybrid:** Kafka for real-time ingestion → Delta Lake for durable storage and analytics
    - Delta Lake can replace Kafka when: Latency requirements relaxed, batch jobs write many objects per transaction, want unified storage

15. **Q:** You need to train multiple ML models on the exact same data. How does Delta Lake help?

    **A:**
    - Query table at specific version: `SELECT * FROM features VERSION AS OF 42`
    - MLflow integration automatically records table version used for training
    - Compare models fairly (same data version)
    - Reproduce results (reload exact data version)
    - Time travel ensures data available within retention period
    - Audit log shows provenance

### Problem-Solving Questions

16. **Q:** A streaming job crashed after writing 100 objects but before committing. How do you handle this with Delta Lake?

    **A:**
    - Objects written (step 3) but log record not created (step 4)
    - Objects are "orphaned" - not referenced in log
    - Job restarts, checks log for its txn action with appId
    - Doesn't find its version → knows to retry from last committed version
    - Options: (1) Write new objects (orphaned ones garbage collected later), or (2) Reuse orphaned objects if references saved
    - Exactly-once semantics preserved

17. **Q:** How would you migrate an existing Hive table (millions of Parquet files in S3) to Delta Lake?

    **A:**
    - **Option 1 - In-place conversion:**
      - Run `CONVERT TO DELTA` command
      - Creates Delta log with add actions for existing Parquet files
      - No data rewriting needed
      - May need to run OPTIMIZE later for better statistics/layout

    - **Option 2 - Copy data:**
      - Create new Delta table
      - Batch insert from Hive table in parallel
      - Opportunity to optimize layout (Z-order, partitioning)
      - More resource-intensive but cleaner result

18. **Q:** A query filters on three dimensions: (date, customerId, productId). The table is partitioned by date. How would you optimize for queries on customerId and productId?

    **A:**
    - Partitioning by date provides coarse filtering (skip date partitions)
    - Within each date partition, Z-order by (customerId, productId)
    - Command: `OPTIMIZE mytable WHERE date >= '2020-01-01' ZORDER BY (customerId, productId)`
    - Statistics (min/max) on Z-ordered files enable skipping
    - Consider incremental optimization (only recent dates)
    - Monitor query performance, adjust if needed

19. **Q:** Two transactions try to commit at the same time. Walk through what happens.

    **A:**
    - Both read current version r (steps 1-2)
    - Both prepare their changes (step 3)
    - Both try to write log record r+1 (step 4)
    - Atomic put-if-absent/rename ensures only one succeeds
    - First transaction: Commits successfully
    - Second transaction: Gets conflict error
    - Second transaction: Retries from step 1 (read new version r+1), checks for conflicts in data modified
    - If no conflicts: Can commit as r+2
    - If conflicts: Application-level conflict resolution

20. **Q:** Explain why Delta Lake stores statistics in add actions rather than in Parquet footers.

    **A:**
    - Parquet footers require reading each Parquet file to get statistics (expensive with cloud latency)
    - Delta Lake stores statistics in log/checkpoint (Parquet format)
    - Query planning reads one checkpoint file, gets statistics for all data files
    - Massive speedup: Read 1 file vs thousands/millions
    - Can also add new statistics types without rewriting data files
    - Checkpoint is column-oriented, efficient to query specific statistics

---

## Quick Reference Card

### Delta Lake Architecture

```
┌─────────────────────────────────────────────┐
│         Delta Lake Table Directory          │
├─────────────────────────────────────────────┤
│ Data Objects (Parquet)                      │
│ • Immutable                                 │
│ • GUID names                                │
│ • Optional Hive-style partitioning          │
├─────────────────────────────────────────────┤
│ Transaction Log (_delta_log/)               │
│ • JSON log records: 000001.json, 000002.json│
│ • Parquet checkpoints: 000010.parquet       │
│ • _last_checkpoint file                     │
├─────────────────────────────────────────────┤
│ Properties                                  │
│ • ACID transactions                         │
│ • Serializable writes                       │
│ • Snapshot isolation reads                  │
│ • No separate servers required              │
└─────────────────────────────────────────────┘
```

### Log Actions

| Action | Purpose | Key Fields |
|--------|---------|------------|
| `add` | Add data file | path, size, partitionValues, dataChange, stats |
| `remove` | Remove data file | path, deletionTimestamp, dataChange |
| `metaData` | Change schema/config | schemaString, partitionColumns, configuration |
| `protocol` | Update protocol version | minReaderVersion, minWriterVersion |
| `commitInfo` | Audit information | timestamp, userId, operation, operationParameters |
| `txn` | App transaction tracking | appId, version |

### Read Protocol

```
1. Read _last_checkpoint → checkpoint_id
2. LIST from checkpoint_id → [log files]
3. Reconstruct state from checkpoint + logs → {active objects}
4. Filter objects using statistics → {relevant objects}
5. Read relevant objects in parallel → query results
```

### Write Protocol

```
1. Identify current version r → read log
2. Read table at version r (if needed) → data
3. Write new data objects → GUIDs
4. Atomically write log record r+1 → COMMIT
5. (Optional) Write checkpoint, update _last_checkpoint
```

### Key SQL Operations

```sql
-- Time travel
SELECT * FROM table TIMESTAMP AS OF '2020-01-01'
SELECT * FROM table VERSION AS OF 42

-- Optimize
OPTIMIZE table
OPTIMIZE table ZORDER BY (col1, col2)

-- Maintenance
VACUUM table RETAIN 168 HOURS  -- Clean up old files
DESCRIBE HISTORY table         -- Audit log

-- Updates
UPDATE table SET col = val WHERE condition
DELETE FROM table WHERE condition
MERGE INTO target USING source ON condition
  WHEN MATCHED THEN UPDATE SET *
  WHEN NOT MATCHED THEN INSERT *
```

### Performance Optimization Checklist

- [ ] Use appropriate object size (default 1 GB)
- [ ] Create checkpoints regularly (default every 10 commits)
- [ ] Z-order by query dimensions for multi-dimensional selectivity
- [ ] Partition by commonly filtered field (avoid too many partitions)
- [ ] Ensure statistics collected (run OPTIMIZE if needed)
- [ ] Configure retention policy based on time travel needs
- [ ] Enable caching for long-lived clusters with local storage
- [ ] Use AUTO OPTIMIZE for streaming ingestion
- [ ] Compact small files from streaming writes
- [ ] Monitor number of data files (LIST/checkpoint overhead)

### Common Pitfalls

❌ **Too many small files** → Run OPTIMIZE to compact
❌ **Too many partitions** → LIST operations expensive
❌ **Single dimension sort** → Use Z-ordering for multi-dimensional queries
❌ **Short retention** → Time travel data deleted too soon
❌ **High write transaction rate** → Optimistic concurrency conflicts
❌ **Ignoring statistics** → Can't skip irrelevant data files
❌ **Not using checkpoints** → Slow table state reconstruction
❌ **Wrong isolation level** → Understanding snapshot vs serializable

### When to Use Delta Lake

✓ **Yes:**
- Analytical workloads on cloud object stores
- ETL pipelines needing ACID guarantees
- Data warehousing and BI on object storage
- Streaming ingestion with exactly-once semantics
- Multi-dimensional selective queries
- Compliance requirements (auditing, deletion)
- Machine learning data management

✗ **No (consider alternatives):**
- Sub-second latency streaming (use Kafka)
- Very high write transaction rate (>100/sec)
- Multi-table transactions required
- Primary key lookups (use HBase/Kudu)
- Need advanced indexes (consider data warehouse)

### Related Technologies

- **Apache Parquet:** Columnar data format used by Delta Lake
- **Apache Hive:** Originated "directories of files" approach
- **Apache Spark:** Primary compute engine for Delta Lake
- **Apache Hudi:** Similar log-based table format (single writer)
- **Apache Iceberg:** Similar log-based table format (different features)
- **AWS S3, Azure Blob Storage, GCS:** Cloud object stores
- **Snowflake:** Custom storage engine approach
- **MLflow:** ML model management, integrates with Delta Lake

### Key Paper Statistics

- **Published:** VLDB 2020
- **Scale:** Exabytes per day, billions of objects
- **Performance:** Up to 100× speedups, 17s for 1M partitions (cached)
- **Adoption:** Thousands of customers
- **Support issues:** Reduced from 50% to nearly zero
- **Open source:** 2019, Apache 2 license
- **Z-ordering:** 54% avg object skipping vs 25% single sort

---

## Summary

Delta Lake solves the fundamental problem of achieving ACID transactions and high performance on cloud object stores by:

1. **Storing a transaction log in the object store itself** - No separate servers required
2. **Using optimistic concurrency protocols** - Atomic log record creation provides serializability
3. **Building on Parquet format** - Easy connectors, broad compatibility
4. **Checkpointing for performance** - Efficient metadata queries
5. **Providing higher-level features** - Time travel, UPSERT/DELETE, streaming, Z-ordering, caching

This design enables a "lakehouse" architecture that combines the cost-effectiveness of data lakes with the management capabilities of data warehouses, simplifying enterprise data architectures and improving both reliability and performance.

The system demonstrates that ACID transactions can be achieved over eventually consistent storage with careful protocol design, and that centralized metadata management is critical for performance at scale with cloud object stores.

---

**End of Study Guide**
