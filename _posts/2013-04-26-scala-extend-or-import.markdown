---
layout: post
title:  "Scala Extend or Import"
date:   2013-04-26
categories: [development]
tags: [scala, tip]
---
yesterday evening at the amsterdam.scala meetup I learned a neat trick, I also learned a lot more about type aliases and type classed but I have to get my head around those first.

The trick being not forcing users of your code to either extend or import, but allowing them to choose

``` scala
trait MyTrait {
    def myMethod = ..
  }
object MyTrait extends MyTrait
```

now my users can either extend their class or use an import