---
layout: post
title: LiveListenerBus源码阅读
description: 
category: scala
---

非常幸运在公司争取到一个阅读spark源码的机会，也就是利用工作时间阅读spark源码。大致看了看，觉得spark源码的可读性很高，注释写的很详细，而且用了很多scala的特性，这点上比kafka强了不少。

spark中得LiveListenerBus是一个简单的异步处理事件的类，因为在做消息中间件的时候也遇到很多需要通过BlockQueue做异步处理的逻辑，这次看了LiveListenerBus的源码之后很有启发，因为源码中对开启和关闭逻辑细节方面考虑很到位。

作为SparkContext在初始化的时候会创建这个LiveListenerBus

	// An asynchronous listener bus for Spark events
	private[spark] val listenerBus = new LiveListenerBus
	

LiveListenerBus实例的成员变量有以下几个：

	private val EVENT_QUEUE_CAPACITY = 10000 //队列的长度
	private val eventQueue = new LinkedBlockingQueue[SparkListenerEvent](EVENT_QUEUE_CAPACITY) //使用基于链表阻塞队列，为什么不使用BlockingArrayQueue我会单独写跟java并发类相关的笔记来进行分析
	private var queueFullErrorMessageLogged = false//当队列满了是否打印日志
	private var started = false//启动成功标示

	private val eventLock = new Semaphore(0)//这个信号量是为了避免消费者线程空跑
	
	
接下来是处理队列的消费者线程

	private val listenerThread = new Thread("SparkListenerBus") {
	  setDaemon(true)//线程本身设为守护线程
	  override def run(): Unit = Utils.logUncaughtExceptions {//工具类，可以记录方法中没有catch的异常
	    while (true) {
    	  eventLock.acquire() //这个是避免消费者线程空跑的办法，尝试获取这个信号量，当前信号量为0，则阻塞
          // Atomically remove and process this event
	      LiveListenerBus.this.synchronized {//同步的获取队列中的元素
    	    val event = eventQueue.poll
	        if (event == SparkListenerShutdown) {//这里使用毒丸使消费者线程优雅退出
    	      // Get out of the while loop and shutdown the daemon thread
	          return
    	    }
	        Option(event).foreach(postToAll)//event有可能为null，为了避免发生空指针的情况（虽然已从逻辑上避免)，但省去了进行非空判断，所以这里用Option对event进行封装，我们知道只有Some(value)才会执行foreach方法
          }
        }
      }
    }

    
    def postToAll(event: SparkListenerEvent) {...}//postToAll方法是对事件的后续处理，在这里不详尽展开了
    
下面再来看看如何往队列中放入待处理的元素：

	def post(event: SparkListenerEvent) {
      val eventAdded = eventQueue.offer(event)
	  if (eventAdded) {//如果成功加入队列，则在信号量中加一，与之对应，消费者线程就可以消费这个元素
	    eventLock.release()
	  } else {//如果队列已满，打出队列满的异常信息
	    logQueueFullErrorMessage()
	  }	
	}
	
接下来看看如何优雅关闭：

	def stop() {
	  if (!started) {
    	throw new IllegalStateException("Attempted to stop a listener bus that has not yet started!")
	  }
	  post(SparkListenerShutdown)//这里放入毒丸
	  listenerThread.join()//然后等待消费者线程自动关闭
	}
	
从代码中可以看出，stop方法是个阻塞方法，当LiveListenerBus中的消费者线程消费完队列中的所有元素之前，都会一直阻塞，知道消费完毕，优雅退出。


