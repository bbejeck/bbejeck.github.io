---
layout: post
title: "Mastering Stream Processing - Testing Flink SQL windowed applications"
author: Bill Bejeck
date: 2024-05-07 12:46:00 -0400
permalink: /mastering-stream-processing-testing-flink-sql
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
description: How to test Flink SQL windowed applications
---

We’ve covered a lot of territory in this blog series about windowing aggregations. Here are the previous posts:

1.  [Introduction to windowing](https://www.codingjunkie.net/introduction-to-windowing)
2.  [Hopping and Tumbling windows](https://www.codingjunkie.net/mastering-stream-processing-hopping-tumbling-windows)
3.  [Sliding windows and OVER aggregation](https://www.codingjunkie.net/mastering-stream-processing-sliding-windows)
4.  [Session windows Cumulating windows](https://www.codingjunkie.net/mastering-stream-processing-session-cumulating-windows)
5.  [Window time semantics](https://www.codingjunkie.net/mastering-stream-processing-time-semantics)
6.  [Viewing and analyzing results](https://www.codingjunkie.net/mastering-stream-processing-viewing-results)
7.  [Testing Kafka Streams windowed applications](https://www.codingjunkie.net/mastering-stream-processing-testing-kafka-streams)

In this final installment, we will cover testing a windowed application. While it may seem obvious, tests are essential to validate your logic. Usually, when testing an application, you’ll assert that N input records result in an expected result. The [time semantics](https://www.codingjunkie.net/mastering-stream-processing-time-semantics)post emphasized that event timestamps drive the window action. So, it’s not enough to feed the application records and assert results; you must provide timestamps to advance the window correctly. In this post, we will cover how to test Apache Flink® SQL to ensure your streaming windowed applications produce correct results.

Now, let’s move on to testing Flink SQL windowed queries.

# Testing Flink SQL windowed aggregations

The Flink SQL client provides an interactive environment for trying different queries but is impossible to use in an automated test. Fortunately, there is the Flink [Table API](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/dev/table/tableapi/#table-api) that allows you to execute Flink SQL statements programmatically. So, we’ll use the Table API to create integration tests you can run in JUnit. The first step to running Flink SQL in a test is to create a `StreamTableEnvironment` that you’ll use to drive the test:

**Setting up Flink SQL test with the Table API**
```java

    StreamTableEnvironment streamTableEnv;

    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);
    env.getConfig().setRestartStrategy(RestartStrategies.noRestart());
    env.setStateBackend(new EmbeddedRocksDBStateBackend());
    streamTableEnv = StreamTableEnvironment.create(env, EnvironmentSettings.newInstance().inStreamingMode().build());
```

Now that you have the `StreamTableEnvironment`, the following steps are to create a table, populate it with data, execute the SQL statement under test, and assert the results. First, let’s make the table for the test:

**Table under test**
```java
    String table = String.format("CREATE TABLE iot_readings (\n" +
    "     device_id STRING,\n" +
    "     reading DOUBLE,\n" +
    "     reading_time TIMESTAMP(3),\n" +
    "     WATERMARK FOR reading_time AS reading_time\n" +
    ") WITH (\n" +
    "    'connector' = 'kafka',\n" +
    "    'topic' = 'iot_readings',\n" +
    "    'properties.bootstrap.servers' = 'localhost:%s',\n" + <1>
    "    'scan.startup.mode' = 'earliest-offset',\n" +
    "    'scan.bounded.mode' = 'latest-offset',\n" +
    "    'key.format' = 'raw',\n" +
    "    'key.fields' = 'device_id',\n" +
    "    'value.format' = 'json',\n" +
    "    'value.fields-include' = 'EXCEPT_KEY'\n" +
    ");", kafkaPort); <2>
```      

    1. A placeholder for String.format to set the Apache Kafka® port

    2. The variable containing the Kafka port determined by the KafkaContainer

Here, the `WITH` clause specifies to use the [Flink Kafka connector](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/table/kafka/#apache-kafka-sql-connector). The test will also use [Testcontainers](https://java.testcontainers.org/modules/kafka/) to provide the running Kafka instance. I won’t go into using Testcontainers in a test, but you can get the details from [this blog post by Atomic Jar](https://www.atomicjar.com/2023/06/testing-kafka-applications-with-testcontainers/).

Now that you have the table definition along with the kafka-sql-connector configuration, the next step is to execute this statement to create the table:

**Running the CREATE TABLE statement**
```java
    streamTableEnv.executeSql(table).await();
```

You’ll use the `await()` method to ensure that the test doesn’t progress until Flink completes creating the table. The next step is to populate the table with data:

**Inserting the testing sensor data**
```java
    String insertStatement = "INSERT INTO iot_readings VALUES\n" +
    "    ('south_sensor', 5.0, TO_TIMESTAMP('2024-04-09 01:01:00')),\n" +
    "    ('south_sensor', 10.0, TO_TIMESTAMP('2024-04-09 01:03:00')),\n" +
    "    ('south_sensor', 9.7, TO_TIMESTAMP('2024-04-09 01:04:00')),\n" +
    "    ('south_sensor', 12.0, TO_TIMESTAMP('2024-04-09 01:06:00')),\n" +
    "    ('south_sensor', 11.9, TO_TIMESTAMP('2024-04-09 01:07:00')),\n" +
    "    ('south_sensor', 8.2, TO_TIMESTAMP('2024-04-09 01:09:00'));";

    streamTableEnv.executeSql(insertStatement).await();
```

Here, we’re simply using the familiar SQL `INSERT` statement to get data into the table, and again, we see the `await` method in use to make sure the test only progresses after the data inserts are finished. With the data inserted into the table, the next step is to execute the windowed query and compare the results against what we expect them to be:

**Executing the query and comparing results**
```java
    String query = "SELECT device_id,\n" +
    "    MAX(reading) AS max_reading,\n" +
    "    window_start,\n" +
    "    window_end\n" +
    "FROM TABLE(TUMBLE(TABLE iot_readings, DESCRIPTOR(reading_time), INTERVAL '5' MINUTES))\n" +
    "GROUP BY device_id, window_start, window_end;";

    TableResult tableResult = streamTableEnv.executeSql(query);  <1>

    List<Row> actualResults = rowObjectsFromTableResult(tableResult); <2>

    List<Row> expectedRowResults = Arrays.asList(Row.ofKind(           <3>
                                                          RowKind.INSERT,
                                                          "south_sensor",
                                                          10.0,
                                                          parseTS("2024-04-09 01:00:00"),
                                                          parseTS("2024-04-09 01:05:00")),
                                                 Row.ofKind(
                                                          RowKind.INSERT,
                                                          "south_sensor",
                                                          12.0,
                                                          parseTS("2024-04-09 01:05:00"),
                                                          parseTS("2024-04-09 01:10:00"))
                                                       );
       assertEquals(expectedRowResults, actualResults); <4>
```

    1. Executing the query

    2. Extracting the results into an ArrayList

    3. Creating the expected results

    4. Asserting the actual results match the expected ones.

So, the final step is straightforward. The test executes the query and compares the returned results to what it expects them to be. Notice that since we’ve inserted simple data, it’s trivial to construct the predicted list of results. For completeness, here’s the code for the `rowObjectsFromTableResult` method:

**Extracting the Row objects**
```java
     public List<Row> rowObjectsFromTableResult(TableResult tableResult) throws Exception {
        try (CloseableIterator<Row> closeableIterator = tableResult.collect()) {
          List<Row> rows = new ArrayList<>();
          closeableIterator.forEachRemaining(rows::add);
          return rows;
        }
      }
```
The method uses the `TableResult.collect()` method to gather the query results from the `ClosableIterator` and put them in a container more suitable for comparison at the end of the test.

There’s one last point I’d like to make before we move on. The last record of the test data has the time of 01:09:00. With this being the last record, the watermark wouldn’t reach 01:10:00 so how can the second window produce results that satisfy the test? It goes back to the configuration for the Flink SQL Kafka connector, specifically the `scan.bounded.mode=latest-offset` configuration. The `scan.bounded.mode` configuration determines when the stream is complete by specifying the latest offsets after consuming from Kafka. In other words, it sets a bound on the stream from Kafka; otherwise, the table is considered unbounded. Using the bounded mode is especially useful in tests, since it flushes out all pending results from any windows, joins, etc that would otherwise hang, waiting for watermarks that aren’t going to come.

This blog post has been a quick tour of testing Flink SQL windowed queries, but it can serve as the basis for writing future tests against other windowed SQL queries.

# Resources

-   [Kafka Streams in Action 2nd Edition](https://www.manning.com/books/kafka-streams-in-action-second-edition) book

-   [Apache Flink® on Confluent Cloud](https://www.confluent.io/product/flink/)

-   [Flink SQL Windows](https://nightlies.apache.org/flink/flink-docs-release-1.18/docs/dev/table/sql/queries/window-tvf/#windowing-table-valued-functions-windowing-tvfs)

-   [Kafka Streams windowing documentation](https://docs.confluent.io/platform/current/streams/developer-guide/dsl-api.html#windowing)
