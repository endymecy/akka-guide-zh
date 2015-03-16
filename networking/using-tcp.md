# 使用TCP

这一节的代码片段都假设有下面的导入：

```scala
import akka.actor.{ Actor, ActorRef, Props }
import akka.io.{ IO, Tcp }
import akka.util.ByteString
import java.net.InetSocketAddress
```

所有的AKKA IO API都通过manager对象访问。当使用一个IO API时，第一步是获得一个适当manager的引用。下面的代码显示了如何获得一个TCP manager引用。

```scala
import akka.io.{ IO, Tcp }
import context.system // implicitly used by IO(Tcp)
val manager = IO(Tcp)
```

这个manager是一个actor，它为特殊的任务（如监听输入的连接）处理潜在的底层的IO资源（selector和channels）以及实例化workers。

## 1 连接

```scala
object Client {
  def props(remote: InetSocketAddress, replies: ActorRef) =
    Props(classOf[Client], remote, replies)
}
class Client(remote: InetSocketAddress, listener: ActorRef) extends Actor {
  import Tcp._
  import context.system
  IO(Tcp) ! Connect(remote)
  def receive = {
    case CommandFailed(_: Connect) =>
      listener ! "connect failed"
      context stop self
    case c @ Connected(remote, local) =>
      listener ! c
      val connection = sender()
      connection ! Register(self)
      context become {
        case data: ByteString =>
          connection ! Write(data)
        case CommandFailed(w: Write) =>
          // O/S buffer was full
          listener ! "write failed"
        case Received(data) =>
          listener ! data
        case "close" =>
          connection ! Close
        case _: ConnectionClosed =>
          listener ! "connection closed"
          context stop self
} }
}
```

连接到远程地址的第一步是发送`Connect`消息给TCP manager，除了上面所示的简单的形式存在，也有可能指定一个特定的本地`InetSocketAddress`去绑定，一个socket选项列表去应用。

> *`SO_NODELAY`（在windows中是`TCP_NODELAY`）这个socket选项在AKKA中默认是为true的。这个设置使Nagle的算法失效，使大部分程序的潜能得到了相当的提高。这个设置可以通过在`Connect`消息的socket选项列表中传递`SO.TcpNoDelay(false)`来覆盖。*

TCP manager然后要么回复一个`CommandFailed`，要么回复一个代表新连接的内部actor。这个新的actor然后发送一个`Connected`消息到`Connect`消息的原始发送方。

为了激活这个新的连接，一个`Register`消息必须发送给连接actor。通知这个actor谁将从socket中接收数据。在这个过程开始之前，连接不可以使用。并且，有一个内部的超时时间，如果没有收到`Register`消息，连接actor将会关闭它自己。

连接actor将观察注册的handler，当某一个终止时关闭连接，然后清除这个连接相关的所有内部资源。

在上面的例子中，这个actor使用`become`从未连接的操作交换到连接的操作，表明在该状态观察到的命令和事件。`CommandFailed`将在下面讨论。`ConnectionClosed`是一个trait，标记不同的连接关闭事件。最后一行用相同的方式处理所有的连接关闭事件。可以在下面了解更多细粒度的连接关闭事件。

## 2 接收连接

```scala
class Server extends Actor {
  import Tcp._
  import context.system
  IO(Tcp) ! Bind(self, new InetSocketAddress("localhost", 0))
  def receive = {
    case b @ Bound(localAddress) =>
      // do some logging or setup ...
    case CommandFailed(_: Bind) => context stop self
    case c @ Connected(remote, local) =>
      val handler = context.actorOf(Props[SimplisticHandler])
      val connection = sender()
      connection ! Register(handler)
} }
```

为了创建一个TCP服务器以及监听入站（inbound）连接，一个`Bind`命令必须发送给TCP manager。这将会通知TCP manager监听在特定`InetSocketAddress`上的TCP连接。为了绑定任意的接口，接口可以指定为0。

发送`Bind`消息的actor将接收一个`Bound`消息表明服务器已经准备接收输入连接。这个消息也包含`InetSocketAddress`，表明socket确实有限制（分解为IP地址和端口号）。

出站连接（outgoing）的处理过程与以上的处理过程是一样的。例子证明，当发送`Register`消息时，从一个特定的连接处理读可以通过命名另外一个actor为处理者来代理给这个actor。来源于系统的任何actor的写都可以发送到连接actor（发送`Connected`消息的actor）。最简化的处理如下：

```scala
class SimplisticHandler extends Actor {
  import Tcp._
  def receive = {
     case Received(data) => sender() ! Write(data)
￼    case PeerClosed     => context stop self
  }
```

出站actor的唯一的不同是内部actor管理监听端口-`Bound`消息的发送者-为`Bind`消息中的`Connected`消息观察命名为接收者的actor。当这个actor终止，监听端口也会关闭，所有与之相关的资源也会被释放；这时，存在的连接将不会被终止。

## 3 关闭连接

可以通过发送`Close`、`ConfirmedClose`或者`Abort`给连接actor来关闭连接。

`Close`将会发送一个`FIN`请求来关闭连接，但是不会等待来自远程端点的确认。待处理的写入将会被清。如果关闭成功了，监听器将会被通知一个`Closed`。

`ConfirmedClose`将通过发送一个`FIN`消息关闭连接的发送方向。但是数据将会继续被接收到直到远程的端点也关闭了连接。待处理的写入将会被清。如果关闭成功了，监听器将会被通知一个`ConfirmedClosed`。

`Abort`将会发送一个`RST`消息给远程端点立即终止连接。待处理的写入将不会被清。如果关闭成功了，监听器将会被通知一个`Aborted`。

如果连接已经被远程端点关闭了，`PeerClosed`将会被发送到监听者。缺省情况下，从这个端点而来的连接将会自动关闭。为了支持半关闭的连接，设置`Register`消息的`keepOpenOnPeerClosed`成员为`true`。在这种情况下，连接保持打开状态直到它接收到以上的关闭命令。

不管什么时候一个错误发生了，`ErrorClosed`都将会发送给监听者，强制关闭连接。

所有的关闭通知都是`ConnectionClosed`的子类型，所以不需要细粒度关闭事件的监听者可能用相同的方式处理所有的关闭事件。

## 4 写入连接

一旦一个连接已经建立了，可以从任意actor发送数据给它。数据的格式是`Tcp.WriteCommand. Tcp.WriteCommand`，它有三种不同的实现。

- Tcp.Write。最简单的`WriteCommand`实现，它包裹一个`ByteString`实例和一个“ack”事件。这个`ByteString`拥有一个或者多个块，不可变的内存大小最大为2G。
- Tcp.WriteFile。如果你想从一个文件发送`raw`数据，你用`Tcp.WriteFile`命令会非常有效。它允许你指定一个在磁盘上的字节块通过连接发送而不需要首先加载它们到JVM内存中。`Tcp.WriteFile`可以保持超过2G的数据以及一个`ack`事件（如果需要）。
- Tcp.CompoundWrite。有时你可能想集合（或者交叉）几个`Tcp.Write`或者`Tcp.WriteFile`命令到一个原子的写命令，这个命令一次性完成写到连接。`Tcp.CompoundWrite`允许你这样做并且提供了三个好处。

    - 1 如下面章节介绍的，TCP连接actor在某一时间仅仅能处理一个单个的写命令。通过合并多个写到一个`CompoundWrite`，你可以以最小的开销传递，并且不需要通过一个`ACK-based`的消息协议一个个的回复它们。
    - 2 因为一个`WriteCommand`是原子的，你可以肯定其它actor不可能“注入”其它的写命令到你组合到一个`CompoundWrite`的写命令中去。几个actor写入相同的连接是一个重要的特征，这在某些情况下很难获得。
    - 3 一个`CompoundWrite`的“子写入（sub writes）”是普通的`Write`和`WriteFile`命令，它们请求“ack”事件。这些ACKs的发出与相应地“子写入”的完成是同时。这允许你附加超过一个`Write`或者`WriteFile`。或者通过在任意的点上发送临时的ACKs去让连接actor知道传递`CompoundWrite`的进度。

## 5 限制（Throttling）写和读

TCP连接actor的基本模型没有内部缓冲（它在同一时刻只能处理一个写，意味着它可以缓冲一个写直到它完全传递到OS内核）。需要在用户层为读和写处理拥挤问题。

对于`back-pressuring`写，有三种操作模型

- ACK-based：每一个`Write`命令携带着一个任意的命令，如果这个对象不是`Tcp.NoAck`。







