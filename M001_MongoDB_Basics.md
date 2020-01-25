# Chapter 1

## Databases, Collections, and Documents
In MongoDB, a database contains collections.
Collections store individual records called documents.

Authorization settings exist at the database or collection level, but not for individual documents.

Each database and collection combination define a namespace.
You typically reference a specific collection by `databaseName.collectionName`.

Object and Document are used as interchangeable terms.
This is discussed in future lessons.

## MongoDB Documents: Fields with Documents as Values
MongoDB supports nesting a document as the value of a field in another document.
MongoDB's query language supports filtering documents based on the values of fields within nested documents.

## MongoDB Documents: Fields with Arrays as Values
MongoDB query lanague supports queries on array fields making it easy to filter documents containing a particular set of supplementary fields.
We simply need to build an index on the array field.

An array can also contain other documents as its elements.

## MongoDB Documents: Geospatial Data
Geospatial is not a value type per-say, but we can use document structures to get geospatial data.
This enables us to do some things like:
- store shapes
- calcualte distance between points
- filter for documents within a specified radius of another point