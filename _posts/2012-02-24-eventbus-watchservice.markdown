---
author: Bill Bejeck
comments: true
date: 2012-02-24 06:30:05+00:00
layout: post
slug: eventbus-watchservice
title: 'Event Programming Example: Google Guava EventBus and Java 7 WatchService'
wordpress_id: 2004
categories:
- java
tags:
- Guava
- Java 7
---

<img class="left" src="{{ site.media_url }}/images/Java_Logo1-150x150.png" /> This post is going to cover using the Guava EventBus to publish changes to a directory or sub-directories detected by the Java 7 WatchService.  The Guava EventBus is a great way to add publish/subscribe communication to an application.  The WatchService, new in the Java 7 java.nio.file package, is used to monitor a directory for changes. Since the EventBus and WatchService have been covered in previous posts, we will not be covering these topics in any depth here.  For more information, the reader is encouraged to view the [EventBus](http://codingjunkie.net/guava-eventbus/) and [WatchService](http://codingjunkie.net/java-7-watchservice/) posts.  [NOTE: post updated on 02/28/2012 for clarity.]
<!--more-->

### Why Use the EventBus


There are two main reasons for using the EventBus with a WatchService.




  1. We don't want poll for events, but would rather receive asynchronous notification.


  2. Once events are processed, the WatchKey.reset method needs to be called to enable any new changes to be queued.  While the WatchKey object is thread safe, it's important that the reset method is called only after all threads have finished processing events, leading to somewhat of a coordination hassle.  Using a single thread to process the events, invoke the reset method, then publish the changes via the EventBus, eliminates this problem.


Our plan to accomplish this is simple and will involve taking the following steps:


  1. Instantiate an instance of the WatchService.


  2. Register every directory recursively, starting with a given Path object.


  3. Take events off the WatchService queue, then process and publish those events.


  4. Start up a separate thread for taking events off the queue and publishing.


The code examples that follow are the more relevant highlights from the [DirectoryEventWatcherImpl](https://github.com/bbejeck/Java-7/blob/master/src/main/java/bbejeck/nio/files/directory/event/DirectoryEventWatcherImpl.java) class that is going to do all of this work.


### Registering Directories with the WatchService


While adding or deleting a sub-directory will generate an event, any changes _inside_ a sub-directory of a watched directory will not. We are going to compensate for this by recursively going through all sub-directories (via the Files.walkFileTree method) and register each one with the WatchService object (previously defined in the example here):

    
    
    
    private void registerDirectories() throws IOException {
            Files.walkFileTree(startPath, new WatchServiceRegisteringVisitor());
    }
    
    private class WatchServiceRegisteringVisitor extends SimpleFileVisitor<Path>{
        @Override
        public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
             dir.register(watchService,ENTRY_CREATE,ENTRY_DELETE,ENTRY_MODIFY);
             return FileVisitResult.CONTINUE;
        }
    }
    


On line 2 the Files.walkFileTree method uses the WatchServiceRegisteringVisitor class defined on line 5 to register every directory with the WatchService.  The registered events are creation of files/directories, deletion of files/directories or updates to a file.  


### Publishing Events


The next step is to create a FutureTask that will do the work of checking the queue and publishing the events.

    
    
     private void createWatchTask() {
        watchTask = new FutureTask<>(new Callable<Integer>() {
           private int totalEventCount;
           @Override
           public Integer call() throws Exception {
               while (keepWatching) {
                   WatchKey watchKey = watchService.poll(10, TimeUnit.SECONDS);
                   if (watchKey != null) {
                      List<WatchEvent<?>> events = watchKey.pollEvents();
                      Path watched = (Path) watchKey.watchable();
                      PathEvents pathEvents = new PathEvents(watchKey.isValid(), watched);
                      for (WatchEvent event : events) {
                            pathEvents.add(new PathEvent((Path) event.context(), event.kind()));
                            totalEventCount++;
                      }
                      watchKey.reset();
                      eventBus.post(pathEvents);
                    }
              }
               return totalEventCount;
            }
          });
        }
    
    private void startWatching() {
      new Thread(watchTask).start();
    }
    


On line 7, we are checking the WatchService every 10 seconds for queued events. When a valid WatchKey is returned, the first step is to retrieve the events (line 9) then get the directory where the events occurred (line 10). On line 11 a PathEvents object is created, taking a boolean and the watched directory as constructor arguments. Lines 12-15 are looping over the events retrieved on line 9, using the target Path and event type as arguments to create PathEvent object.  The WatchKey.reset method is called on line 16, setting the WatchKey state back to ready, making it eligible to receive new events and be placed back into the queue. Finally on line 17 the EventBus publishes the PathEvents object to all subscribers.  It's important to note here that the PathEvents and PathEvent classes are immutable.  The totalEventCount that is returned from the Callable is never exposed in the API, but is used for testing purposes.  The startWatching method on line 25 starts the thread to run the watching/publishing task defined above.


### Conclusion


By pairing the WatchService with the Guava EventBus we are able to manage the WatchKey and process events in a single thread and notify any number of subscribers asynchronously of the events. It is hoped the reader found this example useful.  As always comments and suggestions are welcomed.



#### Resources






  * [Source code](https://github.com/bbejeck/Java-7/tree/master/src/main/java/bbejeck/nio/files/directory/event) and [unit test](https://github.com/bbejeck/Java-7/blob/master/src/test/java/bbejeck/nio/files/directory/event/DirectoryEventWatcherImplTest.java) for this post


  * [EventBus API](http://guava-libraries.googlecode.com/svn/trunk/javadoc/com/google/common/eventbus/package-summary.html)


  * [WatchService API](http://docs.oracle.com/javase/7/docs/api/java/nio/file/WatchService.html)


  * Previous post on the [WatchService](http://codingjunkie.net/java-7-watchservice/).


  * Previous post on the [EventBus


