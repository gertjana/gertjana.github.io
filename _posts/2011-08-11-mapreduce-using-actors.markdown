---
layout: post
title:  "Mapreduce using actors"
date:   2011-08-11
categories: [development]
tags: [akka, actor, mapreduce]
---
In this blog post I would like to do a basic example on how to do map reduce using actors

According to [Wikipedia](http://en.wikipedia.org/wiki/Actor_model)

In computer science, the Actor model is a mathematical model of concurrent computation that treats "actors" as the universal primitives of concurrent digital computation: in response to a message that it receives, an actor can make local decisions, create more actors, send more messages, and determine how to respond to the next message received.

The Actor model adopts the philosophy that everything is an actor. This is similar to the everything is an object philosophy used by some object-oriented programming languages, but differs in that object-oriented software is typically executed sequentially, while the Actor model is inherently concurrent.
An actor is a computational entity that, in response to a message it receives, can concurrently:
<ul>
	<li>send a finite number of messages to other actors;</li>
	<li>create a finite number of new actors;</li>
	<li>designate the behavior to be used for the next message it receives.</li>
</ul>
There is no assumed sequence to the above actions and they could be carried out in parallel.

A number of languages have the actor model implemented in their core (Io, Erlang, Scala), for most other languages it's available as a library (Akka for Java, Retlang for .Net, Haskell-Actor for Haskell, Parely and Pykka for Python)

In this example I'll be using the <a href="http://www.akka.io">Akka</a> framework for Java and Scala, which brings actors to the enterprise level by adding for instance fault tolerance (supervisors, let it crash semantics) and remote actors, If you want to play around with this yourself the best place to start is to use the <a href="http://typesafe.com/">typesafe stack</a> which includes Akka and Scala

I'm using a typical map/reduce use case, namely counting words in a document
I do this by having a 'master' actor that divides the work in lines, have a number of 'worker' actors that count the words for a single line, replying the result to the 'master' to aggregate

Messages that actors send to each other are simply objects (case classes, the receive method typically uses pattern matching to determine which message has arrived), multiple messages to an actor will be queued, and handled sequentially

So let's create the message objects

``` scala
sealed trait MapReduceMessage
case class CountDocument(document: Iterator[String]) extends MapReduceMessage
case class CountLine(line: String) extends MapReduceMessage
case class Result(values: Map[String, Int]) extends MapReduceMessage
```

The <strong>CountDocument </strong>which will be send to the master, the <strong>CountLine </strong>to each worker, and each worker will return a <strong>Result </strong> when done

as each receive method returns a Partial Function[T] which allows for several classes to handle all the messages, the sealed trait super type will actually generate a warning when not all messages are handled. neat trick that.
It will also make it impossible to add messages outside of your class :) which could be something you want. or not. 

Now on to the Master


``` scala
  // Master Actor, creates Worker Actors, distributes work and gathers results
  class Master(nrOfWorkers: Int, latch:CountDownLatch) extends Actor {

    val workers = Vector.fill(nrOfWorkers)(actorOf[CountLineWorker].start());
    val router = Routing.loadBalancerActor(CyclicIterator(workers)).start();

    val resultMap = new HashMap[String, Int]();

    var start : Long = _
    var count : Long = 0

    def receive = {
      case CountDocument(lines : Iterator[String]) =&gt;

        lines.foreach(line =&gt;
              if (!line.isEmpty) {
                count = count+1;
                router ! CountLine(line)
              })

        //shutdown actors
        router ! Broadcast(PoisonPill)
        router ! PoisonPill

      case Result(values: Map[String, Int]) =&gt;

        for ((key, value) &lt;- values) {
          resultMap.put(key, resultMap.getOrElse(key, 0)+value)
        }
        count = count - 1;
        if (count &lt;= 0) self.stop()
    }

    override def preStart() {
      start = System.currentTimeMillis
    }

    override def postStop() {
      val end = System.currentTimeMillis-start
      println("Result after %s ms :".format(end))
      for((key, value) &lt;- resultMap.toList.sortBy(_._2).reverse) {
        println("%s: %s".format(value, key))
      }
      latch.countDown()
    }
  }
```

An actor class needs to implement the Actor interface which has one method called receive.

The  Routing.LoadbalancingActor is a built in actor that forwards messages send to it according to the scheme passed to it, in our case in a round robin style.
When receiving a CountDocument messages it will send each line to a worker, it will then tell the workers and the router to shutdown by sending them a PosionPill message.
When receiving a Result message it will aggregate the result and after receiving all results it will stop itself.
we override the preStart() and postStop() methods to allow us to calculate spend time and reporting the result

the ! in
``` scala 
router ! CountLine(line)
```

is a so called bang method, meaning fire and forget, there are also methods that will reply eventually (!!) and reply eventually with a Future (!!!)

The Worker looks as follows:

``` scala 
  //Actor that counts the words for a single line
  class CountLineWorker extends Actor {

    def receive = {
       case CountLine(line) =&gt;
         self reply Result(countWords(line))
    }

    def countWords(line: String):Map[String, Int] = {
      val result = new HashMap[String, Int]

      "[^A-Za-z0-9u0020]".r.replaceAllIn(line, "")
            .split(" ")
            .foreach(word =&gt; {
              result.put(word, result.getOrElse(word, 0)+1)
            })

      result
    }
  }
```

Should speak for itself, sanitizes and then count the words in a line, and replies to the sending actor with the result.

now wrap it in a main class that loads the document and sends it off to the master

``` scala 
object MapReduce extends App {

  countWordsInFile("src/main/resources/test.txt", 6);

  def countWordsInFile(fileName: String, nrOfWorkers: Int) {

    val source = scala.io.Source.fromFile(fileName)
    val document = source.getLines()

    val latch = new CountDownLatch(1)

    val master = actorOf(new Master(nrOfWorkers, latch)).start()

    master ! CountDocument(document)

    latch.await()

    source.close()
  }
}
```

This will load the document and sends it off to the master also instructing it to create 6 workers, as actors are called asynchronously, the Countdown latch is there to prevent the program from stopping before all the work is done.

The full source code is available as a <a href="https://gist.github.com/1123718">gist </a> on github

A couple of other interesting Akka features are

<strong>Become</strong>:
you can tell each actor to become another, this allows for hot code swapping, which is essential in fault-tolerant, always up systems.

``` scala
def receive = {
  case updateYourSelf(newMe:ActorRef) => {
    become newMe
  }
}
```
  
<strong>Typed Actors</strong> (example blatantly stolen from Akka documentation):
extending any POJO with TypedActor will turn them into actors
your class need to have a separate interface/implementation for this to work

``` scala 
trait RegistrationService {
  def register(user: User, cred: Credentials): Unit
  def getUserFor(username: String): User
}
public class RegistrationServiceImpl extends TypedActor with RegistrationService {
  def register(user: User, cred: Credentials): Unit = {
    ... // register user
  }

  def getUserFor(username: String): User = {
    ... // fetch user by username
   user
  }
}

val service = TypedActor.newInstance(classOf[RegistrationService], classOf[RegistrationServiceImpl], 1000)
//last parameter is timeout for Futures
```

<strong>Remote Actors</strong> (example blatantly stolen from the Akka homepage):

``` scala 
// server code
class HelloWorldActor extends Actor {
  def receive = {
    case msg => self reply (msg + " World")
  }
}
remote.start("localhost", 9999).register(
  "hello-service", actorOf[HelloWorldActor])

// client code
val actor = remote.actorFor(
  "hello-service", "localhost", 9999)
val result = actor !! "Hello"
```