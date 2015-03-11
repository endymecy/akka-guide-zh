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

所有的` ActorRef `都是用` JavaSerializer`， 但如果你写了自己的serializer， 你可能想知道如果正确对它们进行序列化和反序列化。



