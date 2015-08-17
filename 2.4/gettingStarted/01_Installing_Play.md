#安装 Play

##准备工作
你需要先安装 JDK 1.8 或更高版本 (参阅 [一般安装任务](#General-Installation-Tasks))。

##快速入门

* 1.下载最新版 [Typesafe Activator](https://typesafe.com/get-started)。
* 2.将压缩包压缩到你有写权限的位置。
* 3.用终端命令 `cd activator*` 切换到这个目录 (或使用文件管理器)
* 4.用终端命令 `activator ui` 启动(或使用文件管理器)
* 5.访问 [http://localhost:8888](http://localhost:8888/)

你会发现很多应用程序列表和文档，可以让你快速开始。可以先简单试一下 play-java 样板。

###命令行
要想在你的文件系统的任何地方都可以直接使用 play 命令, 需要将activator所在目录添加到你的系统变量的Path中 (查阅 [一般安装任务](#General-Installation-Tasks)).

创建一个基于`play-java`的模板`my-first-app`，就这么简单：

```
activator new my-first-app play-java
cd my-first-app
activator run
```

然后访问 http://localhost:9000。

现在你已经准备好使用Play了!

##<a name="General-Installation-Tasks"></a>一般安装任务
你可能需要处理一些一般任务，以便安装Play!

###JDK 安装
请确认你的系统中有 JDK (Java Development Kit) 1.8 或更高版本。使用以下命令来进行验证：

```
java -version
javac -version
```

如果你还没有安装 JDK, 你需要安装它：

* 1.MacOS系统，Java 是内置的, 但你可能需要[升级到最新版本](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
* 2.Linux系统， 使用最新的Oracle JDK或OpenJDK(不要使用非gcj).
* 3.Windows系统，只需要下载和安装[最新版的JDK安装包](http://www.oracle.com/technetwork/java/javase/downloads/index.html)。

###将可执行文件添加到环境变量的Path中
为方便起见，你应该添加 Activator 安装目录到系统环境变量的`PATH`中。

在Unix类系统, 使用 `export PATH=/path/to/activator:$PATH`

在Windows系统, 添加 `;C:\path\to\activator` 到你的环境变量的 `PATH` 变量中。路径中不要使用空格。

###文件权限
####Unix
在发布版中运行 activator会写入一些文件到目录, 所以不要安装到 /opt, /usr/local 或任何其它你需要特殊定入权限的位置。

确保 activator 脚本是可执行的。如果不是, 运行终端命令 `chmod u+x /path/to/activator` 以添加权限。

###代理设置
如果你隐藏在一个代理后面，确保定义下面的设置：
在Windows上用`set HTTP_PROXY=http://<host>:<port>` ，
或者在UNIX类系统中使用 `export HTTP_PROXY=http://<host>:<port>` 。