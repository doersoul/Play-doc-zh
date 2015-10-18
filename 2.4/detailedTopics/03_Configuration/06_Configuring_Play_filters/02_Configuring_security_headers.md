#配置安全标头

Play 提供一个安全标头过滤器，可以用于配置一些在HTTP响应中的默认标头，以减轻安全问题和为新应用程序提高防御级别。


##启用安全标头过滤器
要启用安全标头过滤器, 添加Play 过滤器项目到你的`build.sbt` 中的 `libraryDependencies` 项:

```scala
libraryDependencies += filters
```

现在添加安全标头过滤器到你的过滤器类, 通常是在项目根目录创建一个`Filters` 类:

```scala
import javax.inject.Inject

import play.api.http.HttpFilters
import play.filters.headers.SecurityHeadersFilter

class Filters @Inject() (securityHeadersFilter: SecurityHeadersFilter) extends HttpFilters {
  def filters = Seq(securityHeadersFilter)
}
```

`Filters` 可以在根包或其它地方, 如果它用另一个名字或在其它包, 需要在`application.conf` 中使用`play.http.filters` 配置:

```scala
play.http.filters = "filters.MyFilters"
```


##配置安全标头
Scaladoc 在 [play.filters.headers](https://playframework.com/documentation/2.4.x/api/scala/play/filters/headers/package.html) 包。

过滤器会在HTTP响应自动设置标头。设置可以在`application.conf` 中通过以下设置来配置。 

* `play.filters.headers.frameOptions` - 设置 [X-Frame-Options](https://developer.mozilla.org/en-US/docs/HTTP/X-Frame-Options), 默认是“DENY”。
* `play.filters.headers.xssProtection` - 设置 [X-XSS-Protection](http://blogs.msdn.com/b/ie/archive/2008/07/02/ie8-security-part-iv-the-xss-filter.aspx),默认是 “1; mode=block” 。
* `play.filters.headers.contentTypeOptions` - 设置 [X-Content-Type-Options](http://blogs.msdn.com/b/ie/archive/2008/09/02/ie8-security-part-vi-beta-2-update.aspx), 默认是“nosniff”。
* `play.filters.headers.permittedCrossDomainPolicies` - 设置 [X-Permitted-Cross-Domain-Policies](https://www.adobe.com/devnet/articles/crossdomain_policy_file_spec.html), 默认是 “master-only”。
* `play.filters.headers.contentSecurityPolicy` - 设置 [Content-Security-Policy](http://www.html5rocks.com/en/tutorials/security/content-security-policy/), 默认是“default-src ‘self’”。

任何标头都可以通过设置一个`null`配置值来禁用, 例如:

```scala
play.filters.headers.frameOptions = null
```

有关配置选项的完整列表, 查阅Play过滤器 [`reference.conf`](https://playframework.com/documentation/2.4.x/resources/confs/filters-helpers/reference.conf)。