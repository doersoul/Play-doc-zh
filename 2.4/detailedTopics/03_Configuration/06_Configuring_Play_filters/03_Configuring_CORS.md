#配置CORS(Cross-Origin Resource Sharing）

Play 提供一个过滤器，它实现跨域资源共享 (CORS)。

CORS 是一个允许web应用程序从浏览器发出一个跨域请求的协议。可以在[这里](http://www.w3.org/TR/cors/)找到完整的规范说明。


##启用 CORS过滤器
要启用CORS过滤器, 添加Play过滤器项目到你的`build.sbt`中的`libraryDependencies` 项:

```sbt
libraryDependencies += filters
```

现在添加CORS filter到你的过滤器, 通常是在项目根目录创建一个`Filters` 类:

```scala
import javax.inject.Inject

import play.api.http.HttpFilters
import play.filters.cors.CORSFilter

class Filters @Inject() (corsFilter: CORSFilter) extends HttpFilters {
  def filters = Seq(corsFilter)
}
```


##配置 CORS 过滤器
过滤器可以从`application.conf`配置。有关配置选项的完整列表, 查阅 Play 过滤器 [`reference.conf`](https://playframework.com/documentation/2.4.x/resources/confs/filters-helpers/reference.conf)。

可用选项包括:

* `play.filters.cors.pathPrefixes` - 过滤器路径，通过路径前缀白名单
* `play.filters.cors.allowedOrigins` - 只允许白名单中的请求来源(默认是允许所有来源)
* `play.filters.cors.allowedHttpMethods` - 仅允许白名单中的HTTP方法以预检请求(默认是允许所有方法)
* `play.filters.cors.allowedHttpHeaders` - 仅允许白名单中的HTTP标头以预检请求(默认是允许所有标头)
* `play.filters.cors.exposedHeaders` - 设置自定义HTTP标头以在响应中暴露(默认是没有标头暴露)
* `play.filters.cors.supportsCredentials` - 禁用/启用 证书支持(默认是启用证书支持)
* `play.filters.cors.preflightMaxAge` - 设置预检请求的结果可以缓存多长时间(默认是1小时)

举例:

```scala
play.filters.cors {
  pathPrefixes = ["/some/path", ...]
  allowedOrigins = ["http://www.example.com", ...]
  allowedHttpMethods = ["GET", "POST"]
  allowedHttpHeaders = ["Accept"]
  preflightMaxAge = 3 days
}
```