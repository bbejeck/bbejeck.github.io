---
author: Bill Bejeck
comments: true
date: 2011-12-23 05:48:58+00:00
layout: post
slug: guava-functions-java-8-lambdas
title: Guava Functions & Java 8 Lambdas
wordpress_id: 1512
categories:
- Guava
- java
tags:
- Guava
- java
- Java 8
- lamba
---

<img class="left" src="{{ site.media_url }}/images/red-lambda.png" /> I recently read Brian Goetz's [The State of the Lambda](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-4.html) and after reading that article I wanted to try using Java 8 lambda expressions.  In his article, Brian goes on to describe interfaces that have one method as "functional" interfaces. Functional interfaces are almost always used as anonymous classes, with the ActionListener being the canonical example.  These "functional" interfaces are the initial target for lambda expressions, with the main goal being removing much of the boilerplate or ceremony around their usage.  As I have been writing a series on the Guava library, the Function interface immediately came to mind as a prime candiate for trying lambda expressions.  My goal is simple, take the unit test from my [Guava Futures]()  blog (it makes heavy use of the Function interface) and convert it to use lambda expressions.  While I will describe the structure of the lamba expression as it pertains to the example presented, this post is not a tutorial on Java 8 lambda expressions.  Rather it's documenting my first attempts at using lambda expressions in Java.
<!--more-->


### Lambda Expressions


The first example is a test for the Futures.chain method, which takes a Function as one of it's arguments:

    
    
    Function<List<String>, ListenableFuture<List<Person>>> queryFunction =
          new Function<List<String>, ListenableFuture<List<Person>>>() {
                @Override
                public ListenableFuture<List<Person>> apply(final List<String> ids) {
                     return dbService.getPersonsByIdAsync(ids);
                }
            };
    
    ListenableFuture<List<String>> indexSearch = 
          luceneSearcher.searchAsync("firstName:martin");
    ListenableFuture<List<Person>> results = 
          Futures.chain(indexSearch,queryFunction,executorService);
    


Using lamba expressions it would now look like:

    
    
    Function<List<String>, ListenableFuture<List<Person>>> queryFunction =
          ids ->(dbService.getPersonsByIdAsync(ids));
    
      ListenableFuture<List<String>> indexSearch = 
          luceneSearcher.searchAsync("firstName:martin");
      ListenableFuture<List<Person>>results = 
          Futures.chain(indexSearch, queryFunction,executorService);
    


Keep in mind that from the code examples above the two highlighted sections are equivalent.  Let's take a look at line 2 and explain how this matches up with lines 1-7 from the first code example




  1. `ids` is the input to the apply method and corresponds to the ` final List<String> ids` parameter on line 4 in the first code example.  The types of the lamba expression are being inferred from the context in which it is being used, so we don't need to repeat them for the `ids` parameter.


  2. Then there is the arrow (->) token which is part of the general syntax of Java 8 lambda expressions in their current form


  3. Then we have the body of the lambda (dbService.getPersonsByIdAsync(ids)) which is a method call that returns a ListenableFuture that in turn yields a List of Person objects.  Note that we did not have to put a return statement, as this is a single expression and is evaluated and returned.


The next example is a utility method from the test that returned ListenableFutures by passing anonymous Callable instances into an ExecutorService:

    
    
    private ListenableFuture<List<Person>> getPersonsByFirstNameFuture(final String firstName, final boolean error) {
    return executorService.submit(new Callable<List<Person>>() {
                @Override
                public List<Person> call() throws Exception {
                    startSignal.await();
                    if (error) {
                        throw new RuntimeException("Ooops!");
                    }
                    List<String> ids = luceneSearcher.search("firstName:" + firstName);
                    return dbService.getPersonsById(ids);
                }
            });
    }
    


And here is the equivalent using a lambda expression:

    
    
    private ListenableFuture<List<Person>> getPersonsByFirstNameFuture(final String firstName, final boolean error) {
     return executorService.submit(() -> {startSignal.await();
                                           if (error) {
                                            throw new RuntimeException("Ooops!");
                                           }
                                         List<String> ids = luceneSearcher.search("firstName:" + firstName);
                                         return dbService.getPersonsById(ids);
                                      });
    }
    


In this example there are no input arguments so the expression starts with empty parentheses () on line 2.  There is the -> token, but in this example, the body contains multiple statements surrounded by { ...}.  Since there are multiple statements an explicit return statement is needed on line 7.


### Environment to Run Java 8


My current laptop is a MacBook Pro, so I needed to set up an environment to run Java 8 with lambda support.  Here are the steps I took: 




  1. Installed LinuxMint 12 on VirtualBox.


  2. Created a directory and shared it with the LinuxMint guest


  3. Installed the [developer preview](http://jdk8.java.net/lambda/) of java 8.


  4. To get the source and test source from the existing maven project I ran `mvn jar:jar jar:test-jar` and put the resulting jar files in the shared directory


  5. Placed all the dependencies in the shared directory (guava,lucene,h2 and junit)


  6. Re-wrote the unit test to use lambdas and ran the new test from the command line




### Conclusion


Although it will be some time before Java 8 with lambda support is released, what is available in the developer preview looks promising.Thanks for your time and as always comments and suggestions are welcomed


#### Resources






  * [State of the Lambda, Part 4](http://cr.openjdk.java.net/~briangoetz/lambda/lambda-state-4.html)


  * [Java 8 ](http://jdk8.java.net/lambda/)developer preview


  * [Source code](https://gist.github.com/1513247) for this post


  * [Guava Project Home](http://code.google.com/p/guava-libraries/)


  * [Source Code](https://github.com/bbejeck/guava-blog) for Guava blog series
