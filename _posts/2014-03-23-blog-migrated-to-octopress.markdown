---
layout: post
slug: blog-migrated-to-octopress
title: "Blog Migrated to Octopress"
date: 2014-03-23 23:28:29 -0400
comments: true
categories: 
- General
- octopress
tags: octopress
---
After a long period of good intentions with no action, I finally moved the codingjunkie.net blog from Wordpress to Octopress. I have nothing bad to say about Wordpress, it's been a great tool for me. It's just that over time I've found myself wanting a simpler blogging platform.  Aside from liking the overall looks of Ocotpress, the big draw for me was the decreased load time from serving up static HTML pages.  Once I sat down and decided to pull the plug, it was really pretty simple. It all boiled down to just a few steps: 

 1. Export my posts into the Wordpress XML format.
 2. Run [exitwp](https://github.com/thomasf/exitwp "exitwp") on the exported XML.
 3. Some basic regex work to fix image tags.
 4. Configure the permalinks to match the form I already use.
 5. Spend 5 minutes reconfiguring my blog on the [WebFaction](https://www.webfaction.com/) control panel.
 6. Deploy the converted posts using rsync.

Overall, it went much easier than I had anticipated.  Only time will tell if switching to the new platform will help me write better content!

