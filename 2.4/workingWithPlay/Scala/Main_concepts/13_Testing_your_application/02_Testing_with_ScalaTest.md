#Testing your application with ScalaTest

Writing tests for your application can be an involved process. Play provides helpers and application stubs, and ScalaTest provides an integration library, [ScalaTest + Play](http://scalatest.org/plus/play), to make testing your application as easy as possible.


##Overview
The location for tests is in the “test” folder.
You can run tests from the Play console.

* To run all tests, run `test`.
* To run only one test class, run `test-only` followed by the name of the class, i.e., `test-only my.namespace.MySpec`.
* To run only the tests that have failed, run `test-quick`.
* To run tests continually, run a command with a tilde in front, i.e. `~test-quick`.
* To access test helpers such as `FakeApplication` in console, run `test:console`.

Testing in Play is based on SBT, and a full description is available in the [testing SBT](http://www.scala-sbt.org/0.13.0/docs/Detailed-Topics/Testing) chapter.


##Using ScalaTest + Play
To use *ScalaTest + Play*, you’ll need to add it to your build, by changing `build.sbt` like this:

```sbt
libraryDependencies ++= Seq(
  "org.scalatest" %% "scalatest" % "2.2.1" % "test",
  "org.scalatestplus" %% "play" % "1.4.0-M3" % "test",
)
```

You do not need to add ScalaTest to your build explicitly. The proper version of ScalaTest will be brought in automatically as a transitive dependency of ScalaTest + Play. You will, however, need to select a version of ScalaTest + Play that matches your Play version. You can do so by checking the [Versions, Versions, Versions](http://www.scalatest.org/plus/play/versions) page for *ScalaTest + Play*.

In [ScalaTest + Play](http://scalatest.org/plus/play), you define test classes by extending the [`PlaySpec`](http://doc.scalatest.org/plus-play/1.0.0/index.html#org.scalatestplus.play.PlaySpec) trait. Here’s an example:

```scala
import collection.mutable.Stack
import org.scalatestplus.play._

class StackSpec extends PlaySpec {

  "A Stack" must {
    "pop values in last-in-first-out order" in {
      val stack = new Stack[Int]
      stack.push(1)
      stack.push(2)
      stack.pop() mustBe 2
      stack.pop() mustBe 1
    }
    "throw NoSuchElementException if an empty stack is popped" in {
      val emptyStack = new Stack[Int]
      a [NoSuchElementException] must be thrownBy {
        emptyStack.pop()
      }
    }
  }
}
```

You can alternatively [define your own base classes](http://scalatest.org/user_guide/defining_base_classes) instead of using `PlaySpec`.

You can run your tests with Play itself, or in IntelliJ IDEA (using the [Scala plugin](https://blog.jetbrains.com/scala/)) or in Eclipse (using the [Scala IDE](http://scala-ide.org/) and the [ScalaTest Eclipse plugin](http://scalatest.org/user_guide/using_scalatest_with_eclipse)). Please see the [IDE page](https://playframework.com/documentation/2.4.x/IDE) for more details.

###Matchers
`PlaySpec` mixes in ScalaTest’s [`MustMatchers`](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.MustMatchers), so you can write assertions using ScalaTest’s matchers DSL:

```scala
import play.api.test.Helpers._

"Hello world" must endWith ("world")
```

For more information, see the documentation for [`MustMatchers`](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.MustMatchers).

###Mockito
You can use mocks to isolate unit tests against external dependencies. For example, if your class depends on an external `DataService` class, you can feed appropriate data to your class without instantiating a `DataService` object.

ScalaTest provides integration with [Mockito](https://github.com/mockito/mockito) via its [`MockitoSugar`](http://doc.scalatest.org/2.1.5/index.html#org.scalatest.mock.MockitoSugar) trait.

To use Mockito, mix `MockitoSugar` into your test class and then use the Mockito library to mock dependencies:

```scala
case class Data(retrievalDate: java.util.Date)

trait DataService {
  def findData: Data
}
import org.scalatest._
import org.scalatest.mock.MockitoSugar
import org.scalatestplus.play._

import org.mockito.Mockito._

class ExampleMockitoSpec extends PlaySpec with MockitoSugar {

  "MyService#isDailyData" should {
    "return true if the data is from today" in {
      val mockDataService = mock[DataService]
      when(mockDataService.findData) thenReturn Data(new java.util.Date())

      val myService = new MyService() {
        override def dataService = mockDataService
      }

      val actual = myService.isDailyData
      actual mustBe true
    }
  }
}
```

Mocking is especially useful for testing the public methods of classes. Mocking objects and private methods is possible, but considerably harder.


##Unit Testing Models
Play does not require models to use a particular database data access layer. However, if the application uses Anorm or Slick, then frequently the Model will have a reference to database access internally.

```scala
import anorm._
import anorm.SqlParser._

case class User(id: String, name: String, email: String) {
   def roles = DB.withConnection { implicit connection =>
      ...
    }
}
```

For unit testing, this approach can make mocking out the `roles` method tricky.

A common approach is to keep the models isolated from the database and as much logic as possible, and abstract database access behind a repository layer.

```scala
case class Role(name:String)

case class User(id: String, name: String, email:String)
trait UserRepository {
  def roles(user:User) : Set[Role]
}
class AnormUserRepository extends UserRepository {
  import anorm._
  import anorm.SqlParser._

  def roles(user:User) : Set[Role] = {
    ...
  }
}
```

and then access them through services:

```scala
class UserService(userRepository : UserRepository) {

  def isAdmin(user:User) : Boolean = {
    userRepository.roles(user).contains(Role("ADMIN"))
  }
}
```

In this way, the `isAdmin` method can be tested by mocking out the `UserRepository` reference and passing it into the service:

```scala
class UserServiceSpec extends PlaySpec with MockitoSugar {

  "UserService#isAdmin" should {
    "be true when the role is admin" in {
      val userRepository = mock[UserRepository]
      when(userRepository.roles(any[User])) thenReturn Set(Role("ADMIN"))

      val userService = new UserService(userRepository)

      val actual = userService.isAdmin(User("11", "Steve", "user@example.org"))
      actual mustBe true
    }
  }
}
```


##Unit Testing Controllers
When defining controllers as objects, they can be trickier to unit test. In Play this can be alleviated by [dependency injection](https://playframework.com/documentation/2.4.x/ScalaDependencyInjection). Another way to finesse unit testing with a controller declared as a object is to use a trait with an [explicitly typed self reference](http://www.naildrivin5.com/scalatour/wiki_pages/ExplcitlyTypedSelfReferences) to the controller:

```scala
trait ExampleController {
  this: Controller =>

  def index() = Action {
    Ok("ok")
  }
}

object ExampleController extends Controller with ExampleController
```

and then test the trait:

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

When testing POST requests with, for example, JSON bodies, you won’t be able to
use the pattern shown above (`apply(fakeRequest)`); instead you should use `call()` on the `testController`:

```scala
trait WithControllerAndRequest {
  val testController = new Controller with ApiController

  def fakeRequest(method: String = "GET", route: String = "/") = FakeRequest(method, route)
    .withHeaders(
      ("Date", "2014-10-05T22:00:00"),
      ("Authorization", "username=bob;hash=foobar==")
  )
}

"REST API" should {
  "create a new user" in new WithControllerAndRequest {
    val request = fakeRequest("POST", "/user").withJsonBody(Json.parse(
      s"""{"first_name": "Alice",
        |  "last_name": "Doe",
        |  "credentials": {
        |    "username": "alice",
        |    "password": "secret"
        |  }
        |}""".stripMargin))
    val apiResult = call(testController.createUser, request)
    status(apiResult) mustEqual CREATED
    val jsonResult = contentAsJson(apiResult)
    ObjectId.isValid((jsonResult \ "id").as[String]) mustBe true

    // now get the real thing from the DB and check it was created with the correct values:
    val newbie = Dao().findByUsername("alice").get
    newbie.id.get.toString mustEqual (jsonResult \ "id").as[String]
    newbie.firstName mustEqual "Alice"
  }
}
```


##Unit Testing EssentialAction
Testing [`Action`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Action.html) or [`Filter`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Filter.html) can require to test an an [`EssentialAction`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/EssentialAction.html) ([more information about what an EssentialAction is](https://playframework.com/documentation/2.4.x/HttpApi))

For this, the test [`Helpers.call`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/Helpers$.html#call) can be used like that:

```scala
class ExampleEssentialActionSpec extends PlaySpec {

  "An essential action" should {
    "can parse a JSON body" in {
      val action: EssentialAction = Action { request =>
        val value = (request.body.asJson.get \ "field").as[String]
        Ok(value)
      }

      val request = FakeRequest(POST, "/").withJsonBody(Json.parse("""{ "field": "value" }"""))

      val result = call(action, request)

      status(result) mustEqual OK
      contentAsString(result) mustEqual "value"
    }
  }
}
```