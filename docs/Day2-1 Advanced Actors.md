## Day 2: 高级Actor

### 使用`ask`

到目前为止，我们已经了解了`tell`,发送者"即发即弃"和`forward`。还有一种方法可以通过发送一条消息给actor并期望返回响应，即使在生产环境中而不是测试代码中，使用`ask`。让我们为`PaymentActorSpec`添加另一个测试：

首先，我们需要处理一些导入：

There is another way to send an actor a message and expect a response back, even in production and not test code, by using `ask`. Lets add another test using just this to `PaymentActorSpec`:

```scala
import akka.pattern._
import org.scalatest.concurrent.ScalaFutures
import scala.concurrent.duration._
```

`import akka.pattern._`导入了 ask `?`和`mapTo`，我们将在下面讨论。而且由于Ask使用Scala `Future`处理，上述`ScalaFutures`特质有助于测试future。该`scala.concurrent.duration._`导入提供了方便的方式来处理时间，比如下面的例子的`3.seconds`。

另外，我们必须将`ScalaFutures`特质混入到测试类中：

```scala
class PaymentActorSpec extends TestKit(ActorSystem("PaymentActorSpec"))
  with FlatSpecLike with Matchers with ImplicitSender with ScalaFutures {
```

然后测试本身：

```scala
  "The Payment Actor" should "approve small payment requests through ask automatically" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000))
      .mapTo[PaymentResponse]
    id += 1
    responseF.futureValue.status shouldBe Approved
  }
```

哇，这看起来很短。让我们使用一个接一个的ask来涵盖这个测试案例：

1. 我们隐式设置了超时。由于 ask `?` 返回一个future, 所以我们不能无限期地等待这个future。超时表明应该等待ask多长时间。`3.seconds`隐式转换为`Timeout`类型。
2. ask或`?`本身。由于我们的actor是无类型的, 因此它将不知道`PaymentActor`的响应类型是什么。所以它返回一个`Future[Any]`
3. `.mapTo[PaymentResponse]`将`Future`值转换为目标类型。
4. 我们在验证中使用的`futureValue`操作是ScalaTest的`ScalaFutures`特质提供的功能。它确实会阻塞。但是，通过在此处使用ScalaTest特性，我们可以确保将此类阻塞代码仅限制在测试中，而不会泄漏到生产代码中。生产代码无权访问ScalaTest。

`Future`可能成功也可能失败。要向一个`ask`发出失败信号，actor必须用一个`akka.actor.Status.Failure`回应。让我们在`matchAny`用例中尝试一下。修改`PaymentActor`的`matchAny`，用于在用例匹配时发回一个`Failure`：

再一次，这个导入：

```scala
import akka.actor.Status
```

然后在`PaymentActor`中修改`matchAny`为如下：

```scala
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
      sender() ! Status.Failure(new IllegalArgumentException("Message type not understood"))
      // ^^^ This line added ^^^
```

让我们在测试中尝试一下。让我们在`PaymentActorSpec`添加一个测试来检查这种失败：

```scala
  "The Payment Actor" should "fail the future with the right exception" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = paymentActor ? "Pay me!"
    responseF.failed.futureValue shouldBe an [IllegalArgumentException]
  }
```

该测试的唯一值得注意的部分是，我们使用的`.failed`投射一个`Future`来验证一个已经失败的future具有某一个异常。

### 改变actor的行为

现在我们来谈谈actor的最后一个属性。它可以根据消息更改行为。让我们尝试让actor有`offline`行为。如果我们发送一个`Offline`消息给`PaymentActor`，它只会发回一个错误。一旦我们发送一个`Online`消息给actor，它将开始恢复处理。在这里，我们将actor的状态更改为脱机，并使其在脱机模式下的行为有所不同。为此，我们首先需要`Offline`和`Online`信息。同时让它们成为单例以节省任何GC成本。它们只是信号，不包含状态。

```scala
case object Offline
case object Online
```

接下来，让我们向`PaymentActor`添加一个接收匹配器以构建离线行为。这是我们接收的final变量，不应更改：

```scala
  def offlineBehavior: Receive = {
    case Online =>
      context.unbecome()
      sender() ! Online
    case _: PaymentRequest =>
      log.error("Received PaymentRequest but still Offline")
      sender() ! Offline
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
      sender() ! Status.Failure(new IllegalArgumentException("Message type not understood"))
  }
```

为`Offline`添加接收匹配，用于该actor切换到`offlineBehavior`。这个匹配可以在`rcvBuilder.matchAny`之前进行：

```scala
    case Offline =>
      context.become(offlineBehavior, discardOld = false)
      sender() ! Offline
```

该`context.become(...)`调用切换actor到`offlineBehavior`。我们可以看到在`offlineBehavior`，`context.unbecome()`将行为设置回到原来的。如果将`context.become(...)`的`discardOld`参数设置为`true`，我们只能继续保持转换到新行为。这是为了防止由于行为更改而引起的内存泄漏。

让我们为此构建一个测试。不过，此测试要更长一些。它将在脱机之前，脱机期间以及返回联机之后检查行为。这是测试代码：

```scala
  "The Payment Actor" should "respond only with `Offline` while offline" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF.futureValue.status shouldBe Approved

    // Now take the actor offline
    val responseF2 = paymentActor ? Offline
    responseF2.futureValue shouldBe Offline

    // Lets try to send in 5 payment requests. Each should return Offline
    (0 until 5).foreach { _ =>
      val responseF3 = paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)
      id += 1
      responseF3.futureValue shouldBe Offline
    }

    // Next we turn the PaymentActor back on-line
    val responseF4 = paymentActor ? Online
    responseF4.futureValue shouldBe Online

    // Now we should be back in business
    val responseF5 = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF5.futureValue.status shouldBe Approved
  }
```

### 存储消息

在某些情况下，我们不只是想说我们处于离线状态。我们希望将收到的消息存储起来，直到我们重新联机并在那时进行处理。通常用在充当缓存的actor中。加载/重新加载缓存的时间很短，我们只想保留请求直到数据完全加载。

对于本练习，我们将替换一些已建立的逻辑。为了处理这种情况，我们想将`PaymentActor`复制到名为`StashingPaymentActor`的新类中。你需要将`PaymentActor`的所有引用复制和修改为`StashingPaymentActor`。

然后，我们将`Stash`特质混入到我们的`StashingPaymentActor`：

```scala
class StashingPaymentActor extends Actor with Stash with ActorLogging {
```

接下来，转到`offlineBehavior`并使其存储消息，而不只是发送脱机响应。同样，我们需要在重新联机时取消所有存储。

```scala
  def offlineBehavior: Receive = {
    case Online =>
      unstashAll()
      // ^^^ Here's your unstash change ^^^
      context.unbecome()
      sender() ! Online
    case _: PaymentRequest =>
      log.error("Received PaymentRequest but still Offline")
      stash()
      // ^^^ Here's your stash change ^^^
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
      sender() ! Status.Failure(new IllegalArgumentException("Message type not understood"))
  }
```

然后，当我们重新联机时，我们想处理所有存储的消息。

现在它应该一切正常。同样的，我们要将`PaymentActorSpec`拷贝到`StashingPaymentActorSpec`用于测试修改。

现在，我们将修改`testOffline()`方法以测试存储行为。这是我们在`StashingPaymentActorSpec`中的新`testOffline()`方法：

```scala
  "The Payment Actor" should "respond only with `Offline` while offline" in {
    implicit val timeout: Timeout = 3.seconds
    val responseF = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF.futureValue.status shouldBe Approved

    // Now take the actor offline
    val responseF2 = paymentActor ? Offline
    responseF2.futureValue shouldBe Offline

    // Lets try to send in 5 payment requests. Each should return Offline
    (0 until 5).foreach { _ =>
      paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)
      id += 1
      expectNoMessage(1.second)
    }

    // Next we turn the PaymentActor back on-line
    val responseF4 = paymentActor ? Online
    responseF4.futureValue shouldBe Online

    // Receive the stashed messages
    receiveN(5, 5.seconds) foreach {
      case PaymentResponse(_, _, status) =>
        status shouldBe Approved
      case _ => fail("Message received is not a PaymentResponse")
    }

    // Now we should be back in business
    val responseF5 = (paymentActor ? PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)).mapTo[PaymentResponse]
    id += 1
    responseF5.futureValue.status shouldBe Approved
  }
```

### Actor生命周期

actor一直活着，直到在内部调用`getContext().stop(self())`或在外部调用`getContext().stop(actorRef)`向其发送了一个`PoisonPill`。

当不再使用actor时，终止actor很重要。如果我们持续创建actor而不终止它们，则会发生内存泄漏。

![Image showing actor lifecycle](https://doc.akka.io/docs/akka/current/images/actor_lifecycle.png)

### 我们没有涵盖的内容

* 异步操作产生的管道
* FSM
* 监督策略

请您自己阅读这些内容。

现在我们已经完成了Actor。是啊！
