## Day 2: 功能组合

### 容器类型
* 我们每天处理的容器类型: `List[T]`, `Set[T]`
* 即使数组是一个`Array[T]`
* 常用的容器类型，我们将需要: `Option[T]`, `Future[T]`
* 一个`List[T]`或`Set[T]`包含0或多个元素
* 你可以认为`Option[T]`是包含0个(空)或一个元素的容器. `Option[T]`有两个子类型: `Some[T]`或`None`
* 一个`Future[T]`包含一个元素或一个错误，其在将来填充到结果中。

### 转换, 就像我们以前用Java做的那样

```scala
  def intListToString(intList: List[Int]): List[String] = {
    val stringBuffer = new ArrayBuffer[String](intList.size)
    intList.foreach { i =>
      stringBuffer += "count: " + i
    }
    stringBuffer.toList
  }
```

### 不! 我们可以做得更好！
* 给定一个`List[Int]`和一个可以转换`Int` => `String`的函数，我们应该能够一次将这一转换应用于整个`List`。

让我们大致看一下这些Scala代码(更容易理解)：

```scala
def intListToString(intList: List[Int]) = intList.map(i => "count: " + i)
```

这将一个`List(1, 2, 3)`转换为一个`List("count: 1", "count: 2", "count: 3")`。

### `map`操作的一些原理

* `List`, `Set`等是容器类型。
* 让容器类型: `C`和容器内的元素类型: `E`。
* 现在我们可以说容器是类型`C[E]`
* 如果应用一个`E => F`类型的函数, 则结果将为`C[F]`类型
* 此应用程序称为`map`操作。
* 所以: `C[E].map(E => F) =>> C[F]`

这应该很容易！

### 组合使用`flatMap`

* 对于`C[E]`类型，假如我们应用一个函数`E -> C<F>`，将会得到什么？
* 更具体的例子: 假如我们有`List[String]`和一个函数`String => List[Char]` 即将`String`分割成一个字符列表, 那么`map`操作将怎么做？
* 答案是, `List[List[Character]]`
* 或`C[C[F]]`根据我们的符号
* 那么，假如我们想要所有字符串的字符列表怎么办？我们必须`flatten`它。
* 这`flatMap`是派上用场的地方。
* 因此，理论上: `C[E].flatMap(E => C[F]) =>> C[F]`
* 我们使用一个容器类型`C[F]`，扁平映射(flatMap)容器`C[E]`，并获得一个`C[F]`类型的组合结果。

明白?

### 尝试`flatMap`两个列表:

使用Scala REPL，我们可以在以下示例中显示`flatMap` vs `map`的效果：

```scala
scala> val l1 = List("one", "two", "three")
l1: List[String] = List(one, two, three)

scala> val l2 = l1.flatMap(s => s.toList)
l2: List[Char] = List(o, n, e, t, w, o, t, h, r, e, e)

scala> val l3 = l1.map(s => s.toList)
l3: List[List[Char]] = List(List(o, n, e), List(t, w, o), List(t, h, r, e, e))
```
### 组合`map`和`flatMap`

* 如果我们想对多个容器一起操作，该怎么办？将它们组合在一起。
* 给定容器`C[E]`和`C[F]`，我们想应用一个函数`(E, F) => T`并从中获得`C[T]`。
* 合并看起来像这样: `C[E].flatMap(E => C[F].map(F => [(E, F) => T]))`
* 同样，应用`(E, F, G) => T`合并`C[E]`, `C[F]`, `C[G]`看起来像这样: `C[E].flatMap(E -> C[F].flatMap(F => C[G].map(G => [(E, F, G) => T])))`并给你一个`C[T]`.

### 合并&处理`Optional`示例

```scala
    def combine(o1: Option[String], o2: Option[String]): Option[String] =
      o1.flatMap(s1 => o2.map(s2 => s1 + s2))
```

* 现在尝试`combine(Option("Hello"), Option("World"))`
* 并尝试使一个或另一个为空。

### 组合使用for-推导式

虽然您可能已经在Scala中看到了一些类似循环构造的`for`用法，但实际上并非如此。for-推导式实际上是语法糖，它允许更容易地合成/组合在前面的部分中描述的`map`和`flatMap`。所以我们上面的组合函数可以这样重写：

```scala
    def combine(o1: Option[String], o2: Option[String]): Option[String] =
      for {
        s1 <- o1
        s2 <- o2
      } yield s1 + s2
```

是的，你多花了几行。但是，生成的代码比flatMap和map组合更具可读性。生成的编译代码完全相同。当您拥有大量要组合的容器时，这一点变得更加明显，如以下四个容器的组成所示：

```scala
  for {
    s1 <- o1
    s2 <- o2
    s3 <- o3
    s4 <- o4
  } yield s1 + s2 + s3 + s4
```

让我们试着用这个来写`flatMap`和`map`，我们将清楚地看到这需要付出更多的努力。

### 合并Future

* 为什么？因为您希望合并future，而不是阻塞每个或任何future。
* 您需要对future的内容进行操作。
* 而Future只是另一种容器类型。
* 适用相同的组合规则。

### 在Actor中使用组合Future

* 异步`Future`操作需要使用`pipe`操作将一个actor消息发送到这个actor或另外一个actor，而不会阻塞。小心不要阻塞

### 练习:

写一个测试用例，由`ask`多次调用`PaymentActor`(多个`PaymentRequests`)，并将`Future`的结果组合成单独的一个。使用Await调用阻塞`Future`的结果，以完成测试用例。这个`Future`应包含一个所有结果的`List`。
