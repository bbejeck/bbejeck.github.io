---
author: Bill Bejeck
comments: true
date: 2013-06-26 12:45:11+00:00
layout: post
slug: mapreduce-reduce-joins
title: MapReduce Algorithms - Understanding Data Joins Part 1
wordpress_id: 2664
categories:
- Hadoop
- java
- MapReduce
tags:
- Hadoop
- java
- MapReduce
---

<img class="left" src="{{ site.media_url }}/images/hadoop-logo.jpeg" />  In this post we continue with our series of implementing the algorithms found in the [Data-Intensive Text Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) book, this time discussing data joins.  While we are going to discuss the techniques for joining data in Hadoop and provide sample code, in most cases you probably won't be writing code to perform joins yourself.  Instead, joining data is better accomplished using tools that work at a higher level of abstraction such as Hive or Pig.  Why take the time to learn how to join data if there are tools that can take care of it for you?  Joining data is arguably one of the biggest uses of Hadoop. Gaining a full understanding of how Hadoop performs joins is critical for deciding which join to use and for debugging when trouble strikes.  Also, once you fully understand how different joins are performed in Hadoop, you can better leverage tools like Hive and Pig. Finally, there might be the one off case where a tool just won't get you what you need and you'll have to roll up your sleeves and write the code yourself. 
<!--more-->

### The Need for Joins


When processing large data sets the need for joining data by a common key can be very useful, if not essential.   By joining data you can further gain insight such as joining with timestamps to correlate events with a time a day.  The need for joining data are many and varied.   We will be covering 3 types of joins, Reduce-Side joins, Map-Side joins  and the Memory-Backed Join over 3 separate posts.  This installment we will consider working with Reduce-Side joins.


### Reduce Side Joins


Of the join patterns we will discuss, reduce-side joins are the easiest to implement.  What makes reduce-side joins straight forward is the fact that Hadoop sends identical keys to the same reducer, so by default the data is organized for us.  To perform the join, we simply need to cache a key and compare it to incoming keys. As long as the keys match, we can join the values from the corresponding keys. The trade off with reduce-side joins is performance, since all of the data is shuffled across the network. Within reduce-side joins there are two different scenarios we will consider: one-to-one and one-to-many.  We'll also explore options where we don't need to keep track of the incoming keys; all values for a given key will be grouped together in the reducer.  


### One-To-One Joins


 A one-to-one join is the case where a value from dataset 'X' shares a common key with a value from dataset 'Y'.  Since Hadoop guarantees that equal keys are sent to the same reducer, mapping over the two datasets will take care of the join for us.  Since sorting only occurs for keys, the order of the values is unknown.  We can easily fix the situation by using [secondary sorting](http://codingjunkie.net/secondary-sort/). Our implementation of secondary sorting will be to tag keys with either a "1" or a "2" to determine order of the values. We need to take a couple extra steps to implement our tagging strategy.


##### Implementing a WritableComparable


First we need to write a class that implements the WritableComparable interface that will be used to wrap our key. 

    
    
    public class TaggedKey implements Writable, WritableComparable<TaggedKey> {
    
        private Text joinKey = new Text();
        private IntWritable tag = new IntWritable();
        
        @Override
        public int compareTo(TaggedKey taggedKey) {
            int compareValue = this.joinKey.compareTo(taggedKey.getJoinKey());
            if(compareValue == 0 ){
                compareValue = this.tag.compareTo(taggedKey.getTag());
            }
           return compareValue;
        }
       //Details left out for clarity
     }
    


When our TaggedKey class is sorted, keys with the same `joinKey` value will have a secondary sort on the value of the `tag` field, ensuring the order we want.


##### Writing a Custom Partitioner


Next we need to write a custom partitioner that will only consider the join key when determining which reducer the composite key and data are sent to:

    
    
    public class TaggedJoiningPartitioner extends Partitioner<TaggedKey,Text> {
    
        @Override
        public int getPartition(TaggedKey taggedKey, Text text, int numPartitions) {
            return taggedKey.getJoinKey().hashCode() % numPartitions;
        }
    }
    


At this point we have what we need to join the data and ensure the order of the values. But we don't want to keep track of the keys as they come into the `reduce()` method.  We want all the values grouped together for us. To accomplish this we will use a `Comparator` that will consider only the join key when deciding how to group the values.


##### Writing a Group Comparator


Our Comparator used for grouping will look like this:

    
    
    public class TaggedJoiningGroupingComparator extends WritableComparator {
    
    
        public TaggedJoiningGroupingComparator() {
            super(TaggedKey.class,true);
        }
    
        @Override
        public int compare(WritableComparable a, WritableComparable b) {
            TaggedKey taggedKey1 = (TaggedKey)a;
            TaggedKey taggedKey2 = (TaggedKey)b;
            return taggedKey1.getJoinKey().compareTo(taggedKey2.getJoinKey());
        }
    }
    




### Structure of the data


Now we need to determine what we will use for our key to join the data. For our sample data we will be using a CSV file generated from the [Fakenames Generator](http://www.fakenamegenerator.com/order.php). The first column is a GUID and that will serve as our join key.  Our sample data contains information like name, address, email, job information, credit cards and automobiles owned.  For the purposes of our demonstration we will take the GUID, name and address fields and place them in one file that will be structured like this:

cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Esther,Garner,4071 Haven Lane,Okemos,MI
81a43486-07e1-4b92-b92b-03d0caa87b5f,Timothy,Duncan,753 Stadium Drive,Taunton,MA
aef52cf1-f565-4124-bf18-47acdac47a0e,Brett,Ramsey,4985 Shinn Street,New York,NY

Then we will take the GUID, email address, username, password and credit card fields and place then in another file that will look like:

cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,517-706-9565,EstherJGarner@teleworm.us,Waskepter38,noL2ieghie,MasterCard,5305687295670850
81a43486-07e1-4b92-b92b-03d0caa87b5f,508-307-3433,TimothyDDuncan@einrot.com,Conerse,Gif4Edeiba,MasterCard,5265896533330445
aef52cf1-f565-4124-bf18-47acdac47a0e,212-780-4015,BrettMRamsey@dayrep.com,Subjecall,AiKoiweihi6,MasterCard,5243379373546690

Now we need to have a Mapper that will know how to work with our data to extract the correct key for joining and also set the proper tag.


##### Creating the Mapper


Here is our Mapper code:

    
    
    public class JoiningMapper extends Mapper<LongWritable, Text, TaggedKey, Text> {
    
        private int keyIndex;
        private Splitter splitter;
        private Joiner joiner;
        private TaggedKey taggedKey = new TaggedKey();
        private Text data = new Text();
        private int joinOrder;
    
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
            keyIndex = Integer.parseInt(context.getConfiguration().get("keyIndex"));
            String separator = context.getConfiguration().get("separator");
            splitter = Splitter.on(separator).trimResults();
            joiner = Joiner.on(separator);
            FileSplit fileSplit = (FileSplit)context.getInputSplit();
            joinOrder = Integer.parseInt(context.getConfiguration().get(fileSplit.getPath().getName()));
        }
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            List<String> values = Lists.newArrayList(splitter.split(value.toString()));
            String joinKey = values.remove(keyIndex);
            String valuesWithOutKey = joiner.join(values);
            taggedKey.set(joinKey, joinOrder);
            data.set(valuesWithOutKey);
            context.write(taggedKey, data);
        }
    
    }
    


Let's review what is going on in the `setup()` method.  




  1. First we get the index of our join key and the separator used in the text from values set in the Configuration when the job was launched.


  2. Then we create a [Guava Splitter](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Splitter.html) used to split the data on the separator we retrieved from the call to `context.getConfiguration().get("separator")`.  We also create a [Guava Joiner](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Joiner.html) used to put the data back together once the key has been extracted.


  3. Next we get the name of the file that this mapper will be processing.  We use the filename to pull the join order for this file that was stored in the configuration.


 We should also discuss what's going on in the `map()` method:




  1. Spitting our data and creating a List of the values


  2. Remove the join key from the list


  3. Re-join the data back into a single String


  4. Set the join key, join order and the remaining data


  5. Write out the data


So we have read in our data, extracted the key, set the join order and written our data back out.  Let's take a look how we will join the data.


### Joining the Data


Now let's look at how the data is joined in the reducer:

    
    
    public class JoiningReducer extends Reduce<TaggedKey, Text, NullWritable, Text> {
    
        private Text joinedText = new Text();
        private StringBuilder builder = new StringBuilder();
        private NullWritable nullKey = NullWritable.get();
    
        @Override
        protected void reduce(TaggedKey key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            builder.append(key.getJoinKey()).append(",");
            for (Text value : values) {
                builder.append(value.toString()).append(",");
            }
            builder.setLength(builder.length()-1);
            joinedText.set(builder.toString());
            context.write(nullKey, joinedText);
            builder.setLength(0);
        }
    }
    


Since the key with the tag of "1" reached the reducer first, we know that the name and address data is the first value and the email,username,password and credit card data is second. So we don't need to keep track of any keys. We simply loop over the values and concatenate them together. 


### One-To-One Join results


Here are the results from running our One-To-One MapReduce job:
`
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Esther,Garner,4071 Haven Lane,Okemos,MI,517-706-9565,EstherJGarner@teleworm.us,Waskepter38,noL2ieghie,MasterCard,5305687295670850
81a43486-07e1-4b92-b92b-03d0caa87b5f,Timothy,Duncan,753 Stadium Drive,Taunton,MA,508-307-3433,TimothyDDuncan@einrot.com,Conerse,Gif4Edeiba,MasterCard,5265896533330445
aef52cf1-f565-4124-bf18-47acdac47a0e,Brett,Ramsey,4985 Shinn Street,New York,NY,212-780-4015,BrettMRamsey@dayrep.com,Subjecall,AiKoiweihi6,MasterCard,5243379373546690
`
As we can see the two records from our sample data above have been merged into a single record. We have successfully joined the  GUID, name,address,email address, username, password and credit card fields together into one file.


##### Specifying Join Order


At this point we may be asking how do we specify the join order for multiple files?  The answer lies in our `ReduceSideJoinDriver` class that serves as the driver for our MapReduce program.

    
    
    public class ReduceSideJoinDriver {
    
    
        public static void main(String[] args) throws Exception {
            Splitter splitter = Splitter.on('/');
            StringBuilder filePaths = new StringBuilder();
    
            Configuration config = new Configuration();
            config.set("keyIndex", "0");
            config.set("separator", ",");
    
            for(int i = 0; i< args.length - 1; i++) {
                String fileName = Iterables.getLast(splitter.split(args[i]));
                config.set(fileName, Integer.toString(i+1));
                filePaths.append(args[i]).append(",");
            }
    
            filePaths.setLength(filePaths.length() - 1);
            Job job = Job.getInstance(config, "ReduceSideJoin");
            job.setJarByClass(ReduceSideJoinDriver.class);
    
            FileInputFormat.addInputPaths(job, filePaths.toString());
            FileOutputFormat.setOutputPath(job, new Path(args[args.length-1]));
    
            job.setMapperClass(JoiningMapper.class);
            job.setReducerClass(JoiningReducer.class);
            job.setPartitionerClass(TaggedJoiningPartitioner.class);
            job.setGroupingComparatorClass(TaggedJoiningGroupingComparator.class);
            job.setOutputKeyClass(TaggedKey.class);
            job.setOutputValueClass(Text.class);
            System.exit(job.waitForCompletion(true) ? 0 : 1);
    
        }
    }
    






  1. First we create a Guava Splitter on line 5 that will split strings by a "/".


  2. Then on lines 8-10 we are setting the index of our join key and the separator used in the files.


  3. In lines 12-17 we setting the tags for the input files to be joined. The order of the file names on the command line determines their position in the join.  As we loop over the file names from the command line, we split the whole file name and retrieve the last value (the base filename) via the Guava [`Iterables.getLast()`](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/Iterables.html) method. We then call `config.set()` with the filename as the key and we use `i + 1` as the value, which sets the tag or join order. The last value in the `args` array is skipped in the loop, as that is used for the output path of our MapReduce job on line 23. On the last line of the loop we append each file path in a StringBuilder which is used later (line 22) to set the input paths for the job.


  4. We only need to use one mapper for all files, the JoiningMapper, which is set on line 25.
   

  5. Lines 27 and 28 set our custom partitioner and group comparator (respectively) which ensure the arrival order of keys and values to the reducer and properly group the values with the correct key.

By using the partitioner and the grouping comparator we know the first value belongs to first key and can be used to join with every other value contained in the `Iterable` sent to the `reduce()` method for a given key.  Now it's time to consider the one-to-many join.


### One-To-Many Join


The good news is with all the work that we have done up to this point, we can actually use the code as it stands to perform a one-to-many join. There are 2 approaches we can consider for the one-to-many join: 1) A small file with the single records and a second file with many records for the same key and 2) Again a smaller file with the single records, but N number of files each containing a record that matches to the first file.  The main difference is that with the first approach the order of the values beyond the join of the first two keys will be unknown.  With the second approach however, we will "tag" each join file so we can can control the order of all the joined values.  For our example the first file will remain our GUID-name-address file, and we will have 3 additional files that will contain automobile, employer and job description records.  This is probably not the most realistic scenario but it will serve for the purposes of demonstration. Here's a sample of how the data will look before we do the join:
`
//The single person records
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Esther,Garner,4071 Haven Lane,Okemos,MI
81a43486-07e1-4b92-b92b-03d0caa87b5f,Timothy,Duncan,753 Stadium Drive,Taunton,MA
aef52cf1-f565-4124-bf18-47acdac47a0e,Brett,Ramsey,4985 Shinn Street,New York,NY
//Automobile records
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,2003 Holden Cruze
81a43486-07e1-4b92-b92b-03d0caa87b5f,2012 Volkswagen T5
aef52cf1-f565-4124-bf18-47acdac47a0e,2009 Renault Trafic
//Employer records
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Creative Wealth
81a43486-07e1-4b92-b92b-03d0caa87b5f,Susie's Casuals
aef52cf1-f565-4124-bf18-47acdac47a0e,Super Saver Foods
//Job Description records
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Data entry clerk
81a43486-07e1-4b92-b92b-03d0caa87b5f,Precision instrument and equipment repairer
aef52cf1-f565-4124-bf18-47acdac47a0e,Gas and water service dispatcher
`


##### One-To-Many Join results


Now let's look at a sample of the results of our one-to-many joins (using the same values from above to aid in the comparison):
`
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Esther,Garner,4071 Haven Lane,Okemos,MI,2003 Holden Cruze,Creative Wealth,Data entry clerk
81a43486-07e1-4b92-b92b-03d0caa87b5f,Timothy,Duncan,753 Stadium Drive,Taunton,MA,2012 Volkswagen T5,Susie's Casuals,Precision instrument and equipment repairer
aef52cf1-f565-4124-bf18-47acdac47a0e,Brett,Ramsey,4985 Shinn Street,New York,NY,2009 Renault Trafic,Super Saver Foods,Gas and water service dispatcher
`
As the results show, we have been able to successfully join several values in a specified order. 


### Conclusion


We have successfully demonstrated how we can perform reduce-side joins in MapReduce.  Even though the approach is not overly complicated, we can see that performing joins in Hadoop can involve writing a fair amount of code.  While learning how joins work is a useful exercise, in most cases we are much better off using tools like Hive or Pig for joining data.  Thanks for your time.


### Resources




    * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


    * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


    * [Source Code and Tests](https://github.com/bbejeck/hadoop-algorithms) from blog


    * [Programming Hive](http://www.amazon.com/Programming-Hive-Edward-Capriolo/dp/1449319335) by Edward Capriolo, Dean Wampler and Jason Rutherglen


    * [Programming Pig](http://www.amazon.com/Programming-Pig-Alan-Gates/dp/1449302645) by Alan Gates


    * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


    * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


