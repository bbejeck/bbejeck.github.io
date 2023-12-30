---
author: Bill Bejeck
comments: true
date: 2011-10-06 02:54:11+00:00
layout: post
slug: android-unit-testing
title: Android Unit Testing
wordpress_id: 791
categories:
- Android
tags:
- Android
- Junit
- Mockito
- PowerMock
---

This post is going to cover unit testing a native Android application.   While working on my own modest Android application, I wanted to add some non-instrumented unit tests and was surprised  how challenging it was to use mock objects.  Admittedly, instrumented tests running on a emulator or actual device is probably a better way to go, but I wanted a faster way to make sure that my internal logic was still working as expected when making changes.  


### Code Under Test
<!--more-->

Here is one example of some of the code I was trying to test:

    
    
    public class ViewOnTouchDragStartListener implements View.OnTouchListener {
    
        @Override
        public boolean onTouch(View view, MotionEvent motionEvent) {
            if (motionEvent.getAction() == MotionEvent.ACTION_DOWN) {
                ClipData clipData = ClipData.newPlainText("", "");
                View.DragShadowBuilder dsb = new View.DragShadowBuilder(view);
                view.startDrag(clipData, dsb, view, 0);
                view.setVisibility(View.INVISIBLE);
                return true;
            } else {
                return false;
            }
        }
    }
    


When I started writing my test method my plan was simple, just use [Mockito](http://code.google.com/p/mockito/) to create mocks for the View and MotionEvent objects, set the expectations, run the test and watch the green bar.   Instead all I received was a slew of **RuntimeException("Stub!")** errors.  So I went back to the drawing board (Google) which led me to PowerMock.   I have been using Mockito for some time now and I was aware of  [PowerMock](http://code.google.com/p/powermock/) that can be used to extend the capabilities of frameworks like Mockito and EasyMock to enable mocking of constructors, private methods and static methods among others. This seemed to be exactly what I was looking for to help with my Android unit testing issues.


### Unit testing take two


I downloaded [powermock-mockit-junit-1.4.10](http://code.google.com/p/powermock/downloads/list) (the most current as of this writing)  There are also downloads available that work with [TestNG](http://testng.org/doc/index.html) and [EasyMock](http://easymock.org/). After some consultation with the [documentation](http://code.google.com/p/powermock/wiki/MockitoUsage13) here is my improved unit test

    
    
    @RunWith(PowerMockRunner.class)
    @PrepareForTest({MotionEvent.class, ClipData.class,ViewOnTouchDragStartListener.class,View.class})
    public class ViewOnTouchDragStartListenerTest {
        private ViewOnTouchDragStartListener dragStartListener;
        private View view;
        private MotionEvent motionEvent;
        private ClipData clipData;
        private View.DragShadowBuilder dragShadowBuilder;
    
        @Before
        public void setUp() throws Exception {
            dragStartListener = new ViewOnTouchDragStartListener();
            view = PowerMockito.mock(View.class);
            PowerMockito.mockStatic(ClipData.class);
            clipData = PowerMockito.mock(ClipData.class);
            motionEvent = PowerMockito.mock(MotionEvent.class);
            dragShadowBuilder = PowerMockito.mock(View.DragShadowBuilder.class);
        }
    
        @Test
        public void testOnTouchActionDown() throws Exception {
            PowerMockito.when(motionEvent.getAction()).thenReturn(MotionEvent.ACTION_DOWN);
            when(ClipData.newPlainText("", "")).thenReturn(clipData);
            PowerMockito.whenNew(View.DragShadowBuilder.class).withArguments(view).thenReturn(dragShadowBuilder);
            boolean isActionDown = dragStartListener.onTouch(view, motionEvent);
            InOrder inOrder = inOrder(view);
            inOrder.verify(view).startDrag(Matchers.<ClipData>anyObject(), Matchers.<View.DragShadowBuilder>anyObject(), eq(view),eq(0));
            inOrder.verify(view).setVisibility(View.INVISIBLE);
            assertThat(isActionDown,is(true));
        }
    


I found this fairly straight forward to implement, and best of all, I started getting green for all of my unit tests!  I was impressed with PowerMock's ability to mock out the constructor call to View.DragShadowBuilder inside the ViewOnTouchDragStartListener's onTouch method  (line 24 of test code sample and line 5 of the sample code, respectively).  Admittedly, there probably should have been a separate method that would create and return the DragShadowBuilder object then use the PowerMock/Mockito **_spy_** functionality that allows you to selectively mock out methods on real objects. 

Hopefully this will help you with your Android unit testing endeavors.

