---
author: Bill Bejeck
comments: true
date: 2012-09-26 04:04:13+00:00
layout: post
slug: text-processing-with-mapreduce-part1
title: Working Through Data-Intensive Text Processing with MapReduce
wordpress_id: 2179
categories:
- Hadoop
tags:
- Hadoop
- java
---

<img class="left" src="{{ site.media_url }}/images/hadoop-logo.jpeg" /> It has been a while since I last posted, as I've been busy with some of the classes offered by [Coursera](https://www.coursera.org/).  There are some very interesting offerings and is worth a look. Some time ago, I purchased [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer.  The book presents several key MapReduce algorithms, but in pseudo code format.  My goal is to take the algorithms presented in chapters 3-6 and implement them in Hadoop,  using [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White as a reference.   I'm going to assume familiarity with Hadoop and MapReduce and not cover any introductory material.  So let's jump into chapter 3 - MapReduce Algorithm Design, starting with local aggregation.
<!--more-->

## Local Aggregation


At a very high level, when Mappers emit data, the intermediate results are written to disk then sent across the network to Reducers for final processing.  The latency of writing to disk then transferring data across the network is an expensive operation in the processing of a MapReduce job.  So it stands to reason that whenever possible, reducing the amount of data sent from mappers would increase the speed of the MapReduce job.  Local aggregation is a technique used to reduce the amount of data and improve the efficiency of our MapReduce job.  Local aggregation can not take the place of reducers, as we need a way to gather results with the same key from different mappers.  We are going to consider 3 ways of achieving local aggregation:




  1. Using Hadoop Combiner functions.


  2. Two approaches of "in-mapper" combining presented in the Text Processing with MapReduce book.


Of course any optimization is going to have tradeoffs and we'll discuss those as well.  
To demonstrate local aggregation, we will run the ubiquitous word count job on a plain text version of [A Christmas Carol](http://www.gutenberg.org/cache/epub/46/pg46.txt) by Charles Dickens (downloaded from [Project Gutenberg](http://www.gutenberg.org/wiki/Main_Page)) on a pseudo distributed cluster installed on my MacBookPro, using the hadoop-0.20.2-cdh3u3 distribution from [Cloudera](http://www.cloudera.com/).  I plan in a future post to run the same experiment on an EC2 cluster with more realistic sized data.



### Combiners


A combiner function is an object that extends the Reducer class. In fact, for our examples here, we are going to re-use the same reducer used in the word count job.  A combiner function is specified when setting up the MapReduce job like so:

    
    
     job.setReducerClass(TokenCountReducer.class);
    


Here is the reducer code:

    
    
    public class TokenCountReducer extends Reducer<Text,IntWritable,Text,IntWritable>{
        @Override
        protected void reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
            int count = 0;
            for (IntWritable value : values) {
                  count+= value.get();
            }
            context.write(key,new IntWritable(count));
        }
    }
    

 
The job of a combiner is to do just what the name implies, aggregate data with the net result of less data begin shuffled across the network, which gives us gains in efficiency.  As stated before, keep in mind that reducers are still required to put together results with the same keys coming from different mappers.  Since combiner functions are an optimization, the Hadoop framework offers no guarantees on how many times a combiner will be called, if at all.


### In Mapper Combining Option 1


The first alternative to using Combiners (figure 3.2 page 41) is very straight forward and makes a slight modification to our original word count mapper:

    
    
    public class PerDocumentMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            IntWritable writableCount = new IntWritable();
            Text text = new Text();
            Map<String,Integer> tokenMap = new HashMap<String, Integer>();
            StringTokenizer tokenizer = new StringTokenizer(value.toString());
    
            while(tokenizer.hasMoreElements()){
                String token = tokenizer.nextToken();
                Integer count = tokenMap.get(token);
                if(count == null) count = new Integer(0);
                count+=1;
                tokenMap.put(token,count);
            }
    
            Set<String> keys = tokenMap.keySet();
            for (String s : keys) {
                 text.set(s);
                 writableCount.set(tokenMap.get(s));
                 context.write(text,writableCount);
            }
        }
    }
    


As we can see here, instead of emitting a word with the count of 1, for each word encountered, we use a map to keep track of each word already processed. Then when all of the tokens are processed we loop through the map and emit the total count for each word encountered in that line.


### In Mapper Combining Option 2


The second option of in mapper combining (figure 3.3 page 41) is very similar to the above example with two distinctions - when the hash map is created and when we emit the results contained in the map.  In the above example, a map is created and has its contents dumped over the wire for _each_ invocation of the map method.  In this example we are going make the map an instance variable and shift the instantiation of the map to the setUp method in our mapper.  Likewise the contents of the map will not be sent out to the reducers until all of the calls to mapper have completed and the cleanUp method is called.

    
    
    public class AllDocumentMapper extends Mapper<LongWritable,Text,Text,IntWritable> {
    
        private  Map<String,Integer> tokenMap;
    
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
               tokenMap = new HashMap<String, Integer>();
        }
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            StringTokenizer tokenizer = new StringTokenizer(value.toString());
            while(tokenizer.hasMoreElements()){
                String token = tokenizer.nextToken();
                Integer count = tokenMap.get(token);
                if(count == null) count = new Integer(0);
                count+=1;
                tokenMap.put(token,count);
            }
        }
    
    
        @Override
        protected void cleanup(Context context) throws IOException, InterruptedException {
            IntWritable writableCount = new IntWritable();
            Text text = new Text();
            Set<String> keys = tokenMap.keySet();
            for (String s : keys) {
                text.set(s);
                writableCount.set(tokenMap.get(s));
                context.write(text,writableCount);
            }
        }
    }
    


As we can see from the above code example, the mapper is keeping track of unique word counts, across all calls to the map method.  By  keeping track of unique tokens and their counts, there should be a substantial reduction in the number of records sent to the reducers, which in turn should improve the running time of the MapReduce job.  This accomplishes the same effect as using the combiner function option provided by the MapReduce framework, but in this case you are guaranteed that the combining code will be called.  But there are some caveats with this approach also.  Keeping state across map calls could prove problematic and definitely is a violation of the functional spirit of a "map" function.  Also, by keeping state across all mappers, depending on the data used in the job, memory could be another issue to contend with.  Ultimately, one would have to weigh all of the trade offs to determine the best approach.


#### Results


Now lets take a look at the some results of the different mappers.  Since the job was run in pseudo-distributed mode, actual running times are irrelevant, but we can still infer how using local aggregation could impact the efficiency of MapReduce job running on a real cluster.  

Per Token Mapper:

    
    
    12/09/13 21:25:32 INFO mapred.JobClient:     Reduce shuffle bytes=366010
    12/09/13 21:25:32 INFO mapred.JobClient:     Reduce output records=7657
    12/09/13 21:25:32 INFO mapred.JobClient:     Spilled Records=63118
    12/09/13 21:25:32 INFO mapred.JobClient:     Map output bytes=302886
    


In Mapper Reducing Option 1:

    
    
    12/09/13 21:28:15 INFO mapred.JobClient:     Reduce shuffle bytes=354112
    12/09/13 21:28:15 INFO mapred.JobClient:     Reduce output records=7657
    12/09/13 21:28:15 INFO mapred.JobClient:     Spilled Records=60704
    12/09/13 21:28:15 INFO mapred.JobClient:     Map output bytes=293402
    


In Mapper Reducing Option 2:

    
    
    12/09/13 21:30:49 INFO mapred.JobClient:     Reduce shuffle bytes=105885
    12/09/13 21:30:49 INFO mapred.JobClient:     Reduce output records=7657
    12/09/13 21:30:49 INFO mapred.JobClient:     Spilled Records=15314
    12/09/13 21:30:49 INFO mapred.JobClient:     Map output bytes=90565
    


Combiner Option:

    
    
    12/09/13 21:22:18 INFO mapred.JobClient:     Reduce shuffle bytes=105885
    12/09/13 21:22:18 INFO mapred.JobClient:     Reduce output records=7657
    12/09/13 21:22:18 INFO mapred.JobClient:     Spilled Records=15314
    12/09/13 21:22:18 INFO mapred.JobClient:     Map output bytes=302886
    12/09/13 21:22:18 INFO mapred.JobClient:     Combine input records=31559
    12/09/13 21:22:18 INFO mapred.JobClient:     Combine output records=7657
    


As expected the Mapper that did no combining had the worst results, followed closely by the first in-mapper combining option (although these results could have been made better had the data been cleaned up before running the word count). The second in-mapper combining option and the combiner function had virtually identical results.  The significant fact is that both produced **_2/3 less_** reduce shuffle bytes as the first two options.  Reducing the amount of bytes sent over the network to the reducers by that amount would surely would have a positive impact on the efficiency of a MapReduce job.  There is one point to keep in mind here and that is Combiners/In-Mapper combining can not just be used in all MapReduce jobs, in this case the word count lends itself very nicely to such an enhancement, but that might not always be true.



### Conclusion


As you can see the benefits of using either in-mapper combining or the Hadoop combiner function require serious consideration when looking to improve the performance of your MapReduce jobs. As for which approach, it is up to you the weigh the trade offs for each approach.



### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


  * [Project Gutenberg](http://www.gutenberg.org/) a great source of books in plain text format, great for testing Hadoop jobs locally.


