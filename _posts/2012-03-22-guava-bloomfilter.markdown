---
author: admin
comments: true
date: 2012-03-22 05:01:04+00:00
layout: post
slug: guava-bloomfilter
title: Google Guava BloomFIlter
wordpress_id: 2144
categories:
- General
- Guava
tags:
- Guava
- java
---

<img class="left" src="../assets/images/google-guava.gif" /> When the Guava project released version 11.0, one of the new additions was the BloomFilter class.  A BloomFilter is a unique data-structure used to indicate if an element is contained in a set.  What makes a BloomFilter interesting is it will indicate if an element is _absolutely not_ contained, or _may be_ contained in a set. This property of never having a false negative makes the BloomFilter a great candidate for use as a guard condition to help prevent performing unnecessary and expensive operations.  While BloomFilters have received good exposure lately, using one meant rolling your own, or doing a Google search for code. The trouble with rolling your own BloomFilter is getting the correct hash function to make the filter effective.  Considering Guava uses the Murmur Hash for its' implementation, we now have the usefulness of an effective BloomFilter just a library away.
<!--more-->

#### BloomFilter Crash Course


BloomFilters are essentially bit vectors.  At a high level BloomFilters work in the following manner: 




  1. Add the element to the filter.


  2. Hash it a few times, than set the bits to 1 where the index matches the results of the hash.

 
When testing if an element is in the set, you follow the same hashing procedure and check if the bits are set to 1 or 0. This process is how a BloomFilter can guarantee an element does not exist. If the bits aren't set, it’s simply impossible for the element to be in the set.  However, a positive answer means the element is in the set or a hashing collision occurred.  A more detaild description of a BloomFilter can be found [here](http://spyced.blogspot.com/2009/01/all-you-ever-wanted-to-know-about.html) and a good tutorial on BloomFilters [here](http://llimllib.github.com/bloomfilter-tutorial/). According to [wikipedia](http://en.wikipedia.org/wiki/Bloom_filter), Google uses BloomFilters in BigTable to avoid disk lookups for non-existent items.  Another interesting usage is [using a BloomFilter to optimize a sql querry](http://asemanfar.com/Using-a-bloom-filter-to-optimize-a-SQL-query).



#### Using the Guava BloomFilter


A Guava BloomFilter is created by calling the static method create on the BloomFilter class, 
passing in a [Funnel](http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/hash/Funnel.html) object and an int representing the expected number of insertions. A Funnel, also new in Guava 11, is an object that can send data into a [Sink](http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/hash/Sink.html).  The following example is the default implementation and has a percentage of false positives of 3%. Guava provides a [Funnels](http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/hash/Funnels.html) class containing two static methods providing implementations of the Funnel interface for inserting a CharSequence or byte Array into a filter.

    
    
    //Creating the BloomFilter
    BloomFilter bloomFilter = BloomFilter.create(Funnels.byteArrayFunnel(), 1000);
    
    //Putting elements into the filter
    //A BigInteger representing a key of some sort
    bloomFilter.put(bigInteger.toByteArray());
    
    //Testing for element in set
    boolean mayBeContained = bloomFilter.mayContain(bitIntegerII.toByteArray());
    
    


UPDATE: based on the comment from Louis Wasserman, here's how to create a BloomFilter for BigIntegers with a custom Funnel implementation:

    
    
    //Create the custom filter
    class BigIntegerFunnel implements Funnel<BigInteger> {
            @Override
            public void funnel(BigInteger from, Sink into) {
                into.putBytes(from.toByteArray());
            }
        }
    
    //Creating the BloomFilter
    BloomFilter bloomFilter = BloomFilter.create(new BigIntegerFunnel(), 1000);
    
    //Putting elements into the filter
    //A BigInteger representing a key of some sort
    bloomFilter.put(bigInteger);
    
    //Testing for element in set
    boolean mayBeContained = bloomFilter.mayContain(bitIntegerII);
    




#### Considerations


It's critical to estimate the number of expected insertions correctly. As insertions into the filter approach or exceeds the expected number, the BloomFilter begins to fill up and as a result will generate more false positives to the point of being useless.  There is another version of the BloomFilter.create method taking an additional parameter, a double representing the desired level of false hit probability (must be greater than 0 and less than one). The level of false hit probability affects the number of hashes for storing or searching for elements.  The lower the desired percentage, the higher number of hashes performed.


#### Conclusion


A BloomFilter is a useful item for a developer to have in his/her toolbox.  The Guava project now makes it very simple to begin using a BloomFilter when the need arises.  I hope you enjoyed this post.  Helpful comments and suggestions are welcomed.



#### References






  * [Unit Test Demo of Guava BloomFilter](https://github.com/bbejeck/guava-blog/blob/master/src/test/java/bbejeck/guava/hash/BloomFilterTest.java).


  * [BloomFilter class](http://docs.guava-libraries.googlecode.com/git-history/v11.0.2/javadoc/com/google/common/hash/BloomFilter.html)


  * [All You Want to Know about BloomFilters](http://spyced.blogspot.com/2009/01/all-you-ever-wanted-to-know-about.html).


  * [BloomFilter Tutorial](http://llimllib.github.com/bloomfilter-tutorial/).


  * [BloomFilter on Wikipedia](http://en.wikipedia.org/wiki/Bloom_filter).


