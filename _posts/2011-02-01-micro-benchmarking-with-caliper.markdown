---
author: Bill Bejeck
comments: true
date: 2011-02-01 05:00:20+00:00
layout: post
slug: micro-benchmarking-with-caliper
title: Micro Benchmarking with Caliper
wordpress_id: 577
categories:
- General
---

From time to time I think all developers have done some form of benchmarking.  I recently discovered [Caliper](http://code.google.com/p/caliper/) which is according to the site -  "Caliper is Google's open-source framework for writing, running and viewing the results of Java Microbenchmarks".  I am aware that micro-benchmarking can be misleading depending on who is writing the tests, but sometimes they are very helpful in getting a feel for how your code is running.
<!--more-->

### Background


Micro benchmarks are dead simple to do.  Get the current time in milliseconds, execute your code, then get the time in milliseconds again and subtract the difference.  So why use a tool like Caliper?  To me Caliper is great because it has a very familiar JUnit type of structure and feel to it. Instead of trying to describe what Caliper does, it's probably easiest to look at a simple example I put together.  I decided to benchmark implementations of Heap Sort,Merge Sort,Quick Sort and the sort method of the java.lang.Arrays class.  Not a terribly original idea I admit, but it was quick to do and it makes the point. **NOTE: Package statement and imports left out intentionally for brevity**

    
    
    public class SortingBenchmarks extends SimpleBenchmark {
    	
    	private static final int SIZE = 100000;
    	private static final int MAX_VALUE = 80000;
    	private int[] values;
    
    	@Override
    	protected void setUp() throws Exception {
    		values = new int[SIZE];
    		Random generator = new Random();
    		for (int i = 0; i < values.length; i++) {
    			values[i] = generator.nextInt(MAX_VALUE);
    		}
    	}
    
    	public void timeHeapSort(int reps) {
    		for (int i = 0; i < reps; i++) {
    			HeapSort.sort(values);
    		}
    	}
    
    	public void timeMergeSort(int reps) {
    		for (int i = 0; i < reps; i++) {
    			MergeSort.sort(values);
    		}
    	}
    	
    	public void timeQuickSort(int reps) {
    		for (int i = 0; i < reps; i++) {
    			QuickSort.sort(values);
    		}
    	}
    
    	public void timeArraysSort(int reps) {
    		for (int i = 0; i < reps; i++) {
    			Arrays.sort(values);
    		}
    	}
    }
    




### Getting Started


Here are the basic steps to writing benchmarks in Caliper:




  1. Extend the class SimpleBenchmark


  2. Do any test setup/clean up in respective setUp or tearDown methods (Similar to JUnit setUp/tearDown)


  3. Write the methods that will execute the code to benchmark starting with the word "time" (Again similar to JUnit, just "time" instead of "test")


  4. Place the code you want to benchmark inside your timeSomeOperation methods




### Getting and Running Caliper


To get started with Caliper




  1. Go here [here](http://code.google.com/p/caliper/source/checkout) to get a read-only svn link to the source code


  2. Check out the code then cd into <CALIPER_INSTALL_DIR> and run ant (obviously you need ant installed and on your path)


To actually run your benchmarks there is a bash script included in the project that you can use to run Caliper from the command line.  I took a little different approach as I wanted to run from inside my Eclipse project.


  1. In <CALIPER_INSTALL_DIR>/build/caliper-0.0/lib/ there are two jar files, allocation.jar and caliper-0.0.jar.  These will need to be on the classpath of your project.  I chose to create a user library in Eclipse


  2. JUnit will also need to be on the project classpath.  Again for me it's referenced in a user library


  3. I created a driver class that simply executes the main method of the com.google.caliper.Runner class.  I pass in the name of my benchmark class by setting up a run configuration in Eclipse for my driver class


Here is the code for my driver class:

    
    
    package bbejeck.caliper;
    
    public class CaliperRunner {
    
        public static void main(String[] args) {
    	com.google.caliper.Runner.main(args[0]);
        }
    
    }
    


After running the driver this is the output received in the Eclipse console screen:
`
0% Scenario{vm=java, trial=0, benchmark=HeapSort} 10889428.57 ns; ?=69810.62 ns @ 3 trials
25% Scenario{vm=java, trial=0, benchmark=MergeSort} 9066618.18 ns; ?=44341.70 ns @ 3 trials
50% Scenario{vm=java, trial=0, benchmark=QuickSort} 3312312.93 ns; ?=21028.91 ns @ 3 trials
75% Scenario{vm=java, trial=0, benchmark=ArraysSort} 3104668.79 ns; ?=23965.54 ns @ 3 trials

 benchmark    ms logarithmic runtime
  HeapSort 10.89 ==============================
 MergeSort  9.07 =========================
 QuickSort  3.31 ==
ArraysSort  3.10 =

vm: java
trial:
`


### Conclusion


Caliper is fairly new and is still has some work to be done, but I think that it's a great tool that could be very useful. Another potentially useful feature is that Caliper can [automatically publish your benchmarks.](http://code.google.com/p/caliper/wiki/OnlineResults)   Full source code for my examples can be found on [github](https://github.com/bbejeck/CaliperBlog)

