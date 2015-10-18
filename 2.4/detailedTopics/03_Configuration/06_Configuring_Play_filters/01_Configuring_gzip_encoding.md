#配置 gzip 编码

Play 提供一个gzip过滤器，可以用于gzip响应。


##启用 gzip 过滤器
要启用gzip过滤器, 添加Play 过滤器项目到你的`build.sbt` 中的`libraryDependencies` 项:

```scala
libraryDependencies += filters
```

现在添加gzip filter到你的过滤器, 通常是在你的项目根目录下创建一个`Filters` 类:

```scala
import javax.inject.Inject

import play.api.http.HttpFilters
import play.filters.gzip.GzipFilter

class Filters @Inject() (gzipFilter: GzipFilter) extends HttpFilters {
  def filters = Seq(gzipFilter)
}
```

`Filters` 类可以放在根包或在其它位置，如果它有另一个名字或在其它包, 需要在`application.conf` 使用`play.http.filters` 配置:

```scala
play.http.filters = "filters.MyFilters"
```


##配置 gzip 过滤器
gzip过滤器支持少量的调整配置选项, 可以在`application.conf`配置。要查看有效的配置选项, 参阅 Play 过滤器[`reference.conf`](https://playframework.com/documentation/2.4.x/resources/confs/filters-helpers/reference.conf)。


##控制哪个响应被 gzipped
要控制哪个响应是否执行, 使用`shouldGzip` 参数, 接受一个请求标头函数和一个响应标头到一个布尔值。

举例, 下面的代码是仅 gzips HTML 响应:

```scala
new GzipFilter(shouldGzip = (request, response) =>
  response.headers.get("Content-Type").exists(_.startsWith("text/html")))
```