## Day 3: Akka Streams简介

### 核心概念

* Source - 来源流组件
* Flow - 处理组件
* Sink - 接收流组件
* Materializer - 使流滴答作响的引擎(Engine that makes the stream tick)
* Materialized值 - 流的结果值
* 背压

### 编写您的第一个流

我们从编写第一个非常简单的流开始。同样，稍后我们将在第一个应用程序上进行扩展，以构建更复杂，更有用的流。您不一定必须在`Actor`中建立流，但我们也将从这开始。

在你的cube项目的`src/main/scala`路径下的所选包中创建Scala类`PaymentStreamActor`，其内容如下：

```scala
import akka.actor.{Actor, ActorLogging}
import akka.stream.ActorMaterializer
import akka.stream.scaladsl.Source

class PaymentStreamActor extends Actor with ActorLogging {

  implicit val mat = ActorMaterializer()

  val src = Source(1 to 100)

  override def receive: Receive = {
    case _ => run()
  }

  private def run() = src.runForeach(log.info("Count: {}", _))
}
```

为了使它滴答(tick)，我们还希望编写测试代码。在你的cube项目的`src/test/scala`路径下所选包内创建`PaymentStreamSpec`, 并添加以下内容：

```scala
import akka.actor.{ActorSystem, Props}
import akka.testkit.{ImplicitSender, TestKit}
import org.scalatest.{FlatSpecLike, Matchers}
import org.scalatest.concurrent.ScalaFutures

class PaymentStreamSpec extends TestKit(ActorSystem("PaymentStreamSpec"))
  with FlatSpecLike with Matchers with ImplicitSender with ScalaFutures {

  val streamActor = system.actorOf(Props[PaymentStreamActor])

  "The stream actor" should "Accept a ping request" in {
    streamActor ! "ping"

    // Just temporarily, we just want to see the results
    Thread.sleep(1000)
  }
}
```

在编辑器中右击测试类名称，然后从弹出菜单中选择`Run 'PaymentStreamSpec'`选项。查看测试结果。由于我们的actor没有执行任何验证, 仅输出了日志，因此我们可以通过查看日志来观察到其正在运行。您应该会看到它输出了1到100的数字。你得到了你的第一个流!

### 添加Sink

到目前为止，我们使用了一种快捷方式方法`runForeach`来安装一个sink，然后一次性运行它。Akka Steams中有很多这样的快捷方法，以确保您不必编写过多的代码。但现在让我们把一切变得非常清楚。在声明源的下面添加一个sink：

```scala
  val src = Source(1 to 100)
  val sink = Sink.foreach[Int](log.info("Count: {}", _))
  // Add this line ^^^^^^^^^^
```

接下来，为了便于理解，我们还想将run变得更正式。让我们将`run()`方法重写如下：

```scala
  private def run() = {
    val stream = src.to(sink)
    stream.run()
  }
```

在这里，我们现在可以看到，我们基本上创建了一个可运行的流图`RunnableGraph`，在这里我们忽略物化(materialized)值。

### 添加流组件

现在我们有了源和接收组件，让我们在流的中间添加一些处理。首先，我们仅使用过滤器获取偶数。让我们创建一个过滤流程：

```scala
  val src = Source(1 to 100)
  val filter = Flow[Int].filter(_ % 2 == 0)
  // Add this line ^^^^^^^^
  val sink = Sink.foreach[Int](log.info("Count: {}", _))
```

然后，让我们的过滤器开始工作。修改我们的`run()`方法，如下所示：

```scala
    val stream = src.via(filter).to(sink)
    stream.run()
```

并再次运行。现在，您会发现一半的数字不再打印。

我们还可以添加另一个流处理组件，用于计算每个数字的平方。让我们这样做。

```scala
  val filter = Flow[Int].filter(_ % 2 == 0)
  val square = Flow[Int].map(i => i * i)
  // Add this line ^^^^^^^^^
  val sink = Sink.foreach[Int](log.info("Count: {}", _))
```

语言说明：我们一直在`filter`中使用快捷方式`_`，意为"it"，但在`map`中使用了完整形式。在这种情况下，快捷方式不能很好地为我们服务。虽然我们可以看到Scala语言中的某些情况使用`(_ * _)`，但含义有所不同。第一个`_`表示闭包的第一个输入，第二个`_`表示第二个输入。在快捷方式中，每个输入只能使用一次。由于我们不遵循此规则，因此需要使用完整的闭包格式`(i => i * i)`

然后，我们还将`square`添加到流中。

```scala
  val stream = src.via(filter).via(square).to(sink)
```

并运行它。嗯，结果并没有什么出人意料的。

在这里，我们只是创建了两个单独的流处理组件，然后将它们组合在一起。我们也可以将所有这些都定义为单个组件，例如：

```scala
  val squareEven = Flow[Int].filter(_ % 2 == 0).map(i => i * i)
```

这个绝对是可行的。但是过滤器和平方组件不再容易重复使用。不过，以这种方式将一些复杂的逻辑组合在一起可能是有价值的。

但是，我们又学到了一件事。流并不总是进一个就会出一个。相反，元素的数量可能会缩小(带有`filter`)，或者可能会在处理阶段之间增加(例如，带有`mapConcat`或`flatMapConcat`阶段)。

### 流组件是可重复使用的

在后面的示例中，您可以清楚地看到我们分开了流的声明和流的运行`stream.run()`。

要注意的是，每个流组件本身都是可重用的。它们可以以任何形式传输，从方法调用返回，通过actor消息发送等。然后，这些逻辑组件可以组成最终的可运行形式，并可以在任何地方运行。甚至最终`RunnableGraph`也可以重复使用，并且可以被传输。我们可以将它们视为流逻辑的模板，它们将被传输和操纵，直到最终通过`run`命令运行。我们一开始的`src.runForeach`调用只是所有这一切的捷径。

### Stream组合&组件化

流组件可组成更复杂的组件。它们中的每一个都可以被传送并构建成一个最终的`RunnableGraph`。不过，了解这种组合的结果类型很重要。

* `Source ~> Flow` ==> `Source`
* `Flow ~> Flow` ==> `Flow`
* `Flow ~> Sink` ==> `Sink`

让我们在这里尝试一些声明。注意：我们显示类型注释以清楚显示组合的结果类型。在您的代码中，这可以由Scala编译器推断出来，可能不需要：

```scala
  val filteredSource: Source[Int, NotUsed] = src.via(filter)
  val filterAndSquare: Flow[Int, Int, NotUsed] = filter.via(square)

  // Use materialized value of flow
  val squareAndLog: Sink[Int, NotUsed] = square.to(sink)

  // Choose materialized value of sink
  val squareAndLogMat: Sink[Int, Future[Done]] = square.toMat(sink)(Keep.right)
```

现在，它们中的每一个都变成了更复杂的流组件，也可以被传送和重用。这对于声明和组合越来越复杂的组件非常方便，它们将成为主流程的一部分。

语言说明：您会注意到`square.toMat`的两组参数。第一个是接收器，与`Keep.right`用不同的括号分隔。这称为"柯里化"，通常用于功能语言中。本质上，传递`sink`给`toMat`创建另一个函数，该函数采用`Keep`值来产生一个实际的`Sink`。一个函数返回另一个函数。

### 物化值(Materialized Value)

物化值是流运行时的结果值。到目前为止，在我们的示例中，我们还真的不在乎物化价值。但是可以说，我们要计算流的平方的和，我们可以分别地将映射阶段`log`。然后，我们可以将其相加。让我们这样做：

```scala
  val logFlow = Flow[Int].map { i =>
    log.info("{}", i)
    i
  }
  
  val sum = Sink.reduce[Int](_ + _)
```

语言说明:

* 在此`map`示例中，您将看到花括号`{`和`}`的用法。在Scala中，这些花括号的含义与普通括号相同`(`和`)`，不同的是，它们可以用于多行闭包，如您在此处看到的。最后一个表达式`i`是该闭包的返回值。
* 在这里，您可以看到使用闭包的reduce示例`(_ + _)`。扩展为`((a, b) => a + b)`。第二个`_`代表第二个输入值，依此类推。这使得编写和读取`reduce`或`fold`闭包非常简单。如果变得更加复杂，请使用标准而不是此快捷方式闭包语法。

然后我们的运行方法将如下所示：

```scala
    val stream = src.via(filter).via(square).via(logFlow).toMat(sum)(Keep.right)
    stream.run()
```

等等，我们只是总结了流的结果。但是去哪儿了？当然，只有在流完成运行后，该总和才可用。这就是为什么我们得到一个`Future[Int]`。我们可以等待。但是请记住，在这种架构中等待是**一种犯罪**。因此，我们需要更具创造力。我们可以对`Future`做很多事情，但是现在我们想将其发送回询问它的测试中。让我们使用一个目前为止尚未介绍的actor特征`pipeTo`。让我们像下面一样修改`run`方法：

```scala
  private def run() = {
    val stream = src.via(filter).via(square).via(logFlow).toMat(sum)(Keep.right)
    stream.run().pipeTo(sender())
  }
```

现在让我们更改测试用例， 以期望返回结果。进行测试并进行如下更改：

```scala
  "The stream actor" should "Accept a ping request" in {
    streamActor ! "ping"
    val sum = expectMsgType[Int]
    sum shouldBe 171700
  }
```

我们现在也可以去掉最后的`sleep`。由于此测试在收到响应之前不会退出，因此在测试逻辑完成之前，我们不再需要补偿退出的测试。

现在运行测试并获得一些乐趣。

### Graph

到目前为止，我们仅处理相对线性的流。但是，大多数应用程序，流片段或组件不能是线性的。流本身根本不需要是线性的。虽然可以使用当前流语法组合非线性流，但是处理起来会很困难。这就是流图语法非常方便的地方。

图由GraphDSL定义，它引入了某些样板，如下所示：

```scala
    val graph = RunnableGraph.fromGraph(GraphDSL.create(sum) { implicit builder =>
      out =>
        import GraphDSL.Implicits._

        val bCast = builder.add(Broadcast[Int](2))
        val merge = builder.add(Merge[Int](2))

        src ~> bCast ~> filter ~> merge ~> logFlow ~> out
               bCast ~> square ~> merge
        
        ClosedShape
    })
```

`GraphDSL`块看起来有点模糊，但是就流图声明本身而言，值得这么做。让我们将其分解为以下组件：

1. 声明和工厂。上面的案例显示了一个`RunnableGraph`，这表示该图是完整的模板，可以立即运行。但是, 一个`GraphDSL`不一定总是需要创建一个`RunnableGraph`。它也可以创造一个`Flow`，`Source`或`Sink`。这些可以再次被其他组件使用`GraphDSL`或简单的流操作来组合。为了从`GraphDSL`构建一个`RunnableGraph`, 你会用`RunnableGraph.fromGraph(...)`构建它。同样，要构建一个`Flow`, 您将从`Flow.fromGraph(...)`开始。存在类似的方法 `Source.fromGraph(...)`和`Sink.fromGraph(...)`。

2. 图本身是由`GraphDSL.create(...)`创建的。`GraphDSL.create(...)`是一个高度重载的方法。它可以不接受不带任何参数到接受非常多的参数。在这个例子中显示的格式，只接收传递`sum`。通过这样做，我们说我们希望这个`RunnableGraph`/`Flow`/`Source`/`Sink`使用`sum`的物化值, 作为这个图的物化值。如果未传递任何参数，则物化值只是一个`Future[Done]`。
	 
	 当我们将多个流组件传递到`GraphDSL.create(...)`其中时，生活会变得更加有趣。在这种情况下，图形具有多个物化值。这当然是无效的。因此，我们需要传入另一个闭包, 名为`combineMat`。该匿名函数将`combineMat`采用*n*个参数，其中*n*是传入的流阶段数。然后，输出是基于组合输入的物化值参数的物化值。这允许并强制开发人员定义如何从所讨论的多个值中产生最终的物化值。最简单的情况是创建一个元组以返回每个物化值作为结果的一部分。
	 `GraphDSL.create`的最后一个参数是生成器匿名函数。这是一个分层的匿名函数，第一个以`builder`作为参数。我们需要标记此构建器为`implicit`以在下面的图中进一步使用。`GraphDSL.create`中的下一个匿名函数采用传入到的流阶段的导入版本。例如，如果您传递了3个阶段，则在此lambda中将有3个输入代表这些导入的阶段。这些将用于组成流图。在这种情况下，out表示sum传入。
The next lambda in takes an imported version of the stream stages passed into `GraphDSL.create`. For instance, if you have 3 stages passed in, you'll have 3 inputs in this lambda representing those imported stages. These will be used to compose the stream graph. In this case the `out` represents the `sum` passed in.
   
3. `builder.add(...)`调用。每一个没有传入的非线性流阶段(因此也不会捕获任何物化值)，通过将其传入`builder.add(...)`调用，就可用于合成。来自此块外部的线性级被隐式添加，因此我们只需要担心非线性级。

4. 图形组合。我们经历了所有的麻烦，就为了这个。在这里您将描述阶段/组件之间如何相互连接。

5. 返回值。这些需要与您要组合的图的类型匹配。根据组合的类型，你想要返回组合的相关形状。这是常见阶段类型的返回值：

   | Type            | Return value                          |
   | --------------- | ------------------------------------- |
   | `RunnableGraph` | `ClosedShape`           |
   | `Source`        | `SourceShape(outputPort)`          |
   | `Sink`          | `SinkShape(inputPort)`             |
   | `Flow`          | `FlowShape(inputPort, outputPort)` |


说的够多了。让我们进行一些代码。我们将创建一个新Scala类，非常类似于我们的`PaymentStreamActor`。让我们称之为`PaymentGraphActor`：

```scala
import akka.actor.{Actor, ActorLogging}
import akka.pattern._
import akka.stream.{ActorMaterializer, ClosedShape, FlowShape}
import akka.stream.scaladsl.{Broadcast, Flow, GraphDSL, Keep, Merge, RunnableGraph, Sink, Source}

class PaymentGraphActor extends Actor with ActorLogging {
  implicit val mat = ActorMaterializer()
  import context.dispatcher

  val src = Source(1 to 100)
  val filter = Flow[Int].filter(_ % 2 == 0)
  val square = Flow[Int].map(i => i * i)
  val logFlow = Flow[Int].map { i =>
    log.info("{}", i)
    i
  }

  val sum = Sink.reduce[Int](_ + _)

  override def receive: Receive = {
    case _ => run()
  }

  private def run() = {
    val graph = RunnableGraph.fromGraph(GraphDSL.create(sum) { implicit builder =>
      out =>
        import GraphDSL.Implicits._

        val bCast = builder.add(Broadcast[Int](2))
        val merge = builder.add(Merge[Int](2))

        src ~> bCast ~> filter ~> merge ~> logFlow ~> out
               bCast ~> square ~> merge
        
        ClosedShape
    })

    graph.run().pipeTo(sender())
  }
}
```

为此，我们还创建了测试`PaymentGraphSpec`类：

```scala
import akka.actor.{ActorSystem, Props}
import akka.testkit.{ImplicitSender, TestKit}
import org.scalatest.concurrent.ScalaFutures
import org.scalatest.{FlatSpecLike, Matchers}

class PaymentGraphSpec extends TestKit(ActorSystem("PaymentGraphSpec"))
  with FlatSpecLike with Matchers with ImplicitSender with ScalaFutures {

  val graphActor = system.actorOf(Props[PaymentGraphActor])

  "The graph actor" should "Accept a ping request" in {
    graphActor ! "ping"
    val sum = expectMsgType[Int]
    sum shouldBe 340900
  }
}
```

### `BidiFlow` - 双向流

在许多情况下，将流或流组件建模为双向流很有用。更常见的是，协议栈可以建模为双向流。甚至0.9中的squbs管道也被建模为一组`BidiFlow`，堆叠在一起。

###### 解剖一个`BidiFlow`

```scala
        +------+
  In1 ~>|      |~> Out1
        | bidi |
 Out2 <~|      |<~ In2
        +------+
```

###### 叠加的`BidiFlow`

```scala
       +-------+  +-------+  +-------+
  In ~>|       |~>|       |~>|       |~> toFlow
       | bidi1 |  | bidi2 |  | bidi3 |
 Out <~|       |<~|       |<~|       |<~ fromFlow
       +-------+  +-------+  +-------+

       bidi1.atop(bidi2).atop(bidi3);
```

但是，`BidiFlow`不仅对"literally"双向流有用。在某些情况下，我们需要在这些位置使用单个入口和出口点来控制关闭一部分流程。`BidiFlow`在这种情况下非常有用。以下squbs流组件是`BidiFlow`：

* TimeoutBidiFlow
* CircuitBreakerBidi

如我们所见，这些阶段避开了一部分流，以测试消息通过流的一部分，是否及时、是否错误。它还具有短路的能力，通过在其入口/出口点选择路径(例如超时消息)，如以下示例所示：

```scala
       +---------+  +------------+
  In ~>|         |~>|            |
       | timeout |  | processing |
 Out <~|         |<~|            |
       +---------+  +------------+
       
       timeout.join(processing);
```

在这里，timeout是一个`BidiFlow`，这里的processing仅是常规流阶段(或流阶段的组合)。箭头只是指向后面，使它连接起来。它只有一个输入和一个输出端口。

### 流阶段

到目前为止，我们只给出了少数几个最常见的阶段，如`Flow.map`，`Flow.filter`，`Broadcast`，`Merge`等，Akka流有许多阶段。众说纷纭。让您的浏览器指向[内置阶段及其语义的概述](https://doc.akka.io/docs/akka/current/stream/operators/index.html?language=scala)，并查看Akka提供的许多阶段中的某些阶段。

如果Akka提供的阶段不够多，则[Alpakka](https://developer.lightbend.com/docs/alpakka/current/)会提供更多阶段。squb(在Alpakka目录中列出)还提供了几个有趣的流阶段：

* PersistentBuffer (带有和不带有提交阶段)
* BroadcastBuffer (有和没有提交阶段)
* Timeout
* CircuitBreaker

#### 自定义阶段

如果我们发现阶段仍然不符合要求，则可以构建自己的阶段。[自定义流处理](https://doc.akka.io/docs/akka/current/stream/stream-customize.html?language=scala)中对此进行了记录。由于自定义流阶段的测试和资格鉴定非常复杂，因此，请让squbs团队参与其中。另外，如果您认为您的需求比用例更通用，那么squbs总是欢迎您的贡献！

### Stream Cookbook

对于如何使用Streams满足您的要求的想法，[Streams Cookbook](https://doc.akka.io/docs/akka/current/stream/stream-cookbook.html?language=scala)是获取您的想法的宝贵资源。
