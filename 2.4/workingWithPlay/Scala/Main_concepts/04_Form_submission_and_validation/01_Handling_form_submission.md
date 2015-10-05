#处理表单提交


##概述
表单处理和提交是每个Web应用程序的重要部分。Play自带了一些功能，使简单表单很容易处理，并使处理复杂表单成为可能。

Play的表单处理方法基于数据绑定的概念。当数据来自一个POST请求, Play会查找格式化的值和绑定他们到一个[`Form`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Form.html) 对象上。然后, Play 可以使用绑定表单数据到一个样例类的值, 调用自定义验证, 如此等等。

通常表单会在`Controller` 实例中直接使用。然而, [`Form`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Form.html) 的定义不需要和样例类或模型一致匹配: 他们纯粹是为了处理输入，不同的POST使用不同的`Form` 是非常合理的。


##导入
要使用表单, 要导入以下包到你的类:

```scala
import play.api.data._
import play.api.data.Forms._
```


##表单基础
让我们看看表单处理的基础:

* 定义一个表单,
* 定义表单中的约束,
* 在一个action中验证表单,
* 在视图模板中显示表单,
* 最后, 在视图模板中处理表单的结果(或错误)。

最后的结果看起来像这样:

![""](lifecycle.png)


###定义一个表单
首先, 定义一个包含了你的表单中的元素的样例类。这里我们想要捕获一个用户的姓名和年龄, 因此我们创建`UserData` 对象:

```scala
case class UserData(name: String, age: Int)
```

现在我们有了一个样例类，下一步是定义一个 [`Form`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Form.html) 结构。`Form` 函数是将表单数据转换到一个绑定的样例类实例，我们像如下定义:

```scala
val userForm = Form(
  mapping(
    "name" -> text,
    "age" -> number
  )(UserData.apply)(UserData.unapply)
)
```

[Forms](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html) 对象定义了 [`mapping`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#mapping%5BR%2CA1%5D\(\(String%2CMapping%5BA1%5D\)\)\(\(A1\)%E2%87%92R\)\(\(R\)%E2%87%92Option%5BA1%5D\)%3AMapping%5BR%5D) 方法。这个方法带有表单的名字和约束做为参数, 也带有二个函数为参数: 一个`apply` 函数和一个`unapply` 函数。因为 UserData 是一个样例类, 我们可以将它的 `apply` 和`unapply` 方法直接插入到 mapping 方法中。


> **注意**: 由于表单处理的实现问题，单个元组或mapping最多只能有22个字段元素。如果你的表单有超过22个字段， 你应该分开你的表单，使用列表或嵌套值。

当你用了Map，一个表单会创建带有绑定值的`UserData` 实例:

```scala
val anyData = Map("name" -> "bob", "age" -> "21")
val userData = userForm.bind(anyData).get
```

但多数时候你会在Action内使用表单, 其中数据由请求提供。[`Form`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Form.html) 包含了 [`bindFromRequest`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Form.html#bindFromRequest\(\)\(Request%5B_%5D\)%3AForm%5BT%5D), 带有一个请求作为隐式参数。如果你定义一个隐式请求, 那么`bindFromRequest` 会找到它。

```scala
val userData = userForm.bindFromRequest.get
```

> **注意**: 这里使用`get` 是有问题的。如果表单没能绑定数据，`get` 会抛出异常。在下面的几章我们会介绍一种更安全的方法来处理输入。

在你的表单映射中，并没有限制只能使用样例类。只要正确映射了 apply 和 unapply 方法，就能传递你想要的任何东西, 如元组可以用 [`Forms.tuple`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html) 来映射，或是模型样例类。不管怎样, 为表单指定定义一个样例类有几个好处:

* **表单指定样例类简单易用**。样例类本来就设计为数据的简单容器, 并提供开箱即用的功能，和`Form` 的功能天然匹配。
* **表单指定样例类功能强大**。元组易于使用，但不允许自定义apply 或 unapply 方法, 且只能通过元数引用包含的数据 (`_1`, `_2`, 等)
* **表单指定样例类专门为表单设计**。重用模型样例类很方便, 但模型通常都含有一些额外的领域逻辑，基至一些持久化的细节，这会导致紧密的耦合。另外, 如果表单和模型不是严格 1:1 映射的话, 那么敏感的字段必须显式忽略，以防御[**参数篡改**](https://www.owasp.org/index.php/Web_Parameter_Tampering)攻击。

###Defining constraints on the form
The `text` constraint considers empty strings to be valid. This means that `name` could be empty here without an error, which is not what we want. A way to ensure that `name` has the appropriate value is to use the `nonEmptyText` constraint.

```scala
val userFormConstraints2 = Form(
  mapping(
    "name" -> nonEmptyText,
    "age" -> number(min = 0, max = 100)
  )(UserData.apply)(UserData.unapply)
)
```

Using this form will result in a form with errors if the input to the form does not match the constraints:

```scala
val boundForm = userFormConstraints2.bind(Map("bob" -> "", "age" -> "25"))
boundForm.hasErrors must beTrue
```

The out of the box constraints are defined on the [**Forms object**](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html):

* [`text`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#text%3AMapping%5BString%5D): maps to `scala.String`, optionally takes `minLength` and `maxLength`.
* [`nonEmptyText`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#nonEmptyText%3AMapping%5BString%5D): maps to `scala.String`, optionally takes `minLength` and `maxLength`.
* [`number`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#number%3AMapping%5BInt%5D): maps to `scala.Int`, optionally takes `min`, `max`, and `strict`.
* [`longNumber`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#longNumber%3AMapping%5BLong%5D): maps to `scala.Long`, optionally takes `min`, `max`, and `strict`.
* [`bigDecimal`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#bigDecimal%3AMapping%5BBigDecimal%5D): takes `precision` and `scale`.
* [`date`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#date%3AMapping%5BDate%5D), [`sqlDate`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#sqlDate%3AMapping%5BDate%5D), [`jodaDate`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#jodaDate%3AMapping%5BDateTime%5D): maps to `java.util.Date`, `java.sql.Date` and `org.joda.time.DateTime`, optionally takes `pattern` and `timeZone`.
* [`jodaLocalDate`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#jodaLocalDate%3AMapping%5BLocalDate%5D): maps to `org.joda.time.LocalDate`, optionally takes `pattern`.
* [`email`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#email%3AMapping%5BString%5D): maps to `scala.String`, using an email regular expression.
* [`boolean`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#boolean%3AMapping%5BBoolean%5D): maps to `scala.Boolean`.
* [`checked`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html#checked%3AMapping%5BBoolean%5D): maps to `scala.Boolean`.
* [`optional`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html): maps to `scala.Option`.

###Defining ad-hoc constraints
You can define your own ad-hoc constraints on the case classes using the [validation package](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/validation/package.html).

```scala
val userFormConstraints = Form(
  mapping(
    "name" -> text.verifying(nonEmpty),
    "age" -> number.verifying(min(0), max(100))
  )(UserData.apply)(UserData.unapply)
)
```

You can also define ad-hoc constraints on the case classes themselves:

```scala
def validate(name: String, age: Int) = {
  name match {
    case "bob" if age >= 18 =>
      Some(UserData(name, age))
    case "admin" =>
      Some(UserData(name, age))
    case _ =>
      None
  }
}

val userFormConstraintsAdHoc = Form(
  mapping(
    "name" -> text,
    "age" -> number
  )(UserData.apply)(UserData.unapply) verifying("Failed form constraints!", fields => fields match {
    case userData => validate(userData.name, userData.age).isDefined
  })
)
```

You also have the option of constructing your own custom validations. Please see the [custom validations](https://www.playframework.com/documentation/2.4.x/ScalaCustomValidations) section for more details.

###Validating a form in an Action
Now that we have constraints, we can validate the form inside an action, and process the form with errors.

We do this using the `fold` method, which takes two functions: the first is called if the binding fails, and the second is called if the binding succeeds.

```scala
userForm.bindFromRequest.fold(
  formWithErrors => {
    // binding failure, you retrieve the form containing errors:
    BadRequest(views.html.user(formWithErrors))
  },
  userData => {
    /* binding success, you get the actual value. */
    val newUser = models.User(userData.name, userData.age)
    val id = models.User.create(newUser)
    Redirect(routes.Application.home(id))
  }
)
```

In the failure case, we render the page with BadRequest, and pass in the form with errors as a parameter to the page. If we use the view helpers (discussed below), then any errors that are bound to a field will be rendered in the page next to the field.

In the success case, we’re sending a `Redirect` with a route to `routes.Application.home` here instead of rendering a view template. This pattern is called [Redirect after POST](https://en.wikipedia.org/wiki/Post/Redirect/Get) , and is an excellent way to prevent duplicate form submissions.

> **Note**: “Redirect after POST” is required when using `flashing` or other methods with [flash scope](https://www.playframework.com/documentation/2.4.x/ScalaSessionFlash), as new cookies will only be available after the redirected HTTP request.

Alternatively, you can use the `parse.form` [body parser](https://www.playframework.com/documentation/2.4.x/ScalaBodyParsers) that binds the content of the request to your form.

```scala
val userPost = Action(parse.form(userForm)) { implicit request =>
  val userData = request.body
  val newUser = models.User(userData.name, userData.age)
  val id = models.User.create(newUser)
  Redirect(routes.Application.home(id))
}
```

In the failure case, the default behaviour is to return an empty BadRequest response. You can override this behaviour with your own logic. For instance, the following code is completely equivalent to the preceding one using `bindFromRequest` and `fold`.

```scala
val userPostWithErrors = Action(parse.form(userForm, onErrors = (formWithErrors: Form[UserData]) => BadRequest(views.html.user(formWithErrors)))) { implicit request =>
  val userData = request.body
  val newUser = models.User(userData.name, userData.age)
  val id = models.User.create(newUser)
  Redirect(routes.Application.home(id))
}
```

###Showing forms in a view template
Once you have a form, then you need to make it available to the template engine. You do this by including the form as a parameter to the view template. For `user.scala.html`, the header at the top of the page will look like this:

```scala
@(userForm: Form[UserData])(implicit messages: Messages)
```

Because `user.scala.html` needs a form passed in, you should pass the empty `userForm` initially when rendering `user.scala.html`:

```scala
def index = Action {
  Ok(views.html.user(userForm))
}
```

The first thing is to be able to create the [form tag](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/form$.html). It is a simple view helper that creates a [form tag](http://www.w3.org/TR/html5/forms.html#the-form-element) and sets the `action` and `method` tag parameters according to the reverse route you pass in:

```scala
@helper.form(action = routes.Application.userPost()) {
  @helper.inputText(userForm("name"))
  @helper.inputText(userForm("age"))
}
```

You can find several input helpers in the [`views.html.helper`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/package.html) package. You feed them with a form field, and they display the corresponding HTML input, setting the value, constraints and displaying errors when a form binding fails.

> **Note**: You can use `@import helper._` in the template to avoid prefixing helpers with `@helper`.

There are several input helpers, but the most helpful are:

* [`form`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/form$.html): renders a [form](http://www.w3.org/TR/html-markup/form.html#form) element.
* [`inputText`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/inputText$.html): renders a [text input](http://www.w3.org/TR/html-markup/input.text.html) element.
* [`inputPassword`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/inputPassword$.html): renders a [password input](http://www.w3.org/TR/html-markup/input.password.html#input.password) element.
* [`inputDate`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/inputDate$.html): renders a [date input](http://www.w3.org/TR/html-markup/input.date.html) element.
* [`inputFile`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/inputFile$.html): renders a [file input](http://www.w3.org/TR/html-markup/input.file.html)  element.
* [`inputRadioGroup`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/inputRadioGroup$.html): renders a [radio input](http://www.w3.org/TR/html-markup/input.radio.html#input.radio) element.
* [`select`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/select$.html): renders a [select](http://www.w3.org/TR/html-markup/select.html#select) element.
* [`textarea`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/textarea$.html): renders a [textarea](http://www.w3.org/TR/html-markup/textarea.html#textarea)  element.
* [`checkbox`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/checkbox$.html): renders a [checkbox](http://www.w3.org/TR/html-markup/input.checkbox.html#input.checkbox) element.
* [`input`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/input$.html): renders a generic input element (which requires explicit arguments).

As with the `form` helper, you can specify an extra set of parameters that will be added to the generated Html:

```scala
@helper.inputText(userForm("name"), 'id -> "name", 'size -> 30)
```

The generic `input` helper mentioned above will let you code the desired HTML result:

```scala
@helper.input(userForm("name")) { (id, name, value, args) =>
    <input type="text" name="@name" id="@id" @toHtmlArgs(args)>
}
```

> **Note**: All extra parameters will be added to the generated Html, unless they start with the _ character. Arguments starting with _ are reserved for [field constructor arguments](https://www.playframework.com/documentation/2.4.x/ScalaCustomFieldConstructors).

For complex form elements, you can also create your own custom view helpers (using scala classes in the `views` package) and [custom field constructors](https://www.playframework.com/documentation/2.4.x/ScalaCustomFieldConstructors).

###Displaying errors in a view template
The errors in a form take the form of `Map[String,FormError]` where [`FormError`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/FormError.html) has:

* `key`: should be the same as the field.
* `message`: a message or a message key.
* `args`: a list of arguments to the message.

The form errors are accessed on the bound form instance as follows:

* `errors`: returns all errors as `Seq[FormError]`.
* `globalErrors`: returns errors without a key as `Seq[FormError]`.
* `error("name")`: returns the first error bound to key as `Option[FormError]`.
* `errors("name")`: returns all errors bound to key as `Seq[FormError]`.

Errors attached to a field will render automatically using the form helpers, so `@helper.inputText` with errors can display as follows:

```html
<dl class="error" id="age_field">
    <dt><label for="age">Age:</label></dt>
    <dd><input type="text" name="age" id="age" value=""></dd>
    <dd class="error">This field is required!</dd>
    <dd class="error">Another error</dd>
    <dd class="info">Required</dd>
    <dd class="info">Another constraint</dd>
</dl>
```

Global errors that are not bound to a key do not have a helper and must be defined explicitly in the page:

```scala
@if(userForm.hasGlobalErrors) {
  <ul>
  @for(error <- userForm.globalErrors) {
    <li>@error.message</li>
  }
  </ul>
}
```

###Mapping with tuples
You can use tuples instead of case classes in your fields:

```scala
val userFormTuple = Form(
  tuple(
    "name" -> text,
    "age" -> number
  ) // tuples come with built-in apply/unapply
)
```

Using a tuple can be more convenient than defining a case class, especially for low arity tuples:

```scala
val anyData = Map("name" -> "bob", "age" -> "25")
val (name, age) = userFormTuple.bind(anyData).get
```

###Mapping with single
Tuples are only possible when there are multiple values. If there is only one field in the form, use `Forms.single` to map to a single value without the overhead of a case class or tuple:

```scala
val singleForm = Form(
  single(
    "email" -> email
  )
)

val emailValue = singleForm.bind(Map("email" -> "bob@example.com")).get
```

###Fill values
Sometimes you’ll want to populate a form with existing values, typically for editing data:

```scala
val filledForm = userForm.fill(UserData("Bob", 18))
```

When you use this with a view helper, the value of the element will be filled with the value:

```scala
@helper.inputText(filledForm("name")) @* will render value="Bob" *@
```

Fill is especially helpful for helpers that need lists or maps of values, such as the [`select`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/select$.html) and [`inputRadioGroup`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/inputRadioGroup$.html) helpers. Use [`options`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/options$.html) to value these helpers with lists, maps and pairs.

###Nested values
A form mapping can define nested values by using [`Forms.mapping`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html) inside an existing mapping:

```scala
case class AddressData(street: String, city: String)

case class UserAddressData(name: String, address: AddressData)
val userFormNested: Form[UserAddressData] = Form(
  mapping(
    "name" -> text,
    "address" -> mapping(
      "street" -> text,
      "city" -> text
    )(AddressData.apply)(AddressData.unapply)
  )(UserAddressData.apply)(UserAddressData.unapply)
)
```

> **Note**: When you are using nested data this way, the form values sent by the browser must be named like `address.street`, `address.city`, etc.

```scala
@helper.inputText(userFormNested("name"))
@helper.inputText(userFormNested("address.street"))
@helper.inputText(userFormNested("address.city"))
```

###Repeated values
A form mapping can define repeated values using [`Forms.list`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html) or [`Forms.seq`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html):

```scala
case class UserListData(name: String, emails: List[String])
val userFormRepeated = Form(
  mapping(
    "name" -> text,
    "emails" -> list(email)
  )(UserListData.apply)(UserListData.unapply)
)
```

When you are using repeated data like this, there are two alternatives for sending the form values in the HTTP request. First, you can suffix the parameter with an empty bracket pair, as in “emails[]”. This parameter can then be repeated in the standard way, as in `http://foo.com/request?emails[]=a@b.com&emails[]=c@d.com`. Alternatively, the client can explicitly name the parameters uniquely with array subscripts, as in `emails[0]`, `emails[1]`, `emails[2]`, and so on. This approach also allows you to maintain the order of a sequence of inputs. 

If you are using Play to generate your form HTML, you can generate as many inputs for the `emails` field as the form contains, using the [`repeat`](https://www.playframework.com/documentation/2.4.x/api/scala/views/html/helper/repeat$.html) helper:

```scala
@helper.inputText(myForm("name"))
@helper.repeat(myForm("emails"), min = 1) { emailField =>
    @helper.inputText(emailField)
}
```

The `min` parameter allows you to display a minimum number of fields even if the corresponding form data are empty.

###Optional values
A form mapping can also define optional values using [`Forms.optional`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html):

```scala
case class UserOptionalData(name: String, email: Option[String])
val userFormOptional = Form(
  mapping(
    "name" -> text,
    "email" -> optional(email)
  )(UserOptionalData.apply)(UserOptionalData.unapply)
)
```

This maps to an `Option[A]` in output, which is `None` if no form value is found.

###Default values
You can populate a form with initial values using [`Form#fill`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Form.html):

```scala
val filledForm = userForm.fill(UserData("Bob", 18))
```

Or you can define a default mapping on the number using [`Forms.default`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html):

```scala
Form(
  mapping(
    "name" -> default(text, "Bob")
    "age" -> default(number, 18)
  )(User.apply)(User.unapply)
)
```

###Ignored values
If you want a form to have a static value for a field, use [`Forms.ignored`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/data/Forms$.html):

```scala
val userFormStatic = Form(
  mapping(
    "id" -> ignored(23L),
    "name" -> text,
    "email" -> optional(email)
  )(UserStaticData.apply)(UserStaticData.unapply)
)
```


##Putting it all together
Here’s an example of what a model and controller would look like for managing an entity.

Given the case class `Contact`:

```scala
case class Contact(firstname: String,
                   lastname: String,
                   company: Option[String],
                   informations: Seq[ContactInformation])
object Contact {
  def save(contact: Contact): Int = 99
}
case class ContactInformation(label: String,
                              email: Option[String],
                              phones: List[String])
```

Note that `Contact` contains a `Seq` with `ContactInformation` elements and a `List` of `String`. In this case, we can combine the nested mapping with repeated mappings (defined with `Forms.seq` and `Forms.list`, respectively).

```scala
val contactForm: Form[Contact] = Form(

  // Defines a mapping that will handle Contact values
  mapping(
    "firstname" -> nonEmptyText,
    "lastname" -> nonEmptyText,
    "company" -> optional(text),

    // Defines a repeated mapping
    "informations" -> seq(
      mapping(
        "label" -> nonEmptyText,
        "email" -> optional(email),
        "phones" -> list(
          text verifying pattern("""[0-9.+]+""".r, error="A valid phone number is required")
        )
      )(ContactInformation.apply)(ContactInformation.unapply)
    )
  )(Contact.apply)(Contact.unapply)
)
```

And this code shows how an existing contact is displayed in the form using filled data:

```scala
def editContact = Action {
  val existingContact = Contact(
    "Fake", "Contact", Some("Fake company"), informations = List(
      ContactInformation(
        "Personal", Some("fakecontact@gmail.com"), List("01.23.45.67.89", "98.76.54.32.10")
      ),
      ContactInformation(
        "Professional", Some("fakecontact@company.com"), List("01.23.45.67.89")
      ),
      ContactInformation(
        "Previous", Some("fakecontact@oldcompany.com"), List()
      )
    )
  )
  Ok(views.html.contact.form(contactForm.fill(existingContact)))
}
```

Finally, this is what a form submission handler would look like:

```scala
def saveContact = Action { implicit request =>
  contactForm.bindFromRequest.fold(
    formWithErrors => {
      BadRequest(views.html.contact.form(formWithErrors))
    },
    contact => {
      val contactId = Contact.save(contact)
      Redirect(routes.Application.showContact(contactId)).flashing("success" -> "Contact saved!")
    }
  )
}
```