---
author: Bill Bejeck
comments: true
date: 2010-05-23 04:31:36+00:00
layout: post
slug: binary-search-tree-programming-praxis-solution
title: Binary Search Tree - Programming Praxis solution
wordpress_id: 295
categories:
- Data Structures
- Puzzles
tags:
- Puzzles
- ruby
---

Earlier this year I was influenced by two things that got me to re-think what I work on in my free time.  The first was [this podcast](http://www.se-radio.net/podcast/2009-11/episode-150-software-craftsmanship-bob-martin) from Software Engineering Radio with “Uncle Bob”  Bob Martin on software craftsmanship.  The second was [a blog on the value of fundamentals in software development](http://www.skorks.com/2010/04/on-the-value-of-fundamentals-in-software-development/).  As a result I am now focusing on incorporating [puzzles](http://sixrevisions.com/resources/10-puzzle-websites-to-sharpen-your-programming-skills/) as well as learning new languages or frameworks .  This entry is a solution to the [Binary Search Tree problem from Programming Praxis](http://programmingpraxis.com/page/3/). While this post does not aim to be a thorough explanation of how a binary search tree works, I will present my code and briefly explain it.
<!--more-->
The problem asked you to implement search, insert delete and enlist  ( I took enlist to mean generate a list of values from the tree in order).  Since I use Java on the day job, and I want to get better in other languages, I implemented my solution in Ruby.   Here are the public methods that fufill the requirements of the problem.

    
    require 'node'
    
    module Trees
      class BinarySearchTree
        attr_reader :root
    
        def initialize(root = nil)
            @root = root
        end
    
        def search(value)
            search_for_node(@root,Node.new(value))
        end
    
        def insert(value)
            @root = insert_value(@root,value)
        end
    
        def delete(value)
          @root = delete_node(@root,Node.new(value))
        end
    
        def in_order_list
            vals = []
             inorder(vals,@root)
            vals
        end


These methods just delegate to private methods that actually do the work.  Let's consider search and insert first.

    
          
    private
    
            def search_for_node(tnode,node)
                if tnode.nil?
                   return nil
                end
    
                if tnode == node
                   tnode = node
                elsif node < tnode
                   tnode = search_for_node(tnode.left,node)
                else
                   tnode = search_for_node(tnode.right,node)
                end
               tnode
            end
    
            def insert_value(tnode,value)
              if tnode.nil?
                   tnode = Node.new(value)
              elsif value < tnode.value               
                   tnode.left = insert_value(tnode.left,value)          
              elsif value > tnode.value
                   tnode.right = insert_value(tnode.right,value)
              elsif value == tnode.value
                   tnode.value = value
              end
             tnode
            end


Search and insert work almost identically.  With search you keep traversing the tree until you find your value, or return nil when you have exhausted your options.  With insert you actually want to keep going until you find a nil node, then you know thats where to insert.  One thing I did in my insert method was that if you supply a value that already exists in the tree I simply update the node with that value.  I'm sure there are better ways of handling that situation, but I felt simply updating with an existing value causes no harm.  Now, deleting is not quite so straight forward, although still not that difficult.

    
    
           def delete_node(tnode,node)
                if tnode == node
                   tnode = remove(tnode)
                elsif node < tnode
                   tnode.left = delete_node(tnode.left,node)
                else
                   tnode.right = delete_node(tnode.right,node)
                end
               tnode
            end
    
            def remove(node)
              if node.left.nil? && node.right.nil?
                 node = nil
              elsif !node.left.nil? && node.right.nil?
                 node = node.left
              elsif node.left.nil? && !node.right.nil?
                 node = node.right
              else
                 node = replace_parent(node)
              end
             node
            end


When deleting a node from the tree there are 4 cases to consider:



	
  1. The node has no children

        
  2. The node has a left child only

	
  3. The node has a right child only

	
  4. The node has both a right and left child


The first three are pretty straight forward to handle.  With no children, obviously the node to be deleted is just set to null.  If the node only has one child you merely set the reference of the node to be deleted to that of it's child.  The situation of 0-1 child nodes is handled by the first three branches of the if-else statements in the **remove** method.  The following methods are used to handle the case of a node with 2 children.

    
    
           def replace_parent(node)
                 node.value = successor_value(node.right)
                 node.right = update(node.right)
                 node
            end
    
            def successor_value(node)
                unless node.left.nil?
                  successor_value(node.left)
                end
                node.value
            end
    
            def update(node)
                unless node.left.nil?
                  node.left = update(node)
                end
                node.right
            end
    


Deleting a node with 2 children is handled in two steps.  First you update the value of the node to be deleted with the value of the node's in-order successor or predecessor.  Then the duplicate node needs to be deleted and handle the case if it has a child node.  It is worth noting due to the properties of binary trees, an in-order successor or predecessor will only ever have a right or left child.  
For me it seemed more natural to use the in-order successor.  The **replace_parent** method first updates the value of the node to be deleted with the value returned from the **successor_value** method.  Then duplicate node is then removed and the right child of the newly updated node is set to the right child of the in-order successor or null.  This last part is handled by the **update** method.  Here is the code to get a list of the values from the tree in ascending order

    
    
           def inorder(list, node)
              unless node.nil?
                 inorder(list, node.left)
                 list.push(node.value)
                 inorder(list, node.right)
              end
            end
    



Finally here is the code for the **Node** class:

    
    
    module Trees
      class Node
        include Comparable
        attr_accessor :left,:right,:value
    
        def initialize(value = nill, left = nil, right = nil)
            @value = value
            @left = left
            @right = right
        end
        
        def <=>(otherNode)
          @value <=> otherNode.value
        end
    
        def to_s
          "[value: #{@value} left=> #{@left} right=> #{@right}]"
        end
      end    
    end
    
