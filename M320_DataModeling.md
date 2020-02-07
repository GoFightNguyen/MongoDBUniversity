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

