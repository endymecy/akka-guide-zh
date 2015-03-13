# 序列化

Akka 提供内置的序列化支持扩展, 你可以选择使用内置的序列化功能，也可以自己写一个。

内置的序列化功能被Akka内部用来序列化消息，你也可以用它做其它的序列化工作。

## 1 用法

### 配置

为了让 Akka 知道对什么任务使用哪个` Serializer `， 你需要编辑你的配置文件， 在`akka.actor.serializers` 部分将名称与` akka.serialization.Serializer`的实现做绑定，象这样：

```scala
akka { actor {
    serializers {
      java = "akka.serialization.JavaSerializer"
      proto = "akka.remote.serialization.ProtobufSerializer"
      myown = "docs.serialization.MyOwnSerializer"
} }
}
```

名称与` Serializer `的不同实现绑定后你需要指定哪些类的序列化需要使用哪种` Serializer`，这部分配置写在`akka.actor.serialization-bindings`部分：

```scala
akka { actor {
    serializers {
      java = "akka.serialization.JavaSerializer"
      proto = "akka.remote.serialization.ProtobufSerializer"
      myown = "docs.serialization.MyOwnSerializer"
}
    serialization-bindings {
      "java.lang.String" = java
      "docs.serialization.Customer" = java
      "com.google.protobuf.Message" = proto
      "docs.serialization.MyOwnSerializable" = myown
      "java.lang.Boolean" = myown
} }
}
```

你只需要指定消息的接口或抽象基类。当消息实现了配置中多个类时，为避免歧义, 将使用最具体类，如其它所有类的子类。如果这个条件不满足，如`java.io.Serializable `和 `MyOwnSerializable `都有配置，而彼此都不是对方的子类型，将生成警告。

Akka 缺省提供使用` java.io.Serializable` 和 [protobuf](http://code.google.com/p/protobuf/) `com.google.protobuf.GeneratedMessage `的序列化工具 (后者仅当定义了对` akka-remote `模块的依赖时才有)， 所以通常你不需要添加这两种配置。由于` com.google.protobuf.GeneratedMessage `实现了` java.io.Serializable`, 在不特别指定的情况下，`protobuf` 消息将总是用` protobuf `协议来做序列化。 要禁止缺省的序列化工具，将其对应的类型设为 “none”:

```scala
￼akka.actor.serialization-bindings {
  "java.io.Serializable" = none
}
```

### 确认

如果你希望确认你的消息是可以被序列化的你可以打开这个配置项:

```scala
akka { actor {
    serialize-messages = on
  }
}
```

> *警告：我们只推荐在运行测试代码的时候才打开这个选项。在其它的场景打开它完全没有道理。*

如果你希望确认你的` Props `可以被序列化你可以打开这个配置项:

```scala
akka { actor {
    serialize-creators = on
  }
}
```
> *警告：我们只推荐在运行测试代码的时候才打开这个选项。在其它的场景打开它完全没有道理。*

### 通过代码

如果你希望通过代码使用 Akka 序列化来进行序列化/反序列化, 以下是一些例子:

```scala
import akka.serialization._
import com.typesafe.config.ConfigFactory

    val system = ActorSystem("example")

    // 获取序列化扩展工具
    val serialization = SerializationExtension(system)

    // 定义一些要序列化的东西
    val original = "woohoo"

    // 为它找到一个 Serializer
    val serializer = serialization.findSerializerFor(original)

    // 转化为字节
    val bytes = serializer.toBinary(original)

    // 转化回对象
    val back = serializer.fromBinary(bytes, manifest = None)

    // 就绪!
    back should be(original)
```

## 2 自定义

你希望创建自己的` Serializer`， 应该已经看到上例中的` akka.docs.serialization.MyOwnSerializer `了吧？

### 创建新的 Serializer

首先你需要为你的 Serializer 写一个类定义，象这样:

```scala
import akka.serialization._
import com.typesafe.config.ConfigFactory
 
class MyOwnSerializer extends Serializer {
 
  // 指定 "fromBinary" 是否需要一个 "clazz" 
  def includeManifest: Boolean = false
 
  // 为你的 Serializer 选择一个唯一标识,
  // 基本上所有的整数都可以用,
  // 但 0 - 16 是Akka自己保留的
  def identifier = 1234567
 
  // "toBinary" 将对象序列化为字节数组 Array of Bytes
  def toBinary(obj: AnyRef): Array[Byte] = {
    // 将序列化的代码写在这儿
    // ... ...
  }
 
  // "fromBinary" 对字节数组进行反序列化,
  // 使用类型提示 (如果有的话, 见上文的 "includeManifest" )
  // 使用可能提供的 classLoader.
  def fromBinary(bytes: Array[Byte],
                 clazz: Option[Class[_]]): AnyRef = {
    // 将反序列化的代码写在这儿
    // ... ...
  }
}
```

然后你只需要填空，在配置文件中将它绑定到一个名称， 然后列出需要用它来做序列化的类即可。

### Actor引用的序列化

所有的` ActorRef `都是用` JavaSerializer`， 但如果你写了自己的serializer， 你可能想知道如何正确对它们进行序列化和反序列化。在一般情况下，使用的本地地址依赖于远程地址的类型，这个远程地址是序列化的信息更容易接受。如下使用`Serialization.serializedActorPath(actorRef)`。

```scala
import akka.actor.{ ActorRef, ActorSystem }
import akka.serialization._
import com.typesafe.config.ConfigFactory
    // Serialize
    // (beneath toBinary)
    val identifier: String = Serialization.serializedActorPath(theActorRef)
    // Then just serialize the identifier however you like
    // Deserialize
    // (beneath fromBinary)
    val deserializedActorRef = extendedSystem.provider.resolveActorRef(identifier)
    // Then just use the ActorRef
```

这假设序列化发生在通过远程传输发送消息的上下文。序列化也有其它的使用，如存储actor应用到一个actor应用程序的外面（如数据库）。在这种情况下，记住一个actor路径的地址部分决定了这个actor如何通讯是非常重要的。如果检索发生在相同的逻辑上下文，存储本地actor路径可能是更好的选择。但是，当在不同的网络主机上面反序列化它时，这是不够的：因为这需要它需要包含系统的远程传输地址。一个actor地址并没有限制只有一个远程传输地址，这使这个问题变得更加有趣。为了找到合适的地址使用，当发送`remoteAddr`时，你可以使用`ActorRefProvider.getExternalAddressFor(remoteAddr)`。

```scala
object ExternalAddress extends ExtensionKey[ExternalAddressExt]
class ExternalAddressExt(system: ExtendedActorSystem) extends Extension {
  def addressFor(remoteAddr: Address): Address =
    system.provider.getExternalAddressFor(remoteAddr) getOrElse
      (throw new UnsupportedOperationException("cannot send to " + remoteAddr))
}
def serializeTo(ref: ActorRef, remote: Address): String =
  ref.path.toSerializationFormatWithAddress(ExternalAddress(extendedSystem).
    addressFor(remote))
```

> *注：如果地址还没有`host`和`port`组件，也就是说它仅仅为本地地址插入地址信息，`ActorPath.toSerializationFormatWithAddress `和`toString`是不同的。`toSerializationFormatWithAddress`还需要添加actor的唯一的id，这个id会随着actor的停止然后以相同的名字重新创建而改变。发送消息到指向旧actor的引用将不会传送到新的actor。如果你不想这种行为，你可以使用`toStringWithAddress`，它不包含这个唯一的id*

这需要你至少知道将要反序列化结果actor引用的系统支持哪种类型的地址。如果你没有具体的地址，你可以使用`Address(protocol, "", "", 0)`为正确的协议创建一个虚拟的地址。

也有一个缺省的远程地址被集群支持（仅仅只有特殊的系统有）。你可以像下面这样得到它

```scala
object ExternalAddress extends ExtensionKey[ExternalAddressExt]
class ExternalAddressExt(system: ExtendedActorSystem) extends Extension {
  def addressForAkka: Address = system.provider.getDefaultAddress
}
def serializeAkkaDefault(ref: ActorRef): String =
  ref.path.toSerializationFormatWithAddress(ExternalAddress(theActorSystem).
    addressForAkka)
```

## 3 关于 Java 序列化

如果在做Java序列化任务时不使用` JavaSerializer `， 你必须保证在动态变量`JavaSerializer.currentSystem`中提供一个有效的` ExtendedActorSystem `。 它是在读取`ActorRef`时将字符串表示转换成实际的引用。 动态变量`DynamicVariable `是一个` thread-local`变量，所以在反序列化任何可能包含actor引用的数据时要保证这个变量有值。


