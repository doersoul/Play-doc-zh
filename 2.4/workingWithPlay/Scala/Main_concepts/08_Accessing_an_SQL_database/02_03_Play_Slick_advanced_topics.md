#Play Slick 高级应用


##连接池
从 Slick 3.0 释放版本起, Slick 启用和控制连接池和线程池，以优化您的数据库操作的异步执行。

在Play Slick我们已经决定让Slick负责创建和管理连接池 ( Slick 3.0 使用的默认连接池是[HikariCP](http://brettwooldridge.github.io/HikariCP/) ), 这意味着调整连接池，你需要查阅 Slick ScalaDoc [Database.forConfig](http://slick.typesafe.com/doc/3.0.0/api/index.html#slick.jdbc.JdbcBackend$DatabaseFactoryDef@forConfig\(String,Config,Driver\):Database)  (确保展开文档中的`forConfig` 这一行)。实际上, 你要知道任何你可以通过设置连接池传递的值(如 在键`play.db.default.hikaricp`) 是简单不通过 Slick的, 因此效果被忽略。

同样, 注意如上所述在[Slick 文档](http://slick.typesafe.com/docs) 中的, 一个合理的连接池默认大小是从线程池的大小计算的。实际上, 多数情况下你应该为每个数据库配置调整`numThreads` 和`queueSize`。

最后, 值得一提的是，Slick 允许使用和[HikariCP](http://brettwooldridge.github.io/HikariCP/)不同的连接池 (虽然Slick目前仅提供HikariCP的内置支持, 并且如果你想要使用不同的连接池，要求你提供一个[JdbcDataSourceFactory](http://slick.typesafe.com/doc/3.0.0/api/index.html#slick.jdbc.JdbcDataSourceFactory) 的隐式实现), 但是Play Slick目前不允许使用和HikariCP不同的连接池。

> 注意: 更改`play.db.pool` 不会影响Slick连接池的使用。此外, 注意`play.db` 下的任何配置 Play Slick都不考虑。


##线程池
从 Slick 3.0 释放版本, Slick 启用和控制连接池和线程池，以优化您的数据库操作的异步执行。

为优化执行, 你可能需要为每个数据库配置调整`numThreads` 和`queueSize` 参数。参阅[Slick 文档](http://slick.typesafe.com/docs)以了解详情。