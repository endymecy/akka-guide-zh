# 开始

## 准备工作

Akka要求你安装了 Java 1.6或更高版本到你的机器中。

## 下载

下载Akka有几种方法。你可以下载包含微内核的完整发布包（包含所有的模块）. 或者也可以从Maven或sbt从Akka Maven仓库下载对akka的依赖。

## 模块

Akka的模块化做得非常好，它为不同的功能提供了不同的Jar包。

- akka-actor – 标准Actor, 有类型Actor，IO Actor等
- akka-agent – Agents, integrated with Scala STM
- akka-camel – Apache Camel集成
- akka-cluster – 集群关系管理, elastic routers.
- akka-kernel – Akka微内核，可运行一个基本的最小应用服务器
- akka-osgi – 在OSGi容器中使用的基本绑定, 包括akka-actor类
- akka-osgi-aries – 为供应actor的系统提供的Aries蓝图
- akka-remote – 远程Actor
- akka-slf4j – SLF4J Logger(事件总线监听器)
- akka-testkit – 用于测试Actor的工具包
- akka-zeromq – ZeroMQ集成
- akka-contrib – an assortment of contributions which may or may not be moved into core modules

## 使用发布版

从[官网](http://akka.io/downloads)下载发布包并解压。

## 使用快照版

Akka的每日快照发布在[http://repo.akka.io/snapshots/](http://repo.akka.io/snapshots/) 版本号中包含 SNAPSHOT 和时间戳. 你可以选择一个快照版，可以决定何时升级到一个新的版本。Akka快照仓库也可以在 [http://repo.typesafe.com/typesafe/snapshots/](http://repo.typesafe.com/typesafe/snapshots/)找到，此处还包含Akka模块依赖的其它仓库。

## 微内核

Akka发布包包含微内核。要运行微内核，将你的应用的jar包放到 deploy 目录下并运行 bin 目录下的脚本.

## 与Maven一起使用Akka

```xml
<dependency>
  <groupId>com.typesafe.akka</groupId>
  <artifactId>akka-actor_2.10</artifactId>
  <version>2.3.9</version>
</dependency>
```

## 与SBT一起使用Akka

SBT安装指导 [https://github.com/harrah/xsbt/wiki/Setup](https://github.com/harrah/xsbt/wiki/Setup)

build.sbt 文件:

```scala
name := "My Project"
version := "1.0"
scalaVersion := "2.10.4"
resolvers += "Typesafe Repository" at "http://repo.typesafe.com/typesafe/releases/"
libraryDependencies +="com.typesafe.akka" %% "akka-actor" % "2.3.9"
```

注意：以上libraryDependencies的设置仅仅适用于sbt v0.12.x或者更高版本。如果是以前的版本，应通过如下方式设置

```scala
￼libraryDependencies +="com.typesafe.akka" % "akka-actor_2.10" % "2.3.9"
```
## 与gradle一起使用Akka

gradle版本至少是1.4 

```
apply plugin: ’scala’
repositories {
  mavenCentral()
}
dependencies {
  compile ’org.scala-lang:scala-library:2.10.4’
}
tasks.withType(ScalaCompile) {
  scalaCompileOptions.useAnt = false
}
dependencies {
￼  compile group: ’com.typesafe.akka’, name: ’akka-actor_2.10’, version: ’2.3.9’
   compile group: ’org.scala-lang’, name: ’scala-library’, version: ’2.10.4’
}
```