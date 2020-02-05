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