#The Play WS API
Sometimes we would like to call other HTTP services from within a Play application. Play supports this via its [WS library](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/package.html), which provides a way to make asynchronous HTTP calls.

There are two important parts to using the WS API: making a request, and processing the response. We’ll discuss how to make both GET and POST HTTP requests first, and then show how to process the response from WS. Finally, we’ll discuss some common use cases.


##Making a Request
To use WS, first add `ws` to your `build.sbt` file:

```sbt
libraryDependencies ++= Seq(
  ws
)
```

Now any controller or component that wants to use WS will have to declare a dependency on the `WSClient`:

```scala
import javax.inject.Inject
import scala.concurrent.Future

import play.api.mvc._
import play.api.libs.ws._

class Application @Inject() (ws: WSClient) extends Controller {

}
```

We’ve called the `WSClient` instance `ws`, all the following examples will assume this name.

To build an HTTP request, you start with `ws.url()` to specify the URL.

```scala
val request: WSRequest = ws.url(url)
```

This returns a [WSRequest](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSRequest.html) that you can use to specify various HTTP options, such as setting headers. You can chain calls together to construct complex requests.

```scala
val complexRequest: WSRequest =
  request.withHeaders("Accept" -> "application/json")
    .withRequestTimeout(10000)
    .withQueryString("search" -> "play")
```

You end by calling a method corresponding to the HTTP method you want to use. This ends the chain, and uses all the options defined on the built request in the `WSRequest`.

```scala
val futureResponse: Future[WSResponse] = complexRequest.get()
```

This returns a `Future[WSResponse]` where the [Response](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSResponse.html) contains the data returned from the server.

###Request with authentication
If you need to use HTTP authentication, you can specify it in the builder, using a username, password, and an [AuthScheme](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSAuthScheme.html). Valid case objects for the AuthScheme are `BASIC`, `DIGEST`, `KERBEROS`, `NONE`, `NTLM`, and `SPNEGO`.

```scala
ws.url(url).withAuth(user, password, WSAuthScheme.BASIC).get()
```

###Request with follow redirects
If an HTTP call results in a 302 or a 301 redirect, you can automatically follow the redirect without having to make another call.

```scala
ws.url(url).withFollowRedirects(true).get()
```

###Request with query parameters
Parameters can be specified as a series of key/value tuples.

```scala
ws.url(url).withQueryString("paramKey" -> "paramValue").get()
```

###Request with additional headers
Headers can be specified as a series of key/value tuples.

```scala
ws.url(url).withHeaders("headerKey" -> "headerValue").get()
```

If you are sending plain text in a particular format, you may want to define the content type explicitly.

```scala
ws.url(url).withHeaders("Content-Type" -> "application/xml").post(xmlString)
```

###Request with virtual host
A virtual host can be specified as a string.

```scala
ws.url(url).withVirtualHost("192.168.1.1").get()
```

###Request with timeout
If you wish to specify a request timeout, you can use `withRequestTimeout` to set a value in milliseconds. A value of `-1` can be used to set an infinite timeout. 

```scala
ws.url(url).withRequestTimeout(5000).get()
```

###Submitting form data
To post url-form-encoded data a `Map[String, Seq[String]]` needs to be passed into `post`.

```scala
ws.url(url).post(Map("key" -> Seq("value")))
```

###Submitting JSON data
The easiest way to post JSON data is to use the [JSON](https://www.playframework.com/documentation/2.4.x/ScalaJson) library.

```scala
import play.api.libs.json._
val data = Json.obj(
  "key1" -> "value1",
  "key2" -> "value2"
)
val futureResponse: Future[WSResponse] = ws.url(url).post(data)
```

###Submitting XML data
The easiest way to post XML data is to use XML literals. XML literals are convenient, but not very fast. For efficiency, consider using an XML view template, or a JAXB library.

```scala
val data = <person>
  <name>Steve</name>
  <age>23</age>
</person>
val futureResponse: Future[WSResponse] = ws.url(url).post(data)
```


##Processing the Response
Working with the [Response](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/ws/WSResponse.html) is easily done by mapping inside the [Future](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future) .

The examples given below have some common dependencies that will be shown once here for brevity.

Whenever an operation is done on a `Future`, an implicit execution context must be available - this declares which thread pool the callback to the future should run in. The default Play execution context is often sufficient:

```scala
implicit val context = play.api.libs.concurrent.Execution.Implicits.defaultContext
```

The examples also use the folowing case class for serialization / deserialization:

```scala
case class Person(name: String, age: Int)
```

###Processing a response as JSON
You can process the response as a [JSON object](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsValue.html) by calling `response.json`.

```scala
val futureResult: Future[String] = ws.url(url).get().map {
  response =>
    (response.json \ "person" \ "name").as[String]
}
```

The JSON library has a [useful feature](https://www.playframework.com/documentation/2.4.x/ScalaJsonCombinators) that will map an implicit [`Reads[T]`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html) directly to a class:

```scala
import play.api.libs.json._

implicit val personReads = Json.reads[Person]

val futureResult: Future[JsResult[Person]] = ws.url(url).get().map {
  response => (response.json \ "person").validate[Person]
}
```

###Processing a response as XML
You can process the response as an [XML literal](http://www.scala-lang.org/api/current/index.html#scala.xml.NodeSeq)  by calling `response.xml`.

```scala
val futureResult: Future[scala.xml.NodeSeq] = ws.url(url).get().map {
  response =>
    response.xml \ "message"
}
```

###Processing large responses
Calling `get()` or `post()` will cause the body of the request to be loaded into memory before the response is made available. When you are downloading with large, multi-gigabyte files, this may result in unwelcome garbage collection or even out of memory errors.

`WS` lets you use the response incrementally by using an [iteratee](https://www.playframework.com/documentation/2.4.x/Iteratees). The `stream()` and `getStream()` methods on `WSRequest` return `Future[(WSResponseHeaders, Enumerator[Array[Byte]])]`. The enumerator contains the response body.

Here is a trivial example that uses an iteratee to count the number of bytes returned by the response:

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

Of course, usually you won’t want to consume large bodies like this, the more common use case is to stream the body out to another location. For example, to stream the body to a file:

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

Another common destination for response bodies is to stream them through to a response that this server is currently serving:

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


##Common Patterns and Use Cases

###Chaining WS calls
Using for comprehensions is a good way to chain WS calls in a trusted environment. You should use for comprehensions together with [Future.recover](http://www.scala-lang.org/api/current/index.html#scala.concurrent.Future)  to handle possible failure.

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

###Using in a controller
When making a request from a controller, you can map the response to a `Future[Result]`. This can be used in combination with Play’s `Action.async` action builder, as described in [Handling Asynchronous Results](https://www.playframework.com/documentation/2.4.x/ScalaAsync).

```scala
def wsAction = Action.async {
  ws.url(url).get().map { response =>
    Ok(response.body)
  }
}
status(wsAction(FakeRequest())) must_== OK
```


##Using WSClient
WSClient is a wrapper around the underlying AsyncHttpClient. It is useful for defining multiple clients with different profiles, or using a mock.

You can define a WS client directly from code without having it injected by WS, and then use it implicitly with `WS.clientUrl()`:

```scala
import play.api.libs.ws.ning._

implicit val sslClient = NingWSClient()
// close with sslClient.close() when finished with client
val response = WS.clientUrl(url).get()
```

> NOTE: if you instantiate a NingWSClient object, it does not use the WS module lifecycle, and so will not be automatically closed in `Application.onStop`. Instead, the client must be manually shutdown using `client.close()` when processing has completed. This will release the underlying ThreadPoolExecutor used by AsyncHttpClient. Failure to close the client may result in out of memory exceptions (especially if you are reloading an application frequently in development mode).

or directly:

```scala
val response = sslClient.url(url).get()
```

Or use a magnet pattern to match up certain clients automatically:

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

By default, configuration happens in `application.conf`, but you can also set up the builder directly from configuration:

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

You can also get access to the underlying [async client](https://static.javadoc.io/com.ning/async-http-client/1.9.20/com/ning/http/client/AsyncHttpClient.html) .

```scala
import com.ning.http.client.AsyncHttpClient

val client: AsyncHttpClient = ws.underlying
```

This is important in a couple of cases. WS has a couple of limitations that require access to the client:

* `WS` does not support multi part form upload directly. You can use the underlying client with [RequestBuilder.addBodyPart](https://static.javadoc.io/com.ning/async-http-client/1.9.20/com/ning/http/client/RequestBuilder.html#addBodyPart\(com.ning.http.client.multipart.Part\)) .
* `WS` does not support streaming body upload. In this case, you should use the `FeedableBodyGenerator` provided by AsyncHttpClient.


##Configuring WS
Use the following properties in `application.conf` to configure the WS client:

* `play.ws.followRedirects`: Configures the client to follow 301 and 302 redirects (default is **true**).
* `play.ws.useProxyProperties`: To use the system http proxy settings(http.proxyHost, http.proxyPort) (default is true).
* `play.ws.useragent`: To configure the User-Agent header field.
* `play.ws.compressionEnabled`: Set it to true to use gzip/deflater encoding (default is **false**).

###Configuring WS with SSL
To configure WS for use with HTTP over SSL/TLS (HTTPS), please see [Configuring WS SSL](https://www.playframework.com/documentation/2.4.x/WsSSL).

###Configuring Timeouts
There are 3 different timeouts in WS. Reaching a timeout causes the WS request to interrupt.

* `play.ws.timeout.connection`: The maximum time to wait when connecting to the remote host (default is **120 seconds**).
* `play.ws.timeout.idle`: The maximum time the request can stay idle (connection is established but waiting for more data) (default is 120 seconds).
* `play.ws.timeout.request`: The total time you accept a request to take (it will be interrupted even if the remote host is still sending data) (default is 120 seconds).

The request timeout can be overridden for a specific connection with `withRequestTimeout()` (see “Making a Request” section).

###Configuring AsyncHttpClientConfig
The following advanced settings can be configured on the underlying AsyncHttpClientConfig.
Please refer to the [AsyncHttpClientConfig Documentation](http://asynchttpclient.github.io/async-http-client/apidocs/com/ning/http/client/AsyncHttpClientConfig.Builder.html)  for more information.

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