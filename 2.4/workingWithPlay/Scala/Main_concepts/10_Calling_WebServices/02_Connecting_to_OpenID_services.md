#Play的OpenID支持

OpenID 是一种使用一个账号访问多个服务的协议。作为一个web开发者, 你可以使用OpenID来让用户用他们已有的账号, 如他们的[Google 账号](https://developers.google.com/accounts/docs/OpenID)。在企业中, 你可以使用OpenID来连接公司的SSO服务器。


##OpenID工作流简述

1. 用户给你他的OpenID (一个URL)。
2. 你的服务器检查URL后面的内容，然后产生一个URL，再将用户重定向到那里。
3. 用户在他的OpenID提供方那里确认授权, 然后重定向回你的服务器。
4. 你的服务器接收重定向信息, 然后检查提供方信息是否正确。

如果你的用户使用的都是同一个OpenID提供方，那么第一步可以忽略(例如你决定全部都使用Google账号)。


##用例
要使用OpenID, 首先添加 `ws` 到你的`build.sbt` 文件:

```scala
libraryDependencies ++= Seq(
  ws
)
```

现在任何想要使用OpenID的控制器和组件需要在[OpenIdClient](https://playframework.com/documentation/2.4.x/api/scala/play/api/libs/openid/OpenIdClient.html)声明依赖:

```scala
import javax.inject.Inject
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

import play.api._
import play.api.mvc._
import play.api.data._
import play.api.data.Forms._
import play.api.libs.openid._

class Application @Inject() (openIdClient: OpenIdClient) extends Controller {

}
```

我们调用`OpenIdClient` 的实例`openIdClient`, 所有下面的示例将假定使用此名称。


##Play中使用OpenID
OpenID API 有二个重要函数:

* `OpenIdClient.redirectURL` 计算用户应该重定向的URL。它包括异步获取用户的OpenID页面, 这就是为什么它返回一个`Future[String]`。如果OpenID无效, 返回的`Future` 就会失败。
* `OpenIdClient.verifiedId` 需要一个`RequestHeader` 并检查它来建立用户的信息, 包括它验证过的OpenID。它会异步调用一下OpenID服务器以检查信息的真实性, 返回一个[UserInfo](https://playframework.com/documentation/2.4.x/api/scala/play/api/libs/openid/UserInfo.html) future。如果信息不正确或服务器的检查结果不正确(例如重定向URL是伪造的), 则返回的`Future` 会失败。

如果`Future` 失败, 你可以定义一个回退将用户重定向回登录页面或返回`BadRequest`。

这里是一个用例(从controller):

```scala
def login = Action {
  Ok(views.html.login())
}

def loginPost = Action.async { implicit request =>
  Form(single(
    "openid" -> nonEmptyText
  )).bindFromRequest.fold({ error =>
    Logger.info("bad request " + error.toString)
    Future.successful(BadRequest(error.toString))
  }, { openId =>
    openIdClient.redirectURL(openId, routes.Application.openIdCallback.absoluteURL())
      .map(url => Redirect(url))
      .recover { case t: Throwable => Redirect(routes.Application.login)}
  })
}

def openIdCallback = Action.async { implicit request =>
  openIdClient.verifiedId(request).map(info => Ok(info.id + "\n" + info.attributes))
    .recover {
    case t: Throwable =>
      // Here you should look at the error, and give feedback to the user
      Redirect(routes.Application.login)
  }
}
```


##扩展属性
用户的OpenID给你的是身份标识。协议也支持获取[扩展属性](http://openid.net/specs/openid-attribute-exchange-1_0.html) 如e-mail地址, 姓或名。

你可以从OpenID服务器请求一些可选属性或必需属性。要求提供的必需属性意味着如果用户没有提供的话，其将无法登录你的服务。

在重定向URL中请求扩展属性:

```scala
openIdClient.redirectURL(
  openId,
  routes.Application.openIdCallback.absoluteURL(),
  Seq("email" -> "http://schema.openid.net/contact/email")
)
```

这样OpenID服务器提供的`UserInfo` 中就会有这个属性了。