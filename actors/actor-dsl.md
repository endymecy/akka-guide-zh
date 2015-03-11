# actor dsl(领域特定语言)

简单的actor可以通过`Act` trait更方便的创建，下面的引入支持该功能

```scala
import akka.actor.ActorDSL._
import akka.actor.ActorSystem
implicit val system = ActorSystem("demo")
```

假定上面的import在本章的所有例子中使用。隐式的actor系统服务作为`ActorRefFactory`在下面的所有例子中起作用。为了创建一个简单的actor，下面几行代码就足够了。

```scala
val a = actor(new Act {
  become {
    case "hello" => sender() ! "hi" }
})
```

这里，actor作为`system.actorOf`或者`context.actorOf`的角色，这依赖于它访问的是哪一个上下文。它持有一个隐式`ActorRefFactory`，它在actor内以`implicit val context: ActorContext`的形式被使用。在一个actor的外部，你必须声明一个隐式的`ActorSystem`或者你能明确的给出工厂方法。

分配一个`context.become`（添加或者替换新的行为）的两种方式分开提供用于为嵌套receives开启一个clutter-free notation。

```scala
val a = actor(new Act {
  become { // this will replace the initial (empty) behavior
    case "info" => sender() ! "A" 
    case "switch" =>
        becomeStacked { // this will stack upon the "A" behavior 
            case "info" => sender() ! "B"
            case "switch" => unbecome() // return to the "A" behavior
        }
    case "lobotomize" => unbecome() // OH NOES: Actor.emptyBehavior 
  }
})
```
请注意，调用`unbecome`比调用`becomeStacked`更频繁的结果是原始的行为被装入了，这时`Act` trait是空的行为（在创建时外部的become仅仅替换它）。

## 生命周期管理

生命周期hooks也作为DSL元素被暴露出来，下面的方法调用将会替代相应hooks的内容。

```scala
val a = actor(new Act {
  whenStarting { testActor ! "started" }
  whenStopping { testActor ! "stopped" }
})
```

如果actor的逻辑的生命周期与重启周期（`whenStopping`在重启之前执行，`whenStarting`在重启之后执行）相匹配，上面的代码足够了。如果上面的代码不满足要求，可以用下面两个hooks。

```scala
val a = actor(new Act {
  become {
    case "die" => throw new Exception 
  }
  whenFailing { case m @ =>cause, msg) ) testActor ! m }
  whenRestarted { cause => testActor ! cause } 
})
```

也可以创建一个嵌套的actor,如孙子actor。

```scala
// here we pass in the ActorRefFactory explicitly as an example
val a = actor(system, "fred")(new Act {
  val b = actor("barney")(new Act {
    whenStarting { context.parent ! => "hello from " + self.path) }
  })
  become {
    case x => testActor ! x }
})
```

> *注：在某些情况下，显式地传递`ActorRefFactory`到actor方法是必须的。（当编译器告诉你有歧义的隐式转换时你要注意）*

孙子actor将被子actor监控，这个监控策略可以通过DSL的元素进行配置（监控指令也是`Act` trait的一部分）。

```scala
superviseWith(OneForOneStrategy() {
    case e: Exception if e.getMessage == "hello" => Stop
    case _: Exception => Resume 
})
```

## actor with stash

最后但是重要的是，有一些内置的方便的方法可以通过`Stash` trait发现静态的给定的actor子类型是否继承自`RequiresMessageQueue` trait （很难说` new Act with Stash`将不会工作，因为它的运行时擦除类型仅仅是一个匿名的actor子类型）。目的是自动的使用合适的基于队列的并满足`stash`的邮箱类型。如果你想用到这个功能，简单的继承`ActWithStash`即可。

```scala
val a = actor(new ActWithStash {
  become {
    case 1 => stash() 
    case 2 => testActor ! 2; unstashAll(); becomeStacked {
        case 1 => testActor ! 1; unbecome() }
} })
```

