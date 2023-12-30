---
comments: true
date: 2014-03-04 13:30:58+00:00
layout: post
slug: java-8-dates-part1
title: What's new in Java 8 - Date API
wordpress_id: 3071
categories:
- java
- Java 8
tags:
- java
- Java 8
---

<img class="left" src="{{ site.media_url }}/images/javaEight.jpeg" /> With the final release of Java 8 around the corner, one of the new features I'm excited about is the new [Date](http://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) API, a result of the work on [JSR 310](https://jcp.org/en/jsr/detail?id=310).  While Lambda expressions are certainly the big draw of Java 8, having a better way to work with dates is a decidedly welcome addition.  This is a quick post (part 1 of 2 or 3) showing some highlights of the new Date functionality, this time mostly around the [LocalDate](http://docs.oracle.com/javase/8/docs/api/java/time/LocalDate.html) class. 

<!--more-->
### Creating New Date Objects


Creating a new Date object representing a specific day is as easy as:

    
    
    LocalDate today = LocalDate.parse("2014-02-27");
    //or this method
    LocalDate bday = LocalDate.of(2014,3,18);
    




### Adding To Dates


As an example of the ease with which we can work with dates in Java 8, consider the case where we need to add either days, months or years to an existing date. There are the methods `LocalDate.plusDays`,`LocalDate.plusWeeks`, `LocalDate.plusMonths`  `LocalDate.plusYears`.  There is also a generic `LocalDate.plus` method where you specify how much to add and the time unit via a [TemporalUnit](http://docs.oracle.com/javase/8/docs/api/java/time/temporal/TemporalUnit.html) type.  Here some examples:

    
    
        @Test
        public void test_add_to_date() {
    
            LocalDate oneMonthFromNow = today.plusDays(30);
            assertTrue(oneMonthFromNow.isEqual(LocalDate.parse("2014-03-29")));
    
            LocalDate nextMonth = today.plusMonths(1);
            assertTrue(nextMonth.isEqual(LocalDate.parse("2014-03-27")));
    
            LocalDate future = today.plus(4, ChronoUnit.WEEKS);
            assertTrue(future.isEqual(LocalDate.parse("2014-03-27")));
    
        }
    




### Subtracting From Dates


To subtract days, weeks, months or years from a date there the expected methods: `LocalDate.minusDays`, `LocalDate.minusMonths` etc. Here's some examples of subtracting from a date:

    
    
       @Test
        public void test_subtract_from_date() {
    
            assertThat(today.minusWeeks(1).toString(), is("2014-02-20"));
    
            assertThat(today.minusMonths(2).toString(), is("2013-12-27"));
    
            assertThat(today.minusYears(4).toString(), is("2010-02-27"));
    
            Period twoMonths = Period.ofMonths(2);
    
            assertThat(today.minus(twoMonths).toString(), is("2013-12-27"));
    
        }
    


In this example we also introduced the [Period](http://docs.oracle.com/javase/8/docs/api/java/time/Period.html) object.


### Determining Difference Between Dates


It could be argued getting the difference between two dates was the most painful operation to do with dates prior to Java 8. The new Date API makes determining the amount of days,weeks,months or years between dates equally as easy with the `LocalDate.until` method:

    
    
    @Test
        public void test_get_days_between_dates() {
            LocalDate vacationStart = LocalDate.parse("2014-07-04");
            Period timeUntilVacation = today.until(vacationStart);
    
            assertThat(timeUntilVacation.getMonths(), is(4));
    
            assertThat(timeUntilVacation.getDays(), is(7));
    
            assertThat(today.until(vacationStart, ChronoUnit.DAYS), is(127L));
    
            LocalDate libraryBookDue = LocalDate.parse("2000-03-18");
    
            assertThat(today.until(libraryBookDue).isNegative(), is(true));
    
    
            assertThat(today.until(libraryBookDue, ChronoUnit.DAYS), is(-5094L));
    
            LocalDate christmas = LocalDate.parse("2014-12-25");
            assertThat(today.until(christmas, ChronoUnit.DAYS), is(301L));
    
        }
    


In this example we see the use of the `Period` object again.


## Conclusion


We've wrapped up our quick tour of the `LocalDate` and the Java 8 Date API.  Obviously, there is so much more to discover about working with dates and time in Java 8, this post is just a quick introduction.  Thanks for your time.



## Resources






  * [Joda Time](http://www.joda.org/joda-time/) a Java date-time library to use for Java versions < 8


  * The [java.time](http://docs.oracle.com/javase/8/docs/api/java/time/package-summary.html) package contains the java doc for the classes discussed in this post.


  * [Functional Programming in Java 8](http://pragprog.com/book/vsjava8/functional-programming-in-java) a great resource on using the new functional components in Java 8


  * [Source code](https://github.com/bbejeck/Java-8/blob/master/test/bbejeck/dates/LocalDateTest.java) for this post



