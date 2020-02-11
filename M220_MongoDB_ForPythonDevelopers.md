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