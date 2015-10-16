#RequireJS

依据[RequireJS](http://requirejs.org/) 官方网站

> RequireJS 是一个 JavaScript 文件和模块加载器。它为浏览使用进行优化, 但它也可以用于其它JavaScript 环境… 使用模块加载器如RequireJS会提高你的代码质量和速度。

也就是说实际上是使用[RequireJS](http://requirejs.org/) 来模块化你的JavaScript。RequireJS通过实现一个叫做 [异步模块定义](http://wiki.commonjs.org/wiki/Modules/AsynchronousDefinition) (其他类似的想法包括 [CommonJS](http://www.commonjs.org/) )的半标准API来达到这一目的。使用 AMD（异步模块定义）可以在客户端解析和加载javascript模块同时允许服务器端优化。对于服务端优化模块依赖可以使用[UglifyJS 2](https://github.com/mishoo/UglifyJS2#uglifyjs-2)缩小和组合。

按照惯例RequireJS 需要 main.js 文件来启动它的模块加载器。


##部署
RequireJS 优化器一般不kick-in，直至执行部署的时候。 比如通过运行`start`, `stage` 或`dist` 任务。

如果你的构建中使用WebJars，那么RequireJS优化器插件也会确保从 WebJar内引用的任何 JavaScript 资源自动从[jsdelivr](http://www.jsdelivr.com/) CDN引用。另外如果有任何`.min.js` 文件，那么它会被用来代替`.js`。 这里的一个好处是，不需要改变你的html!


##实施和配置
当使用`PlayJava` 或`PlayScala` 插件时，可以简单添加插件到你的plugins.sbt文件来启用RequireJS 优化器：

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-rjs" % "1.0.7")
```

要添加插件到资产管道，你可以像以下声明它(假设管道只有一个插件 - 添加其它到序列，如digest 和 gzip ):

```scala
pipelineStages := Seq(rjs)
```

RequireJS 优化器提供了一个标准的构建配置文件，并且应该可满足大多数项目。请参阅[插件的文档](https://github.com/sbt/sbt-rjs#sbt-rjs) 了解更多配置信息。

注意RequireJS 执行很多工作，并且当它工作在Trireme下的in-JVM, 从效率的角度看你最好使用 Node.js 作为 js-引擎。为了方便你可以在`SBT_OPTS`设置`sbt.jse.engineType` 属性。例如在 Unix:

```shell
export SBT_OPTS="$SBT_OPTS -Dsbt.jse.engineType=Node"
```

参阅[插件的文档](https://github.com/sbt/sbt-rjs#sbt-rjs) 了解更多配置的信息。