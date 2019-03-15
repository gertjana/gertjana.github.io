As most developers are also DevOps nowadays, we're not only writing code, but are also at least partly responsible for getting/keeping it running.

That often means writing scripts that have to run on servers.

The SBT install when done with [Conscript](https://github.com/n8han/conscript) adds two scripts that make maintaining servers with scala a breeze
besides SBT it installs a `scalas` and a `screpl` script
the latter basically runs `sbt console` giving you access to the REPL. the scalas takes a file as argument and then compiles and runs it

to get this running you need to install conscript first and then install sbt by doing `cs install sbt --branch 0.13.5`
this will install sbt and the scalas and screpl script to `~/bin` make sure this is added to you path

Then create a test.scala file with the following:

<!-- language: scala -->
    #!/usr/bin/env scalas
    
    /***
    scalaVersion := "2.11.1"
    */
    
    println("hello")

then make it executable by doing `chmod +x test.scala`
and now you can run it by just doing `./test.scala`

the lines between the comments `/**** ... */` are settings that you'd normally put in your build.sbt project so for instance any libraries you need to import just do a `libraryDependencies ++= Seq(...)` 

for more info read the sbt documentation:
http://www.scala-sbt.org/0.13/docs/Scripts.html