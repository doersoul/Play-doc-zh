#防范跨站请求伪造
跨站请求伪造(CSRF)是一种安全漏洞。攻击者欺骗受害者浏览器，并使用受害者的会话发送请求。由于发送的每个请求中都有会话令牌, 如果攻击者能够强制受害者的浏览发送请求，也就相当于以受害者的名义发送请求。

建议先熟悉一下CSRF, 了解哪些攻击是CSRF，哪些不是。建议从[OWASP相关信息](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29)开始。

简单来说, 攻击者能够强迫受害者的浏览器发送以下请求：

* 所有`GET` 请求
* 带有`application/x-www-form-urlencoded`, `multipart/form-data` and `text/plain` 同容的 `POST` 请求 

攻击者不能做的:

* 强迫浏览器使用其它请求方法，如`PUT` 和`DELETE`
* 强迫浏览器发送其它内容类型（content types）, 如`application/json`
* 强迫浏览器发送新cookies, 而不是服务器已经设置了的cookie
* 强迫浏览器设置任意的标头, 而不是浏览器通常会在请求中添加的普通标头

由于 `GET` 请求是不能更改的, 这对一个应用程序没有危险，这是最佳实践。所以防御CSRF仅需要注意上面提到的带有一些内容类型的`POST` 请求。

###Play的CSRF防御
Play 支持多种方法来验证一个请求是否非CSRF请求。主要机制是CSRF令牌。该令牌要放在查询字符串或每个表单提交的正文中, 并还要放在用户会话中。Play 然后会验证目前两个令牌是否存在匹配。

要允许简单的防御那些非浏览器请求，如通过AJAX发送的请求, Play也支持以下几种:

* 如果出现`X-Requested-With` 标头, Play会认为请求安全。很多主流的Javascript库都会在请求中添加`X-Requested-With`, 如jQuery。
* 如果`Csrf-Token` 标头的值为`nocheck` , 或带一个有效的CSRF令牌, Play会认为请求安全。


##应用全局CSRF过滤
Play 提供了全局 CSRF 过滤，可以应用到所有请求。这是给应用程序添加CSRF防御最简单的方式。要启用全局过滤, 可添加Play 过滤助手依赖项到你的项目中的`build.sbt`文件:

```scala
libraryDependencies += filters
```

现在添加他们到[HTTP 过滤](https://www.playframework.com/documentation/2.4.x/ScalaHttpFilters)中描述的`Filters` 类中:

```scala
import play.api.http.HttpFilters
import play.filters.csrf.CSRFFilter
import javax.inject.Inject

class Filters @Inject() (csrfFilter: CSRFFilter) extends HttpFilters {
  def filters = Seq(csrfFilter)
}
```

`Filters` 类可放在根包中、更改为其它名字、或放在其它包。改变名字和位置要在`application.conf`配置文件中使用`play.http.filters`： 

```scala
play.http.filters = "filters.MyFilters"
```

###获得当前令牌
当前CSRF令牌可通过`getToken` 方法获取。它带有一个隐式`RequestHeader`, 所以要确保它在作用域中。

```scala
import play.filters.csrf.CSRF

val token = CSRF.getToken(request)
```

Play提供了一些模板助手，以帮助添加CSRF令牌到表单中。第一个就是添加它到action URL的查询字符串中:

```scala
@import helper._

@form(CSRF(routes.ItemsController.save())) {
    ...
}
```

渲染的表单如下:

```html
<form method="POST" action="/items?csrfToken=1234567890abcdef">
   ...
</form>
```

如果不想要在查询字符串中设置令牌, Play也提供一个助手，将CSRF令牌添加到表单隐藏域中:

```scala
@form(routes.ItemsController.save()) {
    @CSRF.formField
    ...
}
```

渲染后的表单如下:

```html
<form method="POST" action="/items">
   <input type="hidden" name="csrfToken" value="1234567890abcdef"/>
   ...
</form>
```

所有表单助手方法都要在作用域中设置隐式令牌或请求。如果还没有的话，通常是由添加一个隐式`RequestHeader` 参数到你的模板来设置。

###添加CSRF令牌到会话
为确保CSRF令牌在表单中有效渲染, 并发送回客户端。如果在传入的请求中没有已经有效的令牌，全局过滤器会为所有接受HTML的GET请求生成一个新令牌, 


##在每个action上应用CSRF过滤
有时全局CSRF 过滤并不合适, 比如应用可能要允许一些跨站表单提交的情况。一些非基于会话的标准, 如OpenID 2.0, 需要使用跨站表单提交, 或是在服务器到服务器的RPC通讯中使用表单提交。

在这种情况下, Play 提供两个actions，可以组合到应用程序的actions中。

第一个action是`CSRFCheck` action, 它执行检查，会添加到所有接受会话已验证POST表单提交的action中:

```scala
import play.api.mvc._
import play.filters.csrf._

def save = CSRFCheck {
  Action { req =>
    // handle body
    Ok
  }
}
```

第二个 action 是`CSRFAddToken` action, 在传入请求中还没有令牌的情况下会生成一个CSRF令牌。它应该添加到渲染表单的所有 actions 中:

```scala
import play.api.mvc._
import play.filters.csrf._

def form = CSRFAddToken {
  Action { implicit req =>
    Ok(views.html.itemsForm())
  }
}
```

更简便的方法是将这些 actions 和Play的[action composition](https://www.playframework.com/documentation/2.4.x/ScalaActionsComposition)一起组合使用:

```scala
import play.api.mvc._
import play.filters.csrf._

object PostAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    // authentication code here
    block(request)
  }
  override def composeAction[A](action: Action[A]) = CSRFCheck(action)
}

object GetAction extends ActionBuilder[Request] {
  def invokeBlock[A](request: Request[A], block: (Request[A]) => Future[Result]) = {
    // authentication code here
    block(request)
  }
  override def composeAction[A](action: Action[A]) = CSRFAddToken(action)
}
```

这样可以最小化编写actions所需的样板代码:

```scala
def save = PostAction {
  // handle body
  Ok
}

def form = GetAction { implicit req =>
  Ok(views.html.itemsForm())
}
```


##CSRF 配置选项
可以在 [reference.conf](https://www.playframework.com/documentation/2.4.x/resources/confs/filters-helpers/reference.conf)过滤器中找到所有 CSRF配置选项。一些例子包括:

* `play.filters.csrf.token.name` - 应用于会话、请求体/查询字符串中的令牌名称。默认是`csrfToken` 。
* `play.filters.csrf.cookie.name` - 如果配置了这个, Play 会保存CSRF令牌到给定名称的cookie，而非会话中。
* `play.filters.csrf.cookie.secure` - 如果`play.filters.csrf.cookie.name` 已设置, CSRF cookie 要有安全标志设置。默认是和`play.http.session.secure` 的值相同。
* `play.filters.csrf.body.bufferSize` - 为了在读取来自body的令牌, Play 必须首先缓存body和在可能的情况下进行解析。该选项设置了缓存body时最大缓存大小，默认为100k。
* `play.filters.csrf.token.sign` - Play 是否使用签名的CSRF令牌。签名的CSRF令牌保证了每个请求的令牌值是随机的, 以防御BREACH 攻击。