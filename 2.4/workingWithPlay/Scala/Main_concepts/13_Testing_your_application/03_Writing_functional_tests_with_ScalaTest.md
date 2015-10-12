#Writing functional tests with ScalaTest

Play provides a number of classes and convenience methods that assist with functional testing. Most of these can be found either in the [`play.api.test`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/package.html) package or in the [`Helpers`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/Helpers$.html) object. The [`ScalaTest + Play`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.package) integration library builds on this testing support for ScalaTest.

You can access all of Play’s built-in test support and ScalaTest + Play with the following imports:

```scala
import org.scalatest._
import play.api.test._
import play.api.test.Helpers._
import org.scalatestplus.play._
```


##FakeApplication
Play frequently requires a running [`Application`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Application.html) as context: it is usually provided from [`play.api.Play.current`](https://playframework.com/documentation/2.4.x/api/scala/play/api/Play$.html).

To provide an environment for tests, Play provides a [`FakeApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeApplication.html) class which can be configured with a different `Global` object, additional configuration, or even additional plugins.

```scala
val fakeApplicationWithGlobal = FakeApplication(withGlobal = Some(new GlobalSettings() {
  override def onStart(app: Application) { println("Hello world!") }
}))
```

If all or most tests in your test class need a `FakeApplication`, and they can all share the same `FakeApplication`, mix in trait [`OneAppPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneAppPerSuite). You can access the `FakeApplication` from the `app` field. If you need to customize the `FakeApplication`, override `app` as shown in this example:

```scala
class ExampleSpec extends PlaySpec with OneAppPerSuite {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled")
    )

  "The OneAppPerSuite trait" must {
    "provide a FakeApplication" in {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in {
      Play.maybeApplication mustBe Some(app)
    }
  }
}
```

If you need each test to get its own `FakeApplication`, instead of sharing the same one, use `OneAppPerTest` instead:

```scala
class ExampleSpec extends PlaySpec with OneAppPerTest {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override def newAppForTest(td: TestData): FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled")
    )

  "The OneAppPerTest trait" must {
    "provide a new FakeApplication for each test" in {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in {
      Play.maybeApplication mustBe Some(app)
    }
  }
}
```

The reason *ScalaTest + Play* provides both `OneAppPerSuite` and `OneAppPerTest` is to allow you to select the sharing strategy that makes your tests run fastest. If you want application state maintained between successive tests, you’ll need to use `OneAppPerSuite`. If each test needs a clean slate, however, you could either use `OneAppPerTest` or use `OneAppPerSuite`, but clear any state at the end of each test. Furthermore, if your test suite will run fastest if multiple test classes share the same application, you can define a master suite that mixes in [`OneAppPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneAppPerSuite) and nested suites that mix in [`ConfiguredApp`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredApp), as shown in the example in the [documentation for `ConfiguredApp`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredApp). You can use whichever strategy makes your test suite run the fastest.


##Testing with a server
Sometimes you want to test with the real HTTP stack. If all tests in your test class can reuse the same server instance, you can mix in [`OneServerPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneServerPerSuite) (which will also provide a new `FakeApplication` for the suite):

```scala
class ExampleSpec extends PlaySpec with OneServerPerSuite {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/") => Action { Ok("ok") }
      }
    )

  "test server logic" in {
    val myPublicAddress =  s"localhost:$port"
    val testPaymentGatewayURL = s"http://$myPublicAddress"
    // The test payment gateway requires a callback to this server before it returns a result...
    val callbackURL = s"http://$myPublicAddress/callback"
    // await is from play.api.test.FutureAwaits
    val response = await(WS.url(testPaymentGatewayURL).withQueryString("callbackURL" -> callbackURL).get())

    response.status mustBe (OK)
  }
}
```

If all tests in your test class require separate server instance, use [`OneServerPerTest`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneServerPerTest) instead (which will also provide a new `FakeApplication` for the suite):

```scala
class ExampleSpec extends PlaySpec with OneServerPerTest {

  // Override newAppForTest if you need a FakeApplication with other than
  // default parameters.
  override def newAppForTest(testData: TestData): FakeApplication =
    new FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/") => Action { Ok("ok") }
      }
    )

  "The OneServerPerTest trait" must {
    "test server logic" in {
      val myPublicAddress =  s"localhost:$port"
      val testPaymentGatewayURL = s"http://$myPublicAddress"
      // The test payment gateway requires a callback to this server before it returns a result...
      val callbackURL = s"http://$myPublicAddress/callback"
      // await is from play.api.test.FutureAwaits
      val response = await(WS.url(testPaymentGatewayURL).withQueryString("callbackURL" -> callbackURL).get())

      response.status mustBe (OK)
    }
  }
}
```

The `OneServerPerSuite` and `OneServerPerTest` traits provide the port number on which the server is running as the `port` field. By default this is 19001, however you can change this either overriding `port` or by setting the system property `testserver.port`. This can be useful for integrating with continuous integration servers, so that ports can be dynamically reserved for each build.

You can also customize the [`FakeApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeApplication.html) by overriding `app`, as demonstrated in the previous examples.

Lastly, if allowing multiple test classes to share the same server will give you better performance than either the `OneServerPerSuite` or `OneServerPerTest` approaches, you can define a master suite that mixes in [`OneServerPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneServerPerSuite) and nested suites that mix in [`ConfiguredServer`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredServer), as shown in the example in the [documentation for `ConfiguredServer`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredServer).


##Testing with a web browser
The *ScalaTest + Play* library builds on ScalaTest’s [Selenium DSL](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.selenium.WebBrowser) to make it easy to test your Play applications from web browsers.

To run all tests in your test class using a same browser instance, mix [`OneBrowserPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneBrowserPerSuite) into your test class. You’ll also need to mix in a [`BrowserFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.BrowserFactory) trait that will provide a Selenium web driver: one of [`ChromeFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ChromeFactory), [`FirefoxFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.FirefoxFactory), [`HtmlUnitFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.HtmlUnitFactory), [`InternetExplorerFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.InternetExplorerFactory), [`SafariFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.SafariFactory).

In addition to mixing in a [`BrowserFactory`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.BrowserFactory), you will need to mix in a [`ServerProvider`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ServerProvider) trait that provides a `TestServer`: one of [`OneServerPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneServerPerSuite), [`OneServerPerTest`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneServerPerTest), or [`ConfiguredServer`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredServer).

For example, the following test class mixes in `OneServerPerSuite` and `HtmUnitFactory`:

```scala
class ExampleSpec extends PlaySpec with OneServerPerSuite with OneBrowserPerSuite with HtmlUnitFactory {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/testing") =>
          Action(
            Results.Ok(
              "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body>" +
                "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                "</body>" +
                "</html>"
            ).as("text/html")
          )
      }
    )

  "The OneBrowserPerTest trait" must {
    "provide a web driver" in {
      go to (s"http://localhost:$port/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }
}
```

If each of your tests requires a new browser instance, use [`OneBrowserPerTest`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneBrowserPerSuite) instead. As with `OneBrowserPerSuite`, you’ll need to also mix in a `ServerProvider` and `BrowserFactory`:

```scala
class ExampleSpec extends PlaySpec with OneServerPerTest with OneBrowserPerTest with HtmlUnitFactory {

  // Override newAppForTest if you need a FakeApplication with other than
  // default parameters.
  override def newAppForTest(testData: TestData): FakeApplication =
    new FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/testing") =>
          Action(
            Results.Ok(
              "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body>" +
                "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                "</body>" +
                "</html>"
            ).as("text/html")
          )
      }
    )

  "The OneBrowserPerTest trait" must {
    "provide a web driver" in {
      go to (s"http://localhost:$port/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }
}
```

If you need multiple test classes to share the same browser instance, mix [`OneBrowserPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.OneBrowserPerSuite) into a master suite and [`ConfiguredBrowser`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredBrowser) into multiple nested suites. The nested suites will all share the same web browser. For an example, see the [documentation for trait `ConfiguredBrowser`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.ConfiguredBrowser).


##Running the same tests in multiple browsers
If you want to run tests in multiple web browsers, to ensure your application works correctly in all the browsers you support, you can use traits [`AllBrowsersPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.AllBrowsersPerSuite) or [`AllBrowsersPerTest`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.AllBrowsersPerTest). Both of these traits declare a `browsers` field of type `IndexedSeq[BrowserInfo]` and an abstract `sharedTests` method that takes a `BrowserInfo`. The `browsers` field indicates which browsers you want your tests to run in. The default is Chrome, Firefox, Internet Explorer, `HtmlUnit`, and Safari. You can override `browsers` if the default doesn’t fit your needs. You place tests you want run in multiple browsers in the `sharedTests` method, placing the name of the browser at the end of each test name. (The browser name is available from the `BrowserInfo` passed into `sharedTests`.) Here’s an example that uses `AllBrowsersPerSuite`:

```scala
class ExampleSpec extends PlaySpec with OneServerPerSuite with AllBrowsersPerSuite {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/testing") =>
          Action(
            Results.Ok(
              "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body>" +
                "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                "</body>" +
                "</html>"
            ).as("text/html")
          )
      }
    )

  def sharedTests(browser: BrowserInfo) = {
    "The AllBrowsersPerSuite trait" must {
      "provide a web driver " + browser.name in {
        go to (s"http://localhost:$port/testing")
        pageTitle mustBe "Test Page"
        click on find(name("b")).value
        eventually { pageTitle mustBe "scalatest" }
      }
    }
  }
}
```

All tests declared by `sharedTests` will be run with all `browsers` mentioned in the `browsers` field, so long as they are available on the host system. Tests for any browser that is not available on the host system will be canceled automatically. Note that you need to append the `browser.name` manually to the test name to ensure each test in the suite has a unique name (which is required by ScalaTest). If you leave that off, you’ll get a duplicate-test-name error when you run your tests.

[`AllBrowsersPerSuite`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.AllBrowsersPerSuite) will create a single instance of each type of browser and use that for all the tests declared in `sharedTests`. If you want each test to have its own, brand new browser instance, use [`AllBrowsersPerTest`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.AllBrowsersPerTest) instead:

```scala
class ExampleSpec extends PlaySpec with OneServerPerSuite with AllBrowsersPerTest {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/testing") =>
          Action(
            Results.Ok(
              "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body>" +
                "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                "</body>" +
                "</html>"
            ).as("text/html")
          )
      }
    )

  def sharedTests(browser: BrowserInfo) = {
    "The AllBrowsersPerTest trait" must {
      "provide a web driver"  + browser.name in {
        go to (s"http://localhost:$port/testing")
        pageTitle mustBe "Test Page"
        click on find(name("b")).value
        eventually { pageTitle mustBe "scalatest" }
      }
    }
  }
}
```

Although both `AllBrowsersPerSuite` and `AllBrowsersPerTest` will cancel tests for unavailable browser types, the tests will show up as canceled in the output. To can clean up the output, you can exclude web browsers that will never be available by overriding `browsers`, as shown in this example:

```scala
class ExampleOverrideBrowsersSpec extends PlaySpec with OneServerPerSuite with AllBrowsersPerSuite {

  override lazy val browsers =
    Vector(
      FirefoxInfo(firefoxProfile),
      ChromeInfo
    )

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/testing") =>
          Action(
            Results.Ok(
              "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body>" +
                "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                "</body>" +
                "</html>"
            ).as("text/html")
          )
      }
    )

  def sharedTests(browser: BrowserInfo) = {
    "The AllBrowsersPerSuite trait" must {
      "provide a web driver"  + browser.name in {
        go to (s"http://localhost:$port/testing")
        pageTitle mustBe "Test Page"
        click on find(name("b")).value
        eventually { pageTitle mustBe "scalatest" }
      }
    }
  }
}
```

The previous test class will only attempt to run the shared tests with Firefox and Chrome (and cancel tests automatically if a browser is not available).

##PlaySpec
`PlaySpec` provides a convenience “super Suite” ScalaTest base class for Play tests, You get `WordSpec`, `MustMatchers`, `OptionValues`, and `WsScalaTestClient` automatically by extending `PlaySpec`:

```scala
class ExampleSpec extends PlaySpec with OneServerPerSuite with  ScalaFutures with IntegrationPatience {

  // Override app if you need a FakeApplication with other than
  // default parameters.
  implicit override lazy val app: FakeApplication =
    FakeApplication(
      additionalConfiguration = Map("ehcacheplugin" -> "disabled"),
      withRoutes = {
        case ("GET", "/testing") =>
          Action(
            Results.Ok(
              "<html>" +
                "<head><title>Test Page</title></head>" +
                "<body>" +
                "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
                "</body>" +
                "</html>"
            ).as("text/html")
          )
      }
    )

  "WsScalaTestClient's" must {

    "wsUrl works correctly" in {
      val futureResult = wsUrl("/testing").get
      val body = futureResult.futureValue.body
      val expectedBody =
        "<html>" +
          "<head><title>Test Page</title></head>" +
          "<body>" +
          "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
          "</body>" +
          "</html>"
      assert(body == expectedBody)
    }

    "wsCall works correctly" in {
      val futureResult = wsCall(Call("get", "/testing")).get
      val body = futureResult.futureValue.body
      val expectedBody =
        "<html>" +
          "<head><title>Test Page</title></head>" +
          "<body>" +
          "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
          "</body>" +
          "</html>"
      assert(body == expectedBody)
    }
  }
}
```

You can mix any of the previously mentioned traits into `PlaySpec`.


##When different tests need different fixtures
In all the test classes shown in previous examples, all or most tests in the test class required the same fixtures. While this is common, it is not always the case. If different tests in the same test class need different fixtures, mix in trait [`MixedFixtures`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures). Then give each individual test the fixture it needs using one of these no-arg functions: [App](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$App), [Server](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$Server), [Chrome](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$Chrome), [Firefox](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$Firefox), [HtmlUnit](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$HtmlUnit), [InternetExplorer](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$InternetExplorer), or [Safari](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures$Safari).

You cannot mix [`MixedFixtures`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedFixtures) into [`PlaySpec`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.PlaySpec) because `MixedFixtures` requires a ScalaTest [`fixture.Suite`](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.fixture.Suite) and `PlaySpec` is just a regular [`Suite`](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.Suite). If you want a convenient base class for mixed fixtures, extend [`MixedPlaySpec`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.MixedPlaySpec) instead. Here’s an example:

```scala
// MixedPlaySpec already mixes in MixedFixtures
class ExampleSpec extends MixedPlaySpec {

  // Some helper methods
  def fakeApp[A](elems: (String, String)*) = FakeApplication(additionalConfiguration = Map(elems:_*),
    withRoutes = {
      case ("GET", "/testing") =>
        Action(
          Results.Ok(
            "<html>" +
              "<head><title>Test Page</title></head>" +
              "<body>" +
              "<input type='button' name='b' value='Click Me' onclick='document.title=\"scalatest\"' />" +
              "</body>" +
              "</html>"
          ).as("text/html")
        )
    })
  def getConfig(key: String)(implicit app: Application) = app.configuration.getString(key)

  // If a test just needs a FakeApplication, use "new App":
  "The App function" must {
    "provide a FakeApplication" in new App(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new App(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new App(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
  }

  // If a test needs a FakeApplication and running TestServer, use "new Server":
  "The Server function" must {
    "provide a FakeApplication" in new Server(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new Server(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new Server(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
    import Helpers._
    "send 404 on a bad request" in new Server {
      import java.net._
      val url = new URL("http://localhost:" + port + "/boom")
      val con = url.openConnection().asInstanceOf[HttpURLConnection]
      try con.getResponseCode mustBe 404
      finally con.disconnect()
    }
  }

  // If a test needs a FakeApplication, running TestServer, and Selenium
  // HtmlUnit driver use "new HtmlUnit":
  "The HtmlUnit function" must {
    "provide a FakeApplication" in new HtmlUnit(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new HtmlUnit(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new HtmlUnit(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
    import Helpers._
    "send 404 on a bad request" in new HtmlUnit {
      import java.net._
      val url = new URL("http://localhost:" + port + "/boom")
      val con = url.openConnection().asInstanceOf[HttpURLConnection]
      try con.getResponseCode mustBe 404
      finally con.disconnect()
    }
    "provide a web driver" in new HtmlUnit(fakeApp()) {
      go to ("http://localhost:" + port + "/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }

  // If a test needs a FakeApplication, running TestServer, and Selenium
  // Firefox driver use "new Firefox":
  "The Firefox function" must {
    "provide a FakeApplication" in new Firefox(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new Firefox(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new Firefox(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
    import Helpers._
    "send 404 on a bad request" in new Firefox {
      import java.net._
      val url = new URL("http://localhost:" + port + "/boom")
      val con = url.openConnection().asInstanceOf[HttpURLConnection]
      try con.getResponseCode mustBe 404
      finally con.disconnect()
    }
    "provide a web driver" in new Firefox(fakeApp()) {
      go to ("http://localhost:" + port + "/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }

  // If a test needs a FakeApplication, running TestServer, and Selenium
  // Safari driver use "new Safari":
  "The Safari function" must {
    "provide a FakeApplication" in new Safari(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new Safari(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new Safari(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
    import Helpers._
    "send 404 on a bad request" in new Safari {
      import java.net._
      val url = new URL("http://localhost:" + port + "/boom")
      val con = url.openConnection().asInstanceOf[HttpURLConnection]
      try con.getResponseCode mustBe 404
      finally con.disconnect()
    }
    "provide a web driver" in new Safari(fakeApp()) {
      go to ("http://localhost:" + port + "/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }

  // If a test needs a FakeApplication, running TestServer, and Selenium
  // Chrome driver use "new Chrome":
  "The Chrome function" must {
    "provide a FakeApplication" in new Chrome(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new Chrome(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new Chrome(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
    import Helpers._
    "send 404 on a bad request" in new Chrome {
      import java.net._
      val url = new URL("http://localhost:" + port + "/boom")
      val con = url.openConnection().asInstanceOf[HttpURLConnection]
      try con.getResponseCode mustBe 404
      finally con.disconnect()
    }
    "provide a web driver" in new Chrome(fakeApp()) {
      go to ("http://localhost:" + port + "/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }

  // If a test needs a FakeApplication, running TestServer, and Selenium
  // InternetExplorer driver use "new InternetExplorer":
  "The InternetExplorer function" must {
    "provide a FakeApplication" in new InternetExplorer(fakeApp("ehcacheplugin" -> "disabled")) {
      app.configuration.getString("ehcacheplugin") mustBe Some("disabled")
    }
    "make the FakeApplication available implicitly" in new InternetExplorer(fakeApp("ehcacheplugin" -> "disabled")) {
      getConfig("ehcacheplugin") mustBe Some("disabled")
    }
    "start the FakeApplication" in new InternetExplorer(fakeApp("ehcacheplugin" -> "disabled")) {
      Play.maybeApplication mustBe Some(app)
    }
    import Helpers._
    "send 404 on a bad request" in new InternetExplorer {
      import java.net._
      val url = new URL("http://localhost:" + port + "/boom")
      val con = url.openConnection().asInstanceOf[HttpURLConnection]
      try con.getResponseCode mustBe 404
      finally con.disconnect()
    }
    "provide a web driver" in new InternetExplorer(fakeApp()) {
      go to ("http://localhost:" + port + "/testing")
      pageTitle mustBe "Test Page"
      click on find(name("b")).value
      eventually { pageTitle mustBe "scalatest" }
    }
  }

  // If a test does not need any special fixtures, just 
  // write "in { () => ..."
  "Any old thing" must {
    "be doable without much boilerplate" in { () =>
       1 + 1 mustEqual 2
     }
  }
}
```


##Testing a template
Since a template is a standard Scala function, you can execute it from your test, and check the result:

```scala
"render index template" in new App {
  val html = views.html.index("Coco")

  contentAsString(html) must include ("Hello Coco")
}
```


##Testing a controller
You can call any `Action` code by providing a [`FakeRequest`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/FakeRequest.html):

```scala
import scala.concurrent.Future

import org.scalatest._
import org.scalatestplus.play._

import play.api.mvc._
import play.api.test._
import play.api.test.Helpers._

class ExampleControllerSpec extends PlaySpec with Results {

  class TestController() extends Controller with ExampleController

  "Example Page#index" should {
    "should be valid" in {
      val controller = new TestController()
      val result: Future[Result] = controller.index().apply(FakeRequest())
      val bodyText: String = contentAsString(result)
      bodyText mustBe "ok"
    }
  }
}
```

Technically, you don’t need [`WithApplication`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WithApplication.html) here, although it wouldn’t hurt anything to have it.


##Testing the router
Instead of calling the `Action` yourself, you can let the `Router` do it:

```scala
"respond to the index Action" in new App(fakeApplication) {
  val Some(result) = route(FakeRequest(GET, "/Bob"))

  status(result) mustEqual OK
  contentType(result) mustEqual Some("text/html")
  charset(result) mustEqual Some("utf-8")
  contentAsString(result) must include ("Hello Bob")
}
```


##Testing a model
If you are using an SQL database, you can replace the database connection with an in-memory instance of an H2 database using `inMemoryDatabase`.

```scala
val appWithMemoryDatabase = FakeApplication(additionalConfiguration = inMemoryDatabase("test"))
"run an application" in new App(appWithMemoryDatabase) {

  val Some(macintosh) = Computer.findById(21)

  macintosh.name mustEqual "Macintosh"
  macintosh.introduced.value mustEqual "1984-01-24"
}
```


##Testing WS calls
If you are calling a web service, you can use [`WSTestClient`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/WsTestClient.html). There are two calls available, `wsCall` and `wsUrl` that will take a Call or a string, respectively. Note that they expect to be called in the context of `WithApplication`.

```scala
wsCall(controllers.routes.Application.index()).get()
wsUrl("http://localhost:9000").get()
```