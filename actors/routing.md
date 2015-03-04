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

