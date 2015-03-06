# 测试actor系统

## 1 Testkit实例

这是Ray Roestenburg 在[他的博客](http://roestenburg.agilesquad.com/2011/02/unit-testing-akka-actors-with-testkit_12.html)中的示例代码， 作了改动以兼容 Akka 2.x。

```scala
import scala.util.Random
import org.scalatest.BeforeAndAfterAll
import org.scalatest.WordSpecLike
import org.scalatest.Matchers
import com.typesafe.config.ConfigFactory
import akka.actor.Actor
import akka.actor.ActorRef
import akka.actor.ActorSystem
import akka.actor.Props
import akka.testkit.{ TestActors, DefaultTimeout, ImplicitSender, TestKit }
import scala.concurrent.duration._
import scala.collection.immutable

//演示TestKit的示例测试
class TestKitUsageSpec
  extends TestKit(ActorSystem("TestKitUsageSpec",
    ConfigFactory.parseString(TestKitUsageSpec.config)))
  with DefaultTimeout with ImplicitSender
  with WordSpecLike with Matchers with BeforeAndAfterAll {
  import TestKitUsageSpec._
  val echoRef = system.actorOf(TestActors.echoActorProps)
  val forwardRef = system.actorOf(Props(classOf[ForwardingActor], testActor))
  val filterRef = system.actorOf(Props(classOf[FilteringActor], testActor))
  val randomHead = Random.nextInt(6)
  val randomTail = Random.nextInt(10)
  val headList = immutable.Seq().padTo(randomHead, "0")
  val tailList = immutable.Seq().padTo(randomTail, "1")
  val seqRef =system.actorOf(Props(classOf[SequencingActor], testActor, headList, tailList))
  override def afterAll {
    shutdown()
  }
  
  "An EchoActor" should {
      "Respond with the same message it receives" in {
        within(500 millis) {
          echoRef ! "test"
          expectMsg("test")
        }
      }
  }
    
  "A ForwardingActor" should {
      "Forward a message it receives" in {
        within(500 millis) {
          forwardRef ! "test"
          expectMsg("test")
        }
      } 
  }
  
  "A FilteringActor" should {
      "Filter all messages, except expected messagetypes it receives" in {
        var messages = Seq[String]()
        within(500 millis) {
          filterRef ! "test"
          expectMsg("test")
          filterRef ! 1
          expectNoMsg
          filterRef ! "some"
          filterRef ! "more"
          filterRef ! 1
          filterRef ! "text"
          filterRef ! 1
          receiveWhile(500 millis) {
            case msg: String => messages = msg +: messages
  } }
        messages.length should be(3)
        messages.reverse should be(Seq("some", "more", "text"))
      }
  }
  
  "A SequencingActor" should {
      "receive an interesting message at some point " in {
        within(500 millis) {
          ignoreMsg {
            case msg: String => msg != "something"
          }
          seqRef ! "something"
          expectMsg("something")
          ignoreMsg {
            case msg: String => msg == "1"
          }
          expectNoMsg
          ignoreNoMsg
        }
      }
  }
}

object TestKitUsageSpec {
  // Define your test specific configuration here
  val config = """
    akka {
      loglevel = "WARNING"
  } """
  
  //将所有消息转发给另一个Actor的actor
  class ForwardingActor(next: ActorRef) extends Actor {
    def receive = {
      case msg => next ! msg
    }
  }
  //仅转发一部分消息给另一个Actor的Actor
  class FilteringActor(next: ActorRef) extends Actor {
      def receive = {
        case msg: String => next ! msg
        case _           => None
      }
  }
  //此actor发送一个消息序列，此序列具有随机列表作为头，一个关心的值及一个随机列表作为尾
  //想法是你想测试接收到的关心的值，而不用处理其它部分
  
 class SequencingActor(next: ActorRef, head: immutable.Seq[String],
                         tail: immutable.Seq[String]) extends Actor {
     def receive = {
       case msg => {
         head foreach { next ! _ }
         next ! msg
         tail foreach { next ! _ }
       }
     }
 } 
}
```