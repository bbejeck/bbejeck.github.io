---
layout: post
title: "Spark PairRDDFunctions - AggregateByKey"
date: 2015-07-31 08:05:01 -0400
permalink: /spark-agr-by-key
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
description: Exploring functions in the PairRDDFunctions class specivically looking at alternatives to groupByKey for performance improvement
---
One of the great things about the [Spark Framework](http://spark.apache.org/docs/latest/index.html) is the amout of functionality provided out of the box.  There is a class aimed exclusively at working with key-value pairs, the [PairRDDFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions) class.  When working data in the key-value format one of the most common operations to perform is grouping values by key. The PairRDDFunctions class provides a `groupByKey` function that makes grouping by key trivial.  However, `groupByKey` is very expensive and depending on the use case, better alternatives are available.  In a `groupByKey` call, all key-value pairs will be shuffled accross the network to a reducer where the values are collected together.  In some cases the `groupByKey` is merely a starting point to perform additional operations (sum, average) by key.  In other cases, we need to collect the values together in order to return a different value type.    Spark provides some alternatives for grouping that can provide either a performance improvement or ease the ability to combine values into a different type.  The point of this post is to consider one of these alternate grouping functions.
<!-- more -->
### Alternative Grouping Functions
While there are many funtions in the `PairRDDFunctions` class, today we are going to focus on `aggregateByKey`.  The `aggregateByKey` function is used to aggregate the values for each key and adds the potential to return a differnt value type.  
#### AggregateByKey
The aggregateByKey function requires 3 parameters:

1.  An intitial 'zero' value that will not effect the total values to be collected.  For example if we were adding numbers the initial value would be 0. Or in the case of collecting unique elements per key, the initial value would be an empty set.
2. A combining function accepting two paremeters.  The second paramter is merged into the first parameter.  This function combines/merges values within a partition. 
3. A merging function function accepting two parameters.  In this case the paremters are merged into one.  This step merges values across partitions.

As an example let's collect unique values per key. Think of this as an alternative of calling `someRDD.groupByKey().distinct()`
Here's the code:
```scala
    val keysWithValuesList = Array("foo=A", "foo=A", "foo=A", "foo=A", "foo=B", "bar=C", "bar=D", "bar=D")
    val data = sc.parallelize(keysWithValuesList)
    //Create key value pairs
    val kv = data.map(_.split("=")).map(v => (v(0), v(1))).cache()

    val initialSet = mutable.HashSet.empty[String]
    val addToSet = (s: mutable.HashSet[String], v: String) => s += v
    val mergePartitionSets = (p1: mutable.HashSet[String], p2: mutable.HashSet[String]) => p1 ++= p2

    val uniqueByKey = kv.aggregateByKey(initialSet)(addToSet, mergePartitionSets)
```

You will notice we are using *mutable* hashsets in our example.  The reason for using mutable collections is to avoid the extra memeory overhead associated with returning new collections each time we add values to or merge collections.  (This is explicity stated in the PairRDDFunctions documentation). While using `aggregateByKey` is more verbose, if your data has many values but only a few are unique this approach could lead to performance improvements.

For our second example, we'll do a sum of values by key, which should help with performance as less data will be shuffled accross the network.  We provide 3 different parameters to our `aggregateByKey` function.  This time we want to count how many values we have by key regardless of duplicates.

```scala
    val keysWithValuesList = Array("foo=A", "foo=A", "foo=A", "foo=A", "foo=B", "bar=C", "bar=D", "bar=D")
    val data = sc.parallelize(keysWithValuesList)
    //Create key value pairs
    val kv = data.map(_.split("=")).map(v => (v(0), v(1))).cache()

    val initialCount = 0;
    val addToCounts = (n: Int, v: String) => n + 1
    val sumPartitionCounts = (p1: Int, p2: Int) => p1 + p2

    val countByKey = kv.aggregateByKey(initialCount)(addToCounts, sumPartitionCounts)
```
For anyone who has worked with hadoop, this functionality is analogous to using combiners.  

### Results
Running our examples yields the following results:
```text
Aggregate By Key unique Results
bar -> C,D
foo -> B,A
------------------
Aggregate By Key sum Results
bar -> 3
foo -> 5
```


### Conclusion
This concludes our quick tour of the `aggregateByKey` function.  While the use of 3 functions can be a little unwieldly, it is certainly a good tool to have at your disposal.  In subsequent posts we will continue coverage of methods in  Spark's `PairRDDFunctions` class

### Resources

*   [Source Code](https://gist.github.com/bbejeck/b6b5c54130f5c607f1fa)
*   [PairRDD API](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions)
*   [Spark](http://spark.apache.org/docs/latest/index.html) 



