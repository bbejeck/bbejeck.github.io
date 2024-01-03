---
author: Bill Bejeck
comments: true
date: 2012-02-24 06:29:47+00:00
layout: post
slug: java-7-watchservice
title: 'What''s New in Java 7: WatchService'
wordpress_id: 1950
categories:
- General
- java
tags:
- java
- Java 7
---

<img class="left" src="../assets/images/Java_Logo1-150x150.png" /> Of all the new features in Java 7, one of the more interesting is the WatchService, adding the capability to watch a directory for changes.  The WatchService maps directly to the native file event notification mechanism, if available.  If a native event notification mechanism is not available, then the default implementation will use polling.  As a result, the responsiveness, ordering of events and details available are implementation specific.
<!--more-->

### Watching A Directory


The Path interface implements the register method that takes a WatchService object and varargs of type WatchEvent.Kind as arguments.  There are 4 events to watch for:




  1. ENTRY_CREATE


  2. ENTRY_DELETE


  3. ENTRY_MODIFY


  4. OVERFLOW


While the first 3 types are self explanatory, OVERFLOW indicates that events may been lost or discarded. A WatchService is created by calling FileSystem.newWatchService().  Watching a directory is accomplished by registering a Path object with the WatchService:

    
    
    import static java.nio.file.StandardWatchEventKinds.*;
    Path path = Paths.get("/home");
    WatchService watchService = FileSystems.getDefault().newWatchService();
    WatchKey watchKey = path.register(watchService,ENTRY_CREATE,ENTRY_DELETE,ENTRY_MODIFY);
    


As we can see from the example, the register method returns a WatchKey object. The WatchKey is a token that represents the registration of the Path with the WatchService. 


### The WatchKey


As a result of the registration process, the WatchKey is in a 'ready' state and is considered valid.  A WatchKey remains valid until one of the following occurs:




  1. WatchKey.cancel() is called.


  2. The directory being watched is no longer available.


  3. The WatchService object is closed.




### Checking For Changes


When a change is detected, the WatchKey state is set to 'signaled' and it is placed in a queue for processing. Getting WatchKeys off the queue involves calling WatchService.poll() or WatchService.take(). Here is a basic example:

    
    
    private boolean notDone = true;
    while(notDone){
        try{
             WatchKey watchKey = watchService.poll(60,TimeUnit.SECONDS);
             List<WatchEvent.Kind<?>> events = watchKey.pollEvents();
             for(WatchEvent event : events){
                ...process the events
             }
             if(!watchKey.reset()){
                ...handle situation no longer valid
             }
         }catch(InterruptedException e){
                Thread.currentThread().interrupt();
         }
    


On line 5 we are calling the pollEvents method to retrieve all the events for this WatchKey object. On line 9 you'll notice a call to the reset method.  The reset method sets the WatchKey state back to 'ready' and returns a boolean indicating if the WatchKey is still valid.  If there are any pending events, then the WatchKey will be immediately re-queued, otherwise it will remain in the ready state until new events are detected.  Calling reset on a WatchKey that has been cancelled or is in the ready state will have no effect.  If a WatchKey is canceled while it is queued, it will reamin in the queue until retrieved. Cancellation could also happen automatically if the directory was deleted or is no longer available.


### Processing Events


Now that we have detected an event, how do we determine:




  1. On which directory did the event happen? (assuming more than one directory registered)


  2. What was the actual event? (assuming listening for more than one event)


  3. What was the target of the event, i.e what Path object was created,deleted or updated?


Jumping in to line 6 in the previous example, we will parse the needed information from a WatchKey and a WatchEvent:

    
    
     //WatchKey watchable returns the calling Path object of Path.register
     Path watchedPath = (Path) watchKey.watchable();
     //returns the event type
     StandardWatchEventKinds eventKind = event.kind();
     //returns the context of the event
     Path target = (Path)event.context();
    


On line 6 we see the WatchEvent.context method being invoked.  The context method will return a Path object if the event was a creation, delete or update and will be relative to the watched directory. It's important to know that when a event is received there is no guarantee that the program(s) performing the operation have completed, so some level of coordination may be required. 


### Conclusion


The WatchService is a very interesting feature of the new java.nio.file package in Java 7.  That being said, there are two things that about the WatchService to keep in mind:




  1. The WatchService does not pick up events for sub-directories of a watched directory.


  2. We still need to poll the WatchService for events, rather than receive asynchronous notification.


To address the above issues my next post will use the Guava EventBus to process the WatchService events. Thanks for your time


#### Resources






  1. [java.nio.file](http://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html) package that  contains the WatchService, WatchKey and WatchEvent objects discussed here.


  2. A [unit test](https://github.com/bbejeck/Java-7/blob/master/src/test/java/bbejeck/nio/files/watch/WatchDirectoryTest.java) demonstrating the WatchService


