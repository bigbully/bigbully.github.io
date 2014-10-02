akka2.3.6

在看akka源码时注意到一个很巧妙的设计，actorSystem关闭时，众多回调函数一并执行的操作。

akka使用闭锁和重入锁设计了一个TerminationCallbacks来完成这个工作：

	final class TerminationCallbacks extends Runnable with Awaitable[Unit] {
	  ...
	}

首先他实现了Runnable接口，可以在任何线程池中轻松的被执行。而且实现了Awaitable特质，需要实现ready和result两个方法，用来在等待结果时阻塞住。

	private val lock = new ReentrantGuard
    private var callbacks: List[Runnable] = _ //non-volatile since guarded by the lock
    lock withGuard { callbacks = Nil }
    
    private val latch = new CountDownLatch(1)
    
TerminationCallbacks内部使用了一个改良后的重入锁以及闭锁，用来做到线程安全。所有的回调函数都收集在callbacks中。

再来看一下如何添加回调函数：

	final def add(callback: Runnable): Unit = {
      latch.getCount match {
        case 0 ⇒ throw new RejectedExecutionException("Must be called prior to system shutdown.")
        case _ ⇒ lock withGuard {
          if (latch.getCount == 0) throw new RejectedExecutionException("Must be called prior to system shutdown.")
          else callbacks ::= callback
        }
      }
    }
    
这里使用了double-check保证所有回调函数能按顺序准确的保存下来。

当TerminationCallbacks最终被执行时，会调用run方法，这个方法本身放在重入锁中执行，内置了一个尾递归，按顺序依次执行回调函数集，最后闭锁countDown.

	final def run(): Unit = lock withGuard {
      @tailrec def runNext(c: List[Runnable]): List[Runnable] = c match {
        case Nil ⇒ Nil
        case callback :: rest ⇒
          try callback.run() catch { case NonFatal(e) ⇒ log.error(e, "Failed to run termination callback, due to [{}]", e.getMessage) }
          runNext(rest)
      }
      try { callbacks = runNext(callbacks) } finally latch.countDown()
    }
    
实现的ready和result的方法可以给回调函数集的执行设置等待时间，这里是拿闭锁的await方法实现的：

	final def ready(atMost: Duration)(implicit permit: CanAwait): this.type = {
      if (atMost.isFinite()) {
        if (!latch.await(atMost.length, atMost.unit))
          throw new TimeoutException("Await termination timed out after [%s]" format (atMost.toString))
      } else latch.await()

      this
    }

    final def result(atMost: Duration)(implicit permit: CanAwait): Unit = ready(atMost)
    
这里值得注意的是ready，和result方法不应该被直接调用。需要使用Await.ready或Await.result，但如何保证他不被直接调用呢，这里使用隐式参数玩了个小技巧。因为只有Await对象内才有CanAwait类型的参数，直接调用的话会由于未在上下文中找到隐式参数而无法通过编译。    
    
最后提供一个判断状态的api：

	final def isTerminated: Boolean = latch.getCount == 0
	
基本上是回调函数集被一并执行的最佳实践了。用scala写出来就是这么赏心悦目。