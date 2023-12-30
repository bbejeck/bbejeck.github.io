---
author: Bill Bejeck
comments: true
date: 2011-11-03 02:01:18+00:00
layout: post
slug: android-drag-and-drop-part-2
title: Android Drag and Drop (Part 2)
wordpress_id: 948
categories:
- Android
tags:
- Android
- Drag-Drop
- UI
---

<img class="left" src="{{ site.media_url }}/images/android_logo_no_words.gif" /> This is the second post in a 3 part series on implementing Android Drag and Drop.  In my [ last post](http://codingjunkie.net/android-drag-and-drop-part1/) I covered how to hide an ImageView and just show the DragShadowBuilder object at the start of a drag. This post is going to cover my efforts on restricting the drag area.


### Trying to Restrict the Drag Area


Since my application involves moving a chess piece around board, I wanted to restrict the dragging area to the game board itself. My motivation is that the ImageView is hidden at the start of the drag.  If the ImageView is dropped off the board, it will never reappear.  That would leave my application in an inconsistent state.  So I spent some time looking through the Android api, and Googling how I might limit the drag area.  I came up with nothing.  Then I took a step back and thought what problem am I trying to solve? Is it the user dragging the ImageView off the board? No, the real problem is the drop action is never handled, so I never get a chance to make the ImageView visible again.  So I redefined my problem to be determining when a drop is valid or not. 
<!--more-->

### Determining Valid Drops


To figure out how to determine valid drops, I went back to the Android Developer documentation.  I honed in on [handling drag end](http://developer.android.com/guide/topics/ui/drag-drop.html#HandleEnd) events. Here are the key points I found:




  * When a drag ends, an ACTION_DRAG_ENDED event is sent to all DragListeners.


  * A DragListener can determine more information about drag operations by calling the getResult method of the DragEvent.


  * If a DragListener returned true from an ACTION_DROP event, the getResult call will return true, otherwise false.


Now it is very clear to me what I can do when/if the user drags and drops the chess piece in an unintended area.  My DragListener class already returns true for any event that is handled in the onDrag method.  So a return value of false from getResult indicates the drop happened somewhere off the game board. What I need to do now is:


  1. When an ACTION_DRAG_ENDED event happens, call the getResult method


  2. If getResult returns false, immediately set the visibility of the ImageView (used at the start of the drag) back to visible.

Here is the code, the relevant lines are highlighted:


    
    
    @Override
        public boolean onDrag(View view, DragEvent dragEvent) {
            int dragAction = dragEvent.getAction();
            View dragView = (View) dragEvent.getLocalState();
            if (dragAction == DragEvent.ACTION_DRAG_EXITED) {
                containsDragable = false;
            } else if (dragAction == DragEvent.ACTION_DRAG_ENTERED) {
                containsDragable = true;
            } else if (dragAction == DragEvent.ACTION_DRAG_ENDED) {
                if (dropEventNotHandled(dragEvent)) {
                    dragView.setVisibility(View.VISIBLE);
                }
            } else if (dragAction == DragEvent.ACTION_DROP && containsDragable) {
                checkForValidMove((ChessBoardSquareLayoutView) view, dragView);
                dragView.setVisibility(View.VISIBLE);
            }
            return true;
        }
    
        private boolean dropEventNotHandled(DragEvent dragEvent) {
            return !dragEvent.getResult();
        }
    
    



Now users can drag and drop the ImageView anywhere and the game will not end up in an inconsistent state. My second goal is achieved.  The next and final post will cover **determining valid moves**. Be sure to come back!

