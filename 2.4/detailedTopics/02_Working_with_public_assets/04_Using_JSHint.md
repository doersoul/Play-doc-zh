#使用 JSHint

来自[网站文档](http://www.jshint.com/about/):

> JSHint 是一个社区驱动的工具，用来在JavaScript代码中检测错误和潜在问题，并强制你的团队成员的编码惯例。它是非常灵活的，因此你可以很容易地调整您的特定编码准则和你希望代码执行的环境。

任何存在于`app/assets` 的JavaScript文件都会通过JSHint处理和检查其错误。


##检查 JavaScript sanity
当在开发模式下刷新浏览器以及用`assets`命令时JavaScript代码会被编译。就像任何其它编译错误一样，其错误也会显示在浏览器中。


##实施和配置
当使用`PlayJava` 或`PlayScala` 插件时，可以简单添加插件到你的 plugins.sbt 文件来启用JSHint:

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-jshint" % "1.0.3")
```

插件的默认配置通常是足够的。请参阅[插件的文档](https://github.com/sbt/sbt-jshint#sbt-jshint)了解更多信息。