---
layout: post
title: "Guava ImmutableCollections, Multimaps and Java 8 Collectors"
date: 2015-10-23 10:54:47 -0400
permalink: /guava-and-java8-collectors/
comments: true
categories: 
- Java 8
- Guava
- java
tags: 
- Guava
- Java 8
keywords: 
- Java 8
- Guava
- Multimap
- ImmutableCollection
- Collector
description: How to create custom Java 8 collectors from Guava ImmutableCollections and MulitMaps. 
---
In this post we are going to discuss creating custom [Collector](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html) instances.  The `Collector` interface was introduced in the [java.util.stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) package when Java 8 was released.  A `Collector` is used to "collect" the results of stream operations.  Results are collected from a stream when the terminal operation `Stream.collect` method is called.  While there are default  implementations available, there are times we'll want to use some sort of custom container.  Our goal today will be to create `Collector` instances that produce Guava [ImmutableCollections](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/ImmutableCollection.html) and [Multimaps](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Multimap.html). 
<!--more-->
###Background Collector Information

The `Collector` interface is described very well in the [documentation](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html), so here we'll just give a brief overview.  The `Collector` interface defines 3 type 
parameters and 4 methods:

```java Collector Interface
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A,T> accumulator();
    BinaryOperator<T> combiner();
    Function<A,R> finisher();
    Set<Collector.Characteristics>  characteristics();
}
```
1.   The supplier function returns a new instance of a mutable accumulator of type A.
2. The accumulator function takes the accumator and an instance of type T that will result in T being added to the accumulator.
3. The combiner function takes two instances of the accumulator and merges them into one.  The merging could be a new instance or the result of folding one of the accumulators into the other.
4. The finisher function performs a final operation on the accumulator (possibly merged accumulators) into the final type R.  The final type could also be the same type as the accumulator.
5. The characteristics function returns a set of [Collector.Characteristics](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.Characteristics.html) that contain the properties for the collector.  The characteristic values are`CONCURRENT`, `UNOREDRED` and `IDENTITY_FINISH`.  

Addtionally there 2 static methods `Collector.of`.  The `Collector.of` methods return a new collector based on the provided supplier,accumulator, combinbiner and (optionall) finisher definitions.  Since we're here to create custom `Collector` instances, we won't be covering this functionality. For the first example we'll use a Guava [ImmutableList](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/ImmutableList.html) as a collector.

###Guava ImmutableList Collector Example
Our first step is to create an abstract class that defines the accumulator, combiner and finisher operations.  We're using an abstract class because Guava offers several different immutable collections, so we'll need to provide a different supplier for each one.

```java Abstract Base Class for Guava Immutable Collection Collectors

private static abstract class ImmutableCollector<T, A extends ImmutableCollection.Builder, R extends ImmutableCollection<T>> implements Collector {

    @Override
    public BiConsumer<A, T> accumulator() {
        return (c, v) -> c.add(v);
    }

    @Override
    public BinaryOperator<A> combiner() {
        return (c1, c2) -> (A) c1.addAll(c2.build().iterator());
    }

    @Override
    public Function<A, R> finisher() {
        return (bl -> (R) bl.build());
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Sets.newHashSet(Characteristics.CONCURRENT);
    }
    
}
```
Now that we have our collector functionality defined, we'll provide different implementations of our base class to provide suppliers for different types of immutable collectors:

```java Implementations Providing Different Suppliers
private static class ImmutableSetCollector<T> extends ImmutableCollector {
  @Override
  public Supplier<ImmutableSet.Builder<T>> supplier() {
        return ImmutableSet::builder;
  }

 @Override
 public Set<Characteristics> characteristics() {
    return Sets.newHashSet(Characteristics.CONCURRENT, Characteristics.UNORDERED);
    }
 }

 private static class ImmutableSortedSetCollector<T extends Comparable<?>> extends ImmutableCollector {
    @Override
    public Supplier<ImmutableSortedSet.Builder<T>> supplier() {
        return ImmutableSortedSet::naturalOrder;
    }
  }

 private static class ImmutableListCollector<T> extends ImmutableCollector {
    @Override
    public Supplier<ImmutableList.Builder<T>> supplier() {
            return ImmutableList::builder;
    }
 }   
```
You will notice here that in the `ImmutableSetCollector` implementation we have overridden the characteristics of our `ImmutableSet` collector to mark it as concurrent (accumulator function can be called from multiple threads) and unordered (elements won't be maintained in insertion order). Finally we wrap all this up a the [ImmutableCollectors](https://github.com/bbejeck/Java-8/blob/master/src/main/java/bbejeck/collector/ImmutableCollectors.java) class and provdide static methods to easily create our collectors:
```java ImmutableCollectors Class
public class ImmutableCollectors {

   public static <E> ImmutableListCollector<E> ofList() {
       return new ImmutableListCollector<>();
   }

   public static <E> ImmutableSetCollector<E> ofSet() {
        return new ImmutableSetCollector<>();
   }

   public static <E extends Comparable<?>> ImmutableSortedSetCollector<E> ofSortedSet(){
        return new ImmutableSortedSetCollector<>();
   }
   //Other functionality previously shown left out here for clarity
}
```
Here's an example of using the `ImmutableListCollector` in a unit test:
```java Junit Example
@Test
public void testCollectImmutableList(){
 List<String> things = Lists.newArrayList("Apple", "Ajax", "Anna", "banana", "cat", "foo", "dog", "cat");

  List<String> aWords = things.stream().filter(w -> w.startsWith("A")).collect(ImmutableCollectors.ofList());
  assertThat(aWords.contains("Apple"),is(true));
  assertThat(aWords instanceof ImmutableList,is(true));

  assertThat(aWords.size(),is(3));
  boolean unableToModifyList = false;
  try{
        aWords.add("Bad Move");
  }catch (UnsupportedOperationException e){
            unableToModifyList = true;
  }
    assertTrue("Should not be able to modify list",unableToModifyList);
  }
```
###Guava Multimaps as Collectors
After working with Guava collections I have come to find the [Multimap](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/Multimap.html) a very handy abstraction to partition data when you have more than one value for a given key.  So it would be nice a collector to partition the results of our `Stream` operations into Guava `Multimaps`.  We are going to follow the same pattern we used with the `ImmutableCollectors` class.  There is an abstract base class providing the accumulator, combining and finishing functions.  Then different implementations of that abstract class to provide a supplier for the different flavors of Guava `Mulitmaps`.  With that in mind let's look at some code examples:
```java Base Functionality Defined
private static abstract class MultimapCollector<K,T, A extends Multimap<K,T>, R extends Multimap<K,T>> implements Collector  {

  private Function<T, K> keyFunction;

  public MultimapCollector(Function<T, K> keyFunction) {
        Preconditions.checkNotNull(keyFunction,"The keyFunction can't be null");  
        this.keyFunction = keyFunction;
    }

    @Override
    public BiConsumer<A, T> accumulator() {
      return (map, value) -> map.put(keyFunction.apply(value),value);
    }

    @Override
    public BinaryOperator<A> combiner() {
        return (c1, c2) -> {
            c1.putAll(c2);
            return c1;
        };
    }

   @Override
   @SuppressWarnings("unchecked")
    public Function<A, R> finisher() {
        return mmap -> (R) mmap;
    }      
 }
```
There is one small difference here though.  We need to provide a function to determine the key to use when partitioning the data.  The aptly named `keyFunction` provided at object instantiation time fills this need. Here are the implementations of the base class providing different suppliers:

```java Implementations with Different Suppliers
 private static class ListMulitmapCollector<K,T> extends MultimapCollector {
  private ListMulitmapCollector(Function<T, K> keyFunction) {
        super(keyFunction);
   }
   @Override
    public Supplier<ArrayListMultimap<K,T>> supplier() {
        return ArrayListMultimap::create;
    }
  }

 private static class HashSetMulitmapCollector<K,T> extends MultimapCollector {
    private HashSetMulitmapCollector(Function<T, K> keyFunction) {
        super(keyFunction);
    }

    @Override
    public Supplier<HashMultimap<K,T>> supplier() {
        return HashMultimap::create;
    }
 }
 private static class LinkedListMulitmapCollector<K,T> extends MultimapCollector {
    private LinkedListMulitmapCollector(Function<T, K> keyFunction) {
            super(keyFunction);
    }
    @Override
    public Supplier<LinkedHashMultimap<K,T>> supplier() {
        return LinkedHashMultimap::create;
    }
  }
```   
Again we'll up this code in a container class (`MultimapCollectors`) and provide static methods for creating the different collectors:

```java Multimap Collectors Class
    public class MultiMapCollectors {

  public static <K,T> ListMulitmapCollector<K,T> listMultimap(Function<T, K> keyFunction) {
         return new ListMulitmapCollector<>(keyFunction);
  }

     public static <K,T> HashSetMulitmapCollector<K,T> setMulitmap(Function<T, K> keyFunction) {
         return new HashSetMulitmapCollector<>(keyFunction);
     }
    
     public static <K,T> LinkedListMulitmapCollector<K,T> linkedListMulitmap(Function<T,K> keyFunction) {
         return new LinkedListMulitmapCollector<>(keyFunction);
     }
``` 
Finally, here's an exmample of a `Multimap` collector (HashSetMultimap) in action:

```java JUnit example of HashSetMultimap
    public class MultiMapCollectorsTest {

    private List<TestObject> testObjectList;

    @Before
    public void setUp() {
        TestObject w1 = new TestObject("one", "stuff");
        TestObject w2 = new TestObject("one", "foo");
        TestObject w3 = new TestObject("two", "bar");
        TestObject w4 = new TestObject("two", "bar");
        testObjectList = Arrays.asList(w1, w2, w3, w4);
    }

     @Test
    public void testSetMultimap() throws Exception {
        HashMultimap<String, TestObject> testMap = testObjectList.stream().collect(MultiMapCollectors.setMulitmap((TestObject w) -> w.id));

        assertThat(testMap.size(),is(3));
        assertThat(testMap.get("one").size(),is(2));
        assertThat(testMap.get("two").size(),is(1));
    }
   //other details left out for clarity
```
###Conclusion
That wraps up our quick introduction to creating custom `Collector` instances in Java 8.  Hopefully we've demonstrated the usefulness the `Collector` interface and he we could go about creating custom collectors.

###References
*   [ImmutableCollectors](https://github.com/bbejeck/Java-8/blob/master/src/main/java/bbejeck/collector/ImmutableCollectors.java)
*   [MultmapCollectors](https://github.com/bbejeck/Java-8/blob/master/src/main/java/bbejeck/collector/MultiMapCollectors.java)
*   [Guava Collections](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/package-summary.html)
*   [Streams](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html)
*   [Collector Interface](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html)