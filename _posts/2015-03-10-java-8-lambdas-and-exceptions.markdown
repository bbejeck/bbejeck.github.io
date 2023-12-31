---
layout: post
title: Java 8 Functional Interfaces and Checked Exceptions
date: 2015-03-16 22:50:24 -0400
permalink: /functional-iterface-exceptions
comments: true
categories: 
- java
- Java 8
- Functions
- Lambdas
tags: 
- java
- Java 8
- Functions
- Lambdas
- Functional Interface
- Checked Exceptions
keywords: 
description: How to work with Java 8 Functions and Functional interfaces that throw checked exceptions
---
 The Java 8 lambda syntax and functional interfaces have been a productivity boost for Java developers.   But there is one drawback to functional interfaces.  None of the them as currently defined in Java 8 declare any checked exceptions.   This leaves the developer at odds on how best to handle checked exceptions.  This post will present one option for handling checked exceptions in functional interfaces.  We will use use the `Function` in our example, but the pattern should apply to any of the functional interfaces.
### Example of a Function with a Checked Exception
 Here's an example from a recent side-project using a `Function` to open a [directory in Lucene](http://lucene.apache.org/core/5_0_0/core/org/apache/lucene/store/FSDirectory.html).  As expected, opening a directory for writing/searching throws an `IOException`:

``` java
   private Function<Path, Directory> createDirectory =  path -> {
        try{
            return FSDirectory.open(path);
        }catch (IOException e){
            throw new RuntimeException(e);
        }
    };
```
<!-- more -->


While this example works, it feels a bit awkward with the `try/catch` block.  It's adding to the boilerplate that functional interfaces are trying to reduce.  Also, we are just re-throwing the exception up the call stack.  If this how we are going to handle exceptions, is there something we can do to make our code a little bit cleaner?

### A Proposed Solution
The solution is straight forward.  We are going to extend the `Function` interface and add a method called `throwsApply`. The `throwsApply` method  declares a throws clause of type `Exception`.  Then we override the `apply`method as a default method to handle the call to `throwsApply` in a try/catch block.  Any exceptions caught are re-thrown as `RuntimeExceptions`   
```java
@FunctionalInterface
public interface ThrowingFunction<T,R> extends Function<T,R> {

    @Override
    default R apply(T t){
        try{
            return applyThrows(t);
        }catch (Exception e){
            throw new RuntimeException(e);
        }
    }

R applyThrows(T t) throws Exception;
}
```
Here we are doing exactly what we did in the previous example, but from within a functional interface. 
Now we can rework our previous example to this: (made even more concise by using a method handle)
```java
private ThrowingFunction<Path, Directory> createDirectory = FSDirectory::open;
``` 

### Composing Functions that have Checked Exceptions
Now we have another issue to tackle.   How do we compose two or more functions involving checked exceptions?  The solution is to create two new default methods. We create `andThen` and `compose` methods allowing us to compose `ThrowingFunction` objects into one.  (These methods have the same name as the default methods in the `Function` interface for consistency.) 

```java
default <V> ThrowingFunction<T, V> andThen(ThrowingFunction<? super R, ? extends V> after){
        Objects.requireNonNull(after);
        try{
             return (T t) -> after.apply(apply(t));
        }catch (Exception e){
            throw new RuntimeException(e);
        }
  }

default <V> ThrowingFunction<V, R> compose(ThrowingFunction<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        try {
            return (V v) -> apply(before.apply(v));
        }catch (Exception e){
            throw new RuntimeException(e);
        }
  }
```  
The code for the`andThen` and `compose` is the same found in the original `Function` interface.  We've just added try/catch blocks and re-throw any `Exception` as a `RuntimeException`.  Now we are handling exceptions in the same manner as before with the added benefit of our code being a little more concise.

### Caveats
This approach is not without its drawbacks. 
Brian Goetz spoke about this in his blog post ['Exception transparency in Java'](https://blogs.oracle.com/briangoetz/entry/exception_transparency_in_java).  Also, when composing functions we must use the type of `ThrowingFunction` for all parts.  This is regardless if some of the functions don't throw any checked exceptions.  There is one exception, the last function added could be of type `Function`.   

### Conclusion
The purpose of this post is not to say this the best approach for handling checked exceptions, but to present one option.  In a future post we will look at other  ways of handling checked exceptions in functional interfaces.


### Resources
* [Source for post](https://gist.github.com/bbejeck/890dc7b743411271eba7)
* [Stackoverflow topic](http://stackoverflow.com/questions/18198176/java-8-lambda-function-that-throws-exception)
* [Functional Programming in Java](https://pragprog.com/book/vsjava8/functional-programming-in-java) presents good coverage of how to handle checked exceptions in functional interfaces.
* [Java 8 Lambdas](http://www.amazon.com/Java-Lambdas-Pragmatic-Functional-Programming/dp/1449370772/ref=sr_1_1?ie=UTF8&qid=1426211120&sr=8-1&keywords=java+8+lambda) 
* [Throwing Checked Exceptions From Lambdas](http://www.javaspecialists.eu/archive/Issue221.html)
 


