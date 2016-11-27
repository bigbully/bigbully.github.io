---
layout: post
title: Akka中的Scheduler
description: 
category: scala
---

LightArrayRevolverScheduler是Akka内部使用的默认的Scheduler，用来处理延迟或定时任务。

我在这篇文章中想详细讨论一下，LightArrayRevolverScheduler和java并发包中的ScheduledThreadPoolExecutor有什么不同。

LightArrayRevolverScheduler
---------------------------

顾名思义，LightArrayRevolverScheduler内部有一个数组，把所有的任务放在若干个bucket中，每个bucket中的任务会在同一时刻依次执行，bucket的数量及轮询bucket的时间间隔通过ticksPerWheel和tickDuration两个参数设定。

	def schedule(initialDelay: FiniteDuration,
                        delay: FiniteDuration,
                        runnable: Runnable)(implicit executor: ExecutionContext): Cancellable

schedule方法和所有定时执行任务的方法一样，这个方法将会创建一个TimerTask，用来执行定时任务。

如图所示，TimerTask通过内部的定时器，根据所传参数定期创建一个任务对象（TaskHolder），并把它放入Queue中，LightArrayRevolverScheduler内部的TimerThread便会消费Queue中的任务，把它放入不同的bucket中，由用户指定的线程池来执行bucket中的任务。

我想，这里有必要提到TimerTask是如何实现内部定时器的：

	private val InitialRepeatMarker: Cancellable = new Cancellable {
	  def cancel(): Boolean = false
	  def isCancelled: Boolean = false
	}

	try {
      new AtomicReference[Cancellable](InitialRepeatMarker) with Cancellable { self ⇒

        val timerTask: TimerTask = schedule(
          executor,
          new AtomicLong(clock() + initialDelay.toNanos) with Runnable {
            override def run(): Unit = {
              try {
                runnable.run()
                val driftNanos = clock() - getAndAdd(delay.toNanos)
                if (self.get != null)
                  swap(schedule(executor, this, Duration.fromNanos(Math.max(delay.toNanos - driftNanos, 1))))
              } catch {
                case _: SchedulerException ⇒ // ignore failure to enqueue or terminated target actor
              }
            }
          }, roundUp(initialDelay))
          
        compareAndSet(InitialRepeatMarker, timerTask)

      }
    }catch {
      case SchedulerException(msg) ⇒ throw new IllegalStateException(msg)
    }

InitialRepeatMarker是一个随着LightArrayRevolverScheduler创建被初始化好的占位符，schedule直接创建一个混入了Cancellable特质的AtomicReference对象，以此开辟一块内存空间，并使用InitialRepeatMarker占位，随后创建真正的TimerTask，通过CAS替换InitialRepeatMarker占位符。

这个混入了Cancellable特质的AtomicReference会返回给调用方，用来在必要时取消定时任务。

同时创建了一个实现Runnable接口的AtomicLong，	用来保存每次执行的时间。每执行一次任务都会创建一个TimerTask对象，并在上一次任务执行完之后，通过swap方法用新的TimerTask对象替换旧的。

创建TimerTask的同名schedule方法操作很简单，优先考虑了立刻执行的任务、Scheduler被停止的情况。除此之外只是创建TimerTask对象并放入Queue中。

	private def schedule(ec: ExecutionContext, r: Runnable, delay: FiniteDuration): TimerTask =
    if (delay <= Duration.Zero) {
      if (stopped.get != null) throw new SchedulerException("cannot enqueue after timer shutdown")
      ec.execute(r)
      NotCancellable
    } else if (stopped.get != null) {
      throw new SchedulerException("cannot enqueue after timer shutdown")
    } else {
      val delayNanos = delay.toNanos
      checkMaxDelay(delayNanos)

      val ticks = (delayNanos / tickNanos).toInt

      val task = new TaskHolder(r, ticks, ec)
      queue.add(task)
      if (stopped.get != null && task.cancel())
        throw new SchedulerException("cannot enqueue after timer shutdown")
      task
    }
需要提到的是这里使用的Queue是基于Golang核心开发成员Dmitriy Vyukov编写的MPSC(multiple producers single consumer) lock-free队列的Java版本，主要适用于Akka内部多生产者单消费者场景。

消费者线程是在LightArrayRevolverScheduler创建的时候启动的，先来看一下消费者线程中checkQueue函数：

	@tailrec
    private def checkQueue(time: Long): Unit = queue.pollNode() match {
      case null ⇒ ()
      case node ⇒
        node.value.ticks match {
          case 0 ⇒ node.value.executeTask()
          case ticks ⇒
            val futureTick = ((
              time - start + // calculate the nanos since timer start
                (ticks * tickNanos) + // adding the desired delay
                tickNanos - 1 // rounding up
              ) / tickNanos).toInt // and converting to slot number
          // tick is an Int that will wrap around, but toInt of futureTick gives us modulo operations
          // and the difference (offset) will be correct in any case
          val offset = futureTick - tick
            val bucket = futureTick & wheelMask
            node.value.ticks = offset
            wheel(bucket).addNode(node)
        }
        checkQueue(time)
    }

checkQueue函数是个尾递归函数，它会尝试获取所有队列中的任务，知道队列空。获得的任务如果需要马上执行，则交给用户线程池处理，否则计算出bucket index，然后放入bucket中。每个bucket中都放置一个队列用来保证任务执行的顺序。

nextTick函数是消费者线程执行的主逻辑，它会从Queue中取出任务，如果任务到期则立即使用用户线程池执行，如果未到期，则放入bucket中。如果发现Scheduler已经停止，则收集所有待处理的任务，依次cancel。

	@tailrec final def nextTick(): Unit = {
      val time = clock()
      val sleepTime = start + (tick * tickNanos) - time

      if (sleepTime > 0) {
        // check the queue before taking a nap
        checkQueue(time)
        waitNanos(sleepTime)
      } else {
        val bucket = tick & wheelMask
        val tasks = wheel(bucket)
        val putBack = new TaskQueue

        @tailrec def executeBucket(): Unit = tasks.pollNode() match {
          case null ⇒ ()
          case node ⇒
            val task = node.value
            if (!task.isCancelled) {
              if (task.ticks >= WheelSize) {
                task.ticks -= WheelSize
                putBack.addNode(node)
              } else task.executeTask()
            }
            executeBucket()
        }
        executeBucket()
        wheel(bucket) = putBack

        tick += 1
      }
      stopped.get match {
        case null ⇒ nextTick()
        case p ⇒
          assert(stopped.compareAndSet(p, Promise successful Nil), "Stop signal violated in LARS")
          p success clearAll()
      }
    }

LightArrayRevolverScheduler的主逻辑就是这些。

最后来看一下Scheduler的close方法：

	override def close(): Unit = Await.result(stop(), getShutdownTimeout) foreach {
    task ⇒
      try task.run() catch {
        case e: InterruptedException ⇒ throw e
        case _: SchedulerException   ⇒ // ignore terminated actors
        case NonFatal(e)             ⇒ println("exception while executing timer task:" + e.getMessage)
      }
	}

这个方法会最多阻塞getShutdownTimeout的时间，收集全部待处理的任务，如果没来得及收集完会抛超时异常。这里需要注意的是，所有收集到的待处理任务会在这个方法中被依次执行，而不是取消。

	private val stopped = new AtomicReference[Promise[immutable.Seq[TimerTask]]]
    private def stop(): Future[immutable.Seq[TimerTask]] = {
      val p = Promise[immutable.Seq[TimerTask]]()
      if (stopped.compareAndSet(null, p)) {
        // Interrupting the timer thread to make it shut down faster is not good since
        // it could be in the middle of executing the scheduled tasks, which might not
        // respond well to being interrupted.
        // Instead we just wait one more tick for it to finish.
        p.future
      } else Future.successful(Nil)
    }

stop方法使用了scala中的Promise，stopped用来保存这个Promise引用，而上文提到的消费者线程当检测到Schedular被停止后，就负责装填未完成的任务到Promise对应的任务列表中。

而想要了解java并发包中的ScheduledThreadPoolExecutor，就不得不首先了解一下线程池的实现，具体可以看我之前写过的一篇文章[我所知道的线程池](http://bigbully.github.io/%E7%BA%BF%E7%A8%8B%E6%B1%A0)。
概括一下，ScheduledThreadPoolExecutor和LightArrayRevolverScheduler使用的队列差别很大：

 1. ScheduledThreadPoolExecutor使用了延迟队列，而延迟队列的实现方式是堆；LightArrayRevolverScheduler使用的是FIFO队列。
 2. ScheduledThreadPoolExecutor继承自ThreadPoolExecutor，归根结底还是线程池；LightArrayRevolverScheduler是Scheduler，由于采用批量处理的设计，而且遵循scala语言的要义，使用很多常量，降低了并发编程复杂度。


