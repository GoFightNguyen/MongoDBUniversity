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