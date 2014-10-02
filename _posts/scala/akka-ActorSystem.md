akka2.3.6

这次研究的内容是ActorSystem，要知道进入akka世界的入口就是创建一个ActorSystem，比如

	val system = ActorSystem("myWorld")
	
这一行简单地代码到底完成了什么工作，我今天就要开始把他了解清楚。

首先，ActorSystem实现了ActorRefFactory trait，顾名思义，也就是ActorRef的工厂，这个工厂提供了以下几个方法：

	protected def systemImpl: ActorSystemImpl

	protected def provider: ActorRefProvider

	implicit def dispatcher: ExecutionContextExecutor

	protected def guardian: InternalActorRef

	protected def lookupRoot: InternalActorRef

	def actorOf(props: Props): ActorRef

	def actorOf(props: Props, name: String): ActorRef
	
	def actorSelection(path: String): ActorSelection = path match {
	  case RelativeActorPath(elems) ⇒
		if (elems.isEmpty) ActorSelection(provider.deadLetters, "")
		else if (elems.head.isEmpty) ActorSelection(provider.rootGuardian, elems.tail)
		else ActorSelection(lookupRoot, elems)
	  case ActorPathExtractor(address, elems) ⇒
		ActorSelection(provider.rootGuardianAt(address), elems)
	  case _ ⇒
		ActorSelection(provider.deadLetters, "")
	}
   
	def stop(actor: ActorRef): Unit
	
顾名思义每一个ActorRef的工厂都需要有所属的system,要有actor的提供者provider, 分发mail的dispatcher, 当前工厂创建出的所有actor的挂载点guardian，提供查找guardian的方法lookupRoot, 已经最常用的创建actor的方法actorOf, 和唯一一个具体方法用来根据path查找actorRef的ActorSelection，以及stop方法。

混入ActorRefFactory这个trait的实现类中，ActorSystem只是其一，还有我们熟知的ActorContext集成这个trait，以及和Router相关的各种类，等等。

实际上，ActorSystem仍然是一个抽象类，ExtendedActorSystem继承自它.ExtendedActorSystem当前只有一个子类，就是ActorSystemImpl，从注释上来看ExtendedActorSystem并不打算让用户继承它，可能是用来为akka的后续功能提供的。

接下来看看ActorSystem到底做了什么。

当我们调用ActorSystem("myWorld")时，实际上调用的时object ActorSystem的apply(name:String)方法，这是对以下方法的重载：

	def apply(name: String, 
	  config: Option[Config] = None, 
	  classLoader: Option[ClassLoader] = None, 
	  defaultExecutionContext: Option[ExecutionContext] = None): ActorSystem = {
	  
	  val cl = classLoader.getOrElse(findClassLoader())
	  val appConfig = config.getOrElse(ConfigFactory.load(cl))
	  new ActorSystemImpl(name, appConfig, cl, defaultExecutionContext).start()
	}

从这个方法可以看出，当我们初始化ActorSystem的时候，不单单可以指定system名，实际上还可以通过参数传入配置信息，并指定这个system默认的classLoader以及默认的executor。最终会创建一个ActorSystemImpl，并启动。

下面我们来看看创建一个ActorSystemImpl对象内部做了哪些动作：

1.首先创建了一个uncaughtExceptionHandler函数，用来处理线程抛出的异常。这个函数在创建threadFactory：MonitorableThreadFactory的时候会作为参数传入，其他参数还包括actorSystem的名字作为前缀，是否是守护线程的配置项，以及classLoader来保证每一个创建出来的线程属于哪个classLoader。

	final val threadFactory: MonitorableThreadFactory =
	MonitorableThreadFactory(name, settings.Daemonicity, Option(classLoader), 		uncaughtExceptionHandler)

2.创建了dynamicAccess用来从ActorSystem中通过类反射创建对象及其他和类反射相关的操作，这个方法是一个扩展点，在类反射中加入特殊的业务逻辑达到创建出带有特殊属性的对象的目的。

	protected def createDynamicAccess(): DynamicAccess = new ReflectiveDynamicAccess(classLoader)
	
  	private val _pm: DynamicAccess = createDynamicAccess()
  	def dynamicAccess: DynamicAccess = _pm
  	
3.调用logConfiguration方法可以打出actorSystem中加载的所有配置项。

	def logConfiguration(): Unit = log.info(settings.toString)
	
4.提供了systemActorOf方法，用来创建/system目录下的actor，/system目录有别于/user目录，当actorSystem中所有user actor都背shutdown之后，/system目录下的actor才会被shutdown。从代码上可以看出，在/system下创建actor实际上是在systemGuardian下挂上他的子actor，并且设置actor属于systemService

	def systemActorOf(props: Props, name: String): ActorRef = systemGuardian.underlying.attachChild(props, name, systemService = true)
	
5.提供actorOf方法，也就是在/user目录下创建actor。

	def actorOf(props: Props, name: String): ActorRef = guardian.underlying.attachval eventStream: EventStream = new EventStream(DebugEventStream)
  eventStream.startStdoutLogger(settings)Child(props, name, systemService = false)

	def actorOf(props: Props): ActorRef = guardian.underlying.attachChild(props, systemService = false)
	
6.stop方法通过传入的actorRef参数，来停止指定的某一个actor。当然，会根据actor的类别不同，向不同的guardian发送stop的指令

	def stop(actor: ActorRef): Unit = {
	  val path = actor.path
	  val guard = guardian.path
      val sys = systemGuardian.path
      path.parent match {
	    case `guard` ⇒ guardian ! StopChild(actor)
	    case `sys`   ⇒ systemGuardian ! StopChild(actor)
	    case _	   ⇒ actor.asInstanceOf[InternalActorRef].stop()
	  }
	}
	
7.创建了一个事件流eventStream，由于这个内容太复杂，我打算在新的笔记中单独介绍这里地内容。

	val eventStream: EventStream = new EventStream(DebugEventStream)
    eventStream.startStdoutLogger(settings)
    val log: LoggingAdapter = new BusLogging(eventStream, "ActorSystem(" + name + ")", this.getClass)

8.创建了一个调度器Scheduler，可以重复执行任务，例如向某个actorRef发送消息。默认实现是LightArrayRevolverScheduler，但可以通过akka.scheduler.implementation配置项重置。由于这个调度器的实现也有些内容，我打算在另一篇笔记中单独介绍。

	val scheduler: Scheduler = createScheduler()
	
9.创建提供者provider。提供者是akka配置文件中必备的配置之一，也决定了你使用的akka是local, remote还是cluster。三种类型各自有自己的provider实现。akka是通过上文提到的dynamicAccess通过类反射进行创建的。
之后会另起一篇笔记单独介绍provider。

	val provider: ActorRefProvider = try {
      val arguments = Vector(
        classOf[String] -> name,
        classOf[Settings] -> settings,
        classOf[EventStream] -> eventStream,
        classOf[DynamicAccess] -> dynamicAccess)

      dynamicAccess.createInstanceFor[ActorRefProvider](ProviderClass, arguments).get
    } catch {
      case NonFatal(e) ⇒
        Try(stopScheduler())
        throw e
    }
    
10.提供获得provider deadLetter发送目的地的actorRef的方法

	def deadLetters: ActorRef = provider.deadLetters
	
11.创建了mailboxes对象，不过这个对象和Mailbox是不一样的，他只是一个helper类。里面包含了mailbox的一些工具方法。不过在其中明确创建了deadLetterMailbox。另有一篇笔记会单独介绍mailbox

	val mailboxes: Mailboxes = new Mailboxes(settings, eventStream, dynamicAccess, deadLetters)
	
12.创建了Dispatchers并指明默认处理所有actor行为的dispatcher。Dispatchers和mailboxes一样，是个helper类。另有一篇笔记会单独介绍dispatcher.

	val dispatchers: Dispatchers = new Dispatchers(settings, DefaultDispatcherPrerequisites(
    threadFactory, eventStream, scheduler, dynamicAccess, settings, mailboxes, defaultExecutionContext))

    val dispatcher: ExecutionContextExecutor = dispatchers.defaultGlobalDispatcher
    
13.创建了一个供内部使用的线程池internalCallingThreadExecutionContext，默认会初始化为批量提交的线程池。这个思路挺独特的，打算单独写篇笔记介绍他。

	val internalCallingThreadExecutionContext: ExecutionContext =
    dynamicAccess.getObjectFor[ExecutionContext]("scala.concurrent.Future$InternalCallbackExecutor$").getOrElse(
      new ExecutionContext with BatchingExecutor {
        override protected def unbatchedExecute(r: Runnable): Unit = r.run()
        override def reportFailure(t: Throwable): Unit = dispatcher reportFailure t
      })
      
12.之后这四个方法，直接可以调用provider中的四个属性，具体的在provider笔记中介绍。

	def terminationFuture: Future[Unit] = provider.terminationFuture
    def lookupRoot: InternalActorRef = provider.rootGuardian
    def guardian: LocalActorRef = provider.guardian
    def systemGuardian: LocalActorRef = provider.systemGuardian
    
13.定义了/方法，用来拼接path

	def /(actorName: String): ActorPath = guardian.path / actorName
    def /(path: Iterable[String]): ActorPath = guardian.path / path
    
14.非常有趣的start方法，多亏了scala的lazy特性。在start过程中会未关闭actorSystem注册stopScheduler方法，初始化provider，如果设置记录deadLetter,则初始化DeadLetterListener,如果设置了开始时打印conf信息，则进行打印，之后加载扩展程序。如果在初始化阶段有发生异常，则直接关闭。

	private lazy val _start: this.type = try {
      registerOnTermination(stopScheduler())
	  provider.init(this)
      if (settings.LogDeadLetters > 0)
        logDeadLetterListener = Some(systemActorOf(Props[DeadLetterListener], "deadLetterListener"))
      loadExtensions()
      if (LogConfigOnStart) logConfiguration()
        this
      } catch {
        case NonFatal(e) ⇒
          try {
            shutdown()
	      } catch { case NonFatal(_) ⇒ Try(stopScheduler()) }
        throw e
      }
           
    def start(): this.type = _start
    
15.紧接着，ActorSystem创建了一个TerminationCallbacks，他使用默认的dispatcher运行，并且可以通过两种注册回调的方法，注册多个回调函数，在actorSystem关闭时一并执行。

	private lazy val terminationCallbacks = {
      implicit val d = dispatcher
      val callbacks = new TerminationCallbacks
      terminationFuture onComplete (_ ⇒ callbacks.run)
      callbacks
    }
    def registerOnTermination[T](code: ⇒ T) { registerOnTermination(new Runnable { def run = code }) }
    def registerOnTermination(code: Runnable) { terminationCallbacks.add(code) }
    def awaitTermination(timeout: Duration) { Await.ready(terminationCallbacks, timeout) }
    def awaitTermination() = awaitTermination(Duration.Inf)
    def isTerminated = terminationCallbacks.isTerminated
    
    
16.有一部分处理扩展的逻辑

17.printTree方法可以打印出整个actorSystem层级结构

到此为止，真个actorSystem就初始化完成了。    
    
	    