# Akka和Java内存模型

使用包含Scala和Akka在内的Typesafe Stack的主要的好处是它简化了编写并发软件的过程。本文将讨论Typesafe Stack，尤其是Akka是如何在并发应用中访问共享内存的。

## Java内存模型

在Java 5之前，Java内存模型（JMM）是混乱的。当多个线程访问共享内存时很可能得到各种奇怪的结果，例如：

- 一个线程看不到其它线程所写入的值：可见性问题
- 由于指令没有按期望的顺序执行，一个线程观察到其它线程的 ‘不可能’ 行为：指令乱序问题

随着Java 5中JSR 133的实现，很多这种问题都被解决了。 JMM是一组基于 “发生在先” 关系的规则, 限制了内存访问行为何时必须在另一个内存访问行为之前发生，以及反过来，它们何时能够不按顺序发生。这些规则的两个例子包括：

- 监视器锁规则： 对一个锁的释放先于所有后续对同一个锁的获取
- volatile变量规则: 对一个volatile变量的写操作先于所有对同一volatile变量的后续读操作

volatile变量规则: 对一个volatile变量的写操作先于所有对同一volatile变量的后续读操作

## Actors 与 Java 内存模型

使用Akka中的Actor实现，有两种方法让多个线程对共享的内存进行操作：

- 如果一条消息被（例如，从另一个actor）发送到一个actor，大多数情况下消息是不可变的，但是如果这条消息不是一个正确创建的不可变对象，如果没有 “发生先于” 规则, 有可能接收方会看到部分初始化的数据，甚至可能看到无中生有的数据（long/double）。
- 如果一个actor在处理消息时改变了自己的内部状态，而之后又以处理其它消息时访问了这个状态。而重要的是在使用actor模型时你无法保证同一个线程在处理不同的消息时使用同一个actor。

为了避免actor中的可见性和乱序问题，Akka保证以下两条 “发生在先” 规则:

- actor发送规则 : 一条消息的发送动作先于同一个actor对同一条消息的接收
- actor后续处理规则: 一条消息的处理先于同一个actor的下一条消息处理

这两条规则都只应用于同一个actor实例，如何使用不同的actor则无效。

## Futures 与 Java内存模型

一个Future的完成 “先于” 任何注册到它的回调函数的执行。

我们建议不要捕捉（close over）非final的值 (Java中称final，Scala中称val), 如果你 一定 要捕捉非final的值, 它们必须被标记为 volatile 来让它的当前值对回调代码可见。

如果你捕捉一个引用， 你还必须保证它所指代的实例是线程安全的。 我们强烈建议远离使用锁的对象，因为它们会引入性能问题，甚至可能造成死锁。 这些是使用synchronized的风险。

## STM 与 Java内存模型

Akka中的软件事务性内存 (STM) 也提供了一条 “发生在先” 规则:

- 事务性引用规则: 在提交过程中对一个事务性引用的成功的写操作先于所有对同一事务性引用的后续读操作发生。

这条规则非常象JMM中的 ‘volatile 变量’ 规则. 目前Akka STM只支持延迟写，所以对共享内存的实际写操作会被延迟到事务提交之时。事务中的写操作被存放在一个本地缓冲区中 (事务的写操作集) ，对其它的事务是不可见的。这就是为什么脏读是不可能的。

这些规则在Akka中的实现会随时间而变化，精确的细节甚至可能依赖于所使用的配置。但是它们是建立在其它的JMM规则如监视器锁规则和volatile变量规则基础上的。 这意味着Akka用户不需要操心为了提供“发生先于”关系而增加同步，因为这是Akka的工作。这样你可以腾出手来处理你的业务逻辑，让Akka框架来保证这些规则的满足。

## Actor 与 共享的可变状态
因为Akka运行在JVM上，所以还有一些其它的规则需要遵守。

- 捕捉Actor内部状态并暴露给其它线程

```scala
class MyActor extends Actor {
 var state = ...
 def receive = {
    case _ =>
      //错误的做法

    // 非常错误，共享和可变状态，
    // 会让应用莫名其妙地崩溃
      Future { state = NewState }
      anotherActor ? message onSuccess { r => state = r }

    // 非常错误, "发送者" 随每个消息改变
    // 共享可变状态 bug
      Future { expensiveCalculation(sender()) }

      //正确的做法

    // 非常安全， "self" 被捕捉是安全的
    // 并且它是一个Actor引用, 是线程安全的
      Future { expensiveCalculation() } onComplete { f => self ! f.value.get }

    // 非常安全，我们捕捉了一个固定值
    // 并且它是一个Actor引用，是线程安全的
      val currentSender = sender()
      Future { expensiveCalculation(currentSender) }
 }
}
```
- 消息应当是不可变的, 这是为了避开共享可变状态的陷阱。
