#Comet sockets


##使用分块响应来创建 Comet sockets
**分块响应** 的一个好处是用来创建 Comet sockets。一个 Comet socket 只是一块仅包含`<script>` 元素的 `text/html` 响应。在每个块中我们写入一个`<script>` 标签，它会被Web浏览器立即执行。用这种方式我们可以从服务器发送各种事件到浏览器: 每条信息被包装到到一个`<script>` 标签，它会调用一个 JavaScript 回调函数, 再将它写入响应块中。

让我们首先写个例子来验证这个概念: 枚举器生成`<script>` 标签，并且每个都调用浏览器`console.log` JavaScript 函数:

```scala
def comet = Action {
  val events = Enumerator(
    """<script>console.log('kiki')</script>""",
    """<script>console.log('foo')</script>""",
    """<script>console.log('bar')</script>"""
  )
  Ok.chunked(events).as(HTML)
}
```

如果你从web浏览器中运行这个action, 你会看到三个事件在浏览器控制台中打印日志。

我们可以用一种更好的方式来写前面的例子，通过使用`play.api.libs.iteratee.Enumeratee`，它只是一个能转换一个 `Enumerator[A]` 到另一个`Enumerator[B]` 的适配器。下面我们使用它将标准信息包装到`<script>` 标签中:

```scala
import play.twirl.api.Html

// Transform a String message into an Html script tag
val toCometMessage = Enumeratee.map[String] { data =>
  Html("""<script>console.log('""" + data + """')</script>""")
}

def comet = Action {
  val events = Enumerator("kiki", "foo", "bar")
  Ok.chunked(events &> toCometMessage)
}
```

> 提示:  `events &> toCometMessage` 只是`events.through(toCometMessage)` 的另一种表达方式。


##使用`play.api.libs.Comet` 助手方法
Play提供一个Comet助手方法来处理这些 Comet分块流，效果与上面所写的方法基本一样。

> **注意**: 事实上它做得更多, 像为了浏览器的兼容性而提供一个初始的空缓冲数据, 以及它支持字符串和JSON信息。它还可以通过type类的扩展来支持更多的消息类型。

让我们用它来重写前面的例子:

```scala
def comet = Action {
  val events = Enumerator("kiki", "foo", "bar")
  Ok.chunked(events &> Comet(callback = "console.log"))
}
```


##永远的iframe技术
写Comet socket的标准技术是在一个HTML`iframe` 中加载无限分块comet响应，并指定一个回调函数来调用父级frame:

```scala
def comet = Action {
  val events = Enumerator("kiki", "foo", "bar")
  Ok.chunked(events &> Comet(callback = "parent.cometMessage"))
}
```

HTML 页面如下:

```html
<script type="text/javascript">
  var cometMessage = function(event) {
    console.log('Received event: ' + event)
  }
</script>

<iframe src="/comet"></iframe>
```