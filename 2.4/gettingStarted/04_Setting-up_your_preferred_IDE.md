#设置你的首选IDE

使用Play很容易。你甚至不需要复杂的IDE, 因为Play在你修改源文件是会自动编译和刷新，所以你可以轻松地使用一个简单的文本编辑器。

但是使用一个现代Java或Scala IDE可以提供强大的功能，像自动完成,即时编译,重构辅助和调试。

##Eclipse
###设置 sbteclipse
Play需要[sbteclipse](https://github.com/typesafehub/sbteclipse) 4.0.0 或更新版本。添加下面内容到你的 project/plugins.sbt文件:

```
addSbtPlugin("com.typesafe.sbteclipse" % "sbteclipse-plugin" % "4.0.0")
```

在运行`eclipse`命令之前，你必须先`编译`你的工程。通过在`build.sbt` 添加以下设置，你可以强制当`eclipse`命令运行时自动进行编译:

```scala
// 生成Eclipse文件前编译项目, 因此生成的 .scala 或 .class 文件为当前 views 和 routes
EclipseKeys.preTasks := Seq(compile in Compile)
```

如果你的项目中有Scala源代码, 你需要安装[Scala IDE](http://scala-ide.org/)。

如果你不想安装Scala IDE和你的项目中仅有Java源代码, 那么你可以按如下设置:

```scala
EclipseKeys.projectFlavor := EclipseProjectFlavor.Java           // Java 项目. 不用Scala IDE
EclipseKeys.createSrc := EclipseCreateSrc.ValueSet(EclipseCreateSrc.ManagedClasses, EclipseCreateSrc.ManagedResources)  // 为视图和路由使用 .class 文件而非生成 .scala 文件， 
```

###生成配置
Play提供一个命令以简化[Eclipse](https://eclipse.org/)配置。要转换Play应用程序到可以工作的Eclipse工程, 使用`eclipse`命令：

```shell
[my-first-app] $ eclipse
```

如果你想抓取可用的jars源 (这会花费很长时间并且一些源可能会丢失):

```shell
[my-first-app] $ eclipse with-source=true
```

> 注意如果你使用集成子项目, 你需要在`build.sbt`中适当设置`skipParents`:

```scala
EclipseKeys.skipParents in ThisBuild := false
```

或从play控制台, 键入:

```scala
[my-first-app] $ eclipse skip-parents=false
```

然后你需要在 **File/Import/General/Existing project…** 菜单将应用程序导入到你的工作空间(需要先编译你的项目)。

![""](eclipse.png)

要调试, 用 `activator -jvm-debug 9999 run` 启动你的应用程序，并在Eclipse中右击项目，选择 **Debug As, Debug Configurations**。在 **Debug Configurations** 窗口, 右击 **Remote Java Application** 和选择 **New**。将端口改为9999和点击 **Apply**。从现在起你可以点击 **Debug** 来连接到正在运行的应用程序。停止调试会话不会停止服务器。

> 提示: 你可以使用 `~run` 来运行你的应用程序，以启用更改文件时自动直接编译的功能。这样当你在`视图`目录创建一个新模板时，scala模板文件会被自动发现，而且当文件更改时自动编译。如果你使用普通的`run`，那么每次你都要手动在浏览器点击`刷新`。

如果你在应用程序中做了重要的更改, 如改变classpath, 要再次使用`eclipse`以重新生成配置文件。

> 提示: 当你在一个团队中工作时，不要提交Eclipse配置文件!

生成的配置文件包含到你的框架安装位置的绝对引用。这些特定于你自己的安装位置。当你工作在一个团队中时, 每个开发者必须保持他的Eclipse配置文件是私有的。


##IntelliJ
[Intellij IDEA](https://www.jetbrains.com/idea/)让你无须使用命令提示符都可以快速创建一个Play应用程序。你不需要配置IDE以外的任何东西, SBT构建工具会安排下载适当的库, 解决依赖项和构建项目。

在IntelliJ IDEA中开始创建一个Play应用程序之前, 请确保已安装最新Scala插件并启用它。即使你不用Scala开发, 它也会帮助你使用模板引擎和解决依赖项。

要创建一个Play应用程序:

1. 打开 **New Project** 向导, 选择在 **Scala** 下面的 **Play 2.x** ，点击 **Next**。
2. 输入你的项目信息并点击 **Finish**.

IntelliJ IDEA 将使用SBT创建一个空应用程序。

你也可以导入一个已存在的Play项目。

要导入一个Play项目:

1. 打开 Project wizard, 选择 **Import Project**。
2. 在打开的窗口, 选择你想导入的项目，并点击 **OK**。
3. 在向导的下一页, 选择 **Import project from external model** 选项, 选择 **SBT project** 和点击 **Next**。
4. 在向导的下一页，选择附加导入选项，并点击 **Finish**。

检查项目结构, 确保所有必须的依赖都已下载。你可以使用代码辅助, 导航和动态代码分析功能。

你可以运行刚创建的应用程序，并且用浏览器在默认位置`http://localhost:9000`查看结果。要运行Play应用程序:

1. 创建一个新的Run配置 – 从主菜单, 选择 Run -> Edit Configurations
2. 点击 + 来添加一个新的配置
3. 从配置列表，选择 “SBT Task”
4. 在 “tasks” 输入框, 简单输入 “run”
5. 应用更改和选择 OK
6. 现在你可以从主菜单的Run菜单中选择 “Run”，运行你的应用程序

你可以轻松为Play应用程序开始一个调试会话，使用默认 Run/Debug 配置设定。

要了解更详细信息, 参阅下面的Play Framework 2.x 教程:

[https://confluence.jetbrains.com/display/IntelliJIDEA/Play+Framework+2.0](https://confluence.jetbrains.com/display/IntelliJIDEA/Play+Framework+2.0)

###从错误页面导航到源代码位置
使用 `play.editor` 配置选项, 你可以设置Play添加超链接到一个错误页面。其后, 你可以从错误页面轻松导航到IntelliJ, 直接进入源代码位置(你需要首先安装 Remote Call [https://github.com/Zolotov/RemoteCall](https://github.com/Zolotov/RemoteCall) IntelliJ 插件)。

只要安装Remote Call 插件，并用下面的选项运行你的应用程序:

```
-Dplay.editor=http://localhost:8091/?message=%s:%s -Dapplication.mode=dev
```


##Netbeans
###生成配置
目前Play没有原生的Netbeans项目生成支持, 但有一个NetBeans的Scala插件可以帮助使用Scala语言和SBT:

[https://github.com/dcaoyuan/nbscala](https://github.com/dcaoyuan/nbscala)

还有一个SBT插件可创建Netbeans项目定义:

[https://github.com/dcaoyuan/nbsbt](https://github.com/dcaoyuan/nbsbt)

##ENSIME
###安装 ENSIME
按照安装说明，在 [https://github.com/ensime/ensime-emacs](https://github.com/ensime/ensime-emacs)。

###生成配置
编辑 project/plugins.sbt 文件, 和添加以下行 (你应该先检查 [https://github.com/ensime/ensime-sbt](https://github.com/ensime/ensime-sbt) 获得插件最新版本):

```
addSbtPlugin("org.ensime" % "ensime-sbt" % "0.1.5-SNAPSHOT")
```

启动 Play:

```
$ activator
```

在play控制台输入 ‘ensime generate’。插件会生成一个 .ensime 文件到play项目的根目录下。

```shell
$ [MYPROJECT] ensime generate
[info] Gathering project information...
[info] Processing project: ProjectRef(file:/Users/aemon/projects/www/MYPROJECT/,MYPROJECT)...
[info]  Reading setting: name...
[info]  Reading setting: organization...
[info]  Reading setting: version...
[info]  Reading setting: scala-version...
[info]  Reading setting: module-name...
[info]  Evaluating task: project-dependencies...
[info]  Evaluating task: unmanaged-classpath...
[info]  Evaluating task: managed-classpath...
[info] Updating {file:/Users/aemon/projects/www/MYPROJECT/}MYPROJECT...
[info] Done updating.
[info]  Evaluating task: internal-dependency-classpath...
[info]  Evaluating task: unmanaged-classpath...
[info]  Evaluating task: managed-classpath...
[info]  Evaluating task: internal-dependency-classpath...
[info] Compiling 5 Scala sources and 1 Java source to /Users/aemon/projects/www/MYPROJECT/target/scala-2.9.1/classes...
[info]  Evaluating task: exported-products...
[info]  Evaluating task: unmanaged-classpath...
[info]  Evaluating task: managed-classpath...
[info]  Evaluating task: internal-dependency-classpath...
[info]  Evaluating task: exported-products...
[info]  Reading setting: source-directories...
[info]  Reading setting: source-directories...
[info]  Reading setting: class-directory...
[info]  Reading setting: class-directory...
[info]  Reading setting: ensime-config...
[info] Wrote configuration to .ensime
```

###启动 ENSIME
从Emacs, 执行 M-x ensime 和遵从屏幕上的说明。

就这样。现在你的Play项目应该有了类型检查, 自动完成，等等。注意, 如果你添加新的库依赖到你的play项目,你需要重新运行 “ensime generate” 和重启ENSIME。

###更多信息
检出 ENSIME README， 在 [https://github.com/ensime/ensime-emacs](https://github.com/ensime/ensime-emacs)。如果你有什么问题, 可以联系他们的ensime用户组， 在 [https://groups.google.com/forum/?fromgroups=#!forum/ensime](https://groups.google.com/forum/?fromgroups=#!forum/ensime)。

##所有需要的 Scala插件
Scala是一个比较新的编程语言, 所以功能是由插件提供，而不是IDE核心。

1. Eclipse Scala IDE: [http://scala-ide.org/](http://scala-ide.org/)
2. NetBeans Scala插件: [https://github.com/dcaoyuan/nbscala](https://github.com/dcaoyuan/nbscala)
3. IntelliJ IDEA Scala插件: [http://confluence.jetbrains.net/display/SCA/Scala+Plugin+for+IntelliJ+IDEA](http://confluence.jetbrains.net/display/SCA/Scala+Plugin+for+IntelliJ+IDEA)
4. IntelliJ IDEA的插件正在积极发展, 所以使用nightly构建可能会给你附加功能，只是多花一点成本。
5. Nika (11.x) 插件仓库: [https://www.jetbrains.com/idea/plugins/scala-nightly-nika.xml](https://www.jetbrains.com/idea/plugins/scala-nightly-nika.xml)
6. Leda (12.x) 插件仓库: [https://www.jetbrains.com/idea/plugins/scala-nightly-leda.xml](https://www.jetbrains.com/idea/plugins/scala-nightly-leda.xml)
7. IntelliJ IDEA Play 插件(仅在Leda 12.x可用): [http://plugins.intellij.net/plugin/?idea&pluginId=7080](http://plugins.intellij.net/plugin/?idea&pluginId=7080)
8. ENSIME - Emacs的Scala IDE模式: [https://github.com/aemoncannon/ensime](https://github.com/aemoncannon/ensime)(参阅里面的 ENSIME/Play使用说明)