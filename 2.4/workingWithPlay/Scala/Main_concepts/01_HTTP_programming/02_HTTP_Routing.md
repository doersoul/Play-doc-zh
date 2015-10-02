#HTTP 路由

##内置的HTTP路由
路由是一个负责将每个传入的HTTP请求转换到一个Action的组件。

通过MVC框架，一个HTTP请求被视为一个事件。这个事件包含两个主要的信息:

* 请求路径 (例如： `/clients/1542`, `/photos/list`), 包括查询字符串
* HTTP 方法 (例如： `GET`, `POST`, 等).

路由定义在`conf/routes` 文件中, 它会被编译。也即你会直接在浏览器中看到路由错误:

![""](routesError.png)


##依赖注入
Play 支持生成二种类型的路由, 一种是依赖注入路由, 另一种是静态路由。默认为静态路由, 但是如果你使用Play seed Activator模板来创建一个新Play应用程序, 你的项目会在`build.sbt`中包括以下配置，告诉项目使用注入路由:

```scala
routesGenerator := InjectedRoutesGenerator
```

Play的文档中的代码例子假定你使用的是注入路由生成器。如果你不使用这个, 你可以慢慢全面适应静态路由生成器的代码示例, 通过在路由定义尾部controller调用那一部分的前面添加一个@符号做为前缀, 或者每个controllers声明为`object` 而不是`class`。


##路由文件语法
路由的配置使用`conf/routes` 文件。这个文件列出应用程序中需要的所有路由。每个路由都包含了一个HTTP方法和 URI模式, 两者关联到一个`Action` 生成器调用。

让我们看看路由定义是什么样子的:

```
GET   /clients/:id          controllers.Clients.show(id: Long)
```

每个路由都以HTTP方法开头, 后面跟着URI模式。最后是方法调用的定义。

你还可以添加注释到路由文件, 用`#` 字符。

```
# 显示一个 client
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
如果你想定义一个路由，它根据ID检索 client, 则需要添加一个动态部分:

```
GET   /clients/:id          controllers.Clients.show(id: Long)
```

> 注意：一个URI模式可以有多个动态部分。

动态部分的匹配策略是通过正则表达式`[^/]+` 定义的, 这意味着任何动态部分定义为`:id` 只会匹配一个URI部分。

###匹配跨越几个`/` 符号的动态部分
如果你想一个动态部分匹配多个由斜杠分隔开的URI路径段, 你可以使用`*id` 语法来定义, 它使用`.+` 正则表达式:

```
GET   /files/*name          controllers.Application.download(name)
```

像 GET `/files/images/logo.png` 这样的请求, 动态部分 `name` 匹配的是`images/logo.png` 值。

###用自定义正则表达式匹配动态部分
你也可以为动态部分定义你自己的正则表达式, 使用`$id<regex>` 语法:

```
GET   /items/$id<[0-9]+>    controllers.Items.show(id: Long)
```


##调用Action生成器方法
路由定义的最后一部分是方法调用。这个部分必须定义一个返回 `play.api.mvc.Action` 值的有效方法, 通常是一个 controller action 方法。

如果该方法不带任何参数, 则只需给出完整方法名:

```
GET   /                     controllers.Application.homePage()
```

如果action方法定义了一些参数, 所有这些参数则会在请求URI中查找, 不管是从URI路径自身提取，还是从查询字符串中查找。

```
# 从路径中提取 page 参数
GET   /:page                controllers.Application.show(page)
```
或者:

```
# 从查询字符串中提取 page 参数
GET   /                     controllers.Application.show(page)
```

这里是对应的, `show` 方法定义在`controllers.Application` controller中:

```scala
def show(page: String) = Action {
  loadContentFromDatabase(page).map { htmlContent =>
    Ok(htmlContent).as("text/html")
  }.getOrElse(NotFound)
}
```

###参数类型
对于类型为`String` 的参数, 可以不写参数类型。如果你想 Play 转换传入的参数到一个特定的Scala类型,你可以显示声明参数类型:

```
GET   /clients/:id          controllers.Clients.show(id: Long)
```

相应的，`show` 方法同样定义在`controllers.Clients` controller:

```scala
def show(id: Long) = Action {
  Client.findById(id).map { client =>
    Ok(views.html.Clients.display(client))
  }.getOrElse(NotFound)
}
```

###设定参数为固定值
有时候你会想为参数设定一个固定值:

```
# 从路径提取 page 参数, 或为 “/” 设定一个固定值 
GET   /                     controllers.Application.show(page = "home")
GET   /:page                controllers.Application.show(page)
```

###设定参数默认值
您也可以提供一个默认值，当传入的请求中找不到任何相关的值时，就使用默认参数:

```
# 分页链接, 像 /clients?page=3
GET   /clients              controllers.Clients.list(page: Int ?= 1)
```

###可选参数
你也可以指定一个可选参数，它不需要出现在所有请求中:

```
# version 参数是可选的。例如 /api/list-all?version=3.0
GET   /api/list-all         controllers.Api.list(version: Option[String])
```


##路由优先级
多个路由可以匹配到同一个请求。如果有冲突, 第一个定义的路由(按声明的顺序)会被使用。


##反向路由
路由也可以从Scala内部调用来生成URL。这样就可以将你所有的URI模式集中在一个单独的配置文件里, 让你在重构应用程序时更有把握。

对于路由文件中使用的每个controller, 路由会在`routes` 包中生成一个‘反向controller’ , 其中有同样的action方法和同样的签名, 但返回一个`play.api.mvc.Call` 而非`play.api.mvc.Action`。

`play.api.mvc.Call` 定义了一个HTTP调用，并提供HTTP方法和URI。

举例, 如果你创建了一个像这样的controller:

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

并且在`conf/routes` 文件中设置它:

```
# Hello action
GET   /hello/:name          controllers.Application.hello(name)
```

你可以反转URL到`hello` action方法, 通过使用`controllers.routes.Application` 反向controller:

```scala
// Redirect to /hello/Bob
def helloBob = Action {
  Redirect(routes.Application.hello("Bob"))
}
```