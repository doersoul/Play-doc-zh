#关于 SBT 设置


##关于 sbt 设置
`build.sbt` 文件定义你的项目设置。你还可以为你的项目定义自定义设置, 在[sbt 文档](http://www.scala-sbt.org/)有详细说明。尤其是, 它有助于熟悉sbt [设置](http://www.scala-sbt.org/release/docs/Getting-Started/More-About-Settings) 。

要进行基本设置, 使用 `:=` 操作符:

```scala
confDirectory := "myConfFolder"
```


##Java 应用程序的默认设置
Play 定义了适合基于Java的应用程序的默认设置。要启用他们，通过你的项目的enablePlugins方法添加`PlayJava` 插件。这些设置多数定义了生成模板的默认imports，如importing `java.lang.*` ，所以类型像`Long` 默认是 Java的而不是 Scala的。`play.Project.playJavaSettings` 也导入`java.util.*` 所以默认集合库会是Java的。


##Scala 应用程序的默认设置
Play 定义了适合基于Scala的应用程序的默认设置。要启用他们，通过你的项目的enablePlugin方法添加`PlayScala`插件。这些默认设置为生成模板定义了默认imports(如国际化消息, 和核心APIs)。