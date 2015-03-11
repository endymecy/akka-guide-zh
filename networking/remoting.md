# 远程调用

要了解关于Akka的远程调用能力的简介请参阅位置透明性。

## 1 为远程调用准备ActorSystem

Akka 远程调用功能在一个单独的jar包中. 确认你的项目依赖中包括以下依赖：

```scala
"com.typesafe.akka" %% "akka-remote" % "2.3.9"
```

要在Akka项目中使用远程调用，最少要在` application.conf `文件中加入以下内容:

```scala
akka { actor {
    provider = "akka.remote.RemoteActorRefProvider"
  }
  remote {
    enabled-transports = ["akka.remote.netty.tcp"]
    netty.tcp {
      hostname = "127.0.0.1"
      port = 2552 }
} }
```

从上面的例子你可以看到为了开始你需要添加的4个东西:

- 将 provider 从` akka.actor.LocalActorRefProvider `改为` akka.remote.RemoteActorRefProvider`
- 增加远程主机名 - 你希望运行actor系统的主机; 这个主机名与传给远程系统的内容完全一样，用来标识这个系统，并供后续需要连接回这个系统时使用, 所以要把它设置成一个可到达的IP地址或一个可以正确解析的域名来保证网络可访问性。
- 增加端口号 - actor 系统监听的端口号，0 表示让它自动选择

## 2 远程交互的类型

Akka 远程调用有两种方式:

- 查找 : 使用`actorSelection(path)`在远程主机上查找一个actor
- 创建 : 使用` actorOf(Props(...), actorName)`在远程主机上创建一个actor

下一部分将对这两种方法进行详细介绍。

## 3 查找远程 Actors

`actorSelection(path)`会获得远程结点上actor的`ActorSelection`。

```scala
￼val selection =
  context.actorSelection("akka.tcp://actorSystemName@10.0.0.1:2552/user/actorName")
```

可以看到使用这种模式在远程结点上查找一个Actor:

```scala
akka.<protocol>://<actor system>@<hostname>:<port>/<actor path>
```

一旦得到了actor的selection，你就可以象与本地actor通讯一样与它进行通迅

```scala
selection ! "Pretty awesome feature"
```

为了为`ActorSelection`获得一个`ActorRef`，你必须发送一个消息给`selection`并且使用从actor而来的回复的`sender`引用。有一个内置的`Identify`，它可以被所有的actors理解并且自动回复一个包含`ActorRef`的`ActorIdentity`消息。这也可以通过调用`ActorSelection`的`resolveOne`方法做到，这个方法返回一个匹配`ActorRef`的Future。

## 4 创建远程 Actor

在Akka中要使用远程创建actor的功能，需要对` application.conf `文件进行以下修改 (只显示deployment部分):

```scala
akka { actor {
    deployment {
      /sampleActor {
        remote = "akka.tcp://sampleActorSystem@127.0.0.1:2553"
      }
} }
}
```

这个配置告知Akka当一个路径为` /sampleActor `的actor被创建时进行响应, i.e. 调用` system.actorOf(Props(...), sampleActor)`时，指定的actor不会被直接实例化，而是远程actor系统的守护进程被要求创建这个actor, 本例中的远程actor系统是`sampleActorSystem@127.0.0.1:2553`。

一旦配置了以上属性你可以在代码中进行如下操作:

```scala
val actor = system.actorOf(Props[SampleActor], "sampleActor")
actor ! "Pretty slick"
```

在使用SampleActor 时它必须可用, i.e. actor系统的classloader中必须有一个包含这个类的JAR包。

通过设置`akka.actor.serialize-creators=on`项，所有Props的可序列化可以被测试。它的部署有`LocalScope`的Props可以免除测试。

### 用代码进行远程部署

要允许动态部署系统，也可以在用来创建actor的 Props 中包含deployment配置 : 这一部分信息与配置文件中的deployment部分是等价的, 如果两者都有，则外部配置拥有更高的优先级。

加入这些import:

```scala
import akka.actor.{ Props, Deploy, Address, AddressFromURIString }
import akka.remote.RemoteScope
```

和一个这样的远程地址:

```scala
val one = AddressFromURIString("akka.tcp://sys@host:1234")
val two = Address("akka.tcp", "sys", "host", 1234) // this gives the same
```
你可以这样要求系统在此远程结点上创建一个子actor:

```scala
val ref = system.actorOf(Props[SampleActor].
  withDeploy(Deploy(scope = RemoteScope(address))))
```

## 5 生命周期以及错误恢复模型


## 6 监听远程actor

## 7 序列化

对actor使用远程调用时你必须保证这些actor所使用的` props `和` messages `是可序列化的。 如果不能保证会导致系统产生意料之外的行为。

## 8 有远程目标的路由器

将远程调用与路由进行组合是非常实用的。

远程部署的`routees`的一个Pool可以如下配置：

```scala
akka.actor.deployment {
  /parent/remotePool {
    router = round-robin-pool
    nr-of-instances = 10
    target.nodes = ["akka.tcp://app@10.0.0.2:2552", "akka://app@10.0.0.3:2552"]
} }
```
这个配置文件的设定将会克隆定义在`remotePool`的`Props`中的actor 10次并且均匀的部署它们到两个给定的目标节点。

远程actor的一个group可以如下配置：

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

以上的配置将会发送消息到给定的远程actor paths。它需要你在远程节点上创建匹配paths的目标actor。这不是通过路由器做的。




