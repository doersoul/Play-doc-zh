#集成Akka

[Akka](http://akka.io/) 使用 Actor 模型来提升抽象级别和提供一个更好的平台来构建正确的并发和可扩展应用程序。对于容错性，它采用‘Let it crash’模型，该模型被成功应用于电信工业来构建永不停机的自我修复的系统。Actors也提供了对透明抽象的分布，以及构建真正的可扩展和高容错性应用程序的基础。


##应用程序的actor系统
Akka可以和一些叫做actor系统的容器一起工作。一个actor系统管理着它配置的资源，以便运行它包含的actors。

一个Play应用程序会定义一个特殊的actor系统来给该应用程序使用。这个actor系统跟随着应用程序的整个生命周期，并且在应用程序重启时自动重启。

###编写 actors
要开始使用Akka, 你需要编写一个actor。下面是一个简单的actor，无论谁询问它，它都简单的返回hello。

```scala
import akka.actor._

object HelloActor {
  def props = Props[HelloActor]
  
  case class SayHello(name: String)
}

class HelloActor extends Actor {
  import HelloActor._
  
  def receive = {
    case SayHello(name: String) =>
      sender ! "Hello, " + name
  }
}
```

这个 actor 遵循几个Akka 惯例:

* 消息的 发送/接收, 或它的协议，都定义在它的伴生对象
* 在它的伴生对象上也定义一个`props`方法，它返回一个 props

###创建和使用actors
要创建和/或使用actor, 你需要一个`ActorSystem`。这个可以通过声明一个ActorSystem的依赖来获取, 就像:

```scala
import play.api.mvc._
import akka.actor._
import javax.inject._
  
import actors.HelloActor

@Singleton
class Application @Inject() (system: ActorSystem) extends Controller {

  val helloActor = system.actorOf(HelloActor.props, "hello-actor")

  //...
}
```

`actorOf` 方法用于创建一个新actor。注意我们声明这个控制器为单例模式。这是必须的，因为我们创建actor并存储对它的引用, 如果控制器不是单例的作用域, 这会意味着每次创建控制器时都创建一个新actor, 最后会抛出异常，因为你不能在同一个actor系统有两个同名的actors。

###向actors询问
你可以用actor做的最基本的事情是给它发送一个消息。当你发送一个消息到actor, 这里是没有响应的, 这是即发即忘的。这也称为*tell* 模式。

然而在web应用程序, *tell* 模式通常没有什么用, 因为 HTTP 是一个有请求和响应的协议。在这种情况下, 你更可能想要使用 *ask* 模式。 ask 模式返回一个`Future`, 你可以映射到你自己的result类型。

以下是使用我们的`HelloActor` 和ask模式的示例:

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext
import scala.concurrent.duration._
import akka.pattern.ask
implicit val timeout = 5.seconds

def sayHello(name: String) = Action.async {
  (helloActor ? SayHello(name)).mapTo[String].map { message =>
    Ok(message)
  }
}
```

有几件事情要注意:

* ask 模式需要导入, 然后才能为actor提供一个`?` 操作符。
* ask的返回类型是`Future[Any]`, 通常在你ask actor之后你要做的第一件事情是映射你期望的结果类型，这个要使用 `mapTo` 方法。
* 作用域中要有一个隐式timeout - ask模式必须有。如果 actor 需要更长的时间响应, 返回的future会成为一个超时错误。


##依赖注入 actors
如果你喜欢, 你可以为你的控制器和组件的依赖用 Guice实例化你的actors并绑定 actor引用到他们。

举例, 如果你想要一个actor，它依赖于Play Configuration, 你要这样做:

```scala
import akka.actor._
import javax.inject._
import play.api.Configuration

object ConfiguredActor {
  case object GetConfig
}

class ConfiguredActor @Inject() (configuration: Configuration) extends Actor {
  import ConfiguredActor._

  val config = configuration.getString("my.config").getOrElse("none")

  def receive = {
    case GetConfig =>
      sender() ! config
  }
}
```

Play提供一些助手来帮助actor绑定。这些允许actor自身可以被依赖注入, 并允许为actor注入到其它组件提供actor引用。要使用这些助手绑定一个actor, 创建一个如[依赖注入文档](https://playframework.com/documentation/2.4.x/ScalaDependencyInjection#Play-applications)中所述的模块, 然后混入[`AkkaGuiceSupport`](https://playframework.com/documentation/2.4.x/api/scala/play/api/libs/concurrent/AkkaGuiceSupport.html) 特质，再使用`bindActor` 方法来绑定 actor:

```scala
import com.google.inject.AbstractModule
import play.api.libs.concurrent.AkkaGuiceSupport

import actors.ConfiguredActor

class MyModule extends AbstractModule with AkkaGuiceSupport {
  def configure = {
    bindActor[ConfiguredActor]("configured-actor")
  }
}
```

这个 actor 会命名为`configured-actor`, 也会被用于注入`configured-actor` 名称。现在你可以在你的控制器或其它组件中依赖这个 actor:

```scala
import play.api.mvc._
import akka.actor._
import akka.pattern.ask
import akka.util.Timeout
import javax.inject._
import actors.ConfiguredActor._
import scala.concurrent.ExecutionContext
import scala.concurrent.duration._

@Singleton
class Application @Inject() (@Named("configured-actor") configuredActor: ActorRef)
                            (implicit ec: ExecutionContext) extends Controller {

  implicit val timeout: Timeout = 5.seconds

  def getConfig = Action.async {
    (configuredActor ? GetConfig).mapTo[String].map { message =>
      Ok(message)
    }
  }
}
```

###依赖注入 子actors
上述对于注入根actors很好, 但你创建的许多actors会是子actors，这些没有绑定到Play应用的生命周期, 并且可能还有许多状态传递给他们。

为了帮助子actors的依赖注入, Play 利用 Guice的 [AssistedInject](https://github.com/google/guice/wiki/AssistedInject) 提供支持。

假设你有下面的 actor, 它依赖 configuration 被注入, 加上一个 key:

```scala
import akka.actor._
import javax.inject._
import com.google.inject.assistedinject.Assisted
import play.api.Configuration

object ConfiguredChildActor {
  case object GetConfig

  trait Factory {
    def apply(key: String): Actor
  }
}

class ConfiguredChildActor @Inject() (configuration: Configuration,
    @Assisted key: String) extends Actor {
  import ConfiguredChildActor._

  val config = configuration.getString(key).getOrElse("none")

  def receive = {
    case GetConfig =>
      sender() ! config
  }
}
```

注意这个`key` 参数声明为`@Assisted`, 这个表明会是手动提供。

我们也定义一个`Factory` 特质, 它带有`key`参数, 并返回一个`Actor`。我们不用实现它, Guice会为我们做这个, 提供一个实现，它不仅仅传递我们的`key` 参数, 也定位`Configuration` 依赖和注入它。因为特质只返回`Actor`, 当测试这个actor时我们可以注入一个factory，它返回任何actor, 例如可以允许我们注入一个mocked 子 actor, 而非真实的那个。

现在, 依赖于这个的 actor 可以扩展 [`InjectedActorSupport`](https://playframework.com/documentation/2.4.x/api/scala/play/api/libs/concurrent/InjectedActorSupport.html), 和它可以依赖于我们创建的工厂:

```scala
import akka.actor._
import javax.inject._
import play.api.libs.concurrent.InjectedActorSupport

object ParentActor {
  case class GetChild(key: String)
}

class ParentActor @Inject() (
    childFactory: ConfiguredChildActor.Factory
) extends Actor with InjectedActorSupport {
  import ParentActor._

  def receive = {
    case GetChild(key: String) =>
      val child: ActorRef = injectedChild(childFactory(key), key)
      sender() ! child
  }
}
```

它使用 `injectedChild` 来创建和获得到子actor的引用, 传入 key。

最后, 我们需要绑定我们的actors。在我们的模块, 我们使用`bindActorFactory` 方法来绑定父actor, 并绑定子factory到子actor实现:

```scala
import com.google.inject.AbstractModule
import play.api.libs.concurrent.AkkaGuiceSupport

import actors._

class MyModule extends AbstractModule with AkkaGuiceSupport {
  def configure = {
    bindActor[ParentActor]("parent-actor")
    bindActorFactory[ConfiguredChildActor, ConfiguredChildActor.Factory]
  }
}
```

这会让Guice自动绑定一个`ConfiguredChildActor.Factory` 实例, 它会提供一个`Configuration` 到`ConfiguredChildActor`实例，当它被实例化时。


##配置
默认的actor系统配置是从Play应用程序的配置文件中读取的。举例, 要配置默认应用程序Actor系统的dispatcher, 添加以下几行到`conf/application.conf` 文件:

```scala
akka.actor.default-dispatcher.fork-join-executor.parallelism-max = 64
akka.actor.debug.receive = on
```

对于 Akka 日志配置, 参阅 [日志配置](https://playframework.com/documentation/2.4.x/SettingsLogger).

###更改配置前缀
如果你想使用其它Akka actor系统的`akka.*` 设置, 你可以让 Play从另一个位置加载它的 Akka设置。

```scala
play.akka.config = "my-akka"
```

现在会从前缀为`my-akka`的读取而非`akka` 前缀。

```scala
my-akka.actor.default-dispatcher.fork-join-executor.pool-size-max = 64
my-akka.actor.debug.receive = on
```

###自定义内置actor系统的名称
默认Play actor系统的名称是`application`。你可以通过`conf/application.conf`更改它:

```scala
play.akka.actor-system = "custom-name"
```

> **注意**: 如果你想要将你的Play应用程序ActorSystem放到Akka集群中，这个功能很有用。


##调度异步任务
你可以预定发送消息到actors并执行任务 (函数或`Runnable`)。你会取回一个`Cancellable` ，可以调用`cancel` 来取消相应操作的执行。.

举例, 要每300毫秒发送一个消息到`testActor` :

```scala
import scala.concurrent.duration._

val cancellable = system.scheduler.schedule(
  0.microseconds, 300.microseconds, testActor, "tick")
```

> **注意**: 这个例子使用定义在`scala.concurrent.duration`中的隐式转换，来转换数字到不同时间单位的`Duration` 对象。

类似地, 要在10毫秒后运行一个代码块:

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext
system.scheduler.scheduleOnce(10.milliseconds) {
  file.delete()
}
```


##使用你自己的Actor系统
虽然我们建议你使用内置的actor系统, 因为它设置了所有东西，如正确的类加载器, 生命周期钩子, 等等, 但没有什么阻止你使用自己的actor系统。但是，重要的是要确保你做到以下几点:

* 注册一个[stop hook](https://playframework.com/documentation/2.4.x/ScalaDependencyInjection#Stopping/cleaning-up) 来在Play关闭时关闭actor系统。
* 从Play [Environment](https://playframework.com/documentation/2.4.x/api/scala/play/api/Application.html) 传入正确的类加载器，否则Akka 不能找到你的应用程序的类
* 确保不管是你更改了Play用来读取akka配置的位置`play.akka.config` ，还是你不从默认的`akka` 配置读取你的akka配置，因为这会造成问题。如当系统尝试绑定到同样的远程端口。