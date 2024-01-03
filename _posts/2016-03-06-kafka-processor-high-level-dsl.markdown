---
layout: post
title: "Kafka Streams - The KStreams API"
date: 2016-03-26 11:26:08 -0500
permalink: /kafka-streams-part2/
comments: true
categories:
- Kafka
- Kafka-Streams
tags:
- Kafka
- Kafka-Streams 
keywords:
- Kafka
- Kafka-Streams
- Kafka Processor 
description: An introduction to kafka-streams an abstraction for key-value events. 
---
<img class="left" src="../assets/images/kafka_logo.png" />The last [post](http://codingjunkie.net/kafka-processor-part1/) covered the new [Kafka Streams](http://docs.confluent.io/2.1.0-alpha1/streams/index.html) library, specifically the "low-level" Processor API.  This time we are going to cover the "high-level" API, the [Kafka Streams DSL](http://docs.confluent.io/2.1.0-alpha1/streams/developer-guide.html#streams-developer-guide-dsl).  While the Processor API gives you greater control over the details of building streaming applications, the trade off is more verbose code.  In most cases, however, the level of detail provided by the Processor API is not required and the KStream API will get the job done.  Compared to the declarative approach of the Processor API , KStreams uses a more functional style. You'll find building an application is more a matter of stating "what" you want to accomplish versus "how".  Additionally, since many interfaces in the Kafka Streams API are Java 8 syntax compatible (method handles and lambda expressions can be substituted for concrete types), using the KStream DSL allows for building powerful applications quickly with minimal code.  This post won't be as detailed as the previous one, as the description of Kafka Streams applies to both APIs.  Here we'll focus on how to implement the same functionality presented in the previous post with the KStream API. 
<!-- more -->
### KStreams DSL In Brief
Before jumping into examples, a brief overview of the KStreams DSL is in order.  The KStreams DSL is composed of two main abstractions; the [KStream](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/kstream/KStream.java) and [KTable](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/kstream/KTable.java) interfaces.  A KStream is a record stream where each key-value pair is an independent record.  Later records in the stream don't replace earlier records with matching keys.  A KTable on the other hand is a "changelog" stream, meaning later records are considered updates to earlier records with the same key.  Some of the methods provided by the KStream DSL have a functional feel:

*     map
*    mapValue
*    flatMap
*    flatMapValues
*    filter

In addition to the methods listed above, the KStream DSL also offers methods for joining streams and performing aggregations:  

*     join, leftJoin and outerJoin
*    aggragateByKey and reduceByKey

Due to the indefinate nature of stream data all of the joins, aggregation and reduction operations in the KStream DSL are done with "windowing", meaning that data is put into specific slots based on time.  This by no means is a complete list of what's available from the KStream DSL, but it serves as a good starting point.
 
### KStreams First Example
For the first KStream example we are going to re-use the first one from the Processor API post.  In that example we wanted to take a simulated stream of customer purchase data and develop 3 `Processor` instances to do the following operations:

1.    Mask credit card numbers used by customers in the purchase.
2.    Collect the customer name and the amount spent to use in a rewards program.
3.    Collect the zip code and the item purchased to help determine shopping patterns. 

Our requirements have not changed, but instead of providing 3 separate Processor instances, we can use `KStream.mapValues` and use lambda expressions instead:

```java
KStream<String,Purchase> purchaseKStream = kStreamBuilder.stream(stringDeserializer,purchaseJsonDeserializer,"src-topic")
                .mapValues(p -> Purchase.builder(p).maskCreditCard().build());

purchaseKStream.mapValues(p -> PurchasePattern.builder(p).build()).to("patterns",stringSerializer,purchasePatternJsonSerializer);

purchaseKStream.mapValues(p -> RewardAccumulator.builder(p).build()).to("rewards",stringSerializer,rewardAccumulatorJsonSerializer);

purchaseKStream.to("purchases",stringSerializer,purchaseJsonSerializer);
```       
It's impressive that we have constructed our entire event stream operation with 4 lines of code. Contrast this with the lower-level processor approach where we had 3 separate classes for the processors and an addtional 7 lines just for the [TopologyBuilder](http://docs.confluent.io/2.1.0-alpha1/streams/javadocs/org/apache/kafka/streams/processor/TopologyBuilder.html) to connect all of the processors and sinks together.

### KStreams Stateful Operations Example
For the second KStream example, we have a stream of simulated stock purchases.We want to publish the individual trades to one topic and periodic summary analysis of all trades grouped by ticker symbol to another topic.

```java
KStream<String,StockTransaction> transactionKStream =  kStreamBuilder.stream(stringSerde,transactionSerde,"stocks");
 transactionKStream.through(stringSerde, transactionSerde,"stocks-out")
                .map((k,v)-> new KeyValue<>(v.getSymbol(),v))
                .aggregateByKey(StockTransactionCollector::new,
                               (k, v, stockTransactionCollector) -> stockTransactionCollector.add(v),
                               TumblingWindows.of("stock-summaries").with(10000),
                               stringSerde,collectorSerde)
                .to(windowedSerde,collectorSerde,"transaction-summary");
```

Again we can see that the KStream API gives us the ability to create an event based application with just a few lines of code.  While this example is straight forward, let's take a quick review of what's going on:

1.    First we create a KStream from the topic "stocks".  The `stream` method takes 3 parameters a `Serde` for the key, a `Serde` for the value and topic. `Serde` is a container for a serializer and deserializer for a given type.
2. Next the `KStream.through` method is called, which writes the tranaction data to the "stocks-source" topic and returns a KStream instance which also reads from the same topic.  
3. Next we map the transaction (which has no key) to a new `KeyValue` object by extracting the ticker symbol of the trade to use as the key.
4. Now we come to the heart of the stream, `aggregateByKey`.  There are 5 parameters provided:
     5. An [Intitializer](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/kstream/Initializer.java) instance.  In this case we use a java 8 method handle to provide a new instance of a [StockTransactionCollector](https://github.com/bbejeck/kafka-streams/blob/master/src/main/java/bbejeck/model/StockTransactionCollector.java)
     6. Next is an [Aggregator](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/kstream/Aggregator.java) instance.  An `Aggegator` expects 3 types: a key, value and the type of the aggregator.  Again we leverage lambda expressions to provide the needed type.
     7. Then a [TumblingWindow](https://github.com/apache/kafka/blob/trunk/streams/src/main/java/org/apache/kafka/streams/kstream/internals/TumblingWindow.java) is specified.  A `TumblingWindow` will collect information for a specified period of time.  Each window is discrete as there no overlapping records from other time periods. All of the aggregations collected in the "windowed" period are published to the topic. 
     8. Finally the key and value `Serde` instances for the  pre-aggregation data are provided.
9.   The last method in the chain is  `to`, which specifies the topic where the aggreagtion results (of each time window) are publised.  Again we provide 2 serde instances, this time for the keys and values that are a result of the aggreagtion.

### Conclusion

This wraps up a quick introduction to the KStream API.  Although the coverage is brief, hopefully it's enough to demonstrate how  KStreams enable rapid development of powerful key-value event based applications.  In follow on posts we'll go over some of the other methods that involve joining and merging streams as well as how we can apply 3rd party libraries such as [Ling-Pipe](http://alias-i.com/lingpipe/) to do interesting anlysis on our event streams.

### Resouces
*    [Stream Processing Made Simple](http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple) a blog post from Confluent describing Kafak Streams
*    Kafka Streams [Documentation](http://docs.confluent.io/2.1.0-alpha1/streams/index.html)
*    Kafka Streams [Developer Guide](http://docs.confluent.io/2.1.0-alpha1/streams/developer-guide.html)
*    [Confluent](http://www.confluent.io/) 
*    [Source code and instructions to run examples](https://github.com/bbejeck/kafka-streams) for this post.  Note that all of the streaming examples use simulated streams and can run indefinitely.
*    The original post on [Kafka Streams](http://codingjunkie.net/kafka-processor-part1/) covering the Processor API.
