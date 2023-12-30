---
author: Bill Bejeck
comments: true
date: 2012-12-13 14:00:18+00:00
layout: post
slug: order-inversion
title: MapReduce Algorithms - Order Inversion
wordpress_id: 2537
categories:
- Hadoop
- java
- MapReduce
tags:
- Hadoop
- java
- MapReduce
---

<img class="left" src="{{ site.media_url }}/images/hadoop-logo.jpeg" /> This post is another segment in the series presenting MapReduce algorithms as found in the [Data-Intensive Text Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) book.  Previous installments are [Local Aggregation](http://codingjunkie.net/text-processing-with-mapreduce-part1/), [Local Aggregation PartII](http://codingjunkie.net/text-processing-with-mapreduce-part-2/) and [Creating a Co-Occurrence Matrix](http://codingjunkie.net/cooccurrence/).  This time we will discuss the order inversion pattern.  The order inversion pattern exploits the sorting phase of MapReduce to push data needed for calculations to the reducer _ahead of_ the data that will be manipulated..  Before you dismiss this as an edge condition for MapReduce, I urge you to read on as we will discuss how to use sorting to our advantage and cover using a custom partitioner, both of which are useful tools to have available.  Although many MapReduce programs are written at a higher level abstraction i.e Hive or Pig, it's still helpful to have an understanding of what's going on at a lower level.  The order inversion pattern is found in chapter 3 of Data-Intensive Text Processing with MapReduce book.  To illustrate the order inversion pattern we will be using the Pairs approach from the co-occurrence matrix pattern.  When creating the co-occurrence matrix, we track the total counts of when words appear together.  At a high level we take the Pairs approach and add a small twist, in addition to having the mapper emit a word pair such as ("foo","bar") we will emit an additional word pair of ("foo","*") and will do so for every word pair so we can easily achieve a total count for how often the left most word appears, and use that count to calculate our relative frequencies.  This approach raised two specific problems.  First we need to find a way to ensure word pairs ("foo","*") arrive at the reducer first.  Secondly we need to make sure all word pairs with the same left word arrive at the same reducer. Before we solve those problems, let's take a look at our mapper code.
<!--more-->

### Mapper Code


First we need to modify our mapper from the Pairs approach.  At the bottom of each loop after we have emitted all the word pairs for a particular word, we will emit the special token WordPair("word","*") along with the count of times the word on the left was found.

    
    
    public class PairsRelativeOccurrenceMapper extends Mapper<LongWritable, Text, WordPair, IntWritable> {
        private WordPair wordPair = new WordPair();
        private IntWritable ONE = new IntWritable(1);
        private IntWritable totalCount = new IntWritable();
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            int neighbors = context.getConfiguration().getInt("neighbors", 2);
            String[] tokens = value.toString().split("\\s+");
            if (tokens.length > 1) {
                for (int i = 0; i < tokens.length; i++) {
                        tokens[i] = tokens[i].replaceAll("\\W+","");
    
                        if(tokens[i].equals("")){
                            continue;
                        }
    
                        wordPair.setWord(tokens[i]);
    
                        int start = (i - neighbors < 0) ? 0 : i - neighbors;
                        int end = (i + neighbors >= tokens.length) ? tokens.length - 1 : i + neighbors;
                        for (int j = start; j <= end; j++) {
                            if (j == i) continue;
                            wordPair.setNeighbor(tokens[j].replaceAll("\\W",""));
                            context.write(wordPair, ONE);
                        }
                        wordPair.setNeighbor("*");
                        totalCount.set(end - start);
                        context.write(wordPair, totalCount);
                }
            }
        }
    }
    


Now that we've generated a way to track the total numbers of times a particular word has been encountered, we need to make sure those special characters reach the reducer first so a total can be tallied to calculate the relative frequencies. We will have the sorting phase of the MapReduce process handle this for us by modifying the compareTo method on the WordPair object. 


### Modified Sorting

 
We modify the compareTo method on the WordPair class so when a "*" caracter is encountered on the right that particular object is pushed to the top.

    
    
        @Override
        public int compareTo(WordPair other) {
            int returnVal = this.word.compareTo(other.getWord());
            if(returnVal != 0){
                return returnVal;
            }
            if(this.neighbor.toString().equals("*")){
                return -1;
            }else if(other.getNeighbor().toString().equals("*")){
                return 1;
            }
            return this.neighbor.compareTo(other.getNeighbor());
        }
    
    


By modifying the compareTo method we now are guaranteed that any WordPair with the special character will be sorted to the top and arrive at the reducer first.  This leads to our second specialization, how can we guarantee that all WordPair objects with a given left word will be sent to the same reducer? The answer is to create a custom partitioner.


### Custom Partitioner


Intermediate keys are shuffled to reducers by calculating the hashcode of the key modulo the number of reducers.  But our WordPair objects contain two words, so taking the hashcode of the entire object clearly won't work.  We need to wright a custom Partitioner that only takes into consideration the left word when it comes to determining which reducer to send the output to.

    
    
    public class WordPairPartitioner extends Partitioner<WordPair,IntWritable> {
    
        @Override
        public int getPartition(WordPair wordPair, IntWritable intWritable, int numPartitions) {
            return wordPair.getWord().hashCode() % numPartitions;
        }
    }
    


Now we are guaranteed that all of the WordPair objects with the same left word are sent to the same reducer.  All that is left is to construct a reducer to take advantage of the format of the data being sent.


### Reducer


Building the reducer for the inverted order inversion pattern is straight forward. It will involve keeping a counter variable and a "current" word variable. The reducer will check the input key WordPair for the special character "*" on the right. If the word on the left is not equal to the "current" word we will re-set the counter and sum all of the values to obtain a total number of times the given current word was observed.  We will now process the next WordPair objects, sum the counts and divide by our counter variable to obtain a relative frequency. This process will continue until another special character is encountered and the process starts over.

    
    
    public class PairsRelativeOccurrenceReducer extends Reducer<WordPair, IntWritable, WordPair, DoubleWritable> {
        private DoubleWritable totalCount = new DoubleWritable();
        private DoubleWritable relativeCount = new DoubleWritable();
        private Text currentWord = new Text("NOT_SET");
        private Text flag = new Text("*");
    
        @Override
        protected void reduce(WordPair key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
            if (key.getNeighbor().equals(flag)) {
                if (key.getWord().equals(currentWord)) {
                    totalCount.set(totalCount.get() + getTotalCount(values));
                } else {
                    currentWord.set(key.getWord());
                    totalCount.set(0);
                    totalCount.set(getTotalCount(values));
                }
            } else {
                int count = getTotalCount(values);
                relativeCount.set((double) count / totalCount.get());
                context.write(key, relativeCount);
            }
        }
      private int getTotalCount(Iterable<IntWritable> values) {
            int count = 0;
            for (IntWritable value : values) {
                count += value.get();
            }
            return count;
        }
    }
    


By manipulating the sort order and creating a custom partitioner, we have been able to send data to a reducer needed for a calculation, before the data needed for those calculation arrive. Although not shown here, a combiner was used to run the MapReduce job.  This approach is also a good candidate for the "in-mapper" combining pattern. 


### Example & Results


Given that the holidays are upon us, I felt it was timely to run an example of the order inversion pattern against the novel "A Christmas Carol" by Charles Dickens.  I know it's corny, but it serves the purpose.

    
    
    new-host-2:sbin bbejeck$ hdfs dfs -cat relative/part* | grep Humbug
    {word=[Humbug] neighbor=[Scrooge]}	0.2222222222222222
    {word=[Humbug] neighbor=[creation]}	0.1111111111111111
    {word=[Humbug] neighbor=[own]}	0.1111111111111111
    {word=[Humbug] neighbor=[said]}	0.2222222222222222
    {word=[Humbug] neighbor=[say]}	0.1111111111111111
    {word=[Humbug] neighbor=[to]}	0.1111111111111111
    {word=[Humbug] neighbor=[with]}	0.1111111111111111
    {word=[Scrooge] neighbor=[Humbug]}	0.0020833333333333333
    {word=[creation] neighbor=[Humbug]}	0.1
    {word=[own] neighbor=[Humbug]}	0.006097560975609756
    {word=[said] neighbor=[Humbug]}	0.0026246719160104987
    {word=[say] neighbor=[Humbug]}	0.010526315789473684
    {word=[to] neighbor=[Humbug]}	3.97456279809221E-4
    {word=[with] neighbor=[Humbug]}	9.372071227741331E-4
    




### Conclusion


While calculating relative word occurrence frequencies probably is not a common task, we have been able to demonstrate useful examples of sorting and using a custom partitioner, which are good tools to have at your disposal when building MapReduce programs.  As stated before, even if most of your MapReduce is written at higher level of abstraction like Hive or Pig, it's still instructive to have an understanding of what is going on under the hood.  Thanks for your time.


### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code and Tests](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


