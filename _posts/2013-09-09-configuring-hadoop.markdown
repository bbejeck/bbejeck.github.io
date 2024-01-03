---
author: Bill Bejeck
comments: true
date: 2013-09-09 13:30:46+00:00
layout: post
slug: configuring-hadoop
title: Configuring Hadoop with Guava MapSplitters
wordpress_id: 2834
categories:
- Guava
- Hadoop
- java
- MapReduce
tags:
- Guava
- Hadoop
- java
- MapReduce
---

<img class="left" src="../assets/images/hadoop-logo.jpeg" /> In this post we are going to provide a new twist on passing configuration parameters to a Hadoop Mapper via the Context object. Typically, we set configuration parameters as key/value pairs on the Context object when starting a map-reduce job.  Then in the Mapper we use the key(s) to retrieve the value(s) to use for our configuration needs.  The twist is we will set a specially formatted string on the `Context` object and when retrieving the value in the Mapper, use a Guava [`MapSplitter` ](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Splitter.MapSplitter.html) to convert the formatted string into a `HashMap` that will be used for obtaining configuration parameters. We may be asking ourselves why go to this trouble? By doing configuration this way we are able to pass multiple parameters to a Mapper with a single key-value pair set on the Context object.  To illustrate one possible usage, we will revisit the [last post](http://codingjunkie.net/mapreduce-reduce-joins/), where we covered how to perform reduce-side joins.  There are two problems with the proposed solution in that post.  First, we are assuming the key to join on is always the first value in a delimited string from a file. Second, we assume that the same delimiter is used for each file.  What if we want to join data from files where the key is located in different locations per file and some files use different delimiters? Additionally we want to use the same delimiter (if any) for all of the data we output regardless of the delimiter used in any of the input files.  While this is admittedly a contrived situation, it will serve well for demonstration purposes.   First let's go over what the `MapSplitter` class is and how we can use it.
<!--more-->

## MapSplitter


The [`MapSplitter` ](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Splitter.MapSplitter.html) is a nested class in the [`Splitter`](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/base/Splitter.html) class.  The `Spitter` takes a String and splits it into parts with a given delimiter.  The MapSplitter goes a step further by creating a Map<String,String> from a string with key-value pairs separated with one delimiter and the pairs themselves separated with another delimiter altogether. Let's take a look at an example:

    
    
    Map<String,String> configParams = Splitter.splitOn("#")
                                       .withKeyValueSeparator("=")
                                       .split("6=June#7=July#8=August");
    


In the example above the String `"6=June#7=July#8=August"` would be converted into a Map with keys 6,7 and 8 mapped to June, July and August respectively. The `MapSplitter` is a very simple, but powerful class. Now we know how the `MapSplitter` works, let's take a look at how we can use it to help us set configuration parameters for our map-reduce jobs


## Configuration using the MapSplitter


Previously we set the index position of our join key and the delimiter to split on as the same for all files by setting the values in the `Context` object for the map-reduce job. Now we want to be able to set those on a per input file basis as needed.  We will still have default values as needed.  In order to accomplish this change we will create a properties file that will have the name of the file as the key and the value will be a string formatted to be used by the `MapSplitter`.  Our properties file will look like this:

    
    
    oneToManyEmployer2.txt=keyIndex=1&separator;=|
    oneToManyVehicles2.txt=keyIndex=1&separator;=#
    


Here we are indicating the the file oneToManyEmployer2.txt has our join key at index position 1 and the delimiter is a "|" pipe character and the oneToManyVehicles2.txt file has the join key at index position 1 and uses a "," comma as a delimiter.  We are going to make a few changes to our driver class.  First we are going to load the properties file (assuming we have placed the file in a directory located relative to where we invoke hadoop).

    
    
    InputStream inputStream = new FileInputStream(new File("./jobs/join-config.properties"));
    Properties properties = new Properties();
    properties.load(inputStream);
    


First we define a regular `Splitter` object that will split the file names on a forward slash '/'. Next as we loop through the file names we get the base name of the file by calling `Iterables.getLast` on the `Iterable` object returned from the `Splitter.split` method call. Then we attempt to retrieve a configured property string for each file in the `Properties.getProperty` method. Take note we also pass the `defaultMapConfig` variable that supplies default values if the property is not found for the file.  We've also added some additional configuration keys and values; the delimiter to use when joining values together and the join order for the file, which is determined by it's position in the arguments provided to the program.  Then we simply place the formatted string into the `Context` object using the file name as the key.

    
    
    String defaultMapConfig = "keyIndex=0&separator;=,";
    Splitter splitter = Splitter.on('/');
    for (int i = 0; i < args.length - 1; i++) {
        String fileName = Iterables.getLast(splitter.split(args[i]));
        String mapConfig = properties.getProperty(fileName, defaultMapConfig);
        builder.append(mapConfig).append("&joinDelimiter;=,&joinOrder;=").append(i + 1);
        config.set(fileName, builder.toString());
        builder.setLength(0);
        filePaths.append(args[i]).append(",");
    }
    




## Using the configuration values


To use our config values we will first have to retrieve the `HashMap` stored as a String that contains our configuration parameters

    
    
    private Splitter.MapSplitter mapSplitter = Splitter.on("&").withKeyValueSeparator("=");
    .......
    private Map<String,String> getConfigurationMap(Context context){
            FileSplit fileSplit = (FileSplit)context.getInputSplit();
            String configString = context.getConfiguration().get(fileSplit.getPath().getName());
            return mapSplitter.split(configString);
        }
    


Here we are using a `MapSplitter` instance variable and to create our `HashMap` by retrieving the formatted string stored in the `Context` with the file name used by this Mapper. Now we can simply pull the needed configuration parameters out from the map as shown here in the `setup` method:

    
    
    protected void setup(Context context) throws IOException, InterruptedException {
            Map<String,String> configMap = getConfigurationMap(context);
            keyIndex = Integer.parseInt(configMap.get("keyIndex"));
            String separator = configMap.get("separator");
            splitter = Splitter.on(separator).trimResults();
            String joinDelimiter = configMap.get("joinDelimiter");
            joiner = Joiner.on(joinDelimiter);
            joinOrder = Integer.parseInt(configMap.get("joinOrder"));
        }
    


The code in the `map` method remains the same as it was from our previous [post](http://codingjunkie.net/mapreduce-reduce-joins/)
Now we have completely configurable settings per file and we aren't restricted to having the join key in one position or having to use the same delimiter per file.  Of course this is just one example, but the approach outlined here could be used to configure many other settings and it only requires one key in the `Context` object.


## Results


Initially our data looked like this:
**oneToManyEmployer2.txt:**
`
Creative Wealth|cdd8dde3-0349-4f0d-b97a-7ae84b687f9c
Susie's Casuals|81a43486-07e1-4b92-b92b-03d0caa87b5f
Super Saver Foods|aef52cf1-f565-4124-bf18-47acdac47a0e
.....
`
**oneToManyVehicles2.txt:**
`
2003 Holden Cruze#cdd8dde3-0349-4f0d-b97a-7ae84b687f9c
2012 Volkswagen T5#81a43486-07e1-4b92-b92b-03d0caa87b5f
2009 Renault Trafic#aef52cf1-f565-4124-bf18-47acdac47a0e
.....
`
**singlePersonRecords.txt:**
`
cdd8dde3-0349-4f0d-b97a-7ae84b687f9c,Esther,Garner,4071 Haven Lane,Okemos,MI
81a43486-07e1-4b92-b92b-03d0caa87b5f,Timothy,Duncan,753 Stadium Drive,Taunton,MA
aef52cf1-f565-4124-bf18-47acdac47a0e,Brett,Ramsey,4985 Shinn Street,New York,NY
......
`
After running our map-reduce job our results look exactly like we want them:
`
08db7c55-22ae-4199-8826-c67a5689f838,John,Gregory,258 Khale Street,Florence,SC,2010 Nissan Titan,Ellman's Catalog Showrooms
0c521380-f868-438c-9916-4ab4ea76d316,Robert,Eversole,518 Stratford Court,Fayetteville,NC,2002 Toyota Highlander,Specialty Restaurant Group
1303e8a6-0085-45b1-8ea5-26c809635da1,Joe,Nagy,3438 Woodstock Drive,El Monte,CA,2011 Hyundai ix35,Eagle Food Centers
15360125-38d6-4f1e-a584-6ab9d1985ab8,Sherri,Hanks,4082 Old House Drive,Alexandria,OH,2003 Toyota Solara,Odyssey Records & Tapes
......
`


## Resources






  * [Data-Intensive Processing with MapReduce](http://www.amazon.com/Data-Intensive-Processing-MapReduce-Synthesis-Technologies/dp/1608453421) by Jimmy Lin and Chris Dyer


  * [Hadoop: The Definitive Guide](http://www.amazon.com/Hadoop-Definitive-Guide-Tom-White/dp/1449311520/ref=tmm_pap_title_0?ie=UTF8&qid=1347589052&sr=1-1) by Tom White


  * [Source Code and Tests](https://github.com/bbejeck/hadoop-algorithms) from blog


  * [Programming Hive](http://www.amazon.com/Programming-Hive-Edward-Capriolo/dp/1449319335) by Edward Capriolo, Dean Wampler and Jason Rutherglen


  * [Programming Pig](http://www.amazon.com/Programming-Pig-Alan-Gates/dp/1449302645) by Alan Gates


  * [Hadoop API](http://hadoop.apache.org/docs/r0.20.2/api/index.html)


  * [MRUnit](http://mrunit.apache.org/) for unit testing Apache Hadoop map reduce jobs


