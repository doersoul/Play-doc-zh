#Streaming HTTP 响应


##标准响应和`Content-Length` 标头
自从 HTTP 1.1开始, 服务器要保持单个连接开放并服务多个HTTP请求和响应, 服务器必须伴随着响应发送合适的`Content-Length` HTTP标头。 

默认情况下, 当你发送一个简单的result时，并不会指定一个`Content-Length` 标头，如:

```scala
def index = Action {
  Ok("Hello World")
}
```

当然, 因为你要发送的内容是熟知的, Play能够为你计算出内容的大小，以生成适当的标头。

> 要注意基于文本的内容并没有像你所见的这么简单, 因为`Content-Length` 标头的计算必须依照使用的字符编码将字符转换为字节。

事实上, 我们之前看到的响应体都是由`play.api.libs.iteratee.Enumerator` 所指定:

```scala
def index = Action {
  Result(
    header = ResponseHeader(200),
    body = Enumerator("Hello World")
  )
}
```

这意味着为正确计算`Content-Length` 标头, Play必须读取整个枚举器（enumerator）并加载它的内容到内存。 


##发送大量数据
如果对于简单的枚举器来说，加载所有内容到内存并不是问题，但是大数据集呢? 比如我们要返回一个大文件到web客户端。

让我们先看看如何创建一个`Enumerator[Array[Byte]]` 来枚举文件内容:

```scala
val file = new java.io.File("/tmp/fileToServe.pdf")
val fileContent: Enumerator[Array[Byte]] = Enumerator.fromFile(file)
```

这看起来是不是很简单? 接着我们用这个枚举器来指定响应体:

```scala
def index = Action {

  val file = new java.io.File("/tmp/fileToServe.pdf")
  val fileContent: Enumerator[Array[Byte]] = Enumerator.fromFile(file)    
    
  Result(
    header = ResponseHeader(200),
    body = fileContent
  )
}
```

实际上这里有一个问题。我们没有指定`Content-Length` 标头, Play 会自己计算, 唯一的方式就是读取整个枚举器的内容并加载到内存，然后再计算响应大小。

问题在于我们并不想加载整个大文件到内存中。为了避免这个, 我们只需要自己指定`Content-Length` 标头。

```scala
def index = Action {

  val file = new java.io.File("/tmp/fileToServe.pdf")
  val fileContent: Enumerator[Array[Byte]] = Enumerator.fromFile(file)    
    
  Result(
    header = ResponseHeader(200, Map(CONTENT_LENGTH -> file.length.toString)),
    body = fileContent
  )
}
```

用这种方式时 Play 会以惰性方式来读取这个枚举器, 在每个数据块可用时拷贝它到HTTP响应中。


##提供文件
当然, Play 提供简单易用的助手方法来处理提供本地文件的任务:

```scala
def index = Action {
  Ok.sendFile(new java.io.File("/tmp/fileToServe.pdf"))
}
```

这个助手方法也会根据文件名计算出`Content-Type` 标头, 并添加`Content-Disposition` 标头来指定 web浏览器如何处理这个响应。默认是通过在HTTP响应中添加`Content-Disposition: attachment; filename=fileToServe.pdf` 标头，来告诉web浏览器下载这个文件。

你也可以提供你自己的文件名:

```scala
def index = Action {
  Ok.sendFile(
    content = new java.io.File("/tmp/fileToServe.pdf"),
    fileName = _ => "termsOfService.pdf"
  )
}
```

如果你想内联提供这个文件:

```scala
def index = Action {
  Ok.sendFile(
    content = new java.io.File("/tmp/fileToServe.pdf"),
    inline = true
  )
}
```

现在不需要指定文件名了，因为web浏览器根本不会尝试下载, 只是将文件内容显示在web浏览器窗口中。这对web浏览器原生支持的内容类型很有用, 如文本, HTML或图像。


##分块响应
目前为止, 流处理文件内容工作得很好，主要是由于我们能够在流处理前计算出内容的长度。但对于动态计算的内容,还没有得出可用的内容长度呢?

对于这类的响应，我们不得不使用 **分块传输编码（Chunked transfer encoding）**。 

> **分块传输编码（Chunked transfer encoding）** 是一种定义在 Hypertext Transfer Protocol (HTTP) 1.1版本中的数据传输机制，其中web服务器会将内容分块处理。它使用`Transfer-Encoding` HTTP 响应标头而非 `Content-Length` 标头, 否则就需要用这个标头。由于`Content-Length` 标头没有被使用, 服务器在开始传输响应到客户端（通常是web浏览器）之前，无须知道内容的长度。Web 服务器在知道内容的总长度之前，就可以开始传输动态生成的内容响应。

> 在发送每个数据块前，都会先发送块的大小, 因此当客户端完成接收数据块时可以知道是否已完整收到。数据传输会在收到一个长度为零的数据块后终止。

> [https://en.wikipedia.org/wiki/Chunked_transfer_encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding) 

这样的好处是我们可以 **实时** 提供数据, 这意味着一旦数据块可用我们便会发送。缺点是由于web浏览器不知道内容大小，无法显示正确的下载进度条。

假如我们有一个服务，它提供一个动态`InputStream` 来计算一些数据。首先我们要为这个流创建一个`Enumerator` :

```scala
val data = getDataStream
val dataContent: Enumerator[Array[Byte]] = Enumerator.fromStream(data)
```

现在就可以使用`Ok.chunked` 来流处理这些数据了:

```scala
def index = Action {
  val data = getDataStream
  val dataContent: Enumerator[Array[Byte]] = Enumerator.fromStream(data)
  
  Ok.chunked(dataContent)
}
```

当然, 我们可以使用任何`Enumerator` 来指定分块数据:

```scala
def index = Action {
  Ok.chunked(
    Enumerator("kiki", "foo", "bar").andThen(Enumerator.eof)
  )
}
```

我们可以检查服务器发回的HTTP响应:

```http
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked

4
kiki
3
foo
3
bar
0
```

我们得到三个数据块，最后还跟着一个空数据块来结束响应。