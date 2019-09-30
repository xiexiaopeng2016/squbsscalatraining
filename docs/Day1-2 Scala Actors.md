# Akka Scala/squbs培训

## Day1: 您的第一个Actor

### 创建状态枚举和消息类

1. 创建新的Scala类`PaymentActor`

2. 在与`PaymentActor`相同的文件中, 为`PaymentStatus`创建案例对象

   ```scala
   sealed trait PaymentStatus
   case object Accepted extends PaymentStatus
   case object Approved extends PaymentStatus
   case object Rejected extends PaymentStatus
   ```

3. 创建样例类`RecordId`，`PaymentRequest`和`PaymentResponse`。请注意，金额是`Long`类型的，包括两个小数点。 除以100D得到实际的美元金额。
   
   ```scala
   case class RecordId(id: Long, creator: Long, creationTime: Long = System.currentTimeMillis)   
   case class PaymentRequest(id: RecordId, payerAcct: Long, payeeAcct: Long, amount: Long)
   case class PaymentResponse(id: RecordId, requestId: RecordId, status: PaymentStatus)
   ```

### 创建基本Actor

稍后我们将展开这actor。因此，让我们从一个非常基本的东西开始：

```scala
class PaymentActor extends Actor with ActorLogging {

  val creatorId = 100L
  var id = 1L

  override def receive: Receive = {
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      log.info("Received payment request of ${} accounts {} => {}",
        amount / 100D, payerAcct, payeeAcct)
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
  }
}
```

到目前为止，我们的actor做得并不多。它会收到一条付款消息，并输出日志。虽然没什么用处，但至少我们有一个运行的actor。

### 注册你的Actor

这样squbs会自动启动actor。将actor添加到cube项目的`src/main/resources/META-INF/squbs-meta.conf`文件。

```
cube-name = com.paypal.myorg.myservcube
cube-version = "0.0.1-SNAPSHOT"
squbs-actors = [
  {
    class-name = com.paypal.myorg.myserv.cube.PaymentActor
    name = paymentActor
  }
]
```

### 测试你的Actor

1. 在`src/test`下面创建`scala`文件夹。
2. 在`src/test/scala`下面，创建与包含您的actor的包同名的包。
3. 创建测试文件`PaymentActorSpec`，简单地调用actor，如下所示:

   ```scala
   import akka.actor.{ActorSystem, Props}
   import akka.testkit.{ImplicitSender, TestKit}
   import org.scalatest.{FlatSpecLike, Matchers}

   class PaymentActorSpec extends TestKit(ActorSystem("PaymentActorSpec"))
     with FlatSpecLike with Matchers with ImplicitSender {

     val creator = 200L
     var id = 1L

     private val paymentActor = system.actorOf(Props[PaymentActor])

     "The Payment Actor" should "react to payment requests" in {
       paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 30000)
       id += 1

       // Temporarily have the sleep here so the test does not terminate prematurely.
       Thread.sleep(1000)
     }
   }
   ```
	 
	 这个`PaymentActorSpec`混入扩展了Akka TestKit，并增加了以下一些特质：
   * `FlatSpecLike` 使用`FlatSpec`样式定义测试的样式
   * `Matchers` 启用S​​calaTest的匹配器DSL，我们将在后面使用 
   * `ImplicitSender` 在测试内部提供一个发送者，用于测试actor响应发送者	
	 
4. 在编辑器中右键单击测试类名称`PaymentActorSpec`，然后从弹出菜单中选择选项`Run 'PaymentActorSpec'`。查看测试结果。由于我们的actor没有执行验证仅进行日志输出，因此我们可以通过查看日志来观察正在运行。我们将在下一步中添加断言。
5. 如果启用了工具栏，则会在可以运行的项目列表中看到`PaymentActorSpec`。选择测试，然后按`>`图标在以后的时间开始测试。由于我们的actor除了输出日志外什么都不做，因此我们可以通过查看日志来观察它的运行情况。

### 让Actor发回PaymentResponse

1. 编辑`PaymentActor`并添加以下行:

   ```scala
       case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
         log.info("Received payment request of ${} accounts {} => {}",
           amount / 100D, payerAcct, payeeAcct)
         sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Accepted)
         id += 1
         // ^^^ Add these two line here ^^^
   ```

2. 由于`PaymentActor`现在响应了一个消息，因此请更新测试以获取响应消息并验证其状态。
   
   ```scala
     "The Payment Actor" should "react to payment requests" in {
       paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 30000)
       id += 1
       val response = expectMsgType[PaymentResponse]
       response.status shouldBe Accepted
       // ^^^ Add the 2 lines above. ^^^

       // Temporarily have the sleep here so the test does not terminate prematurely.
       // Thread.sleep(1000)
       // ^^^ Remove the sleep, commented lines here ^^^
     }
   ```

### 限定消息

有时我们想区别对待达到的不同消息。例如，在自助餐厅支付$12美元或以下的金额应自动获得批准。因此，让我们添加另一个样例模式匹配。

**注意:** 这种情况需要添加在当前情况之前，因为模式匹配是按顺序完成的。这种情况比现有情况更具体。

```scala
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) if amount < 1200 =>
      log.info("Received small payment request of ${} accounts {} => {}", amount / 100D, payerAcct, payeeAcct)
      sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Approved)
      id += 1
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      ...
```

此匹配器添加了一个限定符测试，该测试值小于1200，即$12.00，它将立即以`Approved`状态响应。

让我们还添加一个测试以查看实际效果。在测试中，添加另一个测试。

```scala
  "The Payment Actor" should "approve small payment requests automatically" in {
    paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 1000)
    id += 1
    val response = expectMsgType[PaymentResponse]
    response.status shouldBe Approved
  }
```

现在再次运行测试。它应该通过了。另请注意日志消息。

### 创建子Actor

请记住，基于Actor计算模型，Actor在收到消息后可以执行以下三个操作中的一项或多项：

* 发送信息
* 创建其它actor
* 更改下一条消息要使用的状态/行为

现在，让我们探索第二个属性：创建另一个子actor。请注意，此actor成为该子actor的父级。

现在，我们创建另一个actor`RiskAssessmentActor`。在这种情况下，为简单起见，我们让它批准每个请求。这是代码：

```java
class RiskAssessmentActor extends Actor with ActorLogging {

  val creatorId = 300L
  var id = 1L

  override def receive: Receive = {
    case PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      log.info("Received payment assessment request of ${} accounts {} => {}",
        amount / 100D, payerAcct, payeeAcct);
      sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Approved)
      id += 1
    case o =>
      log.error("Message of type {} received is not understood", o.getClass.getName)
  }
}
```

就其本身而言，`RiskAssessmentActor`没有什么有趣的。它在很多方面都类似于`PaymentActor`。但是现在，我们让`PaymentActor`将批准请求转发给`RiskAssessmentActor`，然后将其返回给客户端，以下是测试代码。

为此，让我们增强`PaymentActor`。我们将修改当前的高风险支付代码，以进行转发，如下：

```java
    case request @ PaymentRequest(requestId, payerAcct, payeeAcct, amount) =>
      // Change pattern matching to also provide the request itself by prepending request @
      log.info("Received payment request of ${} accounts {} => {}",
        amount / 100D, payerAcct, payeeAcct)
      sender() ! PaymentResponse(RecordId(id, creatorId), requestId, Accepted)
      id += 1
      val riskActor = context.actorOf(Props[RiskAssessmentActor])
      riskActor forward request
      // ^^^ Add these two lines here ^^^
```

注意现在客户端应该收到两条消息。一种带有`Accepted`状态，另一种带有`Approved`状态。因此，让我们修改我们的第一个测试。

```scala
  "The Payment Actor" should "react to payment requests" in {
    paymentActor ! PaymentRequest(RecordId(id, creator), 1001, 2001, 30000)
    id += 1
    val response = expectMsgType[PaymentResponse]
    response.status shouldBe Accepted
    val response2 = expectMsgType[PaymentResponse]
    response2.status shouldBe Approved
    // ^^^ Add the 2 lines above. ^^^
  }
```