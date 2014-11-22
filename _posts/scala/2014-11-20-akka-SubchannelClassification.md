---
layout: post
title: akka-SubchannelClassification笔记
description: 
category: scala
---
SubchannelClassification是我看Akka源码到现在遇到的最复杂的结构。他作为一个工具类扩展了发布订阅模式，增加了订阅者自动关注发布内容子类的功能。

在开始写这篇笔记时我并没有完全看懂代码，但是我发现，如果我不把我看的懂的那一部分先记下来，我很快就会迷失在纷乱的代码中。

SubchannelClassification在akka中只用来做日志的发布和订阅，但设计者把他抽象出来，使用一切发布订阅模式。

EventStream是akka中的日志发布订阅的入口，他发布的event，是针对于不同级别的log日志。例如EventStream发布了Error级别的Event，那么所有订阅了Error级别的subscriber就会收到日志信息。EventStream混入了SubchannelClassification是为了处理日志级别的子类，包含MDC（Mapped Diagnostic Context由map存储的日志上下文诊断信息）的日志级别。举个例子，当subscriber订阅了Error级别的日志后，当有带MDC的Error级别日志发布时，subscriber依然能收到日志信息。抽象之后就是订阅了类型A的subscriber可以收到任何A的子类的事件。

	trait SubchannelClassification { this: EventBus ⇒
		
		private lazy val subscriptions = new SubclassifiedIndex[Classifier, Subscriber]()
	
	}


SubchannelClassification特质最主要的属性，莫过于这个lazy的subscriptions了。因为我们知道EventStream在ActorSystem中是单例的，所以在EventStream创建的时候，并触发发布或订阅事件时，subscriptions这个对象会被创建，EventStream中的这个SubclassifiedIndex类型的subscriptions会作为全局唯一的root节点，这一点非常重要。对于非root节点，会表示为Nonroot，他是SubclassifiedIndex的子类。

	class SubclassifiedIndex[K, V] private (protected var values: Set[V])(implicit sc: Subclassification[K]) {

	protected var subkeys = Vector.empty[Nonroot[K, V]]
	protected val root = this

	}
SubclassifiedIndex类有三个属性非常重要：

 1. root永远指向root节点
 2. values表示当前节点所有的subscriber
 3. subkeys表示当前节点所有的子节点Nonroot

另外，由于EventStream混入的EventBus特质要求实现类需要定义3种类型：

	type Event//事件
	type Classifier//订阅的事件的类型
    type Subscriber//订阅者的类型

在	EventStream中，Event类型为AnyRef，Classifier类型为Class[_]，Subscriber类型为ActorRef。Classifier和Subscriber同时又是SubclassifiedIndex中的Key和Value。

SubclassifiedIndex会把新增key或新增value的结果抽象为一个类型Changes，它实际上是immutable.Seq[(K, Set[V])]的别名，用来表示Classifier与众多Subscriber对应关系组成的集合。

另外一个需要提到的类是Subclassification，还需要实现以下两个方法：

	def isEqual(x: Class[_], y: Class[_]) = x == y//用来表示两个类相同
    def isSubclass(x: Class[_], y: Class[_]) = y isAssignableFrom x//用来表示x类为y的子类或者x类和y类相同

切记isSubclass中包含了类相同的判断。

这个类会作为隐式参数在新增key或新增value的时候发挥作用。

铺垫的差不多了，所以先来看看是如何注册subscriber的吧。

举一个典型场景，以下有三个带继承关系的类，分别是：

	class Person(name:String)
	class Teacher(name:String) extends Person(name)
	class Professor(name:String) extends Teacher(name)

假设我们要 依次 订阅这三种类型的事件：

	eventStream.subscribe(new Subscriber(1), classOf[Professor])
	eventStream.subscribe(new Subscriber(2), classOf[Teacher])
	eventStream.subscribe(new Subscriber(3), classOf[Person])



会是怎样一个流程呢？从subscribe这个方法看起。

	trait SubchannelClassification...

	def subscribe(subscriber: Subscriber, to: Classifier): Boolean = subscriptions.synchronized {
    val diff = subscriptions.addValue(to, subscriber)//在root上添加value，并返回当前未保存的（key->Set[value]）集合
    addToCache(diff)//把之前未保存的集合放入Cache中
    diff.nonEmpty//返回是否有新增的集合
  }

接下来看看root节点的addValue方法是如何处理的：

	def addValue(key: K, value: V): Changes = mergeChangesByKey(innerAddValue(key, value))

root节点的addValue方法有两个步骤：

 1. 执行innerAddValue方法，找到不同Classifier对应的Subscriber集合。注意root节点和Nonroot节点的innerAddValue是不同的。
 2. 执行mergeChangesByKey方法，会把innerAddValue的结果按照Classifier进行merge。

下面先来看看root节点的innerAddValue方法：

	protected def innerAddValue(key: K, value: V): Changes = {
      var found = false
      val ch = subkeys flatMap { n ⇒   
        if (sc.isSubclass(key, n.key)) {
          found = true
          n.innerAddValue(key, value)
        } else Nil
      }

      //--------分割线---------//
      if (!found) {
        val v = values + value
        val n = new Nonroot(root, key, v)
        integrate(n) ++ n.innerAddValue(key, value) :+ (key -> v) 
      } else ch
    }

这个方法非常重要，我把他分为上下两个部分，首先来看下半部分。

每次订阅一个新的类，都会首先在root节点上调用innerAddValue这个方法，所以subkeys代表root的节点的所有nonroot节点的集合，新订阅的Professor这个类自然不包含在subkeys内，found=false。成功进入下半部分。

在root中values这个变量永远是空得，val v = values + value获得包含Subscirber1的集合v，然后创建Nonroot，Nonroot的3个参数分别表示root的引用，当前Nonroot节点的Classifier Professor，以及这个新构建的包含Subscirber1的集合v。

之后要执行root的integrate方法，这个方法实现如下：

	private def integrate(n: Nonroot[K, V]): Changes = {
      val (subsub, sub) = subkeys partition (k ⇒ sc.isSubclass(k.key, n.key))
      subkeys = sub :+ n
      n.subkeys = if (subsub.nonEmpty) subsub else n.subkeys
      n.subkeys ++= findSubKeysExcept(n.key, n.subkeys).map(k ⇒ new Nonroot(root, k, values))
      n.subkeys.map(n ⇒ (n.key, n.values.toSet))
    }

val (subsub, sub) = subkeys partition (k ⇒ sc.isSubclass(k.key, n.key))会把root保存的所有子节点一份为二。subsub代表这些子节点中Perfessor的子类，sub表示这些节点中其他不相干类。注意的判断条件isSubclass虽然表示包含Professor类和他的子类，还记得之前华丽的分割线吗，如果发现在subkeys中发现有Professor类，就不会进入下半部分了。所以这里的subsub就指代Professor的子类。

重点来了，接下来的 subkeys = sub :+ n，表示如果subkeys中存在Perfessor类或他的子类，则一并拿Professor代替。

n.subkeys = if (subsub.nonEmpty) subsub else n.subkeys表示如果Professor的子类不为空，则把挂到Professorde subkeys下，这很合理。

看到这里可以总结出来一个规则，订阅了Person和Professor，那么Professor会挂在Person的subkeys，因为Professor是Person得子类。这时如果订阅Teacher类，那么Teacher类会挂在Person的subkeys下，而Professor类会从Person的subkeys中移动到Teacher的subkeys中。

因为这个规则有些绕，通过这个例子应该可以表达清楚了。

也有可能出现这种情况：

	trait Person
	trait Male 
	class Professor extends Person with Male

	eventStream.subscribe(new Subscriber(1), classOf[Professor])
	eventStream.subscribe(new Subscriber(2), classOf[Person])
	eventStream.subscribe(new Subscriber(3), classOf[Male])
	
如果首先订阅了Person和Professor类，再订阅Male特质，因为Male特质与Person没有继承关系，所以会进入到分割线的下半部分，但在root的subkeys找到自己的Perfessor类，因为Perfessor类会挂在Person的subkeys下。

所以就需要引入下面的逻辑：

	n.subkeys ++= findSubKeysExcept(n.key, n.subkeys).map(k ⇒ new Nonroot(root, k, values))

findSubKeysExcept方法如下：

	  protected final def findSubKeysExcept(key: K, except: Vector[Nonroot[K, V]]): Set[K] = root.innerFindSubKeys(key, except)
	  protected def innerFindSubKeys(key: K, except: Vector[Nonroot[K, V]]): Set[K] =
    (Set.empty[K] /: subkeys) { (s, n) ⇒
      if (sc.isEqual(key, n.key)) s
      else n.innerFindSubKeys(key, except) ++ {
        if (sc.isSubclass(n.key, key) && !except.exists(e ⇒ sc.isEqual(key, e.key)))
          s + n.key
        else
          s
      }
    }

这个方法会调用root开始，递归的查找所有子节点中是否有当前节点的子类，但又不包含已经放入当前节点subkeys中的那一部分类。之后把找到的结果用map转成Nonroot，挂在当前节点的subkeys上。

n.subkeys.map(n ⇒ (n.key, n.values.toSet))方法把Perfessor所有的子节点转化成子类和子类的Subscriber集合返回。

综上所述，integrate方法主要用来整理新订阅的Perfessor类的subkeys，并返回subkeys中的整理结果。

回到root节点的innerAddValue方法，在执行完integrate返回结果后，需要调用n.innerAddValue(key, value)。

	class Nonroot...

	override def innerAddValue(key: K, value: V): Changes = {
      // break the recursion on super when key is found and transition to recursive add-to-set
      if (sc.isEqual(key, this.key)) addValue(value) else super.innerAddValue(key, value)
    }

    private def addValue(value: V): Changes = {
      val kids = subkeys flatMap (_ addValue value)
      if (!(values contains value)) {
        values += value
        kids :+ ((key, Set(value)))
      } else kids
    }

还记得之前提到过的分割线吗，因为从下半部分逻辑调用innerAddValue时，
key, this.key必然相等，所以直接进入addValue方法。在这里会递归的在Perfessor的子类中添加Subscriber（如果子类没有被这个Subscriber订阅过的话），并把添加的结果返回。

回到root的innerAddValue方法，最后会把integrate(n)的结果，n.innerAddValue(key, value)的结果，以及当前订阅的(key -> v)，三者合并后返回。

Professor，Teacher，Person按照这种顺序进行订阅的话，每一个类都会按照刚才介绍的逻辑添加。不过如果按照Person，Teacher， Professor的顺序订阅会怎么样呢？

订阅第Person类时仍然会从分割线下半部分的逻辑进行。但是当订阅Teacher类时，遍历root.subkeys会发现Teacher类是Person的子类，然后执行n.innerAddValue(key, value)，这里的n就是封装着Person类的Nonroot，并调用super.innerAddValue(key, value)方法，再次遍历Person类的subkeys，从而进入分割线下半部分的逻辑，但此时subkeys指代的是Person的subkeys，也就是最终会把Teacher挂到Person的subkeys下面。

Person，Teacher， Professor的顺序订阅就会依次把自己挂到父类的subkeys中。

可喜的是取消订阅的逻辑就是订阅逻辑的逆过程，所以在这里就不用进行分析了。

来看看如何发布一个事件：

	trait SubchannelClassification...

	def publish(event: Event): Unit = {
      val c = classify(event)
      val recv =
        if (cache contains c) cache(c) // c will never be removed from cache
        else subscriptions.synchronized {
          if (cache contains c) cache(c)
          else {
            addToCache(subscriptions.addKey(c))
            cache(c)
          }
        }
      recv foreach (publish(event, _))
    }  

publish方法分为首先需要查找当前事件对应的类是否有Subscriber，经过加锁和double check之后仍然没有发现Subscriber，会按继承关系把当前事件的类添加到合适的位置。获得所有Subscriber后，依次调用publish(event, subscriber)方法发送事件，这里的publish方法根据业务的不同而留给SubchannelClassification的实现类去实现的。


	def addKey(key: K): Changes = mergeChangesByKey(innerAddKey(key))

	protected def innerAddKey(key: K): Changes = {
      var found = false
      val ch = subkeys flatMap { n ⇒
        if (sc.isEqual(key, n.key)) {
          found = true
          Nil
        } else if (sc.isSubclass(key, n.key)) {
          found = true
          n.innerAddKey(key)
        } else Nil
      }
      if (!found) {
        integrate(new Nonroot(root, key, values)) :+ ((key, values))
      } else ch
    }

在这里几乎不用逐句分析，因为逻辑和innerAddValue太相像了。在遍历root.subkeys的过程中如果遇到自己的父类，就需要进行递归。如果found == false时，会把新建的Nonroot作为参数调用integrate方法。



