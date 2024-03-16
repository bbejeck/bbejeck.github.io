---
layout: post
title: "Mastering Stream Processing - Testing Kafka Streams windowed applications"
author: Bill Bejeck
date: 2024-03-16 16:55:00 -0400
permalink: /mastering-stream-processing-testing-kafka-streams
comments: false
categories: 
- Kafka
- Java
- Kafka-Streams
- Stream-Processing
tags: 
- Java
- Kafka-Streams
- Stream-Processing
keywords: 
- Kafka-Streams
- Flink-SQL
- Stream-Processing
description: How to test Kafka Streams windowed applications
---

In this blog series about windowing aggregations we’ve covered a lot of territory. Here are the previous posts: 

1.  [Introduction to windowing](https://www.codingjunkie.net/introduction-to-windowing)
2.  [Hopping and Tumbling windows](https://www.codingjunkie.net/mastering-stream-processing-hopping-tumbling-windows)
3.  [Sliding windows and OVER aggregation](https://www.codingjunkie.net/mastering-stream-processing-sliding-windows)
4.  [Session windows Cumulating windows](https://www.codingjunkie.net/mastering-stream-processing-session-cumulating-windows)
5.  [Window time semantics](https://www.codingjunkie.net/mastering-stream-processing-time-semantics)
6.  [Viewing and analyzing results]((https://www.codingjunkie.net//mastering-stream-processing-viewing-results)

In this final installment, we will cover testing a windowed application. Testing is essential to validate your logic. Typically, when testing an application, you’ll assert that N input records result in an expected result. The [time semantics](https://windowing_time_semantics)post emphasized that event timestamps drive the window action. So, it’s not enough for testing to feed the application records and assert results; you must provide timestamps to advance the window correctly. In the next two posts, we will cover how to test both Kafka Streams and Flink SQL to ensure your streaming windowed applications produce correct results. I originally planned to cover testing on one post, but it grew too long, so I split it in half. This post focuses on testing in Kafka Streams.

# Testing Kafka Streams windowed aggregations

Kafka Streams provides the [TopologyTestDriver](https://kafka.apache.org/37/javadoc/org/apache/kafka/streams/TopologyTestDriver.html)(TTD) for [testing a topology](https://docs.confluent.io/platform/current/streams/developer-guide/test-streams.html#testing-a-streams-application) without the need of a live broker. As a result, TTD tests execute very fast and allow you to thoroughly test all topologies, from the simple to the complex. Generally speaking a TTD test will involve using [TestInputTopic](https://kafka.apache.org/37/javadoc/org/apache/kafka/streams/TestOutputTopic.html) to push some records through your topology and then capture the results with a [TestOutputTopic](https://kafka.apache.org/37/javadoc/org/apache/kafka/streams/TestOutputTopic.html) and will look something like the following code listing (I’ve left out several details for clarity)

**A TopologyTestDriver test of a Kafka Streams application**
```java
    try(TopologyTestDriver driver = new TopologyTestDriver(topology)) {

       TestInputTopic<String, String> inputTopic = driver.createInputTopic(INPUT_TOPIC...);

       TestOutputTopic<String, String> outputTopic = driver.createOutputTopic(OUTPUT_TOPIC...);

       inputTopic.pipeInput("foo");
       inputTopic.pipeInput("bar");

       List<String> expectedOutput = Arrays.asList("FOO", "BAR");
       List<String> actualOutput = outputTopic.readValuesToList();
       assertEquals(expectedOutput, actualOutput);
    }
```

Reviewing the code in the above listing makes testing a Kafka Streams application straightforward. One thing that’s not obvious from this example is the use of timestamps. Under the covers, the TTD will create timestamps for all the input records. This approach is acceptable for a topology without windowing, since they don’t require timestamps to calculate results.

But once you have a topology that requires advancing timestamps, i.e., a windowed aggregation, you’ll want to take another approach and supply custom time values. You’ll want to use just enough values to validate your application for a unit test. The difference in time between records will be so slight that it will not be effective for driving the window behavior. To solve this issue, the `TestInputTopic` provides `pipeInput` [method overloads](https://kafka.apache.org/37/javadoc/org/apache/kafka/streams/TestInputTopic.html#pipeInput(K,V,java.time.Instant)) accepting a [java.time.Instant](https://kafka.apache.org/37/javadoc/org/apache/kafka/streams/TestInputTopic.html#pipeInput(K,V,java.time.Instant)) allowing you effectively advance a windowed operation with a small number of input records.

For example, assume you have a one-minute tumbling window aggregation (no grace period) of string keys and double values. Let’s take a look at the test code where you set timestamps to advance the window to contain a small number of expected values:

**Providing timestamps to drive a windowed operation**
```java
    try(TopologyTestDriver driver = new TopologyTestDriver(tumblingWindows.topology(properties))) {
        TestInputTopic<String, Double> testInputTopic = driver.createInputTopic(inputTopic,
                                                                               stringSerializer,
                                                                               doubleSerializer);

       LocalDate localDate = LocalDate.ofInstant(Instant.now(), ZoneId.systemDefault());<1>
       LocalDateTime localDateTime = LocalDateTime.of(localDate.getYear(),
                                                      localDate.getMonthValue(),
                                                      localDate.getDayOfMonth(), 12, 0, 18);

       Instant instant = localDateTime.toInstant(ZoneOffset.UTC); 

       testInputTopic.pipeInput("deviceOne", 10.0, instant);
       testInputTopic.pipeInput("deviceOne", 35.0, instant.plusSeconds(20)); <2> 
       testInputTopic.pipeInput("deviceOne", 45.0, instant.plusSeconds(40));
       testInputTopic.pipeInput("deviceOne", 15.0, instant.plusSeconds(70)); <3>

    }
```
Let’s walk through this code:

1.  You’re creating an `Instant` from a `LocalDateTime` object

2.  Next, you advance the time 20 and 40 seconds with the second and third record inputs.

3.  With the fourth record, you advance the time by more than 1 minute, so Kafka Streams will close the first window containing records 1-3 and create a new one containing the fourth record.

I want to discuss the block of code setting the `LocalDateTime` :

**Setting current date time for the test**
```java
    LocalDate localDate = LocalDate.ofInstant(Instant.now(), ZoneId.systemDefault());
    LocalDateTime localDateTime = LocalDateTime.of(localDate.getYear(),
                                                   localDate.getMonthValue(),
                                                   localDate.getDayOfMonth(), 12, 0, 18);
```
In this case, we’ll have a window starting at 12:00:00 (tumbling windows are aligned to the epoch), but the 2nd and 3rd records advance the timestamp value, not the window start time. Getting the starting time for the initial timestamp is essential because you’ll want to ensure you have enough room for subsequent advances that align with your testing assertions.

If you have record payloads that contain timestamps, and assuming you’re using a custom [TimestampExtractor](https://kafka.apache.org/37/javadoc/org/apache/kafka/streams/processor/TimestampExtractor.html), then you’ll follow the same approach placing timestamps on the values you’re piping through the test.

Now, let’s walk through asserting windowed results. The result of the windowed aggregation is an `IoTAggregation` object that tracks the number of readings taken, the highest value seen, and the sum of readings to calculate an average. We’ll use this information to validate our aggregation code:

**Validating the windowed aggregation**
```java
    List<KeyValue<Windowed<String>, IotSensorAggregation>> results = testOutputTopic.readKeyValuesToList();
    IotSensorAggregation firstWindowAggregation = results.get(0).value;
    IotSensorAggregation lastWindowAggregation = results.get(2).value;
    IotSensorAggregation secondWindowAggregation = results.get(3).value;

    assertEquals(10.0, firstWindowAggregation.highestSeen());
    assertEquals(1, firstWindowAggregation.numberReadings());
    assertEquals(10.0, firstWindowAggregation.averageReading());

    assertEquals(45.0, lastWindowAggregation.highestSeen());
    assertEquals(3, lastWindowAggregation.numberReadings());
    assertEquals(30.0, lastWindowAggregation.averageReading());

    assertEquals(15.0, secondWindowAggregation.highestSeen());
    assertEquals(1, secondWindowAggregation.numberReadings());
    assertEquals(15.0, secondWindowAggregation.averageReading());
```

This testing code asserts that our windowed aggregation is operating correctly. The first window should only have 1 reading, and the average should equal the highest seen value since it’s the first record. It then asserts that the last aggregation of the 1-minute window should contain 3 readings and the correct average. Finally, it asserts that the last record input should be in a new window since its timestamp advanced over 1 minute, so it should have a similar state to the first window in that it contains 1 reading.

# Resources

-   [Kafka Streams in Action 2nd Edition](https://www.manning.com/books/kafka-streams-in-action-second-edition) book.

-   [Apache Flink® on Confluent Cloud](https://www.confluent.io/product/flink/)

-   [Flink SQL Windows](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/table/sql/queries/window-tvf/#windowing-table-valued-functions-windowing-tvfs)

-   [Kafka Streams windowing documentation](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#windowing)
