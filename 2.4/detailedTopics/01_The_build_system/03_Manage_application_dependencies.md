#管理库依赖


##非托管依赖
多数人最终会使用托管的依赖 - 这允许细粒度的控制, 但在刚开始时非托管的依赖会比较简单。

非托管的依赖这样工作: 创建一个`lib/` 目录，放在项目的根目录下，然后添加 jar文件到这个目录。他们会自动被添加到应用程序的类路径。没有别的了，就这么简单!

使用非托管的依赖，没有东西需要添加到`build.sbt`。但是如果你喜欢使用和 `lib`不同的目录，你需要更改配置的key。


##托管依赖
Play使用Apache Ivy (通过sbt) 来实现管理依赖, 因此如果你熟悉Maven或Ivy, 你不会有什么问题。

多数时候你可以在 `build.sbt` 文件简单地列出你的依赖。

声明一个依赖看起来像这样 (定义 `group`, `artifact` 和 `revision`):

```sbt
libraryDependencies += "org.apache.derby" % "derby" % "10.11.1.1"
```

或像这样, 还一个可选项`configuration`:

```sbt
libraryDependencies += "org.apache.derby" % "derby" % "10.11.1.1" % "test"
```

可以像上面那样添加多个依赖项, 或你可以提供一个Scala 序列:

```sbt
libraryDependencies ++= Seq(
  "org.apache.derby" % "derby" % "10.11.1.1",
  "org.hibernate" % "hibernate-entitymanager" % "4.3.9.Final"
)
```

当然, sbt (通过 Ivy) 必须知道在哪里可以下载模块。如果你的模块是默认仓库之中的，那么可以直接工作。

###用 `%%` 获得正确的Scala版本
如果你使用`groupID %% artifactID % revision` 而非`groupID % artifactID % revision` (不同的地方是`groupID`后面的`%%` ), sbt 会自动添加项目的Scala版本的合适版本。这只是一个快捷方式。

你也可以这样写，无须`%%`:

```sbt
libraryDependencies += "org.scala-tools" % "scala-stm_2.9.1" % "0.3"
```

假设你的构建的 `scalaVersion` 是`2.9.1`, 以下是同等的:

```sbt
libraryDependencies += "org.scala-tools" %% "scala-stm" % "0.3"
```

###Resolvers
sbt 使用标准的Maven2仓库和默认Typesafe Releases ([https://repo.typesafe.com/typesafe/releases](https://repo.typesafe.com/typesafe/releases)) 仓库。如果你的依赖不在默认仓库中, 你需要添加resolver来帮助Ivy来找到它。

使用`resolvers` 设置 key来添加你自己的resolver。

```sbt
resolvers += name at location
```

举例:

```sbt
resolvers += "sonatype snapshots" at "https://oss.sonatype.org/content/repositories/snapshots/"
```

sbt 可以搜索你的本地Maven仓库:

```sbt
resolvers += (
    "Local Maven Repository" at "file:///"+Path.userHome.absolutePath+"/.m2/repository"
)
```