---
layout: post
slug: named-pipes
title: "I/O with Files that Aren't Files"
date: 2014-08-21 23:43:26 -0400
permalink: /named-pipes
comments: true
categories: linux 
tags: 
- linux 
- cmd-line
- named-pipes
keywords: 
- linux 
- named-pipes
description: Using named pipes when working with files.
---

 Recently at work I needed to search through our archived files and provide the results by the end of the day. Here's the parameters of the request:

 1.  The archive files are encrypted and stored in HDFS (Don't ask why we store them in HDFS).
 2.  The files vary in size form 3-9 GB.
 3.  The total number of files to search was 300+
 4.  It takes between 1 - 2 minutes to decrypt each file.

 In the past there have been requests to search one archived file.  In those cases we would copy the file out of HDFS to a server. Then run a shell script to decrypt the file and perform the search.  The decrypting program requires 2 arguments: an encrypted file and a file to write the decrypted data to.  This means the decrypted and encrypted file are on disk at the same time.  

 At an average rate of of 1.5 minutes to decrypt a single file, it was going to take 450 minutes (7.5 hours) for 300 files. To add to my dilema, there wasn't enough time to write custom [RecordReader](https://hadoop.apache.org/docs/r2.5.0/api/org/apache/hadoop/mapreduce/RecordReader.html). The only solution would be to stream the files in parallel. But there 2 problems with that approach:

 1.  The server does not have enough space for 20 (10 encrypted and 10 decrypted) files at a time.  
 2.  The decrypting code does read from stdin or write to stdout. 

  
 What to do? Use named pipes of course! 

 <!--more-->

### A Crash Course in Named Pipes
Pipes are used to compose narrowly focused programs together to solve broader problems.  For example :

``` bash Simple Pipes Example
cat someFile.txt | awk -F "|" '{print $5}' | cut -d ':' f2
```
In this example we are using *annonymous* pipes (desigated by the '|' character).  Pipes used on the command line or in scripts only live for the life of the current process.   Named pipes are similar with these exceptions:

1. You create named pipes with the command `mkfifo name-of-pipe`.
2. Can be re-used by mulitple processes.
2. Namped pipes persist until deleted with the command `rm name-of-pipe`.
3. Exist on the filesystem and appear as files to other processes.
4. Allow for inter-process communication.
 
Here's an example of creating a named pipe and how it looks on the filesystem:

``` bash Create Named Pipe
new-host-2:~ bbejeck$ mkfifo pipe1
new-host-2:~ bbejeck$ ls -l pipe1 
prw-r--r--  1 bbejeck  staff  0 Aug 16 21:28 pipe1
```
Notice here the named pipe shows in the `ls` results and the first column on the left is a `p` .  Once you don't need a named pipe any more you simply delete it like you would a file with the command `rm pipe1`. 

#### My Solution with Named Pipes
First I created a file comprised of the paths to the archives.  Then a script iterated over the file and ran 10 decryption/search processes in parallel.  Once 10 processes were started, the script would wait until all 10 were done before starting another batch.

``` bash Looping over the file list
    count=0
    while read fileName; do
       if [[ $((++count)) -eq 10 ]]; then
              count = 0
              wait
        fi 
        #backgrounded process      
        processFile $fileName &
    done < inputFileNames.txt
```
Here's the code for the `processFile` function.

``` bash processFile function
function processFile {
    encryptedFile=$(basename ${1})
    decryptedFile=${encryptedFile%.enc}

    mkfifo ${encryptedFile}
    mkfifo ${decryptedFile}

    hdfs dfs -cat ${1} > ${encryptedFile} &
    cat ${decryptedFile} | awk '{ if ( $5 == someValue) print $0}' &

    /somepath/decryptFile.sh ${encryptedFile} ${decryptedFile}
    rm ${encryptedFile} ${decryptedFile}
}
```
While the `processFile` function is straight forward, there are several steps we should explain:

1.  Line 2: Create a variable with the basename of the encrypted file from the full path.
2.  Line 3: Create a variable of the name of the encrypted file less the `.enc` file extension.
3.  Lines 5,6: Create named pipes for the encrypted and decrypted file.
4.  Line 8: Stream the encrypted file from HDFS and redirect to the encrypted named pipe and background the process.
5. Line 9: Run the `cat` command on the decrypted file. Then pipe the results to `awk` and search for the information in question.  This is also backgrounded.
6. Line 11: Invoke the decryption script with the required file parameters.
7. Line 12: After decrypting and searching the file, delete the named pipes.


### Conclusion
Using named pipes enabled me to decrypt and search of 300+ files in roughly 1.5 hours.  I also avoided the space issue by never having to land a file on disk.  While named pipes aren't needed every day, they are a useful tool to have in your arsenal.  

### References

 * [Definition of named pipes on Wikipedia](http://en.wikipedia.org/wiki/Named_pipe)
 * [Good description of named pipes in Linux Journal](http://www.linuxjournal.com/article/2156)





