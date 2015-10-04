#Scala 模板常见使用案例

模板作为简单的函数, 可以由任何你想要的方式构成。下面是一些常见场景的示例。


##布局
让我们声明一个`views/main.scala.html` 模板，它将作为一个主要的布局模板:

```scala
@(title: String)(content: Html)
<!DOCTYPE html>
<html>
  <head>
    <title>@title</title>
  </head>
  <body>
    <section class="content">@content</section>
  </body>
</html>
```

如你所见, 这个模板带二个参数: 一个标题（title）和一个HTML内容块（content）。现在我们可以在另一个`views/Application/index.scala.html` 模板中使用它:

```scala
@main(title = "Home") {
  <h1>Home page</h1>
}
```

> **注意**: 有时候我们使用命名参数(像`@main(title = "Home")`, 有时又不用，像`@main("Home")`。这个看你的需要, 选择一个在特定环境中表述清晰的即可。

有时候你需要第二个特定页面的内容作为侧边栏或面包屑。你可以添加一个额外参数来实现:

```html
@(title: String)(sidebar: Html)(content: Html)
<!DOCTYPE html>
<html>
  <head>
    <title>@title</title>
  </head>
  <body>
    <section class="sidebar">@sidebar</section>
    <section class="content">@content</section>
  </body>
</html>
```

在我们的 ‘index’ 模板中使用它:

```html
@main("Home") {
  <h1>Sidebar</h1>
} {
  <h1>Home page</h1>
}
```

或者, 我们可以单独声明侧边栏:

```html
@sidebar = {
  <h1>Sidebar</h1>
}

@main("Home")(sidebar) {
  <h1>Home page</h1>
}
```


##标签 (他们也是函数，对吗?)
让我们来写一个简单的`views/tags/notice.scala.html` 标签，他显示一个 HTML 通知:

```scala
@(level: String = "error")(body: (String) => Html)

@level match {

  case "success" => {
    <p class="success">
      @body("green")
    </p>
  }

  case "warning" => {
    <p class="warning">
      @body("orange")
    </p>
  }

  case "error" => {
    <p class="error">
      @body("red")
    </p>
  }

}
```

然后我们在另一个模板中使用它:

```
@import tags._

@notice("error") { color =>
  Oops, something is <span style="color:@color">wrong</span>
}
```


##Includes
再次, 同样没有什么特别之外。你可以调用任何其它模板(并且事实上是任何地方的所有任何函数):

```html
<h1>Home</h1>

<div id="side">
  @common.sideBar()
</div>
```


##moreScripts 与 moreStyles 等价物
要在Scala模板中定义旧版的 moreScripts 或 moreStyles 变量等价物(像在 Play! 1.x) , 你可以像下面一样在main模板中定义一个变量:

```html
@(title: String, scripts: Html = Html(""))(content: Html)

<!DOCTYPE html>

<html>
    <head>
        <title>@title</title>
        <link rel="stylesheet" media="screen" href="@routes.Assets.at("stylesheets/main.css")">
        <link rel="shortcut icon" type="image/png" href="@routes.Assets.at("images/favicon.png")">
        <script src="@routes.Assets.at("javascripts/jquery-1.7.1.min.js")" type="text/javascript"></script>
        @scripts
    </head>
    <body>
        <div class="navbar navbar-fixed-top">
            <div class="navbar-inner">
                <div class="container">
                    <a class="brand" href="#">Movies</a>
                </div>
            </div>
        </div>
        <div class="container">
            @content
        </div>
    </body>
</html>
```

对一个需要额外脚本的扩展模板:

```html
@scripts = {
    <script type="text/javascript">alert("hello !");</script>
}

@main("Title",scripts){

   Html content here ...

}
```

如扩展模板不需要额外脚本，则:

```scala
@main("Title"){

   Html content here ...

}
```