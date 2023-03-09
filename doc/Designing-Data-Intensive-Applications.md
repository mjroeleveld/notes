![](./assets/Designing.png)
# Designing Data Intensive Applications book summary

- Percentiles (p95, p99, p999) are used for performance monitoring. E.g. if p95 is 1 second, 95% of requests take less then 1 second and 5% more
- High percentiles (tail latencies) are important because they directly affect users’ experience (e.g. customers with most data) 
- SQL lays out data in rows as a collection of tuples.
- Document databases are good for schema flexibility, data locality (faster large updates to big documents). Not so good for many to many relationships. 
- Declarative languages (CSS, XPath, etc) enable optimisation because implementation details are hidden from the language
- SQL queries are declarative; SQL query optimiser decides in which order to run things and which indexes to use.

## Storage and retrieval
- A hash index is an in-memory log-based key-value storage method where keys refer to byte offset of the value. Updates are appended to the segment
- Compaction (throwing away old duplicates) makes segments smaller. These can then be merged (in a background thread) and old segments thrown away.
- Log-based data structures benefits:
    - Sequential write operations are faster than random operations, especially on magnetic disks (but also on SSDs).
    - Concurrency and crash recovery are faster
    - Fragmentation because of compaction
- Range queries inefficient with nonsorted log-based storage

- Sorted String Tables (SSTables) are like hash indexes but sorted and file-based. Segments are merged using a mergesort approach, where the first string in each segment is compared. A sparse in-memory index keeps track of the offsets. A sorted tree is kept in memory and written to a segment after a certain threshold. Lookups are performed first on the first segment, then second, etc. A merging and compaction background process runs to combine segment files and discard duplicate values. No segment is ever updated, only new segments are written to.
  ￼
- Storage engines that use this merging and compacting are often called Log-Structured Merge-Tree (LSM-Tree). ElasticSearch and Solr use this method for its term dictionary (where terms map to document ids).

- B-trees are the most widely used indexing structure. The database is broken down into fixed-sized blocks (pages) often 4KB in size. Read/writes are performed one page at a time. A tree of addresses (ranges) point to the location of an item. A B-tree (balanced) has log (n) access complexity.
  ￼
- B-trees are also used for secondary indexes or multi-column indexes, as opposed to primary (key) indexes.
- A clustering index is an index where the data is stored along with the key. This is faster as no lookup to the data in e.g. a heap file is necessary.

- In-memory databases are faster mostly because they don’t need to encode data in a form that can be written to disk. Not because they don’t read from disk (often disk pages are already cached in memory by the operating system)

- OLAP and OLTP stand for Online Transaction Processing and Online Analytics Processing respectively. Analytics (BI) operations are often different from transaction, because transaction processing operates on single items, and analytics on batches.
- ETL (Extract-Transform-Load) refers to the process of extracting data from a OLTP, transformed and loaded in a data warehouse.
- A star schema is often used in a data warehouse, where a central table (fact table) holds all events (customer buys product) and dimension tables store entities that are referred to by events (e.g. products)
- Column-oriented storage store data not per row, but per column. A query takes the nth item from each column which is faster than retrieving an entire row. Column compression leads to further optimisation.
  ￼
- Fact tables often hold many columns (hundreds). A query typically returns only several columns. Column-oriented storage is faster for these type of queries. Also, indexes do not help much because many (all) rows need to be scanned. Disk bandwidth is often the bottleneck, versus disk seek time for OLTP.
- With column oriented storage, writes can be done in the LSM-way: appended to a sorted log and merged with column files in bulk.

## Encoding and evolution
- JSON, XML and CSV are encodings, that encode in-memory data objects such as structs, lists, arrays, etc to a self-contained sequence of bytes to be sent over the network.
- Some binary forms of JSON such as BSON or MessagePack encode JSON in a binary format, saving a bit of space which can be meaningful with terabytes of data. 
- Thrift (Facebook) and Protocol buffers (Google) and Avro (Apache) are binary encoding formats that use schemas
- Thrift and Protocol buffers encodes identifiers instead of field names, which saves considerable space compared to JSON
- Schemas can evolve because tags identify the fields. New fields can be added if a new tag is assigned.
- Avro does not encode tags or field names, which allows dynamically generated schemas. It matches fields by looking at the writer’s schema (which could be matched by a version, or hash). Schemas are stored in the data object (if large) or in e.g. a database to be able to decode data.
- Client code generation (Thrift and Protocol buffers) is useful for statically typed languages because it allows for type checking. For dynamically typed languages this doesn’t make sense to use.
- These encodings thus allow for the same flexibility as schemaless encodings like JSON but give better data guarantees and tooling

- RPC is fundamentally broken because it treats network calls as local calls which are different in many ways (predictability, timeout, retries, latency, arguments etc)
- RPC is mostly used for internal communications between services

- A message broker (or message queue) has several advantages to RPC:
    - It acts as an (overflow) buffer
    - It provides redelivery
    - Destination IP/host not necessary to know
    - Several recipients possible
    - Decouples sender and receiver
- Async message passing like this is one-way: another channel can be used for sending responses

## Replication
- Replication serves multiple purposes: high availability (tolerating crashes and network issues), lower latency, scalability (serving reads from more than a single machine)

- Most common method of replication is single-leader replication. One of the replicas is designated leader. Clients should send requests to the leader who writes new data to its local storage. It also sends data to all slaves into their replication log
- Leader-based replication can be sync (semi-sync) or async. Semi-sync is where data is written synchronously to one slave only.
- When synchronising a follower with a leader, a snapshot is restored and then pending writes in the replication log are synced.
- When the leader fails, a new leader should be appointed and clients send their request to the new leader
- Replication lag results in the issue that reads from a follower are not yet propagated from the leader. This isn’t easily solved.
- E.g. with sharding, some writes may be committed out of order, which may result in unexpected reads where write 2 may be read before write 1 
- Multi-leader replication makes sense for multi-datacenter operations, because writes can then be served from multiple locations. It is very hard to get right (auto incrementing keys, integrity constraints etc.)
- Offline clients (e.g. calendars) and realtime collaborative editing apps are basically leaders as they accept writes which need to be synced
- Conflict resolution can be either done on write (e.g. using the write with the latest timestamp) or on read (let application handle it)
- In multi-leader configurations, multiple topologies exist (how leaders sync changes). E.g. circular or all-to-all. All-to-all handles node failures better, but is prone to propagation issues.

- Leaderless replication method (used by Amazon Dynamo and Cassandra) is based on sending reads and writes to multiple nodes.
- Leaderless requires a careful tuning of the required nodes to have consistent reading after writing and not read stale values in case of unavailable nodes. However, edge cases are possible with e.g. concurrent writes.

- Multi-leader and leaderless replication can be more robust with faulty nodes, network interruptions and latency spikes but are harder to reason about and providing very weak consistency guarantees because they allow concurrent writes 
 
## Partitioning
- Partitioning and replication go often together, where multiple followers can be on the same node
- Partitioning can be done by key range, which can lead to hotspots or key hash, which is more uniformly spread but doesn’t allow efficient range queries
- One challenge of partitioning is to avoid hot spots and to achieve an evenly spread load 
- Document-partitioned secondary indices are partial indices at every partition which need all to be queried (scatter/gather)
- Term-partitioned secondary indexes are global secondary indices, partitioned itself (e.g. colour a-m, m-z). It needs to be partitioned or it defeats the purpose of partitioning. This could lead to more efficient reads (e.g. one partition only) but more complicated writes (multiple index partitions) 
- One way to rebalance partitions is by having a fixed number of partitions, larger than the number of nodes. When rebalancing, some partitions can be reassigned to other nodes. Choosing the number of partitions is difficult if the size of the dataset is highly varying. This method is often used with hash partitioning
- Dynamic partitioning is when partitions are split if growing past a certain threshold. This works better with key range partitioning (or queries will be scattered). Partitions can then be moved to other nodes
- Partitioning can be unpredictable and is often done/overseen manually

- Request routing is often done through a separate coordination service such as Zookeeper to keep track of this cluster metadata.
- Other databases such as Cassandra use a gossip protocol; any node can receive requests which can then be forwarded to the node with the partition

## Transactions
- Transactions are a way for an application to group several reads and writes together in a logical unit; either it succeeds (commit) or fails (abort, rollback). Error handling becomes much simpler because you don’t need to worry about partial failure.
- ACID atomicity describes what happens if a client wants to make several writes, but a fault occurs after some of the writes; the operation can be undone, or aborted
- ACID consistency is that the database is in a good state; constraints are not validated, schemas correct, etc. It is an application-level property, not enforced by the database
- ACID isolation is that concurrently executing transactions are isolated from each other and do not interfere. E.g. if one transaction commits several writes, then another transaction should see either all or none of those writes
- ACID durability is that the data stays in place and is not lost in case of a fault (write ahead log, RAID)
- Atomicity for single objects can be done through a log for crash recovery (WAL), isolation using a lock on each object
- Multi-object transactions are often needed, e.g. with foreign keys, or denormalised data, or secondary indexes

- Many databases provide weak isolation levels
- Read committed transaction isolation guarantees that you will only read data that has been committed (no dirty reads) and that you will only overwrite data that has been committed (no dirty writes)
- Dirty writes are prevented by obtaining a lock on objects. Dirty reads are prevented by serving the old version until the transaction has completed (no lock used)
- Snapshot isolation is a way for a transaction to read a consistent snapshot from the database. This means that a transaction sees a consistent snapshot in of the database when a long-running query (e.g. backup) is running (data doesn’t change in the meanwhile)
- Databases implementing snapshot isolation maintain multiple versions of the relevant data to serve reads
- Lost updates are situations where applications read data and update it non-atomically, while in the meanwhile another update to the data occurs. The last last update does not include the former.
- Write skew is when two transactions read the same objects and then update the same object. It can result in a lost update or dirty write. This can happen in multiple ways, e.g. when checking a constraint and then incrementing a counter (double spending).
- The effect where a write in one transaction changes the result of a search query in another transaction is called a phantom
- Serialisable isolation is usually regarded as the strongest isolation level. It guarantees that even though transactions may execute in parallel, the end result is the same as if they executed in some serial order (not in the order executed per se, just not concurrently).
- Serialisable isolation can be implemented by removing all concurrency; execute only one transaction at a time on a single thread. This is feasible because RAM is now cheap enough to store entire datasets in memory, enabling fast operations. Also, OLTP transactions are rather short and involve few reads/writes. Redis works this way.
- Stored procedures are a way to execute multiple transactions serially. Systems with single-threaded serial transaction processing do’t
- Breaking up the data in partitions, transactions can be serialisable isolated across multiple CPUs. But transactions across partitions are not serialisable
- 2-phase locking is another way to achieve serialisable isolation. With 2-phase locking a lock means reads lock reads and writes and writes lock reads and writes. With snapshot isolation, in contrast, readers don’t lock writers and writers no readers. This limits throughput.
- 2PL has to also lock all objects related to the query to prevent write skew and phantoms. Databases implement index-range locking to make this faster; instead of matching locks on table items, index entries are locked.
- 2PL has a big performance penalty, serialisable execution is not scalable (single CPU only).
- Serialisable snapshot isolation is promising in that it provides full serialisability while having only a small performance overhead compared to snapshot isolation.
- 2PL and serial execution are pessimistic locks, because they lock everything for in the case something may go wrong. By contrast, serialisable snapshot isolation is an optimistic concurrency control technique.
- SSI is based on snapshot isolation while detecting if anything bad happened (i.e. whether isolation was violated). If so, the transaction is aborted and has to be retried. If contention between transactions is not too high, this performs better than pessimistic concurrency control.
- SSI works by detecting stale reads (old versions of data while writes are performed) and aborting transactions that involve these (before they are committed). Also, writes that affect prior reads are detected; these transactions are notified that the data may no longer be up to date

## Problems with distributed systems
- Any distributed systems, anything that can go wrong, will go wrong. It is necessary to regard distributed systems in a pessimistic way.
- Networks are inherently unreliable. Request or responses may go lost, packets may be queuing up, nodes may fail or stop responding.
- It’s hard to determine how long timeouts should be because delays may be long in high percentiles
- Networks are unreliable because they are packet-switched to utilise their full bandwidth. Delays and unpredictability are therefore inherent.
- Clocks are unreliable; clocks synced with NTP-servers are imprecise because of network latency differences. The hardware clock component in a PC (quartz clock) drifts. In virtual machines, the hardware clock is virtualised.
- Because clocks are unreliable, timestamps in distributed systems cannot be compared with certainty (e.g. to determine the order writes come in).
- And thus nodes must assume that their execution can be stopped at any point in time. E.g. because of other threads, garbage collection, VM environments, OS context switching, IO operations, memory paging, etc.
- In many distributed systems, the truth (e.g. who is leader) is defined by the majority (quorum) to account for nodes that are dead or unresponsive.
- A thread that thinks it obtained a lock, may have been suspended in the meantime. A fencing token can be used that is bound to the lock and invalidates the request.
- Byzantine faults are where nodes may lie, i.e. send arbitrary faulty or corrupted responses.

## Consistency and Consensus
- Linearizability is when a system provides the illusion of reading from a single copy of the database. Thus, guaranteeing that the value read is the most recent, up-to-date value and doesn’t come from a stale cache or replica. Once a new value has been written or read, all subsequent reads see the value that was written until it is overwritten again.
- Linearizability is different from serializibility in that it doesn’t guarantee that transactions are processed in some serial order (so that transactions do not interfere); it is a recency guarantee on reads and writes of a register. Serialisable snapshot isolation is not linearisable; older versions of an object can be served.
- The performance penalty of implementing linearizability is the maximum of network latencies at a minimum (all replicas need to be committed synchronously)
- Use cases for linearizability are locking (e.g. leader election) and constraints (e.g. unique user id) 
- CAP theorem stands for Consistency, Availability, Partition tolerance: pick 2 out of 3. Consistency: all nodes see the same data, availability: all requests get non-erroneous responses, partition tolerance: system functions with crashing of multiple nodes.
- CAP theorem is problematic because network partitions are a kind of fault; it’s not something that you have a choice about. A better formulation is: either consistent or available when partitioned. Also, CAP doesn’t address many other issues like network delays, etc. 
- Total order broadcast is described as a protocol for exchanging messages between nodes. These messages should be defined reliably (no lost messages) and in order (messages can be compared; are in total order) and only once (not multiple times). There is no ambiguity in which message came first such as with a timestamp.
- ZooKeeper and etcd implement total order broadcast. Total order broadcast is also used for database replication (syncing to replicas).

- 2-phase commit is an algorithm for achieving atomic transaction commit across multiple nodes. I.e. to ensure that either all nodes commit or abort. It uses a coordinator that sends a commit request after all participants replied yes to the preparation phase (in which constraints etc are checked).
- However, distributed transactions often carry a heavy performance penalty (e.g. in MySQL can be 10 times slower).
- A protocol like this can be implemented using a message queue. Different systems only ack a message in the queue after it commits the transaction. If the transaction fails, the message in the queue is not acked and can be retried until it succeeds.
- XA (extended architecture) is a standard for implementing 2PC supported by traditional RDMSs. An application server uses a local client to coordinate with an external transaction manager to perform the distributed commit.

- Consensus is formalised as follows: one or more nodes may propose values, and the consensus algorithm decides on one of those values (e.g. two customers booking the same seat).
- A consensus algorithm must handle termination. It is assumed nodes crash (max. less than half of nodes) and may never come online. Consensus should then still be possible. 2PC does not meet this requirement.
- A total order broadcast is equivalent to repeated rounds of consensus (each consensus decision corresponds to one message delivery).
- Most consensus algorithm use a leader. This leader is not unique but is unique within an epoch number. When a node is dead, a vote is started among the nodes to elect a new leader. The election is assigned a new, incremented epoch number. If there is a conflict between two leaders in two different epochs, the leader with the higher epoch prevails. The majority must approve the vote to achieve consensus.
- Consensus algorithms have bad performance and are therefore only used for critical applications such as leader election.

- ZooKeeper and etc are distributed key-value stores or coordination/configuration services. They provide outsourced consensus, failure detection and membership service. They hold small amounts of data in memory this is replicated across nodes using a total order broadcasting algorithm.
- These services also provide: linearisable atomic operations (compare-and-set), total ordering of operations (fencing token for threads that are paused), failure detection (locks held by a dead node are released) and change notifications (new nodes/dead nodes).
- These services can be used for leader election, partition assignation and membership services (e.g. air traffic control; reach consensus about which nodes are dead or alive), uniqueness constraints (e.g. unique user id) and atomic transaction commits (distributed commits)

## Batch processing
- UNIX tools (awk, sort, uniq, less etc.) are a great way to process data that fit in memory. They provide a uniform interface (with file descriptors) and can be easily composed through their in and output. However, they cannot be split across machines. 
- MapReduce is a way to do brute-force, distributed processing. Hadoop is an implementation of MapReduce. The map function maps each input blocks to key/value pairs, the pairs are then sorted by key and then the reducer iterates over the pairs. When a mapper finished reading its input and writing its sorted output files, the reducers fetch the output from the mappers. This is called the shuffle. MapReduce gives engineers the ability to run their own code over large datasets.
- Hadoop uses HDFS, a shared nothing storage system that requires no special hardware, only computers connected by a conventional datacenter network. It works by having one central service that abstracts the file systems of the connected nodes and keeps track of which file blocks are stored where. Hadoop can be compared to a distributed version of Unix, where HDFS is the filesystem and MapReduce a quirky implementation of a Unix process (with in- and output).
- MapReduce jobs are often chained, forming a workflow. Tools as Airflow can be used for this.
- Data joins (e.g. a user for a specific event in a log) can be done by having a local copy of the data, and having a mapper that sorts the data in way that the to be joined data is sorted next to the data that needs it. Both items have the same key and are sent to the same reducer where the joined data can be processed. This is called sort-merge join and it can be described as the mappers emitting data to a reducer. It separates the application logic from the network architecture.
- Another way of performing joins is when (a part of) the index can be fit into memory of a mapper. The mapper can than perform a lookup in a hash table and merge it with the input. This is called a map-side join.
- Building search indexes are a common use case of MapReduce workflows. Another use case is building a key-value store; using a mapper to extract a key and then sorting by that key. Key-value stores can often load these data files while serving an old copy of the data.
- MapReduce jobs should be immutable; you should be able to rerun them without side effects.
- Sushi principle: raw data is better. Hadoop allows for storing large amounts of data in any format, to be processed later on. It’s better to have too much data than not enough.

- MapReduce has some drawbacks; a job can only start when preceding jobs have completed. Also, mappers are often redundant, reading back the same file that was written by a reducer. Also, storing intermediate state is often overkill for temporary data (especially when replicated over HDFS).
- Dataflow engines such as Spark address these problems. They handle an entire workflow as one job, rather than breaking it up into independent subjobs, calling an operator to process a record. They work more like Unix pipes. They are often built on top of MapReduce.
- This model has several advantages to MapReduce: work such as mapping and sorting need only be performed in places where it’s needed. Also, the scheduler has an overview of the workflow and can thus make locality optimisations. State can often be kept on a single machine, instead of writing to HDFS. Also, operators can start as soon as their input is ready, no need to wait for the entire preceding stage to finish. Lastly, existing containers/nodes can be reused instead of spawning new ones.
- Dataflow engines provide higher level abstractions and declarative languages which is why they can optimise operators and scheduling.

- MapReduce is inefficient for graph models. Here, the Pregel model is better used.
- It’s important for MapReduce and data flow engines to pick the right join algorithm, as it makes a big difference in performance.
- MapReduce frequently writes to disk, which makes it fault-tolerant. Dataflow have less intermediate state and need to recompute more if a node fails.

## Stream processing
- Streams is very much like batch processing, but done continuously on unbounded streams rather than fixed-sized input.
- PubSub is a model for processing streams. It implements mechanisms for buffering messages and tolerating faults.
- When multiple consumers read messages in the same topic, two patterns are used: load balancing (one consumer reads the messages), or fan-out (every consumer gets the message)
- Acks make sure that the consumer processed the message 
- AMQP/JMS-style message brokers are message brokers that assign individual messages to consumers. Consumers ack messages when processed. Messages are deleted once acked. This form is appropriate as an async form of RPC (task queue, history and order are not important). 
- Log-based message brokers keep a log of sent messages (similar to replication log). Consumers read from this log. In order to scale to higher throughput than a single disk, the log can be partitioned. Consumers keep track of the offset they read, for each partition. Load balancing can be achieved by making each node process one partition. If messages take long to process, this may stall the queue. The producer also records the offset, to know which messages to resend to a new node when some node fails.
- In situations where throughput is important, where each message is fast to process and message ordering is important, the log-based approach works very well.
- Log-based message brokers are bound by disk space; old messages are dropping old segments when the limit is in sight
- In contrast to other message brokers, with log-based message brokers, consuming messages is more like reading from a file. It is a read-only operation that does not affect the log. The only side effect is the consumer’s offset moving forward. This allows for more experimentation and easier recovery from errors and bugs.

- Change data capture (CDC) is the process of observing all data changes written to a database and extracting them in a form in which they can be replicated to other systems, such as a search index to match the data in the database.
- When using log compaction (deleting old logs), the CDCs topics can contain the entire database and new derived data systems can be build from offset 0.
- Databases that implement this as first-class citizens (Firebase, CouchDB, RethinkDB) represent an output stream in the relational data model as a table into which transactions can insert tuples. The stream then consists of the log of tuples that committed transactions have written to this special table. External consumers can asynchronously consume this log to update derived data systems. 
- Event sourcing is a higher level technique that could be implemented using streams.
- The advantage of storing a changelog durably, is that the current state (database) can be derived from it and that historical events are not erased. Quote: “Transaction logs record all the changes made to the database. Appends are the only way to change the log. From this perspective, the contents of the database hold a caching of the latest record values in the logs. The truth is the log. The database is a cache of a subset of the log. That cached subset happens to be the latest value of each record and index value from the log. “
- By separating mutable state from the immutable event log, you can derive different read-oriented representations from the same log of events.
- Storing data is straightforward if you don’t have to worry about how it is going to be queried. You gain a lot of flexibility by separating the form in which the data is written and read.
- Although event sourcing is that it’s asynchronous. However, it simplifies concurrency in a way that a transaction only has to be committed to the log. Consumers can process the events in serial.

- Purposes of stream processing include searching for event patterns (anomaly detection), computing windowed aggregations (stream analytics) and keeping derived data systems up to date (materialised views).
- The issues with clock synchronisation also apply here; timestamps cannot be trusted. A combination of a timestamp on event, timestamp on send and server timestamp can be used to estimate the actual event time.
- Three types of joins exist can be distinguished: stream-stream joins (e.g. matching a click event with a search query within the same user session); stream-table joins (e.g. joining user data from a local copy of the db); table-table joins (both input streams are database changelogs, e.g. events that update a user’s timeline cache)
- Fault-tolerance can be achieved by implementing microbatching (treat events in certain window as batch), checkpointing (saving state at intervals), transactions (two-phase commit) or idempotent writes.

## The future of data systems
- Unbundling databases could be a useful way to integrate different systems with different strengths and use cases. A database already maintains e.g. secondary indices, which is essentially a materialised view. Instead of adding functionality on top of one system (e.g. PostgreSQL), databases could let specialised consumers (machine learning models, indices, statistical summaries etc.) subscribe to changes, maintaining their own view of the data.
- Observers can have their own observers by downstream consumers. You can even take this dataflow all the way through to the end-user (no need for caches), building user interfaces that dynamically update to reflect data changes and that work offline.
- This design allows for asynchronous and loosely coupled systems, increasing robustness and fault tolerance.
- Expressing dataflows as transformations (write log) helps evolve applications; if you want to change the structure of index or cache, you can rerun the new transformation code on the whole input dataset. Also, if something goes wrong you can fix the code and reprocess the data in order to recover.
- Asynchronous event processing is not timely, i.e. not linearisable, but this is often not a problem. Systems can be designed to gracefully handle these kind of faults where the fault is detected asynchronously. E.g. email a user that a booked seat had already been taken (with an async event-based booking system).
- Because many streaming processors e.g. message queues have strong integrity guarantees (order, exactly-once delivery, guaranteed delivery), unbundled systems can be designed to be fault-tolerant and eventually consistent. With dropping the time constraint, asynchronous, unbundled systems can provide strong integrity while being performant (not requiring distributed transactions). 
