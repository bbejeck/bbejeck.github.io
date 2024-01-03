---
author: Bill Bejeck
comments: true
date: 2011-11-23 04:59:45+00:00
layout: post
slug: google-guava-concurrency-listenablefuture
title: Google Guava Concurrency - ListenableFuture
wordpress_id: 1206
categories:
- General
tags:
- Concurrency
- Guava
- java
---

<img class="left" src="../assets/images/toolbox.jpg" /> In my [last post](http://codingjunkie.net/google-guava-synchronization-with-monitor) I covered using the `Monitor` class from the com.google.common.util.concurrent package in the Guava Library.  In this post I am going to continue my coverage of Guava concurrency utilities and discuss the `ListenableFuture` interface. A `ListenableFuture` extends the `Future` interface from the java.util.concurrent package, by adding a method that accepts a completion listener. 
<!--more-->

### ListenableFuture


a `ListenableFuture` behaves in exactly the same manner as a `java.util.concurrent.Future` but has the method `addCallback(Runnable, ExecutorService)` that executes the callback in the given `executor`.  Here is an example:

    
    
     ListenableFuture futureTask = executorService.submit(callableTask)
     futureTask.addListener(new Runnable() {
                @Override
                public void run() {
                   ..work after futureTask completed
                }
            }, executorService);
    


If the submitted task has completed when you add the callback, it will run immediately.  Using the `addCallback` method has a drawback in that the `Runnable` does not have access to the result produced by the `future`.   For access to the result of the `Future` you would need to use a `FutureCallback`.


### FutureCallback


A `FutureCallback` accepts the results produced from the `Future` and specifies `onSuccess` and `onFailure` methods.   Here is an example:

    
    
     class FutureCallbackImpl implements FutureCallback<String> {
    
            @Override
            public void onSuccess(String result){
                 .. work with result
            }
    
            @Override
            public void onFailure(Throwable t) {
                ... handle exception
            }
        }
    


A `FutureCallback` is attached by using the `addCallback` method in the Futures class:

    
    
      Futures.addCallback(futureTask, futureCallbackImpl);
    


At this point you may be asking how do you get an instance of `ListenableFuture`, when an `ExecutorService` only returns `Futures`?  The answer is to use the `ListenableExecutionService`.


### ListenableExecutionService


To use a `ListenableExecutionService` simply decorate an `ExecutorService` instance with a call to `MoreExecutors.listeningDecorator(ExecutorService)` for example:

    
    
    ExecutorsService executorService = MoreExecutors.listeningDecorator(Executors.newCachedThreadPool());
    




### Conclusion


With the ability to add a callback, whether a `Runnable` or the `FutureCallback` that handles success and failure conditions, the `ListenableFuture` could be a valuable addition to your arsenal.  I have created a unit-test demonstrating using the `ListenableFuture` available as a [gist](https://gist.github.com/1387892).  In my next post I am going to cover the `Futures` class, which  contains static methods for working with `futures`.


#### Resources






  * [Guava Project Home](http://code.google.com/p/guava-libraries/)


  * [ListenableFuture API](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/util/concurrent/ListenableFuture.html)


  * [Sample Code](https://gist.github.com/1387892)


