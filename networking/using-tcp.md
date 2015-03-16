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

