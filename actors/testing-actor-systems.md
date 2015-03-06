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

- 测试独立的不包括actor模型的代码，即没有多线程的内容；在事件发生的次序方面有完全确定性的行为，没有任何并发考虑， 这在下文中称为`单元测试（Unit Testing`
- 测试（多个）包装过的actor，包括多线程调度; 事件的次序没有确定性但由于使用了actor模型，不需要考虑并发，这在下文中被称为`集成测试（Integration Testing`

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