---
layout: post
slug: java-8-dates-part2
title: "Whats new in Java 8 - Date API part II"
date: 2014-03-25 22:07:49 -0400
permalink: /java-8-dates-part2
comments: true
categories:
- java
- Java 8
tags:
- java
- Java 8
- Date
---

<img class="left" src="../assets/images/javaEight.jpeg" /> This post is continues our review of the [Date](http://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) API that came with the release of Java 8.  We are going to continue our concentration on classes that make working with dates/times very easy.  Working with date objects in previous releases of Java was very challenging with respect to adding time or getting the difference between dates.  Hopefully after looking at the classes we present here, your opinion of working with dates and times in Java will change.  Specifically, we are going to take a look at the following classes:


  +  Other classes to represent dates/times `ZonedDateTime`   and `OffsetDateTime`
  + Getting the current snapshot in time with `Instant`
  + Using the `Clock` class to get system time but specify different time zones
  + Represent arbitrary number of days with the `Period` class
  + Represent arbitrary amount of hours with the `Duration` class
  

<!--more-->
### Zoned/Offset Dates and Times


In the [last post](http://codingjunkie.net/java-8-dates-part1) we covered the `LocalDateTime`, `LocalDate` and the `LocalTime` classes.   As the name suggests, these classes give the date and or time for a given locality with no time-zone or offset from UTC/Greenwhich time.  Java 8 provides the [ZonedDateTime](http://docs.oracle.com/javase/8/docs/api/java/time/ZonedDateTime.html) class which provides date-times with a time-zone representation.  Creating a `ZonedDateTime` instance can be done several ways, but here we will demonstrate using two of the many static factory methods: 
```java
 //Uses the system clock using the default time-zone.
 ZonedDateTime zdt = ZonedDateTime.now();
 //Displays as 2014-03-28T21:52:09.122-04:00[America/New_York]

 //Uses the system clock with the specified time-zone
 ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Europe/Dublin"));
 //Displays as 2014-03-29T01:45:34.605Z[Europe/Dublin]
``` 

Next we have the `OffsetDateTime` and the `OffsetTime` classes that represent date-time or time (respectively) with offsets from UTC/Greenwhich time.  Here we show creating `OffsetDateTime` and `OffsetTime` using some of the other available static factory methods: 
```java
OffsetDateTime odt = OffsetDateTime.of(LocaDateTime.now(),ZoneOffset.of("-4"));
//Displays as 2014-03-28T22:30:28.911-04:00

OffsetTime ot = OffsetTime.ofInstant(Intant.now(),ZoneId.of("America/Los_Angeles"));
//Displays as 19:45:40.661-07:00
```
In the last example we see the use of the `Instant` class which we will introduce soon.  It's worth noting at this point that the `ZondedDateTime`,`OffsetDateTime` classes store time to nanosecond precision. Typically we would use the `ZonedDateTime` class when showing date-times to users and the `OffsetDateTime` class when interacting with systems.


### Clock
The [Clock](http://docs.oracle.com/javase/8/docs/api/java/time/Clock.html) class gives us the ability to get the current date/time from the system clock with a specific time-zone. Although there are date-time classes that have a `now()` method returning the current date-time, the system clock uses the default time-zone. The `Clock` class allows us to get the system time with a given time-zone. We can then plug the `Clock` instance into other classes where we want to get the current time using a given time-zone.
```java
//Returns Clock with default time-zone
Clock default = Clock.systemDefaultZone();
//Clock with desired time-zone
Clock clock = Clock.system(ZoneId.of("America/Chicago"));
```
We can now re-work our previous example of creating an `OffsetTime` object using a `Clock` instance to set the desired time-zone:
```java
Clock clock = Clock.system(ZoneId.of("America/Los_Angeles"));
......
public void someOperation(Clock clock){
     OffsetTime ot = OffsetTime.of(clock);
     ... some work involving the OffsetTime intance
}
```

The `Clock` class also allows us to specify how it 'ticks', meaning we can have the time returned from the `Clock` instance ticking on whole minutes or seconds
```java
Clock wholeMinuteClock = Clock.tickMinutes(ZoneId.of("Europe/Athens"));
Clock wholeSecondClock = Clock.tickSeconds(ZonieId.of("Europe/Prauge"));
```

### Instant
The [Instant](http://docs.oracle.com/javase/8/docs/api/java/time/Instant.html) class is used to capture the current 'instant' in time.  The `Instant` class is useful for obtaining event timestamps and also has nanosecond precision. 
```java
//Record the current moment on the time-line from the system clock
Instant now = Instant.now();
//Record the current instant with the given Clock instance
Instance now = Instant.now(clock)
```
The `Instant` class has several other methods for adjusting an `Instant` instance such as adding/subtracting time or converting to a `OffsetDateTime` or `ZonedDateTime`.

### Period
The [Period](http://docs.oracle.com/javase/8/docs/api/java/time/Period.html) class represents an arbitrary amount of time in years, months, or days. `Period` objects can be particularly useful for adding/subtracting time to/from a date. For example:
```java
    @Test
    public void test_add_days(){
        LocalDateTime today = LocalDateTime.parse("2014-03-12T19:36:33");
        Period sixDays = Period.ofDays(6);
        LocalDateTime nextWeek = today.plus(sixDays);
        assertThat(nextWeek.toString(),is("2014-03-18T19:36:33"));
    }

    @Test
    public void test_subtract_days(){
        LocalDate today = LocalDate.parse("2014-03-12");
        Period twoWeeks = Period.ofWeeks(2);
        LocalDate past = today.minus(twoWeeks);
        assertThat(past.toString(),is("2014-02-26"));
    }
```

The `Period` class also offers a static method `Period.between` that is great for determining elapsed time between dates:
```java
    @Test
    public void test_period_between_dates(){
        LocalDate twins = LocalDate.parse("2003-11-18");
        LocalDate mayhem = LocalDate.parse("2009-06-01");
        Period timeBetween = Period.between(twins,mayhem);
        assertThat(timeBetween.getYears(),is(5));
        assertThat(timeBetween.getMonths(),is(6));
        assertThat(timeBetween.getDays(),is(14));
    }    
```
While the `Period.between` method requires types of `LocalDate`,  the date-time classes in the `java.time` package offer a `toLocalDate()` method that returns a `LocalDate` from the date-time instance.


### Duration
The [Duration](http://docs.oracle.com/javase/8/docs/api/java/time/Duration.html) class represents arbitrary amounts of time in hours, minutes or seconds.  The useage pattern for `Duration` is similar to that of the `Period` class. Here are examples of adjusting time objects using the `Duration` class:
```java
    @Test
    public void test_add_time(){
        Duration oneHourThirtyMinutes = Duration.ofHours(1).plusMinutes(30);
        OffsetTime now = OffsetTime.parse("10:15:30-05:00");
        OffsetTime later = now.plus(oneHourThirtyMinutes);
        assertThat(later.toString(),is("11:45:30-05:00"));

        Duration threeHours = Duration.parse("PT3H");
        LocalTime twoPM = LocalTime.parse("14:00:00");
        LocalTime fivePM = twoPM.plus(threeHours);
        assertThat(fivePM.toString(),is("17:00"));
    }

    @Test
    public void test_subtract_time(){
        OffsetTime offsetTime = OffsetTime.parse("13:34:00+01:00");
        LocalTime earlier = LocalTime.parse("09:30:25");
        LocalTime later = LocalTime.parse("15:33:47");
        Duration timeSpan = Duration.between(earlier,later);
        OffsetTime adjustedOffsetTime = offsetTime.minus(timeSpan);
        assertThat(adjustedOffsetTime.toString(),is("07:30:38+01:00"));
    }
```  
The `Duration` class also has a `between` method for determing the amount of time (hours based) between time objects:
```java
    @Test
    public void test_time_between(){
        LocalTime earlier = LocalTime.parse("09:30:25");
        LocalTime later = LocalTime.parse("15:33:47");
        Duration timeSpan = Duration.between(earlier,later);
        assertThat(timeSpan.toString(),is("PT6H3M22S"));
        assertThat(LocalTime.MIDNIGHT.plus(timeSpan).toString(),is("06:03:22"));
    }
```
The `Duration.between` method requires types of `Temporal`. All of the time and date-time classes in the `java.time` package implement the `Temporal` interface.


### Conclusion
This wraps up our brief tour of the new Date API in Java 8.  Hopefully you can see the promise in the new Date API for working with dates and times.  In future posts I plan to continue coverage of the new features found in Java 8.  Thanks for your time.

### Resources

  * [Joda Time](http://www.joda.org/joda-time/) a Java date-time library to use for Java versions < 8


  * The [java.time](http://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) package contains the java doc for the classes discussed in this post.


  * [Functional Programming in Java 8](http://pragprog.com/book/vsjava8/functional-programming-in-java) a great resource on using the new functional components in Java 8


  * [Source code](https://github.com/bbejeck/Java-8/tree/master/test/bbejeck/dates) for this post
