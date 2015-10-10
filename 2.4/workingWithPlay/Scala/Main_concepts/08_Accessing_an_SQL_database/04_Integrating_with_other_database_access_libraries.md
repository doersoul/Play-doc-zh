#集成其它数据库访问库
你可以使用任何你喜欢和Play一起用的 **SQL** 数据库访问库, 可以简单的从`play.api.db.DB` 助手获取 `Connection` 或`Datasource` 。


##集成ScalaQuery
到这里你可以集成任何JDBC访问层，它需要一个JDBC数据源。举例, 要集成 [ScalaQuery](https://github.com/szeiger/scala-query) :

```scala
import play.api.db._
import play.api.Play.current

import org.scalaquery.ql._
import org.scalaquery.ql.TypeMapper._
import org.scalaquery.ql.extended.{ExtendedTable => Table}

import org.scalaquery.ql.extended.H2Driver.Implicit._ 

import org.scalaquery.session._

object Task extends Table[(Long, String, Date, Boolean)]("tasks") {
    
  lazy val database = Database.forDataSource(DB.getDataSource())
  
  def id = column[Long]("id", O PrimaryKey, O AutoInc)
  def name = column[String]("name", O NotNull)
  def dueDate = column[Date]("due_date")
  def done = column[Boolean]("done")
  def * = id ~ name ~ dueDate ~ done
  
  def findAll = database.withSession { implicit db:Session =>
      (for(t <- this) yield t.id ~ t.name).list
  }
  
}
```


##通过 JNDI 扩展数据源
有些库期望从JNDI获取`Datasource` 的引用。你可以在`conf/application.conf`中添加以下配置，来通过JNDI展示Play管理的任何数据源:

```scala
db.default.driver=org.h2.Driver
db.default.url="jdbc:h2:mem:play"
db.default.jndiName=DefaultDS
```