#配置CORS(Cross-Origin Resource Sharing）

Play provides a filter that implements Cross-Origin Resource Sharing (CORS).

CORS is a protocol that allows web applications to make requests from the browser across different domains. A full specification can be found [here](http://www.w3.org/TR/cors/).


##Enabling the CORS filter
To enable the CORS filter, add the Play filters project to your `libraryDependencies` in `build.sbt`:

```sbt
libraryDependencies += filters
```

Now add the CORS filter to your filters, which is typically done by creating a `Filters` class in the root of your project:

```scala
import javax.inject.Inject

import play.api.http.HttpFilters
import play.filters.cors.CORSFilter

class Filters @Inject() (corsFilter: CORSFilter) extends HttpFilters {
  def filters = Seq(corsFilter)
}
```


##Configuring the CORS filter
The filter can be configured from `application.conf`. For a full listing of configuration options, see the Play filters [`reference.conf`](https://playframework.com/documentation/2.4.x/resources/confs/filters-helpers/reference.conf).

The available options include:

* `play.filters.cors.pathPrefixes` - filter paths by a whitelist of path prefixes
* `play.filters.cors.allowedOrigins` - allow only requests with origins from a whitelist (by default all origins are allowed)
* `play.filters.cors.allowedHttpMethods` - allow only HTTP methods from a whitelist for preflight requests (by default all methods are allowed)
* `play.filters.cors.allowedHttpHeaders` - allow only HTTP headers from a whitelist for preflight requests (by default all headers are allowed)
* `play.filters.cors.exposedHeaders` - set custom HTTP headers to be exposed in the response (by default no headers are exposed)
* `play.filters.cors.supportsCredentials` - disable/enable support for credentials (by default credentials support is enabled)
* `play.filters.cors.preflightMaxAge` - set how long the results of a preflight request can be cached in a preflight result cache (by default 1 hour)

For example:

```scala
play.filters.cors {
  pathPrefixes = ["/some/path", ...]
  allowedOrigins = ["http://www.example.com", ...]
  allowedHttpMethods = ["GET", "POST"]
  allowedHttpHeaders = ["Accept"]
  preflightMaxAge = 3 days
}
```