# Chapter 1 Driver Setup
```python
from pymongo import Mongoclient
uri = "mongodb+srv://m220user:m220password@m220-lessons-mcxlm.mongodb.net/test"
client = MongoClient(uri)

client.stats()
client.list_database_name()

# mflix is a db
mflix = client.mflix # could also do client['mflix']
mflix.list_collection_names()
movies = mflix.movies   # movies is a collection
movies.count_documents({})
```

The provided mflix proejct includes Jupyter Notebooks.
To use:
```
cd mflix-python/notebooks
jupyter notebook
```

## Basic Reads
```python
movies.find_one() # first document using MongoDB natural order

# find returns a cursor
# documents where Salma Hayek is in the cast
movies.find({"cast": "Salma Hayek"})

cursor = movies.find({"cast": "Salma Hayek"})
from bson.json_util import dumps
print(dumps(cursor, indent=2))

# use a projection
movies.find({"cast": "Salma Hayek}", {"title": 1})
```

Some methods, such as `find` return a cursor.
You can only iterate through a cursor once, meaning that once you do, then you can no longer pull any documents from it.

# Chapter 2: User-Facing Backend
## Cursor Methods and Aggregation Equivalents

Some aggregation methods, such as `$limit`, `$skip`, and `$sort`, have equivalent cursor methods.
We will use limit as an example, but know that usage is the same for all the others.

Limiting

```python
limited_cursor = movies.find(
    { "directors": "Sam Raimi"},
    {"_id": 0, "title": 1, "cast": 1}
).limit(2)
print(dumps(limited_cursor, indent=2))

pipeline = [
    { "$match": {"directors": "Sam Raimi"}},
    { "$project": { "_id": 0, "title": 1, "cast": 1}},
    { "$limit": 2}
]
limited_aggregation = movies.aggregate(pipeline)
print(dumps(limited_aggregation, indent=2))
```

Some special notes about using the sort method on a cursor:
- add `from pymongy import DESCENDING, ASCENDING`.
- using the `sort` method on a cursor takes an array of tuples when sorting on multiple keys `...sort([("year", ASCENDING), ("title", ASCENDING)])`.

## Facet
```python
# An example of what defining the facet stage
# You then just add it to your pipeline collection
facet_stage = {
        "$facet": {
            "runtime": [{
                "$bucket": {
                    "groupBy": "$runtime",
                    "boundaries": [0, 60, 90, 120, 180],
                    "default": "other",
                    "output": {
                        "count": {"$sum": 1}
                    }
                }
            }],
            "rating": [{
                "$bucket": {
                    "groupBy": "$metacritic",
                    "boundaries": [0, 50, 70, 90, 100],
                    "default": "other",
                    "output": {
                        "count": {"$sum": 1}
                    }
                }
            }],
            "movies": [{
                "$addFields": {
                    "title": "$title"
                }
            }]
        }
    }
```

## Basic Writes
`insert_one()` returns an `InsertOneResult`, which contains the _id of an inserted document and whether the operation was ack'd by the server.
Perform an upsert by using `update_one()` and set `upsert=True`.

## Write Concerns
`writeConcern: {w:1}`
- only requests an acknowledgement that one node applied the write
- this is the default `writeConcern` in MongoDB
- the one indicates the number of nodes that must apply the write before the client recieves an ack
  - the primary sends an ack after it writes, even before the write is replicated to the secondary nodes.

`writeConcern: {w: 'majority'}`
- requests ack that a majority of nodes in the replica set applied the write
- takes longer than w:1
  - although there is a replication lag, there is no additional load on the server; thus, the primary can still perform the same number of writes/sec
- is more durable than w:1
  - useful for ensuring vital writes are majority-committed; even if the primary fails over, the data will not be lost

`writeConcern: {w:0}`
- does not request an ack that any nodes applied the write
  - think "fire-and-forget"
- fastest `writeConcern` level
- least durable `writeConcern`
- can still alert the client of network/socket exceptions

```python
db.users.with_options(
    write_concern=WriteConcern(w='majority')
).insert_one({
    "name": name,
    "email": email,
    "password": hashedpw
})
```

## Basic Updates
Two idiomatic update operations
- `update_one`
- `update_many`
Update operations return an `UpdateResult`
- acknowledged, matched_count, modified_count, and upserted_id
- modified_count and matched_count will be 0 in the case of an upsert

## Basic Joins
https://docs.mongodb.com/manual/reference/operator/aggregation/lookup/

```python
# creating a pipeline going from movies to comments
pipeline = [
    {
        "$match": {
            "_id": ObjectId(id)
        }
    },
    {
        "$lookup": {
            'from': 'comments',
            'let': { 'id_of_the_movie': '$_id'},    # store the movie._id in the $$id_of_the_movie variable, otherwise pipeline cannot access it
            'pipeline': [
                {
                    # $movie_id is the reference key in comments
                    '$match': { '$expr': {'$eq': ['$$id_of_the_movie', '$movie_id']}}
                },
                {
                    '$sort': {'date': DESCENDING}
                }
            ],
            'as': 'comments'
        }
    }
    # if we didn't need to sort, then this simple approach would have worked
    # {
    #     '$lookup': {
    #         'from': 'comments',
    #         'localField': '_id',
    #         'foreignField': 'movie_id',
    #         'as': 'comments'
    #     }
    # }
]
```

## Basic Deletes
`delete_one` will delete the first document matching the supplied predicate.
`delete many` will delete all documents matching the supplied predicate.
The number of documents deleted can be accessed via the `deleted_count` property on the `DeleteResult` object returned from a delete operation.

# Chapter 3: Admin Backend
## Read Concerns
Similar to write concerns, it is about how many nodes are involved in a read operation.
Read concerns
- represent different levels of "read isolation"
- can be used to specify a consistent view of the database

`readConcern: local`
- default
- reads from the primary node only, regardless of whether it has been written to secondary nodes

`readConcern: majority`
- reads whatever the majority of the nodes say
- for reading mission-critical data
- more durable reads

```python
# Return the 20 users who have commented the most on MFlix.
pipeline = [
    {
        '$group': {
            '_id': '$email',
            'count': {'$sum': 1}
        }
    },
    { '$limit': 20}
]

rc = ReadConcern(level='majority')
comments = db.comments.with_options(read_concern=rc)
result = comments.aggregate(pipeline)
```

## Bulk Writes
Bulk Writes returns a single ack for the entire batch.
The default behavior is an in-order-execution of the provided writes.
Any failure stops the execution of the rest of the batch.

If order does not matter, pass `{ordered: false}`.
The writes will be written in parallel.
A failure will not stop the execution of other writes.

In a sharded collection, ordered bulk writes take a little longer because write operations have to be routed to the designated shard.
An unordered bulk write has to be serialized across each designated shard.

```python
db.stock.bulkWrite([
    { updateOne: {'filter': {'item': 'apple'}, 'update': {'$inc': {'quantity': 2}}}},
    { updateOne: {'filter': {'item': 'apple'}, 'update': {'$inc': {'quantity': -1}}}}
])
```