#Play WS API
有时候我们需要从Play应用程序内调用其它HTTP服务。Play 通过它的 [WS 库](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/package.html)提供支持, 它提供了一种异步HTTP调用的方式。

使用WS API有二个重要部分: 创建一个请求以及处理响应。我们首先讨论如何创建GET和POST HTTP 请求, 然后展示如何处理从WS返回的响应。最后, 我们会讨论一些常见的用例。


##创建一个请求
要使用, 首先添加`ws` 到你的`build.sbt` 文件:

```sbt
libraryDependencies ++= Seq(
  ws
)
```

现在想要使用 WS 的控制器和组件需要声明一个 `WSClient` 依赖:

```scala
import javax.inject.Inject
import scala.concurrent.Future

import play.api.mvc._
import play.api.libs.ws._

class Application @Inject() (ws: WSClient) extends Controller {

}
```

我们会调用 `WSClient` 的实例 `ws`, 下面所有示例都假定使用这个名称。

要构建一个HTTP请求, 要先用`ws.url()` 来指定URL。

```scala
val request: WSRequest = ws.url(url)
```

这个返回一个[WSRequest](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSRequest.html) ，你可以用它指定各种HTTP选项, 如设置标头。你可以链式调用来构造复杂的请求。

```scala
val complexRequest: WSRequest =
  request.withHeaders("Accept" -> "application/json")
    .withRequestTimeout(10000)
    .withQueryString("search" -> "play")
```

你可以通过调用一个与HTTP方法对应的方法来结束。在链的结尾, 使用定义在`WSRequest` 构建请求中的所有选项。

```scala
val futureResponse: Future[WSResponse] = complexRequest.get()
```

然后返回一个`Future[WSResponse]` ，这个[响应](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSResponse.html) 包含了从服务器返回的数据。

###带身份验证的请求
如果你需要使用HTTP身份验证, 你可以要构建中指定它, 使用用户名、密码和 [AuthScheme](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSAuthScheme.html)。AuthScheme的有效样例对象是 `BASIC`, `DIGEST`, `KERBEROS`, `NONE`, `NTLM`, 和`SPNEGO`。

```scala
ws.url(url).withAuth(user, password, WSAuthScheme.BASIC).get()
```

###带重定向的请求
如果一个HTTP调用的结果是302或301重定向, 你可以自动重定向，而无须另外调用。

```scala
ws.url(url).withFollowRedirects(true).get()
```

###带查询参数的请求
Parameters can be specified as a series of key/value tuples.

```scala
ws.url(url).withQueryString("paramKey" -> "paramValue").get()
```

###带额外标头的请求
标头可以用一系列的 键/值 元组指定。

```scala
ws.url(url).withHeaders("headerKey" -> "headerValue").get()
```

如果你想以特殊格式发送纯文本, 你要显式定义内容类型。

```scala
ws.url(url).withHeaders("Content-Type" -> "application/xml").post(xmlString)
```

###带虚拟主机的请求
一个虚拟主机可以用字符串来指定。

```scala
ws.url(url).withVirtualHost("192.168.1.1").get()
```

###带超时的请求
如果你希望指定一个请求超时, 你可以使用 `withRequestTimeout` 来设置一个毫秒值。`-1`值用来设为无限超时。 

```scala
ws.url(url).withRequestTimeout(5000).get()
```

###提交表单数据
要post一个 url-form-encoded 数据`Map[String, Seq[String]]` ，你需要将其传递给`post`。

```scala
ws.url(url).post(Map("key" -> Seq("value")))
```

###提交JSON数据
post JSON 数据最简单的方式就是使用[JSON](https://www.playframework.com/documentation/2.4.x/ScalaJson) 库。

```scala
import play.api.libs.json._
val data = Json.obj(
  "key1" -> "value1",
  "key2" -> "value2"
)
val futureResponse: Future[WSResponse] = ws.url(url).post(data)
```

###提交 XML 数据
post XML 数据最简单的方式是使用XML字面量。XML字面量很方便, 但速度不是很快。想要效率的话, 可以考虑使用XML视图模板或 JAXB库。

```scala
val data = <person>
  <name>Steve</name>
  <age>23</age>
</person>
val futureResponse: Future[WSResponse] = ws.url(url).post(data)
```


##处理响应
处理 [响应](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSResponse.html) 可以通过在[Future](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future)内做映射来轻松完成。

下面给出的例子都有一些共同的依赖，为了简便起见，这里将简单说明一下。

任何时候一个由`Future`完成的操作, 都需要一个有效的隐式执行上下文 - 这个声明了回调会运行在哪个线程池。通常默认Play执行上下文就够了:

```scala
implicit val context = play.api.libs.concurrent.Execution.Implicits.defaultContext
```

下面的示例还使用了样例类来 序列化/反序列化:

```scala
case class Person(name: String, age: Int)
```

###处理JSON响应
你可以通过调用`response.json`来将响应处理为 [JSON 对象](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsValue.html) 。

```scala
val futureResult: Future[String] = ws.url(url).get().map {
  response =>
    (response.json \ "person" \ "name").as[String]
}
```

JSON 库有一个[有用的特性](https://www.playframework.com/documentation/2.4.x/ScalaJsonCombinators)，它可以直接映射一个隐式[`Reads[T]`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html) 成一个类:

```scala
import play.api.libs.json._

implicit val personReads = Json.reads[Person]

val futureResult: Future[JsResult[Person]] = ws.url(url).get().map {
  response => (response.json \ "person").validate[Person]
}
```

###处理XML响应
你可以通过调用`response.xml`来将响应处理为[XML 字面量](http://www.scala-lang.org/api/current/index.html#scala.xml.NodeSeq)。

```scala
val futureResult: Future[scala.xml.NodeSeq] = ws.url(url).get().map {
  response =>
    response.xml \ "message"
}
```

###处理大块响应
调用 `get()` 或`post()` 会有一个问题，就是在请求体加载到内存中响应才可用。当你下载一个巨大的、几个G的文件时, 这可能会导致令人讨厌的垃圾回收或甚至内存溢出。

`WS` 可以让你通过使用 [iteratee](https://www.playframework.com/documentation/2.4.x/Iteratees)来增量地使用响应。`WSRequest` 的`stream()` 和`getStream()` 方法返回 `Future[(WSResponseHeaders, Enumerator[Array[Byte]])]`。其中，枚举器包含了响应体。

这里有一个常见的例子，使用iteratee来统计响应返回的字节数:

```scala
import play.api.libs.iteratee._

// Make the request
val futureResponse: Future[(WSResponseHeaders, Enumerator[Array[Byte]])] =
  ws.url(url).getStream()

val bytesReturned: Future[Long] = futureResponse.flatMap {
  case (headers, body) =>
    // Count the number of bytes returned
    body |>>> Iteratee.fold(0l) { (total, bytes) =>
      total + bytes.length
    }
}
```

当然, 通常你不会是只想像上面那样只计算字节数, 更多的情况下是把响应返回的数据转向另一个位置。例如, 要写入到一个文件:

```scala
import play.api.libs.iteratee._

// Make the request
val futureResponse: Future[(WSResponseHeaders, Enumerator[Array[Byte]])] =
  ws.url(url).getStream()

val downloadedFile: Future[File] = futureResponse.flatMap {
  case (headers, body) =>
    val outputStream = new FileOutputStream(file)

    // The iteratee that writes to the output stream
    val iteratee = Iteratee.foreach[Array[Byte]] { bytes =>
      outputStream.write(bytes)
    }

    // Feed the body into the iteratee
    (body |>>> iteratee).andThen {
      case result =>
        // Close the output stream whether there was an error or not
        outputStream.close()
        // Get the result or rethrow the error
        result.get
    }.map(_ => file)
}
```

另一种常见情况是，当前服务器把拿到的响应体流式写入另一个响应，返回给它所服务的对象:

```scala
def downloadFile = Action.async {

  // Make the request
  ws.url(url).getStream().map {
    case (response, body) =>

      // Check that the response was successful
      if (response.status == 200) {

        // Get the content type
        val contentType = response.headers.get("Content-Type").flatMap(_.headOption)
          .getOrElse("application/octet-stream")

        // If there's a content length, send that, otherwise return the body chunked
        response.headers.get("Content-Length") match {
          case Some(Seq(length)) =>
            Ok.feed(body).as(contentType).withHeaders("Content-Length" -> length)
          case _ =>
            Ok.chunked(body).as(contentType)
        }
      } else {
        BadGateway
      }
  }
}
```

`POST` and `PUT` calls require manually calling the `withMethod` method, eg:

```scala
val futureResponse: Future[(WSResponseHeaders, Enumerator[Array[Byte]])] =
  ws.url(url).withMethod("PUT").withBody("some body").stream()
```


##常见模式和用例

###链式 WS 调用
在一个受信任的环境，使用 for推导式是一种链式调用的好方式。for推导式应用和 [Future.recover](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future) 一起使用，以处理可能的失败。

```scala
val futureResponse: Future[WSResponse] = for {
  responseOne <- ws.url(urlOne).get()
  responseTwo <- ws.url(responseOne.body).get()
  responseThree <- ws.url(responseTwo.body).get()
} yield responseThree

futureResponse.recover {
  case e: Exception =>
    val exceptionData = Map("error" -> Seq(e.getMessage))
    ws.url(exceptionUrl).post(exceptionData)
}
```

###在controller中使用
当从controller中制造一个请求, 你可以映射响应到`Future[Result]`。这个可以组合Play的`Action.async` action 构造器一起使用, 详见 [处理异步Results](../02_Asynchronous_HTTP_programming/01_Asynchronous_results.md).

```scala
def wsAction = Action.async {
  ws.url(url).get().map { response =>
    Ok(response.body)
  }
}
status(wsAction(FakeRequest())) must_== OK
```


##使用 WSClient
WSClient 是底层 AsyncHttpClient的包装。使用不同的配置文件定义多个客户端很有用，或使用模似。

你可以直接从代码定义一个WS客户端，无须注入WS, 然后使用隐式`WS.clientUrl()`:

```scala
import play.api.libs.ws.ning._

implicit val sslClient = NingWSClient()
// close with sslClient.close() when finished with client
val response = WS.clientUrl(url).get()
```

> 注意: 如果你实例化一个 NingWSClient 对象, 它不使用WS模块的生命周期, 并不会在`Application.onStop`中自动关闭。 相反, 当处理完成时客户端必须使用`client.close()` 手动关闭。这会释放AsyncHttpClient使用的底层 ThreadPoolExecutor。未能关闭客户端可能导致内存不足异常(如果你频繁地重新加载应用程序，特别是在开发模式)。

或直接:

```scala
val response = sslClient.url(url).get()
```

或使用磁铁（magnet）模式自动匹配确定的客户端:

```scala
object PairMagnet {
  implicit def fromPair(pair: (WSClient, java.net.URL)) =
    new WSRequestMagnet {
      def apply(): WSRequest = {
        val (client, netUrl) = pair
        client.url(netUrl.toString)
      }
    }
}

import scala.language.implicitConversions
import PairMagnet._

val exampleURL = new java.net.URL(url)
val response = WS.url(ws -> exampleURL).get()
```

默认情况下, 配置写在`application.conf`, 但你也可以直接从configuration中设置:

```scala
import com.typesafe.config.ConfigFactory
import play.api._
import play.api.libs.ws._
import play.api.libs.ws.ning._

val configuration = Configuration.reference ++ Configuration(ConfigFactory.parseString(
  """
    |ws.followRedirects = true
  """.stripMargin))

// If running in Play, environment should be injected
val environment = Environment(new File("."), this.getClass.getClassLoader, Mode.Prod)

val parser = new WSConfigParser(configuration, environment)
val config = new NingWSClientConfig(wsClientConfig = parser.parse())
val builder = new NingAsyncHttpClientConfigBuilder(config)
```

你也可以直接访问底层的[async client](https://static.javadoc.io/com.ning/async-http-client/1.9.20/com/ning/http/client/AsyncHttpClient.html)。

```scala
import com.ning.http.client.AsyncHttpClient

val client: AsyncHttpClient = ws.underlying
```

这是重要的几个案例。WS在需要访问到客户端里有一些限制:

* `WS` 不支持多部分的表单直接上传。你可以使用底层客户端的[RequestBuilder.addBodyPart](https://static.javadoc.io/com.ning/async-http-client/1.9.20/com/ning/http/client/RequestBuilder.html#addBodyPart\(com.ning.http.client.multipart.Part\))来做。
* `WS` 不支持流式数据上传。在这种情况下, 你应用使用AsyncHttpClient提供的`FeedableBodyGenerator`。


##配置 WS
在`application.conf` 中使用以下属性来配置 WS客户端:

* `play.ws.followRedirects`: 配置客户端做301和302重定向(默认是 **true**)。
* `play.ws.useProxyProperties`: 使用系统的http代理设置(http.proxyHost, http.proxyPort) (默认为true)。
* `play.ws.useragent`: 配置 User-Agent 标头字段。
* `play.ws.compressionEnabled`: 设置为true来使用 gzip/deflater 编码 (默认为**false**).

###用SSL配置 WS
要配置WS在SSL/TLS (HTTPS)上使用 HTTP, 参阅 [配置 WS SSL](https://www.playframework.com/documentation/2.4.x/WsSSL).

###配置超时
WS中有三种不同的超时。达到超时限制会导致WS请求中断。

* `play.ws.timeout.connection`: 连接远程主机的最大等待时间(默认是 **120 秒**)。
* `play.ws.timeout.idle`: 请求保持空闲的最大时间(此时连接已建立但在等待更多数据) (默认是120秒)。
* `play.ws.timeout.request`: 你接受请求使用的总时间(达到这个时间请求就会中断，即使远程主机仍然在发送数据) (默认是120秒)。

可以用`withRequestTimeout()` 在一个特定连接中覆盖请求超时设置(查阅 “构建请求” 章节)。

###配置 AsyncHttpClientConfig
下面的高级设置可以配置底层AsyncHttpClientConfig。
请参考[AsyncHttpClientConfig 文档](http://asynchttpclient.github.io/async-http-client/apidocs/com/ning/http/client/AsyncHttpClientConfig.Builder.html) 了解更多信息。

* `play.ws.ning.allowPoolingConnection`
* `play.ws.ning.allowSslConnectionPool`
* `play.ws.ning.ioThreadMultiplier`
* `play.ws.ning.maxConnectionsPerHost`
* `play.ws.ning.maxConnectionsTotal`
* `play.ws.ning.maxConnectionLifeTime`
* `play.ws.ning.idleConnectionInPoolTimeout`
* `play.ws.ning.webSocketIdleTimeout`
* `play.ws.ning.maxNumberOfRedirects`
* `play.ws.ning.maxRequestRetry`
* `play.ws.ning.disableUrlEncoding`