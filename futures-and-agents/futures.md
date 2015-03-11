# Futures

## 1 简介

在` Akka `中， 一个`Future`是用来获取某个并发操作的结果的数据结构。这个操作通常是由` Actor `执行或由` Dispatcher `直接执行的。 这个结果可以以同步（阻塞）或异步（非阻塞）的方式访问。

## 2 执行上下文

为了运行回调和操作，` Futures `需要有一个` ExecutionContext`，它与` java.util.concurrent.Executor `很相像。如果你在作用域内有一个` ActorSystem `，它会用它缺省的派发器作为` ExecutionContext`，你也可以用` ExecutionContext `伴生对象提供的工厂方法来将` Executors `和` ExecutorServices `进行包裹，或者甚至创建自己的实例。

```scala
import akka.dispatch.{ ExecutionContext, Promise }

implicit val ec = ExecutionContext.fromExecutorService(yourExecutorServiceGoesHere)

// 用你崭新的 ExecutionContext 干点啥
val f = Promise.successful("foo")

// 然后在程序/应用的正确的位置关闭 ExecutionContext
ec.shutdown()
```

### 在actor中

每个actor都被配置为在`MessageDispatcher`上运行，这个派发器被当作一个`ExecutionContext`。如果Future调用的性质与actor相匹配或者与actor的活动相兼容，那么可以很容易的通过引人`context.dispatcher`在运行Futures时重用这个派发器。

```scala
class A extends Actor {
  import context.dispatcher
  val f = Future("hello")
  def receive = {
    // receive omitted ...
} }
```

## 3 用于 Actor

通常有两种方法来从一个` Actor `获取回应： 第一种是发送一个消息 (`actor ! msg`), 这种方法只在发送者是一个` Actor` 时有效，第二种是通过一个` Future`。

使用` Actor `的方法来发送消息会返回一个` Future`。

```scala
import scala.concurrent.Await
import akka.pattern.ask
import akka.util.Timeout
import scala.concurrent.duration._
implicit val timeout = Timeout(5 seconds)
val future = actor ? msg // enabled by the “ask” import
val result = Await.result(future, timeout.duration).asInstanceOf[String]
```

这会导致当前线程被阻塞，并等待` Actor `通过它的应答来 ‘完成’` Future `。但是阻塞会导致性能问题，所以是不推荐的。导致阻塞的操作位于` Await.result `和` Await.ready `中，这样就方便定位阻塞的位置。 对阻塞方式的替代方法会在本文档中进一步讨论。还要注意` Actor `返回的` Future `的类型是` Future[Any] `，这是因为` Actor `是动态的。这也是为什么上例中使用了 asInstanceOf 。 在使用非阻塞方式时，最好使用` mapTo `方法来将` Future `转换到期望的类型:

```scala
import scala.concurrent.Future
import akka.pattern.ask
val future: Future[String] = ask(actor, msg).mapTo[String]
```

如果转换成功，` mapTo `方法会返回一个包含结果的新的` Future`, 如果不成功，则返回` ClassCastException` 。对异常的处理将在本文档进一步讨论。

发送一个`Future`的结果到一个Actor，你可以用`pipe`构造：

```scala
import akka.pattern.pipe
future pipeTo actor
```

## 4 直接使用

Akka中的一个常见用例是在不需要使用Actor的情况下并发地执行计算。 如果你发现你只是为了并行地执行一个计算而创建了一堆 Actor，这并不是一个好方法，下面是一种更好（也更快）的方法：

```scala
import scala.concurrent.Await
import scala.concurrent.Future
import scala.concurrent.duration._
val future = Future {
  "Hello" + "World"
}
future foreach println
```

在上面的代码中，被传递给` Future `的代码块会被缺省的` Dispatcher `所执行, 代码块的返回结果会被用来完成` Future `(在这种情况下，结果是一个字符串: “HelloWorld”)。 与从Actor返回的` Future `不同，这个` Future `拥有正确的类型, 我们还避免了管理` Actor `的开销。

你还可以用 Promise 伴生对象创建一个已经完成的 Future, 它可以是成功:

```scala
val future = Future.successful("Yay!")
```
或失败的：

```scala
val otherFuture = Future.failed[String](new IllegalArgumentException("Bang!"))
```

也有可能创建一个空的`Promise`，稍后再填充，然后获得相应的`Future`。

```scala
val promise = Promise[String]()
val theFuture = promise.future
promise.success("hello")
```

## 5 函数式 Future

Akka 的` Future `有一些与Scala集合所使用的非常相似的一元的（monadic）方法。 这使你可以构造出结果可以传递的 ‘管道’ 或 ‘数据流’。

### Future是一元的

让Future以函数式风格工作的第一个方法是 map。 它需要一个函数来对Future的结果进行处理, 返回一个新的结果。 map方法的返回值是包含新结果的另一个Future:

```scala
val f1 = Future {
  "Hello" + "World"
}
val f2 = f1 map { x =>
x.length }
f2 foreach println
```

这个例子中我们在` Future `内部连接两个字符串。我们没有等待这个Future结束，而是使用` map `方法来将计算字符串长度的函数作用于它。 现在我们有了新的` Future `，它的最终结果是一个 Int。 当先前的` Future `完成时, 它会应用我们的函数并用其结果来完成第二个` Future `。最终我们得到的结果是 10。先前的` Future `仍然持有字符串“HelloWorld” 不受 map的影响。

如果我们只是修改一个 Future，map 方法就够用了。 但如果有2个以上 Future时 map 无法将他们组合到一起:

```scala
val f1 = Future {
  "Hello" + "World"
}
val f2 = Future.successful(3)
val f3 = f1 map { x =>
    f2 map { y =>x.length * y }
}
f3 foreach println
```

`f3` 的类型是` Future[Future[Int]] `而不是我们所期望的` Future[Int]`。这时我们需要使用` flatMap `方法：

```scala
val f1 = Future {
  "Hello" + "World"
}
val f2 = Future.successful(3)
val f3 = f1 flatMap { x =>
    f2 map { y =>x.length * y }
}
f3 foreach println
```

使用嵌套的`map`或`flatmap`来组合`Future`有时会变得非常复杂和难以阅读，这时使用Scala的`for comprehensions` 一般会生成可读性更好的代码。 见下一节的示例。

如果你需要进行条件筛选，可以使用` filter`:

```scala
val future1 = Future.successful(4)
val future2 = future1.filter(_ % 2 == 0)
future2 foreach println
val failedFilter = future1.filter(_ % 2 == 1).recover {
  // When filter fails, it will have a java.util.NoSuchElementException
  case m: NoSuchElementException => 0
}
failedFilter foreach println
```

### For Comprehensions

由于` Future `拥有` map`,` filter` 和 `flatMap `方法，它可以方便地用于 ‘for comprehension’:

```scala
val f = for {
    a <- Future(10 / 2) // 10 / 2 = 5 
    b<-Future(a+1)// 5+1=6 
    c<-Future(a-1)// 5-1=4 
    if c > 3 // Future.filter
}yieldb*c// 6*4=24

// Note that the execution of futures a, b, and c
// are not done in parallel.
f foreach println
```

做这些事情的时候需要记住的是：虽然看上去上例的部分代码可以并发地运行，`for comprehension`的每一步是顺序执行的。每一步是在单独的线程中运行的，但是相较于将所有的计算在一个单独的 Future中运行并没有太大好处. 只有在先创建好` Future`，然后对其进行组合的情况下才能得到真正的好处。

### 组合 Futures

上例中的` comprehension `是对` Future`进行组合的例子. 这种方法的常见用例是将多个` Actor`的回应组合成一个单独的计算而不用调用` Await.result `或` Await.ready `来阻塞地获得每一个结果。 先看看使用` Await.result `的例子:

```scala
val f1 = ask(actor1, msg1)
val f2 = ask(actor2, msg2)
val a = Await.result(f1, 3 seconds).asInstanceOf[Int]
val b = Await.result(f2, 3 seconds).asInstanceOf[Int]
val f3 = ask(actor3, (a + b))
val result = Await.result(f3, 3 seconds).asInstanceOf[Int]
```

这里我们等待前2个` Actor `的结果然后将其发送给第三个 Actor。 我们调用了3次` Await.result `, 导致我们的程序在获得最终结果前阻塞了3次。 现在跟下例比较:

```scala
val f1 = ask(actor1, msg1)
val f2 = ask(actor2, msg2)
val f3 = for {
  a <- f1.mapTo[Int]
  b <- f2.mapTo[Int]
  c <- ask(actor3, (a + b)).mapTo[Int]
} yield c
f3 foreach println
```

这里我们有两个 2 actor各自处理自己的消息。 一旦这2个结果可用了 (注意我们并没有阻塞地等待这些结果)， 它们会被加起来发送给第三个 Actor， 这第三个actor回应一个字符串，我们把它赋值给 `result`。

当我们知道Actor数量的时候上面的方法就足够了，但是当Actor数量较大时就显得比较笨重。` sequence `和` traverse `两个辅助方法可以帮助处理更复杂的情况。 这两个方法都是用来将` T[Future[A]] `转换为` Future[T[A]]`（`T `是 `Traversable`子类）。 例如:

```scala
// oddActor 以类型 List[Future[Int]] 返回从1开始的奇数序列
val listOfFutures = List.fill(100)(akka.pattern.ask(oddActor, GetNext).mapTo[Int])
// now we have a Future[List[Int]]
val futureList = Future.sequence(listOfFutures)
// 计算奇数的和
val oddSum = futureList.map(_.sum)
oddSum foreach println
```

现在来解释一下，` Future.sequence `将输入的` List[Future[Int]] `转换为` Future[List[Int]]`。这样我们就可以将` map `直接作用于` List[Int]`， 从而得到List的总和。

`traverse `方法与` sequence `类似, 但它以` T[A] `和 `A => Future[B] `函数为参数返回一个` Future[T[B]]`，这里的 T 同样也是` Traversable `的子类。 例如, 用` traverse `来计算前100个奇数的和:

```scala
val futureList = Future.traverse((1 to 100).toList)(x => Future(x * 2 - 1))
val oddSum = futureList.map(_.sum)
oddSum foreach println
```

结果与这个例子是一样的:

```scala
val futureList = Future.sequence((1 to 100).toList.map(x => Future(x * 2 - 1)))
val oddSum = futureList.map(_.sum)
oddSum foreach println
```

但是用` traverse `会快一些，因为它不用创建一个` List[Future[Int]] `临时量。

最后我们有一个方法` fold`，它的参数包括一个初始值 , 一个Future序列和一个作用于初始值、Future类型以及返回与初始值相同类型的函数， 它将这个函数异步地应用于future序列的所有元素，它的执行将在最后一个Future完成之后开始。

```scala
// Create a sequence of Futures
val futures = for (i <- 1 to 1000) yield Future(i * 2)
val futureSum = Future.fold(futures)(0)(_ + _)
futureSum foreach println
```

就是这么简单！

如果传给 fold 的序列是空的, 它将返回初始值, 在上例中，这个值是0。 有时你并不需要一个初始值，而使用序列中第一个已完成的Future的值作为初始值，你可以使用` reduce`， 它的用法是这样的:

```scala
// Create a sequence of Futures
val futures = for (i <- 1 to 1000) yield Future(i * 2)
val futureSum = Future.reduce(futures)(_ + _)
futureSum foreach println
```

与 fold 一样, 它的执行是在最后一个 Future 完成后异步执行的， 你也可以对这个过程进行并行化：将future分成子序列分别进行reduce，然后对reduce的结果再次reduce。


## 6 回调

有时你只需要监听` Future `的完成事件, 对其进行响应，不是创建新的Future，而仅仅是产生副作用。 Akka 为这种情况准备了` onComplete`,` onSuccess `和` onFailure`, 而后两者仅仅是第一项的特例。

```scala
future onSuccess {
  case "bar"     => println("Got my bar alright!")
  case x: String => println("Got some random string: " + x)
}
```

```scala
future onFailure {
  case ise: IllegalStateException if ise.getMessage == "OHNOES" =>
  //OHNOES! We are in deep trouble, do something!
  case e: Exception =>
  //Do something else
}
```

```scala
future onComplete {
  case Success(result)  => doSomethingOnSuccess(result)
  case Failure(failure) => doSomethingOnFailure(failure)
}
```

## 7 定义次序

由于回调的执行是无序的，而且可能是并发执行的, 当你需要一组有序操作的时候需要一些技巧。但有一个解决办法是使用` andThen`。 它会为指定的回调创建一个新的Future, 这个Future与原先的Future拥有相同的结果, 这样就可以象下例一样定义次序：

```scala
val result = Future { loadPage(url) } andThen {
  case Failure(exception) => log(exception)
} andThen {
  case _ => watchSomeTV()
}
result foreach println
```

## 8 辅助方法

`Future fallbackTo` 将两个Futures合并成一个新的Future, 如果第一个Future失败了，它将持有第二个 Future 的成功值。

```scala
val future4 = future1 fallbackTo future2 fallbackTo future3
future4 foreach println
```

你也可以使用`zip`操作将两个Futures组合成一个新的持有二者成功结果的tuple的Future。

```scala
val future3 = future1 zip future2 map { case (a, b) => a + " " + b }
future3 foreach println
```

## 9 异常

由于Future的结果是与程序的其它部分并发生成的，因此异常需要作特殊的处理。 不管是Actor还是派发器正在完成此Future, 如果抛出了 Exception ，Future 将持有这个异常而不是一个有效的值。 如果 Future 持有 Exception, 调用 Await.result 将导致此异常被再次抛出从而得到正确的处理。

通过返回一个其它的结果来处理 Exception 也是可能的。 使用` recover `方法。 例如:

```scala
val future = akka.pattern.ask(actor, msg1) recover {
  case e: ArithmeticException => 0
}
future foreach println
```

在这个例子中，如果actor回应了包含` ArithmeticException`的`akka.actor.Status.Failure` , 我们的 Future 将持有 0 作为结果。 `recover `方法与标准的` try/catch `块非常相似, 可以用这种方式处理多种Exception， 如果其中有没有提到的Exception，这种异常将以没有定义` recover `方法的方式来处理。

你也可以使用` recoverWith `方法, 它和` recover `的关系就象` flatMap `与` map`的关系, 用法如下:

```scala
val future = akka.pattern.ask(actor, msg1) recoverWith {
  case e: ArithmeticException => Future.successful(0)
  case foo: IllegalArgumentException =>
    Future.failed[Int](new IllegalStateException("All br0ken!"))
}
future foreach println
```

## 10 After

`akka.pattern.after`使超时之后完成一个带有值或者异常的Future非常容易。

```scala
// TODO after is unfortunately shadowed by ScalaTest, fix as part of #3759
// import akka.pattern.after
val delayed = akka.pattern.after(200 millis, using = system.scheduler)(Future.failed(
  new IllegalStateException("OHNOES")))
val future = Future { Thread.sleep(1000); "foo" }
val result = Future firstCompletedOf Seq(future, delayed)
```


