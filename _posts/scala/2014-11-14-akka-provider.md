---
layout: post
title: akka-provider笔记
description: 
category: scala
---

akka中的Provider是所有actor挂载的地方，按照akka的使用场景分为以下三种：

 1. akka.actor.LocalActorRefProvider (默认，本地场景)
 2. akka.remote.RemoteActorRefProvider (远程调用场景)
 3. akka.cluster.ClusterActorRefProvider (集群场景)

LocalActorRefProvider
--------------------------

下面首先对LocalActorRefProvider进行介绍。在之前分析ActorSystem初始化的时候可以知道Provider会随着ActorSystem的初始化而创建：

	class ActorSystemImpl...
	
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

还记得dynamicAccess吗，他是使用类反射进行对象初始化的工具类，从代码中可以看到，ActorSystem的name，参数集，eventStream和dynamicAccess是provider的初始化参数。ProviderClass来自于getString("akka.actor.provider")，也就是配置信息。

以下是LocalActorRefProvider的主构造函数和反射调用的构造函数。


	private[akka] class LocalActorRefProvider private[akka] (
	  _systemName: String,
	  override val settings: ActorSystem.Settings,
	  val eventStream: EventStream,
	  val dynamicAccess: DynamicAccess,
	  override val deployer: Deployer,
	  _deadLetters: Option[ActorPath ⇒ InternalActorRef])
	  extends ActorRefProvider {

	  def this(_systemName: String,
           settings: ActorSystem.Settings,
           eventStream: EventStream,
           dynamicAccess: DynamicAccess) =
	    this(_systemName,
	      settings,
	      eventStream,
	      dynamicAccess,
	      new Deployer(settings, dynamicAccess),
	      None)
	...
}

在初始化LocalActorRefProvider以下属性：

	private lazy val defaultDispatcher = system.dispatchers.defaultGlobalDispatcher

	private lazy val defaultMailbox = system.mailboxes.lookup(Mailboxes.DefaultMailboxId)

    override lazy val rootGuardian: LocalActorRef =
    new LocalActorRef(
      system,
      Props(classOf[LocalActorRefProvider.Guardian], rootGuardianStrategy),
      defaultDispatcher,
      defaultMailbox,
      theOneWhoWalksTheBubblesOfSpaceTime,
      rootPath) {
      override def getParent: InternalActorRef = this
      override def getSingleChild(name: String): InternalActorRef = name match {
        case "temp"        ⇒ tempContainer
        case "deadLetters" ⇒ deadLetters
        case other         ⇒ extraNames.get(other).getOrElse(super.getSingleChild(other))
      }
    }

defaultDispatcher和defaultMailbox会根据传入的初始化参数中类型创建，重点说一下rootGuardian。

rootGuardian是guardian和systemGuardian的监督者(supervisor)，顾名思义他是整个system的根节点，主要作用也是用来遍历actor树的。

actor树中的每一个节点实际上都是LocalActorRef（本地场景），LocalActorRef又是对actorCell的一个封装。可以通过调用LocalActorRef的方法触发actorCell相对应的动作。actorCell会作为LocalActorRef的属性underlying存在。

不得不附上ActorCell的主构造器，毕竟这多少能看出ActorCell的职能：

	private[akka] class ActorCell(
	  val system: ActorSystemImpl,
	  val self: InternalActorRef,
	  final val props: Props, // Must be final so that it can be properly cleared in clearActorCellFields
	  val dispatcher: MessageDispatcher,
	  val parent: InternalActorRef)
	  extends UntypedActorContext with AbstractActorContext with Cell
	  with dungeon.ReceiveTimeout
	  with dungeon.Children
	  with dungeon.Dispatch
	  with dungeon.DeathWatch
	  with dungeon.FaultHandling {
	...
	}

可以看出ActorCell除了以下几类特质：

 - UntypedActorContext，AbstractActorContext表示作为Actor可以获得的children相关信息，以及具备Actor的become特性（相当于重置Actor的receive方法）
 - Cell特质表示了Actor的生命周期，所属系统，发送消息等职能
 - dungeon包下的一系列特性，代表的ActorCell具备的零散的特殊职能

dungeon.Children特质中实现了Actor层级如何落地的方法，这也是和下面要介绍的guardian息息相关。

rootGuardian作为根节点创建之后就可以创建一些ActorSystem内置的子节点了。比如下面这个，/user节点。

	override lazy val guardian: LocalActorRef = {
      val cell = rootGuardian.underlying
      cell.reserveChild("user")
      val ref = new LocalActorRef(system,     Props(classOf[LocalActorRefProvider.Guardian],   guardianStrategy),
      defaultDispatcher, defaultMailbox, rootGuardian, rootPath / "user")
      cell.initChild(ref)
      ref.start()
      ref
    }

非常有必要逐句分析一下。
cell.reserveChild("user")实现如下
 
	 trait Children ...

	 @tailrec final def reserveChild(name: String): Boolean = {
      val c = childrenRefs
      swapChildrenRefs(c, c.reserve(name)) || reserveChild(name)
	  }

	
	  private var _childrenRefsDoNotCallMeDirectly: ChildrenContainer = EmptyChildrenContainer
	
	def childrenRefs: ChildrenContainer =
	    Unsafe.instance.getObjectVolatile(this, AbstractActorCell.childrenOffset).asInstanceOf[ChildrenContainer]

	@inline private final def swapChildrenRefs(oldChildren: ChildrenContainer, newChildren: ChildrenContainer): Boolean =
	    Unsafe.instance.compareAndSwapObject(this, AbstractActorCell.childrenOffset, oldChildren, newChildren)


reserveChild方法是在Children特质中实现的，在Children特质中_childrenRefsDoNotCallMeDirectly这个属性被默认设置为EmptyChildrenContainer单例对象，代表每一个新创建的节点最初都是叶节点，叶节点没有任何子节点，当某个节点拥有的自己的子节点之后，会通过自旋+CAS操作设置_childrenRefsDoNotCallMeDirectly属性。顺便一提的是自旋在scala里就非常自然的转化成尾递归了，虽然编译之后还会变成java中的自旋，不过可读性高太多了。

每一个叶子节点引用的EmptyChildrenContainer是一个单例对象，每一个叶子节点都会引用同一个EmptyChildrenContainer。当叶子节点有了自己的子节点之后，EmptyChildrenContainer就会被包含这个子节点的NormalChildrenContainer代替。每次增加新节点，都会创建一个包含所有子节点的NormalChildrenContainer。

回到初始化guardian的过程中，cell.reserveChild("user")之后，创建了/user对应的LocalActorRef，接下来要执行初始化子节点的工作。

cell.initChild(ref)的实现如下：

	trait Children ...
	
	@tailrec final def initChild(ref: ActorRef): Option[ChildRestartStats] = {
      val cc = childrenRefs
      cc.getByName(ref.path.name) match {
        case old @ Some(_: ChildRestartStats) ⇒ old.asInstanceOf[Option[ChildRestartStats]]
        case Some(ChildNameReserved) ⇒
          val crs = ChildRestartStats(ref)
          val name = ref.path.name
          if (swapChildrenRefs(cc, cc.add(name, crs))) Some(crs) else initChild(ref)
        case None ⇒ None
      }

childrenRefs返回的是一个ChildrenContainer，现在我们知道任何非叶子节点都实现都是NormalChildrenContainer。cc.getByName实际上就是从NormalChildrenContainer保存子节点的TreeMap中取出子节点的键值对，之后把原先用来暂存的ChildNameReserved替换为ChildRestartStats。

回到初始化guardian的过程中，创建了guardian对应的LocalActorRef并把他加入到rootGuardian的子节点之后需要启动guardian

	class LocalActorRef...
	
	override def start(): Unit = actorCell.start()

	trait Dispatch...
	
	def start(): this.type = {
    // This call is expected to start off the actor by scheduling its mailbox.
      dispatcher.attach(this)
      this
    }

从代码上可以看出ret.start()的实质是把当前的ActorCell挂到Dispatcher上。

akka Actor的层级系统中除了有guardian(/user目录)还有systemGuardian(/system目录)。systemGuardian与guardian同理，都是作为一个LocalActorRef挂在rootGuardian下，只不过systemGuardian用来作为所有系统actor的父节点。

接下来要介绍最关键的actorOf方法，也就是如何创建Actor。

我们可以在actorSystem下调用actorOf方法，也可以在某个actor内部调用actorOf方法，区别在于新创建的actor的父节点挂在哪里。不过在上面铺垫了那么多内容之后，我们知道实际上actorSystem内部的guardian也是一个actor，actorSystem创建的actor都被挂在到guardian下：

	class ActorSystemImpl...

	def actorOf(props: Props, name: String): ActorRef = guardian.underlying.attachChild(props, name, systemService = false)

	def actorOf(props: Props): ActorRef = guardian.underlying.attachChild(props, systemService = false)

当我们在Actor内部调用context.actorOf时，context实际上是当前ActorCell的引用。

因此actorOf创建Actor实际上都是通过ActorCell来创建的，而attachChild和actorOf方法的实现在都Children特质中：

	def actorOf(props: Props): ActorRef =
    makeChild(this, props, randomName(), async = false, systemService = false)
    def actorOf(props: Props, name: String): ActorRef =
    makeChild(this, props, checkName(name), async = false, systemService = false)
    private[akka] def attachChild(props: Props, systemService: Boolean): ActorRef =
    makeChild(this, props, randomName(), async = true, systemService = systemService)
    private[akka] def attachChild(props: Props, name: String, systemService: Boolean): ActorRef =
    makeChild(this, props, checkName(name), async = true, systemService = systemService)

makeChild这个方法有点长，就不在这里贴出来了，因为他仍然是一个过度方法。makeChild主要做了以下几件事：

 1. 如果actor不是LocalActor，则会验证一下他是否可以正常进行序列化和反序列化。
 2. 如果当前Actor正在关闭，则直接异常抛出
 3. reserveChild，创建actor，initChild，actor.start

重点在于如何创建actor，其他的内容上面都已经介绍过了。创建Actor的代码如下：

	trait Children...

	val actor =
        try {
          val childPath = new ChildActorPath(cell.self.path, name, ActorCell.newUid())
          cell.provider.actorOf(cell.systemImpl, props, cell.self, childPath,
            systemService = systemService, deploy = None, lookupDeploy = true, async = async)
        } catch {
          case e: InterruptedException ⇒
            unreserveChild(name)
            Thread.interrupted() // clear interrupted flag before throwing according to java convention
            throw e
          case NonFatal(e) ⇒
            unreserveChild(name)
            throw e
        }

 1. 首先根据当前路径，actor的name，以及通过ThreadLocal内生成的uid创建ChildActorPath
 2. 调用provider的actorOf方法创建Actor
 3. 创建如果失败，会unreserveChild，并抛出异常

provider的actorOf方法实在太长，根据props中的deploy部分通过一个模式匹配一分为二：NoRouter的逻辑，也就是我现在在学习的部分；Router的逻辑，我打算在学习Router的时候再进行研究。

所以接下来只关注NoRouter，即常规创建Actor的方式，按逻辑分块分析：

	class LocalActorRefProvider...
	
	if (settings.DebugRouterMisconfiguration) {
          deployer.lookup(path) foreach { d ⇒
            if (d.routerConfig != NoRouter)
              log.warning("Configuration says that [{}] should be a router, but code disagrees. Remove the config or add a routerConfig to its Props.", path)
          }
        }

如果在配置中开启了DebugRouterMisconfiguration，而在props.deploy配置信息中却没有配置router，这里会打印warning级别的日志。

	val props2 =
          // mailbox and dispatcher defined in deploy should override props
          (if (lookupDeploy) deployer.lookup(path) else deploy) match {
            case Some(d) ⇒
              (d.dispatcher, d.mailbox) match {
                case (Deploy.NoDispatcherGiven, Deploy.NoMailboxGiven) ⇒ props
                case (dsp, Deploy.NoMailboxGiven)                      ⇒ props.withDispatcher(dsp)
                case (Deploy.NoMailboxGiven, mbx)                      ⇒ props.withMailbox(mbx)
                case (dsp, mbx)                                        ⇒ props.withDispatcher(dsp).withMailbox(mbx)
              }
            case _ ⇒ props // no deployment config found
          }

接下来会根据传入的props进行整理，用mailbox和dispatcher配置项中的全局配置覆盖props中的配置。

	if (!system.dispatchers.hasDispatcher(props2.dispatcher))
          throw new ConfigurationException(s"Dispatcher [${props2.dispatcher}] not configured for path $path")

如果全局dispatchers中不包含props2中的dispatcher，则抛出配置异常。

	try {
          val dispatcher = system.dispatchers.lookup(props2.dispatcher)
          val mailboxType = system.mailboxes.getMailboxType(props2, dispatcher.configurator.config)

          if (async) new RepointableActorRef(system, props2, dispatcher, mailboxType, supervisor, path).initialize(async)
          else new LocalActorRef(system, props2, dispatcher, mailboxType, supervisor, path)
        } catch {
          case NonFatal(e) ⇒ throw new ConfigurationException(
            s"configuration problem while creating [$path] with dispatcher [${props2.dispatcher}] and mailbox [${props2.mailbox}]", e)
        }

接下来会获取dispatcher和mailboxType，判断如果需要异步执行Actor则创建RepointableActorRef，如果是同步执行，则直接创建LocalActorRef。这期间遇到任何异常都会抛出。

异步创建RepointableActorRef的过程我会单独在一篇笔记中介绍。

下面来看一下actorSelection的实现，因为无论是ActorSystem.actorSelection还是是每一个Actor内调用context.actorSelection由于继承关系，实际上调用的都是ActorRefFactory的方法：

	trait ActorRefFactory...
	
	def actorSelection(path: String): ActorSelection = path match {
      case RelativeActorPath(elems) ⇒
        if (elems.isEmpty)   ActorSelection(provider.deadLetters, "")
        else if (elems.head.isEmpty) ActorSelection(provider.rootGuardian, elems.tail)
        else ActorSelection(lookupRoot, elems)
      case ActorPathExtractor(address, elems) ⇒
        ActorSelection(provider.rootGuardianAt(address), elems)
      case _ ⇒
        ActorSelection(provider.deadLetters, "")
    }

分为三种情况：

 - 最常见的相对路径，
 - 形如akka://systemName/user/myActor 的绝对路径
 - 无效路径
 
 最终返回ActorSelection对象。






