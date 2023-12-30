---
layout: post
title: "Learning Scala Implicits with Spark"
date: 2015-10-16 13:17:02 -0400
permalink: /learning-scala-implicits-with-spark/
comments: true
categories: 
- Scala
- Spark
- MapReduce
- Hadoop
tags: 
- Scala
- Spark
keywords: 
- Scala
- Spark
- Implicits
- Type Classes
- Secondary Sorting
- Hadoop 
description: How to use Implicit classes to add functionality in Spark.
---
A while back I wrote two posts on avoiding the use of the `groupBy` function in Spark.  While I won't re-hash both posts here, the bottom line was to take advantage of the [combineByKey](http://codingjunkie.net/spark-combine-by-key/) or [aggreagateByKey](http://codingjunkie.net/spark-agr-by-key/) functions instead.  While both functions hold the potential for improved performance and efficiency in our Spark jobs, at times creating the required arguments over and over for basic use cases could get tedious.  It got me to thinking is there a way of providing some level of abstraction for basic use cases?  For example grouping values into a list or set.  Simultaneously, I've been trying to expand my knowlege of [Scala's](http://www.scala-lang.org/) more advanced features including [implicits](http://stackoverflow.com/questions/10375633/understanding-implicit-in-scala) and [TypeClasses](https://www.safaribooksonline.com/blog/2013/05/28/scala-type-classes-demystified/).  What I came up with is the [GroupingRDDFunctions](https://github.com/bbejeck/spark-experiments/blob/master/src/main/scala-2.10/bbejeck/implicits/GroupingRDDUtils.scala) class that provides some syntactic sugar for basic use cases of the `aggregateByKey` function by using Scala's [implicit class](http://docs.scala-lang.org/overviews/core/implicit-classes.html) functionality.
<!--more-->
###Scala Implicits in Brief
While a full explanation of Scala's implicits is beyond the scope of this post, here's a quick description.  When the Scala compiler finds a variable or expression of the wrong type, it will look for an `implicit` function, expression or class to provide the correct type.  The `implicit` function (or class) needs to be in the current scope for the compiler to do it's work.  This is typically accomplished by importing a Scala object that contains the implicit definition(s).  In this case the `GroupingRDDFunctions` class is wrapped in the `GroupingRDDUtils` object.  Here's the class declaration:
```scala GroupingRDDFunctions Declaration
object GroupingRDDUtils {
  implicit class GroupingRDDFunctions[K: ClassTag, V: ClassTag](self: RDD[(K, V)]) extends Logging with Serializable {
//... details left out for clarity
   }
}
```
To use GroupingRDDFunctions just use the following import statement:
```scala Import Statement
import bbejeck.implicits.GroupingRDDUtils._
```
###Provided functionality

The methods defined on `GroupingRDDFunctions` are:

 1.   groupByKeyToList
 2.   groupByKeyUnique
 3.   countByKey
 4.   sumWithTotal - provides a tuple with the sumation of numeric value as a Double along with the total count of items to create the sum
 5.   averageByKey
    
Here's some examples of using `GroupingRDDFunctions`:
```scala Examples
//To do a grouping of unique values by key using aggregateByKey
val initialSet = mutable.HashSet.empty[String]
val addToSet = (s: mutable.HashSet[String], v: String) => s += v
val mergePartitionSets = (p1: mutable.HashSet[String], p2: mutable.HashSet[String]) => p1 ++= p2
val uniqueByKey = kv.aggregateByKey(initialSet)(addToSet, mergePartitionSets)

//Now can be accomplished doing
val uniqueByKey = kv.groupByKeyUnique()

//Computing Count By Key with aggregateByKey
val initialCount = 0;
val addToCounts = (n: Int, v: String) => n + 1
val sumPartitionCounts = (p1: Int, p2: Int) => p1 + p2
val counts = kv.aggregateByKey(initialCount)(addToCounts, sumPartitionCounts) 

//Now becomes
val counts = kv.countByKey()
```   
There is really nothing special happening here.  We are simply wrapping an RDD instance and providing the ability to use the methods listed above on that RDD instance.  Within the `GroupingRDDFunctions` class we are still leveraging the `aggregateByKey` function. 

###Implicit Parameter Conversion
As another example of implicit useage let's take a look at the `averageByKey` function.  In the code below, we compute the average by key by applying the `averagingFunction` to the results returned from the `sumWithTotal` method.  But if we look closely, our keys and values are generics of 'K' and 'V', but all of these functions work on doubles.
```scala Implicit Function Example

implicit def intToDouble(num: V): Double = {
        num match {
          case i: Int => i.toDouble
          case _ => num.asInstanceOf[Double]
        }
}

def sumWithTotal(): RDD[(K, (Int, Double))] = {
      self.aggregateByKey((0, 0.0))(incrementCountSumValue, sumTuples)
}

def averageByKey(): RDD[(K, Double)] = {
    self.sumWithTotal().map(t => averagingFunction(t))

}

private def averagingFunction(t: (K, (Int, Double))): (K, Double) = {
      val (name, (numberScores, totalScore)) = t
      (name, totalScore / numberScores)
}

private def incrementCountSumValue(t: (Int, Double), v: V): (Int, Double) = {
      (t._1 + 1, t._2 + v)
}

private def sumTuples(t: (Int, Double), t2: (Int, Double)): (Int, Double) = {
      val (numScores1, totalScore1) = t
      val (numScores2, totalScore2) = t2
      (numScores1 + numScores2, totalScore1 + totalScore2)
}
```
So what happens if the values provided are integers instead of doubles?  Also take a look at the `incrementCountSumValue` method, how can a value of type 'V' be added to the double value of the tuple?  This is good example of using an implicit function.  The compiler will look for and find the `intToDouble` function and apply to the parameter of the `incrementCountSumValue` method.  If the value is an integer, it's implicity converted to a double, otherwise we return a double. 

###Conclusion
While the use of implicits in Scala needs to be judicious, the example presented here represents a good use-case in my opinion.  We are adding some useful behavior to a class, just by adding an import statement.  Plus it's very easy to inspect the implicit class to see what's going on under the covers.

###References

*   [Scala Doc Description of Implicit Classes](http://docs.scala-lang.org/overviews/core/implicit-classes.html)
*   [Good Explanation of Implicit Precedence](http://eed3si9n.com/implicit-parameter-precedence-again)
*   [Good Explanation of Implicits and Type Classes](http://www.cakesolutions.net/teamblogs/demystifying-implicits-and-typeclasses-in-scala)
*   [Neophytes guide to Scala - Implicits, Type Classes](http://danielwestheide.com/blog/2013/02/06/the-neophytes-guide-to-scala-part-12-type-classes.html)
*   [tresata/spark-sorted](https://github.com/tresata/spark-sorted) uses implicits to add functionality.
*   [GroupingRDDFunctions source code](https://github.com/bbejeck/spark-experiments/blob/master/src/main/scala-2.10/bbejeck/implicits/GroupingRDDUtils.scala)
*   [GroupingRDDFunctions unit test](https://github.com/bbejeck/spark-experiments/blob/master/src/test/scala-2.10/bbejeck/implicits/GroupingRDDFunctionsTest.scala)

 