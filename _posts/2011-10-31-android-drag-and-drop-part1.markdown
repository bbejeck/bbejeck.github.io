---
author: Bill Bejeck
comments: true
date: 2011-10-31 12:30:34+00:00
layout: post
slug: android-drag-and-drop-part1
title: Android Drag and Drop (Part 1)
wordpress_id: 852
categories:
- Android
tags:
- Android
- Drag-Drop
- UI
---

<img class="left" src="{{ site.media_url }}/images/android_logo_no_words.gif" /> This is the first post in a 3 part series that is going to cover implementing Drag and Drop in an Android application.
(I am currently using version 4.0 of the sdk, but I originally wrote the code in this series with the 3.1 version.)



## Why Use Drag and Drop


When I started working on my Android application, which involves moving a chess piece around a chess board, I chose to use drag and drop functionality for two reasons:




  1. I felt that drag and drop would get as close a possible to the experience of moving the piece around the board.


  2. By using drag and drop it will be much easier to determine valid moves by dropping the chess piece on an individual object (a square in the chess board in this case) rather than keeping track of the chess piece's x,y coordinates.  I'll explain more on how I implemented this in the last installment of this series



<!--more-->

## My Drag and Drop Goals


I had 3 specific goals when it came to implementing drag and drop with regards to moving the chess piece on the board.




  1. When a user presses down and starts to drag, the chess piece would be replaced by a much lighter version of the original, making it obvious that it is being moved.


  2. I wanted to limit the area where the chess piece could be dragged, more specifically I don't want players to drag the piece off the board


  3. If the player tries to drop the game piece that would result in an illegal move, I wanted the game piece to "snap back" to it's original position.


This post is going to cover my first goal for drag and drop in my application.


## Changing the ImageView on Drag Start


As I began working, I realized that the drag drop functionality did not support what I wanted right out of the box, but certainly with a little work there was enough in the api to give me what I was looking for.  When the user presses down on the chess piece (which is an ImageView) to start a drag, here are the steps I take:




  1. Create a DragShadowBuilder object.


  2. Call the startDrag method.


  3. Hide the original ImageView (the chess piece) by calling setVisibility and passing in View.INVISIBLE, leaving only the DragShadowBuilder object visible, clearly indicating the drag has started.


The steps above are all done in the ImageView's OnTouchListner object, in the implemented onTouch method.  Here is the code:

    
    
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
    


Changing the chess piece image on drag start achieved!
Next post - **Limiting the Drag Area**, be sure to come back!
