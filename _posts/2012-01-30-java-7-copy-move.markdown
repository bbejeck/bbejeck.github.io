---
author: Bill Bejeck
comments: true
date: 2012-01-30 05:10:34+00:00
layout: post
slug: java-7-copy-move
title: 'What''s New In Java 7:  Copy and Move Files and Directories'
wordpress_id: 1673
categories:
- General
- java
tags:
- java
---

<img class="left" src="{{ site.media_url }}/images/Java_Logo-150x150.png" /> This post is a continuation of my series on the Java 7 java.nio.file package, this time covering the copying and moving of files and complete directory trees.  If you have ever been frustrated by Java's lack of copy and move methods, then read on, for relief is at hand.  Included in the coverage is the very useful Files.walkFileTree method.  Before we dive into the main content however, some background information is needed. 

<!--more-->
### Path Objects


Path objects represent a sequence of directories that may or may not include a file. There are three ways to construct a Path object:




  1. FileSystems.getDefault().getPath(String first, String... more)


  2. Paths.get(String path, String... more), convenience method that calls FileSystems.getDefault().getPath


  3. Calling the toPath method on a java.io.File object


From this point forward in all our examples, we will use the Paths.get method.  Here are some examples of creating Path objects:

    
    
    //Path string would be "/foo"
    Paths.get("/foo");
    //Path string "/foo/bar"
    Paths.get("/foo","bar");
    


To manipulate Path objects there are the Path.resolve and Path.relativize methods.  Here is an example of using Path.resolve:

    
    
    //This is our base path "/foo"
    Path base = Paths.get("/foo");
    //filePath is "/foo/bar/file.txt" while base still "/foo"
    Path filePath = base.resolve("bar/file.txt");
    


Using the Path.resolve method will append the given String or Path object to the end of the calling Path, unless the given String or Path represents an absolute path, the the given path is returned, for example:

    
    
    Path path = Paths.get("/foo");
    //resolved Path string is "/usr/local"
    Path resolved = path.resolve("/usr/local");
    


The Path.relativize works in the opposite fashion, returning a new relative path that if resolved against the calling Path would result in the same Path string.  Here's an example:

    
    
            // base Path string "/usr"
            Path base = Paths.get("/usr");
            // foo Path string "/usr/foo"
            Path foo = base.resolve("foo");
            // bar Path string "/usr/foo/bar"
            Path bar = foo.resolve("bar");
            // relative Path string "foo/bar"
            Path relative = base.relativize(bar);
    


Another method on the Path class that is helpful is the Path.getFileName, that returns the name of the farthest element represented by this Path object, with the name being an actual file or just a directory.  For example:

    
    
    //assume filePath constructed elsewhere as "/home/user/info.txt"
    //returns Path with path string "info.txt"
    filePath.getFileName()
    
    //now assume dirPath constructed elsewhere as "/home/user/Downloads"
    //returns Path with path string "Downloads"
    dirPath.getFileName()
    


In the next section we are going to take a look at how we can use Path.resolve and Path.relativize in conjunction with Files class for copying and moving files.


### Files Class


The Files class consists of static methods that use Path objects to work with files and directories.  While there are over 50 methods in the Files class, at this point we are only going to discuss the copy and move methods. 


#### Copy A File


To copy one file to another you would use the (any guesses on the name?) Files.copy method - copy(Path source, Path target, CopyOption... options) very concise and no anonymous inner classes, are we sure it's Java?.  The options argument are enums that specify how the file should be copied.  (There are actually 2 different Enum classes, LinkOption and StandardCopyOption, but both implement the CopyOption interface) Here is the list of available options for Files.copy:




  1. LinkOption.NOFOLLOW_LINKS


  2. StandardCopyOption.COPY_ATTRIBUTES


  3. StandardCopyOption.REPLACE_EXISTING


There is also a StandardCopyOption.ATOMIC_MOVE enum, but if this option is specified, an UsupportedOperationException is thrown. If no options are specified, the default is to throw an error if the target file exists or is a symbolic link.  If the path object is a directory then an empty directory is created in the target location. (Wait a minute! didn't it say in the introduction that we could copy the entire contents of a directory? The answer is still yes and that is coming!) Here's an example of copying a file to another with Path objects using the Path.resolve and Path.relativize methods:

    
    
            Path sourcePath ...
            Path basePath ...
            Path targetPath ...
    
            Files.copy(sourcePath, targetPath.resolve(basePath.relativize(sourcePath));
    




#### Move A File


Moving a file is equally as straight forward - move(Path source, Path target, CopyOption... options);
The available StandardCopyOptions enums available are:




  1. StandardCopyOption.REPLACE_EXISTING


  2. StandardCopyOption.ATOMIC_MOVE


If Files.move is called with StandardCopyOption.COPY_ATTRIBUTES an UnsupportedOperationException is thrown.  Files.move can be called on an empty directory or if it does not require moving a directories contents, re-naming for example, the call will succeed, otherwise it will throw an IOException (we'll see in the following section how to move non-empty directories).  The default is to throw an Exception if the target file already exists. If the source is a symbolic link, then the link itself is moved, not the target of the link.  Here's an example of Files.move, again tying in the Path.relativize and Path.resolve methods:

    
    
            Path sourcePath ...
            Path basePath ...
            Path targetPath ...
    
            Files.move(sourcePath, targetPath.resolve(basePath.relativize(sourcePath));
    




### Copying and Moving Directories


One of the more interesting and useful methods found in the Files class is Files.walkFileTree.  The walkFileTree method performs a depth first traversal of a file tree. There are two signatures:




  1. walkFileTree(Path start,Set options,int maxDepth,FileVisitor visitor)


  2. walkFileTree(Path start,FileVisitor visitor)


The second option for Files.walkFileTree calls the first option with EnumSet.noneOf(FileVisitOption.class) and Integer.MAX_VALUE.  As of this writing, there is only one file visit option - FOLLOW_LINKS. The FileVisitor is an interface that has four methods defined:


  1. preVisitDirectory(T dir, BasicFileAttributes attrs) called for a directory before all entires are traversed.


  2. visitFile(T file, BasicFileAttributes attrs) called for a file in the directory.


  3. postVisitDirectory(T dir, IOException exc) only called after all files and sub-directories have been traversed.


  4. visitFileFailed(T file, IOException exc) called for files that could not be visited


All of the methods return one of the four possible FileVisitResult enums :


  1. FileVistitResult.CONTINUE


  2. FileVistitResult.SKIP_SIBLINGS (continue without traversing siblings of the directory or file)


  3. FileVistitResult.SKIP_SUBTREE (continue without traversing contents of the directory)


  4. FileVistitResult.TERMINATE


To make life easier there is a default implementation of the FileVisitor, SimpleFileVisitor (validates arguments are not null and returns FileVisitResult.CONTINUE), that can be subclassed co you can override just the methods you need to work with. Let's take a look at a basic example for copying an entire directory structure. 


### Copying A Directory Tree Example


Let's take a look at a class that extends SimpleFileVisitor used for copying a directory tree (some details left out for clarity):

    
    
     public class CopyDirVisitor extends SimpleFileVisitor<Path> {
        private Path fromPath;
        private Path toPath;
        private StandardCopyOption copyOption = StandardCopyOption.REPLACE_EXISTING;
        ....
        @Override
        public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
            Path targetPath = toPath.resolve(fromPath.relativize(dir));
            if(!Files.exists(targetPath)){
                Files.createDirectory(targetPath);
            }
            return FileVisitResult.CONTINUE;
        }
    
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            Files.copy(file, toPath.resolve(fromPath.relativize(file)), copyOption);
            return FileVisitResult.CONTINUE;
        }
    }
    


On line 9, each directory will be created in the target, 'toPath', as each directory from the source, 'fromPath',is traversed.  Here we can see the power the Path object with respect to working with directories and files.  As the code moves deeper into the directory structure, the correct Path objects are constructed simply from calling relativize and resolve on the fromPath and toPath objects, respectively.  At no point do we need to be aware of where we are in the directory tree, and as a result no cumbersome StringBuilder manipulations are needed to create the correct paths.  On line 17, we see the Files.copy method used to copy the file from the source directory to the target directory.  Next is a simple example of deleting an entire directory tree.


### Deleting A Directory Tree Example


In this example SimpleFileVisitor has been subclassed for deleting a directory structure:

    
    
    public class DeleteDirVisitor  extends SimpleFileVisitor<Path> {
    
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            Files.delete(file);
            return FileVisitResult.CONTINUE;
        }
    
        @Override
        public FileVisitResult postVisitDirectory(Path dir, IOException exc) throws IOException {
            if(exc == null){
                Files.delete(dir);
                return FileVisitResult.CONTINUE;
            }
            throw exc;
        }
    }
    


As you can see, deleting is a very simple operation.  Simply delete each file as you find them then delete the directory on exit.


### Combining Files.walkFileTree with Google Guava


The previous two examples, although useful, were very 'vanilla'.  Let's take a look at two more examples that are a little more creative by combining the Google Gauva Function and Predicate interfaces. 

    
    
    public class FunctionVisitor extends SimpleFileVisitor<Path> {
        Function<Path,FileVisitResult> pathFunction;
    
        public FunctionVisitor(Function<Path, FileVisitResult> pathFunction) {
            this.pathFunction = pathFunction;
        }
    
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            return pathFunction.apply(file);
        }
    }
    


In this very simple example, we subclass SimpleFileVisitor to take a Function object as a constructor parameter and as the directory structure is traversed, apply the function to each file.

    
    
    public class CopyPredicateVisitor extends SimpleFileVisitor<Path> {
        private Path fromPath;
        private Path toPath;
        private Predicate<Path> copyPredicate;
    
        public CopyPredicateVisitor(Path fromPath, Path toPath, Predicate<Path> copyPredicate) {
            this.fromPath = fromPath;
            this.toPath = toPath;
            this.copyPredicate = copyPredicate;
        }
    
        @Override
        public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
            if (copyPredicate.apply(dir)) {
                Path targetPath = toPath.resolve(fromPath.relativize(dir));
                if (!Files.exists(targetPath)) {
                    Files.createDirectory(targetPath);
                }
                return FileVisitResult.CONTINUE;
            }
            return FileVisitResult.SKIP_SUBTREE;
        }
    
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            Files.copy(file, toPath.resolve(fromPath.relativize(file)));
            return FileVisitResult.CONTINUE;
        }
    }
    


In this example the CopyPredicateVisitor takes a Predicate object and based on the returned boolean, parts of the directory structure are not copied.  I would like to point out that the previous two examples, usefulness aside, do work in the unit tests foe the source code provided with this post.


### DirUtils


Building on everything we've covered so far, I could not resist the opportunity to create a utility class, [DirUtils](https://github.com/bbejeck/Java-7/blob/master/src/main/java/bbejeck/nio/util/DirUtils.java), as an abstraction for working with directories that provides the following methods:

    
    
     //deletes all files but leaves the directory tree in place
     DirUtils.clean(Path sourcePath);
     //completely removes a directory tree
     DirUtils.delete(Path sourcePath);
     //replicates a directory tree
     DirUtils.copy(Path sourcePath, Path targetPath);
     //not a true move but performs a copy then a delete of a directory tree
     DirUtils.move(Path sourcePath, Path targetPath);
     //apply the function to all files visited
     DirUtils.apply(Path sourcePath,Path targetPath, Function<Path> function);
    


While I wouldn't go as far to say it's production ready, it was fun to write.


### Conclusion


That wraps up the new copy and move functionality provided by the java.nio.file package.  I personally think it's very useful and will take much of the pain out of working with files in Java.  There's much more to cover, working with symbolic links, stream copy methods, DirectoryStreams etc, so be sure to stick around.  Thanks for your time.  As always comments and suggestions are welcomed.


#### Resouces






  * [Java 7 java.nio.file api](http://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html)


  * [Source code and tests](https://github.com/bbejeck/Java-7) for the post


