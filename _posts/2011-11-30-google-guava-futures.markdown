---
author: Bill Bejeck
comments: true
date: 2011-11-30 03:29:27+00:00
layout: post
slug: google-guava-futures
title: Google Guava - Futures
wordpress_id: 1246
categories:
- Concurrency
- java
tags:
- Concurrency
- Guava
- java
---

<img class="left" src="{{ site.media_url }}/images/toolbox.jpg" /> This post is a continuation of my series on Google Guava, this time covering Futures.   The Futures class is a collection of static utility methods for working with the Future/ListenableFuture interface.  A Future is a handle to an asynchronous task, either a Runnable or Callable, that was submitted to an ExecutorService.  The Future interface provides methods for: getting the results of a task, checking if a task is done, or canceling a task.  The ListenableFuture interface extends the Future interface and adds the ability to set a completion listener to run once a task is finished. To create a ListenableFuture you first need to decorate an ExecutorService instance like so:
<!--more-->
    
    
    ExecutorService executorService = MoreExecutors.listeningDecorator(Executors.newCachedThreadPool());
    


Now all submitted Callables/Runnables will return a ListenableFuture.  [MoreExecutors](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/util/concurrent/MoreExecutors.html) can be found in the com.google.common.util.concurrent package.  ListenableFutures were covered in the [previous post](http://codingjunkie.net/google-guava-concurrency-listenablefuture).  There are too many methods in Futures to effectively cover in one post, so I am only going to cover: chain, transform, allAsList and successfulAsList.  Throughout this post, I will refer to Futures and ListenableFutures interchangeably.


### Chain


The chain method returns a ListenableFuture whose value is calculated by taking the results from an input Future and applying it as an argument a [Function](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/base/Function.html) object, which in turn, returns another ListenableFuture. Let's look at a code example and step through it: 
[java]
ListenableFuture<List<String>> indexSearch = 
              luceneSearcher.searchAsync("firstName:martin");

Function<List<String>, ListenableFuture<List<Person>>> queryFunction = 
 new Function<List<String>, ListenableFuture<List<Person>>>() {
            @Override
            public ListenableFuture<List<Person>> apply(final List<String> ids) {
                 return dataService.getPersonsByIdAsync(ids);
            }
        };

ListenableFuture<List<Person>> results = 
              Futures.chain(indexSearch, queryFunction,executorService);
[/java]




  1. Line 1 is executing an asynchronous search using Lucene and will return a list of id's that represent the primary key of a person record stored in a database. (I created a small index where the only data stored in Lucene is the id's and the rest of the data was indexed only).


  2. Lines 4 - 11 are building the function object where the apply method will use the results from the search future as input.  The future returned from apply is a result of the call to the dataService object.


  3. Line 12 is the future that is returned from the chain call.  The executorService is used to run the function once the input future is completed.


For additional clarity, here is what the searchAsync and getPersonsByIdAsync methods are doing.  These method calls are from lines 2 and 8 respectively, in the previous code example:

    
    
    
    public ListenableFuture<List<String>> searchAsync(final String query)  {
    	        return executorService.submit(new Callable<List<String>>() {
    	            @Override
    	            public List<String> call() throws Exception {
    	                return search(query);
    	            }
    	        });
      }
    
    public ListenableFuture<List<Person>> getPersonsByIdAsync(final List<String> ids) {
            return executorService.submit(new Callable<List<Person>>() {
                @Override
                public List<Person> call() throws Exception {
                    return getPersonsById(ids);
                }
            });
        }
    


The chain method has two signatures:




  1. chain(ListentableFuture, Function)


  2. chain(ListenableFuture, Function, ExecutorService)


When determining which method to use there are a few points to consider.
If the input future is completed by the time chain is called, the supplied function will be executed immediately in the calling thread.  Additionally, if no executor is provided then a MoreExecutors.sameThreadExecutor is used. MoreExecutors.sameThreadExecutor (as the name implies) follows the ThreadPoolExecutor.CallerRunsPolicy, meaning the submitted task is run in the same thread that called execute/submit. 



### Transform


The transform method is similar to the chain method, in that it takes a Future and Function object as arguments.  The difference is that a ListenableFuture is not returned, just the results of applying the given function to the results of the input future.  Consider the following:

    
    
    List<String> ids = ....
    ListenableFuture<List<Map<String, String>>> dbRecords =   
                                               dataService.getPersonDataByIdAsync(ids);
    
     Function<List<Map<String, String>>,List<Person>> transformDbResults = 
      new Function<List<String>, List<Person>>() {
                  @Override
                  public List<Person> apply(List<Map<String, String>> personMapList) {
                          List<Person> personObjList = new ArrayList<Person>();
                          for(Map<String,String> personDataMap : personMapList){
                                 personObjList.add(new Person(personDataMap);
                          } 
                          return personObjList;
                        }
         };
    
    
    ListenableFuture<List<Person>> transformedResults = 
                          Futures.transform(dbRecords, transformDbResults, executorService);
    






  1. On line 2 an asynchronous database lookup is performed


  2. On line 4 a function object is being created, but on line 8, notice the return type is List<Person>


The transform method has the same overloaded method calls as chain, with the same caveats. 


### AllAsList


The allAsList method will take an arbitrary number of ListenableFutures either as varargs or in the form of an Iterator<ListenableFuture>.  A ListenableFuture is returned whose value is a list of all of the results of the inputs.  The values returned in the list are in the same order as the original list. If any of the input values were cancelled or failed then the returned ListenableFuture will be cancelled or fail as well.  Canceling the returned future from the allAsList call will not propagate down to any of the original tasks submitted in the list.

    
    
    ListenableFuture<List<Person>> lf1 = getPersonsByFirstNameFuture("martin");
    ListenableFuture<List<Person>> lf2 = getPersonsByFirstNameFuture("bob");
    ListenableFuture<List<List<Person>>> lfResults = Futures.allAsList(lf1, lf2);
    //assume lf1 failed
    List<List<Person>> personLists = lfResults.get() //call results in exception
    





### SuccessfulAsList


The successfulAsList method is very similar to allAsList, but is more forgiving.  Just like allAsList, successfulAsList returns a list of results in the same order as the input list, but if any of the inputs failed or were cancelled the corresponding value in the list is null. Canceling the returned future will not cancel any of the original inputs as well.    

    
    
    ListenableFuture<List<Person>> lf1 = getPersonsByFirstNameFuture("martin");
    ListenableFuture<List<Person>> lf2 = getPersonsByFirstNameFuture("bob");
    ListenableFuture<List<List<Person>>> lfResults = Futures.successfulAsList(lf1, lf2);
    //assume lf1 failed
    List<List<Person>> personLists = lfResults.get();
    List<Person> listOne = personLists.get(0) //listOne is null
    List<Person> listTwo = personLists.get(1) //listTwo, not null 
    





### Conclusion


Hopefully this was helpful in discovering the usefulness contained in the Futures class from Google Guava.  I have created a unit test that shows example usages of the methods described in this post.  As there is a fair amount of supporting code, I have created a project on  gihub, [guava-blog](https://github.com/bbejeck/guava-blog).  This project will also contain the source code from my previous posts ([Monitor](http://codingjunkie.net/google-guava-synchronization-with-monitor), [ListenableFuture](http://codingjunkie.net/google-guava-concurrency-listenablefuture)) on Guava.  As always comments and suggestions are welcomed.



#### Resources






  * [Guava Project Home](http://code.google.com/p/guava-libraries/)


  * [Futures API](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/util/concurrent/Futures.html)


  * [Source Code](https://github.com/bbejeck/guava-blog) for blog series



