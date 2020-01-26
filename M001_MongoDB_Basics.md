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

# Chapter 2 The MongoDB Query Language + Atlas
Compass does not yet fully support the MongoDB Query Language.
Enter the Mongo Shell.

`mongo --nodb` starts the mongo shell without attempting to connect to any mongo db.
`quit()` stops and exits the mongo shell.

## Connecting to the class Atlas Cluster from the mongo shell
When connecting to a cluster, we want to provide the mongo shell with the name of all servers (in other words, hostnames of all nodes) in the cluser.

MongoDB is designed to provide high-availability (HA) access to your data, it does this by enabling you to maintain redundant copies of your data in a cluster using a Replica Set.
This is why you need to provide the mongo shell with all servers in the cluster.
If the primary node goes down, the shell can connect to other nodes in the cluster instead.

The shell always connects to the primary for a cluster because all writes must be directed to it, and most reads as well.

The shell is a fully-functionaly javascript interpreter.

`show dbs` shows the databases in the cluster.
`use video` switches to the video db.
`show collections` shows the collections within the current db.
`db.movies.find().pretty()` will show the documents inside the movies collection.

`load("something.js")` will execute the js file.

## Creating Documents: insertOne()
In MongoDB we add documents by create (aka insert).

`insertOne()` is a method on the collection class.
`db.moviesScratch.insertOne({json data here})`

If the collection does not exist, then it will be created.

All documents in MongoDB must contain an `_id` field.
This is a unique identifier for a document within a collection.
When inserting a document, if you do not specify the `_id` then MongoDB creates it for you.
When MongoDB creates the `_id` for you, the type will be `ObjectID`.

## Creating Documents: insertMany()
`insertMany()` receives an array.
It is a method on the collection class.
`db.moviesScratch.insertMany([{first}, {second}])`

Default is ordered inserts.
This means `insertMany` will stop inserting as soon as an error is encountered.
To turn this off, in the JSON argument include `{ "ordered": false }`.

## Reading Documents: Scalar Fields
The method for performing read operations in the MongoDB query language is `find`.
`db.movies.find({mpaaRating: "PG-13"}).pretty()`

When using a nested scalar field, you use dot notation.
When doing this in the shell, you must quote the key.
`db.data.find({"wind.direction.angle": 290}).pretty()`

## Reading Documents: Array Fields
We can consider matches on the entire array, any element of the array, and a specific element of the array.
You can also do more complex matches using operators, which is not discussed here.

exact match (including order): `db.movies.find({cast: ["Jeff Bridges", "Tim Robbins"]}).pretty()`

any element: `db.movies.find({cast: "Jeff Bridges"}).pretty()`

specific index: `db.movies.find({"cast.0": "Jeff Bridges}).pretty()`

## Cursors
The `find` method returns a cursor, which is essentially a pointer to a location in a result set.
In the shell, the cursor defaults to 20 documents at a time.
In the shell, iterating the cursor (`it`) performs a `getMore` operation to retreive the next 20 results.

## Projections
Projections reduce network overhead and processing requirements by limiting the fields returned in a results document.
The default is everything.
The `_id` is returned by default for all projections, unless you explicitly exclude it.

To explicitly include a field, use 1.
To explicitly exclude a field, use 0.

The projection is the second argument of `find`.
`db.movies.find({genre: "Action, Adventure"}, {title: 1, _id: 0})`

## Updating Documents: updateOne()
`updateOne()` is for updating a single document, so it only updates the first match.

`$set` takes a document as an arg.
If the specified field(s) exists, then it updates it.
Otherwise, the field is added.

```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $set: {
        poster: "path"
    }
})
```

## Update Operators
There are many, look at the documentation.

Example of `$inc`
```
db.movieDetails.updateOne({
    title: "The Martian"
}, {
    $inc: {
        "tomato.reviews": 3,
        "tomato.userReviews": 25
    }
})
```

Some operators, particularly ones dealing with arrays, have modifiers associated to them.

## Updating Documents: updateMany()
Same as `updateOne()`, but applies to all matches.

## Upserts
upsert is an argument to pass to `updateOne()` and `updateMany()`

## Updating Documents: replaceOne()
`replaceOne()` applies changes to only one document, the first found in the server that matches the filter expression, using the `$natural` order of documents in the collection.

`updateOne` updates a specific value.
`replaceOne` replaces the entire document with whatever document you provide.

## Deleting Documents
`deleteOne()`
`deleteMany()`