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
 2. 执行mergeChangesByKey方法，根据Classifier进行merge。

下面先来看看root节点的innerAddValue方法：

	protected def innerAddValue(key: K, value: V): Changes = {
      var found = false
      val ch = subkeys flatMap { n ⇒   //遍历root节点的nonroot子节点
        if (sc.isSubclass(key, n.key)) {
          found = true
          n.innerAddValue(key, value)
        } else Nil
      }
      if (!found) {
        val v = values + value
        val n = new Nonroot(root, key, v)
        integrate(n) ++ n.innerAddValue(key, value) :+ (key -> v)
      } else ch
    }


									



	



