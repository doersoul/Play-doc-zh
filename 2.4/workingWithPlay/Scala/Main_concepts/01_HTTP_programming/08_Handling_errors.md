#处理错误

一个HTTP应用程序返回的错误类型主要有二种 —— 客户端错误和服务端错误。客户端错误表示连接客户端发生了一些错误，服务端错误表示服务器方面发生了一些错误。

Play在许多情况下会自动检测客户端错误 —— 这些错误包括如标头值格式不正确, 无法理解的content types,和请求的资源不存在。Play在许多情况下也会自动处理服务端错误 —— 如果你的action代码抛出一个异常, Play会捕获它，并生成一个服务端错误页面，将其发送到客户端。

Play处理这些错误是通过[`HttpErrorHandler`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/http/HttpErrorHandler.html)接口。它定义二个方法, `onClientError`, 和`onServerError`。


##提供一个自定义错误处理程序
通过在根包中创建一个叫`ErrorHandler`的类，可以提供一个自定义错误处理程序，它实现了[`HttpErrorHandler`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/http/HttpErrorHandler.html), 举例:

```scala
import play.api.http.HttpErrorHandler
import play.api.mvc._
import play.api.mvc.Results._
import scala.concurrent._

class ErrorHandler extends HttpErrorHandler {

  def onClientError(request: RequestHeader, statusCode: Int, message: String) = {
    Future.successful(
      Status(statusCode)("A client error occurred: " + message)
    )
  }

  def onServerError(request: RequestHeader, exception: Throwable) = {
    Future.successful(
      InternalServerError("A server error occurred: " + exception.getMessage)
    )
  }
}
```

如果你不想把你的错误处理程序放在根包, 或如果你想为不同的环境配置不同的错误处理程序, 你可以通过在`application.conf`文件中配置`play.http.errorHandler` 配置属性来做这个:

```scala
play.http.errorHandler = "com.example.ErrorHandler"
```


##扩展默认错误处理程序
Play的默认错误处理程序提供许多有用的开箱即用的功能。 举例, 在开发模式, 当服务器发生错误, Play会试图确定错误位置并呈现你的应用程序中引起异常的那段代码, 因此你能快速了解和识别问题。你可能希望在生产模式提供自定义服务端错误处理程序, 同时在开发中仍可以维持这个功能。为便于实现这个, Play 提供一个[`DefaultHttpErrorHandler`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/http/DefaultHttpErrorHandler.html) ，它有一些很方便的方法，让你可以重写，以便你可以在Play已存在的行为中混入你自定义的逻辑。

举例, 要只是在生产模式中提供几个自定义的错误消息, 保留开发模式的错误消息不动, 并且你也想提供一个特定的禁止错误页面:

```scala
import javax.inject._

import play.api.http.DefaultHttpErrorHandler
import play.api._
import play.api.mvc._
import play.api.mvc.Results._
import play.api.routing.Router
import scala.concurrent._

class ErrorHandler @Inject() (
    env: Environment,
    config: Configuration,
    sourceMapper: OptionalSourceMapper,
    router: Provider[Router]
  ) extends DefaultHttpErrorHandler(env, config, sourceMapper, router) {

  override def onProdServerError(request: RequestHeader, exception: UsefulException) = {
    Future.successful(
      InternalServerError("A server error occurred: " + exception.getMessage)
    )
  }

  override def onForbidden(request: RequestHeader, message: String) = {
    Future.successful(
      Forbidden("You're not allowed to access this resource.")
    )
  }
}
```

参阅API文档中的[DefaultHttpErrorHandler](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/http/DefaultHttpErrorHandler.html)，以了解有哪些方法可以重写和如何利用他们。