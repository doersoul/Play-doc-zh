#WebSockets

[WebSockets](https://en.wikipedia.org/wiki/WebSocket) 是一种可以在web浏览器使用的socket，它基于一种支持全双工通信的协议。只要在服务器和客户端之间存在一个活动的WebSocket连接，在任意时刻它们之间都可以收发消息。

兼容HTML5 的现代web浏览器通过一个JavaScript WebSocket API原生支持 WebSockets。然而 WebSockets 并不局限于在Web浏览器中使用, 有许多WebSocket 客户端库可用, 允许例如服务器之间通信, 还有原生移动应用也可使用WebSockets。在这些场景使用 WebSockets 有一个很多的优势，就是复用Play服务器已经使用的TCP端口。


##处理 WebSockets
目前为止, 我们都是使用`Action` 实例来处理标准HTTP请求，然后返回标准HTTP响应。WebSockets 则完全不同，它无法通过标准`Action` 来处理。

Play 提供了两种不同的内建机制来处理 WebSockets。第一种是使用actors, 第二种是使用iteratees。二种机制都可以使用提供给[WebSocket](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/mvc/WebSocket$.html)的构建器来访问。


##用actors处理 WebSockets
要用actor来处理WebSocket, 我们需要给Play 一个`akka.actor.Props` 对象来描述 actor，当Play接收WebSocket连接时它会创建这个actor。Play 会给我们一个`akka.actor.ActorRef` 来发送上行消息, 因此我们可以用它来创建`Props` 对象:

```scala
import play.api.mvc._
import play.api.Play.current

def socket = WebSocket.acceptWithActor[String, String] { request => out =>
  MyWebSocketActor.props(out)
}
```

这里我们发送消息的 actor如下:

```scala
import akka.actor._

object MyWebSocketActor {
  def props(out: ActorRef) = Props(new MyWebSocketActor(out))
}

class MyWebSocketActor(out: ActorRef) extends Actor {
  def receive = {
    case msg: String =>
      out ! ("I received your message: " + msg)
  }
}
```

任何从客户端接收到的消息都会被发送到actor（即out）, 并且任何发送给由Play提供的 actor的消息都会被发送给客户端。上面的actor简单地将每条从客户端接收的消息附加上`I received your message`，再返回给客户端。

###检测 WebSocket 何时关闭
当WebSocket关闭, Play会自动停止actor。这意味着你可以通过实现actors的`postStop` 方法来清理WebSocket可能使用的资源。举例:

```scala
override def postStop() = {
  someResource.close()
}
```

###关闭一个 WebSocket
当处理WebSocket的actor终止时，Play会自动关闭 WebSocket。因此, 要关闭WebSocket, 可发送一个`PoisonPill` 到你自己的actor:

```scala
import akka.actor.PoisonPill

self ! PoisonPill
```

###拒绝 WebSocket
有时候你可能想拒绝一个WebSocket请求, 例如, 如果用户必须是已验证的用户才能连接到WebSocket, 或者如果WebSocket关联到某些资源而通过路径中的id, 但那个id的资源不存在。Play 提供`tryAcceptWithActor` 来处理这种情况, 允许你返回一个 result (如 forbidden或not found)，或是返回一个处理WebSocket的actor:

```scala
import scala.concurrent.Future
import play.api.mvc._
import play.api.Play.current

def socket = WebSocket.tryAcceptWithActor[String, String] { request =>
  Future.successful(request.session.get("user") match {
    case None => Left(Forbidden)
    case Some(_) => Right(MyWebSocketActor.props)
  })
}
```

###处理不同类型的消息
目前为止我们看到的都只是在处理`String` 信息。Play也内建`Array[Byte]` 信息的处理, 和从`String` frames解析的`JsValue` 消息。你可以把这些作为类型参数传递给 WebSocket 的构建方法, 举例:

```scala
import play.api.mvc._
import play.api.libs.json._
import play.api.Play.current

def socket = WebSocket.acceptWithActor[JsValue, JsValue] { request => out =>
  MyWebSocketActor.props(out)
}
```

你可能已经注意到了这里有二个类型参数（虽然都是JsValue）, 这允许我们将传入到发送出去的信息处理为不同类型。通常这对低级的类型来说没有什么用, 但如果你解析消息到高级类型时就非常有用。

例如, 我们想接收JSON消息, 然后我们想解析传入的消息作为`InEvent` ，再格式化传出的消息为`OutEvent`。要做的第一件事是为创建为`InEvent` 和 `OutEvent` 类型创建JSON格式:

```scala
import play.api.libs.json._

implicit val inEventFormat = Json.format[InEvent]
implicit val outEventFormat = Json.format[OutEvent]
```

接着我们可以为这些类型创建 WebSocket `FrameFormatter`:

```scala
import play.api.mvc.WebSocket.FrameFormatter

implicit val inEventFrameFormatter = FrameFormatter.jsonFrame[InEvent]
implicit val outEventFrameFormatter = FrameFormatter.jsonFrame[OutEvent]

And finally, we can use these in our WebSocket:
import play.api.mvc._
import play.api.Play.current

def socket = WebSocket.acceptWithActor[InEvent, OutEvent] { request => out =>
  MyWebSocketActor.props(out)
}
```

现在我们的actor中, 我们会接收`InEvent`类型的消息, 和发送`OutEvent`类型的消息。


##用iteratees处理 WebSockets 
actors是一种更好的抽象来处理离散消息，而iteratees是一种更好的抽象来处理流。

要处理 WebSocket 请求, 使用一个`WebSocket` 而非`Action`:

```scala
import play.api.mvc._
import play.api.libs.iteratee._
import play.api.libs.concurrent.Execution.Implicits.defaultContext

def socket = WebSocket.using[String] { request =>

  // Log events to the console
  val in = Iteratee.foreach[String](println).map { _ =>
    println("Disconnected")
  }

  // Send a single 'Hello!' message
  val out = Enumerator("Hello!")

  (in, out)
}
```

一个 `WebSocket` 可以访问初始化WebSocket连接的那个请求的标头, 允许你接收标准标头和session数据。然而它无权访问请求体或HTTP响应。

当以这种方式构建`WebSocket` , 我们必须返回`in` 和`out` 两个通道。

* `in` 通道是一个`Iteratee[A,Unit]` (其中 A 是消息类型 - 这里用的是`String`) ，对于每条通知的消息都会收到。当socket在客户端关闭时它会由到`EOF` 。
* `out` 通道是一个`Enumerator[A]` ，它会生成发送给Web客户端的消息。它可以通过发送`EOF`在服务端关闭连接。

在本例，我们创建一个简单的iteratee，来打印每条消息到控制台。要发送消息, 我们创建一个简单 dummy 枚举器，它会发送一个单独的 **Hello!** 消息。

> **提示**: 你可以测试 WebSockets 在 [https://www.websocket.org/echo.html](https://www.websocket.org/echo.html)。只需要把location设置为`ws://localhost:9000` 。

让我们写另外一个示例，直接忽略输入的数据，在发送完 **Hello!** 消息后就关闭socket:

```scala
import play.api.mvc._
import play.api.libs.iteratee._

def socket = WebSocket.using[String] { request =>

  // Just ignore the input
  val in = Iteratee.ignore[String]

  // Send a single 'Hello!' message and close
  val out = Enumerator("Hello!").andThen(Enumerator.eof)

  (in, out)
}
```

这里是另一个例子，其中输入的数据被记录到标准输出，并利用`Concurrent.broadcast` 广播到客户端。

```scala
import play.api.mvc._
import play.api.libs.iteratee._
import play.api.libs.concurrent.Execution.Implicits.defaultContext

def socket =  WebSocket.using[String] { request =>

  // Concurrent.broadcast returns (Enumerator, Concurrent.Channel)
  val (out, channel) = Concurrent.broadcast[String]

  // log the message to stdout and send response back to client
  val in = Iteratee.foreach[String] {
    msg =>
      println(msg)
      // the Enumerator returned by Concurrent.broadcast subscribes to the channel and will
      // receive the pushed messages
      channel push("I received your message: " + msg)
  }
  (in,out)
}
```