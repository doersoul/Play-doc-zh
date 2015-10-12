#Testing with databases

While it is possible to write functional tests using [ScalaTest](https://playframework.com/documentation/2.4.x/ScalaFunctionalTestingWithScalaTest) or [specs2](https://playframework.com/documentation/2.4.x/ScalaFunctionalTestingWithSpecs2) that test database access code by starting up a full application including the database, starting up a full application is not often desirable, due to the complexity of having many more components started and running just to test one small part of your application.

Play provides a number of utilities for helping to test database access code that allow it to be tested with a database but in isolation from the rest of your app. These utilities can easily be used with either ScalaTest or specs2, and can make your database tests much closer to lightweight and fast running unit tests than heavy weight and slow functional tests.


##Using a database
To connect to a database, at a minimum, you just need database driver name and the url of the [`database`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/Databases$.html), using the Databases companion object. For example, to connect to MySQL, you might use the following:

```scala
import play.api.db.Databases

val database = Databases(
  driver = "com.mysql.jdbc.Driver",
  url = "jdbc:mysql://localhost/test"
)
```

This will create a database connection pool for the MySQL `test` database running on `localhost`, with the name `default`. The name of the database is only used internally by Play, for example, by other features such as evolutions, to load resources associated with that database.

You may want to specify other configuration for the database, including a custom name, or configuration properties such as usernames, passwords and the various connection pool configuration items that Play supports, by supplying a custom name parameter and/or a custom config parameter:

```scala
import play.api.db.Databases

val database = Databases(
  driver = "com.mysql.jdbc.Driver",
  url = "jdbc:mysql://localhost/test",
  name = "mydatabase",
  config = Map(
    "user" -> "test",
    "password" -> "secret"
  )
)
```

After using a database, since the database is typically backed by a connection pool that holds open connections and may also have running threads, you need to shut it down. This is done by calling the `shutdown` method:

```scala
database.shutdown()
```

Manually creating the database and shutting it down is useful if you’re using a test framework that runs startup/shutdown code around each test or suite. Otherwise it’s recommended that you let Play manage the connection pool for you.

###Allowing Play to manage the database for you
Play also provides a `withDatabase` helper that allows you to supply a block of code to execute with a database connection pool managed by Play. Play will ensure that it is correctly shutdown after the block of code finishes executing:

```scala
import play.api.db.Databases

Databases.withDatabase(
  driver = "com.mysql.jdbc.Driver",
  url = "jdbc:mysql://localhost/test"
) { database =>
  val connection = database.getConnection()
  // ...
}
```

Like the `Database.apply` factory method, `withDatabase` also allows you to pass a custom `name` and `config` map if you please.

Typically, using `withDatabase` directly from every test is an excessive amount of boilerplate code. It is recommended that you create your own helper to remove this boiler plate that your test uses. For example:


```scala
import play.api.db.{Database, Databases}

def withMyDatabase[T](block: Database => T) = {
  Databases.withDatabase(
    driver = "com.mysql.jdbc.Driver",
    url = "jdbc:mysql://localhost/test",
    name = "mydatabase",
    config = Map(
      "user" -> "test",
      "password" -> "secret"
    )
  )(block)
}
```

Then it can be easily used in each test with minimal boilerplate:

```scala
withMyDatabase { database =>
  val connection = database.getConnection()
  // ...
}
```

> **Tip**: You can use this to externalise your test database configuration, using environment variables or system properties to configure what database to use and how to connect to it. This allows for maximum flexibility for developers to have their own environments set up the way they please, as well as for CI systems that provide particular environments that may differ to development.

###Using an in-memory database
Some people prefer not to require infrastructure such as databases to be installed in order to run tests. Play provides simple helpers to create an H2 in-memory database for these purposes:

```scala
import play.api.db.Databases

val database = Databases.inMemory()
```

The in-memory database can be configured, by supplying a custom name, custom URL arguments, and custom connection pool configuration. The following shows supplying the `MODE` argument to tell H2 to emulate `MySQL`, as well as configuring the connection pool to log all statements:

```scala
import play.api.db.Databases

val database = Databases.inMemory(
  name = "mydatabase",
  urlOptions = Map(
    "MODE" -> "MYSQL"
  ),
  config = Map(
    "logStatements" -> true
  )
)
```

As with the generic database factory, ensure you always shut the in-memory database connection pool down:

```scala
database.shutdown()
```

If you’re not using a test frameworks before/after capabilities, you may want Play to manage the in-memory database lifecycle for you, this is straightforward using `withInMemory`:

```scala
import play.api.db.Databases

Databases.withInMemory() { database =>
  val connection = database.getConnection()
  
  // ...
}
```

Like `withDatabase`, it is recommended that to reduce boilerplate code, you create your own method that wraps the `withInMemory` call:

```scala
import play.api.db.{Database, Databases}

def withMyDatabase[T](block: Database => T) = {
  Databases.withInMemory(
    name = "mydatabase",
    urlOptions = Map(
      "MODE" -> "MYSQL"
    ),
    config = Map(
      "logStatements" -> true
    )
  )(block)
}
```


##Applying evolutions
When running tests, you will typically want your database schema managed for your database. If you’re already using evolutions, it will often make sense to reuse the same evolutions that you use in development and production in your tests. You may also want to create custom evolutions just for testing. Play provides some convenient helpers to apply and manage evolutions without having to run a whole Play application.

To apply evolutions, you can use `applyEvolutions` from the [`Evolutions`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/evolutions/Evolutions$.html) companion object:

```scala
import play.api.db.evolutions._

Evolutions.applyEvolutions(database)
```

This will load the evolutions from the classpath in the `evolutions/<databasename>` directory, and apply them.

After a test has run, you may want to reset the database to its original state. If you have implemented your evolutions down scripts in such a way that they will drop all the database tables, you can do this simply by calling the `cleanupEvolutions` method:

```scala
Evolutions.cleanupEvolutions(database)
```

###Custom evolutions
In some situations you may want to run some custom evolutions in your tests. Custom evolutions can be used by using a custom [`EvolutionsReader`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/evolutions/EvolutionsReader.html). The simplest of these is the [`SimpleEvolutionsReader`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/evolutions/SimpleEvolutionsReader.html), which is an evolutions reader that takes a preconfigured map of database names to sequences of [`Evolution`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/evolutions/Evolution.html) scripts, and can be constructed using the convenient methods on the [`SimpleEvolutionsReader`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/evolutions/SimpleEvolutionsReader$.html) companion object. For example:

```scala
import play.api.db.evolutions._

Evolutions.applyEvolutions(database, SimpleEvolutionsReader.forDefault(
  Evolution(
    1,
    "create table test (id bigint not null, name varchar(255));",
    "drop table test;"
  )
))
```

Cleaning up custom evolutions is done in the same way as cleaning up regular evolutions, using the `cleanupEvolutions` method:

```scala
Evolutions.cleanupEvolutions(database)
```

Note though that you don’t need to pass the custom evolutions reader here, this is because the state of the evolutions is stored in the database, including the down scripts which will be used to tear down the database.

Sometimes it will be impractical to put your custom evolution scripts in code. If this is the case, you can put them in the test resources directory, under a custom path using the [`ClassLoaderEvolutionsReader`](https://playframework.com/documentation/2.4.x/api/scala/play/api/db/evolutions/ClassLoaderEvolutionsReader.html). For example:

```scala
import play.api.db.evolutions._

Evolutions.applyEvolutions(database, ClassLoaderEvolutionsReader.forPrefix("testdatabase/"))
```

This will load evolutions, in the same structure and format as is done for development and production, from `testdatabase/evolutions/<databasename>/<n>.sql`.

###Allowing Play to manage evolutions
The `applyEvolutions` and `cleanupEvolutions` methods are useful if you’re using a test framework to manage running the evolutions before and after a test. Play also provides a convenient `withEvolutions` method to manage it for you, if this lighter weight approach is desired:

```scala
import play.api.db.evolutions._

Evolutions.withEvolutions(database) {
  val connection = database.getConnection()

  // ...
}
```

Naturally, `withEvolutions` can be combined with `withDatabase` or `withInMemory` to reduce boilerplate code, allowing you to define a function that both instantiates the database and runs evolutions for you:

```scala
import play.api.db.{Database, Databases}
import play.api.db.evolutions._

def withMyDatabase[T](block: Database => T) = {

  Databases.withInMemory(
    urlOptions = Map(
      "MODE" -> "MYSQL"
    ),
    config = Map(
      "logStatements" -> true
    )
  ) { database =>

    Evolutions.withEvolutions(database, SimpleEvolutionsReader.forDefault(
      Evolution(
        1,
        "create table test (id bigint not null, name varchar(255));",
        "drop table test;"
      )
    )) {

      block(database)

    }
  }
}
```

Having defined the custom database management method for our tests, we can now use them in a straight forward manner:

```scala
withMyDatabase { database =>
  val connection = database.getConnection()
  connection.prepareStatement("insert into test values (10, 'testing')").execute()

  connection.prepareStatement("select * from test where id = 10")
    .executeQuery().next() must_== true
}
```