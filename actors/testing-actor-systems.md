# 测试actor系统

## 1 Testkit实例

这是Ray Roestenburg 在[他的博客](http://roestenburg.agilesquad.com/2011/02/unit-testing-akka-actors-with-testkit_12.html)中的示例代码， 作了改动以兼容 Akka 2.x。

```scala
import scala.util.Random
import org.scalatest.BeforeAndAfterAll
import org.scalatest.WordSpecLike
import org.scalatest.Matchers
import com.typesafe.config.ConfigFactory
import akka.actor.Actor
import akka.actor.ActorRef
import akka.actor.ActorSystem
import akka.actor.Props
import akka.testkit.{ TestActors, DefaultTimeout, ImplicitSender, TestKit }
import scala.concurrent.duration._
import scala.collection.immutable

//演示TestKit的示例测试
class TestKitUsageSpec
  extends TestKit(ActorSystem("TestKitUsageSpec",
    ConfigFactory.parseString(TestKitUsageSpec.config)))
  with DefaultTimeout with ImplicitSender
  with WordSpecLike with Matchers with BeforeAndAfterAll {
  import TestKitUsageSpec._
  val echoRef = system.actorOf(TestActors.echoActorProps)
  val forwardRef = system.actorOf(Props(classOf[ForwardingActor], testActor))
  val filterRef = system.actorOf(Props(classOf[FilteringActor], testActor))
  val randomHead = Random.nextInt(6)
  val randomTail = Random.nextInt(10)
  val headList = immutable.Seq().padTo(randomHead, "0")
  val tailList = immutable.Seq().padTo(randomTail, "1")
  val seqRef =system.actorOf(Props(classOf[SequencingActor], testActor, headList, tailList))
  override def afterAll {
    shutdown()
  }
  
  "An EchoActor" should {
      "Respond with the same message it receives" in {
        within(500 millis) {
          echoRef ! "test"
          expectMsg("test")
        }
      }
  }
    
  "A ForwardingActor" should {
      "Forward a message it receives" in {
        within(500 millis) {
          forwardRef ! "test"
          expectMsg("test")
        }
      } 
  }
  
  "A FilteringActor" should {
      "Filter all messages, except expected messagetypes it receives" in {
        var messages = Seq[String]()
        within(500 millis) {
          filterRef ! "test"
          expectMsg("test")
          filterRef ! 1
          expectNoMsg
          filterRef ! "some"
          filterRef ! "more"
          filterRef ! 1
          filterRef ! "text"
          filterRef ! 1
          receiveWhile(500 millis) {
            case msg: String => messages = msg +: messages
  } }
        messages.length should be(3)
        messages.reverse should be(Seq("some", "more", "text"))
      }
  }
  
  "A SequencingActor" should {
      "receive an interesting message at some point " in {
        within(500 millis) {
          ignoreMsg {
            case msg: String => msg != "something"
          }
          seqRef ! "something"
          expectMsg("something")
          ignoreMsg {
            case msg: String => msg == "1"
          }
          expectNoMsg
          ignoreNoMsg
        }
      }
  }
}

object TestKitUsageSpec {
  // Define your test specific configuration here
  val config = """
    akka {
      loglevel = "WARNING"
  } """
  
  //将所有消息转发给另一个Actor的actor
  class ForwardingActor(next: ActorRef) extends Actor {
    def receive = {
      case msg => next ! msg
    }
  }
  //仅转发一部分消息给另一个Actor的Actor
  class FilteringActor(next: ActorRef) extends Actor {
      def receive = {
        case msg: String => next ! msg
        case _           => None
      }
  }
  //此actor发送一个消息序列，此序列具有随机列表作为头，一个关心的值及一个随机列表作为尾
  //想法是你想测试接收到的关心的值，而不用处理其它部分
  
 class SequencingActor(next: ActorRef, head: immutable.Seq[String],
                         tail: immutable.Seq[String]) extends Actor {
     def receive = {
       case msg => {
         head foreach { next ! _ }
         next ! msg
         tail foreach { next ! _ }
       }
     }
 } 
}

```

对于任何软件开发，自动化测试都是开发过程的一个重要组成部分。actor 模型对于代码单元如何划分，它们之间如何交互提供了一种新的视角， 这对如何编写测试也造成了影响。

Akka 有一个专门的` akka-testkit `模块来支持不同层次上的测试, 很明显共有两个类别:

- 测试独立的不包括actor模型的代码，即没有多线程的内容；在事件发生的次序方面有完全确定性的行为，没有任何并发考虑， 这在下文中称为`单元测试（Unit Testing）。`
- 测试（多个）包装过的actor，包括多线程调度; 事件的次序没有确定性但由于使用了actor模型，不需要考虑并发，这在下文中被称为`集成测试（Integration Testing）。`

当然这两个类型有着不同的粒度, 单元测试通常是白盒测试而集成测试是对完整的actor网络进行的功能测试。 其中重要的区别是并发的考虑是否是测试是一部分。我们提供的工具将在下面的章节中详细介绍。

## 2 用TestActorRef做同步单元测试

测试Actor 类中的业务逻辑分为两部分: 首先，每个原子操作必须独立动作，然后输入的事件序列必须被正确处理, 即使事件的次序存在一些可能的变化。 前者是单线程单元测试的主要使用场景，而后者可以在集成测试中进行确认。

通常，` ActorRef `将实际的` Actor `实例与外界隔离开, 唯一的通信通道是actor的邮箱。 这个限制是单元测试的障碍，所以我们引进了`TestActorRef`。这个特殊类型的引用是专门为测试设计的，它允许以两种方式访问actor: 通过获取实际actor实例的引用，通过调用或查询actor的行为(receive)。 以下每一种方式都有专门的部分介绍。

### 获取 Actor 的引用

能够访问到实际的 Actor 对象使得所有传统的单元测试方法可以用于测试其中的方法。 获取引用的方法:

```scala
import akka.testkit.TestActorRef
val actorRef = TestActorRef[MyActor]
val actor = actorRef.underlyingActor
```

由于` TestActorRef `是actor类型的高阶类型，它返回实际actor及其正确的静态类型。这之后你就可以像平常一样将你的任何单元测试工具用于你的actor。

### 测试有限状态机

如果你要测试的actor是一个` FSM`, 你可以使用专门的` TestFSMRef`，它拥有普通` TestActorRef `的所有功能，并且能够访问其内部状态:

```scala
import akka.testkit.TestFSMRef
import akka.actor.FSM
import akka.util.duration._
 
val fsm = TestFSMRef(new Actor with FSM[Int, String] {
  startWith(1, "")
  when(1) {
    case Event("go", _) => goto(2) using "go"
  }
  when(2) {
    case Event("back", _) => goto(1) using "back"
  }
})
 
assert(fsm.stateName == 1)
assert(fsm.stateData == "")
fsm ! "go" // being a TestActorRef, this runs also on the CallingThreadDispatcher
assert(fsm.stateName == 2)
assert(fsm.stateData == "go")
 
fsm.setState(stateName = 1)
assert(fsm.stateName == 1)
 
assert(fsm.timerActive_?("test") == false)
fsm.setTimer("test", 12, 10 millis, true)
assert(fsm.timerActive_?("test") == true)
fsm.cancelTimer("test")
assert(fsm.timerActive_?("test") == false)
```

由于Scala类型推测的限制，只有一个如上所示的工厂方法，所以你可能需要写象` TestFSMRef(new MyFSM) `这样的代码，而不是想象中的类似`ActorRef`的`TestFSMRef[MyFSM]`。上例所示的所有方法都直接访问FSM的状态，不作任何同步；这在使用` CallingThreadDispatcher `(TestFSMRef缺省使用它) 并且没有其它线程参与的情况下是合适的, 但如果你实际上需要处理定时器事件可能会导致意外的情形，因为它们是在` Scheduler `线程中执行的。

### 测试Actor的行为

当消息派发器调用actor中的逻辑来处理消息时，它实际上是对当前注册到actor的行为进行了应用。行为的初始值是代码中声明的` receive `方法的返回值, 但可以通过对外部消息的响应调用` become `和` unbecome `来改变这个行为。所有这些特性使得actor的行为测试起来不太容易。因此` TestActorRef `提供了一种不同的操作方式来对Actor的测试进行补充: 它支持所有正常的` ActorRef`中的操作。发往actor的消息在当前线程中同步处理，应答像正常一样回送。这个技巧来自下面所介绍的` CallingThreadDispatcher`; 这个派发器被隐式地用于所有实例化为`TestActorRef`的actor。

```scala
import akka.testkit.TestActorRef
import scala.concurrent.duration._
import scala.concurrent.Await
import akka.pattern.ask
val actorRef = TestActorRef(new MyActor)
// hypothetical message stimulating a ’42’ answer
val future = actorRef ? Say42
val Success(result: Int) = future.value.get
result should be(42)
```

由于` TestActorRef` 是 `LocalActorRef` 的子类，只不过多加了一些特殊功能，所以像监管和重启也能正常工作，但是要知道只要所有的相关的actors都使用` CallingThreadDispatcher`那么所有的执行过程都是严格同步的。 一旦你增加了一些元素，其中包括比较复杂的定时任务，你就离开了单元测试的范畴，因为你必须要重新将异步性纳入考虑范围（在大多数情况下问题在于要等待希望的结果有机会发生）。

另一个在单线程测试中被覆盖的特殊点是` receiveTimeout`, 由于包含了它会产生异步的` ReceiveTimeout `消息队列, 因此与同步约定矛盾。

> *综上所述: TestActorRef 重写了两个成员: 它设置派发器为` CallingThreadDispatcher.global `，设置`receiveTimeout`为None*

### 介于两者之间方法

如果你希望测试actor的行为，包括热替换，但是不包括消息派发器，也不希望` TestActorRef `吞掉所有抛出的异常, 那么为你准备了另一种模式: 只要使用` TestActorRef `的` receive `方法 , 这将会把消息转发给内部的actor:

```scala
import akka.testkit.TestActorRef
val actorRef = TestActorRef(new Actor {
  def receive = {
    case "hello" => throw new IllegalArgumentException("boom")
  }
})
intercept[IllegalArgumentException] { actorRef.receive("hello") }
```

### 使用场景

当然你也可以根据自己的测试需求来混合使用` TestActorRef `的不同用法:

- 一个常见的使用场景是在发送测试消息之前设置actor进入某个特定的内部状态
- 另一个场景是发送了测试消息之后确认正确的内部状态转换

放心大胆地对各种可能性进行实验，如果你发现了有用的模式，快让Akka论坛知道它!常用操作也许能放进优美的DSL中。

## 3 用TestKit进行并发的集成测试

当你基本确定你的actor的业务逻辑是正确的, 下一个步骤就是确认它在目标环境中也能正确工作 (如果actor分别都比较简单，可能是因为他们使用了 FSM 模块, 这也可能是第一步)。关于环境的定义当然很大程度上由手上的问题和你打算测试的程度决定, 从功能/集成测试到完整的系统测试。 最简单的步骤包括测试流程（说明测试条件）、要测试的actor和接收应答的actor。大一些的系统将被测的actor替换成一组actor网络，将测试条件应用于不同的切入点，并整理将会从不同的输出位置发送的结果，但基本的原则仍然是测试由一个单独的流程来驱动。

`TestKit `类包含一组工具来简化这些常用的工作。

```scala
import akka.actor.ActorSystem
import akka.actor.Actor
import akka.actor.Props
import akka.testkit.{ TestActors, TestKit, ImplicitSender }
import org.scalatest.WordSpecLike
import org.scalatest.Matchers
import org.scalatest.BeforeAndAfterAll
class MySpec(_system: ActorSystem) extends TestKit(_system) with ImplicitSender
  with WordSpecLike with Matchers with BeforeAndAfterAll {
  def this() = this(ActorSystem("MySpec"))
  override def afterAll {
    TestKit.shutdownActorSystem(system)
  }
  "An Echo actor" must {
    "send back messages unchanged" in {
      val echo = system.actorOf(TestActors.echoActorProps)
      echo ! "hello world"
      expectMsg("hello world")
}
} }
```

TestKit 中有一个名为` testActor `的actor作为将要被不同的 `expectMsg...`断言检查的消息的入口，下面会详细介绍这些断言。 当混入了` ImplicitSender` trait后 这个actor在从测试过程中派发消息时将被隐式地用作发送引用。` testActor `也可以被像平常一样发送给其它的actor，通常是订阅成为通知监听器。 有一堆检查方法, 如接收所有匹配某些条件的消息，接收固定的消息序列或类，在某段时间内收不到消息等。

记得在测试完成后关闭actor系统 (即使是在测试失败的情况下) 以保证所有的actor-包括测试actor-被停止。

### 内置断言

上面提到的` expectMsg `并不是唯一的对收到的消息进行断言的方法。 以下是完整的列表:

- `expectMsg[T](d: Duration, msg: T): T`

给定的消息必须在指定的时间内到达；返回此消息

- `expectMsgPF[T](d: Duration)(pf: PartialFunction[Any, T]): T`

在给定的时间内，必须有消息到达，必须为这类消息定义偏函数；返回应用收到消息的偏函数的结果。可以不指定时间段（这时需要一对空的括号），这时使用最深层的[within]()块中的期限。

- `expectMsgClass[T](d: Duration, c: Class[T]): T`

在指定的时间内必须接收到` Class `类型的对象；返回收到的对象。注意它的类型匹配是子类兼容的；如果需要类型是相等的，参考使用单个class参数的` expectMsgAllClassOf`。

- `expectMsgType[T: Manifest](d: Duration)`

在指定的时间内必须收到指定类型 (擦除后)的对象; 返回收到的对象。这个方法基本上与`expectMsgClass(manifest[T].erasure)`等价。

- `expectMsgAnyOf[T](d: Duration, obj: T*): T`

在指定的时间内必须收到一个对象，而且此对象必须与传入的对象引用中的一个相等( 用 == 进行比较); 返回收到的对象。

- `expectMsgAnyClassOf[T](d: Duration, obj: Class[_ <: T]*): T`

在指定的时间内必须收到一个对象，它必须至少是指定的某` Class `对象的实例; 返回收到的对象。

- `expectMsgAllOf[T](d: Duration, obj: T*): Seq[T]`

在指定时间内必须收到与指定的数组中相等数量的对象, 对每个收到的对象，必须至少有一个数组中的对象与它相等(用==进行比较)。 返回收到的整个对象集合。

- `expectMsgAllClassOf[T](d: Duration, c: Class[_ <: T]*): Seq[T]`

在指定时间内必须收到与指定的` Class `数组中相等数量的对象，对数组中的每一个`Class`， 必须至少有一个对象的Class与它相等(用 ==进行比较) (这不是子类兼容的类型检查)。 返回收到的整个对象集合。

- `expectMsgAllConformingOf[T](d: Duration, c: Class[_ <: T]*): Seq[T]`

在指定时间内必须收到与指定的 Class 数组中相等数量的对象，对数组中的每个Class 必须至少有一个对象是这个Class的实例。返回收到的整个对象集合。

- `expectNoMsg(d: Duration)`

在指定时间内不能收到消息。如果在这个方法被调用之前已经收到了消息，并且没有用其它的方法将这些消息从队列中删除，这个断言也会失败。

- `receiveN(n: Int, d: Duration): Seq[AnyRef]`

指定的时间内必须收到n 条消息; 返回收到的消息。

- `fishForMessage(max: Duration, hint: String)(pf: PartialFunction[Any, Boolean]): Any`

只要时间没有用完，并且偏函数匹配消息并返回false就一直接收消息. 返回使偏函数返回true 的消息或抛出异常, 异常中会提供一些提示以供debug使用。

除了接收消息的断言，还有一些方法来对消息流提供帮助:

- `receiveOne(d: Duration): AnyRef`

尝试等待给定的时间以等待收到一个消息，如果失败则返回null 。 如果给定的 Duration 是0，这一调用是非阻塞的(轮询模式)。

- `receiveWhile[T](max: Duration, idle: Duration, messages: Int)(pf: PartialFunction[Any, T]): Seq[T]`

只要满足

    - 消息与偏函数匹配
    - 指定的时间还没用完
    - 在空闲的时间内收到了下一条消息
    - 消息数量还没有到上限
    
就收集消息并返回收集到的所有消息。 时间上限缺省值是最深层的within块中剩余的时间，空闲时间缺省为无限 (也就是禁止空闲超时功能)。 期望的消息数量缺省值为` Int.MaxValue`, 也就是不作这个限制。

- `awaitCond(p: => Boolean, max: Duration, interval: Duration)`

每经过` interval `时间就检查一下给定的条件，直到它返回` true `或者` max `时间用完了. 时间间隔缺省为100 ms而最大值缺省为最深层的within块中的剩余时间。

- `ignoreMsg(pf: PartialFunction[AnyRef, Boolean])`  `ignoreNoMsg`

内部的` testActor` 包含一个偏函数用来忽略消息: 它只会将与偏函数不匹配或使函数返回false的消息放进队列。 这个函数可以用上面的方法进行设置和重设; 每一次调用都会覆盖之前的函数，而不会迭加。

这个功能在你想到忽略正常的消息而只对你指定的一些消息感兴趣时（如测试日志系统时）比较有用。

### 预料的异常

由于集成测试无法进入参与测试的actor的内部处理流程, 无法直接确认预料中的异常。为了做这件事，只能使用日志系统：将普通的事件处理器替换成` TestEventListener `然后使用` EventFilter `可以对日志信息，包括由于异常产生的日志，做断言:

```scala
import akka.testkit.EventFilter
import com.typesafe.config.ConfigFactory
implicit val system = ActorSystem("testsystem", ConfigFactory.parseString("""
  akka.loggers = ["akka.testkit.TestEventListener"]
  """))
try {
  val actor = system.actorOf(Props.empty)
  EventFilter[ActorKilledException](occurrences = 1) intercept {
actor ! Kill }
} finally {
  shutdown(system)
}
```

如果`occurrences`指定了大小，那么`intercept`将会阻塞直到接收到匹配数量的消息或者超过`akka.test.filter-leeway`中配置的时间。超时时，测试将会失败。

### 对定时进行断言

功能测试的另一个重要部分与定时器有关：有些事件不能立即发生（如定时器）, 另外一些需要在时间期限内发生。 因此所有的进行检查的方法都接受一个时间上限，不论是正面还是负面的结果都应该在这个时间之前获得。时间下限需要在这个检测方法之外进行检查，我们有一个新的工具来管理时间期限:

```scala
within([min, ]max) {
  ...
}
```

`within`所带的代码块必须在一个介于` min `和` max`之间的`Duration`之前完成, 其中min缺省值为0。将`max `参数与块的启动时间相加得到的时间期限在所有检查方法块内部都可以隐式得获得，如果你没有指定max值，它会从最深层的` within `块继承这个值。

应注意如果代码块的最后一条接收消息断言是` expectNoMsg `或 `receiveWhile`, 对` within `的最终检查将被跳过，以避免由于唤醒延迟导致的假正值（false positives）。这意味着虽然其中每一个独立的断言仍然使用时间上限，整个代码块在这种情况下会有长度随机的延迟。

```scala
import akka.actor.Props
import akka.util.duration._
 
val worker = system.actorOf(Props[Worker])
within(200 millis) {
  worker ! "some work"
  expectMsg("some result")
  expectNoMsg // 在剩下的200ms中会阻塞
  Thread.sleep(300) // 不会使当前代码块失败
}
```

> *所有的时间都以 System.nanoTime为单位, 它们描述的是墙上时间，而非CPU时间。*

Ray Roestenburg 写了一篇关于使用 TestKit 的好文:[ http://roestenburg.agilesquad.com/2011/02/unit-testing-akka-actors-with-testkit_12.html](http://roestenburg.agilesquad.com/2011/02/unit-testing-akka-actors-with-testkit_12.html). 完整的示例也可以在第一章找到。

#### 考虑很慢的测试系统

你在跑得飞快的笔记本上使用的超时设置在高负载的Jenkins（或类似的）服务器上通常都会导致错误的测试失败。为了考虑这种情况，所有的时间上限都在内部乘以一个系数，这个系数来自配置文件中的` akka.test.timefactor`, 缺省值为 1。

你也可以用`akka.testkit`包对象中的隐式转换来将同样的系数来作用于其它的时限，为Duration添加dilated函数。

```scala
import scala.concurrent.duration._
import akka.testkit._
10.milliseconds.dilated
```

### 用隐式的ActorRef解决冲突

如果你希望在基于TestKit的测试的消息发送者为` testActor`， 只需要为你的测试代码混入` ImplicitSender`。

```scala
￼class MySpec(_system: ActorSystem) extends TestKit(_system) with ImplicitSender
  with WordSpecLike with Matchers with BeforeAndAfterAll {
```

### 使用多个探针 Actors

如果待测的actor会发送多个消息到不同的目标，在使用TestKit时可能会难以分辨到达` testActor`的消息流。 另一种方法是用它来创建简单的探针actor，将它们插入到消息流中。 为了让这种方法更加强大和方便，我们提供了一个具体实现，称为` TestProbe`。 它的功能可以用下面的小例子说明:

```scala
import scala.concurrent.duration._
import akka.actor._
import scala.concurrent.Future
class MyDoubleEcho extends Actor {
  var dest1: ActorRef = _
  var dest2: ActorRef = _
  def receive = {
    case (d1: ActorRef, d2: ActorRef) =>
      dest1 = d1
      dest2 = d2
    case x =>
      dest1 ! x
      dest2 ! x
} }

val probe1 = TestProbe()
val probe2 = TestProbe()
val actor = system.actorOf(Props[MyDoubleEcho])
actor ! ((probe1.ref, probe2.ref))
actor ! "hello"
probe1.expectMsg(500 millis, "hello")
probe2.expectMsg(500 millis, "hello")
```

这里我们用` MyDoubleEcho`来仿真一个待测系统, 它会将输入镜像为两个输出。关联两个测试探针来进行（最简单）行为的确认。 还有一个例子是两个actor A，B， A 发送消息给 B。 为了确认这个消息流，可以插入` TestProbe `作为A的目标, 使用转发功能或下文中的自动导向功能在测试上下文中包含真实的B.

还可以为探针配备自定义的断言来使测试代码更简洁清晰:

```scala
case class Update(id: Int, value: String)
val probe = new TestProbe(system) {
  def expectUpdate(x: Int) = {
    expectMsgPF() {
      case Update(id, _) if id == x => true
    }
    sender() ! "ACK"
  }
}
```

这里你拥有完全的灵活性，可以将TestKit 提供的工具与你自己的检测代码混合和匹配，并为它取一个有意义的名字。在实际中你的代码可能比上面的示例要复杂；要充分利用工具！

#### 对探针收到的消息进行应答

探针在可能的条件下，会记录通讯通道以便进行应答:

```scala
val probe = TestProbe()
val future = probe.ref ? "hello"
probe.expectMsg(0 millis, "hello") // TestActor runs on CallingThreadDispatcher
probe.reply("world")
assert(future.isCompleted && future.value == Some(Success("world")))
```

#### 对探针收到的消息进行转发

假定一个象征性的actor网络中某目标 actor `dest` 从 actor `source`收到一条消息。 如果你使消息先发往` TestProbe probe `, 你可以在保持网络功能的同时对消息流的容量和时限进行断言:

```scala
class Source(target: ActorRef) extends Actor {
  def receive = {
    case "start" => target ! "work"
  }
}
class Destination extends Actor {
  def receive = {
    case x => // Do something..
  }
}
val probe = TestProbe()
val source = system.actorOf(Props(classOf[Source], probe.ref))
val dest = system.actorOf(Props[Destination])
source ! "start"
probe.expectMsg("work")
probe.forward(dest)
```

`dest` actor 将收到同样的消息，就象没有插入探针一样。

#### 自动导向

将收到的消息放进队列以便以后处理，这种方法不错，但要保持测试运行并对其运行过程进行跟踪，你也可以为参与测试的探针(事实上是任何 TestKit)安装一个` AutoPilot（自动导向）`。 自动导向在消息进入检查队列之前启动。 以下代码可以用来转发消息, 例如` A --> Probe --> B`, 只要满足一定的协约。

```scala
val probe = TestProbe()
probe.setAutoPilot(new TestActor.AutoPilot {
  def run(sender: ActorRef, msg: Any): TestActor.AutoPilot =
    msg match {
        case "stop" => TestActor.NoAutoPilot
        case x => testActor.tell(x, sender); TestActor.KeepRunning }
})
```

run 方法必须返回包含在`Option`中的`auto-pilot`供下一条消息使用, 设置成` None `表示终止自动导向。

#### 小心定时器断言

在使用测试探针时，`within` 块的行为可能会不那么直观：你需要记住上文所描述的期限仅对每一个探针的局部作用域有效。因此，探针 不会响应别的探针的期限，也不响应包含它的`TestKit`实例的期限:

```scala
val probe = TestProbe()
within(1 second) {
  probe.expectMsg("hello")
}
```

这里，`expectMsg`调用将会使用缺省的timeout。


### 测试父子关系

一个actor的父actor创建该actor。这造成了两者直接的耦合，使其不可以直接测试。明显地，有三种方法可以提高父子关系的可测性。

- 当创建一个子actor时，传递一个直接的引用到它的父actor。
- 当创建一个父actor时，告诉父actor如何创建它的子actor。
- 当测试时，创建一个虚构(fabricated)的父actor。

例如，你想测试的代码结构如下所示：

```scala
class Parent extends Actor {
  val child = context.actorOf(Props[Child], "child")
  var ponged = false
  def receive = {
    case "pingit" => child ! "ping"
    case "pong"   => ponged = true
} }
class Child extends Actor {
  def receive = {
    case "ping" => context.parent ! "pong"
  }
}
```

#### 使用依赖注入

第一个选项是避免使用`context.parent`函数。用一个自定义的父actor创建子actor，这个自定义的父actor通过传递一个直接的引用到它的父actor。

```scala
class DependentChild(parent: ActorRef) extends Actor {
  def receive = {
    case "ping" => parent ! "pong"
  }
}
```
另一种方法是，你可以告诉父节点怎样创建它的子节点。有两种方式可以做到这件事：给它一个Props对象或者给它一个关注创建子actor的函数。

```scala
class DependentParent(childProps: Props) extends Actor {
  val child = context.actorOf(childProps, "child")
  var ponged = false
  def receive = {
    case "pingit" => child ! "ping"
    case "pong"   => ponged = true
} }
class GenericDependentParent(childMaker: ActorRefFactory => ActorRef) extends Actor {
  val child = childMaker(context)
  var ponged = false
  def receive = {
    case "pingit" => child ! "ping"
    case "pong"   => ponged = true
} }
```

创建Props是直接的，但是创建函数需要像如下的测试代码：

```scala
val maker = (_: ActorRefFactory) => probe.ref
val parent = system.actorOf(Props(classOf[GenericDependentParent], maker))
```

在你的应用程序中，可能如下面这样：

```scala
val maker = (f: ActorRefFactory) => f.actorOf(Props[Child])
val parent = system.actorOf(Props(classOf[GenericDependentParent], maker))
```
#### 使用一个虚拟的父actor

如果你不愿改变一个父actor或者子actor的构造器，你可以在你的测试中创建一个虚拟父actor。但是，这不允许你独立测试父actor。

```scala
"A fabricated parent" should {
  "test its child responses" in {
    val proxy = TestProbe()
    val parent = system.actorOf(Props(new Actor {
      val child = context.actorOf(Props[Child], "child")
      def receive = {
        case x if sender == child => proxy.ref forward x
        case x =>child forward x
      }
    }))
    proxy.send(parent, "ping")
    proxy.expectMsg("pong")
  }
}
```




## 4 CallingThreadDispatcher

如上文所述，` CallingThreadDispatcher `在单元测试中非常重要, 但最初它出现是为了在出错的时候能够生成连续的`stacktrace`。 由于这个特殊的派发器将任何消息直接运行在当前线程中，所以消息处理的完整历史信息在调用堆栈上有记录，只要所有的actor都是在这个派发器上运行。

### 如何使用它

只要象平常一样设置派发器:

```scala
import akka.testkit.CallingThreadDispatcher
val ref = system.actorOf(Props[MyActor].withDispatcher(CallingThreadDispatcher.Id))
```

### 它是如何运作的

在被调用时,` CallingThreadDispatcher `会检查接收消息的actor是否已经在当前线程中了。 这种情况的最简单的例子是actor向自己发送消息。 这时，不能马上对它进行处理，因为这违背了actor模型, 于是这个消息被放进队列，直到actor的当前消息被处理完毕；这样，新消息会被在调用的线程上处理，只是在actor完成其先前的工作之后。 在别的情况下，消息会在当前线程中立即得到处理。 通过这个派发器规划的Future也会立即执行。

这种工作方式使` CallingThreadDispatcher `像一个为永远不会因为外部事件而阻塞的actor设计的通用派发器。

在有多个线程的情况下，有可能同时有两个使用这个派发器的actor在不同线程中收到消息，它们会竞争actor锁，竞争失败的那个必须等待。 这样我们保持了actor模型，但由于使用了受限的调度我们损失了一些并发性。从这个意义上说，它等同于使用传统的基于互斥的并发。

另一个困难是正确地处理挂起和继续: 当actor被挂起时，后续的消息将被放进一个`thread-local`的队列中（和正常情况下使用的队列是同一个)。 但是对resume的调用, 是由一个特定的线程执行的，系统中所有其它的线程可能并没有运行这个特定的actor，这会导致`thread-local`队列无法被它们的本地线程清空。于是，调用` resume `的线程会从所有线程收集所有当前在队列中的消息到自己的队列中，然后进行处理。

### 


