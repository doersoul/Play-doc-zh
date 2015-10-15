#HTTP Request Handlers

Play provides a range of abstractions for routing requests to actions, providing routers and filters to allow most common needs. Sometimes however an application will have more advanced needs that aren’t met by Play’s abstractions. When this is the case, applications can provide custom implementations of Play’s lowest level HTTP pipeline API, the [`HttpRequestHandler`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/http/HttpRequestHandler.html).

Providing a custom `HttpRequestHandler` should be a last course of action. Most custom needs can be met through implementing a custom router or a [filter](https://www.playframework.com/documentation/2.4.x/ScalaHttpFilters).


##Implementing a custom request handler
The `HttpRequestHandler` trait has one method to be implemented, `handlerForRequest`. This takes the request to get a handler for, and returns a tuple of a `RequestHeader` and a `Handler`.

The reason why a request header is returned is so that information can be added to the request, for example, routing information. In this way, the router is able to tag requests with routing information, such as which route matched the request, which can be useful for monitoring or even for injecting cross cutting functionality.

A very simple request handler that simply delegates to a router might look like this:

```scala
import javax.inject.Inject
import play.api.http._
import play.api.mvc._
import play.api.routing.Router

class SimpleHttpRequestHandler @Inject() (router: Router) extends HttpRequestHandler {
  def handlerForRequest(request: RequestHeader) = {
    router.routes.lift(request) match {
      case Some(handler) => (request, handler)
      case None => (request, Action(Results.NotFound))
    }
  }
}
```


##Extending the default request handler
In most cases you probably won’t want to create a new request handler from scratch, you’ll want to build on the default one. This can be done by extending [DefaultHttpRequestHandler](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/http/HttpRequestHandler.html). The default request handler provides a number of methods that can be overridden, this allows you to implement your custom functionality without reimplementing the code to tag requests, handle errors, etc.

One use case for a custom request handler may be that you want to delegate to a different router, depending on what host the request is for. Here is an example of how this might be done:

```scala
import javax.inject.Inject
import play.api.http._
import play.api.mvc.RequestHeader

class VirtualHostRequestHandler @Inject() (errorHandler: HttpErrorHandler,
    configuration: HttpConfiguration, filters: HttpFilters,
    fooRouter: foo.Routes, barRouter: bar.Routes
  ) extends DefaultHttpRequestHandler(
    fooRouter, errorHandler, configuration, filters
  ) {

  override def routeRequest(request: RequestHeader) = {
    request.host match {
      case "foo.example.com" => fooRouter.routes.lift(request)
      case "bar.example.com" => barRouter.routes.lift(request)
      case _ => super.routeRequest(request)
    }
  }
}
```


##Configuring the http request handler
A custom http handler can be supplied by creating a class in the root package called `RequestHandler` that implements `HttpRequestHandler`.

If you don’t want to place your request handler in the root package, or if you want to be able to configure different request handlers for different environments, you can do this by configuring the `play.http.requestHandler` configuration property in `application.conf`:

```scala
play.http.requestHandler = "com.example.RequestHandler"
```

###Performance notes
The http request handler that Play uses if none is configured is one that delegates to the legacy `GlobalSettings` methods. This may have a performance impact as it will mean your application has to do many lookups out of Guice to handle a single request. If you are not using a `Global` object, then you don’t need this, instead you can configure Play to use the default http request handler:

```scala
play.http.requestHandler = "play.api.http.DefaultHttpRequestHandler"
```