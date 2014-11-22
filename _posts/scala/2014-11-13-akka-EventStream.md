---
layout: post
title: akka-eventStream笔记
description: 
category: scala
---

还记得在初始化ActorSystem时要进行EventStream的初始化吗？

	class ActorSystemImpl...
	
	val eventStream: EventStream = new EventStream(DebugEventStream)
	eventStream.startStdoutLogger(settings)


EventStream的层级结构：EventStream -> LoggingBus -> ActorEventBus -> EventBus 可以看到ActorSystem中的EventStream是一个基于事件机制的事件流的组件，主要用来处理日志信息，但也可以用来处理一些系统级的事件。因为是生产者消费者模式，所以事件的产生和收集都是异步进行的。

EventStream初始化完成后会首先启动StdoutLogger，StdoutLogger顾名思义是把log日志在控制台上输出的logger，适用于未接入任何日志系统的情况。

	trait LoggingBus...
	
	private[akka] def startStdoutLogger(config: Settings) {
      setUpStdoutLogger(config)
      publish(Debug(simpleName(this), this.getClass, "StandardOutLogger started"))
    }

	private def setUpStdoutLogger(config: Settings) {
      val level = levelFor(config.StdoutLogLevel) getOrElse {
        // only log initialization errors directly with StandardOutLogger.print
        StandardOutLogger.print(Error(new LoggerException, simpleName(this), this.getClass, "unknown akka.stdout-loglevel " + config.StdoutLogLevel))
        ErrorLevel
      }
      AllLogLevels filter (level >= _) foreach (l ⇒ subscribe(StandardOutLogger, classFor(l)))
      guard.withGuard {
        loggers :+= StandardOutLogger
        _logLevel = level
      }
    }

在初始化StdoutLogger会首先根据配置，获得StdoutLogger的log级别，根据log级别订阅相应的日志。
具体如何订阅的，我会在之后进行详细分析。
订阅完成后，会把StdoutLogger加入到loggers集合中，loggers代表了ActorSystem系统中所有日志的接收方。

接下来在初始化provider时会执行：

	eventStream.startDefaultLoggers(_system)

这是一个比较长的方法，我来逐句分析一下：

	trait LoggingBus@startDefaultLoggers...
	
	val logName = simpleName(this) + "(" + system + ")"
    val level = levelFor(system.settings.LogLevel) getOrElse {
      // only log initialization errors directly with StandardOutLogger.print
      StandardOutLogger.print(Error(new LoggerException, logName, this.getClass, "unknown akka.loglevel " + system.settings.LogLevel))
      ErrorLevel
    }

首先会从系统配置akka.loglevel中获取LogLevel。

	trait LoggingBus@startDefaultLoggers...

	val defaultLoggers = system.settings.Loggers match {
      case Nil     ⇒ classOf[DefaultLogger].getName :: Nil
      case loggers ⇒ loggers
    }

之后从系统配置akka.stdout-loglevel中获得日志信息的接收类，这里可以选择akka-slf4j包中的akka.event.slf4j.Slf4jLogger，这样ActorSystem中产生的日志会接入slf4j日志系统，之后是用log4j还是logback实现就看个人喜好了。当然也可以自行扩展，不过需要注意的是日志信息的接受类必须继承Actor类，因为所有日志信息都是以事件的方式发送的。

	trait LoggingBus@startDefaultLoggers...

	val myloggers =
        for {
          loggerName ← defaultLoggers
          if loggerName != StandardOutLogger.getClass.getName
        } yield {
          system.dynamicAccess.getClassFor[Actor](loggerName).map({
            case actorClass ⇒ addLogger(system, actorClass, level, logName)
          }).recover({
            case e ⇒ throw new ConfigurationException(
              "Logger specified in config can't be loaded [" + loggerName +
                "] due to [" + e.toString + "]", e)
          }).get
        }
	guard.withGuard {
        loggers = myloggers
        _logLevel = level
      }

这里用到了dynamicAccess的getClssFor方法，意图返回一个Actor的子类。返回值用[Try](http://bigbully.github.io/Try/)包装，这样可以把异常处理延后进行。如果可以获取到class，则通过addLogger方法加载这个日志接受类，如果Try中包含的是异常信息，则是用recover方法进行异常信息的处理。之后在锁内更新loggers和_logLevel。
		
	trait LoggingBus...	

	private def addLogger(system: ActorSystemImpl, clazz: Class[_ <: Actor], level: LogLevel, logName: String): ActorRef = {
      val name = "log" + Extension(system).id() + "-" + simpleName(clazz)
      val actor = system.systemActorOf(Props(clazz), name)
      implicit def timeout = system.settings.LoggerStartTimeout
      import akka.pattern.ask
      val response = try Await.result(actor ? InitializeLogger(this), timeout.duration) catch {
        case _: TimeoutException ⇒
          publish(Warning(logName, this.getClass, "Logger " + name + " did not respond within " + timeout + " to InitializeLogger(bus)"))
          "[TIMEOUT]"
      }
      if (response != LoggerInitialized)
        throw new LoggerInitializationException("Logger " + name + " did not respond with LoggerInitialized, sent instead " + response)
      AllLogLevels filter (level >= _) foreach (l ⇒ subscribe(actor, classFor(l)))
      publish(Debug(logName, this.getClass, "logger " + name + " started"))
      actor
    }

addLogger方法会把日志收集类作为一个SystemActor挂到ActorSystem中，并且以ask的方式向这个actor发送一个初始化指令。如果出现超时会通过publish发布一个异常信息到EventStream中。最后为日志收集actor按照日志级别订阅相关日志。

	trait LoggingBus@startDefaultLoggers...
	
	try {
        if (system.settings.DebugUnhandledMessage)
          subscribe(system.systemActorOf(Props(new Actor {
            def receive = {
              case UnhandledMessage(msg, sender, rcp) ⇒
                publish(Debug(rcp.path.toString, rcp.getClass, "unhandled message from " + sender + ": " + msg))
            }
          }), "UnhandledMessageForwarder"), classOf[UnhandledMessage])
      } catch {
        case _: InvalidActorNameException ⇒ // ignore if it is already running
      }

在这里会根据配置信息akka.actor.debug.unhandled决定是否创建一个SystemActor，来接受并转手发布Debug级别的事件来记录UnhandledMessage。UnhandledMessage用来指发送到某个Actor的消息但是Actor并没有在receive方法中设置接收这种类型的消息。

publish，subscribe和unsubscirbe的具体实现在[SubchannelClassification](http://bigbully.github.io/akka-SubchannelClassification/)中有详细分析。

EventStream的所有Subscriber都是ActorRef，而Event则是Logging.LogEvent。EventStream的publish操作如下：

	protected def publish(event: AnyRef, subscriber: ActorRef) = {
	  if (subscriber.isTerminated) unsubscribe(subscriber)
      else subscriber ! event
    }

只不过是向Subscriber发送一条消息而已，非常简单。

那么怎么才能触发publish操作呢，参考下面这个Actor：

	class MyAct extends Actor with ActorLogging{

	  override def receive: Actor.Receive = {
	    case "START" => log.info("start") 
	  }
	}

任何一个Actor都可以混入ActorLogging特质，之后就可以调用ActorLogging中的log方法。


	trait ActorLogging { this: Actor ⇒
	  private var _log: LoggingAdapter = _

	  def log: LoggingAdapter = {
	    // only used in Actor, i.e. thread safe
	    if (_log eq null)
	      _log = akka.event.Logging(context.system, this)
	    _log
	  }
	}

log方法会创建一个_log对象，并在保存下来，放置重复创建。

	object Logging...
	
	def apply[T: LogSource](system: ActorSystem, logSource: T): LoggingAdapter = {
      val (str, clazz) = LogSource(logSource, system)
      new BusLogging(system.eventStream, str, clazz)
	}

_log对象实际上是一个BusLogging对象。而当我们调用_log.info("")时，调用的trait LoggingAdapter中的info方法：

	trait LoggingAdapter...
	
	def info(message: String) { if (isInfoEnabled) notifyInfo(message) }

看到了吗，每一次调用都已经判断了isInfoEnabled，不用担心无意义的日志产出。

而notifyInfo方法是在class BusLogging中实现的：

	class BusLogging...
	
	protected def notifyInfo(message: String): Unit = bus.publish(Info(logSource, logClass, message, mdc))

这下发现所有逻辑都串上了是吧。EventStream就暂时分析到这里了。





									



	



