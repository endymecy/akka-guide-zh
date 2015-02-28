# 配置

你可以在不定义任何配置的情况下使用Akka，这是因为Akka提供了合适的默认值。你可能需要修设置从而改变默认的行为或者适应一个特定的运行时环境。你可能需要改变的设置有如下几点：

- 日志等级和日志备份
- 开启远程访问功能
- 消息序列化
- 定义路由器
- 调度器调优

Akka 使用`Typesafe Config`库, 不管使用或不使用Akka也可以使用它来配置你自己的项目。这个库是用java写的，没有外部依赖；你最好看看它的文档 (特别是`ConfigFactory`), 以下内容是对这个文档的概括。

## 配置数据从哪里来？

Akka的所有配置信息装在`ActorSystem`的实例中, 或者换个说法, 从外界看来, `ActorSystem`是配置信息的唯一消费者. 在构造一个actor系统时，你可以传进来一个`Config object`，如果不传，就相当于传进来`ConfigFactory.load()` (使用正确的`classloader`)。这意味着将会读取classpath根目录下的所有`application.conf`, `application.json` 和`application.properties`这些文件。 然后actor系统会合并classpath根目录下的`reference.conf`来组成其内部使用的缺省配置。

```scala
appConfig.withFallback(ConfigFactory.defaultReference(classLoader))
```
其中的奥妙是代码不包含缺省值，而是依赖于随库提供的`reference.conf`中的配置。

系统属性中覆盖的配置具有最高优先级，见[HOCON 规范](https://github.com/typesafehub/config/blob/master/HOCON.md) (靠近末尾的位置). 要提醒的是应用配置—缺省为 application—可以使用`config.resource`中的属性来覆盖。

注意：

如果你编写的是一个Akka应用，把配置放在classpath根目录下的`application.conf` 中。如果你编写的是一个基于Akka的库，把配置放在jar包根目录下的`reference.conf`中。

## 自定义application.conf

一个自定义的`application.conf`可能如下所示：

```json
# In this file you can override any option defined in the reference files.
# Copy in parts of the reference files and modify as you please.
akka {
  # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
  # to STDOUT)
  loggers = ["akka.event.slf4j.Slf4jLogger"]
  # Log level used by the configured loggers (see "loggers") as soon
  # as they have been started; before that, see "stdout-loglevel"
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  loglevel = "DEBUG"
  # Log level for the very basic logger activated during ActorSystem startup.
  # This logger prints the log messages to stdout (System.out).
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  stdout-loglevel = "DEBUG"
  actor {
    provider = "akka.cluster.ClusterActorRefProvider"
    default-dispatcher {
      # Throughput for default Dispatcher, set to 1 for as fair as possible
      throughput = 10
} }
  remote {
    # The port clients should connect to. Default is 2552.
    netty.tcp.port = 4711
} }

```

## 包含文件

有时候我们需要包含其它配置文件，例如你有一个 application.conf 定义所有与依赖环境的设置，然后在具体的环境中对其中的设置进行覆盖。

通过`-Dconfig.resource=/dev.conf`指定系统属性将会加载`dev.conf`文件，这个文件包含`application.conf`

```
include "application"
akka {
  loglevel = "DEBUG"
}
```
更高级的包含和替换机制在[HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md)规范中有解释

## 配置日志

如果系统属性或配置属性`akka.log-config-on-start`设置为`on`, 那么当actor系统启动时整个配置的日志级别为INFO. 这在你不确定使用哪个配置时会有用。

如果有疑问，你也可以在用它们构造一个actor系统之前或之后很方便地了解配置对象的内容:

``` shell
Welcome to Scala version 2.10.4 (Java HotSpot(TM) 64-Bit Server VM, Java 1.6.0_27).
Type in expressions to have them evaluated.
Type :help for more information.
scala> import com.typesafe.config._
import com.typesafe.config._
scala> ConfigFactory.parseString("a.b=12")
res0: com.typesafe.config.Config = Config(SimpleConfigObject({"a" : {"b" : 12}}))
scala> res0.root.render
res1: java.lang.String =
{
    # String: 1
    "a" : {
# String: 1
"b" : 12 }
}
```
每一条设置之前的注释给出了原有设置的详情信息 (文件和行号) 以及（在参考配置中）可能出现的注释，与参考配置合并并被actor系统解析的设置可以这样显示：

```scala
￼final ActorSystem system = ActorSystem.create();
 System.out.println(system.settings());
 // this is a shortcut for system.settings().config().root().render()
```

## 关于类加载器

在配置文件的某些地方可以指定要被Akka实例化的类的全路径。这是通过Java反射来实现的，会用到类加载器。在应用窗口或OSBi包里正确地使用它并不总是容易的事，目前Akka采取的方式是每个`ActorSystem`实现存有当前线程的上下文类加载器 (如果有的话，否则使用`this.getClass.getClassLoader`的返回值) 并用它来进行所有的反射操作。这意味着把Akka放到启动类路径中会在一些莫名其妙的地方造成`NullPointerException`：这是不被支持的。


