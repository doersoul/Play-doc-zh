#The Play cache API
Caching data is a typical optimization in modern applications, and so Play provides a global cache.

> An important point about the cache is that it behaves just like a cache should: the data you just stored may just go missing.

For any data stored in the cache, a regeneration strategy needs to be put in place in case the data goes missing. This philosophy is one of the fundamentals behind Play, and is different from Java EE, where the session is expected to retain values throughout its lifetime. 

The default implementation of the Cache API uses [EHCache](http://ehcache.org/) .


##Importing the Cache API
Add `cache` into your dependencies list. For example, in `build.sbt`:

```sbt
libraryDependencies ++= Seq(
  cache,
  ...
)
```


##Accessing the Cache API
The cache API is provided by the [CacheApi](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/cache/CacheApi.html) object, and can be injected into your component like any other dependency. For example:

```scala
import play.api.cache._
import play.api.mvc._
import javax.inject.Inject

class Application @Inject() (cache: CacheApi) extends Controller {

}
```

> **Note**: The API is intentionally minimal to allow several implementation to be plugged in. If you need a more specific API, use the one provided by your Cache plugin.

Using this simple API you can either store data in cache:

```scala
cache.set("item.key", connectedUser)
```

And then retrieve it later:

```scala
val maybeUser: Option[User] = cache.get[User]("item.key")
```

There is also a convenient helper to retrieve from cache or set the value in cache if it was missing:

```scala
val user: User = cache.getOrElse[User]("item.key") {
  User.findById(connectedUser)
}
```

You can specify an expiry duration by passing a duration, by default the duration is infinite:

```scala
import scala.concurrent.duration._

cache.set("item.key", connectedUser, 5.minutes)
```

To remove an item from the cache use the `remove` method:

```scala
cache.remove("item.key")
```


##Accessing different caches
It is possible to access different caches. The default cache is called `play`, and can be configured by creating a file called `ehcache.xml`. Additional caches may be configured with different configurations, or even implementations.

If you want to access multiple different ehcache caches, then you’ll need to tell Play to bind them in `application.conf`, like so:

```scala
play.cache.bindCaches = ["db-cache", "user-cache", "session-cache"]
```

Now to access these different caches, when you inject them, use the [NamedCache](https://www.playframework.com/documentation/2.4.x/api/java/play/cache/NamedCache.html) qualifier on your dependency, for example:

```scala
import play.api.cache._
import play.api.mvc._
import javax.inject.Inject

class Application @Inject()(
    @NamedCache("session-cache") sessionCache: CacheApi
) extends Controller {

}
```


##Caching HTTP responses
You can easily create smart cached actions using standard Action composition.

> **Note**: Play HTTP `Result` instances are safe to cache and reuse later.

The [Cached](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/cache/Cached.html) class helps you build cached actions.

```scala
import play.api.cache.Cached
import javax.inject.Inject

class Application @Inject() (cached: Cached) extends Controller {

}
```

You can cache the result of an action using a fixed key like `"homePage"`.

```scala
def index = cached("homePage") {
  Action {
    Ok("Hello world")
  }
}
```

If results vary, you can cache each result using a different key. In this example, each user has a different cached result.

```scala
def userProfile = Authenticated {
  user =>
    cached(req => "profile." + user) {
      Action {
        Ok(views.html.profile(User.find(user)))
      }
    }
}
```

###Control caching
You can easily control what you want to cache or what you want to exclude from the cache.

You may want to only cache 200 Ok results.

```scala
def get(index: Int) = cached.status(_ => "/resource/"+ index, 200) {
  Action {
    if (index > 0) {
      Ok(Json.obj("id" -> index))
    } else {
      NotFound
    }
  }
}
```

Or cache 404 Not Found only for a couple of minutes

```scala
def get(index: Int) = {
  val caching = cached
    .status(_ => "/resource/"+ index, 200)
    .includeStatus(404, 600)

  caching {
    Action {
      if (index % 2 == 1) {
        Ok(Json.obj("id" -> index))
      } else {
        NotFound
      }
    }
  }
}
```


##Custom implementations
It is possible to provide a custom implementation of the [CacheApi](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/cache/CacheApi.html) that either replaces, or sits along side the default implementation.

To replace the default implementation, you’ll need to disable the default implementation by setting the following in `application.conf`:

```scala
play.modules.disabled += "play.api.cache.EhCacheModule"
```

Then simply implement `CacheApi` and bind it in the [DI container](https://www.playframework.com/documentation/2.4.x/ScalaDependencyInjection).

To provide an implementation of the cache API in addition to the default implementation, you can either create a custom qualifier, or reuse the `NamedCache` qualifier to bind the implementation.