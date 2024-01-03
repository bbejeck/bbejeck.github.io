---
author: Bill Bejeck
comments: true
date: 2011-11-16 05:54:16+00:00
layout: post
slug: google-guava-synchronization-with-monitor
title: Google Guava - Synchronization with Monitor
wordpress_id: 1127
categories:
- General
tags:
- Concurrency
- Guava
- java
---

<img class="left" src="../assets/images/toolbox.jpg" /> The Google Guava project is a collection of libraries that every Java developer should become familiar with.   The Guava libraries cover I/O, collections, string manipulation, and concurrency just to name a few.  In this post I am going to cover the Monitor class.  Monitor is a synchronization construct that can be used anywhere you would use a ReentrantLock. Only one thread can occupy a monitor at any time.  The Monitor class has operations of _entering_ and _leaving_  which are semantically the same as the _lock_ and _unlock_ operations in ReentrantLock.  Additionally, the Monitor supports waiting on _boolean_ conditions. 
<!--more-->

### Comparing Monitor and ReentrantLock


For starters it would be helpful to do a side by side comparison the Monitor and ReentrantLock.

    
    
    public class ReentrantLockSample {
        private List<String> list = new ArrayList<String>();
        private static final int MAX_SIZE = 10;
    
        private ReentrantLock rLock = new ReentrantLock();
        private Condition listAtCapacity = rLock.newCondition();
    
        public void addToList(String item) throws InterruptedException {
            rLock.lock();
            try {
                while (list.size() == MAX_SIZE) {
                    listAtCapacity.await();
                }
                list.add(item);
            } finally {
                rLock.unlock();
            }
        }
    }
    



    
    
    public class MonitorSample {
        private List<String> list = new ArrayList<String>();
        private static final int MAX_SIZE = 10;
    
        private Monitor monitor = new Monitor();
        private Monitor.Guard listBelowCapacity = new Monitor.Guard(monitor) {
            @Override
            public boolean isSatisfied() {
                return (list.size() < MAX_SIZE);
            }
        };
    
        public void addToList(String item) throws InterruptedException {
            monitor.enterWhen(listBelowCapacity);
            try {
                list.add(item);
            } finally {
                monitor.leave();
            }
        }
    }
    


As you can see from the example, both have virtually the same number of lines of code.  The `Monitor` adds some complexity around the `Guard` object compared to the `ReentrantLock Condition`.  However, the clarity of the `addToList` method in `Monitor` more than makes up for it.  It could just be my personal preference, but I have always found the 

    
    
      while(something==true){
         condition.await()
    }

to be a little awkward. 


### Usage Guidelines


It should be noted that `enter` methods that return `void` should always take the form of:

    
    
    monitor.enter()
    try{
        ...work..
    }finally{
      monitor.leave();
    }
    


and `enter` methods that return a `boolean` should look like:

    
    
    if(monitor.enterIf(guard)){
      try{
           ...work..
      }finally{
          monitor.leave();
      }
    }else{
       .. monitor not available..
    }
    




### Boolean Conditions


There too many `enter` methods on the Monitor class to effectively cover in one post, so I am going to pick my top three and present then in order from minimal blocking to the maximum.




  1. **tryEnterIf** - threads will not wait to enter the monitor and will only enter if the guard condition returns true.


  2. **enterIf** - threads will wait to enter the monitor but only if the guard condition returns true.  There are also **enterIf** method signatures that allow for specifying a timeout as well as an **enterIfInterruptibly** version.


  3. **enterWhen** - threads will wait indefinitely for the monitor _and_ the condition to return true, but can be interrupted.  Likewise there are options to specify timeouts as well as an **enterWhenUniterruptibly** version.




### Conclusion


I have not yet had the chance to use the Monitor at work, but I can see the usefulness in the granularity of the boolean guard conditions.  I have written some basic example code and an accompanying unit test, that demonstrate some of the functionality covered in this post. They are available  [here](https://gist.github.com/1369371).  As always your comments/suggestions are welcome.  In my next post I will cover more of what can be found in Guava concurrency.


#### Resources






  * [Guava Project Home](http://code.google.com/p/guava-libraries/)


  * [Monitor API](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/util/concurrent/Monitor.html)


  * [Sample Code](https://gist.github.com/1369371)


 
