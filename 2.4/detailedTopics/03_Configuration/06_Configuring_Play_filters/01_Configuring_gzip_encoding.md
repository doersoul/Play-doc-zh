#配置 gzip 编码

Play provides a gzip filter that can be used to gzip responses.


##Enabling the gzip filter
To enable the gzip filter, add the Play filters project to your `libraryDependencies` in `build.sbt`:

```scala
libraryDependencies += filters
```

Now add the gzip filter to your filters, which is typically done by creating a `Filters` class in the root of your project:

```scala
import javax.inject.Inject

import play.api.http.HttpFilters
import play.filters.gzip.GzipFilter

class Filters @Inject() (gzipFilter: GzipFilter) extends HttpFilters {
  def filters = Seq(gzipFilter)
}
```

The `Filters` class can either be in the root package, or if it has another name or is in another package, needs to be configured using `play.http.filters` in `application.conf`:

```scala
play.http.filters = "filters.MyFilters"
```


##Configuring the gzip filter
The gzip filter supports a small number of tuning configuration options, which can be configured from `application.conf`. To see the available configuration options, see the Play filters [`reference.conf`](https://playframework.com/documentation/2.4.x/resources/confs/filters-helpers/reference.conf).


##Controlling which responses are gzipped
To control which responses are and aren’t implemented, use the `shouldGzip` parameter, which accepts a function of a request header and a response header to a boolean.

For example, the code below only gzips HTML responses:

```scala
new GzipFilter(shouldGzip = (request, response) =>
  response.headers.get("Content-Type").exists(_.startsWith("text/html")))
```