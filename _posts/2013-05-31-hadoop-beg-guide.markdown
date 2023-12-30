---
author: Bill Bejeck
comments: true
date: 2013-05-31 04:46:37+00:00
layout: post
slug: hadoop-beg-guide
title: 'Book Review : Hadoop - Beginners Guide'
wordpress_id: 2689
categories:
- Book Review
- General
- Hadoop
---

<img class="left" src="{{ site.media_url }}/images/hadoopBegGuide.png" /> This post I am going to review the book "Hadoop - Beginners Guide" by Gary Turkington. In the interest of full disclosure, while I received a free copy of the book for reviewing, I do not receive any compensation for books purchased from this blog. With that out of the way let's get on with reviewing the book. Hadoop Beginners Guide covers a lot of ground, and despite the title, it's not entirely for people new to Hadoop. There is good coverage of more advanced topics, that even someone who has experience with Hadoop might be able to learn a thing or two. I'm going to break down my coverage into 3 areas - 1) How is the MapReduce framework and underlying technology explained 2) How is writing your own map reduce programs covered and 3) Setting up a cluster locally for development and setting up a real cluster.
<!--more-->

### Map Reduce/Hadoop Details


Hadoop - Beginners Guide does a good job of explaining the underlying functionality of Hadoop. Starting at the level of explaining how Hadoop uses key/value pairs all the way to clearly explaining the functionality of Mappers and Reducers. The book also goes on to explain the important role that Combiners play in speeding up a Hadoop job by reducing data flowing to the Reducers during the shuffle phase.  Readers new to Hadoop should grasp a good understanding of what is going on at each phase of a Hadoop job which goes a long way to writing effective MapReduce code. The topic of joining data in Hadoop is given good coverage, including map-side and reduce-side joins. The The author goes into detail about how Hadoop is about not only expecting failure, but to embrace failure as part of the process. Another key point made is Hadoop is about putting the "smarts" in the software, freeing users from vendor lockin with expensive hardware, instead relying on using commodity servers. 


### Writing MapReduce Programs


An entire chapter(4) is devoted to writing MapReduce programs. There is a twist however, in that Hadoop Streaming is demonstrated and the language used is Ruby.  While developing a real MapReduce program that evaluates a large unique dataset (UFO sightings), several other areas are covered including using multiple mappers in a single job, using the distributed cache for efficiently sharing reference data across all mappers/reducers.  There are several detailed code examples so the reader is never at a loss as to what is going on. The chapter wraps up by showing how to use counters to capture additional information about the MapReduce job.


### Getting Hadoop Setup


Chapter 2 does an excellent job of getting users started using Hadoop.  First the reader is walked through setting up Hadoop in pseudo-distributed mode with explicit details covering downloading and installing Hadoop, setting up SSH keys, formatting the NameNode all culminating with running the requisite WordCount program to test the installation process was successful.  Chapter 2 goes on to discuss in good detail setting up Hadoop on AWS(Amazon Web Services) using the Elastic MapReduce (EMR) service.  In chapter 7, details for setting up a real cluster are discussed in detail.


### Conclusion


Overall I found the Hadoop - Beginners Guide to be a thorough introduction to using Hadoop that goes beyond being just a guide for beginners, but can be used as a reference as well. There are two other books from Packt on Hadoop, published around the same time 




  1. [MapReduce cookbook](http://www.packtpub.com/hadoop-mapreduce-cookbook/book) Which contains strategies for solving tough/complex big data problems


  2. [Hadoop Real-World solutions](http://www.packtpub.com/hadoop-real-world-solutions-cookbook/book) A collection of solutions to common problems when working with Hadoop and detailed code examples for performing analysis of data in Hadoop.

 

