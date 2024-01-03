---
author: Bill Bejeck
comments: true
date: 2012-11-01 12:45:51+00:00
layout: post
slug: testing-hadoop-programs-with-mrunit
title: Testing Hadoop Programs with MRUnit
wordpress_id: 2393
categories:
- Hadoop
- java
- Testing
tags:
- Hadoop
- MapReduce
- MRUnit
- Testing
---

<img class="left" src="../assets/images/hadoop-logo.jpeg" />  This post will take a slight detour from implementing the patterns found in [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) to discuss something equally important, testing. I was inspired in part from a presentation by [Tom Wheeler](http://www.tomwheeler.com/) that I attended while at the 2012 Strata/Hadoop World conference in New York.  When working with large data sets, unit testing might not be the first thing that comes to mind.  However, when you consider the fact that no matter how large your cluster is, or how much data you have, the same code is pushed out to all nodes for running the MapReduce job, Hadoop mappers and reducers lend themselves very well to being unit tested.  But what is not easy about unit testing Hadoop, is the framework itself.  Luckily there is a library that makes testing Hadoop fairly easy - [MRUnit](http://mrunit.apache.org/).  MRUnit is based on JUnit and allows for the unit testing of mappers, reducers and some limited integration testing of the mapper - reducer interaction along with combiners, custom counters and partitioners.  We are using the latest release of MRUnit as of this writing, 0.9.0. All of the code under test comes from the [previous post](http://codingjunkie.net/text-processing-with-mapreduce-part-2/) on computing averages using local aggregation.
<!--more-->

### Setup


To get started, download MRUnit from [here](http://mrunit.apache.org/general/downloads.html). After you have extracted the tar file, cd into the mrunit-0.9.0-incubating/lib directory.  In there you should see the following:




  1. mrunit-0.9.0-incubating-hadoop1.jar


  2. mrunit-0.9.0-incubating-hadoop2.jar


As I'm sure can guess, the mrunit-0.9.0-incubating-hadoop1.jar is for MapReduce version 1 of Hadoop and mrunit-0.9.0-incubating-hadoop2.jar is for working the new version of Hadoop's MapReduce.  For this post, and all others going forward, we will be using hadoop-2.0 version from Cloudera's [CDH4.1.1 release](https://ccp.cloudera.com/display/SUPPORT/CDH4+Downloadable+Tarballs) so we will need the mrunit-0.9.0-incubating-hadoop2.jar file.  I added MRUnit, JUnit and Mockito as libraries in Intellij (JUnit and Mockito are found in the same directory as the MRUnit jar files).  Now that we have set up our dependencies, let's start testing.


### Testing Mappers


Setting up to test a mapper is very straight forward and is best explained by looking at some code first.  We will use the in-mapper combining example from the [previous post](http://codingjunkie.net/text-processing-with-mapreduce-part-2/):

    
    
    @Test
    public void testCombiningMapper() throws Exception {
       new MapDriver<LongWritable,Text,Text,TemperatureAveragingPair>()
               .withMapper(new AverageTemperatureCombiningMapper())
               .withInput(new LongWritable(4),new Text(temps[3]))
               .withOutput(new Text("190101"),new TemperatureAveragingPair(-61,1))
               .runTest();
     }
    


Notice the fluent api style which adds the ease of creating the test.  To write your test you would:




  1. Instantiate an instance of the MapDriver class parameterized exactly as the mapper under test.


  2. Add an instance of the Mapper you are testing in the withMapper call.


  3. In the withInput call pass in your key and input value, in this case a LongWritable with an arbitrary value and a Text object that contains a line from from the NCDC weather dataset contained in a String array called 'temps' that was set up earlier in the test (not displayed here as it would take away from the presentation).


  4. Specify the expected output in the withOutput call, here we are expecting a Text object with the value of "190101" and a TemperatureAveragingPair object containing the values -61 (temperature) and a 1 (count).


  5. The last call runTest feeds the specified input values into the mapper and compares the actual output against the expected output set in the 'withOutput' method.


One thing to note is the MapDriver only allows one input and output per test.  You can call withInput and withOutput multiple times if you want, but the MapDriver will overwrite the existing values with the new ones, so you will only ever be testing with one input/output at any time. To specify multiple inputs we would use the MapReduceDriver, covered a couple of sections later, but next up is testing the reducer. 


### Testing Reducers


Testing the reducer follows the same pattern as the mapper test. Again, let's start by looking at a code example:

    
    
    @Test
    public void testReducerCold(){
      List<TemperatureAveragingPair> pairList = new ArrayList<TemperatureAveragingPair>();
          pairList.add(new TemperatureAveragingPair(-78,1));
          pairList.add(new TemperatureAveragingPair(-84,1));
          pairList.add(new TemperatureAveragingPair(-28,1));
          pairList.add(new TemperatureAveragingPair(-56,1));
    
          new ReduceDriver<Text,TemperatureAveragingPair,Text,IntWritable>()
                    .withReducer(new AverageTemperatureReducer())
                    .withInput(new Text("190101"), pairList)
                    .withOutput(new Text("190101"),new IntWritable(-61))
                    .runTest();
        }
    






  1. The test starts by creating a list of TemperatureAveragingPair objects to be used as the input to the reducer.


  2. A ReducerDriver is instantiated, and like the MapperDriver, is parameterized exactly as the reducer under test.


  3. Next we pass in an instance of the reducer we want to test in the withReducer call.


  4. In the withInput call we pass in the key of "190101" and the pairList object created at the start of the test.


  5. Next we specify the output that we expect our reducer to emit, the same key of "190101" and an IntWritable representing the average of the temperatures in the list.


  6. Finally runTest is called, which feeds our reducer the inputs specified and compares the output from the reducer against the expect output.


The ReducerDriver has the same limitation as the MapperDriver of not accepting more than one input/output pair. So far we have tested the Mapper and Reducer in isolation, but we would also like to test them together in an integration test.  Integration testing can be accomplished by using the MapReduceDriver class.  The MapReduceDriver is also the class to use for testing the use of combiners, custom counters or custom partitioners.


### Integration Testing


To test your mapper and reducer working together, MRUnit provides the MapReduceDriver class.  The MapReduceDriver class as you would expect by now, with 2 main differences.  First, you parameterize the input and output types of the mapper and the input and output types of the reducer.  Since the mapper output types need to match the reducer input types, you end up with 3 pairs of parameterized types.  Secondly you can provide multiple inputs and specify multiple expected outputs.  Here is our sample code:

    
    
    @Test
    public void testMapReduce(){
    
    new MapReduceDriver<LongWritable,Text,
                          Text,TemperatureAveragingPair,
                          Text,IntWritable>()
                    .withMapper(new AverageTemperatureMapper())
                    .withInput(new LongWritable(1),new Text(temps[0]))
                    .withInput(new LongWritable(2),new Text(temps[1]))
                    .withInput(new LongWritable(3),new Text(temps[2]))
                    .withInput(new LongWritable(4),new Text(temps[3]))
                    .withInput(new LongWritable(5),new Text(temps[6]))
                    .withInput(new LongWritable(6),new Text(temps[7]))
                    .withInput(new LongWritable(7),new Text(temps[8]))
                    .withInput(new LongWritable(8),new Text(temps[9]))
                    .withCombiner(new AverageTemperatureCombiner())
                    .withReducer(new AverageTemperatureReducer())
                    .withOutput(new Text("190101"),new IntWritable(-22))
                    .withOutput(new Text("190102"),new IntWritable(-40))
                    .runTest();
        }
    
    


As you can see from the example above, the setup is the same as the MapDriver and the ReduceDriver classes. You pass in instances of the mapper, reducer and optionally a combiner to test.  The MapReduceDriver allows us to pass in multiple inputs that have different keys.  Here the 'temps' array is the same one referenced in the mapper sample and contains a few lines from the NCDC weather dataset and the keys in those sample lines are the months of January and February of the year 1901 represented as "190101" and "190102" respectively.  This test is successful, so we gain a little more confidence around the correctness of our mapper and reducer working together.


### Conclusion


Hopefully, we have made the case for how useful MRUnit can be for testing Hadoop programs. I'd like to wrap this post up with some of my own observations. Although MRUnit makes unit testing easy for mapper and reducer code, the mapper and reducer examples presented here are fairly simple.  If your map and/or reduce code starts to become more complex, it's probably a better to decouple the code from the Hadoop framework and test the new classes on their own. Also, as useful as the MapReduceDriver class is for integration testing, it's very easy to get to a point where you are no longer testing your code, but the Hadoop framework itself, which has already been done.  I've come up with my own testing strategy that I intend to use going forward:




  1. Unit test the map/reduce code.


  2. Possibly write one integration test with the MapReduceDriver class.


  3. As a sanity check, run a MapReduce job on a single node install (on my laptop) to ensure it runs on the Hadoop framework.


  4. Then run my code on a test cluster, on EC2 using Apache Whirr in my case.


Covering how to set up a single node install on my laptop (OSX Lion) and standing up a cluster on EC2 using Whirr would make this post too long, so I'll cover those topics in the next one.  Thanks for your time.


### Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


  * [Project Gutenberg](http://www.gutenberg.org/) a great source of books in plain text format, great for testing Hadoop jobs locally.


