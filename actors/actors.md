# actors

[Actor模型](http://en.wikipedia.org/wiki/Actor_model)为编写并发和分布式系统提供了一种更高的抽象级别。它将开发人员从显式地处理锁和线程管理的工作中解脱出来，使编写并发和并行系统更加容易。Actor模型是在1973年Carl Hewitt的论文中提出的，但只是被`Erlang`语言采用后才变得流行起来，一个成功案例是爱立信使用`Erlang`非常成功地创建了高并发的可靠的电信系统。

Akka Actor的API与Scala Actor类似，并且从Erlang中借用了一些语法。

## 1 创建Actor

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

## 2 Actor API

`Actor` trait 只定义了一个抽象方法，就是上面提到的`receive`, 用来实现actor的行为。

如果当前 actor 的行为与收到的消息不匹配，则会调用unhandled, 它的缺省实现是向actor系统的事件流中发布一个`akka.actor.UnhandledMessage(message, sender, recipient)`（设置`akka.actor.debug.unhandled`为on将它们转换为实际的Debug消息）。

另外，它还包括:

- `self` 代表本actor的 ActorRef
- `sender` 代表最近收到的消息的sender actor，通常用于下面将讲到的回应消息中
- `supervisorStrategy` 用户可重写它来定义对子actor的监管策略
- `context` 暴露actor和当前消息的上下文信息，如：

    - 用于创建子actor的工厂方法 (actorOf)
    - actor所属的系统
    - 父监管者
    - 所监管的子actor
    - 生命周期监控
    - hotswap行为栈

你可以import `context`的成员来避免总是要加上`context.`前缀

```scala
class FirstActor extends Actor {
  import context._
  val myActor = actorOf(Props[MyActor], name = "myactor")
  def receive = {
    case x => myActor ! x
  }
}
```

其余的可见方法是可以被用户重写的生命周期hook，描述如下:

```scala
def preStart(): Unit = ()
def postStop(): Unit = ()
def preRestart(reason: Throwable, message: Option[Any]): Unit = { context.children foreach { child )
    context.unwatch(child)
    context.stop(child)
  }
postStop() }
def postRestart(reason: Throwable): Unit = {
  preStart()
}
```

以上所示的实现是`Actor` trait 的缺省实现。
    
### Actor 生命周期

![actor lifecycle](../imgs/actor_lifecycle.png)

在actor系统中，一个路径代表一个“位置（place）”，这个位置被一个存活的actor占据。path被初始化为空。当调用`actorOf()`时，它分配一个actor的incarnation给给定的path。一个actor incarnation通过path和一个UID来识别。重启仅仅会交换通过Props定义的actor实例，它的实体以及UID不会改变。

当actor停止是，它的incarnation的生命周期结束。在那时，适当的生命周期事件被调用，终止的观察actor被通知。actor incarnation停止后，通过`actorOf()`创建actor时，path可以被重用。在这种情况下，新的incarnation的名字和之前的incarnation的名字是相同的，不同的是它们的UID。

一个`ActorRef`总是表示incarnation而不仅仅是path。因此，如果一个actor被停止，拥有相同名字的新的actor被创建。旧的incarnation的ActorRef不会指向新的Actor。

另一方面，`ActorSelection`指向path（如果用到占位符，是多个path），它完全不知道哪一个incarnation占有它。正是因为这个原因，`ActorSelection`无法被观察。通过发送一个`Identify`消息到`ActorSelection`可以解析当前incarnation存活在path下的`ActorRef`。这个`ActorSelection`将会被回复一个包含正确引用的`ActorIdentity`。这也可以通过`ActorSelection`的`resolveOne`方法实现，这个方法返回一个匹配`ActorRef`的`Future`。

### 使用DeathWatch进行生命周期监控

为了在其它actor结束时 (永久终止, 而不是临时的失败和重启)收到通知, actor可以将自己注册为其它actor在终止时所发布的`Terminated`消息的接收者。 这个服务是由actor系统的`DeathWatch`组件提供的。

注册一个监控器很简单：

```scala
import akka.actor.{ Actor, Props, Terminated }
class WatchActor extends Actor {
  val child = context.actorOf(Props.empty, "child")
  context.watch(child) // <-- this is the only call needed for registration
  var lastSender = system.deadLetters
  def receive = {
    case "kill" =>
      context.stop(child); lastSender = sender()
    case Terminated(‘child‘) => lastSender ! "finished"
} }
```

要注意`Terminated`消息的产生与注册和终止行为所发生的顺序无关。特别是，观察actor将会接收到一个`Terminated`消息，即使观察actor在注册时已经终止了。

多次注册并不表示会有多个消息产生，也不保证有且只有一个这样的消息被接收到：如果被监控的actor已经生成了消息并且已经进入了队列，在这个消息被处理之前又发生了另一次注册，则会有第二个消息进入队列，因为一个已经终止的actor注册监控器会立刻导致`Terminated`消息的发生。

可以使用`context.unwatch(target)`来停止对另一个actor的生存状态的监控, 但很明显这不能保证不会接收到`Terminated`消息因为该消息可能已经进入了队列。

### 启动 Hook

actor启动后，它的`preStart`会被立即执行。

```scala
￼override def preStart() {
  child = context.actorOf(Props[MyActor], "child")
}
```
当actor第一次被创建时，该方法会被调用。在重启的过程中，它会在`postRestart`的默认实现中调用，这意味着，对于这个actor或者每一个重启，重写那个方法你可以决定初始化代码是否被仅仅调用一次。当创建一个actor实例时，作为actor构造器一部分的初始化代码（在每一次重启时发生）总是被调用。

### 重启 Hook

所有的Actor都是被监管的，以某种失败处理策略与另一个actor链接在一起。 如果在处理一个消息的时候抛出的异常，Actor将被重启。这个重启过程包括上面提到的Hook:

- 要被重启的actor的`preRestart`被调用，携带着导致重启的异常以及触发异常的消息; 如果重启并不是因为消息的处理而发生的，所携带的消息为 None , 例如，当一个监管者没有处理某个异常继而被它自己的监管者重启时。 这个方法是用来完成清理、准备移交给新的actor实例的最佳位置。它的缺省实现是终止所有的子actor并调用`postStop`。
- 最初 actorOf 调用的工厂方法将被用来创建新的实例。
- 新的actor的 postRestart 方法被调用，携带着导致重启的异常信息。默认情况下，`preStart`被调用。

actor的重启会替换掉原来的actor对象; 重启不影响邮箱的内容, 所以对消息的处理将在`postRestart hook`返回后继续。 触发异常的消息不会被重新接收。在actor重启过程中所有发送到该actor的消息将象平常一样被放进邮箱队列中。

### 终止 Hook

一个Actor终止后，它的`postStop hook`将被调用, 这可以用来取消该actor在其它服务中的注册。这个hook保证在该actor的消息队列被禁止后才运行， i.e. 之后发给该actor的消息将被重定向到`ActorSystem`的`deadLetters`中。

## 3 通过Actor Selection识别actor



## 4 消息与不可变性

`IMPORTANT`: 消息可以是任何类型的对象，但必须是不可变的。目前Scala还无法强制不可变性，所以这一点必须作为约定。String, Int, Boolean这些原始类型总是不可变的。 除了它们以外，推荐的做法是使用`Scala case class`，它们是不可变的（如果你不专门暴露数据的话），并与接收方的模式匹配配合得非常好。

以下是一个例子:

```scala
// define the case class
case class Register(user: User)
// create a new case class message
val message = Register(user)
```

## 5 发送消息

向actor发送消息是使用下列方法之一：

- `!` 意思是“fire-and-forget”，表示异步发送一个消息并立即返回。也称为`tell`。
- `?` 异步发送一条消息并返回一个 Future代表一个可能的回应。也称为`ask`。

每一个消息发送者分别保证自己的消息的次序。

### Tell: Fire-forget

这是发送消息的推荐方式。不会阻塞地等待消息。它拥有最好的并发性和可扩展性。

```scala
actorRef ! message
```
如果是在一个Actor中调用，那么发送方的actor引用会被隐式地作为消息的`sender(): ActorRef`成员一起发送。目的actor可以使用它来向原actor发送回应，使用`sender() ! replyMsg`。

如果不是从Actor实例发送的，sender缺省为`deadLetters` actor引用。

### Ask: Send-And-Receive-Future

ask模式既包含actor也包含future, 所以它是作为一种使用模式，而不是ActorRef的方法:

```scala
import akka.pattern.{ ask, pipe }
import system.dispatcher // The ExecutionContext that will be used
case class Result(x: Int, s: String, d: Double)
case object Request
implicit val timeout = Timeout(5 seconds) // needed for ‘?‘ below
val f: Future[Result] =
  for {
    x <- ask(actorA, Request).mapTo[Int] // call pattern directly
    s <- (actorB ask Request).mapTo[String] // call by implicit conversion
    d <- (actorC ? Request).mapTo[Double] // call by symbolic name
  } yield Result(x, s, d)
f pipeTo actorD // .. or ..
pipe(f) to actorD
```
上面的例子展示了将 ask 与future上的`pipeTo`模式一起使用，因为这是一种非常常用的组合。 请注意上面所有的调用都是完全非阻塞和异步的：`ask` 产生 `Future`, 三个Future通过for-语法组合成一个新的Future，然后用`pipeTo`在future上安装一个onComplete-处理器来完成将收集到的`Result`发送到其它actor的动作。

使用`ask`将会象`tell`一样发送消息给接收方, 接收方必须通过`sender ! reply`发送回应来为返回的`Future`填充数据。`ask`操作包括创建一个内部actor来处理回应，必须为这个内部actor指定一个超时期限，过了超时期限内部actor将被销毁以防止内存泄露。

*注意：如果要以异常来填充future你需要发送一个 Failure 消息给发送方。这个操作不会在actor处理消息发生异常时自动完成。*

```scala
try {
  val result = operation()
  sender() ! result
} catch {
  case e: Exception =>
    sender() ! akka.actor.Status.Failure(e)
throw e }
```
如果一个actor没有完成future, 它会在超时时限到来时过期， 以`AskTimeoutException`来结束。超时的时限是按下面的顺序和位置来获取的:

- 显式指定超时:

```scala
￼import scala.concurrent.duration._
 import akka.pattern.ask
 val future = myActor.ask("hello")(5 seconds)
```
- 提供类型为`akka.util.Timeout`的隐式参数, 例如，

```scala
import scala.concurrent.duration._
import akka.util.Timeout
import akka.pattern.ask
implicit val timeout = Timeout(5 seconds)
val future = myActor ? "hello"
```

Future的`onComplete`, `onResult`, 或 `onTimeout`方法可以用来注册一个回调，以便在Future完成时得到通知。从而提供一种避免阻塞的方法。

*注意：在使用future回调如`onComplete`, `onSuccess`,和`onFailure`时, 在actor内部你要小心避免捕捉该actor的引用, 也就是不要在回调中调用该actor的方法或访问其可变状态。这会破坏actor的封装，会引起同步bug和race condition, 因为回调会与此actor一同被并发调度。不幸的是目前还没有一种编译时的方法能够探测到这种非法访问。*

### 转发消息

你可以将消息从一个actor转发给另一个。虽然经过了一个‘中转’，但最初的发送者地址/引用将保持不变。当实现功能类似路由器、负载均衡器、备份等的actor时会很有用。

```scala
target forward message
```
## 6 接收消息

Actor必须实现`receive`方法来接收消息：
```scala
￼type Receive = PartialFunction[Any, Unit]
 def receive: Actor.Receive
```
这个方法应返回一个`PartialFunction`, 例如 一个 ‘match/case’ 子句，消息可以与其中的不同分支进行scala模式匹配。如下例:

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

## 7 回应消息

如果你需要一个用来发送回应消息的目标，可以使用`sender()`, 它是一个Actor引用. 你可以用`sender() ! replyMsg` 向这个引用发送回应消息。你也可以将这个Actor引用保存起来将来再作回应。如果没有`sender()` (不是从actor发送的消息或者没有future上下文) 那么` sender() `缺省为 ‘死信’ actor的引用。

```scala
￼￼case request =>
   val result = process(request)
   sender() ! result       // will have dead-letter actor as default
```

## 8 接收超时

在接收消息时，如果在一段时间内没有收到第一条消息，可以使用超时机制。 要检测这种超时你必须设置 `receiveTimeout` 属性并声明一个处理`ReceiveTimeout`对象的匹配分支。

```scala
import akka.actor.ReceiveTimeout
import scala.concurrent.duration._
class MyActor extends Actor {
  // To set an initial delay
  context.setReceiveTimeout(30 milliseconds)
  def receive = {
    case "Hello" =>
      // To set in a response to a message
      context.setReceiveTimeout(100 milliseconds)
    case ReceiveTimeout =>
      // To turn it off
      context.setReceiveTimeout(Duration.Undefined)
      throw new RuntimeException("Receive timed out")
} }
```

## 9 终止Actor

通过调用`ActorRefFactory` 也就是`ActorContext` 或 `ActorSystem` 的stop方法来终止一个actor , 通常`context`用来终止子actor，而`system`用来终止顶级actor。实际的终止操作是异步执行的，也就是说stop可能在actor被终止之前返回。

如果当前有正在处理的消息，对该消息的处理将在actor被终止之前完成，但是邮箱中的后续消息将不会被处理。缺省情况下这些消息会被送到 ActorSystem 的死信, 但是这取决于邮箱的实现。

actor的终止分两步: 第一步actor将停止对邮箱的处理，向所有子actor发送终止命令，然后处理来自子actor的终止消息直到所有的子actor都完成终止， 最后终止自己 (调用`postStop`, 销毁邮箱, 向DeathWatch发布`Terminated`, 通知其监管者). 这个过程保证actor系统中的子树以一种有序的方式终止, 将终止命令传播到叶子结点并收集它们回送的确认消息给被终止的监管者。如果其中某个actor没有响应 (由于处理消息用了太长时间以至于没有收到终止命令), 整个过程将会被阻塞。

在`ActorSystem.shutdown`被调用时, 系统根监管actor会被终止，以上的过程将保证整个系统的正确终止。

`postStop` hook 是在actor被完全终止以后调用的。这是为了清理资源:

```scala
￼override def postStop() {
  // clean up some resources ...
}
```

*注意：由于actor的终止是异步的, 你不能马上使用你刚刚终止的子actor的名字；这会导致`InvalidActorNameException`. 你应该监视正在终止的 actor 而在最终到达的`Terminated`消息的处理中创建它的替代者。*

### PoisonPill

你也可以向actor发送`akka.actor.PoisonPill`消息, 这个消息处理完成后actor会被终止。`PoisonPill`与普通消息一样被放进队列，因此会在已经入队列的其它消息之后被执行。

### 优雅地终止

如果你想等待终止过程的结束，或者组合若干actor的终止次序，可以使用gracefulStop:

```scala
import akka.pattern.gracefulStop
import scala.concurrent.Await
try {
  val stopped: Future[Boolean] = gracefulStop(actorRef, 5 seconds, Manager.Shutdown)
  Await.result(stopped, 6 seconds)
  // the actor has been stopped
} catch {
  // the actor wasn’t stopped within 5 seconds
  case e: akka.pattern.AskTimeoutException =>
}
```

```scala
object Manager {
  case object Shutdown
}
class Manager extends Actor {
  import Manager._
  val worker = context.watch(context.actorOf(Props[Cruncher], "worker"))
  def receive = {
    case "job" => worker ! "crunch"
    case Shutdown =>
      worker ! PoisonPill
      context become shuttingDown
}
  def shuttingDown: Receive = {
    case "job" => sender() ! "service unavailable, shutting down"
    case Terminated(‘worker‘) =>
      context stop self
} }
```

## 10 Become/Unbecome

### 升级

Akka支持在运行时对Actor消息循环的实现进行实时替换: 在actor中调用`context.become`方法。Become 要求一个 `PartialFunction[Any, Unit]`参数作为新的消息处理实现。被替换的代码被存在一个栈中，可以被push和pop。

*请注意actor被其监管者重启后将恢复其最初的行为。*

使用 become 替换Actor的行为:

```scala
class HotSwapActor extends Actor {
  import context._
  def angry: Receive = {
    case "foo" => sender() ! "I am already angry?"
    case "bar" => become(happy)
  }
  def happy: Receive = {
    case "bar" => sender() ! "I am already happy :-)"
    case "foo" => become(angry)
}
  def receive = {
    case "foo" => become(angry)
    case "bar" => become(happy)
} }
```

`become` 方法还有很多其它的用处，一个特别好的例子是用它来实现一个有限状态机 (FSM)。


以下是另外一个使用 become和unbecome的例子:

```scala
case object Swap
class Swapper extends Actor {
  import context._
  val log = Logging(system, this)
  def receive = {
    case Swap =>
      log.info("Hi")
      become({
        case Swap =>
          log.info("Ho")
          unbecome() // resets the latest ’become’ (just for fun)
      }, discardOld = false) // push on top instead of replace
  }
}
object SwapperApp extends App {
  val system = ActorSystem("SwapperSystem")
  val swap = system.actorOf(Props[Swapper], name = "swapper")
  swap ! Swap // logs Hi
  swap ! Swap // logs Ho
  swap ! Swap // logs Hi
  swap ! Swap // logs Ho
  swap ! Swap // logs Hi
  swap ! Swap // logs Ho
}
```

## 11 Stash

`Stash` trait可以允许一个actor临时存储消息，这些消息不能或者不应被当前的行为处理。通过改变actor的消息处理，也就是调用`context.become`或者`context.unbecome`,所有存储的消息都会`unstashed`，从而将它们预先放入邮箱中。通过这种方法，可以与接收消息时相同的顺序处理存储的消息。

*注意：Stash trait继承自标记trait `RequiresMessageQueue[DequeBasedMessageQueueSemantics]`。这个trait需要系统自动的选择一个基于邮箱实现的双端队列给actor。*

如下是一个例子：

```scala
import akka.actor.Stash
class ActorWithProtocol extends Actor with Stash {
  def receive = {
    case "open" =>
      unstashAll()
      context.become({
        case "write" => // do writing...
        case "close" =>
          unstashAll()
          context.unbecome()
        case msg => stash()
      }, discardOld = false) // stack on top instead of replacing
    case msg => stash()
} }
```
调用`stash()`添加当前消息到actor的stash中。通常情况下，actor的消息处理默认case会调用它处理其它case无法处理的stash消息。相同的消息stash两次是不对的，它会导致一个`IllegalStateException`异常被抛出。stash也是可能有边界的，在这种情况下，调用`stash()`可能导致容量问题，抛出`StashOverflowException`异常。可以通过在邮箱的配置文件中配置`stash-capacity`选项来设置容量。

调用`unstashAll()`从stash中取数据到邮箱中，直至邮箱容量（如果有）满为止。在这种情况下，有边界的邮箱会溢出，抛出`MessageQueueAppendFailedException`异常。调用`unstashAll()`后，stash可以确保为空。

stash保存在`scala.collection.immutable.Vector`中。即使大数量的消息要stash，也不会存在性能问题。

## 12 杀死actor

你可以发送` Kill`消息来杀死actor。这会让actor抛出一个`ActorKilledException`异常，触发一个错误。actor将会暂停它的操作，并且询问它的监控器如何处理错误。这可能意味着恢复actor、重启actor或者完全终止它。

用下面的方式使用`Kill`。

```scala
￼// kill the ’victim’ actor
 victim ! Kill
```

## 13 Actor 与异常

在消息被actor处理的过程中可能会抛出异常，例如数据库异常。

### 消息会怎样

如果消息处理过程中（即从邮箱中取出并交给receive后）发生了异常，这个消息将被丢失。必须明白它不会被放回到邮箱中。所以如果你希望重试对消息的处理，你需要自己捕捉异常然后在异常处理流程中重试. 请确保你限制重试的次数，因为你不会希望系统产生活锁 (从而消耗大量CPU而于事无补)。

### 邮箱会怎样

如果消息处理过程中发生异常，邮箱没有任何变化。如果actor被重启，邮箱会被保留。邮箱中的所有消息不会丢失。

### actor会怎样

如果抛出了异常，actor将会被暂停，监控过程将会开始。依据监控器的决定，actor会恢复、重试或者终止。

## 14 使用 PartialFunction 链来扩展actor

有时，在几个actor之间共享相同的行为或者用多个更小的函数组成一个actor的行为是非常有用的。因为一个actor的receive方法返回一个`Actor.Receive`,所以这是可能实现的。`Actor.Receive`是`PartialFunction[Any,Unit]`的类型别名，偏函数可以用`PartialFunction#orElse`方法链接在一起。你可以链接任意多的函数，但是你应该记住“first match”将会赢-当这些合并的函数均可以处理同类型的消息时，这很重要。

例如，假设你有一组actor，要么是`Producers`，要么是`Consumers`。有时候，一个actor拥有这两者的行为是有意义的。可以在不重复代码的情况下简单的实现该目的。那就是，抽取行为到trait中，实现actor的receive函数当作这些偏函数的合并。

```scala
trait ProducerBehavior {
  this: Actor =>
  val producerBehavior: Receive = {
    case GiveMeThings =>
      sender() ! Give("thing")
  }
}
trait ConsumerBehavior {
  this: Actor with ActorLogging =>
  val consumerBehavior: Receive = {
    case ref: ActorRef =>
      ref ! GiveMeThings
    case Give(thing) =>
      log.info("Got a thing! It’s {}", thing)
} }
class Producer extends Actor with ProducerBehavior {
  def receive = producerBehavior
}
class Consumer extends Actor with ActorLogging with ConsumerBehavior {
  def receive = consumerBehavior
}
class ProducerConsumer extends Actor with ActorLogging
  with ProducerBehavior with ConsumerBehavior {
  def receive = producerBehavior orElse consumerBehavior
}
// protocol
case object GiveMeThings
case class Give(thing: Any)
```
## 15 初始化模式

actor丰富的生命周期hooks提供了一个有用的工具包实现多个初始化模式。在`ActorRef`的生命中，一个actor可能经历多次重启，旧的实例被新的实例替换，这对外部的观察者来说是不可见的，它们只能看到`ActorRef`。

人们可能会想到作为"incarnations"的新对象，一个actor的每个incarnation的初始化可能也是必须的，但是有些人可能希望当`ActorRef`创建时，初始化仅仅发生在第一个实例出生时。下面介绍几种不同的初始化模式。

### 通过构造器初始化

用构造器初始化有几个好处。第一，它使用val保存状态成为可能，这个状态在actor实例的生命期内不会改变。这使actor的实现更有鲁棒性。actor的每一个incarnation都会调用构造器，因此，在actor的内部组件中，总是假设正确的初始化已经发生了。这也是这种方法的缺点，因为有这样的场景，人们不希望在重启时重新初始化actor内部组件。例如，这种方法在重启时保护子actor非常有用。下面将介绍适合这种场景的模式。

### 通过preStart初始化

在初始化第一个actor实例时，actor的`preStart()`方法仅仅被直接调用一次，那就是创建它的`ActorRef`。在重启的情况下，`preStart()`在`postRestart()`中调用，因此，如果不重写，`preStart()`会在每一个incarnation中调用。然而，我们可以重写`postRestart()`禁用这个行为，保证只有一个调用`preStart()`。

这中模式有用之处是，重启时它禁止为子actors创建新的`ActorRefs`。这可以通过重写来实现：

```scala
override def preStart(): Unit = {
  // Initialize children here
}
// Overriding postRestart to disable the call to preStart()
// after restarts
override def postRestart(reason: Throwable): Unit = ()
// The default implementation of preRestart() stops all the children
// of the actor. To opt-out from stopping the children, we
// have to override preRestart()
override def preRestart(reason: Throwable, message: Option[Any]): Unit = {
  // Keep the call to postStop(), but no stopping of children
postStop() }
```
注意，子actors仍然重启了，但是没有创建新的`ActorRefs`。人们可以为子actor递归的才有这个准则，保证`preStart()`只在它们的refs创建时被调用一次。

### 通过消息传递初始化

不可能在构造器中为actor的初始化传递所有的必须的信息，例如循环依赖。在这种情况下，actor应该监听初始化消息，用`become()`或者有限状态机状态转换器去编码actor初始的或者未初始的状态。

```scala
var initializeMe: Option[String] = None
override def receive = {
  case "init" =>
    initializeMe = Some("Up and running")
    context.become(initialized, discardOld = true)
}
def initialized: Receive = {
  case "U OK?" => initializeMe foreach { sender() ! _ }
}
```

如果actor在初始化之前收到了消息，一个有用的工具`Stash`可以保存消息直到初始化完成，待初始化完成之后，重新处理它们。

*注意：这种模式应该小心使用，只有在以上模式不可用是使用。一个潜在的问题是，当发送到远程actor时，消息有可能丢失。另外，在一个未初始化的状态下发布一个`ActorRefs`可能导致它在初始化之前接收一个用户消息。*









