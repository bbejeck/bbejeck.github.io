---
layout: post
title: "Partially Applied Functions in Java"
date: 2015-07-17 08:53:03 -0400
permalink: /java_partial_functions
comments: true
categories:
- java
- functional 
- Java 8
tags:
- java
- functional 
- Java 8
keywords: 
- paritally applied
- Java 8
- functional
- functions
- java 
description: How to acheive partially applied functions in java 8.
---
Last year I completed an intro to [functional progamming course](https://courses.edx.org/courses/DelftX/FP101x/3T2014/info) on edX.  The language used in the course was [haskell](https://wiki.haskell.org/Haskell).  I found working in haskell enjoyable. One of my favorite features is functions taking more than one parameter can be partially applied functions automatically.  For example, if you have a function expecting 3 parameters you can pass only the first parameter a function expecting the other two is returned.  But you could supply only one more parameter and a function that accepts the final one will be returned (I believe this is the default behavior for all functions in haskell).  I have used partially applied functions before from working in [scala](http://www.scala-lang.org/), but for some reason, this time the power and implications made more of an impression on me.  For a better explaination of functions and partially applied functions in haskell, go to [Learn You a Haskell](http://learnyouahaskell.com/higher-order-functions#curried-functions).  Now we need to move on to the point of this post, how can we achieve this behavior in java 8.
<!-- more -->

### The Case for Partially Applied Functions
First a little more background information will be helpful.  Functions that accept functions as arguments or return functions are called [*higher order functions*](https://en.wikipedia.org/wiki/Higher-order_function).  Higher order functions are very powerful for solving problems.  Having partially applied functions means we  don't have to have *all* the parameters at one time.  Instead we can gather bits of information as we go and when we have all the required information we finally compute our result.  Partially applied functions also provide great flexibilty in composing new functions.  By simply providing different inital or seed values, we have an almost unlimted abiltity to handle different situations in our code.  Let's look at an example.
### Partially Applied Functions Example
Now we'll show how to use partially applied functions in Java 8.  Here's our base functions:
```java
Function<Integer,Function<Integer,Function<BinaryOperator<Integer>,Integer>>> someComputation = i1 -> i2 -> f -> f.apply(i1,i2);

    BinaryOperator<Integer> mult = (i,j) -> i * j;
    BinaryOperator<Integer> divide = (i,j) -> i / j;
    BinaryOperator<Integer> sumSquares = (i,j) -> (i*i) + (j*j);
```
 What we have is a function `someComputation` that will take two numbers and apply some sort of computation using both numbers.  We also have snuck in a differnt functional interface here [BinaryOperator](https://docs.oracle.com/javase/8/docs/api/java/util/function/BinaryOperator.html).  The `BinaryOperator` is sugar for a function that takes two paremters and has a return value of the same type. Admittedly, the syntax of partially applied functions in java can be cumbersome especially compared to scala or haskell. At least we can use this approach, as prior to java 8, using partially applied functions without lambda expressions would have not have been feasable.  This is a very simple example of partially applied functions, but I think it gets the point accross.  Here's our code in action in a unit test:
```java
public class PartiallyAppliedFunctionsTest {

    Function<Integer,Function<Integer,Function<BinaryOperator<Integer>,Integer>>> someComputation = i1 -> i2 -> f -> f.apply(i1,i2);

    BinaryOperator<Integer> mult = (i,j) -> i * j;
    BinaryOperator<Integer> divide = (i,j) -> i / j;
    BinaryOperator<Integer> sumSquares = (i,j) -> (i*i) + (j*j);

    int first = 10;
    int second = 5;

    Function<Integer,Function<BinaryOperator<Integer>,Integer>> partial1 = someComputation.apply(first);
    Function<BinaryOperator<Integer>,Integer> partial2 = partial1.apply(second);


    @Test
    public void test_multiplication(){
        assertThat(partial2.apply(mult),is(50));
    }

    @Test
    public void test_divide(){
        assertThat(partial2.apply(divide),is(2));
    }

    @Test
    public void test_sum_squares(){
        assertThat(partial2.apply(sumSquares),is(125));
    }

}
``` 

### Conclusion
Although our example is very basic, hopefully we can see the power and flexiblity provided by partially applied functions and how they can be a great tool to have at your disposal.  Thanks for your time.


### Resources
*   [Source Gist from Post](https://gist.github.com/bbejeck/1c47683390a915d0ef90)
*   [Functional Interfaces](https://docs.oracle.com/javase/8/docs/api/java/util/function/package-summary.html)
*   [Learn You A Haskell](http://learnyouahaskell.com/)
*   [Parially Applied Functions (wikipedia)](https://en.wikipedia.org/wiki/Partial_application) 


