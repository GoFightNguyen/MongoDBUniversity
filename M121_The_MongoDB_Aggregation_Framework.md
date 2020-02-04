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
All movies winning an Oscar begin with a string resemblings one of the following in their awards field: `Won 13 Oscars` or `Won 1 Oscar`
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

## $graphLookup
Allows looking up recursively, a set of documents with a defined relationship to a starting document.
This provides graph or graph-like capablitites.
It also provides MongoDB a transitive closure implementation.
```js
$graphLookup: {
  from: <lookup_table>,
  startWith: <expression_for_value_to_start_from>,
  connectFromField: <field_name_to_connect_from>,
  connectToField: <field_name_to_connect_to>,
  as: <field_name_for_result_array>,
  maxDepth: <max_number_of_iterations_to_perform__0_means_1>,
  depthField: <field_name_for_number_of_iterations_to_reach_this_node__this_is_0_for_first_lookup>,
  restrictSearchWithMatch: <match_condition_to_apply_to_lookup>
}
```

```js
// Who all reports to Eliot, indirectly or directly?
db.parent_reference.aggregate([
  {
    $match: {name: 'Eliot'}
  },
  {
    $graphLookup: {
      from: "parent_reference",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "reports_to",
      as: "all_reports"
    }
  }
])
```

### $graphLookup: cross-collection lookup
```js
use air

db.airlines.aggregate([
  {
    $match: {name: "TAP Portugal"}
  },
  {
    $graphLookup: {
      from: "routes",  //other collection. cannot be sharded
      as: "chain",
      startWith: "$base",
      connectFromField: "dst_airport",
      connectToField: "src_airport",
      restrictSearchWithMatch: { "airline.name": "TAP Portugal"},
      maxDepth: 1 //only restricts the number of recursive lookups on the from collection when doing a cross-collection lookup. For this example, it translates to at most 1 layover
    }
  }
]).pretty()
```

Find all possible distinct destinations
- with at most one layover
- departing from the base airports of airlines that are part of the "OneWorld" alliance
- airlines are national carriers from Germany, Spain, or Canada
Include both the destination and which airline services that location.
We will need the air_airlines, air_alliances, and air_routes collections in the aggregations db.
```js
db.air_alliances.aggregate([{
  $match: { name: "OneWorld" }
}, {
  $graphLookup: {
    startWith: "$airlines",
    from: "air_airlines",
    connectFromField: "name",
    connectToField: "name",
    as: "airlines",
    maxDepth: 0,
    restrictSearchWithMatch: {
      country: { $in: ["Germany", "Spain", "Canada"] }
    }
  }
}, {
  $graphLookup: {
    startWith: "$airlines.base",
    from: "air_routes",
    connectFromField: "dst_airport",
    connectToField: "src_airport",
    as: "connections",
    maxDepth: 1
  }
}, {
  $project: {
    validAirlines: "$airlines.name",
    "connections.dst_airport": 1,
    "connections.airline.name": 1
  }
},
{ $unwind: "$connections" },
{
  $project: {
    isValid: { $in: ["$connections.airline.name", "$validAirlines"] },
    "connections.dst_airport": 1
  }
},
{ $match: { isValid: true } },
{ $group: { _id: "$connections.dst_airport" } }
])
```

# Chapter 4: Core Aggregation - Multidimensional Grouping
Facet Navigation enables devs to create an interface characterizing query results across multiple dimensions (facets).

Faceting is an analytics capability allowing users to explore data by applying multiple filters and characterizations.

## Single Facet Query
```js
db.companies.aggregate([
  {$match: {'$text': {'$search': 'network'}}},
  // create a facet by category
  {$sortByCount: '$category_code'}
])
```
$sortByCount outputs documents with two fields:
- `_id` the value the facet is for
- `count` number of documents matching that value
It works just like a $group stage followed immediately by a $sort (in descending) stage.

## $bucket
bucketing is for grouping data by ranges of values.

```js
$bucket: {
  groupBy: <expression>,
  boundaries: [<lowerBound1>, <lowerbound2>,...],
  default: <literal>,
  output: {
    <output1>: {<$accumulator_expression>},
    ...
    <outputN:>: {<$accumulator_expression>}
  }
}

db.movies.aggregate([
  {
    $bucket: {
      groupBy: "$imdb.rating",
      boundaries: [0,5,8,Infinity],
      default: "not rated",
      output: {
        average_per_bucket: {$avg: "$imdb.rating"},
        count: {$sum: 1}
      }
    }
  }
])
```

$bucket:
- groupBy - must evaulate down to a single expression, even if operating on multiple fields
- boudnaries - each value is the lower bound for the group the document will be placed in
  - values must be of the same type, but you can mix types of numbers
  - must be at least two boundary values
  - ex: [0,5,8,Infinity] the first grouping is [0,5)
- default - optional, but important
  - If the groupBy field is not set on the document, this value will be used
  - If you do not set this and a document is missing the field, then it will crashy your query
- output - `count` is the default, but removed when output is specified
  - the default output of bucket is
    - `_id` - the boundary (including default) grouping
    - `count` - how many documents were in that boundary

## $bucketAuto
```js
db.movies.aggregate([
  {
    $match: {"imdb.rating": {$gte: 0}}
  },
  {
    $bucketAuto: {
      groupBy: "$imdb.rating",
      buckets: 4
    }
  }
])
```

similar to $bucket, but key differences:
- buckets - instead of defining boundares, let MongoDB figure them out by specifying how many buckets we want
  - might not get as many buckets back as you specify, especially if
    - there are less documents than buckets specified
    - there are less unique values than buckets specified
- granularity - optional. Attempts to place boundaries along the preferred number series (look at the documentation for types of number series)
  - Setting this means your `groupBy` must be numeric

## multiple facets
$facet allows several sub-pipelines to be executed to produce multiple facets.
It allows the application to generate several different facets with one single database request.
Each entry in $facet is completely independent from the others.
Thus, the output of one cannot be used by another.
For example, the output of Categories cannot be used by Employees
```js
db.companies.aggregate([
  {$match: {"$text": {$search: "Databases"}}},
  {
    $facet: {
      // facet 1
      Categories: [{$sortByCount: "$category_code"}],
      // facet 2
      Employees: [
        {$match: {"founded_year": {$gt: 1980}}},
        {$bucket: {
          groupBy: "$number_of_employees",
          boundaries: [0,20,50,100,500,1000,Infinity],
          default: "Other"
        }}
      ]
    }
  }
])
```

## lab
How many movies are in both the top ten highest rated movies according to the imdb.rating and the metacritic fields?
```js
db.movies.aggregate([
  {
    $match: {
      metacritic: {$gte: 0},
      "imdb.rating": {$gte: 0}
    }
  },
  {
    $facet: {
      TopTenMetacritic : [
        { $sort: { metacritic: -1, title: 1}},
        { $limit: 10},
        { $project: {_id: 0, title: 1}}
      ],
      TopTenImdb: [
        { $sort: { "imdb.rating": -1, title: 1}},
        { $limit: 10},
        { $project: {_id: 0, title: 1}}
      ]
    }
  },
  {
    $project: {
      movies_in_both: {
        $setIntersection: ["$TopTenMetacritic", "$TopTenImdb"]
      }
    }
  }
]).pretty()
```

# Chapter 5: Miscellaneous Aggregation
## $redact
One way to implement access control in regards to what fields are returned.
It is __not__ for restricting access to a collection.

`$redact: <expression>`
The expression must resolve to one of three values:
- $$DESCEND - retains the current level and evaluates the next level down.
If referring to a field in the document, then every level __must__ include that field, otherwise you must specify what to do.
- $$PRUNE - remove; excludes all fields at the current document level without further inspection.
In other words, $$PRUNE automatically applies to lower levels
- $$KEEP - retain; include all fields at the current document level without further inspection.
In other words, automatically applies to lower levels.

```js
db.employees.aggregate([
  {
    $redact: {$cond: [{$in: ["Management", "$acl"]}, "$$DESCEND", "$$PRUNE"]}
  }
]).pretty()
```
This example does:
- look at current level
- If the acl field contains Management, then $$DESCEND is followed.
- Otherwise, $$PRUNE is followed.
- If it was $$DESCEND, then it will keep traversing down, and keeping, until a $$PRUNE occurs

## $out
Useful for persisting the results of an aggregation.
`$out: "output_collection"`

Must be the last stage in a pipeline.
This implies it cannot be used within a $facet.

MongoDB creates the collection if it does not exist, otherwise override the existing one.
The collection must/will exist in the same db.
If overriding an existing collection, the existing indexes will still be in place.

## Views
MongoDB enables non-materialized views, meaning they are computed every time a read operation is performed against that view.
Useful:
- to create vertical/horizontal slices of a collection
  - vertical: occurs through $project and other stages changing the shape of the document(s) being returned. It does not change the number of document being returned.
  - horizontal: occurs through $match stages. It changes the number of document(s) returned, but not the shape.

Views can be created 2 ways:
- db.createView(<view>, <source>, <pipeline>, <collation>)
- db.createCollection(<name>, <options>)

```js
db.createView("bronze_banking", "customers", [
  {
    $match: {accountType: "bronze"}
  },
  {
    $project: {
      _id: 0,
      name: "$name.first",
      account_ending: {$substr: ["$accountNumber", 7, -1]}
    }
  }
])
```

View restrictions:
- no write operations
- no index operations
- no renaming
- no $text
- no geoNear or $geoNear
- find() operations with projection operators are not permitted
- there's more

Views are public.
So avoid sensitive data in them.

# Chapter 6: Aggregation Pipeline Performance
Two high-level categories of aggregation queries:
- "realtime" processing
  - provide data for applications
  - thus, performance is more important
- batch processing
  - provide data for analytics
  - run periodically
  - performance is less important

## Index Usage
Some aggregation operators can use indexes, while others cannot.
When executing the pipeline, as soon as server encounters a stage unable to use indexes, none of the following stages will be able to either.
The Query Optimizer tries to detect when a stage can be moved forward so indexes can be utilized.

Pass `{explain: true}` to the aggregate function to get the details on the aggregation.

Try to have $match, $limit, and $sort as close to the beginning as possible since all these stages can use indexes.

## Memory Constraints
Results are subject to a 16MB document limit.
100MB of RAM per stage.
`{allowDiskUsage: true}` does not work with $graphLookup since $graphLookup does not support spilling to disk.

## Aggregation Pipeline on a Sharded Cluster
Some operators, such as $out and $lookup, will cause a merge stage on the primary shard for a database.

## Pipeline Optimization
Avoid unnecessary stages, the Aggregation Framework can project fields automatically if final shape of the output document can be determined from initial input.

Use accumulator expressions $map, $reduce, $filter in project before an $unwind, if possible.

Every high order array function can be implemented with $reduce if the provided expressions do not meet your needs.


# Final
Which alliance has the most unique airlines operating between the airports JFK and LHR, in either directions?
```js
db.air_routes.aggregate([
  {
    $match: {
        src_airport: {$in: ["JFK","LHR"]},
        dst_airport: {$in: ["JFK","LHR"]}
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
    $match: {alliance: {$ne: []}}
  },
  {
    $project: {
      _id: 0,
      airlineName: "$airline.name",
      allianceName: { $arrayElemAt: ["$alliance.name", 0] }
    }
  },
  {
    $group: {
      _id: "$allianceName",
      airlines: {$addToSet: "$airlineName"}
    }
  }
]).pretty()
```