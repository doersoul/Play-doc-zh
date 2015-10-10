#Play Slick FAQ


##我应该用哪个版本?
查阅 [兼容性矩阵](https://github.com/playframework/play-slick#releases) 以了解你应该使用哪个版本。


##`play.db.pool` 被忽略
的确如此。更变`play.db.pool`的值不会影响 Slick 连接池的使用。原因很简单，Play Slick模块目前不支持使用和[HikariCP](http://brettwooldridge.github.io/HikariCP/)不同的连接池。


##通过Slick改变使用的连接池
而Slick允许使用和[HikariCP](http://brettwooldridge.github.io/HikariCP/)不同的连接池 (虽然Slick当前仅提供HikariCP的内置支持, 并且如果你想要使用不同的连接池，要求你提供一个[JdbcDataSourceFactory](http://slick.typesafe.com/doc/3.0.0/api/index.html#slick.jdbc.JdbcDataSourceFactory) 的隐式实现), 但是Play Slick目前不允许使用和HikariCP不同的连接池。 如果你发现自己需要此功能, 你可以尝试给我们留言在 [playframework-dev](https://groups.google.com/forum/#!forum/play-framework-dev)。


##绑定到`play.api.db.DBApi` 已经被配置
如果启动Play应用程序时，得到以下的异常:

```
1) A binding to play.api.db.DBApi was already configured at play.api.db.slick.evolutions.EvolutionsModule.bindings:
Binding(interface play.api.db.DBApi to ConstructionTarget(class play.api.db.slick.evolutions.internal.DBApiAdapter) in interface javax.inject.Singleton).
 at play.api.db.DBModule.bindings(DBModule.scala:25):
Binding(interface play.api.db.DBApi to ProviderConstructionTarget(class play.api.db.DBApiProvider))
```

很有可能你已经[启用jdbc插件](https://www.playframework.com/documentation/2.4.x/ScalaDatabase), 并且如果你使用Slick来访问你的数据库，这样做法根本没道理。要修复这个issue，可以简单地从你的项目构建中移除 Play jdbc组件。

另外一个可能是有另一个Play模块绑定了[DBApi](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/db/DBApi.html) 到其它一些具体的实现。这意味着你仍在尝试使用 Play Slick和另一个Play的数据库访问模块一起工作, 这可能不是你所想的。


##Play 抛出异常 

###`java.lang.ClassNotFoundException: org.h2.tools.Server`

如果你在启动Play应用程序时获得以下异常:

```
java.lang.ClassNotFoundException: org.h2.tools.Server
        at java.net.URLClassLoader$1.run(URLClassLoader.java:372)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:361)
        ...
```

它意味着你要尝试使用H2数据库,但忘记在你的项目构建中添加它的依赖。修复这个问题很简单, 只要添加丢失的依赖到你的项目构建文件, 如,

```sbt
"com.h2database" % "h2" % "${H2_VERSION}" // replace `${H2_VERSION}` with an actual version number
```