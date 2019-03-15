---
layout: post
title:  "Code coverage in Scala"
date:   2014-08-14
categories: [development]
tags: [scala, codecoverage]
---
Most code coverage tools check lines of codes and branches (which part of the if/then/else is covered)

For scala this is not a very useful metric, i'll let the following example speak for itself

``` scala 
Players.filter(_.active == true).sortBy(- _.score).take(5)
   .foreach(p => println(s"${p.lastname} ${p.score}"))
```

in this case it is all on one (or two) line(s) and any branching (f.i. in the .filter()) is hidden inside the collection logic    
    
Therefor the best metric to use for scala is statement coverage, meaning the coverage tool will look at each statement, f.i. *_.active == true* and *- _.score* etc.

[scoverage](http://scoverage.org) is a [githbub project](http://github.com/scoverage) that provides statement and branch coverage for scala code, it comes with sbt, maven and gradle plugin and its reports can be used in sonar or coveralls

an example using sbt:

in my project/build.sbt I added

``` scala

resolvers += Classpaths.sbtPluginReleases

addSbtPlugin("org.scoverage" %% "sbt-scoverage" % "0.99.7.1")
```

and then in build.sbt

``` scala 
instrumentSettings
```
now you can run 

``` bash 
sbt scoverage:test
```    
    
data and reports are available in target/scala_2.10/scoverage* directories	

At my job we use Team City as our Continuous Integration tool and it only supports a couple of code coverage tools, so it means we can't fail a build when the coverage falls beneath a certain number.

What you could do is add the html report as artifact so at least people can look at the build and click a link to see the report
to do this add the following to your "publish artifacts" section of your build:

``` shell
target/scala_2.10/scoverage-reports/*.html
target/scala_2.10/scoverage-reports/**/*.html
```

To conclude code coverage is not an end-all-solution for code quality, but it surely helps.
And try to resist the urge to add unit-tests just to get the coverage up, think about the functionality that you are testing and what's missing there.



