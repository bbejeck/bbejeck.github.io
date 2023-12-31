---
layout: post
title: "Spark PairRDDFunctions: CombineByKey"
date: 2015-08-03 21:39:25 -0400
permalink: /spark-combine-by-key
comments: true
categories: 
- Scala
- Spark
tags: 
- Scala
- Spark
keywords: 
- Scala
- Spark
- PairRDDFunctions
description: Exploring functions in the PairRDDFunctions class specifically looking at alternatives to groupByKey for performance improvement
---
Last time we covered one of the alternatives to the `groupByKey` function `aggregateByKey`.  In this post we'll cover another alternative PairRDDFunction - `combineByKey`.  The `combineByKey` function is similar in functionality to the `aggregateByKey` function, but is more general.  But before we go into details let's review why we'd even want to avoid using `groupByKey`.  When working in a map-reduce framework such Spark or Hadoop one of the steps we can take to ensure maximum performance is to limit the amount of data sent accross the network during the shuffle phase. The best option is when all operations can be performed on the map-side exclusively, meaning no data is sent at all to reducers.  In most cases though, it's not going to be realistic to do map-side operations only. If you need to do any sort of grouping, sorting or aggregation you'll need to send data to reducers.  But that doesn't mean we still can't attempt to make some optimizations. 
<!-- more -->
### GroupByKey Is Expensive
In the case of a `groupByKey` call, *every* single key-value pair will be shuffled accross the network with identical keys landing on the same reducer.  To state the obvious, when grouping by key, the need for all matching keys to end up on the same reducer can't be avoided.  But one optimization we can attempt is to combine/merge values so we end up sending fewer key-value pairs in total.  Addtionaly, less key-value pairs means reducers won't have as much work to do, leading to additional performance gains.  The `groupByKey` call makes no attempt at merging/combining values, so it's an expensive operation.

### CombineByKey
The `combineByKey` call is just such an optimization.  When using `combineByKey` values are merged into one value at each partition then each partition value is merged into a single value.  It's worth noting that the type of the combined value does not have to match the type of the original value and often times it won't be.  The `combineByKey` function takes 3 functions as arguments:

 1.   A function that creates a combiner. In the `aggregateByKey` function the first argument was simply an initial zero value.  In `combineByKey` we provide a function that will accept our current value as a parameter and return our new value that will be merged with addtional values.
 2.   The second function is a merging function that takes a value and merges/combines it into the previously collecte value(s).
 3.   The third function combines the merged values together.  Basically this function takes the new values produced at the partition level and combines them until we end up with one singular value.

### CombineByKey Example
For our example lets take a look at calculating an average score.  Calculating an average is a litte trickier compared to doing a count for the simple fact that counting is [associative](https://en.wikipedia.org/wiki/Associative_property) and [commutative](https://en.wikipedia.org/wiki/Commutative_property), we just sum all values for each partiton and sum the partition values.  But with averages, it's not that simple, an average of averages is not the same as taking an average across all numbers.  But we can collect the total number scores and total score per partition then divide the total overall score by the number of scores.  Here's our example:
```scala CombineByKey for Averaging
  //type alias for tuples, increases readablity
type ScoreCollector = (Int, Double)
type PersonScores = (String, (Int, Double))

val initialScores = Array(("Fred", 88.0), ("Fred", 95.0), ("Fred", 91.0), ("Wilma", 93.0), ("Wilma", 95.0), ("Wilma", 98.0))

val wilmaAndFredScores = sc.parallelize(initialScores).cache()

val createScoreCombiner = (score: Double) => (1, score)

val scoreCombiner = (collector: ScoreCollector, score: Double) => {
         val (numberScores, totalScore) = collector
        (numberScores + 1, totalScore + score)
      }

val scoreMerger = (collector1: ScoreCollector, collector2: ScoreCollector) => {
      val (numScores1, totalScore1) = collector1
      val (numScores2, totalScore2) = collector2
      (numScores1 + numScores2, totalScore1 + totalScore2)
    }
val scores = wilmaAndFredScores.combineByKey(createScoreCombiner, scoreCombiner, scoreMerger)

val averagingFunction = (personScore: PersonScores) => {
       val (name, (numberScores, totalScore)) = personScore
       (name, totalScore / numberScores)
    }

val averageScores = scores.collectAsMap().map(averagingFunction)

println("Average Scores using CombingByKey")
    averageScores.foreach((ps) => {
      val(name,average) = ps
       println(name+ "'s average score : " + average)
    })
```      
Let's describe what's going on in the example above:

1.   The `createScoreCombiner` takes a double value and returns a tuple of (Int, Double)
2.   The `scoreCombiner` function takes a `ScoreCollector` which is a [type alias](http://www.scala-lang.org/files/archive/spec/2.11/04-basic-declarations-and-definitions.html#type-declarations-and-type-aliases) for a tuple of (Int,Double). We alias the values of the tuple to `numberScores` and `totalScore` (sacraficing a one-liner for readablility). We increment the number of scores by one and add the current score to the total scores received so far.
3. The `scoreMerger` function takes two `ScoreCollector`s adds the total number of scores and the total scores together returned in a new tuple.
4.    We then call the `combineByKey` function passing our previously defined functions.
5.    We take the resulting RDD, scores, and call the `collectAsMap` function to get our results in the form of (name,(numberScores,totalScore)).
6.    To get our final result we call the `map` function on the scores RDD passing in the `averagingFunction` which simply calculates the average score and returns a tuple of (name,averageScore) 

#### Final Results
After running our spark job, the results look like this:
```text Results
Average Scores using CombingByKey
Fred's average score : 91.33333333333333
Wilma's average score : 95.33333333333333
```  
### Conclusion
While the use of `combineByKey` takes a little more work than using a `groupByKey` call, hopefully we can see the benefit in this simple example of how we can improve our spark job performance by reducing the amount of data sent accross the network

### Resources
*   [Source Code](https://gist.github.com/bbejeck/d3f458b6ec26c23848cd)
*   [PairRDD API](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)
*   [Spark](http://spark.apache.org/docs/latest/index.html)

