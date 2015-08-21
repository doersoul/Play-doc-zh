#Session 和 Flash scopes

##在Play中他们有什么不同
如果你需要跨多个HTTP请求的共同数据, 你可以把它们保存到Session或Flash scopes中。存储在Session中的数据是在整个用户会话期间可用, 而存储在Flash scope中的数据 **仅** 在下一次请求中可用。

重要的一点要理解，是Session和Flash数据并非由服务器来存储，而是使用cookie机制，添加到每个后续的HTTP请求。这也就是说数据的大小是很受限制的(最多4 KB)，并且你仅能保存字符串值。cookie的默认名字是`PLAY_SESSION`。这个可以通过在application.conf中改变`session.cookieName` 键的值来修改。

> 如果cookie的名字改变了, 可以使用上一节中[设置和丢弃cookies](03_Manipulating_results.md)同样的方法来丢弃早先的cookie。

当然, cookie 值是用密钥签名的，因此客户端不能修改cookie数据(否则它会失效)。

Play的Session不能当成缓存来使用。如果你需要缓存一些与特定会话相关的数据, 你可以使用Play内置的缓存机制，并在用户会话中存储一个唯一ID以关联特定的用户。

> 默认情况下，在技术上讲Session没有超时控制。它在用户关闭web浏览器后才过期。如果你的特定应用程序需要功能性超时控制, 只需存储一个时间戳到用户会话中，并在应用需要时使用它(例如：最大会话持续时间, 最大非活动时间, 等等)。你也可以通过在`application.conf`中配置`session.maxAge`键（以毫秒为单位），为session cookie设置maximum age。


##存储数据到Session
由于Session是一个Cookie也是一个HTTP标头。你可以用操纵其它results属性同样的方式操纵session数据:

```scala
Ok("Welcome!").withSession(
  "connected" -> "user@gmail.com")
```

请注意这会替换掉整个session。如果你需要添加元素到一个已存在的Session, 只需要将元素添加到传入的Session, 并且把它做为新的session:

```scala
Ok("Hello World!").withSession(
  request.session + ("saidHello" -> "yes"))
```

你可以用同样方式，从传入的session中移除任何值:

```scala
Ok("Theme reset!").withSession(
  request.session - "theme")
```


##读取Session值
你可以从HTTP请求中取回传入的Session:

```scala
def index = Action { request =>
  request.session.get("connected").map { user =>
    Ok("Hello " + user)
  }.getOrElse {
    Unauthorized("Oops, you are not connected")
  }
}
```


##丢弃整个session
这里有一个特殊操作来丢度整个session:

```scala
Ok("Bye").withNewSession
```


##Flash scope
Flash scope 工作方式非常像Session, 但有两点不同:

* 数据仅为一次请求而保留
* Flash cookie未签名, 使得用户可以修改它

> **重要**: Flash scope 应该只用于在简单的非Ajax应用中传送 Success/error 消息。由于数据只为下一次请求保留，而且因为在复杂的Web应用中无法保证请求的顺序, Flash scope 受制于竞争条件。

这里有几个使用Flash scope的例子:

```scala
def index = Action { implicit request =>
  Ok {
    request.flash.get("success").getOrElse("Welcome!")
  }
}

def save = Action {
  Redirect("/home").flashing(
    "success" -> "The item has been created")
}
```

要在你的视图中取回Flash scope值, 添加一个隐式Flash参数:

```scala
@()(implicit flash: Flash)
...
@flash.get("success").getOrElse("Welcome!")
...
```

并且在你的 Action, 指定一个`implicit request =>` ，如下所示:

```scala
def index = Action { implicit request =>
  Ok(views.html.index())
}
```

一个隐式Flash将提供给基于隐式请求的视图。

如果出现 *‘could not find implicit value for parameter flash: play.api.mvc.Flash’* 错误，这是因为你的 Action 没有添加 implicit request 到作用域。