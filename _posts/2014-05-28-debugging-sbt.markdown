---
layout: post
title:  "Debugging SBT"
date:   2014-05-28
categories: [development]
tags: [sbt, debug]
---
I found that the simplest way to debugging sbt project’s is to 
add a standard remote debugging configuration to (in my case) intellij and remember the port the debugger is listening for (default 5005) 
and then in a terminal window start sbt as follows:

``` bash 
sbt -jvm-debug 5005
```

If you open the terminal window in Intellij (alt-F12) you can keep it all in one IDE
