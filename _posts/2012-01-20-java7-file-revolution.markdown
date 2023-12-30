---
author: Bill Bejeck
comments: true
date: 2012-01-20 06:32:43+00:00
layout: post
slug: java7-file-revolution
title: What's new in Java 7 - The (Quiet) NIO File Revolution
wordpress_id: 1700
categories:
- java
tags:
- java
---

<img class="left" src="{{ site.media_url }}/images/Java_Logo1-150x150.png" /> Java 7 ("Project Coin") has been out since July of last year.  The additions with this release are useful, for example Try with resources - having closable resources handled automatically from try blocks, Strings in switch statements, multicatch for Exceptions and the '<>' operator for working with generics.  The addition that everyone was anticipating the most, closures, has been deferred to version 8.  Surprisingly though, there was a small 'revolution' of sorts with the release of Java 7, that for the most part, in my opinion, went unnoticed and could possibly be the best part of the Java 7 release.  The change I'm referring to is the addition of the java.nio.file package. The java.nio.file package added classes and interfaces that make working with files and directories in Java much easier. First and formost of these changes is the ability to copy or move files.  I always found it frustrating that if you want to copy or move a file, you have to roll your own version of 'copy' or 'move'. The utilities found in the Guava project [com.google.common.io](http://docs.guava-libraries.googlecode.com/git-history/v11.0.1/javadoc/com/google/common/io/package-summary.html) package provides these capabilities, but I feel that copy and move operations should be a core part of the language.  Over the next few posts, I'll be going into greater detail (with code examples) on the classes/interfaces discussed here and some others that have not been covered.  This post serves as an introduction and overview of the new functionality in the java.nio.file package. 
<!--more-->

### Breaking Out Responsibilities


If you take a look at the java.io package as it stands now, the vast majority of the classes are for input streams, output streams, readers or writers . There is only one class that defines operations for directly working with the file system, the File class. Some of the other classes in java.io will take a File object as an argument in a constructor, but all file and directory interaction is through the File class.  In the java.nio.file package, functionality has been teased out into other classes/interfaces.  The first ones to discuss are the Path interface and the Files class.


### Path and Files


A Path object is some what analogous to a java.io.File object as it can represent a file or directory on the file system.  A Path object is more abstract though, in that it is a sequence of names that represent a directory hierarchy (that may or may not include a file) on the file system.  There are not methods in the Path interface that allow for working with directories or files. The methods defined are for working with or manipulating Path objects only, resolving one Path to another etc. (There is one method that can be used to obtain a java.io.File object from a Path, toFile.  Likewise the java.io.File class now contains a toPath method.) To work with files and directories, Path objects are used in conjunction with the Files class.  The Files class consists entirely of static methods for manipulating directories and files, including copy, move and functions for working with symbolic links. Another interesting method in the Files class is the newDirectoryStream method that returns a DirectoryStream object that can iterate over all the entries in a directory.  Although the java.io.File class has the list method where you provide a FileFilter instance, the newDirectoryStream can take a String 'glob' like '*.txt' to filter on. 


### FileStore


As previously mentioned, all interaction with the file system in the java.io package is through the File class.  This includes getting information on used or available space in the file system.  In java.nio.file there is a FileStore class that represents the storage type for the files whether it's a device, partition or concreate file system.  The FileStore class defines methods for getting information about the file storage such as getTotalSpace, getUsableSpace, getUnallocated space.  A FileStore can be obtained by calling the Files.getFileStore(Path path) method that will return the FileStore for that particular file.


### FileSystem and FileSystems


A FileSystem, as the name implies, provides access to the file system and is a factory for other objects in the file system.  For example, the FileSystem class defines a getPath method that converts a string (/foo/bar) into a system dependent Path object that can be used for accessing a file or directory.  The FileSystem class also provides a getFileStores method that returns an iterable of all FileStores in the FileSystem.  The FileSystems class provides access to the FileSystem object with the static FileSystems.getDefault method.  There are also static methods for creating custom FileSystem objects.


### Conclusion


This has been a fast, high level view of the new functionality for working with files provided by the java.nio.file package.  There is much more information that has not been covered here, so take a look at the [api docs](http://docs.oracle.com/javase/7/docs/api/java/nio/file/package-summary.html) Hopefully this post has been able get the reader interested in the improved file handling in Java 7.  Be sure to stick around as we begin to explore in more detail what the java.nio.file package has to offer. 


