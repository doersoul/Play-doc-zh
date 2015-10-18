#生产配置

你在生产模式可以有许多不同类型的配置。有三种主要类型:

* [一般配置](#General-configuration)
* [日志配置](#Logging-configuration)
* [JVM配置](#JVM-configuration)

每个类型都具有不同的方法来配置它们。


##<a name="General-configuration"></a>一般配置
Play有很多配置选项。你可以配置数据库连接URLs, 应用程序secret, HTTP端口, SSL配置, 如此等等。

多数Play的配置是定义在多种`.conf` 文件中, 使用[HOCON格式](https://github.com/typesafehub/config/blob/master/HOCON.md)。主配置文件是使用`application.conf` 文件。你可以在项目内的`conf/application.conf` 找到这个文件。`application.conf` 文件在运行从类路径中加载(或者你也可以重写它从哪里加载)。每个项目只能有一个`application.conf`。

其它`.conf` 也是一样加载。库定义的默认设置在`reference.conf` 文件中。这些文件保存在库的JARs中 — 每个JAR只有一个`reference.conf` — 并且在运行时聚合在一起。`reference.conf` 文件提供默认设置; 他们可以通过在`application.conf` 文件中定义的设置重写。

Play的配置也可以通过使用系统属性和环境变量来定义。当设置环境的变化时，这可以方便; 当你的应用程序运行在不同的环境时，你可以使用`application.conf` 作为公共设置, 但使用系统属性和环境变量来改变设置。

系统属性重写在`application.conf` 中的设置, 并且`application.conf` 重写在各种`reference.conf` 文件中的默认设置。

你有几种方式可以在运行时重写配置。当在多变的环境中时这很方便；你可以为每个环境动态更改配置。这里是在运行时配置的几种选择:

* 使用一个替换的`application.conf` 文件。
* 使用系统属性重写单独的设置。
* 使用环境变量注入配置值。

###指定一个替换的配置文件
默认情况下是从类路径加载`application.conf`文件。如果需要，你可以指定一个替换的配置文件:

####使用 `-Dconfig.resource`
这会在应用程序类路径搜索一个替换配置文件(通常是在打包前把这些替换配置文件放到应用程序`conf/` 目录)。Play会查找`conf/` ，所以你不需要添加`conf/`。

```shell
$ /path/to/bin/<project-name> -Dconfig.resource=prod.conf
```

####使用 `-Dconfig.file`
你也可以指定另一个本机其它位置的配置文件，不打包到你的应用程序中:

```shell
$ /path/to/bin/<project-name> -Dconfig.file=/opt/conf/prod.conf
```

> 注意你总是可以在新的`prod.conf` 文件引用原始配置文件，使用`include` 指令, 如:
>
```
include "application.conf"
>
key.to.override=blah
```

###使用系统属性来重写配置
有时候您不想指定一个完整的配置文件, 但只覆盖一堆的特定键。你可以通过指定他们作为Java系统属性:

```shell
$ /path/to/bin/<project-name> -Dplay.crypto.secret=abcdefghijk -Ddb.default.password=toto
```

####使用系统属性指定HTTP服务器地址和端口
你可以使用系统属性很容易的提供HTTP端口和地址。默认是在`0.0.0.0`地址的`9000` 端口(所有地址)。

```shell
$ /path/to/bin/<project-name> -Dhttp.port=1234 -Dhttp.address=127.0.0.1
```

####更改RUNNING_PID的路径
可以更改包含启动应用程序的进程id的文件路径。通常这个文件是放在你的play项目的根目录下, 但是建议你把它放在重新启动时它会自动清除的某个地方，如`/var/run`:

```shell
$ /path/to/bin/<project-name> -Dpidfile.path=/var/run/play.pid
```

> 要确保这个目录是存在的，并且运行Play应用程序的用户有这个目录的写入权限。

使用这个文件, 你可以使用`kill` 命令停止你的应用程序, 例如:

```shell
$ kill $(cat /var/run/play.pid)
```

###使用环境变量
你可以从`application.conf` 文件引用环境变量:

```scala
my.key = defaultvalue
my.key = ${?MY_KEY_ENV}
```

这里, 如果没有`MY_KEY_ENV`这个环境变量的值，`my.key = ${?MY_KEY_ENV}` 字段就会忽略, 但如果你设置了环境变量`MY_KEY_ENV` , 它就会被使用。

###服务器配置选项
服务器配置选项的完整列表, 包括默认的, 就像下面看到的:

```scala
play {
  server {

    # The root directory for the Play server instance. This value can
    # be set by providing a path as the first argument to the Play server
    # launcher script. See `ServerConfig.loadConfiguration`.
    dir = ${?user.dir}

    # HTTP configuration
    http {
      # The HTTP port of the server. Use a value of "disabled" if the server
      # shouldn't bind an HTTP port.
      port = 9000
      port = ${?http.port}

      # The interface address to bind to.
      address = "0.0.0.0"
      address = ${?http.address}
    }

    # HTTPS configuration
    https {

      # The HTTPS port of the server.
      port = ${?https.port}

      # The interface address to bind to
      address = "0.0.0.0"
      address = ${?https.address}

      # The SSL engine provider
      engineProvider = "play.core.server.ssl.DefaultSSLEngineProvider"
      engineProvider = ${?play.http.sslengineprovider}

      # HTTPS keystore configuration, used by the default SSL engine provider
      keyStore {
        # The path to the keystore
        path = ${?https.keyStore}

        # The type of the keystore
        type = "JKS"
        type = ${?https.keyStoreType}

        # The password for the keystore
        password = ""
        password = ${?https.keyStorePassword}

        # The algorithm to use. If not set, uses the platform default algorithm.
        algorithm = ${?https.keyStoreAlgorithm}
      }

      # HTTPS truststore configuration
      trustStore {

        # If true, does not do CA verification on client side certificates
        noCaVerification = false
      }
    }

    # The type of ServerProvider that should be used to create the server.
    # If not provided, the ServerStart class that instantiates the server
    # will provide a default value.
    provider = ${?server.provider}

    # The path to the process id file created by the server when it runs.
    # If set to "/dev/null" then no pid file will be created.
    pidfile.path = ${play.server.dir}/RUNNING_PID
    pidfile.path = ${?pidfile.path}

    # Configuration options specific to Netty
    netty {
      # The maximum length of the initial line. This effectively restricts the maximum length of a URL that the server will
      # accept, the initial line consists of the method (3-7 characters), the URL, and the HTTP version (8 characters),
      # including typical whitespace, the maximum URL length will be this number - 18.
      maxInitialLineLength = 4096
      maxInitialLineLength = ${?http.netty.maxInitialLineLength}

      # The maximum length of the HTTP headers. The most common effect of this is a restriction in cookie length, including
      # number of cookies and size of cookie values.
      maxHeaderSize = 8192
      maxHeaderSize = ${?http.netty.maxHeaderSize}

      # The maximum length of body bytes that Netty will read into memory at a time.
      # This is used in many ways.  Note that this setting has no relation to HTTP chunked transfer encoding - Netty will
      # read "chunks", that is, byte buffers worth of content at a time and pass it to Play, regardless of whether the body
      # is using HTTP chunked transfer encoding.  A single HTTP chunk could span multiple Netty chunks if it exceeds this.
      # A body that is not HTTP chunked will span multiple Netty chunks if it exceeds this or if no content length is
      # specified. This only controls the maximum length of the Netty chunk byte buffers.
      maxChunkSize = 8192
      maxChunkSize = ${?http.netty.maxChunkSize}

      # Whether the Netty wire should be logged
      log.wire = false
      log.wire = ${?http.netty.log.wire}

      # Netty options. Possible keys here are defined by:
      #
      # http://netty.io/3.9/api/org/jboss/netty/channel/socket/SocketChannelConfig.html
      # http://netty.io/3.9/api/org/jboss/netty/channel/socket/ServerSocketChannelConfig.html
      # http://netty.io/3.9/api/org/jboss/netty/channel/socket/nio/NioSocketChannelConfig.html
      #
      # Options that pertain to the listening server socket are defined at the top level, options for the sockets associated
      # with received client connections are prefixed with child.*
      option {

        # Set whether connections should use TCP keep alive
        # child.keepAlive = false

        # Set whether the TCP no delay flag is set
        # child.tcpNoDelay = false

        # Set the size of the backlog of TCP connections.  The default and exact meaning of this parameter is JDK specific.
        # backlog = 100
      }
    }
  }

  # Configuration specific to Play's experimental Akka HTTP backend
  akka {
    # How long to wait when binding to the listening socket
    http-bind-timeout = 5 seconds
  }
}
```


##<a name="Logging-configuration"></a>日志配置
可以通过创建一个logback配置文件来配置日志。您的应用程序通过以下方法使用日志：

###绑定一个自定义logback配置文件到你的应用程序
创建一个叫`logback.xml`的替换logback配置文件，并拷贝到`<app>/conf`

你也可以通过系统属性指定另一个logback配置文件。请注意如果配置文件没有指定，那么play 会使用默认`logback.xml` ，它在play生产模式自带。这意味着在`application.conf` 文件中的任何日志级别设置会被重写。一个好的方式是总是指定你的`logback.xml`。

###使用 `-Dlogger.resource`
指定另一个logback配置文件以从类路径中加载:

```shell
$ /path/to/bin/<project-name> -Dlogger.resource=conf/prod-logger.xml
```

###使用 `-Dlogger.file`
指定另一个logback配置文件以从文件系统加载:

```shell
$ /path/to/bin/<project-name> -Dlogger.file=/opt/prod/prod-logger.xml
```

###使用 `-Dlogger.url`
指定另一个logback配置文件以从一个URL加载:

```shell
$ /path/to/bin/<project-name> -Dlogger.url=http://conf.mycompany.com/logger.xml
```


##<a name="JVM-configuration"></a>JVM 配置
你可以指定任何JVM参数到应用程序启动脚本，否则就使用默认JVM设置:

```shell
$ /path/to/bin/<project-name> -J-Xms128M -J-Xmx512m -J-server
```

为方便起见，你也可以一次性设置内存min, max, permgen 和保留代码缓存大小; 有一个计算公式用来确定这些值给定的支持参数(代表最大内存):

```shell
$ /path/to/bin/<project-name> -mem 512 -J-server
```