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