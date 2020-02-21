# Chapter 1: Introduction
## Hardware Considerations & Configurations
MongoDB has storage engines that are either very dependent on RAM, or even completely in-memory execution modes for data management operations.
Some operations using RAM:
- aggregation
- index traversing
- write operations
- query engine
- connections (~1MB per)

MongoDB uses CPU generally for
- storage engine
- concurrency model
By default, MongoDB attempts to use all CPU available CPU cores to respond to incoming requests.
CPU is also used for:
- page compression
- data calculation
- aggregation framework operations
- map reduce

The recommeded RAID architecture for MongoDB is RAID Level 10.
This architecture provides good performance and safety/redundancy.
Discourage RAID 5 and 6 because performance is not good enough.
Discourage RAID 0 because it has limited availability despite high performance.

# Chapter 2: MongoDB Indexes
__Collection Scan__: when the database has to look at every document.
As the collection grows, then it is less performant.
O(n) and linear running time, meaning the running time is linearly proportional the number of documents.

Index keys are stored in order.
So MongoDB does not have to look at every single index key.
Indexes reduce the number of documents examined to satisfy a query.
Indexes _can_ decrease write, update, and delete performance.
MongoDB uses a B-Tree to store indexes.

_id is automatically indexed.

Index Overhead. Each additional index decreases write speed because it might cause the B-Tree to rebalance or the indexes might need to be updated.

## How Data is Stored on Disk
The way MongoDB stores data differs based on the storage engine used:
- MMAPv1
- Wired Tiger
- Other

For each collection/index, Wired Tiger creates an individual file in the dbPath.
- _mdb_catalog.wt: contains the catalog of all collections and indexes

The MongoDB files do not have to be a flat structure.
When launching mongod you can add the `--directoryperdb` arg.
Following that with the `--wiredTigerDirectoryForIndexes` arg will create a collection and index folder inside the db folder.
This can be beneficial for performance by enabling I/O parallilization if you have multiple disks such as a Data disk and an Index disk.

MongoDB also offers compression for storing data on disk.
Impacts performance by performing smaller I/O operations at the cost of CPU cycles.

The Journal file acts as a safeguard against corruption.
Includes individual write operations.
You can force data to be synced to the Journal before ack'ing a write with `{writeConcern: {w:whatever, j: true}}`.

## Single Field Indexes
- keys from only one field
- can find a single value for the indexed field
- can find a range of values
- can use dot notation to index fields in subdocuments
- can be used to find several distinct values in a single query

`db.<collection>.createIndex({'ssn': 1})` creates an ascending index.

Run a query appending `.explain("executionStats")`.
We want to see "winningPlan.inputStage.stage" be "IXSCAN".
"COLLSCAN" is bad, means no index was used and all documents had to be examined.

Never index on a field pointing to a subdocument.
If you do, then you have to query using an entire subdocument in the predicate.

## Querying on Compound Indexes
An index on two or more fields.
In MongoDB, a compound index is an ordered list; it is not two-dimensional.
The order of the fields in your index matters.

An __index prefix__ is a _continuous_ subset of a compound index moving from left to right.
For example, given a compound index on item, location, and stock, then the index prefixes are:
- item
- item, location