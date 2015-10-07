#自定义表单域构造器
一个表单域的渲染不仅仅由`<input>` 标签组成, 它也需要`<label>` 和其他一些可能被你的CSS框架用来装饰表单域的标签。

所有input助手带有一个隐式的[`FieldConstructor`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/FieldConstructor.html) 来处理这一部分。[默认构造器](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/defaultFieldConstructor$.html) (在作用域内没有指定其它表单域构造器时会被使用), 生成的HTML像这样:

```html
<dl class="error" id="username_field">
    <dt><label for="username">Username:</label></dt>
    <dd><input type="text" name="username" id="username" value=""></dd>
    <dd class="error">This field is required!</dd>
    <dd class="error">Another error</dd>
    <dd class="info">Required</dd>
    <dd class="info">Another constraint</dd>
</dl>
```

这个默认表单域构造器支持传入额外的选项到input助手参数:

```scala
'_label -> "Custom label"
'_id -> "idForTheTopDlElement"
'_help -> "Custom help"
'_showConstraints -> false
'_error -> "Force an error"
'_showErrors -> false
```


##编写你的自定义表单域构造器
你常常需要编写自定义的表单域构造器。首先要写一个模板，像这样:

```html
@(elements: helper.FieldElements)

<div class="@if(elements.hasErrors) {error}">
    <label for="@elements.id">@elements.label</label>
    <div class="input">
        @elements.input
        <span class="errors">@elements.errors.mkString(", ")</span>
        <span class="help">@elements.infos.mkString(", ")</span>
    </div>
</div>
```

> **注意**: 这只是一个例子。你想要实现复杂的功能都可以。你还可以使用`@elements.field`来获取原始的表单域。

现在使用这个模板函数来创建一个[`FieldConstructor`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/FieldConstructor.html):

```scala
object MyHelpers {
  import views.html.helper.FieldConstructor
  implicit val myFields = FieldConstructor(html.myFieldConstructorTemplate.f)
}
```

然后让表单助手来使用它, 只需导入它到你的模板:

```scala
@import MyHelpers._
@helper.inputText(myForm("username"))
```

它会使用你的表单域构造器来渲染input文本。

你也可以为你的[`FieldConstructor`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/FieldConstructor.html) 内联设置一个隐式值:

```scala
@implicitField = @{ helper.FieldConstructor(myFieldConstructorTemplate.f) }
@helper.inputText(myForm("username"))
```
