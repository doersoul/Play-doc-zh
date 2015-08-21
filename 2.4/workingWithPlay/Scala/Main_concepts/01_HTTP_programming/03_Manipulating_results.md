#操纵Results

##更改默认的 `Content-Type`
result的content type是从你指定的响应体的Scala值中自动推断的。

举例:

```scala
val textResult = Ok("Hello World!")
```

会自动设置`Content-Type` 标头为`text/plain`, 而:

```scala
val xmlResult = Ok(<message>Hello World!</message>)
```

会设置 Content-Type 标头为`application/xml`.

> 提示: 这是由`play.api.http.ContentTypeOf` type类完成的。

这相当有用, 但有时候你想要更改它。只需使用result的`as(newContentType)` 方法来创建一个新的、类似的、带有不同的`Content-Type`标头的 result:

```scala
val htmlResult = Ok(<h1>Hello World!</h1>).as("text/html")
```

或甚至更好的方法, 使用:

```scala
val htmlResult2 = Ok(<h1>Hello World!</h1>).as(HTML)
```

> 注意: 使用`HTML`而非`"text/html"` 的好处是会为你自动处理字符集，并且实际Content-Type 标头会设置为`text/html; charset=utf-8`。我们会 [看到这一点](#Changing-the-charset-for-text-based-HTTP-responses).


##操纵HTTP标头
你也可以添加(或更新)任何HTTP标头到result:

```scala
val result = Ok("Hello World!").withHeaders(
  CACHE_CONTROL -> "max-age=3600",
  ETAG -> "xx")
```

请注意如果原来的result已经有一个标头了，设置一个新的HTTP标头会自动覆盖前面的值。


##设置和丢弃cookies
Cookies 只是一种特殊的HTTP标头，但我们提供了一系列的助手方法来简化操作。

你可以很容易地添加一个Cookie到HTTP响应中:

```scala
val result = Ok("Hello world").withCookies(
  Cookie("theme", "blue"))
```

此外, 要丢弃之前存储在Web浏览器中的Cookie:

```scala
val result2 = result.discardingCookies(DiscardingCookie("theme"))
```

你也可以在同一个响应中同时设置和移除cookies:

```scala
val result3 = result.withCookies(Cookie("theme", "blue")).discardingCookies(DiscardingCookie("skin"))
```


##<a name="Changing-the-charset-for-text-based-HTTP-responses"></a>改变基于文本的HTTP响应的字符集
对于基于文本的HTTP响应，正确处理字符集是非常重要的。Play为你处理它，并采用`utf-8` 为默认值(参阅[为什么使用utf-8](http://www.w3.org/International/questions/qa-choosing-encodings#useunicode))。

字符集既转换文本响应到相应的字节来通过网络套接字发送, 亦使用正确的`;charset=xxx`扩展来更新`Content-Type` 标头。

字符集通过`play.api.mvc.Codec` type类自动处理。只需导入一个隐式`play.api.mvc.Codec` 实例到当前作用域，以改变所有操作将会用到的字符集:

```scala
class Application extends Controller {

  implicit val myCustomCharset = Codec.javaSupported("iso-8859-1")

  def index = Action {
    Ok(<h1>Hello World!</h1>).as(HTML)
  }

}
```

这里因为作用域中有一个隐式字符集的值, 它会用于`Ok(...)` 方法来转换 XML 消息为`ISO-8859-1` 编码的字节，同时也生成`text/html; charset=iso-8859-1` Content-Type 标头。

现在如果你想知道`HTML` 方法是如何工作的, 以下是它的定义:

```scala
def HTML(implicit codec: Codec) = {
  "text/html; charset=" + codec.charset
}
```

如果你的API中需要用通用的方法处理字符集，你可以按上面同样的方法操作。