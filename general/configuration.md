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

