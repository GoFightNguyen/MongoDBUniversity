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
```js

```