---
layout: post
title: "FlatMap in Guava"
date: 2015-05-19 22:51:34 -0400
permalink: /guava_flatmap
comments: true
categories:
- Guava
- functional 
- java
tags:
- Guava
- functional 
- java
keywords: 
- Guava
- flatMap
- functional
- java
description: Using a flatMap function in Guava.
---
This is a short post about a method I recently discovered in Guava.  
### The Issue
I had a situation at work where I was working with objects structured something like this:
```java
public class Outer {
    String outerId;
    List<Inner> innerList;
    .......
}

public class Inner {
    String innerId;
    Date timestamp;
}

public class Merged {
    String outerId;
    String innerId;
    Date timestamp;
}
```
My task was flatten a list `Outer` objects (along with the list of `Inner` objects) into a list of `Merged` objects. Since I'm working with Java 7, using streams is not an option. 
<!-- more --> 
### The First Solution
Instead I turn to the [FluentIterable](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/FluentIterable.html) class from [Guava](https://code.google.com/p/guava-libraries/).  My first instinct is to go with the [FluentIterable.transform](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/FluentIterable.html#transform\(com.google.common.base.Function\)) method (which is essentially a map function):
```java
List<Outer> originalList = getListOfObjects();

Function<Outer,List<Merged>> flattenFunction //Details left out for clarity

//returns an Iterable of Lists!
Iterable<List<Merged>> mergedObjects = FluentIterable.from(originalList).tranform(flattenFunction);
```
But I really want a single collection of `Merged` objects, not an iterable of lists! The missing ingredient here is a flatMap function.  Since I'm not using Scala, Clojure or Java 8, I feel that I'm out of luck.
### A Better Solution
I decide to take a closer look at the `FluentIterable` class and I discover the [FluentIterable.transformAndConcat](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/FluentIterable.html#transformAndConcat\(com.google.common.base.Function\)) method. The transformAndConcat method applies a function to each element of the fluent iterable and appends the results into a *single* iterable instance.  I have my flatMap function in Guava!  Now my solution looks like this:
```java

List<Outer> originalList = getListOfObjects();

Function<Outer,List<Merged>> flattenFunction //Details left out for clarity

Iterable<Merged> mergedObjects = FluentIterable.from(originalList).transformAndConcat(flattenFunction);
```
### Conclusion
While this is a very short post, it goes to show how useful the Guava library is and how functional programming concepts can make our code more concise.






