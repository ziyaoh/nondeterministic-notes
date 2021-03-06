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

### Partition

Partitioning is usually combined with replication so that copies of each partition are stored on multiple nodes. Partition schema and replication schema are mostly independent from each other.

#### Partitioning of Key-Value Data

- by key range

    friendly for range scan

    distribution can easily be skewed

- by hash of key

    distribution is usually balanced

    hard to do range query

#### Partitioning and Secondary Indexes

Secondary indexes don't map neatly to partitions. Two main approaches

- document-based partitioning

    partition by primary key, with local secondary indexes managed independently by each partition

    writing is easy but querying secondary index is hard, since we have to query all partitions

- term-based partitioning

    global secondary indexes themselves are partitioned
    
    querying a secondary index is easier since we only need contact one partition now; writing is harder because secondary indexes of a single document may scatter across multiple partitions

#### Rebalancing Partitions

- fixed number of partitions

    create many more partitions than there are nodes, change parition to node assignment to do rebalancing

- dynamic partitioning

    dynamically merge or split partitions according to data size

- partitioning proportionally to nodes

    each node has fixed number of partitions

#### Request Routing (Service Discovery)

### Transactions

A transaction is a way for an application to group several reads and writes together into a logical unit.

#### ACID
- Atomicity: all or nothing
- Consistency: application invariants always be true (application dependent)
- Isolation: concurrent transactions are isolated from each other
- Durability: data being persistent once committed

### Trouble with Distrbuted Systems

Distributed Systems in the cloud computing area suffers from partial failures. Some part may fail while the rest working correctly. As opposed to the single machine scenario, where a hardware failure usually triggers the whole system shutting down.

#### Unreliable Networks

#### Unreliable Clocks

clocks on different machines represents different timing, because they could drift slowly

syncing is possible through NTP protocol, but some problems remain: precision can only be as good as network delay, which is again unreliable

clocks issues are hard to notice, and thus can accumulate errors over time

case study
- timestamps for ordering events: use logic or vector clock instead
- sync clocks for global snapshots: requires monotonically increasing transaction ID accross multiple machines; intuitive but problematic solution: timestamp from clocks
- problems when using local clocks in case process paused for a long time

#### Knowledge, Truth, and Lies

distributed systems cannot simply rely on a single node; instead they require decision from a quorum

distributed system algorithm correctness can only be proved under some idealized system models, which makes assumptions about the system that are supposed to always hold; in reality those assumptions could break as well

### Consistency and Consensus

#### Linearizability

Appear as if there's only one single data replica.
- reads reflect the latest changes (as appeared to user)
- reads are monotonic

From perspective of CAP: linearizability chooses Consistency, non-linearizability chooses Availability

#### Ordering Guarantees

Total Ordering vs Partial Ordering (e.g. Causal Ordering)

Causal consistency is the strongest possible consistency model that does not slow down due to network delays, and remains available in the face of network failures. So **CAP doesn't apply here**.

Use Vector Clock to determine causal relation

Total order broadcast provides total ordering across nodes (heartbeats in Raft)
- reliable delivery
- totally ordered delivery

Total order broadcast is equivalent to consensus.

#### Distributed Transactions and Consensus

Two-Phase Commit (2PC)

Heterogeneous distributed transactions is useful to integrate multiple different systems together. e.g. exactly once message broker

Fault-tolerant consensus

Tools like ZooKeeper, etcd are usually used as coordination service in production. e.g. consensus, operation ordering, failure detection, sharding allocation, etc.

## Derived Data (Data Processing)

Types of systems

- Services (online systems)

    wait for requests and returns responses; measured by response time

- Batch Processing Systems (offline systems)

    periodically runs a job to process large amount of input data; measured by throughput

- Stream Processing Systems (near-real-time systems)

    TODO: buid upon batch processing?

### Batch Processing

Unix Tooling Philosophy
- each tool does one thing well
- expect output to be input to another tool
    
    uniform interface, files (file descriptors in unix)

- separation of logic and wiring

    decoupling, using stdin and stdout

- transparency and experimentation

#### MapReduce and Distributed Filesystems

Like unix tools in many ways. Reads and write files on a distributed filesystem. (HDFS in Hadoop implementation)

**HDFS** (Hadoop Distributed File System) runs on cluster of machines, each running a daemon process exposing a network service that allows other nodes to access files stored on that machine. A central service **NameNode** keeps track of which file blocks are stored on which machine.

**MapReduce Job** consists of four steps

- read a set of input files, break it up into records
- custom **Mapper** function to extract any number of key-value pairs from each record
- sort the pairs by key (implicit in MapReduce framework)
- custom **Reducer** function to aggregate information from all values belong to each key

Each MapReduce job is executed in distributed fashion. Every relevant machine runs a Mapper function to reduce network load. Reducers functions are run on different set of machines (conceptually). 

Mapping between keys and Reducer tasks by done using hash of the key. Each Mapper partitions, sorts, and writes out its output by target Reducer. Reducers then fetch their corresponding data from each Mapper. This process is known as **shuffle**. Each Reducer function eventually process values from one key. 

**Join**

- Reduce Side Join and Grouping

    sort-merge join: multiple sets of mappers sort their records by the same key (some ID for example), and same key goes to the same reducer, reducer then joins the records with the same key together

    GroupBy could be implemented with a sort-merge join intuitively

- Map Side Join

    - broadcaset hash join

        if one input dataset is small enough to fit in a mapper function memory (or disk)

    - partitioned hash join

        if all input datasets are partitioned in the same way, so each mapper could only work on one partition of both datasets

    - map-side merge join

        if all input datasets are partitioned and sorted in the same way


**Handling Skews (especially for Joining)**

- sample job to determine hot keys, then randomized reducer for those hot keys; in case of joining, some records for those keys need to be replicated to all candidate reducer

- predefined hot keys, then shard them

- predefined hot keys, and related records are stored seperately; use a map-side joining to join those hot keys

#### Beyond MapReduce

MapReduce model is generic, but slower in some cases

**Materialization of Intermediate State**

A workflow consists of multiple MapReduce jobs, each of which writes its output to Distributed Filesystem for easy and intuitive recovery purpose. This is called materialization of intermediate state. But a lot of times this is an overkill. Removing this step helps

- save the cost of writing data to Distributed Filesystem
- save redundant mapper work if previous reducer output the data in the same partitions and orders
- allow stream-ish processing because downstream jobs don't have to wait for previous jobs to completely finish

Dataflow Engines, s.t. **Spark**, Tez, and Flink, are some solutions for the problem above. Their workflows consist of a series of *operators* user defined functions. The intermediate states are kept in memory or written to local disk, which is faster than HDFS.

For Fault Tolerance, dataflow engines recomputes the data if some operators fail. This requires the knowledge of how a given piece of data was computed —- which input partitions it used, and which operators were applied to it. Spark uses resilient distributed dataset (RDD) to track the ancestry of data. To this end, it is best to make operators deterministic.

**Graph Model Dataset Processing**

Iterative, while style execution. Every vertex process its own partition of data for each iteration. Output could be transferred to neighboring vertices as input the next iteration.

### Stream Processing

#### Message Systems

**Direct Messaging from Producer to Consumer**

A message channel like TCP or Unix pipe. Assumes that both producer and consumers are constantly online. May lose message otherwise.

**Message Broker (Message Queue)**

Producer - Message Broker - Consumer

In case of multiple Consumers: *load balancing* vs *fan-out*

- traditional message broker

    store messages temporarily (mostly in memory, could be written to disk in case of message queue backlog), delete messages once the consumer confirms the consumption

    RabbitMQ, Azure Service Bus

- log-based message broker

    messages are kept appended to log (on disk) by producer, each of which are assigned a monotonically increasing sequence number; consumers read through the log sequentially (tracked by consumer offset); messages are retained for a while and deleted after the circular buffer is full

    log can be partitioned for scaling purpose

    Apache Kafka, Amazon Kinesis Streams, Twtter's DistributedLog

