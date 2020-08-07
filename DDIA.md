# Design Data Intensive Application

## Foundation of Data Systems

### Reliable, Scalable, and Maintainable Applications

#### Reliability: fault-tolerant / resilient

- Hardware Faults
- Software Errors
- Human Errors

#### Scalability

Scalability depends on the load parameter we care about: if the system grows in **a particular way**, what are the options for coping with the growth? Meaning we have to consider the specific use case.

load parameters
- requests per second for web servers
- ratio of reads to wries for databases
- hit rate for a caches
- etc.

**Performance**
- throughput for batch job
- response time for online system

    use percentile for better understanding of the service performance

**Scaling**
- scale up
- scal out

#### Maintainability
- operability: make life easy for operations
- simplicity: clean code basically
- evlvability: make changes easy

### Data Models and Query Languages

#### Relational Data Model vs Document Model vs Graph Model

- relation db natrually provides join functionality and many-to-one, many-to-many relations; document model makes one-to-many relation explicit, and usually provides better data locality
- relation db enforce schema on the db side, while document dbs are schemaless, which is more flexible if we want to change the structure
- data in relation db is easier to update; in-place update is possible in document db only if document size keeps the same

Graph databases target use cases where anything is potentially related to everything

### Storage and Retrieval (database implementation)

Index: some metadata the db keeping track of that speeds up read operation but might slows down writes.

#### Hash Indexes
log structured storage + in memory hash map from key to its latest update location

Compaction is required to prevent disk space full

benifits
- all write operations are sequential writes, faster on dick
- segment files are append-only or immutable, make concurrency and crash recovery simpler
- compacting old segments avoids data files getting fragmented

limits
- hash table has to fit in memory
- range queries are not efficient

#### Sorted String Table (SSTable) and Log-Structured Merge-Tree (LSM-Tree)

Comparing to log structured storage, SSTable requires the sequence of key-value pairs to be sorted by key.

in memory tree structured cache to keep keys sorted (memtable), written to disk when tree is large enough - **disk writes are still sequential**

goods:
- merging segments is simple: the merge step in mergesort
- a sparse in memory hash index is sufficient to look up key quickly
- could compress a segment before writing to disk, with each hash index pointing to the start of a segment, saving space and disk IO bandwidth

#### B-Tree

#### Other Indexing Structures
- storing values within the index
- multi-column indexes (for geographic location search for example)
- full-text search and fuzzy indexes
- memory databases

#### Data Warehousing

Specifically for analytics purpose. Data are usually dump from OLTP databases to warehourses through the ETL (extract-trasform-load) process. Data warehouses often provide SQL interface as well.

Start schema and snowflake schema: a huge central fact table storing lots of events, with reference to many surrounding dimension tables. Fact table tends to be very wide -- a few hundreds columns wide -- to be flexible enough for different analytics.

Since single analytics tends only uses a few columns of a lot of rows, data warehouses go for column oriented storage -- store columns in seperate columns files. This relies on each column file containing the rows in the same order.

### Encoding and Evolution

#### Encoding and Decoding
In case application requirement changes, we don't want to deploy new version all at once to prevent service downtime. Rolling upgrade means coexistance of old version and new version, both code and data. This requires **backward compatibility** and **forward compatibility**.

**Language bulit in package**

Language built-in encoding packages are not ideal under most cases due to several reasons:
- the encoding would be tied to a particular programming language
- security reason
- versioning data is hard
- efficiency (performance) issues

**Textual formats**

Json, XML, and CSV are popular and human readable, but not very space efficient. Schema is optional for Json and XML.

**Thrift and Protocol Buffers**

Thrift and Protobuf are binary encodings, which requires schema. Both support forward and backward compatibility with the help of field tag number.

**Avro**

Avro is another binary encoding. It needs both the writer schema and reader schema in the decoding process. To get the writer schema as reader:
- large file with lots of records: indicate schema once at the begining
- database with individually written records: include a Avro schema version number + a database for history of Avro schemas
- sending records over a network: communication schema during connection setup

Avro is usful for dynamically generated schemas.

#### Modes of Dataflow
- through database
- REST and RPC
- message passing

## Distributed Data Storage

### Replication

#### Leader based replication

single leader writes to leader only, reads to all nodes

Sync vs Async vs Semi-Sync replication: when does a leader respond to a write request

Membership change

Handle node outage: Follower fail? Leader fail?

#### Replication Logs
- statement-based replication: leader logs every request and sends those to followers
- write-ahead log (WAL) shipping
- logical (row-based) log replication
- trigger-based replication

#### Multi-Leader Replication

#### Leaderless Replication

All replicas accept all write and read requests. Client is responsible for talking to multiple nodes to ensure consistency. Specifically, the sets of nodes used by the read and write operations must overlap in at least one node. Although this quorom doesn't guarantee consistency under some edge cases (e.g. sloppy quorum and hinted handoff)

Vector clocks are employed to determine concurrent writes to the same value on different replicas. Conflict resolution is required in case of concurrent write (e.g. last write wins).