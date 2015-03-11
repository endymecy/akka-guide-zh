# agents

Akka中的Agent受到[Clojure agent](http://clojure.org/agents) 的启发。

Agent 提供对独立的内存位置的异步修改。 Agent在其生命周期中绑定到一个存储位置，对这个存储位置的数据的修改仅允许在一个操作中发生。 对其进行修改的操作是函数，该函数被异步地应用于Agent的状态，其返回值成为Agent的新状态。 Agent的状态应该是不可变的。

虽然对Agent的修改是异步的，但是其状态总是可以随时被任何线程 (通过` get `或` apply`)来获得而不需要发送消息。

Agent是活动的。 对所有agent的更新操作在一个线程池的不同线程中并发执行。在每一个时刻，每一个Agent最多只有一个` send `被执行。 从某个线程派发到agent上的操作的执行次序与其发送的次序一致，但有可能与从其它（线程）源派发来的操作交织在一起。


## 1 创建agents

调用`Agent(value)`并传入它的初始值来创建agents。它提供了一个隐式的`ExecutionContext`供使用。这些例子将使用默认全局的那个`ExecutionContext`，但是花费不一样。

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import akka.agent.Agent
val agent = Agent(5)
```

## 2 读取agents的值

Agent可以用（）调用来去引用 (获取Agent的值) ：

```scala
val result = agent()
```

或者通过get方法

```scala
val result = agent.get
```
对Agent的值的读取不包括任何消息传递，立即执行。所以说虽然Agent的更新是异步的，对它的状态的读取却是同步的。

## 3 更新 Agent

你可以通过发送一个函数转换当前的值或者仅仅发送一个新值来更新一个agent。这个Agent将会并发的自动采用这个值或者函数。更新是以一种“启动然后忘掉”的方式完成的，唯一的保证是它会被执行。 至于什么时候执行则没有保证。不过从同一个线程发到Agent的操作将被顺序执行。

```scala
// send a value, enqueues this change
// of the value of the Agent
agent send 7
// send a function, enqueues this change
// to the value of the Agent
agent send (_ + 1)
agent send (_ * 2)
```

你也可以在一个独立的线程中派发一个函数来改变内部状态。这样将不使用活动线程池，可以用于长时间运行或阻塞的操作。 相应的方法是`sendOff`。 不管使用` sendOff `还是` send `都会顺序执行。

```scala
// the ExecutionContext you want to run the function on
implicit val ec = someExecutionContext()
// sendOff a function
agent sendOff longRunningOrBlockingFunction
```

所有的`send`方法也有一个相应的`alter`方法返回一个Future.

```scala
// alter a value
val f1: Future[Int] = agent alter 7
// alter a function
val f2: Future[Int] = agent alter (_ + 1)
val f3: Future[Int] = agent alter (_ * 2)
```

```scala
// the ExecutionContext you want to run the function on
implicit val ec = someExecutionContext()
// alterOff a function
val f4: Future[Int] = agent alterOff longRunningOrBlockingFunction
```

## 4 等待Agent的返回值

也可以在所有当前排队的send请求都完成以后读取值，使用 await:

```scala
implicit val timeout = Timeout(5 seconds)
val future = agent.future
val result = Await.result(future, timeout.duration)
```

## 5 一元的用法

Agent 也支持monadic操作, 这样你就可以用`for-comprehensions`对操作进行组合。 在一元的用法中, 旧的Agent不会变化，而是创建新的Agent。 这就是所谓的‘持久化’。

```scala
import scala.concurrent.ExecutionContext.Implicits.global
val agent1 = Agent(3)
val agent2 = Agent(5)
// uses foreach
for (value <- agent1)
  println(value)
// uses map
val agent3 = for (value <- agent1) yield value + 1
// or using map directly
val agent4 = agent1 map (_ + 1)
// uses flatMap
val agent5 = for {
  value1 <- agent1
  value2 <- agent2
} yield value1 + value2
```

## 6 废弃的事务agent

如果在事务中使用Agent,那么它将成为事务的一部分。Agent与Scala STM是集成在一起的，所有在事务中提交的派发操作都直到事务被提交时才执行，在重试或放弃的情况下，Agent的操作将被丢弃。 见下例：

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import akka.agent.Agent
import scala.concurrent.duration._
import scala.concurrent.stm._
def transfer(from: Agent[Int], to: Agent[Int], amount: Int): Boolean = {
  atomic { txn =>
    if (from.get < amount) false
    else {
      from send (_ - amount)
      to send (_ + amount)
      true
} }
}
val from = Agent(100)
val to = Agent(20)
val ok = transfer(from, to, 50)
val fromValue = from.future // -> 50
val toValue = to.future // -> 70
```