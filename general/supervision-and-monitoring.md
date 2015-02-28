# 监管和重启

## 监管的意思

在[ Actor系统](actor-systems.md)中说过，监管描述的是actor之间的依赖关系：监管者将任务委托给下属并对下属的失败状况进行响应。当一个下属出现了失败（即抛出一个异常），它自己会将自己和自己所有的下属挂起然后向自己的监管者发送一个提示失败的消息。依赖所监管的工作的性质和失败的性质，监管者可以有4种基本选择：

- 让下属继续执行，保持下属当前的内部状态
- 重启下属，清除下属的内部状态
- 永久地终止下属
- 将失败沿监管树向上传递

重要的是始终要把一个actor视为整个监管树形体系中的一部分，这解释了第4种选择存在的意义（因为一个监管者同时也是其上方监管者的下属），并且隐含在前3种选择中：让actor继续执行同时也会继续执行它的下属，重启一个actor也必须重启它的下属，相似地终止一个actor会终止它所有的下属。
需要强调的是一个actor的缺省行为是在重启前终止它的所有下属，这种行为可以用 Actor 类的preRestart hook来重写；对所有子actor的递归重启操作在这个hook之后执行。

每个监管者都配置了一个函数，它将所有可能的失败原因归结为以上四种选择之一；注意，这个函数并不将失败actor本身作为输入。我们很快会发现在有些结构中这种方式看起来不够灵活，会希望对不同的下属采取不同的策略。在这一点上我们一定要理解监管是为了组建一个递归的失败处理结构。如果你试图在某一个层次做太多事情，这个层次会变得复杂难以理解，这时我们推荐的方法是增加一个监管层次。

Akka实现的是一种叫“父监管”的形式。Actor只能由其它的actor创建，而顶部的actor是由库来提供的——每一个创建出来的actor都是由它的父亲所监管。这种限制使得actor的树形层次拥有明确的形式，并提倡合理的设计方法。必须强调的是这也同时保证了actor们不会成为孤儿或者拥有在系统外界的监管者（被外界意外捕获）。这样就产生了一种对actor应用(或其中子树)自然又干净的关闭过程。

## Top-Level监管者

![guardians](../imgs/guardians.png)

### `/user`: 监护人（guardian）Actor

actor可能和它们的父母actor相互作用得最多，它的监护人命名为 `/user`。利用`system.actorOf()`创建的actors是这个actor的孩子。这意味着，当这个监护人终止，系统中所有正常的actors也关闭了。这也意味这，这个监护人的监管测量决定了top-level actor如何被监控。从
2.1开始，可以通过`akka.actor.guardian-supervisor-strategy`来配置策略。当监护人升级为一个失败，root监护人的响应将会终止它的监护人，这将导致整个actor系统关闭。

### `/system`: 系统监护人

这个特定监护人的引人是为了获取一个有序的关闭序列（所有正常的actor终止，但是logging保持可用，即使logging本身也是用actor实现的）。通过系统监护人观察用户监护人并且发起它自己的shut-down（通过收到期望的Terminated消息）。top-level系统actors通过一个策略来监控，
这个策略将会根据所有类型的Exception（除了ActorInitializationException和ActorKilledException）无限的重启actors。所有的其它异常抛出将会升级，最终关闭整个actor系统。

### `/`: Root监护人

The root guardian is the grand-parent of all so-called “top-level” actors and supervises all the special actors mentioned in Top-Level Scopes for Actor Paths using the SupervisorStrategy.stoppingStrategy, whose purpose is to terminate the child upon any type of Exception. All other throwables will be escalated . . . but to whom? Since every real actor has a supervisor, the supervisor of the root guardian cannot be a real actor. 
And because this means that it is “outside of the bubble”, it is called the “bubble-walker”. This is a synthetic ActorRef which in effect stops its child upon the first sign of trouble and sets the actor system’s isTerminated status to true as soon as the root guardian is fully terminated (all children recursively stopped).

## 重启的意思

当actor在处理消息时出现失败，失败的原因分成以上三类:

- 对收到的特定消息的系统错误（程序错误）
- 处理消息时一些外部资源的（临时性）失败
- actor内部状态崩溃了

以下是重启过程中发生的事件的精确顺序：

- actor被挂起，并且递归地挂起所有的子actor
- 调用旧实例的preRestart方法 (默认情况是发生终止请求到所有的子actors，并且调用postStop)
- 在preRestart过程中，等待所有被请求终止的子actors终止，最后一个子actor终止后，到下一步
- 再次调用之前提供的actor工厂创建新的actor实例
- 对新实例调用 postRestart
- 给所有在第三步没有终止的子actor发送重启请求。重启子actor将会遵循第二步的相同的递归处理
- 恢复运行新的actor





