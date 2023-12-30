---
layout: post
title: "Completable Futures - Error Handling."
date: 2018-10-30 20:23:23 -0400
permalink: /completable-futures-part2/
comments: true
categories: 
- java
- Java 8
- Concurrency
tags: 
- java
- Java 8
- Functions
- Lambdas
- Functional Interface
- CompletableFuture
keywords: 
- Java 8
- Java
- CompletableFuture
description: Error handling with Java CompletableFutures. 
---
Some time ago, over 2 years, I started a 3 part series on the `CompletableFuture`.  I'm just now getting around to doing part two now.  My long time delay in completing this series was due to working in my book [Kafka Streams in Action](URL).  But now that's done I can get back to doing some blogging again.

Earlier this year I started a series on a new class introduced in Java 8, the [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) class.  Since the `CompletableFuture` is such a feature rich class, I decided to break the coverage up into three stages.  The [first post](http://codingjunkie.net/completable-futures-part1/) covered the creation of `CompletableFuture` tasks and how to specify followup tasks to execute when the original one completes.  The examples in the first post only dealt with the happy path scenarios, however.  Today we are going to go over dealing with failures and errors including specifying actions to take when an error is encountered.  
<!-- more -->

### Why Having Separate Error Handling Methods
CompletableFutures give you the ability to define functionality that can be executed then you can come back to it later and magically extract the result of your asynchronous task. But there is one drawback, what to do when errors occur? You can use try-catch blocks, but you lose the conciseness of lambda syntax.  Plus you lose flexibility as you have to handle errors the same way, you can't use a CompleteableFuture with error handling strategy A then five lines later use a different error handling strategy (assuming you are passing the same lambda with different parameters).  What we need is a pluggable solution where different functions can be specified at any point to handle errors that best fit the specific situation.

### Error Handling Strategies
When it comes to handling errors with `Completeable` futures, there are two approaches.  The first is to run a given function when an `Exception` occurs.  The second is a [BiFunction](URL) with the expected result type and a [Throwable] as parameters.  If an error occurred, the `Throwable` instance is not null and you can take the action at that point.  While not error handling strategies, two methods will force an exception to be thrown when any attempt to call the `CompletableFuture.get` method is called. The two approaches differ in that the `CompleteableFuture.excepionally` is used when you don't want to take any further action with the result, if the future completes normally, then the returned result is good enough.  The only way the provided function executes is in the event of an error. But the case of `CompletableFuture.handle` method is different.  If the future completes without error, the result is available for extra processing.  Otherwise the `Throwable` parameter is not null and you can react at that point.  The key point here is `CompleteableFuture.handle` method is *always* executed. 

#### Functions That Run On Error
The first strategy for handling errors is the `CompletableFuture.exceptionally` method.  The `exceptionally` method takes a [Function](URL) that expects to receive an instance of [Throwable](URL) and returns the same type of the original `CompletebleFuture`.  If the case of normal completion, the result of the `CompletableFuture` is returned to the caller.  But if there are any errors then the supplied function is executed, and that result is returned instead.
//Code Here
In other words `CompleteableFuture.exceptionally` only runs when there is an exception.

#### Handling Success or Failure

The second method we have for error handling is `CompleteableFuture.handle`.  In contrast to `exceptionally` the `handle` method _always_ executes the function parameter.  We can see the difference in execution by the types the two methods accept.  The `exceptionally` method requires a `Function` returning the same type as the `CompletebleFuture`.  On the other hand, the `handle` method takes a `BiFunction` where the first parameter _is_
the result of the `CompleteableFuture` computation and the second parameter is a 
//Code Here
`Throwable`.  So in our function, if the `Throwable` is not null we know an error occurred and took the appropriate action.  Otherwise, we return the result of the `CompleteableFuture`.  Or course we can perform additional operations on a successful result as long as we return the same type.  

### Choosing a Strategy

We have two strategies for asynchronous error handling, so the question is which type to use? While there are no hard rules here's some quick advice:

* When the result stands alone use `CompleteableFuture.exceptional`.
* If the result requires more processing,  use `CompleteableFuture.handle` instead.

### Conclusion

We have reached the end of our coverage on `CompletableFuture` error handling.  The takeaway(s) here are don't add error handling inside your `CompletableFuture`.  Instead, rely on the error handling process provided by the class.  In next post on the `CompleteableFuture,` we'll cover canceling and forcing completion.

### Resources
*   [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
*   [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)
*   [Source Code](https://github.com/bbejeck/Java-8/blob/master/src/test/java/bbejeck/concurrent/CompletableFutureTest.java) for this post.
