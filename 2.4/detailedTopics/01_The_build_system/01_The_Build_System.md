#构建系统

Play构建系统使用[sbt](http://www.scala-sbt.org/), 一个Scala和Java项目的高性能集成构建系统。使用sbt作为我们的构建工具是play必备的，本页会说明这一点。


##理解 sbt
sbt与许多传统的构建任务相当不同。从根本上, sbt 是一个任务引擎。你的构建是表示为需要执行的任务依赖树, 例如, `compile` 任务依赖于`sources` 任务, 而这个又依赖于`sourceDirectories` 任务和`sourceGenerators` 任务, 如此等等。

sbt将典型的构建执行打破，成为非常细粒度的任务, 任何任务在树的任何点都可以在你的构建中任意定义。这让sbt非常强大, 但是如果你是从其它粗粒度的构建工具转过来的，还需要思想的转变。

这里的文档在一个非常高的级别描述Play的sbt用法。当你开始更多地使用sbt项目, 建议您遵循 [sbt 教程](http://www.scala-sbt.org/0.13/tutorial/index.html) 以理解sbt如何组合。另一个人们发现的有用的资源是 [这一系列的博客文章](https://jazzy.id.au/2015/03/03/sbt-task-engine.html).


##Play 应用程序目录结构
多数人刚开始使用Play时，是使用`activator new` 命令，它产生一个目录结构，如:

* `/`: 应用程序的根文件夹
* `/README`: 一个文本文件，描述应用程序部署信息。
* `/app`: 你的应用程序的代码主要保存在这里。
* `/build.sbt`:  [sbt](http://www.scala-sbt.org/) 设置，描述应用程序的构建。
* `/conf`: 应用程序的配置文件。
* `/project`: 进一步建立描述信息。
* `/public`: 静态文件, 你的应用程序的公共资产保存在这里。
* `/test`: 你的应用程序的测试代码保存在这里。

现在, 我们要关注的是`/build.sbt` 文件和`/project` 目录了。


##`/build.sbt` 文件
当你使用`activator new foo` 命令, 构建说明文件`/build.sbt`会生成，就像这样:

```scala
name := "foo"

version := "1.0-SNAPSHOT"

libraryDependencies ++= Seq(
  jdbc,
  anorm,
  cache
)

lazy val root = (project in file(".")).enablePlugins(PlayScala)
```

`name` 这一行定义应用程序的名称，它会和你的应用程序的根目录名字相同, `/`, 这是根据你的`activator new` 命令后的参数决定的。

`version` 这一行提供你的应用程序的版本，这是用来作为生产版本名称的一部分。

`libraryDependencies` 这一行指这下你的应用程序要依赖的库。下面会详细讲解。

你要为Java或Scala分别使用`PlayJava` 或`PlayScala` 插件来配置sbt。

###使用 scala 构建
Activator 也可以从项目的`project` 下的scala文件构造构建需求。建议的做法是使用`build.sbt` ，但有时需要直接使用scala。也许是因为你正在迁移较早的项目, 那么以下是几个有用的导入:

```scala
import sbt._
import Keys._
import play.Play.autoImport._
import PlayKeys._
```

`autoImport` 这一行是导入一个sbt插件的自动声明属性的正确方法。按同样的方法, 如果您要导入一个sbt web插件，那么得这样:

```scala
import com.typesafe.sbt.less.autoImport._
import LessKeys._
```


##`/project` 目录
与构建你的项目相关的一切都是保留在你的应用程序目录的`/project` 目录下面。这是一个[sbt](http://www.scala-sbt.org/) 必备的条件。在这个目录里面, 有二个文件:

* `/project/build.properties`: 这是个标记文件，用来声明使用的sbt的版本。
* `/project/plugins.sbt`: 通过项目构建包括的Play使用的SBT插件。


##sbt的Play 插件(`/project/plugins.sbt`)
Play 控制台和所有它的开发功能像自动重载等，都是通过sbt插件实现。它注册在`/project/plugins.sbt` 文件中:

```scala
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % playVersion) // 其中版本是当前Play版本, 如"2.4.0" 
```

> 注意当你更改play版本时，`build.properties` 和 `plugins.sbt` 必须手动更新。
