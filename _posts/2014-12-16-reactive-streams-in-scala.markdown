---
layout: post
title:  "Reactive streams in scala"
date:   2014-12-16
categories: [development]
tags: [scala, reactive]
---
In this world of multicore-cpu's we need to change the way we write software to make fully use of those resources. "reactive streams" are a way to do this:

Normally streams have two problems:

* if the producer is fast and the consumer is slow, items need to be buffered and eventually the system will run out of memory. 
* if the consumer is fast and the producer is slow, the consumer will block waiting for the next item 

Reactive streams solve this by apply-ing backpressure:
A reactive stream consists of one or more chains of Producers and Consumers divided by async boundaries.
In a reactive stream it is always the Consumer that initiates data transfer by telling the Producer how much items it can handle. 
![](/content/images/2014/Dec/reactive-streams--1-.png)

In the picture above it all starts with Step 3 telling Step 2 that it can handle x items, Step 2 will then send upto x items of data to step 3 but no more than x items. once step 3 processed the items it will ask for more.
The same thing applies for the communication between step 1 and 2. so we always have a 'demand' stream going from right to left and then a 'data' stream going from left to right.


The http://www.reactive-streams.org/ website is an initiative by some large companies to as they say themselves:

>Reactive Streams is an initiative to provide a standard for asynchronous stream processing with non-blocking back pressure on the JVM.

They provide a simple java [api](https://github.com/reactive-streams/reactive-streams/tree/v1.0.0.M3/api/src/main/java/org/reactivestreams) which the team at Akka turned into a nice simple DSL for scala developers

It consists basically of 3 objects

* `Source[T]` which wraps the Publisher
* `Sink[T]` which wraps the Subscriber
* `Flow[U,T]` which is both a Source and Sink

A very simple implementation would look something like this

``` scala
import akka.stream.scaladsl._
import akka.actor.ActorSystem
import akka.stream.FlowMaterializer

implicit val system = ActorSystem("StreamTest")
implicit val ec =  system.dispatcher
implicit val materializer = FlowMaterializer()

val source:Source[Int] = Source(Stream.from(1))
val sink:Sink[Int] = ForeachSink[Int](i => println(i))

def isPrime(n: Int) = (2 until n) forall (n % _ != 0)

source
  .filter(i => isPrime(i))
  .to(sink)
  .run()
```

In here we create a `Source` from a normal scala stream
and a `ForEachSink` that will execute the function passed to it on every item, in our case just print it out. the last for lines composes the stream processing, so we're getting nr in from the source, only keep the prime numbers and printing those to the console.
Important here is to note that nothing happens until you call `run()` the type before calling `run()` is `RunnableFlow` which means it has composed the entire flow and is ready to run it.

For a more contrived example, imagine we have an Actor System and we want to process incoming messages as a stream. 
For this i've created a TickActor, which will send itself a message every second and then turn that Actor into a `Source` by using the `ActorPublisher[T]` trait.

``` scala 
case class Tick()

class TickActor extends ActorPublisher[Int] {
  import scala.concurrent.duration._

  implicit val ec = context.dispatcher

  val tick = context.system.scheduler.schedule(1 second, 1 second, self, Tick())

  var cnt = 0
  var buffer = Vector.empty[Int]

  override def receive: Receive = {
    case Tick() => {
      cnt = cnt + 1
      if (buffer.isEmpty && totalDemand > 0) {
        onNext(cnt)
      }
      else {
        buffer :+= cnt
        if (totalDemand > 0) {
          val (use,keep) = buffer.splitAt(totalDemand.toInt)
          buffer = keep
          use foreach onNext
        }
      }
    }
  }

  override def postStop() = tick.cancel()
}
```

As can see for each `Tick` message the actor receives it tries to send the current count to the stream.
Now the rule is you can only send items to the stream with `onNext(element: T)` when there is demand `totalDemand > 0`
So I'm keeping a buffer to store incoming items. Ideally you would solve this by apply-ing backpressure to the systems sending this messages.
If the buffer is empty and there is a demand i'm sending the count directly, otherwise i'm adding the item to the buffer and then take as much items out again as there is demand and send them.
The stream flow will look like this:

``` scala 
implicit val system = ActorSystem("StreamTest")
implicit val ec =  system.dispatcher
implicit val materializer = FlowMaterializer()

val tickActor = system.actorOf(Props[TickActor])

val source:Source[Int] = Source(ActorPublisher[Int](tickActor))
val sink:Sink[Int] = ForeachSink[Int](println)

source
  .map(i => i * 2)
  .to(sink)
  .run()
```

In this case we are using a `.map()` to transform items in the stream to their double value.

In you want to consume a stream and process it further on as actor messages have a look at the `ActorSubscriber[T]` trait

This will run in somewhat constant memory regardless of the size of the stream. combining with the fact that the underlying implementation are Akka Actors, this will make optimal use of the computer resources at hand.
You could scale out by using broadcasts or split the stream in multiple flows and then merging them at the end. and now no more running your server at 10% load because you need to handle that occasional spike.

This is all still in development and not final, if you want to play around with it add this to your build.sbt

``` scala 
libraryDependencies += "com.typesafe.akka" %% "akka-stream-experimental" % "1.0-M1"
```

The example code is in a [gist](https://gist.github.com/gertjana/398350eb91f4cd999d1f)