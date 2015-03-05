# 有限状态机

## 1 概述

FSM (有限状态机) 可以mixin到akka Actor中 它的概念在[Erlang 设计原则](http://www.erlang.org/documentation/doc-4.8.2/doc/design_principles/fsm.html)中有最好的描述。

一个 FSM 可以描述成一组具有如下形式的关系 :

> State(S) x Event(E) -> Actions (A), State(S’)

这些关系的意思可以这样理解 :

> 如果我们当前处于状态S，发生了E事件, 我们应执行操作A，然后将状态转换为S’。

## 2 一个简单的例子

为了演示 FSM trait 的大部分功能, 考虑一个actor，它接收到一组突然爆发的 消息而将其送入邮箱队列，然后在消息爆发期过后或收到flush请求时对消息进行发送。

首先，假设以下所有代码都使用这些import语句:

```scala
import akka.actor.{ Actor, ActorRef, FSM }
import scala.concurrent.duration._
```

我们的`Buncher` actor会接收和发送以下消息:

```scala
// received events
case class SetTarget(ref: ActorRef)
case class Queue(obj: Any)
case object Flush
// sent events
case class Batch(obj: immutable.Seq[Any])
```

`SetTarget`用来启动，为`Batches` 设置发送目标; `Queue` 添加数据到内部队列而` Flush `标志着消息爆发的结束。

```scala
// states
sealed trait State
case object Idle extends State
case object Active extends State
sealed trait Data
case object Uninitialized extends Data
case class Todo(target: ActorRef, queue: immutable.Seq[Any]) extends Data
```

这个actor可以处于两种状态: 队列中没有消息 (即`Idle`) 或有消息 (即` Active`)。只要一直有消息进来并且没有flush请求，它就停留在active状态。这个actor的内部状态数据是由批消息的发送目标actor引用和实际的消息队列组成。

现在让我们看看我们的FSM actor的框架:

```scala
class Buncher extends Actor with FSM[State, Data] {

  startWith(Idle, Uninitialized)

  when(Idle) {
    case Event(SetTarget(ref), Uninitialized) => stay using Todo(ref, Vector.empty)
  }

  // 此处省略转换过程 ...

  when(Active, stateTimeout = 1 second) {
    case Event(Flush | FSM.StateTimeout, t: Todo) ＝> goto(Idle) using t.copy(queue = Vector.empty)
  }

  // 此处省略未处理消息 ...

  initialize
}
```

基本方法就是声明actor类, 混入`FSM` trait将可能的状态和数据作为类型参数。在actor的内部使用一个DSL来声明状态机。

- `startsWith` 定义初始状态和初始数据
- 然后对每一个状态有一个` when(<state>) { ... }` 声明待处理的事件(可以是多个，PartialFunction将用`orElse`进行连接)进行处理
- 最后使用 initialize来启动它, 这会在初始状态中执行转换并启动定时器（如果需要的话）。

在这个例子中，我们从 `Idle` 和` Uninitialized `状态开始, 这两种状态下只处理` SetTarget() `消息; `stay `准备结束这个事件的处理而不离开当前状态, 而`using`使得FSM将其内部状态(这时为`Uninitialized` ) 替换为一个新的包含目标actor引用的 `Todo() `对象。Active 状态声明了一个状态超时, 意思是如果1秒内没有收到消息, 将生成一个` FSM.StateTimeout `消息。在本例中这与收到` Flush `指令消息有相同的效果, 即转回` Idle` 状态并将内部队列重置为空vector。 但消息是如何进入队列的？ 由于在两种状态下都要做这件事， 我们利用了任何未被` when()` 块处理的消息发送到了` whenUnhandled() `块这个事实。

```scala
whenUnhandled {
  // 两种状态的通用代码 
  case Event(Queue(obj), t @ Todo(_, v)) =>
    goto(Active) using t.copy(queue = v :+ obj)
 
  case Event(e, s) ＝>
    log.warning("received unhandled request {} in state {}/{}", e, stateName, s)
    stay
}
```

这里第一个case是将 `Queue()` 请求加入内部队列中并进入 `Active` 状态 (当然如果已经在` Active `状态则保留), 前提是收到 `Queue()`时FSM数据不是` Uninitialized` 。 否则—在另一个case中—记录一个警告到日志并保持内部状态。

最后剩下的只有` Batches `是如何发送到目标的, 这里我们使用` onTransition `机制: 你可以声明多个这样的块，在状态切换发生时（i.e. 只有当状态真正改变时） 所有的块都将被尝试来作匹配。

```scala
onTransition {
  case Active -> Idle ＝>
    stateData match {
      case Todo(ref, queue) => ref ! Batch(queue)
    }
}
```

状态转换回调是一个以一对状态作为输入的函数 —— 当前状态和新状态. FSM trait为此提供了一个方便的箭头形式的extractor, 非常贴心地提醒你所匹配到的状态转换的方向。在状态转换过程中, 可以看到旧状态数据可以通过 stateData获得, 新状态数据可以通过 nextStateData获得。

要确认这个调制器真实可用，可以写一个测试将`ScalaTest trait`融入`AkkaSpec`中:

```scala
import akka.testkit.AkkaSpec
import akka.actor.Props

class FSMDocSpec extends AkkaSpec {

  "simple finite state machine" must {
    // 此处省略fsm代码 ...

    "batch correctly" in {
      val buncher = system.actorOf(Props(new Buncher))
      buncher ! SetTarget(testActor)
      buncher ! Queue(42)
      buncher ! Queue(43)
      expectMsg(Batch(Seq(42, 43)))
      buncher ! Queue(44)
      buncher ! Flush
      buncher ! Queue(45)
      expectMsg(Batch(Seq(44)))
      expectMsg(Batch(Seq(45)))
    }

    "batch not if uninitialized" in {
      val buncher = system.actorOf(Props(new Buncher))
      buncher ! Queue(42)
      expectNoMsg
    }
  }
}
```

## 3 参考

### FSM Trait 和 FSM Object

FSM trait直接继承自actor。当你希望继承FSM，你必须知道一个actor已经创建。

```scala
class Buncher extends FSM[State, Data] {
  // fsm body ...
  initialize()
}
```
> *注：FSM trait定义了一个receive方法，这个方法处理内部消息，通过FSM逻辑传递其它的东西（根据当前的状态）。当重写receive方法时，记住状态过期处理依赖于通过FSM逻辑传输的消息*

FSM trait 有两个类型参数 :

- 所有状态名称的父类型，通常是一个sealed trait，状态名称作为case object来继承它。
- 状态数据的类型，由 FSM 模块自己跟踪。

> *包含状态名字的状态数据描述了状态机的内部状态，如果你坚持这个schema，并且不添加可变字段到FSM类，you have the advantage of making all changes of the internal state explicit in a few well-known places*

### 定义状态

状态的定义是通过一次或多次调用`when(<name>[, stateTimeout = <timeout>])(stateFunction)`方法

给定的名称对象必须与为FSM trait指定的第一个参数类型相匹配. 这个对象将被用作一个hash表的键, 所以你必须确保它正确地实现了`equals`和`hashCode`方法; 特别是它不能是可变量。 满足这些条件的最简单的就是` case objects`。

如果给定了` stateTimeout` 参数, 那么所有到这个状态的转换，包括停留, 缺省都会收到这个超时。初始化转换时显式指定一个超时可以用来覆盖这个缺省行为。在操作执行的过程中可以通过` setStateTimeout(state, duration)`来修改任何状态的超时时间. 这使得运行时配置（e.g. 通过外部消息）成为可能。

参数` stateFunction` 是一个` PartialFunction[Event, State]`, 用偏函数的语法来指定，见下例:

```scala
when(Idle) {
  case Event(SetTarget(ref), Uninitialized) =>
    stay using Todo(ref, Vector.empty)
}
when(Active, stateTimeout = 1 second) {
  case Event(Flush | StateTimeout, t: Todo) =>
    goto(Idle) using t.copy(queue = Vector.empty)
}
```

`Event(msg: Any, data: D)`这个case类以FSM所持有的数据类型为参数，以便进行模式匹配。

推荐声明状态为一个继承自封闭trait的对象，然后验证每个状态有一个`when`子句。如果你想以`unhandled`状态离开处理，可以像下面这样声明。

```scala
when(SomeState)(FSM.NullFunction)
```

### 定义初始状态

每个FSM都需要一个起点, 用`startWith(state, data[, timeout])`来声明。

可选的超时参数将覆盖所有为期望的初始状态指定的值。想要取消缺省的超时, 使用 Duration.Inf。

### 未处理事件

如果一个状态未能处理一个收到的事件，日志中将记录一条警告. 这种情况下如果你想做点其它的事，你可以指定 `whenUnhandled(stateFunction)`:

```scala
whenUnhandled {
  case Event(x: X, data) =>
    log.info("Received unhandled event: " + x)
    stay
  case Event(msg, _) =>
    log.warning("Received unknown event: " + msg)
    goto(Error)
}
```

重要: 这个处理器不会叠加，每一次启动` whenUnhandled `都会覆盖先前指定的处理器。

### 发起状态转换

任何`stateFunction`的结果都必须是下一个状态的定义，除非FSM正在被终止, 这种情况在[从内部终止]中介绍。 状态定义可以是当前状态, 由 stay 指令描述, 或由 goto(state)指定的另一个状态。 结果对象可以通过下面列出的修饰器作进一步限制:

- `forMax(duration)`

这个修饰器为新状态指定状态超时。这意味着将启动一个定时器，它过期时将向FSM发送一个`StateTimeout`消息。 其间接收到任何其它消息时定时器将被取消; 你可以确定的事实是`StateTimeout`消息不会在任何一个中间消息之后被处理。

这个修饰器也可以用于覆盖任何对目标状态指定的缺省超时。如果要取消缺省超时，用`Duration.Inf`。

- `using(data)`

这个修饰器用给定的新数据取代旧的状态数据. 如果你遵循上面的建议, 这是内部状态数据被修改的唯一位置。

- `replying(msg)`

这个修饰器为当前处理的消息发送一个应答，不同的是它不会改变状态转换

所有的修饰器都可以链式调用来获得优美简洁的表达方式:

```scala
when(SomeState) {
  case Event(msg, _) =>
    goto(Processing) using (newData) forMax (5 seconds) replying (WillDo)
}
```

事实上这里所有的括号都不是必须的, 但它们在视觉上将修饰器和它们的参数区分开，因而使代码对于他人有更好的可读性。

> *注意：请注意` return `语句不可以用于` when`或类似的代码块中; 这是Scala的限制。 要么重构你的代码，使用` if () ... else ... `或将它改写到一个方法定义中。*

### 监控状态转换

在概念上，转换发生在 “两个状态之间” , 也就是在你放在事件处理代码块执行的任何操作之后; 这是显然的，因为只有在事件处理逻辑返回了值以后才能确定新的状态。 你不需要担心相对于设置内部状态变量的顺序的细节，因为FSM actor中的所有代码都是在一个线程中运行的。

#### 内部监控

到目前为止，FSM DSL都围绕着状态和事件。 另外一种视角是将其描述成一系列的状态转换。 方法`onTransition(handler)`将操作与状态转换而不是状态或事件联系起来。 这个处理器是一个偏函数，它以一对状态作为输入; 不需要结果状态因为不可能改变正在进行的状态转换。

```scala
onTransition {
  case Idle -> Active => setTimer("timeout", Tick, 1 second, true)
  case Active -> _    => cancelTimer("timeout")
  case x -> Idle      => log.info("entering Idle from " + x)
}
```

`->` 用来以清晰的形式解开状态对并表达了状态转换的方向。 与通常的模式匹配一样, 可以用` _ `来表示不关心的内容; 或者你可以将不关心的状态绑定到一个变量, e.g. 像上一个例子那样供记日志使用。

也可以向 onTransition给定一个以两个状态为参数的函数, 例如你的状态转换处理逻辑是定义成方法的:

```scala
onTransition(handler _)
def handler(from: StateType, to: StateType) {
  // handle it here ...
}
```

用这个方法注册的处理器是迭加的, 这样你可以将 onTransition 块和 when 块分散定义以适应设计的需要. 但必须注意的是，每一次状态转换都会调用所有的处理器, 而不是最先匹配的那个。 这是一个故意的设计，使得你可以将某一部分状态转换处理放在某一个地方而不用担心先前的定义会屏蔽后面的；当然这些操作还是按定义的顺序执行的。

> *注意：这种内部监控可以用于通过状态转换来构建你的FSM，这样在添加新的目标状态时不会忘记。例如在离开某个状态时取消定时器这种操作。*
