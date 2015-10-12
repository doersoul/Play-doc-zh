#Play 缓存 API
缓存数据在现代应用程序中是一个典型的优化, 因此Play提供一个全局缓存。

> 关于缓存，重要的一点是它的行为就像缓存应该做的: 你只保存可能会丢失的数据。

为在缓存中保存任何数据, 要有再生策略以防止数据丢失。这是Play背后的基本哲学之一, 不同于Java EE, 会话被期望贯穿它的整个生命周期保留值。 

Cache API默认实现使用[EHCache](http://ehcache.org/)。


##导入缓存 API
添加 `cache` 到你的依赖列表。例如在`build.sbt`:

```sbt
libraryDependencies ++= Seq(
  cache,
  ...
)
```


##访问缓存API
缓存 API 由 [CacheApi](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/cache/CacheApi.html) 对象提供, 并且可以像任何其它依赖一样注入到你的组件中。举例:

```scala
import play.api.cache._
import play.api.mvc._
import javax.inject.Inject

class Application @Inject() (cache: CacheApi) extends Controller {

}
```

> **注意**: 这个API特意最小化，来允许几个实现可被嵌入。如果你需要更多具体的API, 使用由你的缓存插件提供的那个。

使用这个简单API你既可以存储数据在缓存中:

```scala
cache.set("item.key", connectedUser)
```

然后在不久后还可以获取它:

```scala
val maybeUser: Option[User] = cache.get[User]("item.key")
```

当其不存在时，还有一个简便的助手方法来从缓存中获取值或在缓存中设置值:

```scala
val user: User = cache.getOrElse[User]("item.key") {
  User.findById(connectedUser)
}
```

你可以通过传递一个duration来指定一个持续期间, 默认下duration是内联的:

```scala
import scala.concurrent.duration._

cache.set("item.key", connectedUser, 5.minutes)
```

要从缓存移除物品，使用`remove` 方法:

```scala
cache.remove("item.key")
```


##访问不同的缓存
可以使用不同的缓存。默认缓存叫`play`, 并可以通过创建一个叫`ehcache.xml`的文件来配置。附加缓存可以用不同的配置文件来配置，甚至实现。

如果你想要访问多个不同的ehcache缓存, 那么你需要告诉Play在`application.conf`中绑定他们, 如:

```scala
play.cache.bindCaches = ["db-cache", "user-cache", "session-cache"]
```

现在访问这些不同的缓存, 当你注入他们后, 在你的依赖上使用 [NamedCache](https://www.playframework.com/documentation/2.4.x/api/java/play/cache/NamedCache.html) 限定符, 例如:

```scala
import play.api.cache._
import play.api.mvc._
import javax.inject.Inject

class Application @Inject()(
    @NamedCache("session-cache") sessionCache: CacheApi
) extends Controller {

}
```


##缓存 HTTP 响应
你可以使用标准的Action组合轻松创建智能缓存actions。

> **注意**: Play HTTP `Result` 实例对缓存是安全并可重用的。

[Cached](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/cache/Cached.html) 类可以帮助你创建缓存actions。

```scala
import play.api.cache.Cached
import javax.inject.Inject

class Application @Inject() (cached: Cached) extends Controller {

}
```

你可以使用一个fixed key如`"homePage"`来缓存action的result。

```scala
def index = cached("homePage") {
  Action {
    Ok("Hello world")
  }
}
```

如果results多变, 你可以使用不同的key来缓存每个result。在本例, 每个用户有不同的缓存result。

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

###缓存控制
你可以很容易的控制什么是你想缓存的或什么是你想从缓存中排除的。

你可能只想缓存返回 "200 Ok" 的 results。

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

或缓存 “404 Not Found”,仅存在几分钟

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


##自定义实现
可以提供一个[CacheApi](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/cache/CacheApi.html) 的自定义实现，要么替换它,或和默认实现一起使用。

要替换默认实现, 你需要通过以下在`application.conf` 中的设置禁用默认实现:

```scala
play.modules.disabled += "play.api.cache.EhCacheModule"
```

然后简单的实现`CacheApi` 并在[DI container](https://www.playframework.com/documentation/2.4.x/ScalaDependencyInjection)中绑定它。

要提供一个cache API实现，并附加到默认实现中, 你可以创建一个自定义限定符, 或重用`NamedCache` 限定符来绑定实现。