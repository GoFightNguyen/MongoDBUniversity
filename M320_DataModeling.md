# Chapter 1: Introduction to Data Modeling
## Data Modeling in MongoDB
MongoDB has a very flexible data model, but is not schemaless.
All data has some form of structure and therefore, some schema.

The following can help you extract a good model even before writing a full application:
- usage pattern
- how you access your data
- which queries are critical to your application
- ratios between reads and writes

When you have an idea what documents should look like and which data types those fields should have, you can enforce those rules using Document Validation.

Working Set: the total body of data the application uses in the course of normal operations.

## The Data Modeling Methodology
Workload
- size data
- quantify ops
- qualify ops

Schema
- queries
- indexes
- data sizing
- operations
- assumptions
- collections
- fields
- shapes

Relationships
- identify
- quantify
- embed or link

A 1-to-1 relationship should probably be in the same collection.
A 1-to-n relationship should probably be in different collections.

Patterns
- recognize
- apply

## Model for Simplicity or Performance
| Goal | Simplicity | Simplicity & Performance | Performance |
| --- | --- | --- | --- |
| Identify Workload | Most frequent op | Data Size, Quantify Ops | Data Size, Quantify Ops, Qualify Ops |
| Entities and Relationship | mostly embed | embed & link | embed & link |
| Transformation Patterns | Pattern A | Patterns A,B | Patterns A,B,C |

# Chapter 2: Relationships
## Type and Cardinality
One way to discuss cardinality is [min,likely,max].

## One-to-Many Representations
In general, when embedding, embed in the side most often queried.

- Embed in the "one" side
  - the documents from the "many" side are embedded
  - most common representation for simple applications or when there are few documents to embed
  - need to process main object and the N related documents together (think a movie and its top 20 reviews)
  - indexing is done on the array
- Embed in the "many" side
  - less often used
  - useful if "many" side is queried more ofen than the "one" side
  - embbeded object is duplicated (this can be a downside)
    - duplication may be preferable for dynamic objects (think of orders and the address it is shipped to)
- reference in the "one" side
  - array of references
  - allows for large documents and a high count of these
  - list of references available when retrieving the main object
  - cascade deletes are not supported by MongoDB and must be managed by the application
- reference in the "many" side
  - preferred representation using references
  - allows for large documents and a high count of these
  - no need to manage the references on the "one" side

## Many-to-Many Relationship
- embed in the main side
  - the documents from the less queried side are embedded
  - results in duplication
  - keep "source" for the embedded documents in another collection
  - indexing is done on the array
- reference in the main side
  - array of references to the documents of the other collection
  - references readily available upon first query on the "main" collection
- references in the secondary side
  - array of references to the documents of the other collection
  - needs a secondary query to get more information

## One-to-One
- embed fields at the same level (same document)
- embed using subdocuments (preferred representation)
  - documents are clearer
  - preserved simplicity
- reference
  - adds complexity, so only do for schema optimization scenarios
  - useful when you do not care about the other info as often

## One-to-Zillions
- Reference in the many/zillions side
  - quantify the relationship to understand the maximum N value

# Chapter 3: Patterns (Part 1)
Patterns are small, reusable units of knowledge to apply.
We will look at patterns for Data Modeling and Schema Design.

## Handling Duplication, Staleness and Integrity
Applying patterns may lead to:
1. duplication - duplicating data across documents
2. data staleness - accepting staleness in some pieces of data
3. data integrity issues - writing extra application side logic to ensure referential integrity

## Attribute Pattern
** The Wildcard Index functionality can replace some use cases of the Attribute Pattern **

Problem:
- lots of similar fields
- want to search across many fields at once
- fields present in only a small subset of documents

Solution:
- transform those fields from field/value form to an array of subdocuments where each subdocument is a key/value pair

Use case examples:
- characteristics of a product
- set of fields all having the same value type

Pros & Cons:
- easier to index
  - Create an index on the keys
  - Create an index on the values
- allow for non-deterministic field names
- ability to qualify the relationship of the original field and value

## Extended Reference Pattern
Likely seen in a many-to-one relationship.

Problem:
- too many repetitive joins

Solution:
- identify fields on the lookup side
- bring those fields into the main object
- choose fields that do not change often
- choose only the fields you need to avoid joins ($lookup, $graphLookup)

Use case examples:
- catalog
- mobile apps
- real-time analytics

Pros & Cons:
- faster reads
- reduce number of joins and lookups
- may introduce lots of duplication if extended reference contains fields that mutate a lot

## Subset Pattern
The following could help when the working set is too large:
- add RAM
- scale with sharding
- reduce the size of the working set (that's what this pattern is about)

Problem:
- working set is too big
- lot of pages are evicted from memory
- a large part of the documents are rarely needed

Solution:
- split the collection into 2 collections
  - most used part of documents
  - least used part of documents
- duplicate part of a 1-to-many or many-to-many relationship that is often used in the most used side

Use case examples:
- list of reviews for a product
- list of comments on an article
- list of actors in a movie

Pros and Cons:
- smaller working set, as often used documents are smaller
- shorter disk access for bringing in additional documents from the most used collection
- more round trips to the server
- a little more space used on disk

# Chapter 4: Patterns (Part 2)

## Computed Pattern
Mathematical Operations.
Fan Out Operations - many tasks to represent one logical task.
Roll Up Operations - merging data together, looking at data at a high-level.

Problem:
- costly computation or manipulation of data
  - overuse of resources, like CPU
- executed frequently on the same data, producing the same result
- need to reduce latency for read operations

Solution:
- perform the operation and store the result in the appropriate document and collection
- if need to redo the operations, keep the source of them

Use case examples:
- IoT
- Event Sourcing
- Time Series Data
- Frequent Aggregation Framework queries

Pros & Cons:
- read queries are faster
- saving on resources like CPU and Disk
- may be difficult to identity the need
- avoid applying or overusing it unless needed

## Bucket Pattern
Problem:
- avoiding too many documents, or too big documents
- a 1-to-many relationship that can't be embedded

Solution:
- define the optimal amount of information to group together
- create arrays to store the information in the main object
- it is basically an embedded 1-to-many relationship, where you get n documents each having an average of many/n subdocuments

Use Case Examples:
- IoT
- Data Warehouse
- lots of info associated to one object

Pros & Cons:
- good balance between number of data access and size of data returned
- makes data more manageable
- easy to prune data
- can lead to poor query results if not designed correctly
- difficult to sort across buckets
- ad hoc queries may be more complex, again across buckets
- works best when the "complexity" (the storage pattern) is hidden through the application code
- not great if you need to do random/specific insertions/deletions in buckets

## Schema Versioning
Each document indicates its schema version.
Problem:
- avoid downtime while doing schema upgrades
- upgrading all documents can take hours, days or even weeks when dealing with big data
- don't want to update all documents

Solution:
- each document gets a "schema_version" field
- application can handle all versions
- choose your strategy to migrate the documents

Use Case Examples:
- every application using a database
- system with a lot of legacy data

Pros & Cons:
- no downtime
- feel in control of the migration
- less future tech debt

## Tree Patterns
There are four patterns:
- Parent References - the document holds a reference to the parent node
- Child References - the parent documents contains an array of all immediate children
- Array of Ancestors - an ordered array storing all of a node's ancestors
- Materialized Paths - a string value describes the nodes ancestors with some value separator
  - similar to array of ancestors, but allows a single field path index on the ancestors

In the following table, let:
- Y indicate the pattern supports the query
- ~ denote it is possible, however, it may take more work in the application code, such as the use of multiple queries

| Pattern Name | Who are the ancestors of node X? | Who reports to Y ? | Find all nodes under Z | Change all categories under N to Under P |
| --- | --- | --- | --- | --- |
| Parent References | ~ | Y | ~ | Y |
| Child References | ~ | ~ | Y | ~ |
| Array of Ancestors | Y | Y | Y | ~ |
| Materialized Paths | Y | ~ | ~ | ~ |

Problem:
- representation of hierarchical structured data
- different access patterns to navigate the tree
- provide optimized model for common operations

Solution:
- the different patterns are all listed above

Use Case Example:
- org charts
- product categories

Pros & Cons:
- Look at the table above

## Polymorphic Pattern
Problem:
- objects are more similar than different
- want to keep objects in same collection

Solution:
- field tracks the type of document or sub-document
- app has different code paths per document type, or has subclasses

Use Case Example:
- single view implementation
- product catalog
- content management

Pros & Cons:
- easier to implement
- allow to query across a single collection

## Approximation Pattern
Used to reduce the nubmer of resources needed to perform some write operations.
Uses an approximation to produce a result.
For example, an app wanting to track page views.
Instead of writing to the database for every page view, do it every 10 views.

Problem:
- data is expensive to calculate
- it does not matter if the nubmer is precise

Solution:
- fewer writes, but with higher payload (not larger payload)

Use Case Examples:
- web page counters
- any counters with tolerance to imprecision
- metric statistics

Pros & Cons:
- less writes
- less contention on documents
- statistically valid numbers
- numbers are not exact
- must be implemented in the application

## Outlier Pattern
Problem:
- few documents would drive the solution
- impact would be negative on the majority of queries

Solution:
- implementation that works for the majority
- field identifies outliers as exception
- outliers are handled differently on the app side

Use Case Examples:
- social networks
- popularity

Pros & Cons:
- optimized solution for most use cases
- differences handled application side
- difficult for aggregation or ad hoc queries

## Summary of all patterns
| | Catalog | Content Management | IoT | Mobile | Personalization | Real-Time Analytics | Single View |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Approximation | Y | | Y | Y | | Y | |
| Attribute | Y | Y | | | | | Y |
| Bucket | | | Y | | | Y | |
| Computed | Y | | Y | Y | Y | Y | Y |
| Extended Reference | Y | | | Y | | Y | |
| Outlier | | | Y | Y | Y | | |
| Preallocated | | | Y | | | Y | |
| Polymorphic | Y | Y | | Y | | | Y |
| Schema Versioning | Y | Y | Y | Y | Y | Y | Y |
| Subset | Y | Y | | Y | Y | | |
| Tree and Graph | Y | Y | | | | | |