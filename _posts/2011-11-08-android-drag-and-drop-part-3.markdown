---
author: Bill Bejeck
comments: true
date: 2011-11-08 12:00:51+00:00
layout: post
slug: android-drag-and-drop-part-3
title: Android Drag and Drop (Part 3)
wordpress_id: 1027
categories:
- Android
tags:
- Android
- Drag-Drop
- UI
---

<img class="left" src="../assets/images/android_logo_no_words.gif" /> This is the final post in a 3 part series on implementing Drag and Drop in an Android application.  The [first post](http://codingjunkie.net/android-drag-and-drop-part1/) covered hiding an ImageView when starting a drag.  The [second post](http://codingjunkie.net/android-drag-and-drop-part-2/) covered handling drops outside an intended drop zone. This final installment covers moving a game piece as a result of a valid move or having that game piece “snap back” to it's original postion when an invalid move is made.
<!--more-->

### Application Background Information


Some background information on my application might be helpful for this post.  The game involves moving a single chess piece (a knight) around a chessboard. The chessboard is a TableLayout and each square is a subclass of LinearLayout and has an OnDragListener set. Additionally, each OnDragListener has a reference to a "mediator" object that:




  1. Handles communication between objects.


  2. Keeps track of the current square (landing square of the last valid move).




### Determining Valid Moves


When the user starts to drag the ImageView and passes over a given square, the following actions are possible:




  * Use the ACTION_DRAG_ENTERED event to set the instance variable 'containsDraggable' to true.


  * Use the ACTION_DRAG_EXITED event to set 'containsDraggable' to false.


  * When an ACTION_DROP event occurs on a square, the mediator is asked if the attempted move is valid by checking all possible valid moves (pre-calculated) from the "current" square.


Here is the code with the relevant sections highlighted:

    
    
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
    


As you can see from above, regardless if the move is valid or not, the ImageView is made visible. By making the ImageView visible immediately after the drop, my goals of a seamless move, or "snapping back" to it's original postion are met. 


### Moving the ImageView


Earlier, I mentioned that each square was a subclass of LayoutView.  This was done for the ease of moving the ImageView from square to square.  Here are the details from the checkForValidMove method (line 14 in the code sample above) that show how the move of the ImageView is accomplished.

    
    
    private void checkForValidMove(ChessBoardSquareLayoutView view, View dragView) {
            if (mediator.isValidMove(view)) {
                ViewGroup owner = (ViewGroup) dragView.getParent();
                owner.removeView(dragView);
                view.addView(dragView);
                view.setGravity(Gravity.CENTER);
                view.showAsLanded();
                mediator.handleMove(view);
            }
        }
    




This concludes the series on Android Drag and Drop.  It is hoped that the reader found this series helpful in their own Android development efforts.


