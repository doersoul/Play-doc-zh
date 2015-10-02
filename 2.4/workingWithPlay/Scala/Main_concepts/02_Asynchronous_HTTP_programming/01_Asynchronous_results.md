#处理异步results


##制造异步controllers
Play框架从一开始内部就是异步的。Play用非阻塞的方式异步处理每个请求。

默认配置就是调成异步controllers。换言之, 应用程序代码应该避免在controllers中阻塞操作, 例如, 让controller代码等待某个操作。常见的阻塞操作例子如 调用JDBC, 流处理（streaming） API, HTTP请求和长时间计算。

虽然可以在默认执行环境（execution context）增加数程数以允许阻塞型controllers处理更多并发请求, 但推荐的方法是保持controllers异步以使程序易于扩展和在负载较大的情况下保持响应。


##创建非阻塞 actions
因为Play的异步工作方式, action代码必须尽可能快, 例如非阻塞。所以在还没有生成结果的情况下，我们应该返回什么作为result呢? 响应是一个future result!

一个 `Future[Result]` 最终会兑换回一个`Result`类型的值。通过提供一个`Future[Result]` 而非一个普通的 `Result`, 我们能够快速生成结果而无须阻塞。一旦Promise的结果得到兑换，Play马上提供result。

web客户端在等待响应是还是阻塞的, 但在服务器端没有任何阻塞, 而且服务端资源可以服务其它客户端。



##如何创建一个`Future[Result]`
要创建一个`Future[Result]` ，我们首先需要另一个future: 这个future会给我们一个真实的值以计算result:

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext

val futurePIValue: Future[Double] = computePIAsynchronously()
val futureResult: Future[Result] = futurePIValue.map { pi =>
  Ok("PI value computed: " + pi)
}
```

所有Play的异步API调用返给你一个`Future`。不管你是在使用 `play.api.libs.WS` API调用一个外部web服务, 还是使用Akka调度异步任务，或者使用`play.api.libs.Akka`与actors通讯。

这里是一个简单的执行异步代码块的方式，以及获得`Future`:

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext

val futureInt: Future[Int] = scala.concurrent.Future {
  intensiveComputation()
}
```

> 注意: 理解哪个线程代码运行了futures非常重要。在以上二个代码段, 都导入了Play的默认执行环境。这是一个隐式参数，会被传入future API上所有接受回调的的方法中。执行环境（execution context）通常等价于一个线程池, 虽然不是一定的。

> 你不能神奇的通过简单把同步IO封装到一个`Future`来转换为异步的。如果你不能通过更改应用程序的架构来避免阻塞操作, 那么该操作总在某一时间点会被执行, 而这个线程会被阻塞。所以除了将操作封装到`Future`, 还必须配置它以运行在一个单独的执行环境，且这个执行环境已配置有足够的线程来处理预期并发。 查阅[理解Play线程池](https://www.playframework.com/documentation/2.4.x/ThreadPools) 以了解更多信息。

> 这对使用Actors来阻塞操作也非常有用。Actors提供了一个简洁的模型来处理超时和失败, 设置阻塞执行环境, 并且管理可能与该服务关联的任何状态。Actors还提供了像`ScatterGatherFirstCompletedRouter`这样的模式来 同时处理缓存和数据库请求，并且允许在后端服务器集群中远程执行。但使用一个Actor可能会过度了，主要还是取决于你想要的是什么。


##返回futures
目前为止我们使用`Action.apply` 构建方法来创建actions, 而要发送异步result我们得使用`Action.async`方法:

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext

def index = Action.async {
  val futureInt = scala.concurrent.Future { intensiveComputation() }
  futureInt.map(i => Ok("Got result: " + i))
}
```


##Actions默认是异步的
Play [actions](../01_HTTP_programming/01_Actions_Controllers_and_Results.md) 默认即是异步的。比如说, 在下面的controller代码中, 代码的`{ Ok(...) }` 部分不是controller的方法体。这是一个匿名函数，其传入到`Action` 对象的`apply` 方法, 用于创建一个`Action`类型的对象。你写的这个匿名函数会被调用并且它的result会被封装到一个`Future`。

```scala
val echo = Action { request =>
  Ok("Got request [" + request + "]")
}
```

> 注意: `Action.apply` 和`Action.async` 二者创建的`Action` 对象在内部会以同样方式处理。只有单个类型的`Action`, 它是异步的, 并不是两种类型(一个同步一个异步)。`.async` 构造器只是一个简单创建基于Actions的API的工厂并返回`Future`, 让非阻塞的代码更加容易编写。


##处理超时
合理的处理超时通常是有用的, 以避免出现问题时web浏览器阻塞和等待。你可以简单的组合promise和promise超时来处理这些情况:

```scala
import play.api.libs.concurrent.Execution.Implicits.defaultContext
import scala.concurrent.duration._

def index = Action.async {
  val futureInt = scala.concurrent.Future { intensiveComputation() }
  val timeoutFuture = play.api.libs.concurrent.Promise.timeout("Oops", 1.second)
  Future.firstCompletedOf(Seq(futureInt, timeoutFuture)).map {
    case i: Int => Ok("Got result: " + i)
    case t: String => InternalServerError(t)
  }
}
```