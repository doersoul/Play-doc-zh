#请求体解析器

##什么是请求体解析器?
一个HTTP PUT 或 POST 请求中包含一个请求体。这个请求体可以是任何格式, 指定在`Content-Type` 请求标头中。在Play中,一个 **请求体解析器** 转换这个请求体到一个Scala值。

然而一个HTTP请求的请求体可能非常大， **请求体解析器** 不能等到所有数据集都加载到内存再去解析它。`BodyParser[A]` 基本上是一个`Iteratee[Array[Byte],A]`, 也就是说它是一块一块的接收字节数据(只要web浏览器在上传数据)并计算类型`A` 的值作为结果。

让我们来看些例子。

* 一个 **text** 请求体解析器能拼接逐块的字节数据成一个字符串, 并把计算的字符串作为结果(`Iteratee[Array[Byte],String]`)。
* 一个 **file** 请求体解析器能保存逐块的字节数据块到一个本地文件, 并把`java.io.File` 引用作为结果(`Iteratee[Array[Byte],File]`)。
* 一个 **s3** 请求体解析器能推送逐块的字节数据到 Amazon S3 并把 S3 对象id作为结果(`Iteratee[Array[Byte],S3ObjectId]`)。

此外，一个 **请求体解析器** 在它开始解析请求体之前已经访问了HTTP请求标头, 因此有机会做一些预先检查。例如,一个请求体解析器可以检查一些HTTP标头是否正确设置, 或检查用户尝试上传一个大文件时是否有这个权限。

> **注意**: 这就是为什么请求体解析器不是一个真正的`Iteratee[Array[Byte],A]` ，更确切地说是`Iteratee[Array[Byte],Either[Result,A]]`, 也就是说如果它在无法为请求体计算出正确的值时，它自已可以直接发送HTTP结果 (通常是 `400 BAD_REQUEST`, `412 PRECONDITION_FAILED` 或者 `413 REQUEST_ENTITY_TOO_LARGE`) 。

一旦请求体解析器完成它的工作并返回一个类型为`A` 的值, 相应的 `Action` 函数就会被执行，计算出来的请求体值也已传递到请求中。


##更多关于 Actions的内容
前面我们说到`Action` 是一个`Request => Result` 函数。这不完全对。让我们更深入地看`Action` 特质:

```scala
trait Action[A] extends (Request[A] => Result) {
  def parser: BodyParser[A]
}
```

首先我们看到有一个泛型类型`A`, 然后一个action必须定义一个`BodyParser[A]`。`Request[A]` 定义为:

```scala
trait Request[+A] extends RequestHeader {
  def body: A
}
```

`A` 类型是请求体的类型。我们可以使用任何Scala类型作为请求体, 例如`String`, `NodeSeq`, `Array[Byte]`, `JsonValue`, 或者 `java.io.File`, 只要我们有一个请求体解析器能够处理它。

总的来说,一个`Action[A]` 使用一个`BodyParser[A]` 从HTTP请求取回类型`A`的值 , 并构建一个`Request[A]` 对象传递给 action 代码。


##默认请求体解析器: AnyContent
在我们前面的例子中还没有指定过请求体解析器。那么它是如何工作的? 如果你不指定自己的请求体解析器, Play会使用默认的, 它把请求体作为一个`play.api.mvc.AnyContent` 实例处理。

这个请求体检查`Content-Type` 标头以决定要处理的请求体类型:

* **text/plain**: `String`
* **application/json**: `JsValue`
* **application/xml**, **text/xml** 或 **application/XXX+xml**: `NodeSeq`
* **application/form-url-encoded**: `Map[String, Seq[String]]`
* **multipart/form-data**: `MultipartFormData[TemporaryFile]`
* 任何其它content type: RawBuffer

举例:

```scala
def save = Action { request =>
  val body: AnyContent = request.body
  val textBody: Option[String] = body.asText

  // Expecting text body
  textBody.map { text =>
    Ok("Got: " + text)
  }.getOrElse {
    BadRequest("Expecting text/plain request body")
  }
}
```


##指定一个请求体解析器
Play中可用的请求体解析器定义在`play.api.mvc.BodyParsers.parse` 中。

例如, 定义一个处理文本类型请求体的Action (就像前面的示例一样):

```scala
def save = Action(parse.text) { request =>
  Ok("Got: " + request.body)
}
```

知道代码为何更简单了吗? 这是因为如果发生了错误，`parse.text` 请求体解析器发送一个`400 BAD_REQUEST` 响应。在我们的action代码没有必要再次进行检查, 我们可以安全地认为`request.body` 包含有效的`String` 请求体。

另外我们可以用:

```scala
def save = Action(parse.tolerantText) { request =>
  Ok("Got: " + request.body)
}
```

这个不检查 `Content-Type` 标头，而总是加载请求体为`String`。

> 提示: Play中所有请求体解析器都提供有一个`tolerant` 样式的方法。

这是另一个例子, 它将把请求体存储为一个文件:

```scala
def save = Action(parse.file(to = new File("/tmp/upload"))) { request =>
  Ok("Saved the request content to " + request.body)
}
```


##组合请求体解析器
在前面的例子中, 所有的请求体都会存在同一文件中。这会有点问题，不是吗? 我们来编写另一个定制的请求体解析器，它会从请求Session中获取用户名, 并为每个用户生成不同的文件:

```scala
val storeInUserFile = parse.using { request =>
  request.session.get("username").map { user =>
    file(to = new File("/tmp/" + user + ".upload"))
  }.getOrElse {
    sys.error("You don't have the right to upload here")
  }
}

def save = Action(storeInUserFile) { request =>
  Ok("Saved the request content to " + request.body)
}
```

> 注意: 这里我们并不是真正写自己的请求体解析器, 只不过是组合了现有的。通常这样足够了，可以应付多数情况。从头开始写一个请求体解析器会在高级应用部分讲到。


##最大内容长度
基于文本的请求体解析器(如 **text**, **json**, **xml** 或 **formUrlEncoded**) 会有一个最大内容长度，因为他们要加载所有内容到内存中。默认情况下, 他们可解析的最大内容长度是100KB。也可通过在`application.conf` 文件中指定`play.http.parser.maxMemoryBuffer` 属性来重写它:

```scala
play.http.parser.maxMemoryBuffer=128K
```

为解析在磁盘上缓冲的内容, 如raw 解析器或 `multipart/form-data`, 最大内容长度是使用`play.http.parser.maxDiskBuffer`属性来指定, 它默认是10MB。`multipart/form-data parser` 还为数据字段的总和强制文本最大长度属性。

你也可以为给定的action重写默认的最大长度:

```scala
// 仅接受10KB数据
def save = Action(parse.text(maxLength = 1024 * 10)) { request =>
  Ok("Got: " + text)
}
```

你还可以用`maxLength` 包装任何请求体解析器:

```scala
// 仅接受10KB数据
def save = Action(parse.maxLength(1024 * 10, storeInUserFile)) { request =>
  Ok("Saved the request content to " + request.body)
}
```