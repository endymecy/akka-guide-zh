# 消息派发器

Akka `MessageDispatcher`是维持 Akka Actor “运作”的部分, 可以说它是整个机器的引擎. 所有的` MessageDispatcher `实现也同时是一个` ExecutionContext`, 这意味着它们可以用来执行任何代码。

## 1 缺省派发器

在没有为 Actor作配置的情况下，一个` ActorSystem `将有一个缺省的派发器。 缺省派发器是可配置的，缺省情况下是一个拥有特定`default-executor`的`Dispatcher`。创建ActorSystem时传递了一个ExecutionContext给它，那么这个ExecutionContext作为所有派发器的默认执行者使用。如果没有给定ExecutionContext，它将会使用在`akka.actor.default-dispatcher.default-executor.fallback`中指定的执行者。缺省情况下这是一个“fork-join-executor”，在大多数情况下拥有非常好的性能。

## 2 发现派发器

派发器实现了`ExecutionContext`接口，因此可以用来运行`Future`调用。

```scala
// for use with Futures, Scheduler, etc.
implicit val executionContext = system.dispatchers.lookup("my-dispatcher")
```

## 3 为 Actor 指定派发器

如果你想为你的actor分配与缺省派发器不同的派发器，你需要做两件事情，第一件事情是配置派发器：

```scala
my-dispatcher {
  # Dispatcher 是基于事件的派发器的名称
  type = Dispatcher
  # 使用何种ExecutionService
  executor = "fork-join-executor"
  # 配置 fork join 池
  fork-join-executor {
    # 容纳基于倍数的并行数量的线程数下限
    parallelism-min = 2
    #并行数（线程） ... ceil(可用CPU数＊倍数）
    parallelism-factor = 2.0
    #容纳基于倍数的并行数量的线程数上限
    parallelism-max = 10
  }
  # Throughput 定义了线程切换到另一个actor之前处理的消息数上限
  # 设置成1表示尽可能公平.
  throughput = 100
}
```

以下是另一个使用 “thread-pool-executor” 的例子:

```scala
my-thread-pool-dispatcher {
  # Dispatcher是基于事件的派发器的名称
  type = Dispatcher
  # 使用何种 ExecutionService 
  executor = "thread-pool-executor"
  #配置线程池
  thread-pool-executor {
    #容纳基于倍数的并行数量的线程数下限
    core-pool-size-min = 2
    # 核心线程数 .. ceil(可用CPU数＊倍数）
    core-pool-size-factor = 2.0
    #容纳基于倍数的并行数量的线程数上限
    core-pool-size-max = 10
  }
  #Throughput 定义了线程切换到另一个actor之前处理的消息数上限
  # 设置成1表示尽可能公平.
  throughput = 100
}
```

然后你就可以正常的创建actor，并在部署配置中定义派发器。

```scala
￼import akka.actor.Props
val myActor = context.actorOf(Props[MyActor], "myactor")

akka.actor.deployment {
  /myactor {
    dispatcher = my-dispatcher
  }
}
```

部署配置的另一种选择是在代码中定义派发器。如果你在部署配置中定义派发器，这个值将会覆盖编程提供的参数值。    

```scala
￼￼import akka.actor.Props
  val myActor =
   context.actorOf(Props[MyActor].withDispatcher("my-dispatcher"), "myactor1")
```

> *你在withDispatcher中指定的 “dispatcherId” 其实是配置中的一个路径. 所以在这种情况下它位于配置的顶层，但你可以把它放在下面的层次，用.来代表子层次，象这样: "foo.bar.my-dispatcher"*

## 4 派发器的类型

一共有4种类型的消息派发器:

- Dispatcher

    - 这是一个基于事件的分派器，它将一个actor集合与一个线程池结合在一起，它是缺省的派发器
    - 可共享性: 无限制
    - 邮箱: 任何，为每一个Actor创建一个
    - 使用场景: 缺省派发器，Bulkheading
    - 底层使用: `java.util.concurrent.ExecutorService`，用`fork-join-executor`, `thread-pool-executor` 或者一个`akka.dispatcher.ExecutorServiceConfigurator`的全类名来指定ExecutorService。
    
- PinnedDispatcher

    - 可共享性: 无
    - 邮箱: 任何，为每个Actor创建一个
    - 使用场景: Bulkheading
    - 底层使用: 任何`akka.dispatch.ThreadPoolExecutorConfigurator`，缺省为一个`thread-pool-executor`

- BalancingDispatcher

    - 可共享性: 仅对同一类型的Actor共享
    - 邮箱: 任何，为所有的Actor创建一个
    - 使用场景: Work-sharing
    - 底层使用: `java.util.concurrent.ExecutorService`，用`fork-join-executor`, `thread-pool-executor` 或者一个`akka.dispatcher.ExecutorServiceConfigurator`的全类名来指定ExecutorService。
    - 注意你不能使用BalancingDispatcher作为路由派发器
- CallingThreadDispatcher

    - 可共享性: 无限制
    - 邮箱: 任何，每Actor每线程创建一个（需要时）
    - 使用场景: 测试
    - 底层使用: 调用的线程 (duh)
    
### 更多 dispatcher 配置的例子

配置 PinnedDispatcher:

```scala
my-pinned-dispatcher {
  executor = "thread-pool-executor"
  type = PinnedDispatcher
}
```

然后使用它：

```scala
￼val myActor =
  context.actorOf(Props[MyActor].withDispatcher("my-pinned-dispatcher"), "myactor2")
```


