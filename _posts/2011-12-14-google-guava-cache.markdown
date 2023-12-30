---
author: Bill Bejeck
comments: true
date: 2011-12-14 05:44:37+00:00
layout: post
slug: google-guava-cache
title: Google Guava Cache
wordpress_id: 1438
categories:
- Guava
- java
tags:
- cache
- Guava
- java
---

<img class="left" src="{{ site.media_url }}/images/toolbox.jpg" /> This Post is a continuation of my series on Google Guava, this time covering Guava Cache. Guava Cache offers more flexibility and power than either a HashMap or ConcurrentHashMap, but is not as heavy as using EHCache or Memcached (or robust for that matter, as Guava Cache operates solely in memory).  The Cache interface has methods you would expect to see like 'get', and 'invalidate'.  A method you won't find is 'put', because Guava Cache is 'self-populating', values that aren't present when requested are fetched or calculated, then stored. This means a 'get' call will never return null. In all fairness, the previous statement is not %100 accurate.  There is another method 'asMap' that exposes the entries in the cache as a thread safe map. Using 'asMap' will result in not having any of the self loading operations performed, so calls to 'get' will return null if the value is not present (What fun is that?).  Although this is a post about Guava Cache, I am going to spend the bulk of the time talking about CacheLoader and CacheBuilder.  CacheLoader specifies how to load values, and CacheBuilder is used to set the desired features and actually build the cache. 
<!--more-->

### CacheLoader


[CacheLoader](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/cache/CacheLoader.html) is an abstract class that specifies how to calculate or load values, if not present.  There are two ways to create an instance of a CacheLoader:




  1. Extend the CacheLoader<K,V> class


  2. Use the static factory method CacheLoader.from

 
If you extend CacheLoader you need to override the ` V load(K key)` method, instructing how to generate the value for a given key.  Using the static `CacheLoader.from` method you build a CacheLoader either by supplying a Function or Supplier interface.  When supplying a Function object, the Function is applied to the key to calculate or retrieve the results.  Using a Supplier interface the value is obtained independent of the key. 


### CacheBuilder


The [CacheBuilder](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/cache/CacheBuilder.html) is used to construct cache instances.  It uses the fluent style of building and gives you the option of setting the following properties on the cache:




  * Cache Size limit (removals use a LRU algorithm)


  * Wrapping keys in WeakReferences (Strong references used by default for keys)


  * Wrapping values in either WeakReferences or SoftReferences (Strong references used by default)


  * Time to expire entires after last access


  * Time based expiration of entries after being written or updated


  * Setting a RemovalListener that can recieve events once an entry is removed from the cache


  * Concurrency Level of the cache (defaults to 4)


The concurrency level option is used to partition the table internally such that updates can occur without contention. The ideal setting would be the maximum number of threads that could potentially access the cache at one time.  Here is an example of a possible usage scenario for Guava Cache.

    
    
    public class PersonSearchServiceImpl implements SearchService<List<Person>> {
    
    public PersonSearchServiceImpl(SampleLuceneSearcher luceneSearcher, SampleDBService dbService) {
            this.luceneSearcher = luceneSearcher;
            this.dbService = dbService;
            buildCache();
        }
    
        @Override
        public List<Person> search(String query) throws Exception {
            return cache.get(query);
        }
    
        private void buildCache() {
            cache = CacheBuilder.newBuilder().expireAfterWrite(10, TimeUnit.MINUTES)
                    .maximumSize(1000)
                    .build(new CacheLoader<String, List<Person>>() {
                        @Override
                        public List<Person> load(String queryKey) throws Exception {
                            List<String> ids = luceneSearcher.search(queryKey);
                            return dbService.getPersonsById(ids);
                        }
                    });
        }
    }
    


In this example, I am setting the cache entries to expire after 10 minutes of being written or updated in the cache, with a maximum amount of 1,000 entires. Note the usage of CacheLoader on line 15.


### RemovalListener


The [RemovalListener](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/cache/RemovalListener.html) will receive notification of an item being removed from the cache.  These notifications could be from manual invalidations or from a automatic one due to time expiration or garbage collection.  The RemovalListener<K,V> parameters can be set to listen for specific type.  To receive notifications for any key or value set them to use Object.  It should be noted here that a RemovalListener will receive a RemovalNotification<K,V> object that implements the Map.Entry interface.  The key or value could be null if either has already been garbage collected.  Also the key and value object will be strong references, regardless of the type of references used by the cache.


### CacheStats


There is also a very useful class [CacheStats](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/cache/CacheStats.html) that can be retrieved via a call to Cache.stats().  The CacheStats object can give 
insight into the effectiveness and performance of your cache by providing statistics such as:




  * hit count


  * miss count


  * total load timme


  * total requests


CacheStats provides many other counts in addition to the ones listed above.


### Conclusion


The Guava Cache presents some very compelling functionality.  The decision to use a Guava Cache really comes down to the tradeoff between memory availability/usage versus increases in performance.  I have added a unit test [CacheTest](https://github.com/bbejeck/guava-blog/blob/master/src/test/java/bbejeck/guava/cache/CacheTest.java) demonstrating the usages discussed here. As alway comments and suggestions are welcomed. Thanks for your time.



#### Resources






  * [Guava Project Home](http://code.google.com/p/guava-libraries/)


  * [Cache API](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/cache/package-summary.html)


  * [Source Code](https://github.com/bbejeck/guava-blog) for blog series


