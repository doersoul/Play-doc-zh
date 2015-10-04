#为模板引擎添加自定义格式支持

内建的模板引擎支持常见的模板格式(HTML, XML, 等)，但如果需要，你可以很容易地添加你自己的格式支持。本页总结了添加自定义格式支持的步骤。


##模板化进程概述
模板引擎通过组合静态和动态部分的模板内容来构建最终结果。考虑如下模板:

```scala
foo @bar baz
```

它包含着二个静态部分(`foo` 和 `baz`) 和一个被静态部分包围的动态部分 (`bar`)。模板引擎将这些部分拼接在一起并生成结果。事实上, 为了避免跨站脚本攻击, `bar` 的值在拼接到其它结果之前可以先进行转义。转义过程与特定的格式相关: 例如，对于HTML， “<” 转义为 “&lt;”。

模板引擎是如何得知模板文件的格式的呢? 它会检查扩展名: 例如文件扩展名是 `.scala.html` ，它会关联HTML格式到这个文件。

最后, 通常你需要将模板文件用作HTTP响应的响应体, 所以你还要定义如何用模板渲染的结果来构建Play result。

综上所述, 要支持你自己的模板格式，你需要执行以下步骤:

* 为该格式实现文本整合进程;
* 关联文件扩展名到该格式;
* 最终告诉Play如何发送渲染为HTTP响应体的模板结果


##实现一个格式
实现 `play.twirl.api.Format[A]` 特质，它包含`raw(text: String): A` 和 `escape(text: String): A` 方法，分别用于整合模板中的静态和动态部分。

格式的类型参数`A` 定义模板渲染后的结果类型, 如相对于HTML模板的`Html`。这个类型必须是`play.twirl.api.Appendable[A]` 特质的子类，这个特质定义了如何把各部分接接到一起。

为方便起见, Play 提供一个`play.twirl.api.BufferedContent[A]` 抽象类，它实现了`play.twirl.api.Appendable[A]` 特质，使用一个`StringBuilder` 来构建它的结果。还实现了`play.twirl.api.Content` 特质，因此 Play知道如何将它作为一个HTTP 响应体来序列化(详情参见本页最后一节)。

简而言之, 你需要写两个类: 一个定义结果(实现`play.twirl.api.Appendable[A]`) 和一个定义文本整合过程(实现 `play.twirl.api.Format[A]`)。例如, 这里是 HTML格式的定义:

```scala
// `Html` 结果类型。我们扩展`BufferedContent[Html]` 而不是`Appendable[Html]` ，
// 因此 Play 知道如何从一个`Html`值构建 HTTP 结果
class Html(buffer: StringBuilder) extends BufferedContent[Html](buffer) {
  val contentType = MimeTypes.HTML
}

object HtmlFormat extends Format[Html] {
  def raw(text: String): Html = …
  def escape(text: String): Html = …
}
```


##关联文件扩展名到这个格式
在编译整个项目源码之前，模板文件会在构建过程中编译到 `.scala` 文件。`TwirlKeys.templateFormats` 键是一个`Map[String, String]`类型的 sbt 配置项，定义文件扩展名和模板格式之间的映射。例如, 假设 HTML不是 Play开箱即用支持的, 你就需要在构建文件中写入下面的配置来关联`.scala.html` 文件到`play.twirl.api.HtmlFormat` 格式:

```scala
TwirlKeys.templateFormats += ("html" -> "my.HtmlFormat.instance")
```

注意，箭头右边包含了一个类型为`play.twirl.api.Format[_]`的值的全限定名称。


##告诉 Play如何从模板渲染类型构建 HTTP result
只要类型 `A`的值可以隐式转换为`play.api.http.Writeable[A]` 值，则Play可以将`A` 的任何值写入HTTP响应体中。因此所有你要做的就是定义如一个模板结果类型的值。例如, 这是如何为HTTP定义值:

```scala
implicit def writableHttp(implicit codec: Codec): Writeable[Http] =
  Writeable[Http](result => codec.encode(result.body), Some(ContentTypes.HTTP))
```

> **注意**: 如果你的模板结果类型扩展了`play.twirl.api.BufferedContent` ，你只需要定义一个隐式`play.api.http.ContentTypeOf` 值:
`scala implicit def contentTypeHttp(implicit codec: Codec): ContentTypeOf[Http] = ContentTypeOf[Http](Some(ContentTypes.HTTP))`