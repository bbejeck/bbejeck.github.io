---
layout: post
title: "Scripting Tmux for Kafka"
date: 2015-12-06 22:16:57 -0500
permalink: /kafka-tmux/
categories: 
- Tmux
- linux
- General
tags: 
- Tmux
- linux
- General
keywords:
- Kafka
- Tmux 
description: How to leverage Tmux for increased productivity using Kafka as an example.
---
I've known about [tmux](https://tmux.github.io/) for some time now, but I kept putting off working with it. Lately I've started looking at the [Processor API](https://cwiki.apache.org/confluence/display/KAFKA/KIP-28+-+Add+a+processor+client) available in [Kafka](http://kafka.apache.org/).  After opening up 4-5 terminal windows (Zookeeper, Kafka, sending messgaes, recieving messages) and toggling in between them, I quickly realized I needed a way to open up these terminal sessions at one time and have them centralized. I had the perfect use case to use tmux!  This post is about scripting tmux to save time during devlopment, in this case working with kafka.  

<!-- more -->
In this post we are going to assume the reader is alreay familiar with zookeeper and kafka.
### Getting Tmux
Before we start it's helpful to know how we can install tmux.  Tmux can be [downloaded](https://github.com/tmux/tmux/releases/download/2.1/tmux-2.1.tar.gz) from the main site.  In my case I'm using a mac so I used homebrew - `brew install tmux`.


### Scripting Tmux
I wanted a script to create the following workspace:

1.  Zookeeper running in it's own pane.
2.  Kafka server running in it's own pane.
3.  A separate pane to send messages.
4.  A separate pane to receive messages.
5.  Finally a pane to run my kafka-processor code or perform ad-hoc tasks.

I wanted to have zookeeper and kafka start automatically, the message producer and consumer I would be stopping and starting as needed. But I would like each respective pane to cd to the base kafka install directory.  Once I found the relevant tmux commands it was simple to set up.  Here's the script:
```bash Kafka Tmux Script
#!/bin/sh

KAFKA_DIR=/usr/local/kafka_2.11-0.9.0.0-SNAPSHOT

START_ZK="./bin/zookeeper-server-start.sh"
ZK_PROPS="config/zookeeper.properties"

START_KAFKA="./bin/kafka-server-start.sh"
KAFKA_PROPS="config/server.properties"

tmux new-session -s kafka-work -d

#split screen in have horizontally
tmux split-window -v 
#split the second half in half again
tmux split-window -v 

#split the top window in half vertically
tmux split-window -h -t 0
#split the new half top window in half horizontally
tmux split-window -v 

#cd into kafka directory
tmux send-keys -t 1 "cd $KAFKA_DIR"  C-m
#start zookeeper
tmux send-keys -t 1 "$START_ZK  $ZK_PROPS" C-m 

tmux send-keys -t 2 "cd $KAFKA_DIR"  C-m
tmux send-keys -t 2 "$START_KAFKA $KAFKA_PROPS" C-m 

tmux send-keys -t 3 "cd $KAFKA_DIR"  C-m
tmux send-keys -t 4 "cd $KAFKA_DIR"  C-m

tmux attach -t kafka-work
```
While in depth coverage of tmux is beyond the scope of this post, it will be helpful to give a brief description of the tmux commands in the script.

#### Splitting Panes
There are serveral `split-window` commands in the script and they do what you'd expect, split the window either horizontally or vertically.  What's not so obvious is that `split-window -v` splits the window *horizontally* and `split-window -h` splits *vertically*.  Tmux sees the resulting panes differently than we humans.  When the window is divided horizontally, two panes stacked on top of each other *vertically* and for vertical splits, panes are horizontally adjacent to each other.  There is one final note about splitting a window into panes that needs mentioning.  When the `split-window` command is issued the cursor ends up in the most recently created pane.  This is important to keep in mind if you are going to issue additional commands but don't want them executed in the pane just created.

#### Pane Numbers
By default, tmux numbers the panes in a given window starting at 0.  If the initial window is split in two panes horizontally, the top pane is still 0, and the bottom half is now number 1.  While this seems very natural, when additional panes are split in different areas, the numbering may seem confusing. Just keep in mind panes are numbered in order of creation.

#### Sending Commands to Panes
Finally we have the `send-keys -t N <some command>` commands in the script.  This is so we can target a particular pane with the given command.  In our case we want 4 of our panes to be in the base kakfa install directory so we issue the `tmux send-keys -t N "cd $KAFKA_DIR"  C-m` command.  There are 4 such commands, each one with a specific target pane 1-4. We want to start zookeeper in pane 1 so the `tmux send-keys -t 1 "$START_ZK  $ZK_PROPS" C-m ` command is used.  To start the kafka server we issue a very similar command, except pane 2 is the target.

### Final Tmux - Kafka Results
Here's a screen shot of the final results of running the the kafka-tmux script.  The numbers in the panes are for illustration purposes only.
<img class="center" src="{{ site.media_url }}/images/kafka-tmux.png" /> 

### Conclusion
This has been a quick tour of tmux.  It is woefully incomplete and only scratches the surface on what we can do with tmux and how it can be configured. I just wanted to share my script in case someone might find it helpful.   Thanks for your time.

### Resources

1.  [Tmux Man Page](http://www.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man1/tmux.1?query=tmux&sec=1)
2.   [Tmux - Productive Mouse Free Development](https://pragprog.com/book/bhtmux/tmux) This is a book on tmux written by Brian Hogan.  It's an excellent resource and it's what I used to help me stitch together the script in this post.
3.   [Kafka-Tmux Template](https://gist.github.com/bbejeck/0ed144c047763702a042) - the script source.