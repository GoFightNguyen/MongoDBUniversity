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
Another benefit is supporting case-insensitive indexes (set `strenth` to 1).