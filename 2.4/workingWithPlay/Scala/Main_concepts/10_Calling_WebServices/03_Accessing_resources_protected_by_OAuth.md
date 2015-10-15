#OAuth

[OAuth](http://oauth.net/) 是发布受保护数据及与之交互的一种简单方式。它授权访问的更安全可靠的方式。例如， 可以用它访问你在[Twitter](https://dev.twitter.com/docs/auth/using-oauth)上的用户数据。

有二种不同版本的OAuth: [OAuth 1.0](http://tools.ietf.org/html/rfc5849) 和 [OAuth 2.0](http://oauth.net/2/)。版本 2 非常简单，无需库和助手, 所以Play仅提供 OAuth 1.0支持。


##用法
要使用OAuth, 首先添加`ws` 到你的`build.sbt` 文件:

```scala
libraryDependencies ++= Seq(
  ws
)
```


##需要的信息
OAuth 需要你注册你的应用程序到服务提供方。确保检查了你提供的回调URL, 因为如果不匹配的话，服务提供方可能会拒绝你的调用。在本地使用时，你可以在 `/etc/hosts` 中为本机伪造一个域名。

服务提供方会给你:

* 应用程序 ID
* 安全密钥
* 请求令牌 URL
* 访问令牌 URL
* 授权 URL


##验证流程
大部分工作流由Play库完成。

1. 从服务器获取请求令牌 (服务器之间调用)
2. 将用户重定向到服务提供方, 在那里他会授权你的应用可以使用他的数据。
3. 服务提供方会将用户重定向回去, 并给你一个检验器
4. 用这个检验器, 你就可以将请求令牌换成访问令牌 (服务器之间调用)

现在用访问令牌可以去访问受保护的数据了。


##示例

```scala
object Twitter extends Controller {

  val KEY = ConsumerKey("xxxxx", "xxxxx")

  val TWITTER = OAuth(ServiceInfo(
    "https://api.twitter.com/oauth/request_token",
    "https://api.twitter.com/oauth/access_token",
    "https://api.twitter.com/oauth/authorize", KEY),
    true)

  def authenticate = Action { request =>
    request.getQueryString("oauth_verifier").map { verifier =>
      val tokenPair = sessionTokenPair(request).get
      // We got the verifier; now get the access token, store it and back to index
      TWITTER.retrieveAccessToken(tokenPair, verifier) match {
        case Right(t) => {
          // We received the authorized tokens in the OAuth object - store it before we proceed
          Redirect(routes.Application.index).withSession("token" -> t.token, "secret" -> t.secret)
        }
        case Left(e) => throw e
      }
    }.getOrElse(
      TWITTER.retrieveRequestToken("http://localhost:9000/auth") match {
        case Right(t) => {
          // We received the unauthorized tokens in the OAuth object - store it before we proceed
          Redirect(TWITTER.redirectUrl(t.token)).withSession("token" -> t.token, "secret" -> t.secret)
        }
        case Left(e) => throw e
      })
  }

  def sessionTokenPair(implicit request: RequestHeader): Option[RequestToken] = {
    for {
      token <- request.session.get("token")
      secret <- request.session.get("secret")
    } yield {
      RequestToken(token, secret)
    }
  }
}
object Application extends Controller {

  def timeline = Action.async { implicit request =>
    Twitter.sessionTokenPair match {
      case Some(credentials) => {
        WS.url("https://api.twitter.com/1.1/statuses/home_timeline.json")
          .sign(OAuthCalculator(Twitter.KEY, credentials))
          .get
          .map(result => Ok(result.json))
      }
      case _ => Future.successful(Redirect(routes.Twitter.authenticate))
    }
  }
}
```