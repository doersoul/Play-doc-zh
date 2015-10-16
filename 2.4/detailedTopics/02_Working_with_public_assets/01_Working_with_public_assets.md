#用公共资产工作

本章会讲到提供你的应用程序的静态资源，如JavaScript, CSS 和图像。

在Play中提供公共资源和服务其它任何HTTP请求是相同的。它使用同样的路由作为常规资源，使用 controller/action 的路径来发布CSS, JavaScript或图像文件到客户端。


## public/ 文件夹
按照惯例公共资产保存在应用程序的`public` 文件夹。可以将此文件夹按您喜欢的方式组织。我们推荐以下方式:

```
public
 └ javascripts
 └ stylesheets
 └ images
```

按照这个结构你就可以简单开始了, 但你理解其工作原理后可以随便修改它。


##WebJars
[WebJars](http://www.webjars.org/) 提供一个便利和传统的打包机制，它是Activator的sbt的一部分。例如你可以声明使用流行的[Bootstrap 库](http://getbootstrap.com/) ，简单地通过添加以下依赖到你的构建文件:

```sbt
libraryDependencies += "org.webjars" % "bootstrap" % "3.3.4"
```

为方便，WebJars 自动提取到与公共资产相关的`lib` 文件夹。例如, 如果你声明一个依赖[RequireJs](http://requirejs.org/) ，然后你可以从视图引用它，如下:

```html
<script data-main="@routes.Assets.at("javascripts/main.js")" type="text/javascript" src="@routes.Assets.at("lib/requirejs/require.js")"></script>
```

注意这个`lib/requirejs/require.js` 路径。`lib` 文件夹表示提取WebJar assets, `requirejs` 文件夹对应于WebJar artifactId, 和require.js 引用到WebJar的根所指的需要的资产。


##公共资产是如何打包的?
在构建处理期间, `public` 文件夹的内容被处理和添加到应用程序的类路径。

当你打包你的应用程序, 应用程序的所有资产, 包括所有子项目，都聚合到单个jar中, 在`target/my-first-app-1.0.0-assets.jar`这里。这个jar包括到发布中，因此Play应用程序可以提供他们。这个jar还可以用于部署资产到CDN或反向代理。


##资产控制器
Play自带内置控制器来提供公共资产。默认情况下, 这个控制器提供缓存, ETag, gzip和压缩支持。

这个控制器可以在默认的Play JAR获得，为`controllers.Assets`，并且定义了带二个参数的单个`at` action:

```scala
Assets.at(path: String, file: String)
```

`path` 参数必须是固定的，并且定义由action管理的目录。`file` 参数通常从请求路径中动态提取。

这里是在`conf/routes` 文件中常用的`Assets` 控制器的映射:

```scala
GET  /assets/*file        controllers.Assets.at(path="/public", file)
```

注意我们定义`*file` 动态部分匹配`.*` 正则表达式。 因此, 如果你发送这个请求到服务器:

```scala
GET /assets/javascripts/jquery.js
```

路由会调用`Assets.at` action，并带以下参数:

```scala
controllers.Assets.at("/public", "javascripts/jquery.js")
```

这个 action会查找和提供文件（如果文件存在的话）。


##公共资产的反向路由
对于在路由文件中映射的任何控制器, 一个反向路由会创建在`controllers.routes.Assets`。你可使用这个来反转需要获取公共资源的URL。例如, 从一个模板:

```html
<script src="@routes.Assets.at("javascripts/jquery.js")"></script>
```

这会产生以下结果:

```html
<script src="/assets/javascripts/jquery.js"></script>
```

注意当使用反向路由时，我们没有指定第一个`folder` 参数。这是因为我们的路由文件定义了单个`Assets.at` action映射,而`folder` 参数已经固定。因此它不需要指定。

但是,如果你定义二个`Assets.at` action映射, 像这个:

```scala
GET  /javascripts/*file        controllers.Assets.at(path="/public/javascripts", file)
GET  /images/*file             controllers.Assets.at(path="/public/images", file)
```

那么当使用反向路由时，你就需要指定二个参数:

```html
<script src="@routes.Assets.at("/public/javascripts", "jquery.js")"></script>
<img src="@routes.Assets.at("/public/images", "logo.png")" />
```


##公共资产的反向路由和指纹识别（fingerprinting）
[sbt-web](https://github.com/sbt/sbt-web) 给Play带来高度可配置的资产管道， 比如在你的构建文件:

```sbt
pipelineStages := Seq(rjs, digest, gzip)
```

上述代码会按RequireJs 优化 (`sbt-rjs`),  digester (`sbt-digest`) 和接着是压缩(`sbt-gzip`)来排序。不像许多sbt任务, 这些任务按照声明的顺序执行, 一个接一个。

本质上资产指纹允许您的静态资产与积极的缓存指令提供到浏览器。这将导致一种改进的用户体验，因为以后访问您的网站将减少需要下载的资产。Rails还描述了[资产指纹](http://guides.rubyonrails.org/asset_pipeline.html#what-is-fingerprinting-and-why-should-i-care-questionmark)的好处。

上述`pipelineStages` 的声明和必需的在`plugins.sbt`中的`addSbtPlugin` 声明的插件是你需要的起点。你必须接着声明Play要什么版本化的资产。以下路由文件记录声明所有版本化的资产:

```scala
GET  /assets/*file  controllers.Assets.versioned(path="/public", file: Asset)
```

> 确保你通过写上`file: Asset`指出这个`file` 是一个资产。

然后你可以使用反向路由, 例如在`scala.html` 视图内:

```html
<link rel="stylesheet" href="@routes.Assets.versioned("assets/css/app.css")">
```

我们强烈建议使用资产指纹。


##Etag 支持
`Assets` 控制器自动管理 **ETag** HTTP 标头。ETag 值是从 digest 生成(如果 `sbt-digest` 开始使用于资产管道) ，否则是从资源名称和文件的最后修改日期。如果资源文件嵌入到文件, JAR文件的最后修改日期会被使用。

当web浏览器创建一个请求指定这个 **Etag** ，那么服务器可以响应 **304 NotModified**。


##Gzip 支持
如果具有相同名称的资源但使用`.gz`后缀，那么`Assets` 控制器也会提供后者，并添加以下HTTP标头:

```scala
Content-Encoding: gzip
```

包含`sbt-gzip` 插件到你的构建，并在`pipelineStages`声明它的位置， 所有需求的文件会生成gzip文件。


##额外的`Cache-Control` 指令
对于缓存为目的来说使用 Etag 通常就够了。但是如果您要为特殊的资源指定一个自定义 `Cache-Control` 标头，你可以在`application.conf` 文件中指定它。例如:

```sbt
# Assets configuration
# ~~~~~
"assets.cache./public/stylesheets/bootstrap.min.css"="max-age=3600"
```


##托管资产
从Play 2.3开始通过基于插件的[sbt-web](https://github.com/sbt/sbt-web#sbt-web) 管理资产处理。在 2.3 之前Play 托管资产处理是以CoffeeScript, LESS, JavaScript linting (ClosureCompiler) 和 RequireJS 优化的形式。以下各节描述 sbt-web 和 如何实现相当于2.2的功能。 尽管注意到 Play 不限于资产处理技术，慢慢地sbt-web上越来越来的插件可以使用。请检出 [sbt-web](https://github.com/sbt/sbt-web#sbt-web) 项目以学习更多关于插件的知识。

许多插件使用sbt-web的[js-engine 插件](https://github.com/sbt/sbt-js-engine)。js-engine 能够执行写入插件到 Node API 或者在JVM内通过优秀的 [Trireme](https://github.com/apigee/trireme#trireme) 项目, 或直接在性能优越的 [Node.js](https://nodejs.org/) 。注意这些工具仅用在开发周期期间，并且不参与Play应用程序的运行期。如果你安装了Node.js，那么你 可以声明以下环境变量。对于Unix, 如果`SBT_OPTS` 被定义，那么你可以:

```sbt
export SBT_OPTS="$SBT_OPTS -Dsbt.jse.engineType=Node"
```

以上声明确保当执行任何sbt-web插件时Node.js被使用。