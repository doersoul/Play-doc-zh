#Testing your application with specs2

Writing tests for your application can be an involved process. Play provides a default test framework for you, and provides helpers and application stubs to make testing your application as easy as possible.


##Overview
The location for tests is in the “test” folder. There are two sample test files created in the test folder which can be used as templates.

You can run tests from the Play console.

* To run all tests, run `test`.
* To run only one test class, run `test-only` followed by the name of the class i.e. `test-only my.namespace.MySpec`.
* To run only the tests that have failed, run `test-quick`.
* To run tests continually, run a command with a tilde in front, i.e. `~test-quick`.
* To access test helpers such as `FakeApplication` in console, run `test:console`.

Testing in Play is based on SBT, and a full description is available in the [testing SBT](http://www.scala-sbt.org/0.13.0/docs/Detailed-Topics/Testing) chapter.


##Using specs2
To use Play’s specs2 support, add the Play specs2 dependency to your build as a test scoped dependency:

```scala
libraryDependencies += specs2 % Test
```

In [specs2](https://etorreborre.github.io/specs2/), tests are organized into specifications, which contain examples which run the system under test through various different code paths.

Specifications extend the [`Specification`](https://etorreborre.github.io/specs2/api/SPECS2-3.4/index.html#org.specs2.mutable.Specification) trait and are using the should/in format:

```scala
import org.specs2.mutable._

class HelloWorldSpec extends Specification {

  "The 'Hello world' string" should {
    "contain 11 characters" in {
      "Hello world" must have size(11)
    }
    "start with 'Hello'" in {
      "Hello world" must startWith("Hello")
    }
    "end with 'world'" in {
      "Hello world" must endWith("world")
    }
  }
}
```

Specifications can be run in either IntelliJ IDEA (using the [Scala plugin](https://blog.jetbrains.com/scala/)) or in Eclipse (using the [Scala IDE](http://scala-ide.org/)). Please see the [IDE page](https://playframework.com/documentation/2.4.x/IDE) for more details.

NOTE: Due to a bug in the [presentation compiler](https://scala-ide-portfolio.assembla.com/spaces/scala-ide/support/tickets/1001843-specs2-tests-with-junit-runner-are-not-recognized-if-there-is-package-directory-mismatch#/activity/ticket:), tests must be defined in a specific format to work with Eclipse:

* The package must be exactly the same as the directory path.
* The specification must be annotated with `@RunWith(classOf[JUnitRunner])`.

Here is a valid specification for Eclipse:

```scala
package models // this file must be in a directory called "models"

import org.specs2.mutable._
import org.specs2.runner._
import org.junit.runner._

@RunWith(classOf[JUnitRunner])
class ApplicationSpec extends Specification {
  ...
}
```

###Matchers
When you use an example, you must return an example result. Usually, you will see a statement containing a `must`:

```scala
"Hello world" must endWith("world")
```

The expression that follows the `must` keyword are known as [`matchers`](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Matchers.html). Matchers return an example result, typically Success or Failure. The example will not compile if it does not return a result.

The most useful matchers are the [match results](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Matchers.html#out-of-the-box). These are used to check for equality, determine the result of Option and Either, and even check if exceptions are thrown.

There are also [optional matchers](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Matchers.html#optional) that allow for XML and JSON matching in tests.

###Mockito
Mocks are used to isolate unit tests against external dependencies. For example, if your class depends on an external `DataService` class, you can feed appropriate data to your class without instantiating a `DataService` object.

[Mockito](https://github.com/mockito/mockito) is integrated into specs2 as the default [mocking library](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.UseMockito.html).

To use Mockito, add the following import:

```scala
import org.specs2.mock._
```

You can mock out references to classes like so:

```scala
trait DataService {
  def findData: Data
}

case class Data(retrievalDate: java.util.Date)
import org.specs2.mock._
import org.specs2.mutable._

import java.util._

class ExampleMockitoSpec extends Specification with Mockito {

  "MyService#isDailyData" should {
    "return true if the data is from today" in {
      val mockDataService = mock[DataService]
      mockDataService.findData returns Data(retrievalDate = new java.util.Date())

      val myService = new MyService() {
        override def dataService = mockDataService
      }

      val actual = myService.isDailyData
      actual must equalTo(true)
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
object UserServiceSpec extends Specification with Mockito {

  "UserService#isAdmin" should {
    "be true when the role is admin" in {
      val userRepository = mock[UserRepository]
      userRepository.roles(any[User]) returns Set(Role("ADMIN"))

      val userService = new UserService(userRepository)
      val actual = userService.isAdmin(User("11", "Steve", "user@example.org"))
      actual must beTrue
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
import play.api.mvc._
import play.api.test._
import scala.concurrent.Future

object ExampleControllerSpec extends PlaySpecification with Results {

  class TestController() extends Controller with ExampleController

  "Example Page#index" should {
    "should be valid" in {
      val controller = new TestController()
      val result: Future[Result] = controller.index().apply(FakeRequest())
      val bodyText: String = contentAsString(result)
      bodyText must be equalTo "ok"
    }
  }
}
```


##Unit Testing EssentialAction
Testing [`Action`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Action.html) or [`Filter`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Filter.html) can require to test an [`EssentialAction`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/EssentialAction.html) ([more information about what an EssentialAction is](https://playframework.com/documentation/2.4.x/HttpApi))

For this, the test [`Helpers.call`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/Helpers$.html#call) can be used like that:

```scala
object ExampleEssentialActionSpec extends PlaySpecification {

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