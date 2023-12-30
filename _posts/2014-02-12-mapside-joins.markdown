---
comments: true
date: 2014-02-12 14:30:26+00:00
layout: post
slug: mapside-joins
title: MapReduce Algorithms - Understanding Data Joins Part II
wordpress_id: 2981
categories:
- Hadoop
- java
- MapReduce
tags:
- Hadoop
- java
- MapReduce
---

<img class="left" src="{{ site.media_url }}/images/hadoop-logo.jpeg" /> It's been awhile since I last posted, and like last time I took a big break, I was taking some classes on Coursera.  This time it was [Functional Programming Principals in Scala](https://www.coursera.org/course/progfun) and [Principles of Reactive Programming](https://www.coursera.org/course/reactive).  I found both of them to be great courses and would recommend taking either one if you have the time.  In this post we resume our series on implementing the algorithms found in [Data-Intensive Text Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421), this time covering map-side joins.  As we can guess from the name, map-side joins join data exclusively during the mapping phase and completely skip the reducing phase.  In the last post on data joins we covered [reduce side joins](http://codingjunkie.net/mapreduce-reduce-joins/).  Reduce-side joins are easy to implement, but have the drawback that all data is sent across the network to the reducers.   Map-side joins offer substantial gains in performance since we are avoiding the cost of sending data across the network. However, unlike reduce-side joins, map-side joins require very specific criteria be met.  Today we will discuss the requirements for map-side joins and how we can implement them.
<!--more-->

### Map-Side Join Conditions


To take advantage of map-side joins our data must meet one of following criteria:




  1. The datasets to be joined are already sorted by the same key and have the same number of partitions


  2. Of the two datasets to be joined, one is small enough to fit into memory


We are going to consider the first scenario where we have two (or more) datasets that need to be joined, but are too large to fit into memory. We will assume the worst case scenario, the files aren't sorted or partitioned the same.


### Data Format


Before we start, let's take a look at the data we are working with.  We will have two datasets:




  1. The first dataset consists of a GUID, First Name, Last Name, Address, City and State


  2. The second dataset consists of a GUID and Employer information


Both datasets are comma delimited and the join-key (GUID) is in the first position.  After the join we want the employer information from dataset two to be appended to the end of dataset one.  Additionally, we want to keep the GUID in the first position of dataset one, but remove the GUID from dataset two.
Dataset 1:

    
    
      aef9422c-d08c-4457-9760-f2d564d673bc,Linda,Narvaez,3253 Davis Street,Atlanta,GA
      08db7c55-22ae-4199-8826-c67a5689f838,John,Gregory,258 Khale Street,Florence,SC
      de68186a-1004-4211-a866-736f414eac61,Charles,Arnold,1764 Public Works Drive,Johnson City,TN
      6df1882d-4c81-4155-9d8b-0c35b2d34284,John,Schofield,65 Summit Park Avenue,Detroit,MI
    


Dataset 2:

    
      
      de68186a-1004-4211-a866-736f414eac61,Jacobs
      6df1882d-4c81-4155-9d8b-0c35b2d34284,Chief Auto Parts
      aef9422c-d08c-4457-9760-f2d564d673bc,Earthworks Yard Maintenance
      08db7c55-22ae-4199-8826-c67a5689f838,Ellman's Catalog Showrooms 
    


Joined results:

    
    
    08db7c55-22ae-4199-8826-c67a5689f838,John,Gregory,258 Khale Street,Florence,SC,Ellman's Catalog Showrooms
    6df1882d-4c81-4155-9d8b-0c35b2d34284,John,Schofield,65 Summit Park Avenue,Detroit,MI,Chief Auto Parts
    aef9422c-d08c-4457-9760-f2d564d673bc,Linda,Narvaez,3253 Davis Street,Atlanta,GA,Earthworks Yard Maintenance
    de68186a-1004-4211-a866-736f414eac61,Charles,Arnold,1764 Public Works Drive,Johnson City,TN,Jacobs
    


Now we move on to how we go about joining our two datasets.


### Map-Side Joins with Large Datasets


To be able to perform map-side joins we need to have our data sorted by the same key and have the same number of partitions, implying that all keys for any record are in the same partition.  While this seems to be a tough requirement, it is easily fixed.  Hadoop sorts all keys and guarantees that keys with the same value are sent to the same reducer. So by simply running a MapReduce job that does nothing more than output the data by the key you want to join on and specifying the exact same number of reducers for all datasets, we will get our data in the correct form.  Considering the gains in efficiency from being able to do a map-side join, it may be worth the cost of running additional MapReduce jobs. It bears repeating at this point it is crucial all datasets specify the _exact_ same number of reducers during the "preparation" phase when the data will be sorted and partitioned. In this post we will take two data-sets and run an initial MapReduce job on both to do the sorting and partitioning and then run a final job to perform the map-side join. First let's cover the MapReduce job to sort and partition our data in the same way.


#### Step One: Sorting and Partitioning


First we need to create a `Mapper` that will simply choose the key for sorting by a given index:

    
    
     public class SortByKeyMapper extends Mapper<LongWritable, Text, Text, Text> {
    
        private int keyIndex;
        private Splitter splitter;
        private Joiner joiner;
        private Text joinKey = new Text();
    
    
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
            String separator =  context.getConfiguration().get("separator");
            keyIndex = Integer.parseInt(context.getConfiguration().get("keyIndex"));
            splitter = Splitter.on(separator);
            joiner = Joiner.on(separator);
        }
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            Iterable<String> values = splitter.split(value.toString());
            joinKey.set(Iterables.get(values,keyIndex));
            if(keyIndex != 0){
                value.set(reorderValue(values,keyIndex));
            }
            context.write(joinKey,value);
        }
    
    
        private String reorderValue(Iterable<String> value, int index){
            List<String> temp = Lists.newArrayList(value);
            String originalFirst = temp.get(0);
            String newFirst = temp.get(index);
            temp.set(0,newFirst);
            temp.set(index,originalFirst);
            return joiner.join(temp);
        }
    }
    


The `SortByKeyMapper` simply sets the value of the `joinKey` by extracting the value from the given line of text found at the position given by the configuration parameter `keyIndex`.  Also, if the `keyIndex` is not equal to zero, we swap the order of the values found in the first position and the `keyIndex` position.  Although this is a questionable feature, We'll discuss why we are doing this later. Next we need a `Reducer`:

    
    
    public class SortByKeyReducer extends Reducer<Text,Text,NullWritable,Text> {
    
        private static final NullWritable nullKey = NullWritable.get();
    
        @Override
        protected void reduce(Text key, Iterable<Text> values, Context context) throws IOException, InterruptedException {
            for (Text value : values) {
                 context.write(nullKey,value);
            }
        }
    }
    


The `SortByKeyReducer` writes out all values for the given key, but throws out the key and writes a `NullWritable` instead.  In the next section we will explain why we are not using the key.


#### Step Two: The Map-Side join


When performing a map-side join the records are merged _before_ they reach the mapper.  To achieve this, we use the [CompositeInputFormat](http://hadoop.apache.org/docs/current2/api/org/apache/hadoop/mapreduce/lib/join/CompositeInputFormat.html). We will also need to set some configuration properties. Let's look at how we will configure our map-side join:

    
    
    private static Configuration getMapJoinConfiguration(String separator, String... paths) {
            Configuration config = new Configuration();
            config.set("mapreduce.input.keyvaluelinerecordreader.key.value.separator", separator);
            String joinExpression = CompositeInputFormat.compose("inner", KeyValueTextInputFormat.class, paths);
            config.set("mapred.join.expr", joinExpression);
            config.set("separator", separator);
            return config;
        }
    


First, we are specifying the character that separates the key and values by setting the `mapreduce.input.keyvaluelinerecordreader.key.value.separator` property.  Next we use the `CompositeInputFormat.compose` method to create a "join expression" specifying an inner join by using the word "inner", then specifying the input format to use, the [KeyValueTextInput](http://hadoop.apache.org/docs/current2/api/org/apache/hadoop/mapreduce/lib/input/KeyValueTextInputFormat.html)class and finally a String varargs representing the paths of the files to join (which are the output paths of the map-reduce jobs ran to sort and partition the data). The `KeyValueTextInputFormat` class will use the separator character to set the first value as the key and the rest will be used for the value. 


#### Mapper for the join


Once the values from the source files have been joined, the `Mapper.map` method is called, it will receive a `Text` object for the key (the same key across joined records) and a `TupleWritable` that is composed of the values joined from our input files for a given key. Remember we want our final output to have the join-key in the first position, followed by all of joined values in one delimited `String`. To achieve this we have a custom mapper to put our data in the correct format:

    
    
    public class CombineValuesMapper extends Mapper<Text, TupleWritable, NullWritable, Text> {
    
        private static final NullWritable nullKey = NullWritable.get();
        private Text outValue = new Text();
        private StringBuilder valueBuilder = new StringBuilder();
        private String separator;
    
        @Override
        protected void setup(Context context) throws IOException, InterruptedException {
            separator = context.getConfiguration().get("separator");
        }
    
        @Override
        protected void map(Text key, TupleWritable value, Context context) throws IOException, InterruptedException {
            valueBuilder.append(key).append(separator);
            for (Writable writable : value) {
                valueBuilder.append(writable.toString()).append(separator);
            }
            valueBuilder.setLength(valueBuilder.length() - 1);
            outValue.set(valueBuilder.toString());
            context.write(nullKey, outValue);
            valueBuilder.setLength(0);
        }
    }
    


In the `CombineValuesMapper` we are appending the key and all the joined values into one delimited `String`.  Here we can finally see the reason why we threw the join-key away in the previous MapReduce jobs.  Since the key is the first position in the values for all the datasets to be joined, our mapper naturally eliminates the duplicate keys from the joined datasets.  All we need to do is insert the given key into a `StringBuilder`, then append the values contained in the `TupleWritable`.


### Putting It All Together


Now we have all the code in place to run a map-side join on large datasets.  Let's take a look at how we will run all the jobs together.  As was stated before, we are assuming that our data is not sorted and partitioned the same, so we will need to run N (2 in this case) MapReduce jobs to get the data in the correct format.  After the initial sorting/partitioning jobs run, the final job performing the actual join will run.

    
    
    public class MapSideJoinDriver {
    
        public static void main(String[] args) throws Exception {
            String separator = ",";
            String keyIndex = "0";
            int numReducers = 10;
            String jobOneInputPath = args[0];
            String jobTwoInputPath = args[1];
            String joinJobOutPath = args[2];
    
            String jobOneSortedPath = jobOneInputPath + "_sorted";
            String jobTwoSortedPath = jobTwoInputPath + "_sorted";
    
            Job firstSort = Job.getInstance(getConfiguration(keyIndex, separator));
            configureJob(firstSort, "firstSort", numReducers, jobOneInputPath, jobOneSortedPath, SortByKeyMapper.class, SortByKeyReducer.class);
    
            Job secondSort = Job.getInstance(getConfiguration(keyIndex, separator));
            configureJob(secondSort, "secondSort", numReducers, jobTwoInputPath, jobTwoSortedPath, SortByKeyMapper.class, SortByKeyReducer.class);
    
            Job mapJoin = Job.getInstance(getMapJoinConfiguration(separator, jobOneSortedPath, jobTwoSortedPath));
            configureJob(mapJoin, "mapJoin", 0, jobOneSortedPath + "," + jobTwoSortedPath, joinJobOutPath, CombineValuesMapper.class, Reducer.class);
            mapJoin.setInputFormatClass(CompositeInputFormat.class);
    
            List<Job> jobs = Lists.newArrayList(firstSort, secondSort, mapJoin);
            int exitStatus = 0;
            for (Job job : jobs) {
                boolean jobSuccessful = job.waitForCompletion(true);
                if (!jobSuccessful) {
                    System.out.println("Error with job " + job.getJobName() + "  " + job.getStatus().getFailureInfo());
                    exitStatus = 1;
                    break;
                }
            }
            System.exit(exitStatus);
        }
      
    


The `MapSideJoinDriver` does the basic configuration for running MapReduce jobs.  One interesting point is the sorting/partitioning jobs specify 10 reducers each, while the final job explicitly sets the number of reducers to 0, since we are joining on the map-side and don't need a reduce phase. Since we don't have any complicated dependencies, we put the jobs in an ArrayList and run the jobs in linear order (lines 24-33).


### Results


Initially we had 2 files; name and address information in the first file and employment information in the second.  Both files had a unique id in the first column.
File one:

    
    
    ....
    08db7c55-22ae-4199-8826-c67a5689f838,John,Gregory,258 Khale Street,Florence,SC
    ...
    


File two:

    
    
    ....
    08db7c55-22ae-4199-8826-c67a5689f838,Ellman's Catalog Showrooms
    ....
    


Results:

    
    
    08db7c55-22ae-4199-8826-c67a5689f838,John,Gregory,258 Khale Street,Florence,SC,Ellman's Catalog Showrooms
    


As we can see here, we've successfully joined the records together and maintained the format of the files without duplicate keys in the results.


#### Conclusion


In this post we've demonstrated how to perform a map-side join when both data sets are large and can't fit into memory.  If you get the feeling this takes a lot of work to pull off, you are correct.  While in most cases we would want to use higher level tools like Pig or Hive, it's helpful to know the mechanics of performing map-side joins with large datasets.  This especially true on those occasions when you need to write a solution from scratch.  Thanks for your time.


### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code and Tests](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Programming Hive](http://www.amazon.com/Programming-Hive-Edward-Capriolo/dp/1449319335) by Edward Capriolo, Dean Wampler and Jason Rutherglen


  * [Programming Pig](http://www.amazon.com/Programming-Pig-Alan-Gates/dp/1449302645) by Alan Gates


  * [Hadoop API](http://hadoop.apache.org/docs/current2/api/)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


