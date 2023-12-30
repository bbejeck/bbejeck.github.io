---
layout: post
title: "Working With Java 8 Optionals"
date: 2015-10-30 13:21:50 -0400
permalink: /working-with-java8-optionals/
categories: 
- java
- Java 8
tags: 
- java
- Java 8
- Functions
- Lambdas
- Functional Interface
- Optional
keywords: 
description: How to work with Java 8 Optional methods for maximum data safety.
---
In this post we are going to cover working with the [Optional](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html) class introduced in Java 8.  The introduction of `Optional` was new only to Java.  Guava has had a version of [Optional](https://github.com/google/guava/wiki/UsingAndAvoidingNullExplained) and Scala has had the [Option](http://www.scala-lang.org/api/current/index.html#scala.Option) type for some time.  Here's a description of Optional from the Java API docs:
> A container object which may or may not contain a non-null value.

Since so many others have done a good job of [describing](http://blog.joda.org/2014/11/optional-in-java-se-8.html) the `Optional` type, we won't be doing so here.  Rather for this post, we are going to cover how to use the `Optional` type without resorting to directly accessing the value contained or doing explicit checks if a value is present. 
<!--more-->
###Working with Optionals
Using the `Optional` class helps, to a large degree,  prevent NullPointerExceptions.  This is accomplished by allowing developers to more clearly define where missing values are to be expected.  But at some point we'll need to work with those potential values.  The `Optional` class provides two methods that fit the bill right away: the`Optional.isPresent()` and `Optional.get()` methods.  But our goal today is to work with `Optional` types and avoid (or at least delay until absolutely neccessary) methods that directly query or access the potential value contained within.

####Conditional Operations
Java developers are used to writing something like the following:
```java Adding Element to as List or Collection
List<String> values = new ArrayList<>;
String someString = getValue();
if(someString != null){
    values.add(someString)
}
```
Now instead we can use the `Optional.ifPresent(Consumer<? super T> consumer)` method. If the value is present the provided [Consumer](https://docs.oracle.com/javase/8/docs/api/java/util/function/Consumer.html) will be invoked with the value contained as a parameter.  Here's an example presented in a unit test:
```java Adding Optional Value to a List
    @Test
    public void optional_is_present_add_to_list_without_get_test() {
        List<String> words = Lists.newArrayList();

        Optional<String> month = Optional.of("October");
        Optional<String> nothing = Optional.ofNullable(null);

        month.ifPresent(words::add);
        nothing.ifPresent(words::add);

        assertThat(words.size(), is(1));
        assertThat(words.get(0), is("October"));
    }
```
From above, if the value is present it will be added to the list via a `List.add()` call, otherwise the operation is skipped.

####Conditionally Changing Optional Values
Another common scenario is to check if a value is not null before we transform it.  Conisider this very simple example:
```java Substring
String longString = getValue();
String smallerWord = null;
if(longString != null){
    smallerWord = longString.subString(0,4)
}
```
To transform the value of an `Optional` we can use the [map](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html#map-java.util.function.Function-) method.  Here's an example, again in the context of a unit test:
```java Optional.map Substring Example
    @Test
    public void optional_map_substring_test() {
        Optional<String> number = Optional.of("longword");
        Optional<String> noNumber = Optional.empty();

        Optional<String> smallerWord = number.map(s -> s.substring(0,4));
        Optional<String> nothing = noNumber.map(s -> s.substring(0,4));

        assertThat(smallerWord.get(), is("long"));
        assertThat(nothing.isPresent(), is(false));
    }
```
While this is a trivial example, we perform the mapping operation without explicitly checking if a value is present or retrieving the value to apply the given function.  We simply provide the function and allow the API to handle the details.  There also is an [Optional.flatMap](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html#flatMap-java.util.function.Function-) function.  We use the `Optional.flatMap` method when we have an existing `Optional` and want to apply a function that also returns an a type of `Optional`.  For example:

```java Optional FlatMap example
    @Test
    public void optional_flat_map_test() {
        Function<String, Optional<String>> upperCaseOptionalString = s -> (s == null) ? Optional.empty() : Optional.of(s.toUpperCase());

        Optional<String> word = Optional.of("apple");

        Optional<Optional<String>> optionalOfOptional = word.map(upperCaseOptionalString);

        Optional<String> upperCasedOptional = word.flatMap(upperCaseOptionalString);

        assertThat(optionalOfOptional.get().get(), is("APPLE"));

        assertThat(upperCasedOptional.get(), is("APPLE"));

    }
```
By using the `flatMap` method we can avoid the awkward return type of `Option<Option<String>>` by "flattening" out the results into a single `Optional` container.

####Filtering Optionals
There is also the ability to filter `Optional` objects.  If the value is present and matches the given predicate, the `Optional` is returned with it's value intact, otherwise an empty `Optional` is returned.  Here's an exmaple of the `Optional.filter` method:

```java Optional Filter Example
    @Test
    public void optional_filter_test() {
        Optional<Integer> numberOptional = Optional.of(10);
        Optional<Integer> filteredOut = numberOptional.filter(n -> n > 100);
        Optional<Integer> notFiltered = numberOptional.filter(n -> n < 100);

        assertThat(filteredOut.isPresent(), is(false));
        assertThat(notFiltered.isPresent(), is(true));
    }
```
####Specifying Default Values
At some point we'll want to retieve the value contained within an `Optional`, but if it's not found, provide a default value instead.  But instead of resorting to the "if not present then get" pattern, we can specify default values.  There are two methods that allow for setting default values  `Oprional.orElse` and `Optional.orElseGet`. With `Optional.orElse` we direclty supply the default value and with `Optional.orElseGet` we provide a [Supplier](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html) that is used to provide the default value.

```java Optional.orElse Optional.orElseGet Example
    @Test
    public void optional_or_else_and_or_else_get_test() {
        String defaultValue = "DEFAULT";

        Supplier<TestObject> testObjectSupplier = () -> {
            //Mimics set up needed to create object
            //Pretend these values need to be retrieved from some datastore
            String name = "name";
            String category = "justCreated";
            return new TestObject(name, category, new Date());

        };

        Optional<String> emptyOptional = Optional.empty();
        Optional<TestObject> emptyTestObject = Optional.empty();

        assertThat(emptyOptional.orElse(defaultValue), is(defaultValue));

        TestObject testObject = emptyTestObject.orElseGet(testObjectSupplier);

        assertNotNull(testObject);
        assertThat(testObject.category, is("justCreated"));
    }
```
####When a Null/Absent Value is not Expected
For those cases where a missing value represents an error condition there is the `Optional.orElseThrow` method:

```java Optional.orElseThrow Example
    @Test(expected = IllegalStateException.class)
    public void optional_or_else_throw_test() {
        Optional<String> shouldNotBeEmpty = Optional.empty();

        shouldNotBeEmpty.orElseThrow(() -> new IllegalStateException("This should not happen!!!"));
    }
```

###Working with Collections of Optionals
Finally, let's cover how we could use these methods in conjunction with [Collections](https://docs.oracle.com/javase/8/docs/api/java/util/Collections.html) of `Optional` instances.  It would be nice if we could specify a [Collector](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html) that simply returned the values found or values plus defaults.  Just for fun let's define one:
```java Collector for Optionals
 public static <T> Collector<Optional<T>, List<T>, List<T>> optionalToList() {
    return optionalValuesList((l, v) -> v.ifPresent(l::add));
 }


public static <T> Collector<Optional<T>, List<T>, List<T>> optionalToList(T defaultValue) {
   return optionalValuesList((l, v) -> l.add(v.orElse(defaultValue)));
}

private static <T> Collector<Optional<T>, List<T>, List<T>> optionalValuesList(BiConsumer<List<T>, Optional<T>> accumulator) {
  Supplier<List<T>> supplier = ArrayList::new;
  BinaryOperator<List<T>> combiner = (l1, l2) -> {
            l1.addAll(l2);
            return l1;
        };
   Function<List<T>, List<T>> finisher = l1 -> l1;
return Collector.of(supplier, accumulator, combiner, finisher);

}    
```
Here we've defined a `Collector` that returns a `List` containing the values found in a `Collection` of `Optional` types.  There is also an overloaded version where we can specify a default value.  Let's wrap this up with a unit test showing the new `Collector` in action:

```java Collector of Optional Types
//some details left out for clarity

private List<String> listWithNulls = Arrays.asList("foo", null, "bar", "baz", null);
private List<Optional<String>> optionals;

@Before
public void setUp() {
  optionals = listWithNulls.stream().map(Optional::ofNullable).collect(Collectors.toList());
}

@Test
public void collect_optional_values_test(){
   List<String> upperCasedWords = optionals.stream().map(o -> o.map(String::toUpperCase)).collect(CustomCollectors.optionalToList());
   List<String> expectedWords = Arrays.asList("FOO","BAR","BAZ");

   assertThat(upperCasedWords,is(expectedWords));
}

@Test
public void collect_optional_with_default_values_test(){
  String defaultValue = "MISSING";

  List<String> upperCasedWords = optionals.stream().map(o -> o.map(String::toUpperCase)).collect(CustomCollectors.optionalToList(defaultValue));
  List<String> expectedWords = Arrays.asList("FOO", defaultValue, "BAR", "BAZ", defaultValue);

   assertThat(upperCasedWords,is(expectedWords));
}
```    
In the first test, a `List` containing only the present values is returned, and in the second test there are defualt values as specified by the given parameter.

###Conclusion 
We've covered how to use the methods found in the `Optional` class.  By making use of methods demonstrated here, working with missing data is bound to be easier and less error prone.

###Resources
*   [Unit Test for Optional Methods](https://github.com/bbejeck/Java-8/blob/master/src/test/java/bbejeck/optional/OptionalTest.java)
*   [CustomCollectors](https://github.com/bbejeck/Java-8/blob/master/src/main/java/bbejeck/collector/CustomCollectors.java) for Optional types.
*   [Using And Avoiding Null](https://github.com/google/guava/wiki/UsingAndAvoidingNullExplained)
*   [Optional](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html) API documentation.
*   [Why Even Use Java 8 Optional](https://www.voxxed.com/blog/2015/05/why-even-use-java-8-optional/)
*   [Java SE 8 Optional, a pragmatic approach](http://blog.joda.org/2015/08/java-se-8-optional-pragmatic-approach.html)


