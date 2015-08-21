#内容协商

内容协商是一个使得相同的资源(URI)可以提供不同表现的机制。它是很有用的，例如为编写Web Services支持几种不同的输出格式(XML, JSON, 等等)。服务端驱动的协商基本上是用`Accept*` 请求标头来完成的。你可以在[HTTP 规范](http://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html)找到更多关于内容协商的信息。

##语言

你可以使用`play.api.mvc.RequestHeader#acceptLanguages` 方法获得针对一个请求的可接受语言列表，该方法从`Accept-Language` 标头获取这些语言并根据他们的品质值排序。Play 在`play.api.mvc.Controller#lang` 方法中使用它，该方法提供一个隐式`play.api.i18n.Lang` 值到你的actions, 因此他们自动使用最有可能的语言(如果你的应用程序支持, 否则会使用应用程序的默认语言)。

##内容

与上面类似, `play.api.mvc.RequestHeader#acceptedTypes` 方法给出针对一个请求的可接受result的MIME类型列表。该方法从`Accept`请求标头获取这些MIME类型，并根据他们的品质因子排序。

事实上, `Accept` 标头并不是真的包含MIME类型，而是媒体区间 (*例如：* 一个请求接受所有文本结果，可以设置为`text/*` 区间, 而`*/*` 区间即所有结果类型都可接受)。Controllers提供一个更高级`render` 方法来帮助你处理媒体区间。例如考虑以下的action定义:

```scala
val list = Action { implicit request =>
  val items = Item.findAll
  render {
    case Accepts.Html() => Ok(views.html.list(items))
    case Accepts.Json() => Ok(Json.toJson(items))
  }
}
```

`Accepts.Html()` 和 `Accepts.Json()` 是一个提取器，分别测试给定的媒体区间是匹配`text/html` 还是 `application/json` 。`render` 方法带有一个`play.api.http.MediaRange => play.api.mvc.Result` 的偏函数做为参数，并按优先顺序，在请求的`Accept` 标头中找到的每个媒体区间尝试应用它。如果所有可接受的媒体区间都不被你的函数支持, 那么返回一个`NotAcceptable` 。

举例, 如果一个客户端发出一个请求带以下`Accept` 标头值: `*/*;q=0.5,application/json`, 意味着它接受任意result类型，但是JSON优先, 上面那段代码就会返回JSON结果。如果另一个客户端发出一个请求带以下`Accept` 标头值: `application/xml`, 意味着它仅接受XML, 上面的代码会返回`NotAcceptable`。

##请求提取器

参阅 API 文档中的`play.api.mvc.AcceptExtractors.Accepts` 对象，了解通过Play在`render`方法中开箱即用支持的MIME类型列表。使用`play.api.mvc.Accepting` 样例类，你可以很轻松地为给定的MIME类型创建你自己的提取器。例如以下代码创建一个提取器，用于检查媒体区间是否匹配`audio/mp3` MIME类型:

```scala
  val AcceptsMp3 = Accepting("audio/mp3")
  render {
    case AcceptsMp3() => ???
  }
}
```