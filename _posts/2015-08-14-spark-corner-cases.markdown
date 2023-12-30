---
layout: post
title: "Spark Corner Cases"
date: 2015-08-14 09:53:36 -0400
permalink: /spark-corner-cases
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
description: Looking at solutions to issues with Spark not encountered on a daily basis.
---
In the last two posts, we covered alternatives to using the `groupByKey` method, `aggregateByKey` and `combineByKey`.  In this post we are going to consider methods/situtaions you might not encounter for your everyday Spark job, but will come in handy when the need arises.

###Stripping The First Line (or First N Lines) of a File
While doing some testing, I wanted to strip off the first line of a file.  Granted I could have easily created a copy with the header removed, but curiosity got the better of me.  Just how would I remove the first line of a file?  The answer is to use the `mapPartitionsWithIndex` method.
<!-- more -->
```scala Stripping First Line of a File example
val inputData = sc.parallelize(Array("Skip this line XXXXX","Start here instead AAAA","Second line to work with BBB"))

val valuesRDD = inputData.mapPartitionsWithIndex((idx, iter) => skipLines(idx, iter, numLinesToSkip))

def skipLines(index: Int, lines: Iterator[String], num: Int): Iterator[String] = {
    if (index == 0) {
      lines.drop(num)
    }
    lines
  } 
``` 
Here are the results from printing the resulting RDD
```text Printing Results
Start here instead AAAA
Second line to work with BBB
```
The `mapPartitionsWithIndex` method passes the index (zero based) of the partition and an iterator containing the data to a user provided function (here `skipLines`).  Using this example we could remove any number lines or work with the entire partition at one time (but that's the subject of another post!)

###Avoiding Lists of Iterators
Often when reading in a file, we want to work with the individual values contained in each line separated by some delimiter.  Splitting a delimited line is a trivial operation:
```scala Splitting Lines with Coma Separted Values
newRDD = textRDD.map(line => line.split(","))
```
But the issue here is the returned RDD will be an iterator composed of iterators.  What we want is the individual values obtained after calling the `split` function. In other words, we need an`Array[String]` not an `Array[Array[String]]`.  For this we would use the `flatMap` function.  For those with a functional programming background, using a flatMap operation is nothing new. But if you are new to functional programming it's a great operation to become familiar with.
```scala Avoiding Iterator of Iterators
val inputData = sc.parallelize (Array ("foo,bar,baz", "larry,moe,curly", "one,two,three") ).cache ()

val mapped = inputData.map (line => line.split (",") )
val flatMapped = inputData.flatMap (line => line.split (",") )

val mappedResults = mapped.collect ()
val flatMappedResults = flatMapped.collect ();

println ("Mapped results of split")
println (mappedResults.mkString (" : ") )

println ("FlatMapped results of split")
println (flatMappedResults.mkString (" : ") )
```
When we run the program we see these results:
```text Results
Mapped results of split
[Ljava.lang.String;@45e22def : [Ljava.lang.String;@6ae3fb94 : [Ljava.lang.String;@4417af13
FlatMapped results of split
foo : bar : baz : larry : moe : curly : one : two : three
```
As we can see the `map` example returned an `Array` containing 3 `Array[String]` instances, while the `flatMap` call returned individual values contained in one `Array`.

###Maintaining An RDD's Original Partitoning 
When working in Spark, you quickly come up to speed with the `map` function.  It's used to take a collection of values and 'map' them into another type.  Typically when working with key-value pairs, we don't need the key to remain the same type, if we need to keep the key at all.  However, there are cases where we'll need to keep the original RDD partitioning.  This presents a problem; when calling the `map` function there is no guarantee that the key has not been modified.  What's our option? We use the `mapValues` function (on the [PairRDDFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions) class) which allows us to change the values and retain the original key and partitioning of the RDD.
```scala Mapping Values Only
//Details left out for clarity
val data = sc.parallelize(Array( (1,"foo"),(2,"bar"),(3,"baz"))).cache()
val mappedRdd = data.map(x => (x._1,x._2.toUpperCase))
val mappedValuesRdd = data.mapValues(s => s.toUpperCase)

println("Mapping Results")
printResults(mappedRdd.collect())
printResults(mappedValuesRdd.collect())
```
And we'll see the results from our simple example are exactly the same:
```text Results
Mapping Results
(1,FOO) -> (2,BAR) -> (3,BAZ)
(1,FOO) -> (2,BAR) -> (3,BAZ)
```
###Conclusion 
This has been a short tour of some methods we can use when we encounter an unfamiliar situation writing Spark jobs.  Thanks for your time.

###Resources
*   [Source Code](https://github.com/bbejeck/spark-experiments)
*   [Spark Scala API](http://spark.apache.org/docs/latest/api/scala/index.html#package)
*   [Learning Spark](http://shop.oreilly.com/product/0636920028512.do)
*   [Spark](http://spark.apache.org/docs/latest/index.html)

    