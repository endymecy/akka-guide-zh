# 路由

消息可以通过路由器有效的发送到目的actor，这称为`routees`。一个路由器可以在actor内部和外部使用，你能够自己管理`routees`或者使用一个配置容量的自包含路由器actor。

根据你的应用程序的需求，可以使用不同的路由器策略。Akka包含了几个开箱可用的路由器策略。

## 1 一个简单的路由器

下面的例子证明了怎样使用`Router`以及管理`routees`。

```scala
import akka.routing.ActorRefRoutee
import akka.routing.Router
import akka.routing.RoundRobinRoutingLogic
class Master extends Actor {
  var router = {
    val routees = Vector.fill(5) {
      val r = context.actorOf(Props[Worker])
      context watch r
      ActorRefRoutee(r)
}
    Router(RoundRobinRoutingLogic(), routees)
  }
  def receive = {
    case w: Work =>
      router.route(w, sender())
    case Terminated(a) =>
      router = router.removeRoutee(a)
      val r = context.actorOf(Props[Worker])
      context watch r
      router = router.addRoutee(r)
} }
```

我们创建一个路由器并且当路由消息到routee时指定使用`RoundRobinRoutingLogic`。

Akka中自带的路由逻辑有：

- akka.routing.RoundRobinRoutingLogic
- akka.routing.RandomRoutingLogic
- akka.routing.SmallestMailboxRoutingLogic
- akka.routing.BroadcastRoutingLogic
- akka.routing.ScatterGatherFirstCompletedRoutingLogic
- akka.routing.TailChoppingRoutingLogic
- akka.routing.ConsistentHashingRoutingLogic

我们创建routees为包裹ActorRefRoutee的普通子actor，我们监视routees，当它们停止时可以置换它们。

通过路由器发送消息用`route`方法，如发送上例中的Work消息。

`Router`是不可变的，`RoutingLogic`是线程安全的。这意味着它可以在actor之外使用。

> *注：一般请求下，发送到路由器的任何消息将会进一步发送到routees，但是有一个例外， Broadcast Messages将会发送到所有路由器的routees*

## 2 一个路由器actor

一个路由器可以创建为一个自包含的actor，它自己管理routees以及从配置中加载路由逻辑和其它设置。

这种类型的路由器actor有两种不同的风格：

- Pool：路由器创建routees为子actors，如果它们终止了，那么就从路由器中删除它们
- Group：外部创建routee actors给路由器，路由器使用actor selection发送消息到特定的路径，不观察它的终止

可以用配置或者编码定义路由器actor的设置。虽然路由器actor可以在配置文件中配置，但是它还是必须通过变成创建。如果你在配置文件中定义了路由器actor，那么这些设置将会被用来替换编程提供的参数。

你通过路由器actor发送消息到routees与普通的actor的方式（通过它的`ActorRef`）是一样的。路由器actor转发消息到它的routees而不需要改变它的原始发送者。当routee回复一个路由过的消息，这个回复将会发送到原始发送者，而不是路由器actor。

### Pool

下面的代码和配置片段显示了怎样创建一个轮询的路由器，这个路由器转发消息到五个`Worker` routees。routees将会创建为路由器的孩子。

```scala
akka.actor.deployment {
  /parent/router1 {
    router = round-robin-pool
    nr-of-instances = 5
  }
}
```

```scala
￼val router1: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router1")
```

下面是一个相同的例子，只是路由器配置通过编码而不是配置文件获得。

```scala
￼val router2: ActorRef =
  context.actorOf(RoundRobinPool(5).props(Props[Worker]), "router2")
```

#### 远程部署的Routees
 
除了可以将本地创建的actors作为routees, 你也可以让路由actor将自己创建的子actors部署到一组远程主机上; 这是以轮询的方式执行的。要完成这个工作，将配置包在` RemoteRouterConfig`中, 并附上作为部署目标的结点的远程地址。自然地这要求你在classpath中包括`akka-remote `模块:

```scala
import akka.actor.{ Address, AddressFromURIString }
import akka.remote.routing.RemoteRouterConfig
val addresses = Seq(
  Address("akka.tcp", "remotesys", "otherhost", 1234),
  AddressFromURIString("akka.tcp://othersys@anotherhost:1234"))
val routerRemote = system.actorOf(
  RemoteRouterConfig(RoundRobinPool(5), addresses).props(Props[Echo]))
```

#### 发送者（Sender）

默认情况下，当一个routee发送消息时，它将隐式地设置它自己为发送者

```scala
sender() ! x // replies will go to this actor
```

然而，对于routees而言，设置路由器为发送者通常是有用的。例如，你有想隐藏路由器背后routees的细节时，你有可能想设置路由器为发送者。下面的代码片段显示怎样设置父路由器为发送者。

```scala
￼sender().tell("reply", context.parent) // replies will go back to parent
 sender().!("reply")(context.parent) // alternative syntax 
```

#### 监视(Supervision)

通过一个pool路由器创建的routees是路由器的子actors，所有路由器是子actors的监视器。

路由器actor的监视策略可以通过Pool的`supervisorStrategy`的属性配置。如果没有提供配置，那么缺省的策略是“一直升级（always escalate）”。这意味着错误会传递到路由器的监视器上进行处理。路由器的监视器将会决定去做什么。

注意路由器监控器将会将错误当作一个带有路由器本身的错误。因此，一个停止或者重启指令将会造成路由器自己停止或者重启。路由器的停止又会造成子actors停止或者重启。

需要提出的一点是，路由器的重启行为已经被重写了，所以它将会重新创建子actors，并且保证Pool中拥有相同数量的actors。

这意味着，如果你没有指定路由器或者它的父actor的`supervisorStrategy`，routees中的失败将会升级到路由器的父actor，这将默认导致路由器重启，进而重启所有的routees。这是因为默认行为-添加`withRouter`到子actor的定义，不会改变应用到子actor的监控策略。这可能是无效的，所以你应该避免在定义路由器时指定监督策略。

This means that if you have not specified supervisorStrategy of the router or its parent a failure in a routee will escalate to the parent of the router, which will by default restart the router, which will restart all routees (it uses Escalate and does not stop routees during restart). The reason is to make the default behave such that adding withRouter to a child’s definition does not change the supervision strategy applied to the child. This might be an inefficiency that you can avoid by specifying the strategy when defining the router.

可以很简单的设置策略：

```scala
￼val escalator = OneForOneStrategy() {
    case e ＝> testActor ! e; SupervisorStrategy.Escalate 
 }
 val router = system.actorOf(RoundRobinPool(1, supervisorStrategy = escalator).props(
   routeeProps = Props[TestActor]))
```

> *注：一个Pool路由器的子actors终止，Pool路由器不会自动创建一个新的子actor。如果一个Pool路由器的所有子actors终止，路由器自己也会终止，除非它是一个动态路由器，如使用一个resizer*

## Group

有时，与其用路由器actor创建它的routees，分开创建routees并把它们提供给路由器使用更令人满意。你可以通过传递一个routees的路径到路由器的配置做到这一点。消息将会利用`ActorSelection`发送到这些路径。

下面的例子显示了通过提供给路由器三个routee actors的路径字符串来创建这个路由器。

```scala
akka.actor.deployment {
  /parent/router3 {
    router = round-robin-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
￼val router3: ActorRef =
  context.actorOf(FromConfig.props(), "router3")
```

下面是相同的例子，只是路由器的配置通过编程设置而不是配置文件。

```scala
￼val router4: ActorRef =
  context.actorOf(RoundRobinGroup(paths).props(), "router4")
```

routee actors在路由器外部创建：

```scala
system.actorOf(Props[Workers], "workers")
```

```scala
class Workers extends Actor {
  context.actorOf(Props[Worker], name = "w1")
  context.actorOf(Props[Worker], name = "w2")
  context.actorOf(Props[Worker], name = "w3")
  // ...
```

路径可能包括为运行在远程机器上的actors提供的协议和地址信息。Remoting需要将`akka-remote`模块包含在类路径下

```scala
akka.actor.deployment {
  /parent/remoteGroup {
    router = round-robin-group
    routees.paths = [
      "akka.tcp://app@10.0.0.1:2552/user/workers/w1",
      "akka.tcp://app@10.0.0.2:2552/user/workers/w1",
      "akka.tcp://app@10.0.0.3:2552/user/workers/w1"]
} }
```

## 3 路由器的使用

这一章，我们将介绍怎样创建不同类型的路由器actor。

这一章的路由器actors通过一个名叫`parent`的顶级actor创建。注意在配置中，部署路径以`/parent/`开头，后跟着路由器actor的名字。

```scala
system.actorOf(Props[Parent], "parent")
```

### RoundRobinPool和RoundRobinGroup

以轮询的方式路由到routee

在配置文件中定义的`RoundRobinPool`

```scala
akka.actor.deployment {
  /parent/router1 {
    router = round-robin-pool
    nr-of-instances = 5
  }
}
```

```scala
￼val router1: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router1")
```

在代码中定义的`RoundRobinPool`

```scala
￼val router2: ActorRef =
  context.actorOf(RoundRobinPool(5).props(Props[Worker]), "router2")
```

在配置文件中定义的`RoundRobinGroup`

```scala
akka.actor.deployment {
  /parent/router3 {
    router = round-robin-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
￼val router3: ActorRef =
  context.actorOf(FromConfig.props(), "router3")
```

在代码中定义的`RoundRobinGroup`

```scala
￼val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
 val router4: ActorRef =
  context.actorOf(RoundRobinGroup(paths).props(), "router4")
```

### RandomPool和RandomGroup

这个路由器类型为每个消息随机选择一个routee

在配置文件中定义的`RandomGroup`

```scala
akka.actor.deployment {
  /parent/router5 {
    router = random-pool
    nr-of-instances = 5
  }
}
```

```scala
￼val router5: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router5")
```
在代码中定义的`RandomGroup`

```scala
￼val router6: ActorRef =
  context.actorOf(RandomPool(5).props(Props[Worker]), "router6")
```

在配置文件中定义的`RandomGroup`

```scala
akka.actor.deployment {
  /parent/router7 {
    router = random-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
val router7: ActorRef =
  context.actorOf(FromConfig.props(), "router7")
```

在代码中定义的`RandomGroup`

```scala
￼val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
 val router8: ActorRef =
  context.actorOf(RandomGroup(paths).props(), "router8")
```

### BalancingPool

这个路由器重新分配工作，从繁忙的routees到空闲的routees。所有的routees共享相同的邮箱

在配置文件中定义的`BalancingPool`

```scala
akka.actor.deployment {
  /parent/router9 {
    router = balancing-pool
    nr-of-instances = 5
  }
}
```

```scala
￼val router9: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router9")
```

在代码中定义的`BalancingPool`

```scala
￼val router10: ActorRef =
  context.actorOf(BalancingPool(5).props(Props[Worker]), "router10")
```
平衡派发器有额外的配置，这可以被Pool使用，在路由器部署配置的`pool-dispatcher`片段中配置。

```scala
akka.actor.deployment {
  /parent/router9b {
    router = balancing-pool
    nr-of-instances = 5
    pool-dispatcher {
      attempt-teamwork = off
    }
} }
```

###  SmallestMailboxPool

这个路由器选择未挂起的邮箱中消息数最少的routee。选择顺序如下所示：

- 选取任何一个空闲的（没有正在处理的消息）邮箱为空的 routee
- 选择任何邮箱为空的routee
- 选择邮箱中等待的消息最少的 routee
- 选择任何一个远程 routee, 由于邮箱大小未知，远程actor被认为具有低优先级

定义在配置文件中的`SmallestMailboxPool`

```scala
akka.actor.deployment {
  /parent/router11 {
    router = smallest-mailbox-pool
    nr-of-instances = 5
  }
}
```

```scala
￼val router11: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router11")
```
在代码中定义的`SmallestMailboxPool`

```scala
￼val router12: ActorRef =
  context.actorOf(SmallestMailboxPool(5).props(Props[Worker]), "router12")
```

### BroadcastPool和BroadcastGroup

一个广播路由器转发消息到所有的routees

定义在配置文件中的`BroadcastPool`

```scala
akka.actor.deployment {
  /parent/router13 {
    router = broadcast-pool
    nr-of-instances = 5
  }
}
```

```scala
￼val router13: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router13")
```

定义在代码中的`BroadcastPool`

```scala
￼val router14: ActorRef =
  context.actorOf(BroadcastPool(5).props(Props[Worker]), "router14")
```

定义在配置文件中的`BroadcastGroup`

```scala
akka.actor.deployment {
  /parent/router15 {
    router = broadcast-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
  }
}
```

```scala
￼val router15: ActorRef =
  context.actorOf(FromConfig.props(), "router15")
```

定义在代码中的`BroadcastGroup`

```scala
￼val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
 val router16: ActorRef =
  context.actorOf(BroadcastGroup(paths).props(), "router16")
```
### ScatterGatherFirstCompletedPool和ScatterGatherFirstCompletedGroup

ScatterGatherFirstCompletedRouter发送消息到它的每个routees，然后等待返回的第一个回复。这个结果将会返回原始发送者（original sender）。其它回复丢弃。

它期待至少一个带有配置时间的回复。否则它将回复一个带有`akka.pattern.AskTimeoutException`的`akka.actor.Status.Failure`。

在配置文件中定义的`ScatterGatherFirstCompletedPool`

```scala
akka.actor.deployment {
  /parent/router17 {
    router = scatter-gather-pool
    nr-of-instances = 5
    within = 10 seconds
} }
```

```scala
val router17: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router17")
```

在代码中定义的`ScatterGatherFirstCompletedPool`

```scala
￼val router18: ActorRef =
  context.actorOf(ScatterGatherFirstCompletedPool(5, within = 10.seconds).
    props(Props[Worker]), "router18")
```

在配置文件中定义的`ScatterGatherFirstCompletedGroup`

```scala
akka.actor.deployment {
  /parent/router19 {
    router = scatter-gather-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    within = 10 seconds
} }
```

```scala
￼val router19: ActorRef =
  context.actorOf(FromConfig.props(), "router19")
```

在代码中定义的`ScatterGatherFirstCompletedGroup`

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router20: ActorRef =
  context.actorOf(ScatterGatherFirstCompletedGroup(paths,
    within = 10.seconds).props(), "router20")
```

### TailChoppingPool和TailChoppingGroup

TailChoppingPool首先发送一个消息给一个随机选择的routee，然后等待一段时间，发送第二个消息给一个随机选择的routee，依此类推。它等待返回的第一个回复，然后讲回复发送给原始发送者。其它回复丢弃。

这个路由器的目标是减少通过到多个routees的路由冗余查询而产生的性能延迟，假定其它actors中的某一个比初始化的那个反应速度快。

在配置文件中定义的`TailChoppingPool`

```scala
￼akka.actor.deployment {
  /parent/router21 {
    router = tail-chopping-pool
        nr-of-instances = 5
        within = 10 seconds
        tail-chopping-router.interval = 20 milliseconds
    } }
```

```scala
￼val router21: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router21")
```

在代码中定义的`TailChoppingPool`

```scala
￼val router22: ActorRef =
  context.actorOf(TailChoppingPool(5, within = 10.seconds, interval = 20.millis).
    props(Props[Worker]), "router22")
```

在配置文件中定义的`TailChoppingGroup`

```scala
akka.actor.deployment {
  /parent/router23 {
    router = tail-chopping-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    within = 10 seconds
    tail-chopping-router.interval = 20 milliseconds
} }
```
```scala
￼val router23: ActorRef =
  context.actorOf(FromConfig.props(), "router23")
```

在代码中定义的`TailChoppingGroup`

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router24: ActorRef =
  context.actorOf(TailChoppingGroup(paths,
    within = 10.seconds, interval = 20.millis).props(), "router24")
```

### ConsistentHashingPool 和 ConsistentHashingGroup

`ConsistentHashingPool`使用[一致性哈希](http://en.wikipedia.org/wiki/Consistent_hashing)选择routee发送消息。这个[文章](http://weblogs.java.net/blog/tomwhite/archive/2007/11/consistent_hash.html)给出了一致性哈希实现的一些好的建议。

对于一致性哈希而言有三种方式定义使用哪些数据。

- 你可以定义路由器的`hashMapping`去映射输入消息到它们的一致性哈希key，这使得决策对发送者（sender）是透明的。
- 消息可能实现`akka.routing.ConsistentHashingRouter.ConsistentHashable`。key是消息的一部分，把它和消息定义在一起是方便的。
- 消息可以包裹在`akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope`中定义一致性哈希key使用的数据。发送者知道使用的key。

定义一致性哈希的这些方式可以一起使用，并且可以在一个路由器上同时使用。首先尝试使用`hashMapping`

例子代码：

```scala
import akka.actor.Actor
import akka.routing.ConsistentHashingRouter.ConsistentHashable
 
class Cache extends Actor {
  var cache = Map.empty[String, String]
 
  def receive = {
    case Entry(key, value) => cache += (key -> value)
    case Get(key)          => sender() ! cache.get(key)
    case Evict(key)        => cache -= key
  }
}
 
case class Evict(key: String)
 
case class Get(key: String) extends ConsistentHashable {
  override def consistentHashKey: Any = key
}
 
case class Entry(key: String, value: String)
```

```scala
import akka.actor.Props
import akka.routing.ConsistentHashingPool
import akka.routing.ConsistentHashingRouter.ConsistentHashMapping
import akka.routing.ConsistentHashingRouter.ConsistentHashableEnvelope
 
def hashMapping: ConsistentHashMapping = {
  case Evict(key) => key
}
 
val cache: ActorRef =
  context.actorOf(ConsistentHashingPool(10, hashMapping = hashMapping).
    props(Props[Cache]), name = "cache")
 
cache ! ConsistentHashableEnvelope(
  message = Entry("hello", "HELLO"), hashKey = "hello")
cache ! ConsistentHashableEnvelope(
  message = Entry("hi", "HI"), hashKey = "hi")
 
cache ! Get("hello")
expectMsg(Some("HELLO"))
 
cache ! Get("hi")
expectMsg(Some("HI"))
 
cache ! Evict("hi")
cache ! Get("hi")
expectMsg(None)
```

在上面的例子中，你可以看到`Get`消息实现了`ConsistentHashable`，`Entry`消息包裹在了`ConsistentHashableEnvelope`中。`Evict`消息通过`hashMapping`偏函数处理。

定义在配置文件中的`ConsistentHashingPool`

```scala
akka.actor.deployment {
  /parent/router25 {
    router = consistent-hashing-pool
    nr-of-instances = 5
    virtual-nodes-factor = 10
  }
}
```

```scala
val router25: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router25")
```

定义在代码中的`ConsistentHashingPool`

```scala
val router26: ActorRef =
  context.actorOf(ConsistentHashingPool(5).props(Props[Worker]),
    "router26")
```

定义在配置文件中的`ConsistentHashingGroup`

```scala
akka.actor.deployment {
  /parent/router27 {
    router = consistent-hashing-group
    routees.paths = ["/user/workers/w1", "/user/workers/w2", "/user/workers/w3"]
    virtual-nodes-factor = 10
  }
}
```

```scala
val router27: ActorRef =
  context.actorOf(FromConfig.props(), "router27")
```

定义在代码中的`ConsistentHashingGroup`

```scala
val paths = List("/user/workers/w1", "/user/workers/w2", "/user/workers/w3")
val router28: ActorRef =
  context.actorOf(ConsistentHashingGroup(paths).props(), "router28")
```

`virtual-nodes-factor`是每个routee 虚拟节点的个数，这些节点用在一致哈希环中使分布更加均匀。

## 4 特殊处理的消息

大部分发送到路由器actor的消息都会根据路由器的路由逻辑转发。然而有几种类型的消息有特殊的行为。

注意，除了`Broadcast`消息，这些特殊的消息仅仅被自包含的路由器actor处理，而不会被`akka.routing.Router`组件处理。

### 广播消息

一个广播消息可以被用来发送消息到所有个routees。当一个路由器接收到一个广播消息，它将广播这个消息的负载（payload）到所有routees，不管这个路由器如何路由这个消息。

下面的例子展示了怎样使用一个广播消息发送一个非常重要的消息到路由器的每个routee。

```scala
￼import akka.routing.Broadcast
 router ! Broadcast("Watch out for Davy Jones’ locker")
```

在这个例子中，路由器接收到广播消息，抽取它的负载“Watch out for Davy Jones’ locker”，然后发送这个负载到所有的routees。

### PoisonPill消息

`PoisonPill`消息对所有的actors（包括路由器）拥有特殊的处理。当任何actor收到一个`PoisonPill`消息，这个actor将被停止。

```scala
￼import akka.actor.PoisonPill
 router ! PoisonPill
```

对于一个正常传递消息到routees的路由器来说，认识到`PoisonPill`消息仅仅被路由器处理是非常重要的。发送到路由器的`PoisonPill`消息不会被转发给routees。

然而，发送给路由器的`PoisonPill`消息仍然有可能影响到它的routees。因为它将停止路由器，路由器停止了，它的子actors也会停止。停止子actors是正常的actor行为。路由器将会停止routees。每个子actor将会处理它当前的消息然后停止。这可能导致一些消息没有被处理。

如果你希望停止一个路由器以及它的routees，但是希望routees首先处理它们邮箱中的所有消息，你不应该发送一个`PoisonPill`消息到路由器，而应该在`Broadcast`消息内包裹一个`PoisonPill`消息，这样每个routee都会收到`PoisonPill`消息。注意这将停止所有的routees，即使这个routee不是路由器的子actor。

```scala
￼import akka.actor.PoisonPill
 import akka.routing.Broadcast
 router ! Broadcast(PoisonPill)
```
上面的代码显示，每个routee都将收到`PoisonPill`消息。每个routee都将正常处理消息，最后处理`PoisonPill`消息。这将造成routee停止。当所有的routee停止后，路由器也会自动停止，除非它是动态路由器。

### Kill消息

Kill消息是另一种有特殊处理的消息。

当一个Kill消息发送到路由器，路由器将会在内部处理这个消息，不会把它发送到routees。路由器将会抛出一个`ActorKilledException`异常并且失败。然后它将会恢复、重启或者终止，这依赖于它被怎样监视。

作为路由器子actor的routees也将悬起，这受到应用于路由器的监督指令的影响。不是路由器子actor的routees不会收到影响。

```scala
￼import akka.actor.Kill
 router ! Kill
```

杀死路由器的所有routees可以将kill消息包裹在广播消息中

```scala
import akka.actor.Kill
import akka.routing.Broadcast
router ! Broadcast(Kill)
```

### 管理消息

- 发送`akka.routing.GetRoutees`给路由器actor将会使它返回其当前在一个`akka.routing.Routees`消息中使用的routee。
- 发送`akka.routing.AddRoutee`给路由器actor将会添加这个routee到它的routee集合中
- 发送`akka.routing.RemoveRoutee`给路由器将会从routee集合中删除这个routee
- 发送`akka.routing.AdjustPoolSize`给一个Pool路由器actor将会调整包含routees的集合的容量

管理消息可能在其它消息之后处理，所以如果你发送`AddRoutee`消息之后马上发送一个普通的消息到路由器，你无法保证当普通消息路由时，routees已经改变了。如果你需要知道哪些改变已经应用了，你可以在`AddRoutee`之后发送`GetRoutees`，当你收到`Routees`的回复之后你就能指定之前的那个改变应用了。

## 5 动态调整Pool大小

大部分的池被使用，它们有一个固定数量routees。也可以用调整大小策略动态调整routees的数量。

在配置文件中定义的可调整大小的Pool

```scala
akka.actor.deployment {
  /parent/router29 {
    router = round-robin-pool
    resizer {
      lower-bound = 2
      upper-bound = 15
      messages-per-resize = 100
} }
}
```

```scala
￼val router29: ActorRef =
  context.actorOf(FromConfig.props(Props[Worker]), "router29")
```
在代码中定义的可调整大小的Pool

```scala
val resizer = DefaultResizer(lowerBound = 2, upperBound = 15)
val router30: ActorRef =
  context.actorOf(RoundRobinPool(5, Some(resizer)).props(Props[Worker]),
    "router30")
```

如果你在配置文件中定义了`router`，那么这个值会代替任何代码中设置的值。

## 6 Akka中的路由是如何设计的

表面上路由器看起来像一般的actor，但是它们实际的实现是完全不同的。路由器被设计为非常有效地接收消息然后很快地传递它们到routees。

一个一般的actor可以被用来路由消息，但是一个actor单线程的处理可能成为一个瓶颈。路由器可以获得更高的吞吐量，它使用的消息处理管道可以允许并发路由。这可以通过直接嵌套路由器的路由逻辑到`ActorRef`而不是路由器actor来实现。发送到`ActorRef`的消息可以立即路由到routee，完全绕过单线程路由器actor。

当然，这样的代价比一般actor实现的路由器的路由代码更加复杂。幸运的是，所有的复杂性对消费者来说是不可见的。

## 7 自定义路由器

你也可以创建你自己的路由器。

下面的例子创建一个路由器复制每个消息到几个目的地。

从路由逻辑开始：

```scala
import scala.collection.immutable
import scala.concurrent.forkjoin.ThreadLocalRandom
import akka.routing.RoundRobinRoutingLogic
import akka.routing.RoutingLogic
import akka.routing.Routee
import akka.routing.SeveralRoutees
class RedundancyRoutingLogic(nbrCopies: Int) extends RoutingLogic {
  val roundRobin = RoundRobinRoutingLogic()
  def select(message: Any, routees: immutable.IndexedSeq[Routee]): Routee = {
    val targets = (1 to nbrCopies).map(_ => roundRobin.select(message, routees))
    SeveralRoutees(targets)
  }
}
```

在这个例子中，通过重用已经存在的`RoundRobinRoutingLogic`，包裹结果到`SeveralRoutees`实例。为每个消息调用`select`,并且通过轮询方式选择几个目的地。

路由逻辑的实现必须是线程安全的，因为它可能会在actor外部使用。

一个路由逻辑的单元测试

```scala
case class TestRoutee(n: Int) extends Routee {
  override def send(message: Any, sender: ActorRef): Unit = ()
}
  val logic = new RedundancyRoutingLogic(nbrCopies = 3)
  val routees = for (n <- 1 to 7) yield TestRoutee(n)
  val r1 = logic.select("msg", routees)
  r1.asInstanceOf[SeveralRoutees].routees should be(
    Vector(TestRoutee(1), TestRoutee(2), TestRoutee(3)))
  val r2 = logic.select("msg", routees)
  r2.asInstanceOf[SeveralRoutees].routees should be(
    Vector(TestRoutee(4), TestRoutee(5), TestRoutee(6)))
  val r3 = logic.select("msg", routees)
  r3.asInstanceOf[SeveralRoutees].routees should be(
    Vector(TestRoutee(7), TestRoutee(1), TestRoutee(2)))
```

让我们继续把这变成一个自包含、可配置的路由器actor。

创建一个继承自`Pool`、`Group`或者`CustomRouterConfig`的类。这个类是路由逻辑的工厂类，保有路由器的配置。

```scala
import akka.dispatch.Dispatchers
import akka.routing.Group
import akka.routing.Router
import akka.japi.Util.immutableSeq
import com.typesafe.config.Config

case class RedundancyGroup(override val paths: immutable.Iterable[String], nbrCopies: Int) extends Group {

  def this(config: Config) = this(
    paths = immutableSeq(config.getStringList("routees.paths")),
    nbrCopies = config.getInt("nbr-copies"))

  override def createRouter(system: ActorSystem): Router =
    new Router(new RedundancyRoutingLogic(nbrCopies))

  override val routerDispatcher: String = Dispatchers.DefaultDispatcherId
}
```

这可以作为被Akka支持的路由器actors正确的使用

```scala
for (n <- 1 to 10) system.actorOf(Props[Storage], "s" + n)
val paths = for (n <- 1 to 10) yield ("/user/s" + n)
val redundancy1: ActorRef =
  system.actorOf(RedundancyGroup(paths, nbrCopies = 3).props(),
    name = "redundancy1")
redundancy1 ! "important"
```

注意我们在`RedundancyGroup`中增加了一个构造函数持有`Config`参数。这使它可以在配置文件中配置。

```scala
akka.actor.deployment {
  /redundancy2 {
    router = "docs.routing.RedundancyGroup"
    routees.paths = ["/user/s1", "/user/s2", "/user/s3"]
    nbr-copies = 5
  }
}
```

在`router`属性中描述完整的类名。路由器类必须继承自`akka.routing.RouterConfig (Pool, Group or CustomRouterConfig) `，并且拥有一个包含`com.typesafe.config.Config`参数的构造器。

```scala
val redundancy2: ActorRef = system.actorOf(FromConfig.props(),
  name = "redundancy2")
redundancy2 ! "very important"
```

## 8 配置派发器

为了简单地定义Pool的routees的派发器，你可以在配置文件的部署片段定义派发器。

```scala
akka.actor.deployment {
  /poolWithDispatcher {
    router = random-pool
    nr-of-instances = 5
    pool-dispatcher {
      fork-join-executor.parallelism-min = 5
      fork-join-executor.parallelism-max = 5
    }
  }
}
```
这是为一个Pool提供派发器唯一需要做的事情。

> *如果你使用一个actor的group路由到它的路径，它将会一直使用在Props中配置的相同的派发器，在actor创建之后，不能修改actor的派发器*

“head”路由actor, 不能运行在同样的派发器上, 因为它并不处理相同的消息，这个特殊的actor并不使用 Props中配置的派发器, 而是使用从`RouterConfig` 来的 `routerDispatcher` , 它缺省为actor系统的缺省派发器. 所有的标准路由actor都允许在其构造方法或工厂方法中设置这个属性，自定义路由必须以合适的方式自己实现这个方法。

```scala
val router: ActorRef = system.actorOf(
  // “head” router actor will run on "router-dispatcher" dispatcher
  // Worker routees will run on "pool-dispatcher" dispatcher
  RandomPool(5, routerDispatcher = "router-dispatcher").props(Props[Worker]),
  name = "poolWithDispatcher")
```

> *注：不允许为一个`akka.dispatch.BalancingDispatcherConfigurator`配置`routerDispatcher`，因为特殊路由器的消息不能被其它的消息处理*