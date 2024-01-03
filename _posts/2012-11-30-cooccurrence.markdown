---
author: Bill Bejeck
comments: true
date: 2012-11-30 14:00:51+00:00
layout: post
slug: cooccurrence
title: Calculating A Co-Occurrence Matrix with Hadoop
wordpress_id: 2476
categories:
- Hadoop
- java
tags:
- Hadoop
- java
- MapReduce
---

<img class="left" src="../assets/images/hadoop-logo.jpeg" /> This post continues with our series of implementing the MapReduce algorithms found in the [Data-Intensive Text Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) book.  This time we will be creating a word co-occurrence matrix from a corpus of text. Previous posts in this series are:




  1. [Working Through Data-Intensive Text Processing with MapReduce](http://codingjunkie.net/text-processing-with-mapreduce-part1/)


  2. [Working Through Data-Intensive Text Processing with MapReduce – Local Aggregation Part II](http://codingjunkie.net/text-processing-with-mapreduce-part-2/)


 A [co-occurrence](http://en.wikipedia.org/wiki/Co-occurrence) matrix could be described as the tracking of an event, and given a certain window of time or space, what other events seem to occur.  For the purposes of this post, our "events" are the individual words found in the text and we will track what other words occur within our "window", a position relative to the target word.  For example, consider the phrase "The quick brown fox jumped over the lazy dog". With a window value of 2, the co-occurrence for the word "jumped" would be [brown,fox,over,the].  A co-occurrence matrix could be applied to other areas that require investigation into when "this" event occurs, what other events seem to happen at the same time. To build our text co-occurrence matrix, we will be implementing the Pairs and Stripes algorithms found in chapter 3 of Data-Intensive Text Processing with MapReduce. The body of text used to create our co-occurrence matrix is the collective works of [William Shakespeare](http://www.gutenberg.org/ebooks/100). 
<!--more-->

### Pairs


Implementing the pairs approach is straightforward.  For each line passed in when the map function is called, we will split on spaces creating a String Array.  The next step would be to construct two loops.  The outer loop will iterate over each word in the array and the inner loop will iterate over the "neighbors" of the current word.  The number of iterations for the inner loop is dictated by the size of our "window" to capture neighbors of the current word.  At the bottom of each iteration in the inner loop, we will emit a WordPair object (consisting of the current word on the left and the neighbor word on the right) as the key, and a count of one as the value. Here is the code for the Pairs implementation:

    
    
    public class PairsOccurrenceMapper extends Mapper<LongWritable, Text, WordPair, IntWritable> {
        private WordPair wordPair = new WordPair();
        private IntWritable ONE = new IntWritable(1);
    
        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
            int neighbors = context.getConfiguration().getInt("neighbors", 2);
            String[] tokens = value.toString().split("\\s+");
            if (tokens.length > 1) {
              for (int i = 0; i < tokens.length; i++) {
                  wordPair.setWord(tokens[i]);
    
                 int start = (i - neighbors < 0) ? 0 : i - neighbors;
                 int end = (i + neighbors >= tokens.length) ? tokens.length - 1 : i + neighbors;
                  for (int j = start; j <= end; j++) {
                      if (j == i) continue;
                       wordPair.setNeighbor(tokens[j]);
                       context.write(wordPair, ONE);
                  }
              }
          }
      }
    }
    


The Reducer for the Pairs implementation will simply sum all of the numbers for the given WordPair key:

    
    
    public class PairsReducer extends Reducer<WordPair,IntWritable,WordPair,IntWritable> {
        private IntWritable totalCount = new IntWritable();
        @Override
        protected void reduce(WordPair key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
            int count = 0;
            for (IntWritable value : values) {
                 count += value.get();
            }
            totalCount.set(count);
            context.write(key,totalCount);
        }
    }
    
    




### Stripes


Implementing the stripes approach to co-occurrence is equally straightforward.  The approach is the same, but all of the "neighbor" words are collected in a HashMap with the neighbor word as the key and an integer count as the value.  When all of the values have been collected for a given word (the bottom of the outer loop), the word and the hashmap are emitted.  Here is the code for our Stripes implementation:

    
    
    public class StripesOccurrenceMapper extends Mapper<LongWritable,Text,Text,MapWritable> {
      private MapWritable occurrenceMap = new MapWritable();
      private Text word = new Text();
    
      @Override
     protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
       int neighbors = context.getConfiguration().getInt("neighbors", 2);
       String[] tokens = value.toString().split("\\s+");
       if (tokens.length > 1) {
          for (int i = 0; i < tokens.length; i++) {
              word.set(tokens[i]);
              occurrenceMap.clear();
    
              int start = (i - neighbors < 0) ? 0 : i - neighbors;
              int end = (i + neighbors >= tokens.length) ? tokens.length - 1 : i + neighbors;
               for (int j = start; j <= end; j++) {
                    if (j == i) continue;
                    Text neighbor = new Text(tokens[j]);
                    if(occurrenceMap.containsKey(neighbor)){
                       IntWritable count = (IntWritable)occurrenceMap.get(neighbor);
                       count.set(count.get()+1);
                    }else{
                       occurrenceMap.put(neighbor,new IntWritable(1));
                    }
               }
              context.write(word,occurrenceMap);
         }
       }
      }
    }
    


The Reducer for the Stripes approach is a little more involved due to the fact we will need to iterate over a collection of maps, then for each map, iterate over all of the values in the map:

    
    
    public class StripesReducer extends Reducer<Text, MapWritable, Text, MapWritable> {
        private MapWritable incrementingMap = new MapWritable();
    
        @Override
        protected void reduce(Text key, Iterable<MapWritable> values, Context context) throws IOException, InterruptedException {
            incrementingMap.clear();
            for (MapWritable value : values) {
                addAll(value);
            }
            context.write(key, incrementingMap);
        }
    
        private void addAll(MapWritable mapWritable) {
            Set<Writable> keys = mapWritable.keySet();
            for (Writable key : keys) {
                IntWritable fromCount = (IntWritable) mapWritable.get(key);
                if (incrementingMap.containsKey(key)) {
                    IntWritable count = (IntWritable) incrementingMap.get(key);
                    count.set(count.get() + fromCount.get());
                } else {
                    incrementingMap.put(key, fromCount);
                }
            }
        }
    }
    




### Conclusion


When looking at the two approaches, we can see that the Pairs algorithm will generate more key value pairs compared to the Stripes algorithm.  Also, the Pairs algorithm captures each individual co-occurrence event while the Stripes algorithm captures all co-occurrences for a given event.  Both the Pairs and Stripes implementations would benefit from using a Combiner. Because both produce commutative and associative results, we can simply re-use each Mapper's Reducer as the Combiner. As stated before, creating a co-occurrence matrix has applicability to other fields beyond text processing, and represent useful MapReduce algorithms to have in one's arsenal.  Thanks for your time.



### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code and Tests](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


  
