# actors

[Actor模型](http://en.wikipedia.org/wiki/Actor_model)为编写并发和分布式系统提供了一种更高的抽象级别。它将开发人员从显式地处理锁和线程管理的工作中解脱出来，使编写并发和并行系统更加容易。Actor模型是在1973年Carl Hewitt的论文中提出的，但只是被`Erlang`语言采用后才变得流行起来，一个成功案例是爱立信使用`Erlang`非常成功地创建了高并发的可靠的电信系统。

Akka Actor的API与Scala Actor类似，并且从Erlang中借用了一些语法。

## 创建Actor

*注：由于Akka采用强制性的父子监管，每一个actor都被监管着，并且会监管它的子actors；我们建议你熟悉一下[Actor系统](general/actor-systems)和[监管与监控](general/supervision-and-monitoring.md)*

### 定义一个 Actor 类

要定义自己的Actor类，需要继承Actor类并实现`receive`方法。 `receive`方法需要定义一系列`case`语句(类型为`PartialFunction[Any, Unit])` 来描述你的Actor能够处理哪些消息（使用标准的Scala模式匹配），以及实现对消息如何进行处理的代码。

下面是一个例子

```scala
import akka.actor.Actor
import akka.actor.Props
import akka.event.Logging
class MyActor extends Actor {
  val log = Logging(context.system, this)
  def receive = {
    case "test" => log.info("received test")
    case _      => log.info("received unknown message")
  }
}
```

请注意Akka Actor receive消息循环是穷尽的(exhaustive), 这与Erlang和Scala的Actor行为不同。 这意味着你需要提供一个对它所能够接受的所有消息的模式匹配规则，如果你希望处理未知的消息，你需要象上例一样提供一个缺省的case分支。否则会有一个`akka.actor.UnhandledMessage(message, sender, recipient)`被发布到`Actor系统（ActorSystem)`的事件流（EventStream中。

注意，以上定义的行为的返回类型是`Unit`；如果actor需要回复接收的消息，这需要显示的实现。

`receive`方法的结果是一个偏函数对象，它作为初始化行为(initial behavior)保存在actor中，请查看[become/unbecome]了解更进一步的信息。

### Props

`Props`是一个用来在创建actor时指定选项的配置类。因为它是不可变的，所以可以在创建actor时共享包含部署信息的props。以下是使用如何创建`Props`实例的一些例子。

```scala
import akka.actor.Props
val props1 = Props[MyActor]
val props2 = Props(new ActorWithArgs("arg")) // careful, see below
val props3 = Props(classOf[ActorWithArgs], "arg")
```

第二种方法显示了创建actor时如何传递构造器参数。但它应该仅仅用于actor外部。

最后一行显示了传递构造器参数的一种可能的方法，无论它使用的context是什么。如果在构造Props的时候，需要找到了一个匹配的构造器。如果没有或者有多个匹配的构造器，那么会报`IllegalArgumentEception`异常。

#### 危险的方法

```scala
￼// NOT RECOMMENDED within another actor:
// encourages to close over enclosing class
val props7 = Props(new MyActor)
```

在其它的actor中使用这种方法是不推荐的。因为它返回了没有实例化的Props以及可能的竞争条件（打破了actor的封装）。我们将在未来提供一个`macro-based`的解决方案提供相似的语法，从而将这种方法抛弃。

*注意：在其它的actor中定义一个actor是非常危险的，它打破了actor的封装。永远不要传递actor的this引用到`Props`中去*

#### 推荐的方法

在每个actor的伴生对象中提供工厂方法是一个好主意。这可以保持合适`Props`的创建与actor的定义的距离。

```scala
object DemoActor {
  /**
   * Create Props for an actor of this type.
   * @param magciNumber The magic number to be passed to this actor’s constructor.
   * @return a Props for creating this actor, which can then be further configured
   *
   (e.g. calling ‘.withDispatcher()‘ on it)
   */
     def props(magicNumber: Int): Props = Props(new DemoActor(magicNumber))
   }
   class DemoActor(magicNumber: Int) extends Actor {
     def receive = {
       case x: Int => sender() ! (x + magicNumber)
     }
   }
   class SomeOtherActor extends Actor {
     // Props(new DemoActor(42)) would not be safe
     context.actorOf(DemoActor.props(42), "demo")
     // ...
   }
```

### 使用Props创建Actor

Actor可以通过将 `Props` 实例传入 `actorOf` 工厂方法来创建。这个工厂方法可以用于`ActorSystem`和`ActorContext`。

```scala
import akka.actor.ActorSystem
// ActorSystem is a heavy object: create only one per application
val system = ActorSystem("mySystem")
val myActor = system.actorOf(Props[MyActor], "myactor2")
```
使用`ActorSystem`将会创建top-level actor，被actor系统提供的监护人actor监控，同时，利用actor的context将会创建子actor。

```scala
class FirstActor extends Actor {
  val child = context.actorOf(Props[MyActor], name = "myChild")
  // plus some behavior ...
}
```

推荐创建一个由子actor、孙actor等构成的树形结构，从而使其适合应用程序的逻辑错误处理结构。

调用`actorOf`会返回一个`ActorRef`对象。它是与actor实例交互的唯一手段。`ActorRef`是不可变的，并且和actor拥有一对一的关系。`ActorRef`也是可序列化的以及网络可感知的。这就意味着，你可以序列化它、通过网络发送它、在远程机器上使用它，它将一直代表同一个actor。

name参数是可选的，但是你最后为你的actor设置一个name，它可以用于日志消息以及识别actor。name不能为空或者以$开头，但它可以包含URL编码字符(如表示空格的20%)。如果给定的name在其他子actor中已经存在，那么会抛出`InvalidActorNameException`异常。

当创建actor后，它自动以异步的方式开始。

### 依赖注入

如上所述，当你的actor拥有一个持有参数的构造器，那么这些参数也必须是Props的一部分。但是，当一个工厂方法必须被使用时除外。例如，当实际的构造器参数由一个依赖注入框架决定。

```scala
import akka.actor.IndirectActorProducer
class DependencyInjector(applicationContext: AnyRef, beanName: String)
  extends IndirectActorProducer {
    override def actorClass = classOf[Actor]
    override def produce =
      // obtain fresh Actor instance from DI framework ...
  }
  val actorRef = system.actorOf(
    Props(classOf[DependencyInjector], applicationContext, "hello"),
    "helloBean")
```

### 收件箱(inbox)

当在actor外编写代码与actor进行通讯时，`ask`模式可以解决，但是有两件事情它们不能做：接收多个回复、观察其它actor的生命周期。我们可以用inbox类实现该目的：

```scala
￼implicit val i = inbox()
 echo ! "hello"
 i.receive() should be("hello")
```
从inbox到actor引用的隐式转换意味着，在这个例子中，sender引用隐含在inbox中。它允许在最后一行接收回复。观察actor也非常简单。

```scala
￼val target = // some actor
 val i = inbox()
 i watch target
```







