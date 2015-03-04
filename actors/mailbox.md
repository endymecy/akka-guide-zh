# 邮箱

> 一个AKKA邮箱保有发向actor的消息，一般情况下，每个actor都有自己的邮箱，but with for example a BalancingPool all routees will share a single mailbox instance。

## 1 邮箱选择

### 为一个actor指定一个消息队列类型

通过继承参数化的trait `RequiresMessageQueue`为一个确定类型的actor指定一个确定类型的消息类型是可能的。下面是一个例子。

```scala
import akka.dispatch.RequiresMessageQueue
import akka.dispatch.BoundedMessageQueueSemantics
class MyBoundedActor extends MyActor
  with RequiresMessageQueue[BoundedMessageQueueSemantics]
```

`RequiresMessageQueue` trait需要在配置文件中映射到一个邮箱上。如下所示：

```scala
bounded-mailbox {
  mailbox-type = "akka.dispatch.BoundedMailbox"
  mailbox-capacity = 1000
  mailbox-push-timeout-time = 10s
}
akka.actor.mailbox.requirements {
  "akka.dispatch.BoundedMessageQueueSemantics" = bounded-mailbox
}
```

现在，你每次创建`MyBoundedActor`类型的actor，它都会尝试得到一个有边界的邮箱。如果在部署中有不同的邮箱配置（或者是直接的，或者是通过一个拥有特定邮箱类型的派发器），那将覆盖这个映射。

> *注意：为一个actor创建的邮箱类型队列将会在trait中进行核对，检查是不是所需类型。如果队列没有实现所需类型，actor创建将会失败*

### 为一个派发器指定一个消息队列类型

一个派发器可能也有这样的需求：需要一个邮箱类型。一个例子是BalancingDispatcher需要一个消息队列，这个队列对多个并发消费者来说是线程安全的。这个需求在派发器的配置中配置，如下所示：

```scala
￼my-dispatcher {
  mailbox-requirement = org.example.MyInterface
}
```

`mailbox-requirement`需要一个类或者接口名，这个接口然后能够被确认为消息队列实现的超类型。万一冲突了-如果actor需要的邮箱类型不能满足这个需求，actor创建将会失败。

### 怎样选择邮箱类型

当一个actor创建时，`ActorRefProvider`首先确定执行它的派发器，如何确定邮箱，如下所述：

- 如果actor的部署配置片段包含了一个邮箱key，那么这个配置部分描述的邮箱类型将被采用
- 如果actor的Props包含邮箱选择-在其上调用了`withMailbox`，那么这个配置部分描述的邮箱类型将被采用
- 如果派发器的配置片段包含了一个`mailbox-type` key，这个配置片段将会用来配置邮箱类型
- 如果派发器需要一个邮箱类型，这个需求的映射将会用来决定使用的邮箱类型
- 缺省的`akka.actor.default-mailbox`将会被使用

### 缺省的邮箱

当邮箱没有指定时，缺省的邮箱将会使用。默认情况下，这个邮箱是没有边界的。由`java.util.concurrent.ConcurrentLinkedQueue.SingleConsumerOnlyUnboundedMailbox`支持的邮箱是效率更高的邮箱，可以作为缺省的邮箱，但是不能作为和BalancingDispatcher一起使用。

配置`SingleConsumerOnlyUnboundedMailbox`为缺省邮箱，如下所示：

```scala
￼akka.actor.default-mailbox {
  mailbox-type = "akka.dispatch.SingleConsumerOnlyUnboundedMailbox"
}
```
### 哪个配置传递给邮箱类型

每个邮箱类型通过一个继承`MailboxType`的类实现，这个类的构造器有两个参数，一个`ActorSystem.Settings`对象，一个`Config`片段。后者通过从actor系统配置中获得的具名配置片段计算得到，用邮箱类型的配置路径覆盖它的`id` key，并且添加一个`fall-back`到缺省的邮箱配置片段。（he latter is computed by obtaining the named configuration section from the actor system’s configuration, overriding its id key with the configuration path of the mailbox type and adding a fall-back to the default mailbox configuration section）

## 2 内置实现

Akka内置有几个邮箱实现：

- UnboundedMailbox-默认的邮箱

    - 由`java.util.concurrent.ConcurrentLinkedQueue`支持
    - 阻塞：无
    - 边界：无
    - 配置名：`unbounded`或者`akka.dispatch.UnboundedMailbox`
    
- SingleConsumerOnlyUnboundedMailbox

    - 由一个高效的的多生产者单消费者队列支持，不能和`BalancingDispatcher`一起使用
    - 阻塞：无
    - 边界：无
    - 配置名：`akka.dispatch.SingleConsumerOnlyUnboundedMailbox`
    
- BoundedMailbox

    - 由`java.util.concurrent.LinkedBlockingQueue`支持
    - 阻塞：有
    - 边界：有
    - 配置名：`bounded`或者`akka.dispatch.BoundedMailbox`
    
- UnboundedPriorityMailbox

    - 由`java.util.concurrent.PriorityBlockingQueue`支持
    - 阻塞：有
    - 边界：无
    - 配置名：`akka.dispatch.UnboundedPriorityMailbox`
    
- BoundedPriorityMailbox

    - 由`java.util.PriorityBlockingQueue`包裹一个`akka.util.BoundedBlockingQueue`支持
    - 阻塞：有
    - 边界：有
    - 配置名：`kka.dispatch.BoundedPriorityMailbox`
    
## 3 邮箱配置例子

怎样创建一个PriorityMailbox

```scala
import akka.dispatch.PriorityGenerator
import akka.dispatch.UnboundedPriorityMailbox
import com.typesafe.config.Config
// We inherit, in this case, from UnboundedPriorityMailbox
// and seed it with the priority generator
class MyPrioMailbox(settings: ActorSystem.Settings, config: Config)
  extends UnboundedPriorityMailbox(
    // Create a new PriorityGenerator, lower prio means more important
    PriorityGenerator {
      // ’highpriority messages should be treated first if possible
      case ’highpriority => 0
      // ’lowpriority messages should be treated last if possible
      case ’lowpriority  => 2
      // PoisonPill when no other left
      case PoisonPill    => 3
      // We default to 1, which is in between high and low
      case otherwise     => 1
    })
```

然后将它添加到配置文件中：

```scala
prio-dispatcher {
  mailbox-type = "docs.dispatcher.DispatcherDocSpec$MyPrioMailbox"
  //Other dispatcher configuration goes here
}
```

然后是一个如何使用它的例子：

```scala
// We create a new Actor that just prints out what it processes
class Logger extends Actor {
  val log: LoggingAdapter = Logging(context.system, this)
  self ! ’lowpriority
  self ! ’lowpriority
  self ! ’highpriority
  self ! ’pigdog
  self ! ’pigdog2
  self ! ’pigdog3
  self ! ’highpriority
  self ! PoisonPill
  def receive = {
    case x => log.info(x.toString)
} }
val a = system.actorOf(Props(classOf[Logger], this).withDispatcher(
  "prio-dispatcher"))
/*
 * Logs:
 * ’highpriority
 * ’highpriority
 * ’pigdog
 * ’pigdog2
 * ’pigdog3
 * ’lowpriority
 * ’lowpriority
 */
```
也可以用下面的方式直接配置邮箱

```scala
prio-mailbox {
  mailbox-type = "docs.dispatcher.DispatcherDocSpec$MyPrioMailbox"
  //Other mailbox configuration goes here
}
akka.actor.deployment {
  /priomailboxactor {
    mailbox = prio-mailbox
  }
}
```

然后使用它：

```scala
￼import akka.actor.Props
val myActor = context.actorOf(Props[MyActor], "priomailboxactor")
```
或者：

```scala
￼import akka.actor.Props
 val myActor = context.actorOf(Props[MyActor].withMailbox("prio-mailbox"))
```

## 4 创建你自己的邮箱类型

一个例子胜过千言万语

```scala
import akka.actor.ActorRef
import akka.actor.ActorSystem
import akka.dispatch.Envelope
import akka.dispatch.MailboxType
import akka.dispatch.MessageQueue
import akka.dispatch.ProducesMessageQueue
import com.typesafe.config.Config
import java.util.concurrent.ConcurrentLinkedQueue
import scala.Option
// Marker trait used for mailbox requirements mapping
trait MyUnboundedMessageQueueSemantics
object MyUnboundedMailbox {
  // This is the MessageQueue implementation
  class MyMessageQueue extends MessageQueue
    with MyUnboundedMessageQueueSemantics {
    private final val queue = new ConcurrentLinkedQueue[Envelope]()
    // these should be implemented; queue used as example
    def enqueue(receiver: ActorRef, handle: Envelope): Unit =queue.offer(handle)
    def dequeue(): Envelope = queue.poll()
    def numberOfMessages: Int = queue.size
    def hasMessages: Boolean = !queue.isEmpty
    def cleanUp(owner: ActorRef, deadLetters: MessageQueue) {
        while (hasMessages) {
          deadLetters.enqueue(owner, dequeue())
        } 
    }
  }
}
// This is the Mailbox implementation
class MyUnboundedMailbox extends MailboxType
  with ProducesMessageQueue[MyUnboundedMailbox.MyMessageQueue] {
  import MyUnboundedMailbox._
  // This constructor signature must exist, it will be called by Akka
  def this(settings: ActorSystem.Settings, config: Config) = {
    // put your initialization code here
    this()
  }
  // The create method is called to create the MessageQueue
  final override def create(owner: Option[ActorRef],
                            system: Option[ActorSystem]): MessageQueue =
    new MyMessageQueue()
}
```

然后，你仅仅需要你的`MailboxType`的全类名作为派发器配置项或者邮箱配置项`mailbox-type`的值。

> *注意：确保包含一个构造器，这个构造器持有`akka.actor.ActorSystem.Settings`和`com.typesafe.config.Config`两个参数，这个构造器在构造你的邮箱类型时会被直接调用。The config passed in as second argument is that section from the configuration which describes the dispatcher or mailbox setting using this mailbox type; the mailbox type will be instantiated once for each dispatcher or mailbox setting using it*

你也可以使用邮箱作为派发器的requirement

```scala
custom-dispatcher {
  mailbox-requirement =
  "docs.dispatcher.MyUnboundedJMessageQueueSemantics"
}
akka.actor.mailbox.requirements {
  "docs.dispatcher.MyUnboundedJMessageQueueSemantics" =
  custom-dispatcher-mailbox
}
custom-dispatcher-mailbox {
  mailbox-type = "docs.dispatcher.MyUnboundedJMailbox"
}
```

或者在你的actor类里面定义requirement

```scala
class MySpecialActor extends Actor
  with RequiresMessageQueue[MyUnboundedMessageQueueSemantics] {
  // ...
}
```

## 5 system.actorOf 的特殊语义

为了使`system.actorOf`在保证（keeping）返回类型`ActorRef`的同时（返回ref的语义是完全函数式的）既是同步的又是非阻塞的，特殊的处理将会进行。在后台，一个空的actor引用被构造，它被送到系统监控actor，这个监控actor创建actor和它的context，并且将它们放入引用。直到这发生后，发送给`ActorRef`的消息将会在本地排队，并且只在交换真实的内容后才将它们转移到邮箱。

Until that has happened, messages sent to the ActorRef will be queued locally, and only upon swapping the real filling in will they be transferred into the real mailbox。

所以，

```scala
val props: Props = ...
// this actor uses MyCustomMailbox, which is assumed to be a singleton
system.actorOf(props.withDispatcher("myCustomMailbox")) ! "bang"
assert(MyCustomMailbox.instance.getLastEnqueuedMessage == "bang")
```
将有可能失败。








