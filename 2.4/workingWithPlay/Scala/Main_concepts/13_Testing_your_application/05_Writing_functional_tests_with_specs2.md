#Writing functional tests with specs2

Play provides a number of classes and convenience methods that assist with functional testing. Most of these can be found either in the [`play.api.test`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/package.html) package or in the [`Helpers`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/Helpers$.html) object.

You can add these methods and classes by importing the following:

```scala
import play.api.test._
import play.api.test.Helpers._
```


##FakeApplication
Play frequently requires a running [`Application`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Application.html) as context: it is usually provided from [`play.api.Play.current`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Play$.html).

To provide an environment for tests, Play provides a [`FakeApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeApplication.html) class which can be configured with a different Global object, additional configuration, or even additional plugins.

```scala
val fakeApplicationWithGlobal = FakeApplication(withGlobal = Some(new GlobalSettings() {
  override def onStart(app: Application) { println("Hello world!") }
}))
```


##WithApplication
To pass in an application to an example, use [`WithApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithApplication.html). An explicit [`Application`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Application.html) can be passed in, but a default [`FakeApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeApplication.html) is provided for convenience.

Because [`WithApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithApplication.html) is a built in [`Around`](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Contexts.html#aroundeach) block, you can override it to provide your own data population:

```scala
abstract class WithDbData extends WithApplication {
  override def around[T: AsResult](t: => T): Result = super.around {
    setupData()
    t
  }

  def setupData() {
    // setup data
  }
}

"Computer model" should {

  "be retrieved by id" in new WithDbData {
    // your test code
  }
  "be retrieved by email" in new WithDbData {
    // your test code
  }
}
```


##WithServer
Sometimes you want to test the real HTTP stack from within your test, in which case you can start a test server using [`WithServer`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithServer.html):

```scala
"test server logic" in new WithServer(app = fakeApplicationWithBrowser, port = testPort) {
  // The test payment gateway requires a callback to this server before it returns a result...
  val callbackURL = s"http://$myPublicAddress/callback"

  // await is from play.api.test.FutureAwaits
  val response = await(WS.url(testPaymentGatewayURL).withQueryString("callbackURL" -> callbackURL).get())

  response.status must equalTo(OK)
}
```

The `port` value contains the port number the server is running on. By default this is 19001, however you can change this either by passing the port into [`WithServer`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithServer.html), or by setting the system property `testserver.port`. This can be useful for integrating with continuous integration servers, so that ports can be dynamically reserved for each build.

A [`FakeApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeApplication.html) can also be passed to the test server, which is useful for setting up custom routes and testing WS calls:

```scala
val appWithRoutes = FakeApplication(withRoutes = {
  case ("GET", "/") =>
    Action {
      Ok("ok")
    }
})

"test WS logic" in new WithServer(app = appWithRoutes, port = 3333) {
  await(WS.url("http://localhost:3333").get()).status must equalTo(OK)
}
```


##WithBrowser
If you want to test your application using a browser, you can use [Selenium WebDriver](https://github.com/seleniumhq/selenium). Play will start the WebDriver for you, and wrap it in the convenient API provided by [FluentLenium](https://github.com/FluentLenium/FluentLenium) using [`WithBrowser`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithBrowser.html). Like [`WithServer`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithServer.html), you can change the port, [`Application`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Application.html), and you can also select the web browser to use:

```scala
val fakeApplicationWithBrowser = FakeApplication(withRoutes = {
  case ("GET", "/") =>
    Action {
      Ok(
        """
          |<html>
          |<body>
          |  <div id="title">Hello Guest</div>
          |  <a href="/login">click me</a>
          |</body>
          |</html>
        """.stripMargin) as "text/html"
    }
  case ("GET", "/login") =>
    Action {
      Ok(
        """
          |<html>
          |<body>
          |  <div id="title">Hello Coco</div>
          |</body>
          |</html>
        """.stripMargin) as "text/html"
    }
})

"run in a browser" in new WithBrowser(webDriver = WebDriverFactory(HTMLUNIT), app = fakeApplicationWithBrowser) {
  browser.goTo("/")

  // Check the page
  browser.$("#title").getTexts().get(0) must equalTo("Hello Guest")

  browser.$("a").click()

  browser.url must equalTo("/login")
  browser.$("#title").getTexts().get(0) must equalTo("Hello Coco")
}
```


##PlaySpecification
[`PlaySpecification`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/PlaySpecification.html) is an extension of [`Specification`](https://etorreborre.github.io/specs2/api/SPECS2-3.4/index.html#org.specs2.mutable.Specification) that excludes some of the mixins provided in the default specs2 specification that clash with Play helpers methods. It also mixes in the Play test helpers and types for convenience.

```scala
object ExamplePlaySpecificationSpec extends PlaySpecification {
  "The specification" should {

    "have access to HeaderNames" in {
      USER_AGENT must be_===("User-Agent")
    }

    "have access to Status" in {
      OK must be_===(200)
    }
  }
}
```


##Testing a view template
Since a template is a standard Scala function, you can execute it from your test, and check the result:

```scala
"render index template" in new WithApplication {
  val html = views.html.index("Coco")

  contentAsString(html) must contain("Hello Coco")
}
```


##Testing a controller
You can call any `Action` code by providing a [`FakeRequest`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeRequest.html):

```scala
"respond to the index Action" in {
  val result = controllers.Application.index()(FakeRequest())

  status(result) must equalTo(OK)
  contentType(result) must beSome("text/plain")
  contentAsString(result) must contain("Hello Bob")
}
```

Technically, you don’t need [`WithApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithApplication.html) here, although it wouldn’t hurt anything to have it.


##Testing the router
Instead of calling the `Action` yourself, you can let the `Router` do it:

```scala
"respond to the index Action" in new WithApplication(fakeApplication) {
  val Some(result) = route(FakeRequest(GET, "/Bob"))

  status(result) must equalTo(OK)
  contentType(result) must beSome("text/html")
  charset(result) must beSome("utf-8")
  contentAsString(result) must contain("Hello Bob")
}
```


##Testing a model
If you are using an SQL database, you can replace the database connection with an in-memory instance of an H2 database using `inMemoryDatabase`.

```scala
val appWithMemoryDatabase = FakeApplication(additionalConfiguration = inMemoryDatabase("test"))
"run an application" in new WithApplication(appWithMemoryDatabase) {

  val Some(macintosh) = Computer.findById(21)

  macintosh.name must equalTo("Macintosh")
  macintosh.introduced must beSome.which(_ must beEqualTo("1984-01-24"))
}
```