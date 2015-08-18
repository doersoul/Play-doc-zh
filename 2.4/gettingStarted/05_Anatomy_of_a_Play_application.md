#解析一个Play应用程序

##Play应用程序布局
Play应用程序的布局是标准化的，以让事情尽可能简单。在首次成功编译之后,一个Play应用程序看起来像这样:

```
app                      → 应用程序主要源代码
 └ assets                → 编译的资产源文件
    └ stylesheets        → 通常是 LESS CSS 源代码
    └ javascripts        → 通常是 CoffeeScript 源代码
 └ controllers           → 应用程序控制器
 └ models                → 应用程序商业逻辑层
 └ views                 → 模板
build.sbt                → 应用程序构建脚本
conf                     → 配置文件和其它非编译资源 (在classpath)
 └ application.conf      → 主配置文件
 └ routes                → 路由定义
dist                     → 包含到你的工程发布中的任意文件
public                   → 公共资产
 └ stylesheets           → CSS 文件
 └ javascripts           → Javascript 文件
 └ images                → 图像文件
project                  → sbt 配置文件
 └ build.properties      → sbt 项目的标记
 └ plugins.sbt           → sbt 插件包括Play 自身声明
lib                      → 非托管的库依赖项
logs                     → 日志文件夹
 └ application.log       → 默认日志文件
target                   → 生产的内容
 └ resolution-cache      → 关于依赖的信息
 └ scala-2.10
    └ api                → 生成的 API docs
    └ classes            → 编译的类文件
    └ routes             → 从路由生成的源
    └ twirl              → 从模板生成的源
 └ universal             → 应用程序打包
 └ web                   → 编译的 web 资产
test                     → 单元和功能测试的源文件夹
```

##`app/` 目录
`app`目录包含所有可执行的artifacts: Java和Scala源代码, 模板和编译的资产的源文件。

在`app`目录有三个包, 是MVC架构模式的每一个组件:

* `app/controllers`
* `app/models`
* `app/views`

当然你可以添加自己的包, 例如一个 `app/utils` 包。

> 注意在Play, controllers, models 和 views 包的名字现在只是一个约定，有需要可以更改它(比如每个都加上`com.yourcompany`为前缀)。

这里还有一个可选的目录叫 `app/assets`，为了编译某些资产，如[LESS源文件](http://lesscss.org/)和[CoffeeScript源文件](http://coffeescript.org/)。

##`public/` 目录
资源存储在`public`目录，这是一些静态的资产，这些由Web服务器直接提供服务。

这个目录分成三个子目录来保存图像, CSS样式表和JavaScript文件。你应当像这样整理你的静态资产，以保持所有Play应用程序的一致性。

> 在一个新建的应用程序,  `/public` 目录映射到 `/assets` URL路径, 但你可以很容易的更改它, 甚至还可以为你的静态资产使用几个目录。

##`conf/` 目录
`conf` 目录包含应用程序的配置文件。这里有二个主要的配置文件:

* `application.conf`, 应用程序的主配置文件, 包含配置参数
* `routes`, 路由定义文件

如果你需要添加特定于应用程序的配置选项, 添加更多选项到`application.conf`文件是一个好主意。

如果一个库需要一个特定的配置文件, 可以试试在`conf`目录下的文件。

##`lib/` 目录
`lib` 目录是可选的，包含非托管库依赖关系, 例如所有你想要在构建系统外手动管理的JAR文件。只需要拖动任何JAR文件到这里，他们会添加到你的应用程序的Classpath。

##`build.sbt` 文件
你的项目的主构建声明通常是在工程根目录下的`build.sbt`中。在 `project/` 目录中的 `.scala`文件也能用来声明你的项目的构建。

##`project/` 目录
`project` 目录包含sbt构建定义:

* `plugins.sbt` 定义这个项目使用的sbt插件
* `build.properties` 包含用来构建你的app的sbt版本。

##`target/` 目录
`target` 目录包含构建系统生成的所有东西。它对了解这里生成了什么东西是很有用的。

* `classes/` 包含所有编译的类(从Java源代码和Scala源代码).
* `classes_managed/` 仅包含通过框架管理的类(如通过路由或模板系统生成的类)。它对在你的IDE项目中，添加此类文件夹做为外部类文件夹很有用。
* `resource_managed/` 包含生成的资源, 通常是编译的资产，如LESS CSS和CoffeeScript编译的结果。
* `src_managed/` 包含生成的源文件, 如通过模板系统生成的Scala源文件。
* `web/` 包含通过sbt-web处理的资产，如那些`app/assets` 和 `public` 文件夹中的。

##典型的 .gitignore 文件
某些生成的文件夹应该被你的版本管理系统忽略。这里是一个你的Play应用典型的 `.gitignore` 文件:

```
logs
project/project
project/target
target
tmp
dist
.cache
```

##默认 SBT 布局
你也可以选择用SBT和Maven的默认布局。请注意这个布局是实验性的和可能会有些问题。为了使用这个布局, 得使用 `disablePlugins(PlayLayoutPlugin)`。这样会停止Play重写默认SBT布局, 看起来像这样:

```
build.sbt                  → 应用程序构建脚本
src                        → 应用程序源代码
 └ main                    → 编译的资产源
    └ java                 → Java 源
       └ controllers       → Java 控制器
       └ models            → Java 商业逻辑层
    └ scala                → Scala 源
       └ controllers       → Scala 控制器
       └ models            → Scala 商业逻辑层
    └ resources            → 配置文件和其它非编译资源 (在 classpath)
       └ application.conf  → 主配置文件
       └ routes            → 路由定义
    └ twirl                → 模板
    └ assets               → 编译的资产源
       └ css               → 通常是LESS CSS 源文件
       └ js                → 通常是CoffeeScript 源文件
    └ public               → 公共资产
       └ css               → CSS 文件
       └ js                → Javascript 文件
       └ images            → 图像文件
 └ test                    → 单元或功能测试
    └ java                 → 单元或功能测试的Java 源文件夹
    └ scala                → 单元或功能测试的Scala 源文件夹
    └ resources            → 单元或功能测试的资源文件夹
 └ universal               → 包括在你的项目发布中的任意文件
project                    → sbt 配置文件
 └ build.properties        → sbt 项目的标记
 └ plugins.sbt             → sbt 插件包括Play 自身声明
lib                        → 非托管的库依赖
logs                       → 日志文件夹
 └ application.log         → 默认日志文件
target                     → 生成的东西
 └ scala-2.10.0            
    └ cache              
    └ classes              → 编译的类文件
    └ classes_managed      → 管理的类文件 (模板, ...)
    └ resource_managed     → 管理的资源 (less, ...)
    └ src_managed          → 生成的源 (模板, ...)
```