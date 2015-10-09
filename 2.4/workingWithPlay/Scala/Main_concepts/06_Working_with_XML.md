#处理和提供XML


##处理XML请求
一个XML请求是指使用一个有效的XML载体作为请求体的HTTP请求。必须在它的`Content-Type` 标头中指定`application/xml` 或`text/xml` MIME 类型。

`Action` 默认使用 **any content** body解析器, 这让你可以接受XML格式的请求体 (实际是作为`NodeSeq`):

```scala
def sayHello = Action { request =>
  request.body.asXml.map { xml =>
    (xml \\ "name" headOption).map(_.text).map { name =>
      Ok("Hello " + name)
    }.getOrElse {
      BadRequest("Missing parameter [name]")
    }
  }.getOrElse {
    BadRequest("Expecting Xml data")
  }
}
```

更好(也更简单)的方法是指定我们自己的`BodyParser` 来告诉Play直接解析请求体内容为XML:

```scala
def sayHello = Action(parse.xml) { request =>
  (request.body \\ "name" headOption).map(_.text).map { name =>
    Ok("Hello " + name)
  }.getOrElse {
    BadRequest("Missing parameter [name]")
  }
}
```

> **注意**: 当使用一个XML body解析器, `request.body`值直接就是一个有效的 `NodeSeq`。 

你可以从命令行用[cURL](http://curl.haxx.se/) 测试它:

```shell
curl 
  --header "Content-type: application/xml" 
  --request POST 
  --data '<name>Guillaume</name>' 
  http://localhost:9000/sayHello
```

它返回:

```
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 15

Hello Guillaume
```


##提供XML响应
在之前的例子中，我们处理XML请求, 但回复的是`text/plain` 响应。让我们改变它，发送一个有效的XML HTTP响应:

```scala
def sayHello = Action(parse.xml) { request =>
  (request.body \\ "name" headOption).map(_.text).map { name =>
    Ok(<message status="OK">Hello {name}</message>)
  }.getOrElse {
    BadRequest(<message status="KO">Missing parameter [name]</message>)
  }
}
```

现在它返回:

```
HTTP/1.1 200 OK
Content-Type: application/xml; charset=utf-8
Content-Length: 46

<message status="OK">Hello Guillaume</message>
```