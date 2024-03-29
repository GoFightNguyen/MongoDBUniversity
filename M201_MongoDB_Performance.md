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
A query can use an index scan for both filtering and sorting _if the query includes equality conditions_ on the prefix keys that precede the sort keys.
If using the previous example, index scans would still occur if `find({item: 'x', location:'NY'}).sort({stock: 12})`.
In other words, we can filter _and_ sort our queries by splitting up our index prefix between the query and sort predicates.

We can also sort with an index if our sort predicate inverts our index keys or their prefixes (but everything included has to be inverted).

## Multikey Indexes
Indexing on an array is called a multikey index.
It is called this because for each entry in the array, an index key is created.
With multikey indexes we can index on more than scalar values, we can also use fields in a nested document (think an array of documents and using a field in those documents).

For each index document, we can have at most 1 index field whose value is an array.

Be careful that the arrays being indexed will not grow too large and thus, potentially causing the index not to fit in memory.

Multikey indexes do not support covered queries.
Covered queries prevent the reading of documents (?).

## Partial Indexes
This is when you index a subset of your documents.
Might do this to:
- lower storage requirements
- reduce the performance cost of creating/maintaining indexes

```sh
db.restaurants.createIndex(
    { 'address.city': 1, cuisine: 1},
    {partialFilterExpression: { 'stars': {$gte: 3.5}}}
)
```

Sparse indexes are a special case of partial indexes.
`db.restaurants.createIndex({stars: 1}, {sparse: true})`.
A sparse index only indexes documents where the field actually exists, instead of creating an index key with a null value.
Could also do the previous like `db.restaurants.createIndex({stars: 1}, {partialFilterExpression: {'stars': {$exists: true}}})`.
If taking the partial approach, you can also use fields you are not indexing on; for example `db.restaurants.createIndex({stars: 1}, {partialFilterExpression: {'cuisine': {$exists: true}}})`.

For a query to use a partial index, the query must be guaranteed to match a subset of the documents specified by the filter expression.

Restrictions:
- cannot specify both the partialFilterExpression and the sparse options
- _id indexes cannot be partial indexes
- shard key indexes cannot be partial indexes.

## Text Indexes
Create these by specifying "text" instead of ascending/descending.
`db.products.createIndex({productName: 'text'})`.
This allows us to leverage MongoDB's full text search capabilities while avoiding collection scans.
MongoDB creates text indexes by creating an index key for every unique word in the string.
*** Unicode treats both spaces and hyphens as text delimiters".

Text indexes are case-insensitive by default; each index key will be lowercase.
Text queries _logically or_ each individual word.

Beware if the strings are large:
- more keys to examine
- increased index size
- increased time to build index
- decreased write performance

`db.textExample.find({$text:{$serach: "MongoDB best"}}, {score: {$meta: "textScore"}})`.
$text assigns a score to each document based on its relevance to the query.
We can now sort using `sort({score:{$meta:'textScore'}})` to get the documents that best match at the top of our results.

## Collations
Collations allow users to specify language-specific rules for string comparison.
A collation is defined in MongoDB by the following options:
- locale: string determining ICU supported locale
- caseLevel: bool
- caseFirst: string
- strength: int
- numericOrdering: bool
- alternate: string
- maxVariable: string
- backwards: bool
The presenter only discussed locale.

Collations can be defined at multiple levels:
- collection: collation is applied to all queries and indexes on the collection
  - `db.createCollection("foreign_text", {collation: {locale: "pt"}})`
- specific requests like queries and aggregations
  - `db.foreign_text.find({predicate}).collation({locale:'it'})`
- create indexes with a collation overriding either the default one or collection-level one
  - `db.foreign_text.createIndex({name:1}, {collation: {locale: 'it'}})`
  - to enabling using this index on a query, the query must specify the same collation

Although collations provide correctness for specific locales, it provides marginal performance impact.
Another benefit is supporting case-insensitive indexes (set `strength` to 1).

# Chapter 3: Index Operations
## Building Indexes
MongoDB allows creating index in the foreground or background.
- Foreground indexes are fast to create, but will block all incoming operations to the db containing the collection.
- Background indexes are slower to create, but do not block operations.
  - `db.<collection>.createIndex({'cuisine':1, 'name':1, 'address.zipcode':1}, {'background':true})`
  - check the status using `db.currentOp()` or
  ```sh
  db.currentOp(
    {
      $or: [
        {op: "command", "query.createIndexes": {$exists: true}},
        {op: "insert", ns:/\.system\.indexes\b/}
      ]
    }
  )
  ```

## Query Plans
A query plan is a series of stages feeding into one another.
For a given query, there can be multiple query plans depending on the indexes.

MongoDB chooses the query plan:
- finds the candidate indexes
- creates candidate plans based on the candidate indexes
- MongoDB evaluates each candidate plan by executing it for a trial period
  - MongoDB has an empirical query planner
- The plan performing "best" is chosen

MongoDB caches which plans should be used for a given query shape.
This is so plans do not need to be generated and compared against each other every time a query is executed.

## Understanding Explain
Using explain on a query is the best way to understand what happened when it was executed: `db.people.find({<predicate>}).explain()`

It can also tell you what would happen without the query being executed:
```sh
# create an explainable object
# same as exp = db.people.explain("queryPlanner")
exp = db.people.explain()

exp.find({<predicate>})
exp.find({<differentPredicate>})
```

When using `explain`, the default is "queryPlanner", which does not execute the query.
If you pass it "executionStats" or "allPlansExecution", then query will execute.

Explain can tell us:
- is the query using the index we expected
- is the query using an index to provide a sort
- is the query using an index to provide the projection
- how selective is the index
- which part of the plan is the most expensive

## Forcing Indexes with Hint
`hint()` forces a query to use a specified index, thereby overriding MOngoDB's default index selection.
`hint` can take the index shape as an argument: `db.<collection>.find({<predicate>}).hint({name: 1, zipcode: 1})`.
`hint` can also take the index name: `db.<collection>.find({<predicate>}).hint('name_1_zipcode_1')`.

Use this only if you know you must.

## Resource Allocation for Indexes
To help determine how much of an index(es) is in memory, you can start with ` db.<collection>.stats({indexDetails:true})` and from there drill down.

*** Rule of thumb: always have enough memory to allocate indexes
Some exceptions:
- occassional reports such as for a BI tool
  - to mitigate, we can run the queries against the secondary nodes and have the indexes for the report only on those nodes
- right-end-side index increments (monotonically increasing)
  - ex: counters, dates
  - it is likely that an index will become unbalanced and grow only to the right
  - if you only need to query on the most recent, then you only to support the right side of the B-Tree index in memory

Indexes are not required to be entirely placed in RAM, however performance will suffer by constant disk access to retreive index information.

## Basic Benchmarking
Types of performance benchmarking:
- low level
  - file I/O performance
  - scheduler performance
  - memory allocation and transfer speed
  - thread
  - database server
  - ...
- distributed systems
  - linearization
  - serialization
  - fault tolerance

Benchmarking Anti-Patterns:
- database swap replace (comparing MongoDB to a relational database without changing the schema)
- using mongo shell for write and read requests
- using mongoimport to test write operations
- local laptop to run tests
- using default MongoDB parameters

# Chapter 4: CRUD Optimization
Equality, sort, range: when building an index:
- the beginning should start with equality conditions
- followed by sort conditions
- and finally range conditions
When creating indexes, ensure the most selective fields are first.

## Covered Queries
Covered queries are a highly performant way to service the queries to the database.
They are so fast because they are completely satisfied by index keys.
This means 0 documents have to be examined.
To do this, the query _must_ project to only those fields in the index.
It will not work if your projection omits fields in the index or you include fields not in the index.

In other words, an index covers a query only when both all fields in the query are a part of the index and all the fields returned are in the same index.
This implies you need to filter out _id.

You cannot cover a query if:
- any of the indexed fields are arrays
- any of the indexed fields are embedded documents
- when run against a mongos if the index does not contain the shard key

## Regex Performance
Even if you index a field, if you then query against it using regex then MongoDB must evaluate every index key.
You can help this by using something like `username:/^kirby/` because this will ignore index keys not starting with kirby.
This optimization can only work if you are using a starts-with regex.

## Insert Performance
Three parts to a writeConcern:
- w: number of nodes to write to before MongoDB returns ack
- j: whether or not to wait for the write to go to the journal
- wtimeout: how long, in ms, to wait for the write to timeout
  - even if a timeout occurs, the write can still happen

## Data Type Implications
It is possible to have the same field with different data types across your documents.
There are implications though:
- query matching: you will only find matches where the data type is the same as in the predicate
- sorting: MongoDB will first group by data type, and then sort within that type
- index: the same type of ordering mentioned in sorting applies to indexing

## Aggregation Performance
There are two high-level categories of aggregation queries:
- "Realtime"
  - provide data for apps
  - query perf is more important
- Batch
  - provide data for analytics
  - query perf is less important
We will be mostly focusing on realtime.

With an aggregation query we form a pipeline of different aggregation operators.
Some of the operators can use indexes while others cannot.
Once a stage in the pipeline that is unable to use indexes, then all the following stages cannot use an index either.
The query optimizer does attempt to move stages in order to utilize indexes.
Transforming data in a pipeline stage also prevents using indexes in stages that follow.

We can also use explain on aggregate pipelines `db.<collection>.aggregate([<operators>], {explain: true})`.

`$match`, `$sort`, `$limit` can use indexes, so we want this at the front of the pipeline.
Using limit right before sort is a top-k sorting algorithm.

Memory Constraints:
- results are subject to 16MB document limit
  - use $limit and $project
- 100 MB of RAM per stage, so use indexes
  - if you must exceed this memory constraint, use `{allowDiskUse: true}`
    - does not work with $graphLookup

# Chapter 5: Performance on Clusters
In MongoDB, a distributed system can be a Replica Cluster/Set or a Shard Cluster.

Before Sharding:
- sharding is a horizontal scaling solution
- have we reached the limits of our vertical scaling?
- understand how your data grows and how it is accessed
- sharding works by defining key based ranges - our shard key
- important to get a good shard key

There's a lot of info about reading/writing with shards, but I already noted that in M103.

## Reading from Secondaries
When reading data, there is always an associated read preference which defaults to primary `db.<collection>.find().readPref('primary')`.
There is also. These are discussed in more detail in M103.:
- primaryPreferred
- secondary
- secondaryPreferred
- nearest

Reading from a secondary is good for:
- offloading work (like doing analytics)
- local reads
Reading from a secondary is bad for:
- general
- providing extra capacity for reads
- reading from shards

*** Special Note: As of MongoDB 3.6, because of changes to both logic in chunk migrations and read guarantees, it is now safe to read from secondaries in a shard as long as the appropriate read concern is set.

## Replica Sets with Differing Indexes
In general, do not have different indexes on secondaries than on primaries.
Here are a few, specific examples where it might make sense:
- specific analytics performed against secondary nodes
- reporting on delayed consistency data
- text search

Secondary node considerations:
- do NOT allow this secondary to become a primary
  - priority: 0
  - hidden
  - delayed secondary

## Aggregation Pipeline on a Sharded Cluster
In general, shard merging happens on a random shard.
But in certain cases, it will occur only on the primary shard:
- $out
- $facet
- $lookup
- $graphLookup