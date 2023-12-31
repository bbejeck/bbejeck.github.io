---
layout: post
title: "Java 8 CompletableFutures Part I"
date: 2016-01-23 16:10:47 -0500
permalink: /completable-futures-part1/
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
description: How to work with Java 8 CompletableFutures.
---
When Java 8 was released a while ago, a great concurrency tool was added, the [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) class.  The `CompletableFuture` is a [Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html) that can have it's value explicity set and more interestingly can be chained together to support dependent actions triggered by the CompletableFutures completion.  CompletableFutures are analogous to the [ListenableFuture](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/util/concurrent/ListenableFuture.html) class found in Guava.  While the two offer similar functionality, there won't be any comparisons done in this post.  I have previously covered ListenableFutures in [this post](http://codingjunkie.net/google-guava-futures/).  While the coverage of ListenableFutures is a little dated, most of the information should still apply.  The documentation for the CompletableFuture class is comprehensive, but lacks concrete examples of how to use them. My goal is show how to use CompletableFutures through a series of simple examples in unit tests. Originally I was going to cover the CompleteableFuture in one post, but there is so much information, it seems better to break up coverage into 3 parts - 

1.    Creating/combining tasks and adding listeners for follow on work.
2.    Handling errors and error recovery
3.    Canceling and forcing completion.
<!-- more -->

### CompletableFuture Primer
Before we dig into using CompleteableFutures, some background information is needed.  The CompleteableFuture implements the [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) interface.  The javadoc consisely explains what the CompletionStage is:

>A stage of a possibly asynchronous computation, that performs an action or computes a value when another CompletionStage completes. A stage completes upon termination of its computation, but this may in turn trigger other dependent stages.  

The full documentation for the CompletionStage is too long to include here, so we'll briefly summarize the key points:

1.   Computations can be be represented by a [Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html), [Consumer](https://docs.oracle.com/javase/8/docs/api/java/util/function/Consumer.html) or a [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html) with the respective method names of *apply*, *accept* or *run* 
2.    Execution of computations can be one of the following 
      3. Default execution (possibly the calling thread)
      4. Async execution using the default async execution provider of the CompletionStage.  These methods are denoted by the form of *someActionAsync*
      5. Async execution by using a provided [Executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html).  These methods also follow the form of *someActionAsync* but take an `Executor` instance as an additonal parameter.

For the rest of this post I will be refering to `CompletableFuture` and `CompletionStage` interchangably.

###Creating A CompleteableFuture
Creating a CompletableFuture is simple, but not always clear.  The simplest way is the `CompleteableFuture.completedFuture` method which returns an a new, finished CompleteableFuture:
```java
    @Test
    public void test_completed_future() throws Exception {
      String expectedValue = "the expected value";
      CompletableFuture<String> alreadyCompleted = CompletableFuture.completedFuture(expectedValue);
      assertThat(alreadyCompleted.get(), is(expectedValue));
    }
```    
As unexciting as this may seem, the ability to create an already completed CompleteableFuture can come in handy as we'll see a little later.  
Now let's take a look at how to create a `CompletableFuture` that represents an asynchronous task:
```java
private static ExecutorService service = Executors.newCachedThreadPool();

    @Test
    public void test_run_async() throws Exception {
        CompletableFuture<Void> runAsync = CompletableFuture.runAsync(() -> System.out.println("running async task"), service);
        //utility testing method
        pauseSeconds(1);
        assertThat(runAsync.isDone(), is(true));
    }

    @Test
    public void test_supply_async() throws Exception {
        CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(simulatedTask(1, "Final Result"), service);
        assertThat(completableFuture.get(), is("Final Result"));
    }
```
In the first code sample we see an example of a `runAsync` task and the second sample is an example of `supplyAsync`.  This may be stating the obvious, but the decision to use supplyAsync vs runAsync is determined by whether the task is expected to return a value or not.  In both examples here we are supplying a custom `Executor` which is the asynchronous execution provider.  When it comes to the supplyAsync method I personally think it would have been more natural to use a [Callable](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Callable.html) and not a `Supplier`.  While both are [functional interfaces](https://docs.oracle.com/javase/8/docs/api/java/lang/FunctionalInterface.html), the `Callable` is associated more with asynchronous tasks and can throw checked exceptions while the `Supplier` does not (although with a small amount of code we can have [Suppliers that throw](https://github.com/bbejeck/Java-8/blob/master/src/main/java/bbejeck/function/throwing/ThrowingSupplier.java) checked exceptions).

### Adding Listeners
Now that we can create `CompletableFuture` objects to run asynchronous tasks, let's learn how to 'listen' when a task completes to perform follow up action(s).  It's important to mention here that when adding follow on `CompletionStage` objects, the previous task needs to complete *successfully* in order for the follow on task/stage to run.  There are methods to deal with failed tasks, but handling errors in the `CompletableFuture` chain are covered in a follow up post.

```java
   @Test
    public void test_then_run_async() throws Exception {
        Map<String,String> cache = new HashMap<>();
        cache.put("key","value");
        CompletableFuture<String> taskUsingCache = CompletableFuture.supplyAsync(simulatedTask(1,cache.get("key")),service);
        CompletableFuture<Void> cleanUp = taskUsingCache.thenRunAsync(cache::clear,service);
        cleanUp.get();
        String theValue = taskUsingCache.get();
        assertThat(cache.isEmpty(),is(true));
        assertThat(theValue,is("value"));
    }
```
Here in this example we are running a task that "cleans up" after the first `CompletableFuture` finishes sucessfully. While the previous example used a `Runnable` task to execute after the original task completed successfully, there really is no connection between the two.  We can also specify a follow on task that takes the result of the previous successful task directly:

```java
@Test
public void test_accept_result() throws Exception {
        CompletableFuture<String> task = CompletableFuture.supplyAsync(simulatedTask(1, "add when done"), service);
        CompletableFuture<Void> acceptingTask = task.thenAccept(results::add);
        pauseSeconds(2);
        assertThat(acceptingTask.isDone(), is(true));
        assertThat(results.size(), is(1));
        assertThat(results.contains("add when done"), is(true));

    }
```
This is an example of *Accept* methods that take the result of the `CompletableFuture` and pass it to a `Consumer` object.  In Java 8 `Consumer` instances have no return value and are expected to work by side-effects, in this case adding the result to a list.

### Combining And Composing Tasks
In addition to adding listeners to run follow up tasks or accept the results of a succesful CompletableFuture, we can combine and/or compose tasks.   

#### Composing Tasks
Composing means taking the results of one successful CompletableFuture as input to another CompletableFuture via a [Function](https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html). 
Here's an example of `CompletableFuture.thenComposeAsync`
```java
@Test
public void test_then_compose() throws Exception {

 Function<Integer,Supplier<List<Integer>>> getFirstTenMultiples = num ->
                ()->Stream.iterate(num, i -> i + num).limit(10).collect(Collectors.toList());

 Supplier<List<Integer>> multiplesSupplier = getFirstTenMultiples.apply(13);

 //Original CompletionStage
 CompletableFuture<List<Integer>> getMultiples = CompletableFuture.supplyAsync(multiplesSupplier, service);

 //Function that takes input from orignal CompletionStage
 Function<List<Integer>, CompletableFuture<Integer>> sumNumbers = multiples ->
            CompletableFuture.supplyAsync(() -> multiples.stream().mapToInt(Integer::intValue).sum());

  //The final CompletableFuture composed of previous two.
  CompletableFuture<Integer> summedMultiples = getMultiples.thenComposeAsync(sumNumbers, service);

   assertThat(summedMultiples.get(), is(715));
  }
```
In this example the first CompletionStage is providing a list of 10 multiples of a number, 13 in this case.  The supplied Function takes those results and creates another CompletionStage which then sums the list of numbers.



#### Combining Tasks
Combining is accomplished by taking 2 successful CompletionStages and having the results from both used as parameters to a [BiFunction](https://docs.oracle.com/javase/8/docs/api/java/util/function/BiFunction.html) to produce another result.  Here's a very simple example to demonstrate taking results from combined CompletionStages.

```java
@Test
public void test_then_combine_async() throws Exception {
 CompletableFuture<String> firstTask = CompletableFuture.supplyAsync(simulatedTask(3, "combine all"), service);

 CompletableFuture<String> secondTask = CompletableFuture.supplyAsync(simulatedTask(2, "task results"), service);

 CompletableFuture<String> combined = firstTask.thenCombineAsync(secondTask, (f, s) -> f + " " + s, service);

 assertThat(combined.get(), is("combine all task results"));
}
```
While the previous example showed combining two CompletionStages that could be asynchronous tasks, we could also combine an asynchronous task with an already completed CompletableFuture.  It is good way to combine a known value with a value that needs to be computed:
```java
@Test
public void test_then_combine_with_one_supplied_value() throws Exception {
 CompletableFuture<String> asyncComputedValue = CompletableFuture.supplyAsync(simulatedTask(2, "calculated value"), service);
 CompletableFuture<String> knowValueToCombine = CompletableFuture.completedFuture("known value");

 BinaryOperator<String> calcResults = (f, s) -> "taking a " + f + " then adding a " + s;
 CompletableFuture<String> combined = asyncComputedValue.thenCombine(knowValueToCombine, calcResults);

 assertThat(combined.get(), is("taking a calculated value then adding a known value"));
}
``` 
Finally here's an example of using the `CompletableFuture.runAfterbothAsync`
```java
@Test
public void test_run_after_both() throws Exception {

CompletableFuture<Void> run1 = CompletableFuture.runAsync(() -> {
        pauseSeconds(2);
        results.add("first task");
    }, service);

CompletableFuture<Void> run2 = CompletableFuture.runAsync(() -> {
        pauseSeconds(3);
        results.add("second task");
    }, service);

CompletableFuture<Void> finisher = run1.runAfterBothAsync(run2,() -> results. add(results.get(0)+ "&"+results.get(1)),service);
 pauseSeconds(4);
 assertThat(finisher.isDone(),is(true));
 assertThat(results.get(2),is("first task&second task"));
}
```

### Listening For The First Finished Task
In all of the previous examples final results required waiting for all CompletionStages to finish, but this doesn't always need to be the case.  We can get results from which ever task completes first.  Here's an example where the first completed result is accepted using a `Consumer`:
```java
@Test
public void test_accept_either_async_nested_finishes_first() throws Exception {

 CompletableFuture<String> callingCompletable = CompletableFuture.supplyAsync(simulatedTask(2, "calling"), service);
 CompletableFuture<String> nestedCompletable = CompletableFuture.supplyAsync(simulatedTask(1, "nested"), service);

 CompletableFuture<Void> collector = callingCompletable.acceptEither(nestedCompletable, results::add);

 pauseSeconds(2);
 assertThat(collector.isDone(), is(true));
 assertThat(results.size(), is(1));
 assertThat(results.contains("nested"), is(true));
}
```
And the analogous `CompletableFuture.runAfterEither` 
```java
 @Test
  public void test_run_after_either() throws Exception {

 CompletableFuture<Void> run1 = CompletableFuture.runAsync(() -> {
            pauseSeconds(2);
            results.add("should be first");
    }, service);

 CompletableFuture<Void> run2 = CompletableFuture.runAsync(() -> {
            pauseSeconds(3);
            results.add("should be second");
    }, service);

  CompletableFuture<Void> finisher = run1.runAfterEitherAsync(run2,() -> results.add(results.get(0).toUpperCase()),service);

 pauseSeconds(4);
 assertThat(finisher.isDone(),is(true));
 assertThat(results.get(1),is("SHOULD BE FIRST"));
 }
```
### Multiple Combinations
Up to this point, all combining/composing examples have been two CompletableFuture objects only.  This was done intentionally in an effort to make the examples more clear. But we can nest an arbitrary number of CompletionStages together. Please note that the following example is for illustration purposes only!

```java

    Test
    public void test_several_stage_combinations() throws Exception {
        Function<String,CompletableFuture<String>> upperCaseFunction = s -> CompletableFuture.completedFuture(s.toUpperCase());

        CompletableFuture<String> stage1 = CompletableFuture.completedFuture("the quick ");

        CompletableFuture<String> stage2 = CompletableFuture.completedFuture("brown fox ");

        CompletableFuture<String> stage3 = stage1.thenCombine(stage2,(s1,s2) -> s1+s2);

        CompletableFuture<String> stage4 = stage3.thenCompose(upperCaseFunction);

        CompletableFuture<String> stage5 = CompletableFuture.supplyAsync(simulatedTask(2,"jumped over"));

        CompletableFuture<String> stage6 = stage4.thenCombineAsync(stage5,(s1,s2)-> s1+s2,service);

        CompletableFuture<String> stage6_sub_1_slow = CompletableFuture.supplyAsync(simulatedTask(4,"fell into"));

        CompletableFuture<String> stage7 = stage6.applyToEitherAsync(stage6_sub_1_slow,String::toUpperCase,service);

        CompletableFuture<String> stage8 = CompletableFuture.supplyAsync(simulatedTask(3," the lazy dog"),service);

        CompletableFuture<String> finalStage = stage7.thenCombineAsync(stage8,(s1,s2)-> s1+s2,service);

        assertThat(finalStage.get(),is("THE QUICK BROWN FOX JUMPED OVER the lazy dog"));

    }
```
   It's important to note that ordering is not guarateed when combining CompletionStages.  In these unit tests, times were provided to the simulated tasks to ensure completion order.

### Conclusion

   This wraps up the first part of using the CompletableFuture class.  In upcoming posts we'll cover error handling/recovery and forcing completion/cancellation.

### Resources
*   [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
*   [CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)
*   [Source Code](https://github.com/bbejeck/Java-8/blob/master/src/test/java/bbejeck/concurrent/CompletableFutureTest.java) for this post.
