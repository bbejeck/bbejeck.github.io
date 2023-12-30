---
author: Bill Bejeck
comments: true
date: 2012-10-16 03:50:04+00:00
layout: post
slug: text-processing-with-mapreduce-part-2
title: Working Through Data-Intensive Text Processing with MapReduce - Local Aggregation
  Part II
wordpress_id: 2282
categories:
- Hadoop
- java
tags:
- Hadoop
- java
---

<img class="left" src="{{ site.media_url }}/images/hadoop-logo.jpeg" /> This post continues with the series on implementing algorithms found in the [Data Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) book. Part one can be found [here](http://codingjunkie.net/text-processing-with-mapreduce-part1/). In the previous post, we discussed using the technique of local aggregation as a means of reducing the amount of data shuffled and transferred across the network.  Reducing the amount of data transferred is one of the top ways to improve the efficiency of a MapReduce job. A word-count MapReduce job was used to demonstrate local aggregation.  Since the results only require a total count, we could re-use the same reducer for our combiner as changing the order or groupings of the addends will not affect the sum.  But what if you wanted an _average_? Then the same approach would not work because calculating an average of averages is not equal to the average of the original set of numbers. With a little bit of insight though, we can still use local aggregation.  For these examples we will be using a sample of the [NCDC weather dataset](https://github.com/tomwhite/hadoop-book/tree/master/input/ncdc/all) used in [Hadoop the Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520) book. We will calculate the average temperature for each month in the year 1901. The averages algorithm for the combiner and the in-mapper combining option can be found in chapter 3.1.3 of Data-Intensive Processing with MapReduce.
<!--more-->

### One Size Does Not Fit All


Last time we described two approaches for reducing data in a MapReduce job, Hadoop Combiners and the in-mapper combining approach.  Combiners are considered an optimization by the Hadoop framework and there are no guarantees on how many times they will be called, if at all.  As a result, mappers must emit data in the form expected by the reducers so if combiners aren't involved, the final result is not changed.  To adjust for calculating averages, we need to go back to the mapper and change it's output. 


### Mapper Changes


 In the word-count example, the non-optimized mapper simply emitted the word and the count of 1. The combiner and in-mapper combining mapper optimized this output by keeping each word as a key in a hash map with the total count as the value.  Each time a word was seen the count was incremented by 1.  With this setup, if the combiner was not called, the reducer would receive the word as a key and a long list of 1's to add together, resulting in the same output (of course using the in-mapper combining mapper avoided this issue because it's guaranteed to combine results as it's part of the mapper code). To compute an average, we will have our base mapper emit a string key (the year and month of the weather observation concatenated together) and a [custom writable](http://hadoop.apache.org/docs/r0.20.2/api/org/apache/hadoop/io/Writable.html) object, called TemperatureAveragingPair.  The [TemperatureAveragingPair](https://github.com/bbejeck/hadoop-algorithms/blob/master/src/bbejeck/mapred/aggregation/TemperatureAveragingPair.java) object will contain two numbers (IntWritables), the temperature taken and a count of one.  We will take the MaximumTemperatureMapper from Hadoop: The Definitive Guide and use it as inspiration for creating an AverageTemperatureMapper:

    
    
    public class AverageTemperatureMapper extends Mapper<LongWritable, Text, Text, TemperatureAveragingPair> {
     //sample line of weather data
     //0029029070999991901010106004+64333+023450FM-12+000599999V0202701N015919999999N0000001N9-00781+99999102001ADDGF10899199999999999
    
    
        private Text outText = new Text();
        private TemperatureAveragingPair pair = new TemperatureAveragingPair();
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
                outText.set(yearMonth);
                pair.set(temp, 1);
                context.write(outText, pair);
            }
        }
    }
    


By having the mapper output a key and TemperatureAveragingPair object our MapReduce program is guaranteed to have the correct results regardless if the combiner is called.


### Combiner


We need to reduce the amount of data sent, so we will sum the temperatures, and sum the counts and store them separately. By doing so we will reduce data sent, but preserve the format needed for calculating correct averages.  If/when the combiner is called, it will take all the TemperatureAveragingPair objects passed in and emit a single TemperatureAveragingPair object for the same key, containing the summed temperatures and counts.  Here is the code for the combiner: 
 
    
    
      public class AverageTemperatureCombiner extends Reducer<Text,TemperatureAveragingPair,Text,TemperatureAveragingPair> {
        private TemperatureAveragingPair pair = new TemperatureAveragingPair();
    
        @Override
        protected void reduce(Text key, Iterable<TemperatureAveragingPair> values, Context context) throws IOException, InterruptedException {
            int temp = 0;
            int count = 0;
            for (TemperatureAveragingPair value : values) {
                 temp += value.getTemp().get();
                 count += value.getCount().get();
            }
            pair.set(temp,count);
            context.write(key,pair);
        }
    }
     


But we are really interested in being guaranteed we have reduced the amount of data sent to the reducers, so we'll have a look at how to achieve that next.


### In Mapper Combining Averages


Similar to the word-count example, for calculating averages, the in-mapper-combining mapper will use a hash map with the concatenated year+month as a key and a TemperatureAveragingPair as the value. Each time we get the same year+month combination, we'll take the pair object out of the map, add the temperature and increase the count by by one.  Once the cleanup method is called we'll and emit all pairs with their respective key:

    
    
    public class AverageTemperatureCombiningMapper extends Mapper<LongWritable, Text, Text, TemperatureAveragingPair> {
     //sample line of weather data
     //0029029070999991901010106004+64333+023450FM-12+000599999V0202701N015919999999N0000001N9-00781+99999102001ADDGF10899199999999999
    
    
        private static final int MISSING = 9999;
        private Map<String,TemperatureAveragingPair> pairMap = new HashMap<String,TemperatureAveragingPair>();
    
    
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
                TemperatureAveragingPair pair = pairMap.get(yearMonth);
                if(pair == null){
                    pair = new TemperatureAveragingPair();
                    pairMap.put(yearMonth,pair);
                }
                int temps = pair.getTemp().get() + temp;
                int count = pair.getCount().get() + 1;
                pair.set(temps,count);
            }
        }
    
    
        @Override
        protected void cleanup(Context context) throws IOException, InterruptedException {
            Set<String> keys = pairMap.keySet();
            Text keyText = new Text();
            for (String key : keys) {
                 keyText.set(key);
                 context.write(keyText,pairMap.get(key));
            }
        }
    }
    


By following the same pattern of keeping track of data between map calls, we can achieve reliable data reduction by implementing an in-mapper combining strategy.  The same caveats apply for keeping state across all calls to the mapper, but looking at the gains that can be made in processing efficiency, using this approach merits some consideration.



### Reducer


At this point, writing our reducer is easy, take a list of pairs for each key, sum all the temperatures and counts then divide the sum of the temperatures by the sum of the counts.

    
    
    public class AverageTemperatureReducer extends Reducer<Text, TemperatureAveragingPair, Text, IntWritable> {
        private IntWritable average = new IntWritable();
    
        @Override
        protected void reduce(Text key, Iterable<TemperatureAveragingPair> values, Context context) throws IOException, InterruptedException {
            int temp = 0;
            int count = 0;
            for (TemperatureAveragingPair pair : values) {
                temp += pair.getTemp().get();
                count += pair.getCount().get();
            }
            average.set(temp / count);
            context.write(key, average);
        }
    }
    





### Results


The results are predictable with the combiner and in-mapper-combining mapper options showing substantially reduced data output.
Non-Optimized Mapper Option:

    
    
    12/10/10 23:05:28 INFO mapred.JobClient:     Reduce input groups=12
    12/10/10 23:05:28 INFO mapred.JobClient:     Combine output records=0
    12/10/10 23:05:28 INFO mapred.JobClient:     Map input records=6565
    12/10/10 23:05:28 INFO mapred.JobClient:     Reduce shuffle bytes=111594
    12/10/10 23:05:28 INFO mapred.JobClient:     Reduce output records=12
    12/10/10 23:05:28 INFO mapred.JobClient:     Spilled Records=13128
    12/10/10 23:05:28 INFO mapred.JobClient:     Map output bytes=98460
    12/10/10 23:05:28 INFO mapred.JobClient:     Total committed heap usage (bytes)=269619200
    12/10/10 23:05:28 INFO mapred.JobClient:     Combine input records=0
    12/10/10 23:05:28 INFO mapred.JobClient:     Map output records=6564
    12/10/10 23:05:28 INFO mapred.JobClient:     SPLIT_RAW_BYTES=108
    12/10/10 23:05:28 INFO mapred.JobClient:     Reduce input records=6564
    


Combiner Option:

    
    
    12/10/10 23:07:19 INFO mapred.JobClient:     Reduce input groups=12
    12/10/10 23:07:19 INFO mapred.JobClient:     Combine output records=12
    12/10/10 23:07:19 INFO mapred.JobClient:     Map input records=6565
    12/10/10 23:07:19 INFO mapred.JobClient:     Reduce shuffle bytes=210
    12/10/10 23:07:19 INFO mapred.JobClient:     Reduce output records=12
    12/10/10 23:07:19 INFO mapred.JobClient:     Spilled Records=24
    12/10/10 23:07:19 INFO mapred.JobClient:     Map output bytes=98460
    12/10/10 23:07:19 INFO mapred.JobClient:     Total committed heap usage (bytes)=269619200
    12/10/10 23:07:19 INFO mapred.JobClient:     Combine input records=6564
    12/10/10 23:07:19 INFO mapred.JobClient:     Map output records=6564
    12/10/10 23:07:19 INFO mapred.JobClient:     SPLIT_RAW_BYTES=108
    12/10/10 23:07:19 INFO mapred.JobClient:     Reduce input records=12
    


In-Mapper-Combining Option:

    
    
    12/10/10 23:09:09 INFO mapred.JobClient:     Reduce input groups=12
    12/10/10 23:09:09 INFO mapred.JobClient:     Combine output records=0
    12/10/10 23:09:09 INFO mapred.JobClient:     Map input records=6565
    12/10/10 23:09:09 INFO mapred.JobClient:     Reduce shuffle bytes=210
    12/10/10 23:09:09 INFO mapred.JobClient:     Reduce output records=12
    12/10/10 23:09:09 INFO mapred.JobClient:     Spilled Records=24
    12/10/10 23:09:09 INFO mapred.JobClient:     Map output bytes=180
    12/10/10 23:09:09 INFO mapred.JobClient:     Total committed heap usage (bytes)=269619200
    12/10/10 23:09:09 INFO mapred.JobClient:     Combine input records=0
    12/10/10 23:09:09 INFO mapred.JobClient:     Map output records=12
    12/10/10 23:09:09 INFO mapred.JobClient:     SPLIT_RAW_BYTES=108
    12/10/10 23:09:09 INFO mapred.JobClient:     Reduce input records=12
    


Calculated Results:
(NOTE: the temperatures in the sample file are in Celsius * 10)






  
Non-Optimized
Combiner
In-Mapper-Combiner Mapper


  


    

       190101	-25
190102	-91
190103	-49
190104	22
190105	76
190106	146
190107	192
190108	170
190109	114
190110	86
190111	-16
190112	-77
    


    

      190101	-25
190102	-91
190103	-49
190104	22
190105	76
190106	146
190107	192
190108	170
190109	114
190110	86
190111	-16
190112	-77
    


    

190101	-25
190102	-91
190103	-49
190104	22
190105	76
190106	146
190107	192
190108	170
190109	114
190110	86
190111	-16
190112	-77
    

  




### Conclusion


We have covered local aggregation for both the simple case where one could reuse the reducer as a combiner and a more complicated case where some insight on how to structure the data while still gaining the benefits of locally aggregating data for increase processing efficiency. 



### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


  * [Project Gutenberg](http://www.gutenberg.org/) a great source of books in plain text format, great for testing Hadoop jobs locally.


