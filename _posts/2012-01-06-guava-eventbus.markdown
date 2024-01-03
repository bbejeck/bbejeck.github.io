---
author: Bill Bejeck
comments: true
date: 2012-01-06 07:41:19+00:00
layout: post
slug: guava-eventbus
title: Event Programming with Google Guava EventBus
wordpress_id: 1597
categories:
- General
- Guava
- java
tags:
- Guava
- java
---

<img class="left" src="../assets/images/toolbox.jpg" />  It's a given in any software application there are objects that need to share information in order to get work done.  In Java applications, one way of achieving information sharing is to have event listeners, whose sole purpose is to take some action when a desired event occurs.  For the most part this process works and most experienced Java developers are used to writing the requisite anonymous inner class that implements some event listener interface.  This post is about taking a different approach to handling Java events using Guava's EventBus.  The EventBus allows for objects to _subscribe_ for or _publish_ events, without having explicit knowledge of each other. The EventBus is not meant to be a general purpose publish/subscribe system or support interprocess communication. *NOTE*: This is an updated post per reader comments below.  
<!--more-->

### EventBus Class


The EventBus is very flexible and can be used as a singleton, or an application can have several instances to accomodate transferring events in different contexts.  The EventBus will dispatch all events serially, so it is important to keep the event handling methods lightweight.  If you need to do heavier processing in the event handlers, there is another flavor of the EventBus, AsyncEventBus.  The AsyncEventBus is identical in functionality, but takes an ExecutorService as a constructor argument to allow for asynchronous dispatching of events.


### Subscribing for Events


An object subscribes for events by taking the following steps: 




  1. Define a public method that takes a single argument of the desired event type and place a @Subscribe annotation on that method.


  2. Register with the EventBus by passing an instance of the object to the EventBus.register method.


Here is a brief example with details omitted for clarity:

    
    
     public class PurchaseSubscriber {
        @Subscribe
        public void handlePurchaseEvent(PurchaseEvent event) {
           .....
        }
       .....
    
      EventBus eventBus = new EventBus();
      PurchaseSubscriber purchaseSubscriber = new PurchaseSubscriber();
      eventBus.register(purchaseSubscriber);
    


There is another annotation that can be used in conjunction with @Subscribe, and that is @AllowConcurrentEvents. The @AllowConcurrentEvents marks a handler method as thread safe so an EventBus (most likely an AsyncEventBus) could potentially call the event handler from simultaneous threads.  One interesting thing I discovered in unit testing is that if a handler method does not have the @AllowConcurrentEvents annotation, it will invoke handlers for an event serially even when using the AsyncEventBus.  It's important to note that @AllowConcurrentEvents will not mark a method as an event handler, the @Subscribe annotation still needs to be present. Finally, the event handling method must have one and only one parameter or an IllegalArgumentException will be thrown when you register the object with the EventBus.



### Publishing Events


Publishing events with the EventBus is equally straight forward.  In the section of code where you want to send the notification of an event, call EventBus.post and all subscribers registered for that event object will be notified. 

    
    
      public void handleTransaction(){
        purchaseService.purchase(item,amount);
        eventBus.post(new CashPurchaseEvent(item,amount));
        ....
      }
    


While it might be obvious, it's important that the subscribing and publishing classes share the same EventBus instance, and it would make sense to use Guice or Spring to help manage the dependencies.



### More About Event Handlers


A very powerful feature of the EventBus is that you can make your handlers as course or fine grained as needed.  The EventBus will call the registered subscribers for all subtypes and implemented interfaces of a posted event object. For example, to handle any and all events one could create an event handler that takes a parameter of type Object. To handle only singular events, create a handler that is very type specific.  To help illustrate, consider the following simple event hierarchy:

    
    
    public abstract class PurchaseEvent {
            String item;
       public PurchaseEvent(String item){
           this.item = item;
       }
    }
    
    public class CashPurchaseEvent extends PurchaseEvent {
           int amount;
           public CashPurchaseEvent(String item, int amount){
              super(item);
              this.amount = amount;
           }
    }
    
    public class CreditPurchaseEvent extends PurchaseEvent {
           int amount;
           String cardNumber;
           public CreditPurchaseEvent(String item,int amount, String cardNumber){
              super(item);
              this.amount = amount;
              this.cardNumber = cardNumber;
           }
    }
    


Here are the corresponding event handling classes:

    
    
    //Would only be notified of Cash purchase events
     public class CashPurchaseSubscriber {
         @Subscribe
         public void handleCashPurchase(CashPurchaseEvent event){
          ... 
         }
     }
     //Would only be notified of credit purchases
     public class CreditPurchaseSubscriber {
        @Subscribe
        public void handleCreditPurchase(CreditPurchaseEvent event) {
         ....
        }
    }
    //Notified of any purchase event
     public class PurchaseSubscriber {
        @Subscribe
        public void handlePurchaseEvent(PurchaseEvent event) {
           .....
        }
    }
    


If there is a need to capture a broad range of event types, an alternative would be to have more than one event handling method in a class.  Having multiple handlers could be a better solution as you would not have to do any "instanceof" checks on the event object parameter.  Here's a simple example from my unit test:

    
    
    public class MultiHandlerSubscriber {
        
    private List<CashPurchaseEvent> cashEvents = new ArrayList<CashPurchaseEvent>();
    private List<CreditPurchaseEvent> creditEvents = new ArrayList<CreditPurchaseEvent>();
    private List<SimpleEvent> simpleEvents = new ArrayList<SimpleEvent>();

    private MultiHandlerSubscriber() {}

    @Subscribe
    public void handleCashEvents(CashPurchaseEvent event) {
        cashEvents.add(event);
    }

    @Subscribe
    public void handleCreditEvents(CreditPurchaseEvent event) {
        creditEvents.add(event);
    }

    @Subscribe
    public void handleSimpleEvents(SimpleEvent event) {
        simpleEvents.add(event);
    }

    //Using a factory method to prevent the 'this' ref from escaping during object construction
    public static MultiHandlerSubscriber instance(EventBus eventBus) {
        MultiHandlerSubscriber subscriber = new MultiHandlerSubscriber();
        eventBus.register(subscriber);
        return subscriber;
    }
    ....
    

*NOTE-* This example has been updated after a reader pointed out registering with the EventBus in the constructor allows the "this" reference to escape. Now the a Factory Method is used instead per "Safe Construction Practices" section 3.2.1 from [Java Concurrency in Practice](http://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601)



### Testing


Since the event handlers are just plain methods, they can easily be tested by instantiating an EventBus in the test case or simulate the EventBus by passing the appropriate event object.  When working with the EventBus I found it was very easy to:




  * Forget to register the subscribing object with the EventBus


  * Neglect to add the @Subscribe annotation.


If it seems that an event handler is not being called, check for those two errors first.
A useful debugging technique is to subscribe to the DeadEvent class.  An EventBus will wrap any published event with no handlers in a DeadEvent instance. DeadEvent provides the getEvent method that returns the original event object. 


### Conclusion


The Guava EventBus class provides an attractive and useful alternative to the standard Java event handling mechanism.  It is hoped the reader will find the EventBus as useful as I do.  As always comments and suggestions are welcomed.



#### Resources






  * [EventBus API](http://docs.guava-libraries.googlecode.com/git-history/v10.0.1/javadoc/com/google/common/eventbus/package-summary.html)


  * [Guava API](http://code.google.com/p/guava-libraries/)


  * [Sample Code](https://github.com/bbejeck/guava-blog/blob/master/src/test/java/bbejeck/guava/eventbus/EventBusTest.java)


  * [Event Collaboration](http://www.martinfowler.com/eaaDev/EventCollaboration.html)



