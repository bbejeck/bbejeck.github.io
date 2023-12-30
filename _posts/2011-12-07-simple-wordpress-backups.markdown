---
author: Bill Bejeck
comments: true
date: 2011-12-07 04:36:16+00:00
layout: post
slug: simple-wordpress-backups
title: Simple WordPress Backups
wordpress_id: 1404
categories:
- General
tags:
- ssh
---

<img class="left" src="{{ site.media_url }}/images/black-drive-backup-256x256-150x150.png" /> Backing up your data is an important task.  As we all know, it's not a matter of if you are going to experience a crash or failure, but when.  Blogs are no exception.  I wanted to take a break from my regular style of posts to share my simple backup script.  While I'm know there ar more sophisticated approaches, I really enjoy automating tasks, like remote backups, with bash scripts. My approach is simple. I create a gzipped tar of my WordPress install directory, and then run a [mysqldump](http://dev.mysql.com/doc/refman/5.1/en/mysqldump.html) command.  It's all done locally from my MacBook Pro via ssh and the results end up in my Dropbox directory.  
<!--more-->

### Setting up Password-less SSH


While this might be obvious, it is very helpful to set up your account to use password-less logins via public keys.  If you have not generated your private/public ssh keys, you will need to open up a terminal session on your local computer and from the command line type:
[bash]ssh-keygen -t rsa[/bash] After you run this command you will be prompted to:




  1. Enter the file where you want to store the key, hit return to accept the default (~/.ssh/id_rsa)


  2. Enter a passphrase.  It's important to just hit return, then return again when asked to confirm the empty passphrase.


You now have a private key, id_rsa and a public key, id_rsa.pub in the ~/.ssh directory.  It is very important that you do not share your private key with anyone. The next step is to copy the public key, id_rsa.pub to the ~/.ssh/authorized_keys file on the remote server.  Run the following from the terminal:

    
    
    ssh USERNAME@HOST cat < ~/.ssh/id_rsa.pub ">>" ~/.ssh/authorized_keys 
    


With that step completed, it should possible to run remote commands via ssh without being prompted for a password.


### Running the Backup


Now it's time to run the backup script:

    
    
    #! /bin/sh
    
    . backup_config.sh
    
    NOW=$(date +"%m%d%Y")
    FILENAME="${BLOG}_${NOW}.tar.gz"
    DATABASE_BAK="Database_${BLOG}_${NOW}.gz"
    
    echo "Running remote tar backup of blog directory"
    ssh $USER@$IP_ADDRESS "cd $BASE_DIR && tar -zcf ~/$FILENAME ."
    
    
    echo "Begin remote copy"; scp $USER@$IP_ADDRESS:~/$FILENAME $DEST_DIR \
    && echo "Remote copy successful, removing backup on server"; ssh $USER@$IP_ADDRESS "rm ~/$FILENAME"
    
    echo "Starting Mysql dump"
    ssh $USER@$IP_ADDRESS "mysqldump -u$DBUSER -p$PASS $DATABASE | gzip -c" >  $DEST_DIR/$DATABASE_BAK
    echo "Mysql dump completed"
    
    echo "Created backup files:"
    ls -la  $DEST_DIR | grep $NOW
    


Let's go over the interesting parts of the backup script:




  1. Line 3 sources the backup_config.sh file that sets all the variables.


  2. Line 10 changes into the specified directory then makes a gzip compressed tar file of it's contents, placing the resulting file in the home directory


  3. Line 12 then copies the tar file from the remote server to your local machine.  If the copy is successful, the remote tar file is deleted


  4. Line 16 runs a mysqldump commmand (the contents and structure of the database) compressing the output with gzip then sends the contents over the wire to the locally specified database backup file.

As final convenience, I set an alias to the script so I am always just one short command away from running a backup.


### Conclusion


So that is my basic, but effective backup script.  The script has lots of room for improvement:




  * There is no error handling.


  * Copying the entire WordPress install is a little heavy.


  * Automatic removal of old backups should be included.


If you want to try this out for yourself just edit the backup_config.sh file with the appropriate values. Hopefully others will be able to benefit from this simple backup script as well.


#### Resources


All sources for this post can be found at this [gist](https://gist.github.com/1441472).
