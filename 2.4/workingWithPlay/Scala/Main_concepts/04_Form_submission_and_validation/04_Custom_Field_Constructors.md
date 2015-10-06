#Custom Field Constructors
A field rendering is not only composed of the `<input>` tag, but it also needs a `<label>` and possibly other tags used by your CSS framework to decorate the field.

All input helpers take an implicit [`FieldConstructor`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/FieldConstructor.html) that handles this part. The [default one](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/defaultFieldConstructor$.html) (used if there are no other field constructors available in the scope), generates HTML like:

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

This default field constructor supports additional options you can pass in the input helper arguments:

```scala
'_label -> "Custom label"
'_id -> "idForTheTopDlElement"
'_help -> "Custom help"
'_showConstraints -> false
'_error -> "Force an error"
'_showErrors -> false
```


##Writing your own field constructor
Often you will need to write your own field constructor. Start by writing a template like:

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

> **Note**: This is just a sample. You can make it as complicated as you need. You also have access to the original field using `@elements.field`.

Now create a [`FieldConstructor`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/FieldConstructor.html) using this template function:

```scala
object MyHelpers {
  import views.html.helper.FieldConstructor
  implicit val myFields = FieldConstructor(html.myFieldConstructorTemplate.f)
}
```

And to make the form helpers use it, just import it in your templates:

```scala
@import MyHelpers._
@helper.inputText(myForm("username"))
```

It will then use your field constructor to render the input text.

You can also set an implicit value for your [`FieldConstructor`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/FieldConstructor.html) inline:

```scala
@implicitField = @{ helper.FieldConstructor(myFieldConstructorTemplate.f) }
@helper.inputText(myForm("username"))
```
