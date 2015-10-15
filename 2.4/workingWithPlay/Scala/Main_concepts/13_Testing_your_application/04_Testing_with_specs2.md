#用specs2测试你的应用程序

为你的应用程序编写测试是一个很耗时的过程。Play为你提供一个默认的测试框架, 并提供助手和应用程序存根来让测试你的应用程序尽可能的简单。


##概述
测试文件的位置在 “test” 文件夹。测试文件夹中有二个简单的测试文件，可以用做模板。

你可以从Play控制台运行测试。

* 要运行所有测试，运行`test`。
* 要仅运行某个测试类, 运行`test-only` 并在后面加上类名，例如`test-only my.namespace.MySpec`。
* 要仅运行会失败的测试类, 运行`test-quick`。
* 要持续运行测试, 在运行命令前面加上波浪线, 如`~test-quick` 。
* 要访问测试助手，如在控制台访问`FakeApplication` , 运行`test:console`。

Play中的测试是基本SBT的, 完整的描述请参阅 [测试 SBT](http://www.scala-sbt.org/0.13.0/docs/Detailed-Topics/Testing)章节。


##使用 specs2
要使用Play的 specs2支持, 添加 Play specs2 依赖到你的构建文件中，作为测试作用域依赖:

```scala
libraryDependencies += specs2 % Test
```

在 [specs2](https://etorreborre.github.io/specs2/), 测试被组织到Specifications中, 其包含运行基于不同代码路径测试的系统的示例。

Specifications扩展了[`Specification`](https://etorreborre.github.io/specs2/api/SPECS2-3.4/index.html#org.specs2.mutable.Specification) 特质，并使用 should/in 格式:

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

Specifications 可以在 IntelliJ IDEA (使用[Scala 插件](https://blog.jetbrains.com/scala/)) 或 Eclipse (使用 [Scala IDE](http://scala-ide.org/))中运行。请参阅 [IDE 页面](https://playframework.com/documentation/2.4.x/IDE) 详细了解。

注意: 基于[presentation compiler](https://scala-ide-portfolio.assembla.com/spaces/scala-ide/support/tickets/1001843-specs2-tests-with-junit-runner-are-not-recognized-if-there-is-package-directory-mismatch#/activity/ticket:)中的一个BUG, 在Eclipse中测试必须被定义为一个特定的格式:

* 包名字必须和目录路径完全一致。
* specification 必须用`@RunWith(classOf[JUnitRunner])`注解。

这里是一个在Eclipse中有效的specification:

```scala
package models // 这个文件必须存在于叫"models"的目录中

import org.specs2.mutable._
import org.specs2.runner._
import org.junit.runner._

@RunWith(classOf[JUnitRunner])
class ApplicationSpec extends Specification {
  ...
}
```

###Matchers
当你使用一个示例, 你必须返回一个示例结果。通常, 你会看到一个包含`must` 的声明:

```scala
"Hello world" must endWith("world")
```

尾随着`must` 关键词的表达式被叫做 [`matchers`](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Matchers.html)。Matchers返回示例结果, 通常是成功或失败。如果它不能返回结果则这个示例不会被编译。

最有用的 matchers 是[match results](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Matchers.html#out-of-the-box)。这些用来检查相等性, 判断部分和两者其一的结果, 甚至检测是否抛出异常。

还有[optional matchers](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.Matchers.html#optional) 允许在测试中使用XML和JSON匹配。

###Mockito
Mocks 用来隔离单元测试和外部依赖。例如, 如果你的类依赖于一个外部的 `DataService` 类, 你可以填充适当的数据到你的类，而无须实例化一个`DataService` 对象。

[Mockito](https://github.com/mockito/mockito) 已经集成到specs2中作为默认的[mocking 库](https://etorreborre.github.io/specs2/guide/SPECS2-3.4/org.specs2.guide.UseMockito.html)。

要使用Mockito, 添加以下import:

```scala
import org.specs2.mock._
```

你可以先模拟出引用类，如:

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

Mocking在测试类的公共方法是尤其有用。Mocking对象和私有方法也可以，但是非常困难。


##单元测试模块
Play不需要模块来使用特定的数据库数据访问层。然而，如果应用程序使用Anorm或Slick, 那么模型内部将有一个针对数据库访问的引用。

```scala
import anorm._
import anorm.SqlParser._

case class User(id: String, name: String, email: String) {
   def roles = DB.withConnection { implicit connection =>
      ...
    }
}
```

为单元测试, 这个方法可以有技巧的模拟出`roles` 方法。

一个通用的方法是保持模块从数据库中分离出来，并尽可能逻辑化，以及抽象出一个库层后的数据库访问。

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

然后通过服务访问他们:

```scala
class UserService(userRepository : UserRepository) {

  def isAdmin(user:User) : Boolean = {
    userRepository.roles(user).contains(Role("ADMIN"))
  }
}
```

以这种方式, `isAdmin` 方法可以通过模拟出`UserRepository` 引用并传递其到服务中来测试：

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


##单元测试控制器
当定义控制器为对象, 他们会更难以被单元测试。在Play这个可以通过[依赖注入](https://playframework.com/documentation/2.4.x/ScalaDependencyInjection)来缓解。另一种方式处理有控制器的单元测试，是这个控制器使用一个有[显式类型的自我引用](http://www.naildrivin5.com/scalatour/wiki_pages/ExplcitlyTypedSelfReferences)的特质:

```scala
trait ExampleController {
  this: Controller =>

  def index() = Action {
    Ok("ok")
  }
}

object ExampleController extends Controller with ExampleController
```

然后测试特质:

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


##单元测试 EssentialAction
测试 [`Action`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Action.html) 或 [`Filter`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/Filter.html) 需要测试一个[`EssentialAction`](https://playframework.com/documentation/2.4.x/api/scala/play/api/mvc/EssentialAction.html) ([关于什么是EssentialAction的更多信息](https://playframework.com/documentation/2.4.x/HttpApi))

对此, 这个测试[`Helpers.call`](https://playframework.com/documentation/2.4.x/api/scala/play/api/test/Helpers$.html#call) 可以像这样使用:

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