---
layout: post
title: "Kafka Clients performance part 0 "
author: Bill Bejeck
date: 2025-11-09 15:00:00 -0400
permalink: /kafka-clients-perf-part-0
comments: false
categories: 
- Java
- Kafka
- Kafka-Streams
tags:
- Kafka-Streams
- Kafka
- Java
keywords: 
- Kafka-Streams
- KafkaConsumer
- KafkaProducer
description: Strategies and guidelines for improving performance of Kafka clients including KafkaStreams
---

I presented recently (as of this writing) at Current in New Orleans.  The subject of my talk was serializing and its relationship to Kafka consumers, producers, and Kafka Streams.  As one might expect, I eventually discussed the performance of the various serialization formats.  As I continued to work on the presentation, it occurred to me that, while serialization is an essential aspect of working with records in a Kafka application, it's just a piece of a larger picture.

I think that, in retrospect, given the chance to do the presentation again, I would have expanded the scope beyond serialization and discussed the various components that can affect client performance in a Kafka-based application.  But that's the beauty of writing, you can always go back and expand on previous work.  To that end, I will be conducting a blog series on Kafka client performance.  I realize this is a well-worn topic (no pun intended), but it's something that I've wanted to cover for some time.