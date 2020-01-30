# Chapter 0 Introduction and Aggregation Concepts
connection string: `mongo "mongodb://cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/aggregations?replicaSet=Cluster0-shard-0" --authenticationDatabase admin --ssl -u m121 -p aggregations --norc`

[Aggregation Pipeline Quick Reference](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/)

[$match (aggregation)](https://docs.mongodb.com/manual/reference/operator/aggregation/match/)

[$project](https://docs.mongodb.com/manual/reference/operator/aggregation/project/)

## Pipelines
The Aggregation Framework provides many stages to filter and transform data.
Documents flow through the pipeline, passing from one stage to the next.

## Aggregation Structure and Syntax
`db.userColl.aggregate([{stage1}, {stage2}, {...stageN}], {options})`
Each stage is a JSON object composed of one or more aggregation operators or expressions.

Operators refers to either query operators or aggregation stages.
Operators generally appear in the key position.
Expressions are like fuctions, we provide args and they provide computed output.
Expressions can be composed to form new data transformations.
Expressions always appear in the value position.

Field Path: $fieldName, i.e. $numberOfMoons
System Variable: $$UPPERCASE, i.e. $$CURRENT
User Variable: $$foo

# Chapter 1: Basic Aggregation - $match and $project
## $match: Filtering documents
$match should come early in an aggregation pipeline.
If $match is the first stage, then it can take advantage of indexes.
Uses standard MongoDB query operators, woohoo!
$match does not allow for projections.

Limitations to query operators you can use with $match:
- cannot use the $where operator
- to contain a $text query operator, $match must be the first stage

Think of $match as a filter.
If your pipeline only does $match, you could probably just use $find.

```js
db.solarSystem.aggregate([
    { $match: {type: {$ne: "Star"}}}
]).pretty()
```

Find all movies where:
- imdb.rating is at least 7
- genres does not contain "Crime" or "Horror"
- rated is either "PG" or "G"
- languages contains "English" and "Japanese"
```js
db.movies.aggregate([
    { $match: {"imdb.rating": {$gte: 7},
                rated: {$in: ["PG", "G"]},
                languages: {$all: ["English","Japanese"]},
                genres: {$nin: ["Crime","Horror"]}
              }
    }
]).pretty()
```

## Shaping documents with $project
$project is more than the projection functionality you can do in the `find` operator, you can also:
- add new fields
- reassign values to existing field names and derive entirely new fields

`_id` is automatically included unless you explicitly exclude it.
$project can be used multiple times within an Aggregation pipeline

Same as the example above, but with a projection
```js
db.movies.aggregate([
    { $match: {"imdb.rating": {$gte: 7},
                rated: {$in: ["PG", "G"]},
                languages: {$all: ["English","Japanese"]},
                genres: {$nin: ["Crime","Horror"]}
              }
    },
    { $project: {_id: 0, title: 1, rated: 1}}
]).pretty()
```

How many movies have a title composed of one word.
To clarify, "Cinderalla" and "3-25" should count, where as "Cast Away" would not.
```js
db.movies.aggregate([
    {
        $project: { "titleWords": {$split: ["$title", " "] } }
    },
    {
        $match: {titleWords: {$size: 1}}
    }
]).itcount()
```

How many movies are a "labor of love", where the same person appears in cast, directors, and writers?
Some writers are like "Jason Smith (story)".
This is not finished :(
```js
db.movies.aggregate([
    {
        $match:{
            // writers where an array and not empty
            writers: {$elemMatch: {$exists: true}}
        }
    },
    {
        $project: {
            _id: 0,
            title: 1,
            writers: {
                $map: {
                    input: "$writers",
                    as: "writer",
                    in: {
                        $arrayElemAt: [
                            { $split: ["$$writer", " ("] }, 0
                        ]
                    }
                }
            }
        }
    }
]).pretty()
```

# Chapter 2: Basic Aggregation - Utility Stages
$addFields only allows you to modify existing fields or add new ones.
$addFields does not remove any fields from the document from being displayed.
So, we can use $addFields in place of $project if we do not want to limit which fields are included from the original document.

$geoNear is useful when working with geo-JSON data.
When used, this must be the first stage in the pipeline.
$geoNear requires the collection to have one-and-only-one geo-index.
$geoNear can be used on charted collections.

`{$sample: { size: <N number of documents>}}`
$sample selects a set of random documents from the collection in one of two ways:
- a sudo-random cursor will choose the number of documents when
  - N <= 5% of number of documents in sourc collection, and
  - Source collection has >= 100 documents, and
  - $sample is the first stage
- otherwise, an in-memory random sort occurs choosing the specified number of documents

## Cursor-like stages
`$limit: {<integer>}` think of this like First/Top in other languages

`$skip: {<integer>}`

`$count: {<name we want the count called>}`

`$sort: {<field we want to sort on>: <integer, direction to sort>}` -1 indicates descending.
Can sort on multiple fields.
If $sort is first (there are some nuances to "first"), then it can utilize indexes; otherwise, it is an in-memory sort.
By default, $sort will only use up to 100MB of RAM.
Setting `allowDiskUse: true` will allow for larger sorts.

## Labs
For movies released in the USA with a `tomatoes.viewer.rating` >= 3, calculate a new field called num_favs representing how many favorites appear in the `cast` field of the movie.
Sort the results by num_favs, `tomatoes.viewer.rating`, and `title`, all in descending order.
What is the title of the 25th film?
```js
favorites = ["Sandra Bullock", "Tom Hanks", "Julia Roberts", "Kevin Spacey", "George Clooney"]

db.movies.aggregate([
  {
    $match: {
      "tomatoes.viewer.rating": { $gte: 3 },
      countries: "USA",
      cast: {
        $in: favorites
      }
    }
  },
  {
    $project: {
      _id: 0,
      title: 1,
      "tomatoes.viewer.rating": 1,
      num_favs: {
        $size: {
          $setIntersection: [
            "$cast",
            favorites
          ]
        }
      }
    }
  },
  {
    $sort: { num_favs: -1, "tomatoes.viewer.rating": -1, title: -1 }
  },
  {
    $skip: 24
  },
  {
    $limit: 1
  }
])
```

Calculate an average rating for each movie where
- English is an available language
- minimum imdb.rating is at least 1
- released in 1990 or after
- minimum imdb.votes is at least 1
  - need to rescale (or normalize) imdb.votes
  - look at `scaling.js`
```js
db.movies.aggregate([
  {
    $match: {
      year: { $gte: 1990 },
      languages: "English",
      "imdb.votes": { $gte: 1 },
      "imdb.rating": { $gte: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      title: 1,
      "imdb.rating": 1,
      "imdb.votes": 1,
      normalized_rating: {
        $avg: [
          "$imdb.rating",
          {
            // scaled votes
            $add: [
              1,
              {
                $multiply: [
                  9,
                  {
                    $divide: [
                      { $subtract: ["$imdb.votes", 5] },
                      { $subtract: [1521105, 5] }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }
    }
  },
  { $sort: { normalized_rating: 1 } },
  { $limit: 1 }
])
```

# Chapter 3: Core Aggregation - Combining Information
## $group
`$group: {_id: criteria, fieldName: accumulator_expression}`
`$group: {_id: "$year", num_films_in_year: {$sum: 1}}`
You can have as many accumulator expressions as you want.

Accumulators will ignore the field if the type is not as expected; thus, might need to sanitize data.
If you do _id: null, then it creates one group and thus applies the accumulator across all documents.
$group can be used multiple times within a pipeline.

## Accumulator Expressions in $project
Accumulator Expressions in $project operate over an array in the current document, they do not carry values over all documents.

For all films that won at least 1 Oscar, calculate the std deviation, highest, lowest, and average imdb.rating.
Use the sample std deviation expression.
All movies winning an Oscar begen with a string resemblings one of the following in their awards field: `Won 13 Oscars` or `Won 1 Oscar`
```js
db.movies.aggregate([
  {
    $match: {
      awards: { $regex: "^Won [0-9]+ O" }
      // awards: /Won \d{1,2} Oscars?/
    }
  },
  {
    $group: {
      _id: null,
      highest_rating: {$max: "$imdb.rating"},
      lowest_rating: {$min: "$imdb.rating"},
      average_rating: {$avg: "$imdb.rating"},
      deviation: {$stdDevSamp: "$imdb.rating"}
    }
  }
]).pretty()
```

## $unwind
This stage unwinds an array field to create a new document for every entry.
In each new document, the field will now have that specific value.
This can be useful when you want to group on each entry instead of the array as a whole.
```js
// for each year, get the highest rated genre

db.movies.aggregate([
  {
    $match: {
      "imdb.rating": {$gt: 0},
      year: {$gte: 2010, $lte: 2015},
      runtime: {$gte: 90}
    }
  },
  {
    $unwind: "$genres"
  },
  {
    $group: {
      _id: {
        year: "$year",
        genre: "$genres"
      },
      average_rating: {$avg: "$imdb.rating"}
    }
  },
  {
    $sort: {"_id.year": -1, average_rating: -1}
  },
  {
    $group: {
      _id: "$_id.year",
      genre: {$first: "$_id.genre"},
      average_rating: {$first: "$average_rating"}
    }
  }
])
```

We have seen the short form of $unwind: `$unwind: <field_path>`.
There is a long form too: `$unwind: { path: <field_path>, includeArrayIndex: <field_name_to_use_in_new_doc>, preserveNullAndEmptyArrays: <boolean>}`

Calculate how many English-language movies every cast member has been in and get an average imdb.rating for each cast member.
```js
db.movies.aggregate([
  {
    $match: {
      languages: "English"
    }
  },
  {
    $project: {
      _id: 0,
      cast: 1,
      "imdb.rating": 1
    }
  },
  {
    $unwind: "$cast"
  },
  {
    $group: {
      _id: "$cast",
      numFilms: {$sum: 1},
      average: {$avg: "$imdb.rating"}
    }
  },
  {
    $sort: {numFilms: -1}
  }
]).pretty()
```

## The $lookup stage
This stage enables combining information from multiple collections.
Think `Left Outer Join` in `SQL`.

```js
$lookup: {
  from: <collection_to_join>,
  localField: <field_from_the_input_documents>,
  foreignField: <field_from_the_docs_of_the_from_collection>,
  as: <output_array_field>
}
```
The `from` collection cannot be sharded.
The `from` collection must be in the same database.
The values in `localField` and `foreignField` are matched on equality.
If the `as` field you specified already exists, it will be overwritten.
If there are no matches, the `as` field you specified will be an empty array.

```js
db.air_alliances.aggregate([
  {
    $lookup: {
      from: "air_airlines",
      localField: "airlines",
      foreignField: "name",
      as: "airlines"
    }
  }
]).pretty()
```

Which alliance from air_alliances flies the most routes with either a Boeing 747 or an Airbus A380 (abbreviated 747 and 380 in air_routes)?
```js
db.air_routes.aggregate([
  {
    $match: {
      airplane: /747|380/
    }
  },
  {
    $lookup: {
      from: "air_alliances",
      localField: "airline.name",
      foreignField: "airlines",
      as: "alliance"
    }
  },
  {
    $unwind: "$alliance"
  },
  {
    $group: {
      _id: "$alliance.name",
      count: {$sum: 1}
    }
  }
])
```