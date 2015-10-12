#日志 API

在你的应用程序中使用日志有助于监控、调试、错误跟踪以及商业智能分析。Play 提供一个日志API，可通过[`Logger`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Logger$.html)来访问，并且使用[`Logback`](http://logback.qos.ch/) 作为日志引擎。


##日志架构
日志API使用一个组件集来帮助你实现高效的日志策略。

####Logger
你的应用程序可以定义[`Logger`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Logger.html) 实例来发送日志消息请求。每个`Logger` 有一个名字，它会出现在日志消息中，该名字用于配置中。

Loggers 根据他们的名字遵循一个层级继承结构。如果一个 logger的名字后面加个点作为另一个logger名字的前缀，那么可以说前一个logger是后一个logger的祖先。举例, 一个名称为 “com.foo”的logger是名称为“com.foo.bar.Baz.”的logger的祖先。所有loggers都继承自根logger。Logger 继承允许你通过配置共同的祖先来配置一个logger集。

Play 应用程序提供一个默认的logger，叫 “application”，或者你可以创建你自己的loggers。Play库使用一个logger叫 “play”, 以及一些第三方库会有他们自己的logger名。

####日志级别
日志级别是用来区分日志消息的严重性。当你写日志请求声明时，你要指定严重程序，它会出现在生成的日志消息里。

以下是可用的日志级别的集合, 以严重性的降序列出。

* `OFF` - 用于关闭日志, 不作为消息分类。
* `ERROR` - 运行时错误, 或不可预料的情况。
* `WARN` - 用于一些废弃的 APIs, 极少用的 API, ‘almost’ 错误, 其它不可预料但又算不上错误的运行时情况。
* `INFO` - 感兴趣的运行时事件，如应用程序启动和关闭时。
* `DEBUG` - 系统工作流程上的一些细节信息。
* `TRACE` - 多数细节信息。

为附加消息分类, 日志级别用来配置loggers和appenders上的严重程度阈值。例如, 一个logger的日志级别设置为`INFO`，则会记录任何级别为`INFO`或更高(`INFO`, `WARN`, `ERROR`)的请求，并忽略低严重性的日志 (`DEBUG`, `TRACE`)请求。使用`OFF` 会忽略所有日志请求。

####Appenders
日志 API 允许日志请求打印到一个或多个叫“appenders.”的输出目标。Appenders 在配置中指定，可选的有控制台, 文件, 数据库或其它输出。

Appenders和loggers组合可以帮助你路由和过滤日志消息。举例, 你可以使用一个appender对有用的数据进行分析，用另一个appender记录错误，用于运维团队的监控。

> 注意: 要了解更多关于日志架构的信息，查阅 [Logback 文档](http://logback.qos.ch/manual/architecture.html)。


##使用 Loggers
首先导入`Logger` 类及其伴生对象:

```scala
import play.api.Logger
```

####默认 Logger
`Logger` 对象是默认的logger，名称为“application.”。你可以使用它打印日志:

```scala
// Log some debug info
Logger.debug("Attempting risky calculation.")

try {
  val result = riskyCalculation
  
  // Log result if successful
  Logger.debug(s"Result=$result")
} catch {
  case t: Throwable => {
    // Log error with message and Throwable.
    Logger.error("Exception with riskyCalculation", t)
  }
}
```

使用Play的默认日志配置, 上面的语句会产生类似下面的控制台输出:

```shell
[debug] application - Attempting risky calculation.
[error] application - Exception with riskyCalculation
java.lang.ArithmeticException: / by zero
    at controllers.Application$.controllers$Application$$riskyCalculation(Application.scala:32) ~[classes/:na]
    at controllers.Application$$anonfun$test$1.apply(Application.scala:18) [classes/:na]
    at controllers.Application$$anonfun$test$1.apply(Application.scala:12) [classes/:na]
    at play.api.mvc.ActionBuilder$$anonfun$apply$17.apply(Action.scala:390) [play_2.10-2.3-M1.jar:2.3-M1]
    at play.api.mvc.ActionBuilder$$anonfun$apply$17.apply(Action.scala:390) [play_2.10-2.3-M1.jar:2.3-M1]
```

注意消息包含了日志级别, logger名称, 消息内容, 并且如果抛出异常的话还会有堆栈跟踪信息。

####创建你自己的 loggers
虽然可能在所有地方都使用默认logger有吸引力，但这通常是一种不好的设计实践。用不同的名字创建你自己的loggers，会让配置更灵活, 可以过滤你的日志输出, 并且精确知道日志消息的来源。

你可以使用`Logger.apply` 工厂方法创建一个新logger，它需要一个名字为参数:

```scala
val accessLogger: Logger = Logger("access")
```

对于日志记录应用程序事件的一个常用的策略是，为每个类配置一个不同的logger，并用类名来命名它。日志API支持这个，用一个接受类作为参数的工厂方法:

```scala
val logger: Logger = Logger(this.getClass())
```

####日志模式
高效使用日志可以帮助你达到许多目标:

```scala
import scala.concurrent.Future
import play.api.Logger
import play.api.mvc._

trait AccessLogging {
  
  val accessLogger = Logger("access")

  object AccessLoggingAction extends ActionBuilder[Request] {

    def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
      accessLogger.info(s"method=${request.method} uri=${request.uri} remote-address=${request.remoteAddress}")
      block(request)
    }
  }
}

object Application extends Controller with AccessLogging {
  
  val logger = Logger(this.getClass())
  
  def index = AccessLoggingAction {
    try {
      val result = riskyCalculation
      Ok(s"Result=$result")
    } catch {
      case t: Throwable => {
        logger.error("Exception with riskyCalculation", t)
        InternalServerError("Error in calculation: " + t.getMessage())
      }
    }
  }
}
```

这个示例使用 [action composition](https://playframework.com/documentation/2.4.x/ScalaActionsComposition) 来定义一个`AccessLoggingAction` ，它会打印日志到一个名为“access.”的logger中。 `Application` controller 使用这个 action，并且它也使用自己的logger (在它的类后命名) 来记录应用程序中的事件。在配置中你可以路由这些loggers到不同的appenders, 如一个访问日志和应用日志。

如果你只想为指定的action打印请求数据，上面的设计就可以很好的工作了。要想打印所有的请求日志，最好使用[过滤器](https://playframework.com/documentation/2.4.x/ScalaHttpFilters):

```scala
import scala.concurrent.ExecutionContext.Implicits.global
import scala.concurrent.Future
import play.api.Logger
import play.api.mvc._
import play.api._

object AccessLoggingFilter extends Filter {
  
  val accessLogger = Logger("access")
  
  def apply(next: (RequestHeader) => Future[Result])(request: RequestHeader): Future[Result] = {
    val resultFuture = next(request)
    
    resultFuture.foreach(result => {
      val msg = s"method=${request.method} uri=${request.uri} remote-address=${request.remoteAddress}" +
        s" status=${result.header.status}";
      accessLogger.info(msg)
    })
    
    resultFuture
  }
}

object Global extends WithFilters(AccessLoggingFilter) {
  
  override def onStart(app: Application) {
    Logger.info("Application has started")
  }

  override def onStop(app: Application) {
    Logger.info("Application has stopped")
  }
}
```

在上面这个过滤器版本的代码中，我们通过打印日志添加了响应状态到日志请求中，当`Future[Result]` 完成时会打印输出。


##配置
查阅 [日志配置](https://playframework.com/documentation/2.4.x/SettingsLogger) 了解更多配置信息。