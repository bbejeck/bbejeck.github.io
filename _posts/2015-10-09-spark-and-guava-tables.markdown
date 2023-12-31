---
layout: post
title: "Spark And Guava Tables"
date: 2015-10-09 13:36:21 -0400
permalink: /spark-guava-tables
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
- Secondary Sorting
- Hadoop 
description: How to use Guava Tables as broadcast variables in Spark.
---
Last time we covered [Secondary Sorting](http://codingjunkie.net/spark-secondary-sort/) in [Spark](http://spark.apache.org/).  We took airline performance data and sorted results by airline, destination airport and the amount of delay.  We used id's for all our data. While that approach is good for performance, viewing results in that format loses meaning.  Fortunately, the [Bureau of Transportation](http://transtats.bts.gov/DL_SelectFields.asp?Table_ID=236&DB_Short_Name=On-Time) site offers reference files to download. The reference files are in CSV format with each line consisting of key-value pair.  Our goal is to store the refrence data in hashmaps and leverage [broadcast variables](http://spark.apache.org/docs/latest/programming-guide.html#broadcast-variables) so all operations on different partitions will have easy access to the same data.  We have four fields with codes: airline, origin city airport, orgin city, destination airport and destination city. Two of our code fields use the same reference file (airport id), so we'll need to download 3 files.  But is there an easier approch to loading 3 files into hashmaps and having 3 separate broadcast variables?  There is, by using [Guava Tables](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Table.html).
<!-- more -->
###Guava Tables in Brief
While a full discussion of the Guava `Table` is beyond the scope of this post, a brief description will be helpful.  It's basically an abstraction for a "hashmap of hashmaps", taking the boiler-plate out of adding or retieving data. For example:
```java
Map<String,Map<String,String>> outerMap = new HashMap<>();

Map<String,String> inner = outerMap.get("key");
//getting a value
if(inner == null){
    inner = new HashMap<>();
    outerMap.put("key",inner);
    return null;
}else{
   String value = inner.get("innerKey");
   return value; 
}

//adding a value
if(inner == null){
    inner = new HashMap<>();
    outerMap.put("key",inner);
}
 inner.put("innerKey","innerValue");  


Table<String,String,String> table = HashBasedTable.create();
//expected behavior if not found - returns null
String innerValue = table.get("key","innerKey");
//if no value exists for "key" hashmap is created.
table.put("key","innerKey","value")
```
Hopefully this example is enough to show why we'd want to use guava tables over the "hashmap of hashmaps" approach.
###Loading The Table
We have 3 files to load into our table for lookups.  The code for doing so is straight forward:
```scala
object GuavaTableLoader {

  //custom type for convenience
  type RefTable = Table[String, String, String]

  def load(path: String, filenames: List[String]): RefTable = {
    val lookupTable = HashBasedTable.create[String, String, String]()
    for (filename <- filenames) {
      val lines = Source.fromFile(path + "/" + filename).getLines()
      val baseFilename = filename.substring(0, filename.indexOf('.'))
      loadFileInTable(lines, baseFilename, lookupTable)
    }

    lookupTable
  }

  def load(path: String, filenames: String): RefTable = {
    val fileList = filenames.split(",").toList
    load(path, fileList)
  }

  private def loadFileInTable(lines: Iterator[String], rowKey: String, tb: RefTable): Unit = {
    for (line <- lines) {
      if (!line.trim().isEmpty) {
        val keyValue = line.split("#")
        tb.put(rowKey, keyValue(0), keyValue(1))
      }
    }
  }
}
```
The `load` method takes the base-path where the reference files are located and list of filenames (there is another `load` method that accepts a comma separated list of filenames).  We iterate over the list of filenames, re-using the basename as the "row-key" and then iterate over the key-value pairs found in the file storing them in the table. Here we are splitting the line on a '#' character.  The values in the reference data contained commas and were surrounded by quotes.  The files were cleaned up by removing the double quotes and changing the delimiter to a '#'.
###Setting the Guava Tables as a Broadcast Variables
Now we need to integrate the table object into our Spark job as a broadcast variable. For this we will re-use the `SecondarySort` object from the last post: 
```scala
    val dataPath = args(0)
    val refDataPath = args(1)
    val refDataFiles = args(2)

    val sc = context("SecondarySorting")
    val rawDataArray = sc.textFile(dataPath).map(line => line.split(","))

    val table = GuavaTableLoader.load(refDataPath, refDataFiles)

    val bcTable = sc.broadcast(table)
```
We've added two parameters, the base-path for the reference files and a comma separated list of reference file names.  After loading our table we create a broadcast variable with the `sc.broadcast` method call.
###Looking up the Reference Data
Now all we have left is to take the sorted results and convert all the id's to more meaningful names.
```scala
val keyedDataSorted = airlineData.repartitionAndSortWithinPartitions(new AirlineFlightPartitioner(5))

val translatedData = keyedDataSorted.map(t => createDelayedFlight(t._1, t._2, bcTable))
//printing out only done locally for demo purposes, usually write out to HDFS
    translatedData.collect().foreach(println)

//supporting code
def createDelayedFlight(key: FlightKey, data: List[String], bcTable: Broadcast[RefTable]): DelayedFlight = {
    val table = bcTable.value
    val airline = table.get(AIRLINE_DATA, key.airLineId)
    val destAirport = table.get(AIRPORT_DATA, key.arrivalAirportId.toString)
    val destCity = table.get(CITY_DATA, data(3))
    val origAirport = table.get(AIRPORT_DATA, data(1))
    val originCity = table.get(CITY_DATA, data(2))

    DelayedFlight(airline, data.head, origAirport, originCity, destAirport, destCity, key.arrivalDelay)
  }
```
Here we map the sorted results into `DelayedFlight` objects via the `createDelayedFlight` method.  There are a two things to take note of here:

 1.   To use the table object we need to first "unwrap" it from the `Broadcast` object.
 2.   The arrival airport id needs to be converted to a `String` as it's an int in the `FlightKey` class but our reference table contains only strings.
   
###Results
Now the results look like this:
```text
DelayedFlight(American Airlines Inc.,2015-01-01,Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,Atlanta, GA: Hartsfield-Jackson Atlanta International,Atlanta, GA (Metropolitan Area),-2.0)
DelayedFlight(American Airlines Inc.,2015-01-01,Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,Washington, DC: Ronald Reagan Washington National,Washington, DC (Metropolitan Area),-2.0)
DelayedFlight(American Airlines Inc.,2015-01-01,Los Angeles, CA: Los Angeles International,Los Angeles, CA (Metropolitan Area),Washington, DC: Ronald Reagan Washington National,Washington, DC (Metropolitan Area),-14.0)
DelayedFlight(American Airlines Inc.,2015-01-01,Chicago, IL: Chicago O'Hare International,Chicago, IL,Denver, CO: Denver International,Denver, CO,24.0)
DelayedFlight(American Airlines Inc.,2015-01-01,Ontario, CA: Ontario International,Los Angeles, CA (Metropolitan Area),Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,133.0)
DelayedFlight(American Airlines Inc.,2015-01-01,Honolulu, HI: Honolulu International,Honolulu, HI,Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,109.0)
DelayedFlight(American Airlines Inc.,2015-01-01,Phoenix, AZ: Phoenix Sky Harbor International,Phoenix, AZ,Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,55.0)
DelayedFlight(American Airlines Inc.,2015-01-01,New York, NY: John F. Kennedy International,New York City, NY (Metropolitan Area),Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,49.0)
DelayedFlight(American Airlines Inc.,2015-01-01,San Francisco, CA: San Francisco International,San Francisco, CA (Metropolitan Area),Dallas/Fort Worth, TX: Dallas/Fort Worth International,Dallas/Fort Worth, TX,40.0)
```
At quick glance and scrolling all the way to the right we can see that flights on this day into Dallas, TX had some sizable delays.
###Conclusion
That wraps up how we could use Guava Tables as broadcast variables in a Spark job.  Hopefully the reader can see the benefits of using such an approach.  Thanks for your time.
###Resources
*   [Source Code for post](https://github.com/bbejeck/spark-experiments/blob/master/src/main/scala-2.10/bbejeck/sorting/SecondarySort.scala)
*   [Guava Tables](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Table.html)
*   [OrderedRDDFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.OrderedRDDFunctions)
*   [Spark Scala API](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.package)
*   Cloudera Blog Post referencing [secondary sorting](http://blog.cloudera.com/blog/2015/03/how-to-tune-your-apache-spark-jobs-part-1/) 
*   [Department Of Transportation On-Time Flight Data](http://transtats.bts.gov/DL_SelectFields.asp?Table_ID=236&DB_Short_Name=On-Time)

