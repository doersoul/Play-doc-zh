#Action 组合

本章介绍几种定义通用action的方法


##自定义action构造器
我们见过[之前](01_Actions_Controllers_and_Results.md)有多种声明一个action的方式 —— 带请求参数的,不带请求参数的,有请求体解析器的，等等。实际上还有更多的方法, 我们会在[异步编程](../02_Asynchronous_HTTP_programming/01_Asynchronous_results.md)这一章中看到。

这些构造action的方法事实上都是由一个叫[`ActionBuilder`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/ActionBuilder.html)的特质定义的，而我们用来声明我们的action的[`Action`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Action$.html)对象只不过是这个特质的一个实例。通过实现自己的`ActionBuilder`, 你可以声明一些可重用的action栈,以用来构建action。

让我们以一个简单的日志装饰器例子开始, 我们想记录每一次对这个action的调用。

第一种方式是在`invokeBlock` 方法中实现该功能, 每个由`ActionBuilder` 构建的action都会调用该方法:

```scala
import play.api.mvc._

object LoggingAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    Logger.info("Calling action")
    block(request)
  }
}
```

现在我们就可以像使用`Action` 一样来使用它了:

```scala
def index = LoggingAction {
  Ok("Hello World")
}
```

因为`ActionBuilder` 提供了所有构建actions的不同方式, 这同样适用于例如声明一个自定义的请求体解析器:

```scala
def submit = LoggingAction(parse.text) { request =>
  Ok("Got a body " + request.body.length + " bytes long")
}
```

###组合 actions
在多数应用程序中, 我们会想要多个action构造器, 有些用来做各种类型的身份验证, 有些提供各种类型的通用功能, 等等。这种情况下, 我们不想为每个类型的action构造器都重写日志action的代码, 就要定义一种可重用的方式。

可重用的action代码可以通过包装actions来实现:

```scala
import play.api.mvc._

case class Logging[A](action: Action[A]) extends Action[A] {

  def apply(request: Request[A]): Future[Result] = {
    Logger.info("Calling action")
    action(request)
  }

  lazy val parser = action.parser
}
```

我们也可以使用`Action` action构造器来构建actions，这样就不有定义我们自己的action类了:

```scala
import play.api.mvc._

def logging[A](action: Action[A])= Action.async(action.parser) { request =>
  Logger.info("Calling action")
  action(request)
}
```

Actions可以使用`composeAction` 方法混入action构造器中:

```scala
object LoggingAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    block(request)
  }
  override def composeAction[A](action: Action[A]) = new Logging(action)
}
```

现在构造器能够像之前那样使用了:

```scala
def index = LoggingAction {
  Ok("Hello World")
}
```

我们也可以不用action构造器来混入包装的actions:

```scala
def index = Logging {
  Action {
    Ok("Hello World")
  }
}
```

###更多复杂的actions
到目前为止我们演示的actions都不影响传入的请求。当然, 我们也可以读取和修改传入的请求对象:

```scala
import play.api.mvc._

def xForwardedFor[A](action: Action[A]) = Action.async(action.parser) { request =>
  val newRequest = request.headers.get("X-Forwarded-For").map { xff =>
    new WrappedRequest[A](request) {
      override def remoteAddress = xff
    }
  } getOrElse request
  action(newRequest)
}
```

> 注意: Play 已经内置了对`X-Forwarded-For` 标头的支持。

我们可以阻塞一个请求:

```scala
import play.api.mvc._

def onlyHttps[A](action: Action[A]) = Action.async(action.parser) { request =>
  request.headers.get("X-Forwarded-Proto").collect {
    case "https" => action(request)
  } getOrElse {
    Future.successful(Forbidden("Only HTTPS requests allowed"))
  }
}
```

最后，我们还可以修改返回的结果:

```scala
import play.api.mvc._
import play.api.libs.concurrent.Execution.Implicits._

def addUaHeader[A](action: Action[A]) = Action.async(action.parser) { request =>
  action(request).map(_.withHeaders("X-UA-Compatible" -> "Chrome=1"))
}
```


##不同的请求类型
当action组合允许你在HTTP请求和响应层级进行一些额外的操作时, 你往往会想到构建数据转换的管道，为请求添加上下文或执行一些验证。`ActionFunction` 可以被认为是一个在请求上的函数, 该函数参数化了输入的请求类型和输出类型，并将输出类型传至下一层。每个action函数可以是一个模块化的处理，如身份验证,数据库查找对象, 权限检查,或其它你想要在action中组合并重用的操作。

这里还有一些预定义的特质，它们实现了`ActionFunction`，这对不同类型的处理都非常有用:

* [`ActionTransformer`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/ActionTransformer.html) 可以更改请求, 例如添加额外信息。
* [`ActionFilter`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/ActionFilter.html) 可以选择性拦截请求,例如无须改变请求值就可以处理错误。.
* [`ActionRefiner`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/ActionRefiner.html) 是以上两种的通用用例。
* [`ActionBuilder`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/ActionBuilder.html) 是一种特殊用例，带Request参数作为输入, 从而可以构建actions。

你还可以通过实现`invokeBlock` 方法随意定义你自己的`ActionFunction` 。通常为了方便会让输入和输出类型为`Request` (使用 `WrappedRequest`), 但这不是必须的。

###身份验证
Action函数最常见的用例之一就是身份验证。我们可以轻松实现我们自己的身份验证action变换器，从原始请求中获取用户信息，并添加到一个新的`UserRequest`。要注意这同样也是一个`ActionBuilder` ，因为它带有一个简单的`Request` 作为输入:

```scala
import play.api.mvc._

class UserRequest[A](val username: Option[String], request: Request[A]) extends WrappedRequest[A](request)

object UserAction extends
    ActionBuilder[UserRequest] with ActionTransformer[Request, UserRequest] {
  def transform[A](request: Request[A]) = Future.successful {
    new UserRequest(request.session.get("username"), request)
  }
}
```

Play 也提供了内置的身份验证action构造器。更多信息和如何使用它请参阅[这里](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Security$$AuthenticatedBuilder$.html)。

> **注意**: 内置的身份验证action构造器只是一个轻便的助手，为用尽可能少的代码为简单的应用实现身份验证功能,其实现和上面的例子非常相似。

> 如果你相比内置的身份验证action，有更多复杂的需求，推荐实现你自己的身份验证action。

###添加信息到请求
现在让我们考虑这样一个REST API，处理`Item`类型的对象。在`/item/:itemId` 路径下可能有多个路由, 并且每个都要查找`item`。在这种情况下，将逻辑放在action函数中很有用。

首先, 我们创建一个请求对象，添加一个`Item` 到我们的`UserRequest`:

```scala
import play.api.mvc._

class ItemRequest[A](val item: Item, request: UserRequest[A]) extends WrappedRequest[A](request) {
  def username = request.username
}
```

现在我们创建一个action精炼器，查找该item并返回`Either` 一个错误(`Left`) 或一个新的`ItemRequest` (`Right`)。注意这个action精炼器是定义在一个方法中，用来获取item的id:

```scala
def ItemAction(itemId: String) = new ActionRefiner[UserRequest, ItemRequest] {
  def refine[A](input: UserRequest[A]) = Future.successful {
    ItemDao.findById(itemId)
      .map(new ItemRequest(_, input))
      .toRight(NotFound)
  }
}
```

###验证请求
最后, 我们想要一个action函数用来验证是否继续处理一个请求。举例,也许我们想要检查从`UserAction` 中得到的用户，是否有权限访问从`ItemAction` 得到的item, 否则返回一个错误:

```scala
object PermissionCheckAction extends ActionFilter[ItemRequest] {
  def filter[A](input: ItemRequest[A]) = Future.successful {
    if (!input.item.accessibleByUser(input.username))
      Some(Forbidden)
    else
      None
  }
}
```

###把他们合并在一起
现在我们将这些action函数链接在一起(从`ActionBuilder` 开始)，使用`andThen` 来创建一个action:

```scala
def tagItem(itemId: String, tag: String) =
  (UserAction andThen ItemAction(itemId) andThen PermissionCheckAction) { request =>
    request.item.addTag(tag)
    Ok("User " + request.username + " tagged " + request.item.id)
  }
```

Play 也提供一个[全局过滤器API](../../Advanced_topics/02_HTTP_architecture/02_HTTP_filters.md) , 这对全局横切关注点非常有用。