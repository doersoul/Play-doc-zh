#HTTP 路由

##内置的HTTP路由
路由是一个负责将每个传入的HTTP请求转换到一个Action的组件。

通过MVC框架，一个HTTP请求被视为一个事件。这个事件包含两个主要的信息:

* 请求路径 (例如： `/clients/1542`, `/photos/list`), 包括查询字符串
* HTTP 方法 (例如： `GET`, `POST`, 等).

路由定义在`conf/routes` 文件中, 它会被编译。也即你会直接在浏览器中看到路由错误:

![](routesError.png)


##依赖注入
Play 支持生成二种类型的路由, 一种是依赖注入路由, 另一种是静态路由。默认为静态路由, 但是如果你使用Play seed Activator模板来创建一个新Play应用程序, 你的项目会在`build.sbt`中包括以下配置，告诉项目使用注入路由:

```scala
routesGenerator := InjectedRoutesGenerator
```

Play的文档中的代码示例假写上你使用的是注入路由生成器。如果你不使用这个, you can trivially adapt the code samples for the static routes generator, either by prefixing the controller invocation part of the route with an @ symbol, or by declaring each of your controllers as an `object` rather than a `class`.


##路由文件语法
`conf/routes` is the configuration file used by the router. This file lists all of the routes needed by the application. Each route consists of an HTTP method and URI pattern, both associated with a call to an `Action` generator.

让我们看看路由定义是什么样子的:

```
GET   /clients/:id          controllers.Clients.show(id: Long)
```

每个路由都以HTTP方法开头, 后面跟着URI模式。最后是方法调用的定义。

你还可以添加注释到路由文件, 用`#` 字符。

```
# Display a client.
GET   /clients/:id          controllers.Clients.show(id: Long)
```


##HTTP 方法
HTTP 方法可以是HTTP支持的任何有效方法 (`GET`, `POST`, `PUT`, `DELETE`, `HEAD`).


##URI 模式
URI 模式定义路由的请求路径。部分请求路径可以是动态的。

###静态路径
举例, 为精确匹配传入的`GET /clients/all` 请求, 你可以定义这个路由:

```
GET   /clients/all          controllers.Clients.list()
```

###动态路径
If you want to define a route that retrieves a client by ID, you’ll need to add a dynamic part:

```
GET   /clients/:id          controllers.Clients.show(id: Long)
```

> Note that a URI pattern may have more than one dynamic part.

The default matching strategy for a dynamic part is defined by the regular expression `[^/]+`, meaning that any dynamic part defined as `:id` will match exactly one URI part.

###匹配跨越几个`/` 符号的动态部分
If you want a dynamic part to capture more than one URI path segment, separated by forward slashes, you can define a dynamic part using the `*id` syntax, which uses the `.+` regular expression:

```
GET   /files/*name          controllers.Application.download(name)
```

Here for a request like GET `/files/images/logo.png`, the `name` dynamic part will capture the `images/logo.png` value.

###用自定义正则表达式匹配动态部分
你也可以为动态部分定义你自己的正则表达式, 使用`$id<regex>` 语法:

```
GET   /items/$id<[0-9]+>    controllers.Items.show(id: Long)
```


##调用Action生成器方法
The last part of a route definition is the call. This part must define a valid call to a method returning a `play.api.mvc.Action` value, which will typically be a controller action method.

If the method does not define any parameters, just give the fully-qualified method name:

```
GET   /                     controllers.Application.homePage()
```

If the action method defines some parameters, all these parameter values will be searched for in the request URI, either extracted from the URI path itself, or from the query string.

```
# Extract the page parameter from the path.
GET   /:page                controllers.Application.show(page)
```
或者:

```
# Extract the page parameter from the query string.
GET   /                     controllers.Application.show(page)
```

Here is the corresponding, `show` method definition in the `controllers.Application` controller:

```scala
def show(page: String) = Action {
  loadContentFromDatabase(page).map { htmlContent =>
    Ok(htmlContent).as("text/html")
  }.getOrElse(NotFound)
}
```

###参数类型
For parameters of type `String`, typing the parameter is optional. If you want Play to transform the incoming parameter into a specific Scala type, you can explicitly type the parameter:

```
GET   /clients/:id          controllers.Clients.show(id: Long)
```

And do the same on the corresponding `show` method definition in the `controllers.Clients` controller:

```scala
def show(id: Long) = Action {
  Client.findById(id).map { client =>
    Ok(views.html.Clients.display(client))
  }.getOrElse(NotFound)
}
```

###设定参数为固定值
Sometimes you’ll want to use a fixed value for a parameter:

```
# Extract the page parameter from the path, or fix the value for /
GET   /                     controllers.Application.show(page = "home")
GET   /:page                controllers.Application.show(page)
```

###设定参数默认值
You can also provide a default value that will be used if no value is found in the incoming request:

```
# Pagination links, like /clients?page=3
GET   /clients              controllers.Clients.list(page: Int ?= 1)
```

###可选参数
You can also specify an optional parameter that does not need to be present in all requests:

```
# The version parameter is optional. E.g. /api/list-all?version=3.0
GET   /api/list-all         controllers.Api.list(version: Option[String])
```


##路由优先级
Many routes can match the same request. If there is a conflict, the first route (in declaration order) is used.


##反向路由
The router can also be used to generate a URL from within a Scala call. This makes it possible to centralize all your URI patterns in a single configuration file, so you can be more confident when refactoring your application.

For each controller used in the routes file, the router will generate a ‘reverse controller’ in the `routes` package, having the same action methods, with the same signature, but returning a `play.api.mvc.Call` instead of a `play.api.mvc.Action`.

The `play.api.mvc.Call` defines an HTTP call, and provides both the HTTP method and the URI.

For example, if you create a controller like:

```scala
package controllers

import play.api._
import play.api.mvc._

class Application extends Controller {

  def hello(name: String) = Action {
    Ok("Hello " + name + "!")
  }

}
```

And if you map it in the `conf/routes` file:

```
# Hello action
GET   /hello/:name          controllers.Application.hello(name)
```

You can then reverse the URL to the `hello` action method, by using the `controllers.routes.Application` reverse controller:

```scala
// Redirect to /hello/Bob
def helloBob = Action {
  Redirect(routes.Application.hello("Bob"))
}
```