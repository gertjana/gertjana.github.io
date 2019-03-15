---
layout: post
title:  "Mesos on a Raspberry Pi cluster part 2"
date:   2016-05-26
categories: [development]
tags: [mesos, cluster, raspberrypi]
---
#### Calculating Pi on Pi's
In the first part I managed to get mesos running on a cluster of raspberry pi's

now to have it do something... and what could be better then to have it calculate Pi in a distributed manner. 

As my main programming language is Scala, the examples will also be in Scala
The full code itself is in a [Github](https://github.com/gertjana/example-mesos-scheduler) repository

Mesos itself only does resource management, slaves offer their cpu/memory/disk to the master.
it needs frameworks to listen to these resource offers and schedule tasks on the slaves that consume these resources.
To build you own framework you need to implement two classes:

#### A Scheduler that receives offers and can launch tasks

``` scala
class ExampleScheduler(val executorInfo: ExecutorInfo) extends Scheduler {
  override def resourceOffers(driver: SchedulerDriver, offers: util.List[Offer]) {
    offers.foreach { offer =>
      //do something with the offer
    }
  }
}
object ExampleScheduler extends App {
  val scheduler = new ExampleScheduler(getExecutorInfo)
  val driver = new MesosSchedulerDriver(scheduler, getFrameworkInfo, mesosMaster)
  ...
}
```

#### An Executor that executes the task.

``` scala
class ExampleExecutor extends Executor {
  override def launchTask(driver: ExecutorDriver, task: TaskInfo) = {
    // execute the task and report status back to the scheduler
  }
}
object ExampleExecutor extends App {
  val driver = new MesosExecutorDriver(new ExampleExecutor)
  val status = driver.run()
}
```

#### Calculating Pi

One way of calculating pi is using the [Gregory–Leibniz Formula](https://en.wikipedia.org/wiki/Leibniz_formula_for_%CF%80)

In short this is an infinite series of additions and substractions:

``` bash
pi/4 = (1/1) - (1/3) + (1/5) - (1/7) + (1/9) - (1/11) .... etc
```

multiply-ing both sides with 4 gives me 

``` bash
pi = (4/1) - (4/3) + (4/5) ... etc
```

So very simply sending 1, -3, 5, -7 etc to the slaves, have them divide 4 by it and report back the result, I can in the scheduler add them all back together and end up with a close approximation of pi 
As additions are commutative the order in which i'm getting the results back is not important. so I don't need to do extra work 

#### Launching a Task with the number from the scheduler on an executor

There is a counter that is used for the task id, I'll use that to calculate the data I need to send to the executor
Inside the scheduler's resourceOffers method I put:

``` scala
var taskCounter = 0

override def resourceOffers(driver: SchedulerDriver, offers: util.List[Offer]) = {
  offers.foreach { offer =>
    taskCounter += 1
    val taskId = Protos.TaskID.newBuilder.setValue(taskCounter.toString)       
    
    val data = (taskCounter % 2) match {
      case 0 => BigDecimal(taskCounter*2+1)
      case 1 => BigDecimal(-(taskCounter*2+1))
    }
    val task = Protos.TaskInfo.newBuilder
      .setName("task " + taskId)
      .setTaskId(taskId)
      .setSlaveId(offer.getSlaveId)
      .addResources(buildResource("cpus", 1))
      .addResources(buildResource("mem", 128))
      .setData(ByteString.copyFromUtf8(data.toString))
      .setExecutor(Protos.ExecutorInfo.newBuilder(executorInfo))
      .build

    driver.launchTasks(Seq(offer), Seq(task))
  }
}

``` 

in the executor's launchTask method do the calculation and send it back and setting the task to finished:

``` scala
override def launchTask(driver: ExecutorDriver, task: TaskInfo) = {
  val result = BigDecimal(4) / BigDecimal(task.getData.toString("UTF-8"))

  driver.sendFrameworkMessage(ByteString.copyFromUtf8(result.toString).toByteArray)
  val status = Protos.TaskStatus.newBuilder
    .setTaskId(task.getTaskId)
    .setState(Protos.TaskState.TASK_FINISHED).build

  driver.sendStatusUpdate(status)
}
```

and collect the result

``` scala
var pi = 4

override def frameworkMessage(driver: SchedulerDriver, executorId: ExecutorID, slaveId: SlaveID, data: Array[Byte]) = {
    pi += BigDecimal(new String(data, "UTF-8"))
    println(s"Pi ~ $pi") 
  }
``` 

To Conclude:
I skipped over a lot of things that are necessary to have a fully functional mesos cluster with proper frameworks running on it, but I hope i've shown how a little bit how mesos works and how master and slaves communicate to schedule and run tasks on it
and all that on about a 100 euro hardware.

What's next?
I'm not sure, maybe get zookeeper running, and some other frameworks like marathon or chronos.

I also have an small lcd display lie-ing around, maybe attach that to the master pi and show the resources that are available/in use, that would be very cool.




