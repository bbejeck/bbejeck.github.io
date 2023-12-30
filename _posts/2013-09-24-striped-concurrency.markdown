---
author: Bill Bejeck
comments: true
date: 2013-09-24 15:00:25+00:00
layout: post
slug: striped-concurrency
title: Fine-Grained Concurrency with the Guava Striped Class
wordpress_id: 2906
categories:
- Concurrency
- Guava
- java
tags:
- Concurrency
- Guava
- java
---

<img class="left" src="{{ site.media_url }}/images/google-guava.gif" /> This post is going to cover how to use the Striped class from Guava to achieve finer-grained concurrency. The [ConcurrentHashMap](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentHashMap.html) uses a striped locked approach to increase concurrency and the [Striped](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/util/concurrent/Striped.html) class extends this principal by giving us the ability to have striped [Locks](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantLock.html), [ReadWriteLocks](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html) and [Semaphores](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Semaphore.html). When accessing an object or data-structure such as an Array or HashMap typically we would synchronize on the entire object or data-structure, but is this always necessary? In several cases the answer is yes, but there may be times where we don't that level of course-grained locking and can improve the performance of the application by using a finer-grained approach.  
<!--more-->

### The Use Case for Striped Locking/Semaphores


It's considered a best practice to always synchronize the smallest portion of code that is possible.  In other words, we only want to synchronize around the parts of the code that are changing shared data.  Striping takes this a step further by allowing us to reduce synchronization by distinct _tasks_.  For example, let's say we have an ArrayList of URL's that represent remote resources and we want to restrict the total number of threads accessing these resources at any given time.  Restricting access for N number of threads is a natural fit for a `java.util.concurrent.Semaphore` object. But the problem is restricting access to our ArrayList becomes a bottleneck.  This is where the `Striped` class helps. What we really want to do is restrict access to the resources, and not the container of the URL's.  Assuming each one is distinct, we use a "striped" approach where access to _each_ URL is limited to N threads at a time, increasing the throughput and responsiveness of the application.  
  


### Creating Striped Locks/Semaphores


To create a Striped instance we simply use one of the static factor methods available from the `Striped` class.  For example, here's how we would create an instance of a striped `ReadWriteLock`: 
``` java Create striped read-write lock
Striped<ReadWriteLock> rwLockStripes = Striped.readWriteLock(10);
```


In the above example we created a `Striped` instance where the individual "stripes" are eagerly created, strong references.  The impact is the locks/semaphores won't be garbage collected unless the Striped object itself is removed. However, the Striped class gives us another option when it comes to creating locks/semaphores:

    
    
    int permits = 5;
    int numberStripes = 10;
    Striped<Semaphore> lazyWeakSemaphore = Striped.lazyWeakSemaphore(numberStripes,permits);
    


In this example the `Semaphore` instances will be _lazily_ created and wrapped in `WeakReferences` thus available immediately for garbage collection unless another object is holding on to it.  


### Accessing/Using Striped Locks/Semaphores


To use the Striped `ReadWriteLock` created earlier, we do the following:

    
    
    String key = "taskA";
    ReadWriteLock rwLock = rwLockStripes.get(key);
    try{
         rwLock.lock();
         .....
    }finally{
         rwLock.unLock();
    }
    


By calling the `Striped.get` method we are returned a `ReadWriteLock` instance that corresponds with the given object key.  We will more extensive examples below using the Striped class.


#### Lock/Semaphore Retrieval Guarantees


When it comes to retrieving lock/semaphore instances the Striped class guarantees when objectA equals objectB (assuming properly implemented equals and hashcode methods) a call to the `Striped.get` method will return the same lock/semaphore instance.  But the converse is not guaranteed, that is to say when objectA is not equal to ObjectB, we may still retrieve locks/semaphores that are the same reference.  According to the Javadoc for the Striped class, specifying fewer stripes increased the probability of getting the same reference for two different keys. 


### An Example


Let's take a look at a very simple example based on the scenario we laid out in the introduction.  We've constructed a class `ConcurrentWorker` where we want to restrict concurrent access to some resource. But let's make an assumption there are some number of _distinct_ resources so ideally, we want to restrict access per resource and not just restrict the `ConcurrentWorker`.

    
    
      public class ConcurrentWorker {
    
        private Striped<Semaphore> stripedSemaphores = Striped.semaphore(10,3);
        private Semaphore semaphore = new Semaphore(3);
    
        public void stripedConcurrentAccess(String url) throws Exception{
           Semaphore stripedSemaphore  = stripedSemaphores.get(url);
            stripedSemaphore.acquire();
           try{
               //Access restricted resource here
               Thread.sleep(25);
           }finally{
               stripedSemaphore.release();
           }
        }
    
        public void nonStripedConcurrentAccess(String url) throws Exception{
            semaphore.acquire();
            try{
               //Access restricted resource here
                Thread.sleep(25);
            }finally{
                semaphore.release();
            }
        }
    }
    


In this example we have two methods `ConcurrentWorker.stripedConcurrentAccess` and `ConcurrentWorker.nonStripedConcurrentAccess` so we can make an easy comparison.  In both cases we simulate work by calling `Thread.sleep` for 25 milliseconds. We've create a single `Semaphore` instance and a `Striped` instance that contains 10 eagerly created, strongly referenced `Semaphore` objects.  In both cases we've specified 3 permits, so at any time 3 threads will have access to our "resource".  Here is a simple driver class used to measure the throughput between both approaches.

    
    
    public class StripedExampleDriver {
    
        private ExecutorService executorService = Executors.newCachedThreadPool();
        private int numberThreads = 300;
        private CountDownLatch startSignal = new CountDownLatch(1);
        private CountDownLatch endSignal = new CountDownLatch(numberThreads);
        private Stopwatch stopwatch = Stopwatch.createUnstarted();
        private ConcurrentWorker worker = new ConcurrentWorker();
        private static final boolean USE_STRIPES = true;
        private static final boolean NO_STRIPES = false;
        private static final int POSSIBLE_TASKS_PER_THREAD = 10;
        private List<String> data = Lists.newArrayList();
    
    
        public static void main(String[] args) throws Exception {
            StripedExampleDriver driver = new StripedExampleDriver();
            driver.createData();
            driver.runStripedExample();
            driver.reset();
            driver.runNonStripedExample();
            driver.shutdown();
        }
    
        private void runStripedExample() throws InterruptedException {
            runExample(worker, USE_STRIPES, "Striped work");
        }
    
        private void runNonStripedExample() throws InterruptedException {
            runExample(worker, NO_STRIPES, "Non-Striped work");
        }
    
    
        private void runExample(final ConcurrentWorker worker, final boolean isStriped, String type) throws InterruptedException {
            for (int i = 0; i < numberThreads; i++) {
                final String value = getValue(i % POSSIBLE_TASKS_PER_THREAD);
                executorService.submit(new Callable<Void>() {
                    @Override
                    public Void call() throws Exception {
                        startSignal.await();
                        if (isStriped) {
                            worker.stripedConcurrentAccess(value);
                        } else {
                            worker.nonStripedConcurrentAccess(value);
                        }
                        endSignal.countDown();
                        return null;
                    }
                });
            }
            stopwatch.start();
            startSignal.countDown();
            endSignal.await();
            stopwatch.stop();
            System.out.println("Time for" + type + " work [" + stopwatch.elapsed(TimeUnit.MILLISECONDS) + "] millis");
        }
      //details left out for clarity
    


We took 300 threads and called both methods in two separate passes and used the `StopWach` class to record the times. 


#### Results


As you might expect the striped version faired much better.  Here's the console output from one of the test runs:

    
    
    Time forStriped work work [261] millis
    Time forNon-Striped work work [2596] millis
    


While the `Striped` class is not a fit for every situation, it really comes in handy when you need concurrency around distinct data.  Thanks for your time.


### Resources






  * [Guava API](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/index.html?overview-summary.html)


  * [Concurrency in Practice](http://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601)


  * [Source code for this post](https://github.com/bbejeck/guava-blog/tree/master/src/main/java/bbejeck/guava/striped)


