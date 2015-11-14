---
layout: post
title: akka中request/response实现策略
description: 
category: scala
---

在Akka处理请求&响应的策略
=======

使用Akka的过程中最常见的就是处理请求&响应，在《Akka Concurrency》一书中列举了几种典型的处理请求&响应的策略，我翻译了一下，希望对大家有帮助。


----------


我们知道，Akka处理请求&响应的关键就是创建一个用来解析响应的上下文。举个例子，我们设计了一个Actor，用它来控制飞机上升和下降（贯穿《Akka Concurrency》一书的例子）。这个Actor会接收到控制高度的消息（如GoUp, GoDown），需要添加一些由DeflectionCalculator这个Actor计算的随机偏移量。

下面是一个糟糕的实现：

	class BadIdea extends Actor {
          val deflection = context.actorOf(Props[DeflectionCalculator])
          def receive = {
            case GoUp(amount) =>
              deflection ! GetDeflectionAmount
            case GoDown(amount) =>
              deflection ! GetDeflectionAmount
            case DeflectionAmount(amount) =>
              // 这...现在我是应该上升还是下降呢？
		}
	 }

当我们接收到DeflectionCalculator返回的偏移量amount时，我们无从知道是应该上升还是下降（因为我们丢失了关键的上下文）。

基于这个例子，有以下几种解决方法：

Future
-----------
 
使用Future可以让我们无需带着数据到处跑，而是很自然的在上下文的操作中保留一个数据的引用。下面给出用Future解决飞机上升下降的例子：

	class FutureIdea(controlSurfaces: ActorRef) extends Actor {
	    val deflection = context.actorOf(Props[DeflectionCalculator])
        def receive = {
	        case GoUp(amount) => // 获取偏移量，并把它转化成StickBack的消息，发送给controlSurfaces
                (deflection ? GetDeflectionAmount).mapTo[DeflectionAmount].map { amt =>
                  val DeflectionAmount(deflection) = amt
                  StickBack(amount + deflection)
                } pipeTo controlSurfaces
			case GoDown(amount) => // 获取偏移量，并把它转化成StickBack的消息，发送给controlSurfaces
                (deflection ? GetDeflectionAmount).mapTo[DeflectionAmount].map { amt =>
                 val DeflectionAmount(deflection) = amt
		  StickForward(amount + deflection)
				} pipeTo controlSurfaces
		}
	}

使用Future很方便的一点是，我们可以在请求时设置超时时间。假如响应方超时了，Future会抛出异常，我们就可以对异常进行处理。而且，因为我们是在上下文中处理的异常，所以我们可以明确知道这次异常是哪一次请求造成的。


 
Actor
----
有些时候使用Future不能解决我们遇到的问题。例如我们有很多消息需要处理、很多状态需要转换、或是业务逻辑特别复杂。这时即便我们可以把这段逻辑写在Future内，使用Actor则会使逻辑更加清晰。你可以创建一种Actor来处理这段逻辑，或是索性使用匿名的Actor。下面给出使用匿名Actor的实现方式：

	class ActorIdea(controlSurfaces: ActorRef) extends Actor {
	    val deflection = context.actorOf(Props[DeflectionCalculator])
	    def receive = {
		    case GoUp(amount) =>  //运行一个匿名的Actor来发送包含偏移量的StickBack消息给controlSurfaces
		        context.actorOf(Props(new Actor {
			        override def preStart() = deflection ! GetDeflectionAmount
					def receive = {
			            case DeflectionAmount(deflection) =>
							controlSurfaces ! StickBack(amount + deflection)			                
				            context.stop(self) //记得处理完之后关闭自己
					}
				}
			))	
			case GoDown(amount) => //运行一个匿名的Actor来发送包含偏移量的StickBack消息给controlSurfaces
	            context.actorOf(Props(new Actor {
					override def preStart() = deflection ! GetDeflectionAmount
					def receive = {
			            case DeflectionAmount(deflection) =>
				              controlSurfaces ! StickForward(amount + deflection)              
				              context.stop(self)//记得处理完之后关闭自己
				}
			}
		))
		}
	}

这个例子与上文提到的Future的例子很像，只不过使用Actor不容易设置响应的超时时间。有时候你不得不接收ReceiveTimeout消息去表示你等待的响应超时了。但是如果你在同时等待多条消息，那么只要有一条消息返回，超时时间就会被重置，仍然无法完美解决超时问题。
 
在Actor内部变量保存数据
---
在处理消息的过程中，我们也可以使用Actor内部的变量来保存上下文。

	class VarIdea(controlSurfaces: ActorRef) extends Actor {
		val deflection = context.actorOf(Props[DeflectionCalculator])
		var lastRequestWas = ""
		var lastAmount = 0f
		def receive = {
		    case GoUp(amount) =>
		        lastRequestWas = "GoUp"
		        lastAmount = amount
		        deflection ! GetDeflectionAmount
			case GoDown(amount) =>
		        lastRequestWas = "GoDown"
		        lastAmount = amount
		        deflection ! GetDeflectionAmount
			case DeflectionAmount(deflection) if lastRequestWas == "GoUp" =>
				controlSurfaces ! StickBack(deflection + lastAmount)
			case DeflectionAmount(deflection) if lastRequestWas == "GoDown" =>
				controlSurfaces ! StickForward(deflection + lastAmount)
		}
	}
  
第一印象，这种写法非常丑陋，不过在某些特殊场景下效果还不错。需要格外注意以下几点：

 1. 你的请求和响应是确定顺序的，比如：GoUp, DeflectionAmount, GoDown, DeflectionAmount, GoDown, DeflectionAmount...如果你接到的顺序是GoDown, GoDown, GoUp, DeflectionAmount...那一切都会乱套。
 2. 遇到Actor需要重启的时候会遇到问题，因为Actor内部的变量会被重置为初始值。
 3. 对变化适应力弱。当你的Actor变得非常复杂，对状态的修改会变得难以控制，请求中的状态很难和请求本身联系上，相反它们会散落在Actor的各个角落。这种解决方案只适合一个Actor处理一件事情，而不是迷失在纷繁杂乱的各种请求和响应中。
 4. 超时处理。如果响应方没有及时响应，你必须决定接下来该怎么做。

 
在消息中携带上下文数据
---
你也可以选择在消息携带上下文数据，这样给你带来的好处是减少资源使用（毕竟不用创建Future或Actor），但是会扰乱你在消息传递时使用的协议。

	class MsgIdea(controlSurfaces: ActorRef) extends Actor {
		val deflection = context.actorOf(Props[DeflectionCalculator2])
		def receive = {
			case GoUp(amount) =>
			    deflection ! GetDeflectionAmount("GoUp", amount)
		    case GoDown(amount) =>
				deflection ! GetDeflectionAmount("GoDown", amount)
		    case DeflectionAmount(op, amount, deflection) if op == "GoUp" =>
		        controlSurfaces ! StickBack(deflection + amount)
		    case DeflectionAmount(op, amount, deflection) if op == "GoDown" =>
		        controlSurfaces ! StickForward(deflection + amount)
		}
	}

 DeflectionCalculator2这个Actor作为响应方，将不得不把上下文中用到的数据放在响应的消息中。需要格外注意以下几点：

 1. 在响应消息中携带的数据如果由于未知原因导致损坏，有可能导致运行时异常。（响应的数据与请求的数据无法对应上）
 2. 增加了请求和响应之间的耦合
 3. 当你想要添加或移除任何属性时，代码改动不够灵活。当然，你也可以使用一个包含所有属性的Case类会，这样响应方只需要在消息中持有这个Case类而不需要关心属性的增减。
 4. 当你使用Remote Actor的时候，在消息中携带过多的数据会影响网络负载。
 5. 你仍然需要处理超时的情况。




Tag
---

我们可以在“Actor内部保存数据”与“使用消息携带数据”两种情况做一个折中。你可以在上下文中选出一个最简数据放入消息中，我们称之为Tag，之后使用这个Tag作为Actor内部数据的唯一标识。

	class TagIdea(controlSurfaces: ActorRef) extends Actor {
	    import scala.collection.mutable.Map
	    val deflection = context.actorOf(Props[DeflectionCalculator3])
	    //用来保存Tag的Map
	    val tagMap = Map.empty[Int, Tuple2[String, Float]]
		//我们的Tag是Int类型的
		var tagNum = 1
		def receive = {
            case GoUp(amount) =>
              //把上下文放入Map
              tagMap += (tagNum -> ("GoUp", amount))
              deflection ! GetDeflectionAmount(tagNum)
              tagNum += 1
            case GoDown(amount) =>
              //把上下文放入Map
              tagMap += (tagNum -> ("GoDown", amount))
              deflection ! GetDeflectionAmount(tagNum)
              tagNum += 1
            case DeflectionAmount(tag, deflection) =>
              //从Map中获取上下文
              val (op, amount) = tagMap(tag)
              val amt = amount + deflection
              //从Map中移除上下文
              tagMap -= tag
              if (op == "GoUp") controlSurfaces ! StickBack(amt)
              else controlSurfaces ! StickForward(amt)
		} 
	}

通过使用Tag我们瞬间解决了不少问题，不过需要注意以下几点：

 1. 消息协议仍然有些杂乱。假如我们收到的响应中包含一个错误的Tag，好吧，我们肯定会死的很惨。
 2. 我们仍然需要考虑超时问题。你或许需要在Map中保存发起请求的时间戳，这样你可以适时的清除那些过期的Tag。
 3. 当Actor重启之后你需要决定该如何处理。

综述
---
我们列举了几种处理请求&响应的策略，那么哪种场景需要选择哪种策略呢？你需要从以下两点考虑：

1.实现难易度。Future是最简单的。你可以随时关闭Actor，自然的创建响应需要的上下文，轻松又愉快。

2.运行速度。使用Future来处理业务逻辑对你来说可能有些笨重，所以你应该避免使用它。即便如此，请不要预先优化（**don’t prematurely optimize**）。假如你的需求是每秒处理百万级别的请求&响应，那Future可能显得笨重了。如果产品跟你说要的就是这种效果，并且用户感受到的延迟确实是由Future造成的，那么你可以考虑换个思路，到那时再考虑使用适合你的特定场景的策略吧。


----------
以上就是这段内容的翻译。

我们在开发过程中用到了Akka，处理请求&响应时使用的是Future和Tag两种策略，打磨过程中确实遇到了上面提及的几个问题。现在想想要是提前看了这篇文章是不是能少走一些弯路呢？

在此分享给大家。