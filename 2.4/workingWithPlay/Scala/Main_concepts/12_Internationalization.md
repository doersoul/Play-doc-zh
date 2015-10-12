#Messages和国际化


##通过你的应用程序支持指定语言
可以按照有效的 **ISO 639-2 语言代码** 来指定有效的语言代码, 还可以按 **ISO 3166-1 alpha-2 国家代码**, 如`fr` 或`en-US`。

你需要在 `conf/application.conf` 文件中指定语言支持:

```scala
play.i18n.langs = [ "en", "en-US", "fr" ]
```


##外部化 messages
你可以在`conf/messages.xxx` 文件中外部化messages。

默认`conf/messages` 文件匹配所有语言。另外你可以指定特定的语言message文件，如`conf/messages.fr` or `conf/messages.en-US`.

然后你可以用`play.api.i18n.Messages` 对象检索messages:

```scala
val title = Messages("home.title")
```

所有国际化API调用带有一个隐式`play.api.i18n.Messages` 参数，其中参数值从当前作用域检出。这个隐式值包含要使用的语言和 (基本上) 国际化message。

获取如一个隐式值的简单方式是使用`I18nSupport` 特质。比如你可以像下面的控制器那样使用它:

```scala
import play.api.i18n.I18nSupport
class MyController(val messagesApi: MessagesApi) extends Controller with I18nSupport {
  // ...
```

`I18nSupport` 特质给你一个隐式`Messages` 值，只要有一个`Lang` 或`RequestHeader` 在隐式作用域。

> **注意**: 如果你在隐式作用域有一个`RequestHeader` , 程序会使用`Accept-Language` 标头中提取的推荐语言并匹配`MessagesApi` 支持语言中的一个。你要添加一个`Messages` 隐式参数到你的模板，像这样: `@()(implicit messages: Messages)`。
>
> **注意**: 同样, Play开箱即用的 “懂得” 如何注入一个`MessagesApi` 值 (使用`DefaultMessagesApi` 实现), 所以你可以只使用`@javax.inject.Inject` 注解添加到你的控制器，并让Play自动为你管理组件。


##Messages 格式
Messages使用`java.text.MessageFormat` 库来格式化。举例, 假定你有message定义如下:

```scala
files.summary=The disk {1} contains {0} file(s).
```

那么你可以这样指定参数:

```scala
Messages("files.summary", d.files.length, d.name)
```

##注意单引号
因为 Messages 使用`java.text.MessageFormat`, 请注意单引号被会用作元字符来转义参数代入。

举例, 如果你定义了以下messages:

```scala
info.error=You aren''t logged in!
example.formatting=When using MessageFormat, '''{0}''' is replaced with the first parameter.
```

你应该期望以下结果:

```scala
Messages("info.error") == "You aren't logged in!"
Messages("example.formatting") == "When using MessageFormat, '{0}' is replaced with the first parameter."
```

##从HTTP请求中取回支持的语言
你可以从给定的HTTP请求取回支持的语言:

```scala
def index = Action { request =>
  Ok("Languages: " + request.acceptLanguages.map(_.code).mkString(", "))
}
```