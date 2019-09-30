# Akka Scala/squbs培训

## Day1: 概念&概述

### Akka概述

#### 为什么是Akka?
   * Actor
      * 简单抽象
      * 高性能
      * 异步
      * 无争执(No contention)
   * 使用层级结构监督的容错
   * 流: 弹性 & 背压
   * 面向消息的架构

#### 响应式宣言
   * 在所有情况下都响应灵敏
   * 能够承受内部和外部故障
   * 弹性负载尖峰
   * 消息驱动, 异步体系结构使这一切成为可能

#### ActorSystem
   * 基本基础结构, 用于运行actor，流等
   * 有一个调度器，用来调度actor
   * 安排actor在调度程序(线程池)上运行

#### Actor计算模型
   * 由Hewitt等人定义
   * 目前我们拥有的最具可扩展性的架构
   * **规则**:
      * Actor, 在接收消息时可以:
         * 发送信息
         * 创建其他actor
         * 更改状态/行为, 用于下一条消息

#### Akka Actor实例
   * 有一个邮箱
   * 处理邮箱中的消息
   * 可以在一个调度周期内处理多条消息

#### 主流Akka Actor是无类型的
   * 他们可以接收/发送任何类型的任何消息
   * Akka Typed尝试为actor添加强类型

#### Future
   * Akka API使用Scala `Future`处理将来完成的事情
   * `Future` 是可变形/可组合

#### 集群/远程
   * Actor是位置透明的
   * 通过网络发送/接收消息

#### 流
   * 比Actor更高层次的抽象
   * 类型安全
   * 提供背压，因此具有弹性(系统永不过载)
   * 提供类似电路板(circuit board)的组合流组件的编程模型
   * squbs的核心部件，从squbs 0.9.0 (我们即将发布0.11.0)
   * **功课**: 请观看QCon的视频，有关squbs和流处理。 该视频可从[http://squbs](http://squbs)获得

### 一些编程规则
* 首先是不可变
   * 即Spring beans本质上是可变的. Java bean通常是可变的
   * 默认情况下，Scala集合和样例类是不可变的，但有可用的可变的版本
   * 如果需要可变，则遵循以下优先顺序
      a. 不可变的
      b. 对不可变对象的可变引用
      c. 对可变对象的不可变引用
   * **不可接受**: 对可变对象的可变引用
   * 将可变性范围限制到最小
   * 不要使用构建器模式去构建更复杂的不可变对象
* 永远不要阻塞
   * 即永远不要等待任何类型的future-这样的等待块。对于Scala `Future`，这就是`Await`调用。除测试外，请勿使用它。
   * 使用功能组合来避免阻塞
   * 如果您无法避免阻塞，例如需要进行阻塞的I/O调用，请为阻塞代码指定一个阻塞的调度程序。调整这个调度程序。
* 没有`null`。使用`Option`... 这是Scala的标准约定，而不是Java。
* 不要使用并发性结构和库。除了原子(`java.util.concurrent.atomic`程序包的成员)以外，几乎所有功能在此体系结构中都是禁入的。并发已经为您管理。
   * 不要创建线程或线程池。
* 不要依靠`ThreadLocal`。它们不起作用，并确实会导致间歇性故障。
* 不要创建其他`ActorSystem`. 让squbs管理`ActorSystem`。
* 尽可能少创建流`Materializer`并最大程度地重用 - 它们非常昂贵。提供了一种无需创建新实例即可巧妙的访问默认实例化器的功能。
* 对于HttpClient，请**始终**消费或丢弃`Entity`，即使收到错误响应。否则会导致背压，并可能使流卡住。

### 什么是squbs?
* PayPal的开源项目
   * https://github.com/paypal/squbs
* 包含
   * 启动 & 生命周期管理
   * 模块管理
   * 服务管理
   * 扩展管理
   * HttpClient
   * 管道(客户端/服务器)
   * 测试工具
   * 编排器(Orchestrator)
   * 流组件
   * 复杂的 marshallers/unmarshallers
   * 验证
   * 管理控制台
   * 监控
   * 更多东西

### rocksqubs
* squbs的PayPal内部扩展
* ASF
* CAL
* 远程配置
* DAL
* HttpClient扩展
* Kafka
* Kernel
* Mayfly
* 元数据
* 消息生产者和守护程序架构
   * AMQ
   * YAM
   * Kafka
* 模式
* 管道
* ValidateInternals & Servlet引擎

### 如何获得帮助？
* Slack #help-squbs
* https://gitter.im/paypal/squbs - 为外部问题/社区
* 文档位于 http://squbs (内部)和 https://squbs.readthedocs.io/en/latest/ (外部). 有一个https://github.com/paypal/squbs链接。
* Akka文档: http://akka.io/docs/ 指向JavaDoc

### 让我们创建您的第一个squbs应用
* Altus演示并允许所有人创建应用程序
* 将应用程序导入IDE
* 查看默认的应用程序结构
