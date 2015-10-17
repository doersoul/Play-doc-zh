#配置应用程序 Secret

Play很多事情都使用密钥, 包括:

* 签署会话cookies和CSRF令牌
* 内置的加密实用程序

它配置在`application.conf`, 用一个叫`play.crypto.secret` 的属性, 并且默认为`changeme`。作为默认的建议, 在生产版本应该更改它。

> 当开始在生产模式时, 如果Play 发现secret没有设置, 或如果它是`changeme`, Play会抛出错误。


##最佳实践
谁能获得secret的访问，就可以生成任何他们喜欢的session, 高效地允许他们作为任何用户登录到你的系统。因此，强烈建议你不要提交你的应用程序secret到源码控制系统中。而是, 它应该配置在你的生产服务器上。这也就是说把应用程序的secret放在`application.conf`中是不好的做法。

在生产服务器配置应用程序secret的一种方式是在启动脚本作为一个系统属性传递。例如:

```scala
/path/to/yourapp/bin/yourapp -Dplay.crypto.secret="QCY?tAnfk?aZ?iwrNwnxIlR6CTf:G3gf:90Latabg@5241AB`R5W:1uDFN];Ik@n"
```

这个方法非常简单, 在Play文档中我们将使用这个方法，以生产模式运行你的应用程序，保留应用程序secret设置。然而在某些环境中, 将secrets放在命令行参数也不是一个好做法。还有第二种方式来处理这个。

###环境变量
首先把应用程序secret放在环境变量。在这种情况下, 我们在你的`application.conf`文件中加上以下设置:

```scala
play.crypto.secret="changeme"
play.crypto.secret=${?APPLICATION_SECRET}
```

配置中的第二行设置secret为来自一个叫`APPLICATION_SECRET` 的环境变量。前提是已经设置了这个环境变量，否则, 它保留secret和前面的一行一样不变。

这种方法特别适用于基于云的部署方案, 通常的做法是通过云提供商的API配置的环境变量来设置密码和其它secrets。

###生产配置文件
另一个方法是创建一个`production.conf` 文件，它存在于服务器, 并包含`application.conf`, 但还会覆盖任何敏感的配置, 如应用程序secret和密码。

例如:

```scala
include "application"

play.crypto.secret="QCY?tAnfk?aZ?iwrNwnxIlR6CTf:G3gf:90Latabg@5241AB`R5W:1uDFN];Ik@n"
```

那么你可以用下面命令启动Play:

```scala
/path/to/yourapp/bin/yourapp -Dconfig.file=/path/to/production.conf
```


##生成一个应用程序secret
Play 提供一个实用工具，你可以用来生成一个新secret。在Play控制台运行`playGenerateSecret` 。这个会生成一个新secret，你可以用在新的应用程序。举例:

```shell
[my-first-app] $ playGenerateSecret
[info] Generated new secret: QCYtAnfkaZiwrNwnxIlR6CTfG3gf90Latabg5241ABR5W1uDFNIkn
[success] Total time: 0 s, completed 28/03/2014 2:26:09 PM
```


##在application.conf更新应用程序secret 
如果你为了开发或测试服务器设一个特定的secret配置，Play也提供一个便利的实用工具来在`application.conf`中更新secret。当你已经有使用应用程序secret加密的数据，这常常是很有用的，和你想确保运行在开发模式下每次都使用同样的secret。

要在`application.conf`更新secret, 在Play控制台运行`playUpdateSecret` :

```shell
[my-first-app] $ playUpdateSecret
[info] Generated new secret: B4FvQWnTp718vr6AHyvdGlrHBGNcvuM4y3jUeRCgXxIwBZIbt
[info] Updating application secret in /Users/jroper/tmp/my-first-app/conf/application.conf
[info] Replacing old application secret: play.crypto.secret="changeme"
[success] Total time: 0 s, completed 28/03/2014 2:36:54 PM
```