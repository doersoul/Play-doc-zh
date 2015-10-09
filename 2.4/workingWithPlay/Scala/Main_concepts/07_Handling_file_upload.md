#处理文件上传


##在表单中使用multipart/form-data上传文件
在web应用程序中上传文件的标准方式是使用指定`multipart/form-data` 编码的表单, 这样让你能混合标准表单数据和附件数据。 

> **注意**: 表单提交必须用`POST` HTTP方法(不能用`GET`)。 

我们从写一个HTML表单开始:

```scala
@helper.form(action = routes.Application.upload, 'enctype -> "multipart/form-data") {
    <input type="file" name="picture">
    <p>
        <input type="submit">
    </p>
}
```

现在使用一个`multipartFormData` body解析器定义`upload` action:

```scala
def upload = Action(parse.multipartFormData) { request =>
  request.body.file("picture").map { picture =>
    import java.io.File
    val filename = picture.filename
    val contentType = picture.contentType
    picture.ref.moveTo(new File(s"/tmp/picture/$filename"))
    Ok("File uploaded")
  }.getOrElse {
    Redirect(routes.Application.index).flashing(
      "error" -> "Missing file")
  }
}
```

`ref` 属性给你一个到`TemporaryFile`的参考。这是`mutipartFormData` 解析器处理文件上传的默认方式。

> **注意**: 你也可以使用`anyContent` body解析器，作为`request.body.asMultipartFormData`来检索。

最后, 添加一个`POST` 路由：

```
POST  /          controllers.Application.upload()
```


##直接文件上传
另一个上传文件到服务器的方式是在表单中使用Ajax来异步上传文件。在这种情况下请求体不再是`multipart/form-data`, 但纯粹只包含文件内容。

在这种情况下我们可以只用一个body解析器来保存请求体内容到文件。例如, 使用`temporaryFile` body 解析器:

```scala
def upload = Action(parse.temporaryFile) { request =>
  request.body.moveTo(new File("/tmp/picture/uploaded"))
  Ok("File uploaded")
}
```


##编写你自己的body解析器
如果你不想经过临时文件缓存而是直接处理上传的文件, 你可以自己写一个`BodyParser`。在这种情况下，你可以接收数据块，自由决定如何处理。

如果你想使用`multipart/form-data` 编码, 你仍可以使用默认的`mutipartFormData` 解析器，通过提供你自己的`PartHandler[FilePart[A]]`，接收部分标头, 然后提供一个`Iteratee[Array[Byte]`, `FilePart[A]]` 来生成正确的FilePart。