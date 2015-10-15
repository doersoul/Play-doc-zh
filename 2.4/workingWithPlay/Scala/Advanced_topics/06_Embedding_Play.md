#Embedding a Play server in your application

While Play apps are most commonly used as their own container, you can also embed a Play server into your own existing application. This can be used in conjunction with the Twirl template compiler and Play routes compiler, but these are of course not necessary, a common use case for embedding a Play application will be because you only have a few very simple routes.

The simplest way to start an embedded Play server is to use the [`NettyServer`](https://www.playframework.com/documentation/2.4.x/api/scala/play/core/server/NettyServer$.html) factory methods. If all you need to do is provide some straightforward routes, you may decide to use the [String Interpolating Routing DSL](https://www.playframework.com/documentation/2.4.x/ScalaSirdRouter) in combination with the `fromRouter` method:

```scala
import play.core.server._
import play.api.routing.sird._
import play.api.mvc._

val server = NettyServer.fromRouter() {
  case GET(p"/hello/$to") => Action {
    Results.Ok(s"Hello $to")
  }
}
```

By default, this will start a server on port 9000 in prod mode. You can configure the server by passing in a [`ServerConfig`](https://www.playframework.com/documentation/2.4.x/api/scala/play/core/server/ServerConfig.html):

```scala
import play.core.server._
import play.api.routing.sird._
import play.api.mvc._

val server = NettyServer.fromRouter(ServerConfig(
  port = Some(19000),
  address = "127.0.0.1"
)) {
  case GET(p"/hello/$to") => Action {
    Results.Ok(s"Hello $to")
  }
}
```

You may want to customise some of the components that Play provides, for example, the HTTP error handler. A simple way of doing this is by using Play’s components traits, the [`NettyServerComponents`](https://www.playframework.com/documentation/2.4.x/api/scala/play/core/server/NettyServerComponents.html) trait is provided for this purpose, and can be conveniently combined with [`BuiltInComponents`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/BuiltInComponents.html) to build the application that it requires:

```scala
import play.core.server._
import play.api.routing.Router
import play.api.routing.sird._
import play.api.mvc._
import play.api.BuiltInComponents
import play.api.http.DefaultHttpErrorHandler
import scala.concurrent.Future

val components = new NettyServerComponents with BuiltInComponents {

  lazy val router = Router.from {
    case GET(p"/hello/$to") => Action {
      Results.Ok(s"Hello $to")
    }
  }

  override lazy val httpErrorHandler = new DefaultHttpErrorHandler(environment,
    configuration, sourceMapper, Some(router)) {

    override protected def onNotFound(request: RequestHeader, message: String) = {
      Future.successful(Results.NotFound("Nothing was found!"))
    }
  }
}
val server = components.server
```

In this case, the server configuration can be overridden by overriding the `serverConfig` property.

To stop the server once you’ve started it, simply call the `stop` method:

```
server.stop()
```

> **Note**: Play requires an application secret to be configured in order to start. This can be configured by providing an `application.conf` file in your application, or using the `play.crypto.secret` system property.