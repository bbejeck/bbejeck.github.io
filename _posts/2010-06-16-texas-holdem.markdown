---
author: Bill Bejeck
comments: true
date: 2010-06-16 03:54:14+00:00
layout: post
slug: texas-holdem
title: Texas Hold'Em
wordpress_id: 326
categories:
- Puzzles
- ruby
---

This post is my solution to [Texas Hold'Em](http://programmingpraxis.com/2010/03/23/texas-hold-em/).  The problem involved taking 7 cards and determining the single best 5 card hand.  Like the previous problem, I decided to implement this in Ruby.  I probably went too far because what I ended up with was a command line program that when you hit the enter key you see a print out of the 7 cards dealt and the best hand from those cards, not to mention a lot more code than I originally thought I was going to write.    To me this seems like a classic [Dynamic Programming](http://en.wikipedia.org/wiki/Dynamic_programming) problem, so that was the direction I went.  
The first thing I did after dealing a hand was to sort the cards numerically.   By virtue of the fact that the cards will only go up in value, it is much easier to rule out what kind of hand is possible just by the position of the cards.  We simply take two cards away at time figure out the hand of the five remaining cards and store the results.  After that we loop over the results and only keep the highest scoring hand.  What follows is the main class that can be run from the command line.

<!--more-->
**texasholdem.rb:**

    
    
    require 'deck'
    require 'card'
    require 'player'
    
      card_deck = Deck.new
      keep_running = true
      
      while keep_running
        results = {}
        card_deck.shuffle
        seven_cards = card_deck.deal
        seven_cards.sort!
        copy = Array.new(seven_cards)
    
        puts "Press enter to deal cards.."
        line = gets
       
        if line.chop == "quit"
           keep_running = false
        else
            for i in 0..5
              for j in i..5
                copy.delete_at(i)
                copy.delete_at(j)
                hand = Player.check_hand(copy,0,0,nil)
                if results[hand]
                  copy = Player.compare_hands(copy, results[hand], 4)
                end
                results[hand] = copy
                copy = Array.new(seven_cards)
              end
              copy = Array.new(seven_cards)
            end
    
            max_score = -1
            winning_hand = nil
            best_cards = nil
            results.each do |hand, cards|
                score = Player.ranks[hand]
                score = score.nil? ? cards[4].value : score
              if score > max_score
                 max_score = score
                 score = Player.ranks[hand]
                 best_cards = cards
                 winning_hand = hand
              end
            end
             puts "Best hand from #{seven_cards.join(", ")} "
             puts "\n\t #{winning_hand} : #{best_cards.join(", ")}"
      end
      
    end
    

   
The Player class recursively goes through the five cards in question to determine the hand, returning the last card in the array if no other hand is determined.  The Player class also contains a method for breaking ties of the same hand by comparing the values of the cards.

**player.rb:**

    
    
    require 'card'
    
    class Player
      
      STRAIGHT="straight"
      STRAIGHT_FLUSH="straight flush"
      PAIR = "pair"
      TWO_PAIR ="two pair"
      FLUSH = "flush"
      FULL_HOUSE = "full house"
      THREE_KIND = "3 of a kind"
      FOUR_KIND = "4 of a kind"
      @@ranks =  {STRAIGHT_FLUSH=>22,FOUR_KIND=>22,FULL_HOUSE=>21,FLUSH=>20,STRAIGHT=>19,THREE_KIND=>18,TWO_PAIR=>17,PAIR=>16}
    
      def initialize
        
      end
    
       def self.ranks
          @@ranks
       end
       def self.compare_hands(hand,other_hand, index)
            if index < 0
               return hand
            elsif hand[index].value > other_hand[index].value
               return hand
            elsif hand[index].value < other_hand[index].value
               return other_hand
            else
               compare_hands(hand,other_hand,index-1)
            end
       end
       
       def self.check_hand(cards,index,match_index,current_hand)
          if(index+1 < cards.length)
               if cards[index].eq?(cards[index+1])
                     current_hand = hand_eq(current_hand,(index-match_index)<=1)
                     match_index = index
               elsif cards[index].sequential?(cards[index+1])
                     current_hand = hand_seq(current_hand,index)
               elsif current_hand == STRAIGHT
                     current_hand = nil
              end
              current_hand = check_hand(cards,index+1,match_index,current_hand)
          end
          current_hand = check_flush(cards,current_hand)
    
          if current_hand.nil?
             current_hand = cards[4]
          end
         current_hand
       end
    
    private
    
       def self.check_flush(cards,current_hand)
         if current_hand != FULL_HOUSE && current_hand != FOUR_KIND && current_hand != STRAIGHT_FLUSH && current_hand != FLUSH
               suits_match = true
               for i in 0..3
                  if cards[i].suit != cards[i+1].suit
                     suits_match = false
                  end
               end
               if suits_match
                 current_hand = (current_hand == STRAIGHT) ? STRAIGHT_FLUSH : FLUSH
               end
           end
         current_hand
       end
    
       def self.hand_eq(current_hand, sequential)
           case current_hand
                 when nil
                   hand = PAIR
                 when PAIR
                     if sequential
                       hand = THREE_KIND
                     else
                       hand = TWO_PAIR
                     end
                   when TWO_PAIR
                     hand = FULL_HOUSE
                   when STRAIGHT
                     hand = PAIR
                   when THREE_KIND
                     if sequential
                       hand = FOUR_KIND
                     else
                       hand = FULL_HOUSE
                     end
    
            else
                   hand = current_hand
            end
           hand
       end
    
       def self.hand_seq(current_hand,current_index)
            case current_hand
               when nil
                 if current_index == 0
                   hand = STRAIGHT
                 end
               when STRAIGHT
                 hand = STRAIGHT
               else
                   hand = current_hand
            end
           hand
       end
    end
    

   
Finally there is a Card class and a Deck class.  I'm not sure a separate classes are necessary as they definitely add to the total line count, but I think it makes the code cleaner and easier to read by abstracting away behavior into separate classes.  For example the Deck class has deal and shuffle methods that to me make reading the code easier, so that is a trade off I am willing to take most of the time.
**card.rb:**

    
    
    
    class Card
      attr_reader :suit, :value, :face_value
      
      def initialize (suit, face_value)
          @suit = suit;
          @face_value = face_value;
          case @face_value
           when "Jack"
              @value = 11
           when "Queen"
              @value = 12
           when "King"
              @value = 13
           when "Ace"
              @value = 15
          else
            @value = @face_value.to_i
          end
      end
    
      def eq? (other)
         @value == other.value
      end
    
      def sequential? (other)
         (@value + 1) == other.value
      end
    
      def <=> (other)
        @value <=> other.value
      end
      
      def to_s
        "#{@face_value} of #{@suit}s"
      end
    end
    


**deck.rb**

    
    
    require 'card'
    
    class Deck
      attr_reader :deck
      def initialize
        @deck = []
        values = ["2","3","4","5","6","7","8","9","10","Jack","Queen","King","Ace"]
        suits = ["Heart","Club","Spade","Diamond"]
        suits.each do |suit|
               values.each do |value|
                     @deck.push(Card.new(suit,value))
               end
        end
      end
      
      def to_s
         @deck.join(',')
      end
      
      def deal
        hand = []
        start = rand(46)
        index = -1
        start.upto((start+6)) { |i| hand[index+=1] = @deck[i]; }
        hand
      end
    
      def shuffle
         @deck.shuffle!
      end
    end
    
