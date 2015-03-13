# IO

## 1 简介

`akka.io`包在Akka和[spray.io](http://spray.io/)组合作开发。它设计时整合了`spray-io`模块的经验以及为作为基于actor的服务的更一般的消费共同开发的改进。

这个IO实现的设计指南的目标是达到极端的可扩展性，毫无妥协地提供了底层传输机制相匹配的正确的API，完全的事件驱动、无阻塞、并发。该API对网络协议的实现以及构建更高的抽像提供了坚实的基础。对用户来说，它不是一个全服务、高级别的NIO包装。

## 2 术语，概念

IO API是完全基于actor的，意味着所有的操作都利用消息传递代替直接方法调用来实现。每个IO驱动（TCP和UDP）都有特定的actor，叫做`manager`，担当API的入口点。IO被分解到几个驱动。一个特殊驱动的manager可以通过IO入口点访问。例如，下面的例子发现一个TCP manager并返回它的`ActorRef`。

```scala
import akka.io.{ IO, Tcp }
import context.system // implicitly used by IO(Tcp)
val manager = IO(Tcp)
```

这个manager接收IO命令消息，回应实例化的工作actor。工作actor在回复中将它们自己返回给API用户说明命令已经传输。例如，当`Connect`命令发送给TCP manager后，这个manager创建一个actor代表TCP连接。所有和给定TCP连接相关的操作都会发送消息给连接actor，这个actor通过发送`Connected`消息告知它自己。

### DeathWatch和资源管理

IO工作actor接收命令并且也发送事件。它们通常需要用户端副本（counterpart）actor监听这些事件（这些事件可以是入站连接、输入字节或者写确认）。这些工作actor观察它们的监听器副本。如果监听器停止，那么工作actor将会自动的释放它们持有的所有资源。这个设计使API在应对资源泄露时更强健。

### 写模式（Ack，Nack）

IO设备有一个吞吐量最大值，这个值限制了写的次数和频率。当一个应用程序尝试压比设备能够处理更多的数据，驱动器必须缓冲这些数据直到设备能够写它们。拥有缓冲，这使短时间的密集写成为可能-但是没有无限的缓冲。需要“Flow control”来避免设备缓冲过载。

AKKA支持两种类型的流控制

- Ack-based，当写成功时，驱动器通知writer
- Nack-based，当写失败时，驱动器通知writer

这两种模型在AKKA IO的TCP和UDP实现中都可以使用。

个别的写可以通过在写消息中提供一个ack对象来确认。当写完成后，worker将会发送ack对象到写actor。这可以用来实现Ack-based的流控制；只有当旧的数据已经被确认了之后，才能发送新数据。

如果一个写（或者其他命令）失败了，driver用一个特殊的消息（在TCP和UDP中是`CommandFailed`）通知发送命令的actor。这个消息也会通知失败写的writer，作为那个写的否定应答。请注意，在一个基于NACK的流量控制中，设定了这样一个事实，那就是故障写可能不是它发送的最新的写。例如，`w1`写命令的失败通知可能在写命令`w2`以及`w3`发送之后到达。如果writer想重发任何NACK消息，它可能需要保持一个等候操作的缓冲区。

> *一个确认写并不意味着确认交付或者存储，收到一个写的回复简单地表示IO driver已经成功的处理了写。这里描述的Ack/Nack协议是流控制的手段不是错误处理*


### ByteString

Akka IO 模块的主要目标是actor之间的通信只使用不可变对象。 在jvm上处理网络IO时，常使用` Array[Byte] `和` ByteBuffer `来表示Byte集合, 但它们是可变的。Scala的集合库也缺乏一个合适的高效不可变Byte集合类。安全高效地移动Byte对IO模块是非常重要的，所以我们开发了`ByteString`。

`ByteString` 是一个[象绳子一样的](http://en.wikipedia.org/wiki/Rope_(computer_science))数据结构，它不可变而高效。 当连接两个`ByteString`时是将两者都保存在结果`ByteString`内部而不是将它们复制到新的Array。像` drop `和` take `这种操作返回的`ByteString`仍引用之前的Array, 只是改变了外部可见的offset和长度。我们花了很大力气保证内部的Array不能被修改。 每次当不安全的` Array `被用于创建新的` ByteString `时，会创建一个防御性拷贝。

`ByteString `继承所有` IndexedSeq`的方法, 并添加了一些新的。 要了解更多信息, 请查阅` akka.util.ByteString `类和它的伴生对象的ScalaDoc。

`ByteString`也带有优化的构建器`ByteStringBuilder`和迭代类`ByteIterator`，它们提供了额外的特征。

### 和java.io的兼容性

可以通过`asOutputStream`方法将一个`ByteStringBuilder`包裹在一个`java.io.OutputStream`中。同样，可以通过`asInputStream`方法将一个`ByteIterator`包裹在一个`java.io.InputStream`中。使用这些，`akka.io`应用程序可以集成基于`java.io`的遗留代码。

