# 面向Scala用户的squbs培训

### 先决条件

1. 准备好一台笔记本电脑。 Windows和Mac都不错。你坐在办公桌前参加，你也可以使用台式电脑。
2. 更新的Java版本。支持的最低版本是JDK 1.8.0_60。任何后续版本或Oracle站点上的最新版本。
3. 安装IntelliJ Idea社区版，或升级到最新版本。它是免费。确保您至少拥有2017年后的版本，因为其中一些界面已大大简化。您可以在这里获得它：https : //www.jetbrains.com/idea/download/
4. 启动IntelliJ Idea并安装Scala插件。
5. 从其[下载站点](https://www.scala-sbt.org/download.html)安装sbt.
6. 使用[squbs-scala-seed](https://github.com/paypal/squbs-scala-seed.g8)创建新项目, 通过运行`sbt new ,paypal/squbs-scala-seed.g8`
7. 在IntelliJ中，通过选择`File`->`Open`打开项目，然后选择位于克隆的git目录的根级别的`build.sbt`文件(其他子项目中有很多`build.sbt`文件。请确保选择一个是位于克隆的项目的根目录中的文件)。
8. IntelliJ将提示您是否要以`File`或`Project`打开。选择`Project`。
9. 接下来是`Import`屏幕。
   *  选中Library sources复选框。
   *  检查Project JDK并确保它是您想要的版本。同样，我们需要使用JDK 1.8.0_60或更高版本。要将新的JDK注册到IntelliJ，请点`New`按钮。选择JDK，然后找到并选择所需的JDK。
   *  点 `Continue`
10. 在此阶段，IntelliJ将下载并解析项目所需的所有库。根据网络带宽的不同，这可能需要2分钟到3个小时, 让它结束。
11. 我们应该在教程开始时准备好它们的工作区。

有关更多详细信息，请参见squbs文档中的[如何设置开发环境](https://engineering.paypalcorp.com/squbs/doc/Squbs/0.11.X/docs/mds/development_environment.md)。

### 入门

从[docs](docs)文件夹开始教程。它是按顺序组织的。
