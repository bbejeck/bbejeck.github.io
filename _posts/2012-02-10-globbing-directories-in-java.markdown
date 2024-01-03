---
author: Bill Bejeck
comments: true
date: 2012-02-10 07:13:04+00:00
layout: post
slug: globbing-directories-in-java
title: Creating An Asynchronous, Recursive DirectoryStream in Java 7
wordpress_id: 1865
categories:
- General
- java
tags:
- Concurrency
- java
- Java 7
---

<img class="left" src="../assets/images/Java_Logo1.png" /> Continuing with my series on the Java 7 java.nio.file package, this time covering the DirectoryStream interface.  In this post we are going implement our own DirectoryStream that will iterate over the files in an entire directory tree, not just a single directory.  Our goal in the end is to have something that works similar to Ruby's [Dir.glob("**")](http://ruby-doc.org/core-1.9.3/Dir.html#method-c-glob) method. 
<!--more-->

### Requirements


Here are the requirements:




  1. Starting from a given Path object, it should recursively go through each directory looking for files that match a given pattern.


  2. It needs to be a single method call, something like DirUtils.glob(Path path, String pattern).


  3. We want the search to be asynchronous, so we can iterate over files as they are found.


The plan to meet our objective is straight forward:


  1. Create a DirectoryStream.Filter object from the given pattern.


  2. Use the Files.walkFileTree method to recursively search for files that match our pattern.


  3. Run the search in a separate thread, and when a matching file is found, place it in a queue.


  4. The DirectoryStream iterator will pull the files out of the queue as they become available


**Disclosure**: This post is merely an attempt to see if making an asynchronous, recursive DirectorySteam will work, I make no guarantees about the code presented here, YMMV.


### The Filter


DirectoryStream has a static interface, Filter, that is used to determine if the path object should be accepted.  The filter object is constructed by first creating a [PathMatcher](http://docs.oracle.com/javase/7/docs/api/java/nio/file/FileSystem.html#getPathMatcher(java.lang.String)) object then using the PathMatcher in the Filter's accept method:

    
    
    public static DirectoryStream.Filter<Path> buildGlobFilter(String pattern) {
            final PathMatcher pathMatcher = getPathMatcher("glob:"+pattern);
            return new DirectoryStream.Filter<Path>() {
                @Override
                public boolean accept(Path entry) throws IOException {
                    return pathMatcher.matches(entry);
                }
            };
        }
    




### The Recursive Search


To perform the search we are going the use the Files.walkFileTree method with a [FunctionVisitor](https://github.com/bbejeck/Java-7/blob/master/src/main/java/bbejeck/nio/files/visitor/FunctionVisitor.java) object. FunctionVisitor is a subclass of SimpleFileVisitor that takes a Guava Function as a constructor argument. The provided function is called as each file is visited and is checked by the [DirectoryStream.Filter](http://docs.oracle.com/javase/7/docs/api/java/nio/file/DirectoryStream.Filter.html) object for a match.  We wrap the creation of the Function object in a method call:

    
    
     private Function<Path, FileVisitResult> getFunction(final Filter<Path> filter) {
            return new Function<Path, FileVisitResult>() {
                @Override
                public FileVisitResult apply(Path input) {
                    try {
                        if (filter.accept(input.getFileName())) {
                            pathsBlockingQueue.offer(input);
                        }
                    } catch (IOException e) {
                        throw new RuntimeException(e.getMessage());
                    }
                    return (pathTask.isCancelled()) ? FileVisitResult.TERMINATE : FileVisitResult.CONTINUE;
                }
            };
        }
    


On line 6 the filter is checking for a match on the filename. If a match is found, the path object is placed in a LinkedBlockingQueue, pathsBlockingQueue, on line 7. On line 12, we see the pathTask instance variable, which is a FutureTask handle to the search thread.  If pathTask has been cancelled we terminate the search, otherwise continue.


### The Search Thread


To run the search, we wrap the Files.walkFileTree method in a Callable object, which is used as a constructor argument to pathTask, the FutureTask object.  By using a FutureTask, we have a hook into canceling the search as well as being able to check it's running status.

    
    
     private void findFiles(final Path startPath, final Filter filter) {
            pathTask = new FutureTask<Void>(new Callable<Void>() {
                @Override
                public Void call() throws Exception {
                    Files.walkFileTree(startPath, new FunctionVisitor(getFunction(filter)));
                    return null;
                }
            });
            start(pathTask);
        }
    


On line 5 we see the call to the getFunction method from the previous example.  On line 9 the method call start is used to spin off the search in it's own thread:

    
    
    private void start(FutureTask<Void> futureTask) {
            new Thread(futureTask).start();
        }
    


We use a Thread object instead of an ExecutorService because we only need a single thread to execute once. Since we are not submitting any subsequent tasks, an ExecutorService really is not necessary.


### Implementing A DirectoryStream


To implement a DirectoryStream there are two methods that need to be overridden - iterator and close. DirectoryStream extends the AutoCloseable interface, so when used with the try-with-resources construct, the close method is automatically invoked, releasing any underlying resources. The most interesting part of this code is overriding the iterator method:

    
    
    public Iterator<Path> iterator() {
            confirmNotClosed();
            findFiles(startPath, filter);
            return new Iterator<Path>() {
                Path path;
                @Override
                public boolean hasNext() {
                    try {
                        path = pathsBlockingQueue.poll();
                        while (!pathTask.isDone() && path == null) {
                            path = pathsBlockingQueue.poll(5, TimeUnit.MILLISECONDS);
                        }
                        return (path != null) ? true : false;
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                    return false;
                }
    
                @Override
                public Path next() {
                    return path;
                }
    
                @Override
                public void remove() {
                    throw new UnsupportedOperationException("Removal not supported");
                }
            };
        }
    


On line 2 we check to see if the iterator was previously closed and throw an UnsupportedOperationException if that is the case.  Line 3 kicks off the search thread as we saw from the previous example.  The hasNext method is where the real "brains" of the class resides.  Since the search is asynchronous the DirectoryStream will attempt to iterate over the files immediately, but there needs to be some coordination with the search thread as it finds matching path objects and places them into the queue.  On line 9 we first call poll (a non blocking call) and set the result to the path variable defined on line 5.  If the path object is null we drop into a while loop that calls poll again, but this time it's a blocking call with a timeout of 5 milliseconds.  We'll stay in that loop until a non-null path is returned or the search thread has completed or cancelled.  On line 13 we check for null to determine if we have a valid result or if there are no more path objects to return.  If hasNext returned true, the next method returns the path object that was retrieved by the previous hasNext call.  This process will continue until all results have been returned. 


### Conclusion


Putting this all together we now have the [AsynchronousRecursiveDirectoryStream](https://github.com/bbejeck/Java-7/blob/master/src/main/java/bbejeck/nio/files/directory/AsynchronousRecursiveDirectoryStream.java) class.  By combining our new class with [DirUtils](https://github.com/bbejeck/Java-7/blob/master/src/main/java/bbejeck/nio/util/DirUtils.java) we can iterate over an entire directory tree:

    
    
    try(DirectoryStream<Path> directoryStream = DirUtils.glob(path,"*")){
       for(Path path : directoryStream){
          ....
        }
    }
    


It is hoped that the reader found this useful.  As always, comments and suggestions are welcomed. Thanks for your time.


#### Resources






  * [source code](https://github.com/bbejeck/Java-7/blob/master/src/main/java/bbejeck/nio/files/directory/AsynchronousRecursiveDirectoryStream.java) and [unit test](https://github.com/bbejeck/Java-7/blob/master/src/test/java/bbejeck/nio/files/directory/AsynchronousRecursiveDirectoryStreamTest.java) for material covered


  * [Java 7 java.nio.file api](http://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html)
