---
author: Bill Bejeck
comments: true
date: 2013-01-14 14:00:50+00:00
layout: post
slug: secondary-sort
title: MapReduce Algorithms - Secondary Sorting
wordpress_id: 2599
categories:
- General
- Hadoop
- java
- MapReduce
tags:
- Hadoop
- java
- MapReduce
---

<img class="left" src="{{ site.media_url }}/images/hadoop-logo.jpeg" /> We continue with our series on implementing MapReduce algorithms found in Data-Intensive Text Processing with MapReduce book. Other posts in this series: 




  1. [Working Through Data-Intensive Text Processing with MapReduce](http://codingjunkie.net/text-processing-with-mapreduce-part1/)


  2. [Working Through Data-Intensive Text Processing with MapReduce – Local Aggregation Part II](http://codingjunkie.net/text-processing-with-mapreduce-part-2/)


  3. [Calculating A Co-Occurrence Matrix with Hadoop](http://codingjunkie.net/cooccurrence/)


  4. [MapReduce Algorithms – Order Inversion](http://codingjunkie.net/order-inversion/)


This post covers the pattern of secondary sorting, found in chapter 3 of [Data-Intensive Text Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421).  While Hadoop automatically sorts data emitted by mappers before being sent to reducers, what can you do if you also want to sort by value? You use secondary sorting of course.  With a slight manipulation to the format of the key object, secondary sorting gives us the ability to take the value into account during the sort phase. There are two possible approaches here.  The first approach involves having the reducer buffer all of the values for a given key and do an in-reducer sort on the values.  Since the reducer will be receiving all values for a given key, this approach could possibly cause the reducer to run out of memory.  The second approach involves creating a composite key by adding a part of, or the entire value to the natural key to achieve your sorting objectives.  The trade off between these two approaches is doing an explicit sort on values in the reducer would most likely be faster(at the risk of running out of memory) but implementing a "value to key" conversion approach,  is offloading the sorting the MapReduce framework, which lies at the heart of what Hadoop/MapReduce is designed to do.  For the purposes of this post, we will consider the "value to key" approach.  We will need to write a custom partitioner to ensure all the data with same key (the natural key not including the composite key with the value) is sent to the same reducer and a custom Comparator so the data is grouped by the natural key once it arrives at the reducer.  
<!--more-->

### Value to Key Conversion


Creating a composite key is straight forward.  What we need to do is analyze what part(s) of the value we want to account for during the sort and add the appropriate part(s) to the natural key.  Then we need to work on the compareTo method either in key class, or comparator class to make sure the composite key is accounted. We will be re-visiting the weather data set and include the temperature as part of the natural key (the natural key being the year and month concatenated together).  The result will be a listing of the coldest day for a given month and year.  This example was inspired from the secondary sorting example found in [Hadoop, The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=dp_ob_title_bk) book.  While there are probably better ways to achieve this objective, but it will be good enough to demonstrate how secondary sorting works.


### Mapper Code


Our mapper code already concatenates the year and month together, but we will also include the temperature as part of the key.  Since we have included the value in the key itself, the mapper will emit a NullWritable, where in other cases we would emit the temperature.

    
    
    public class SecondarySortingTemperatureMapper extends Mapper<LongWritable, Text, TemperaturePair, NullWritable> {
    
        private TemperaturePair temperaturePair = new TemperaturePair();
        private NullWritable nullValue = NullWritable.get();
        private static final int MISSING = 9999;
    @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            String line = value.toString();
            String yearMonth = line.substring(15, 21);
    
            int tempStartPosition = 87;
    
            if (line.charAt(tempStartPosition) == '+') {
                tempStartPosition += 1;
            }
    
            int temp = Integer.parseInt(line.substring(tempStartPosition, 92));
    
            if (temp != MISSING) {
                temperaturePair.setYearMonth(yearMonth);
                temperaturePair.setTemperature(temp);
                context.write(temperaturePair, nullValue);
            }
        }
    }
    


Now we have added the temperature to the key, we set the stage for enabling secondary sorting.  What's left to do is write code taking temperature into account when necessary.  Here we have two choices, write a Comparator or adjust the compareTo method on the TemperaturePair class (TemperaturePair implements WritableComparable).  In most cases I would recommend writing a separate Comparator, but the TemperaturePair class was written specifically to demonstrate secondary sorting, so we will modify the TemperaturePair class compareTo method. 

    
    
     @Override
        public int compareTo(TemperaturePair temperaturePair) {
            int compareValue = this.yearMonth.compareTo(temperaturePair.getYearMonth());
            if (compareValue == 0) {
                compareValue = temperature.compareTo(temperaturePair.getTemperature());
            }
            return compareValue;
        }
    


If we wanted to sort in descending order, we could simply multiply the result of the temperature comparison by a -1.
Now that we have completed the part necessary for sorting, we need to write a custom partitioner. 


### Partitoner Code


To ensure only the natural key is considered when determining which reducer to send the data to, we need to write a custom partitioner.  The code is straight forward and only considers the yearMonth value of the TemperaturePair class when calculating the reducer the data will be sent to.

    
    
    public class TemperaturePartitioner extends Partitioner<TemperaturePair, NullWritable>{
        @Override
        public int getPartition(TemperaturePair temperaturePair, NullWritable nullWritable, int numPartitions) {
            return temperaturePair.getYearMonth().hashCode() % numPartitions;
        }
    }
    


While the custom partitioner guarantees that all of the data for the year and month arrive at the same reducer, we still need to account for the fact the reducer will group records by key.  


### Grouping Comparator


Once the data reaches a reducer, all data is grouped by key.  Since we have a composite key, we need to make sure records are grouped solely by the natural key.  This is accomplished by writing a custom GroupPartitioner. We have a Comparator object only considering the yearMonth field of the TemperaturePair class for the purposes of grouping the records together.

    
    
    public class YearMonthGroupingComparator extends WritableComparator {
        public YearMonthGroupingComparator() {
            super(TemperaturePair.class, true);
        }
        @Override
        public int compare(WritableComparable tp1, WritableComparable tp2) {
            TemperaturePair temperaturePair = (TemperaturePair) tp1;
            TemperaturePair temperaturePair2 = (TemperaturePair) tp2;
            return temperaturePair.getYearMonth().compareTo(temperaturePair2.getYearMonth());
        }
    }
    




### Results


Here are the results of running our secondary sort job:

    
    
    new-host-2:sbin bbejeck$ hdfs dfs -cat secondary-sort/part-r-00000
    190101	-206
    190102	-333
    190103	-272
    190104	-61
    190105	-33
    190106	44
    190107	72
    190108	44
    190109	17
    190110	-33
    190111	-217
    190112	-300
    




### Conclusion


While sorting data by value may not be a common need, it's a nice tool to have in your back pocket when needed.  Also, we have been able to take a deeper look at the inner workings of Hadoop by working with custom partitioners and group partitioners.  Thank you for your time.


### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code and Tests](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


