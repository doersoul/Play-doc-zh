#配置 Play 线程池

Play框架是从上到下都是一个异步web框架。使用iteratees异步处理流。和传统的web框架比较，Play是的线程池是调整为较少的, 因为在play核心，IO永不会阻塞。

正因为这个, 如果你计划写阻塞IO的代码, 或代码可能会做很CPU密集的工作, 你需要确切地知道其中线程池负担的工作量, 并相应的调整。做阻塞IO没有考虑到这可能会导致Play框架的性能变差, 例如, 您可以看到只有少数每秒请求被处理, 而 CPU 使用在5%。相比之下, 在典型的开发硬件的基准 (比如, 一个MacBook Pro)，显示Play在正确设置时，可以处理的工作量在数以百计甚至数以千计的每秒请求数，一点压力都没有。


##当你以阻塞工作时要知道的
The most common place where a typical Play application will block is when it’s talking to a database. Unfortunately, none of the major databases provide asynchronous database drivers for the JVM, so for most databases, your only option is to using blocking IO. A notable exception to this is [ReactiveMongo](http://reactivemongo.org/), a driver for MongoDB that uses Play’s Iteratee library to talk to MongoDB.

Other cases when your code may block include:

* Using REST/WebService APIs through a 3rd party client library (ie, not using Play’s asynchronous WS API)
* Some messaging technologies only provide synchronous APIs to send messages
* When you open files or sockets directly yourself
* CPU intensive operations that block by virtue of the fact that they take a long time to execute

In general, if the API you are using returns `Futures`, it is non-blocking, otherwise it is blocking.

> Note that you may be tempted to therefore wrap your blocking code in Futures. This does not make it non-blocking, it just means the blocking will happen in a different thread. You still need to make sure that the thread pool that you are using has enough threads to handle the blocking.

In contrast, the following types of IO do not block:

* The Play WS API
* Asynchronous database drivers such as ReactiveMongo
* Sending/receiving messages to/from Akka actors


##Play的线程池
Play uses a number of different thread pools for different purposes:

* **Netty boss/worker thread pools** - These are used internally by Netty for handling Netty IO. An application’s code should never be executed by a thread in these thread pools.
* **Play default thread pool** - This is the thread pool in which all of your application code in Play Framework is executed. It is an Akka dispatcher, and is used by the application `ActorSystem`. It can be configured by configuring Akka, described below.

> Note that in Play 2.4 several thread pools were combined together into the Play default thread pool.


##使用默认线程池
All actions in Play Framework use the default thread pool. When doing certain asynchronous operations, for example, calling `map` or `flatMap` on a future, you may need to provide an implicit execution context to execute the given functions in. An execution context is basically another name for a `ThreadPool`.

In most situations, the appropriate execution context to use will be the **Play default thread pool**. This can be used by importing it into your Scala source file:

```scala
import play.api.libs.concurrent.Execution.Implicits._

def someAsyncAction = Action.async {
  import play.api.Play.current
  WS.url("http://www.playframework.com").get().map { response =>
    // This code block is executed in the imported default execution context
    // which happens to be the same thread pool in which the outer block of
    // code in this action will be executed.
    Results.Ok("The response code was " + response.status)
  }
}
```

###配置Play默认线程池
The default thread pool can be configured using standard Akka configuration in `application.conf` under the `akka` namespace. Here is default configuration for Play’s thread pool:

```scala
akka {
  fork-join-executor {
    # Settings this to 1 instead of 3 seems to improve performance.
    parallelism-factor = 1.0

    parallelism-max = 24

    # Setting this to LIFO changes the fork-join-executor
    # to use a stack discipline for task scheduling. This usually
    # improves throughput at the cost of possibly increasing
    # latency and risking task starvation (which should be rare).
    task-peeking-mode = LIFO
  }
}
```

This configuration instructs Akka to create 1 thread per available processor, with a maximum of 24 threads in the pool.

You can also try the default Akka configuration:

```scala
akka {
  fork-join-executor {
    # The parallelism factor is used to determine thread pool size using the
    # following formula: ceil(available processors * factor). Resulting size
    # is then bounded by the parallelism-min and parallelism-max values.
    parallelism-factor = 3.0

    # Min number of threads to cap factor-based parallelism number to
    parallelism-min = 8

    # Max number of threads to cap factor-based parallelism number to
    parallelism-max = 64
  }
}
```

The full configuration options available to you can be found [here](http://doc.akka.io/docs/akka/2.3.11/general/configuration.html#Listing_of_the_Reference_Configuration).


##使用其它线程池
In certain circumstances, you may wish to dispatch work to other thread pools. This may include CPU heavy work, or IO work, such as database access. To do this, you should first create a `ThreadPool`, this can be done easily in Scala:

```scala
object Contexts {
  implicit val myExecutionContext: ExecutionContext = Akka.system.dispatchers.lookup("my-context")
}
```

In this case, we are using Akka to create the `ExecutionContext`, but you could also easily create your own `ExecutionContexts` using Java executors, or the Scala fork join thread pool, for example. To configure this Akka execution context, you can add the following configuration to your `application.conf`:

```scala
my-context {
  fork-join-executor {
    parallelism-factor = 20.0
    parallelism-max = 200
  }
}
```

To use this execution context in Scala, you would simply use the scala `Future` companion object function:

```scala
Future {
  // Some blocking or expensive code here
}(Contexts.myExecutionContext)
```

or you could just use it implicitly:

```scala
import Contexts.myExecutionContext

Future {
  // Some blocking or expensive code here
}
```


##类加载器和线程局部变量
Class loaders and thread locals need special handling in a multithreaded environment such as a Play program.

###应用程序类加载器
In a Play application the [thread context class loader](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#getContextClassLoader--) may not always be able to load application classes. You should explicitly use the application class loader to load classes.

```scala
val myClass = Play.current.classloader.loadClass(myClassName)
```

Being explicit about loading classes is most important when running Play in development mode (using `run`) rather than production mode. That’s because Play’s development mode uses multiple class loaders so that it can support automatic application reloading. Some of Play’s threads might be bound to a class loader that only knows about a subset of your application’s classes.

In some cases you may not be able to explicitly use the application classloader. This is sometimes the case when using third party libraries. In this case you may need to set the [thread context class loader](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html#getContextClassLoader--) explicitly before you call the third party code. If you do, remember to restore the context class loader back to its previous value once you’ve finished calling the third party code.

###Java 线程池局部变量
Java code in Play uses a `ThreadLocal` to find out about contextual information such as the current HTTP request. Scala code doesn’t need to use `ThreadLocals` because it can use implicit parameters to pass context instead. `ThreadLocals` are used in Java so that Java code can access contextual information without needing to pass context parameters everywhere.

Java `ThreadLocals`, along with the correct context `ClassLoader`, are propagated automatically by `ExecutionContextExecutor` objects provided through the `HttpExecution` class. (An `ExecutionContextExecutor` is both a Scala `ExecutionContext` and a Java `Executor`.) These special `ExecutionContextExecutor` objects are automatically created and used by Java actions and Java `Promise` methods. The default objects wrap the default user thread pool. If you want to do your own threading then you should use the `HttpExecution` class’ helper methods to get an `ExecutionContextExecutor` object yourself.

In the example below, a user thread pool is wrapped to create a new `ExecutionContext` that propagates thread locals correctly.

```scala
import play.libs.HttpExecution;
import scala.concurrent.ExecutionContext;
public Promise<Result> index2() {
  // Wrap an existing thread pool, using the context from the current thread
  ExecutionContext myEc = HttpExecution.fromThread(myThreadPool);
  return Promise.promise(() -> intensiveComputation(), myEc)
          .map((Integer i) -> ok("Got result: " + i), myEc);
}
```


##最佳实践
How you should best divide work in your application between different thread pools greatly depends on the types of work that your application is doing, and the control you want to have over how much of which work can be done in parallel. There is no one size fits all solution to the problem, and the best decision for you will come from understanding the blocking-IO requirements of your application and the implications they have on your thread pools. It may help to do load testing on your application to tune and verify your configuration.

> Given the fact that JDBC is blocking thread pools can be sized to the # of connections available to a db pool assuming that the thread pool is used exclusively for database access. Any lesser amount of threads will not consume the number of connections available. Any more threads than the number of connections available could be wasteful given contention for the connections.

Below we outline a few common profiles that people may want to use in Play Framework:

###纯异步
In this case, you are doing no blocking IO in your application. Since you are never blocking, the default configuration of one thread per processor suits your use case perfectly, so no extra configuration needs to be done. The Play default execution context can be used in all cases.

###高度同步
This profile matches that of a traditional synchronous IO based web framework, such as a Java servlet container. It uses large thread pools to handle blocking IO. It is useful for applications where most actions are doing database synchronous IO calls, such as accessing a database, and you don’t want or need control over concurrency for different types of work. This profile is the simplest for handling blocking IO.

In this profile, you would simply use the default execution context everywhere, but configure it to have a very large number of threads in its pool, like so:

```scala
akka {
  akka.loggers = ["akka.event.slf4j.Slf4jLogger"]
  loglevel = WARNING
  actor {
    default-dispatcher = {
      fork-join-executor {
        parallelism-min = 300
        parallelism-max = 300
      }
    }
  }
}
```

This profile is recommended for Java applications that do synchronous IO, since it is harder in Java to dispatch work to other threads.

Note that we use the same value for `parallelism-min` and `parallelism-max`. The reason is that the number of threads is defined by the following formulas :

> base-nb-threads = nb-processors * parallelism-factor
> parallelism-min <= actual-nb-threads <= parallelism-max

So if you don’t have enough available processors, you will never be able to reach the `parallelism-max` setting.

###大量特定线程池
This profile is for when you want to do a lot of synchronous IO, but you also want to control exactly how much of which types of operations your application does at once. In this profile, you would only do non blocking operations in the default execution context, and then dispatch blocking operations to different execution contexts for those specific operations.

In this case, you might create a number of different execution contexts for different types of operations, like this:

```scala
object Contexts {
  implicit val simpleDbLookups: ExecutionContext = Akka.system.dispatchers.lookup("contexts.simple-db-lookups")
  implicit val expensiveDbLookups: ExecutionContext = Akka.system.dispatchers.lookup("contexts.expensive-db-lookups")
  implicit val dbWriteOperations: ExecutionContext = Akka.system.dispatchers.lookup("contexts.db-write-operations")
  implicit val expensiveCpuOperations: ExecutionContext = Akka.system.dispatchers.lookup("contexts.expensive-cpu-operations")
}
```

These might then be configured like so:

```scala
contexts {
  simple-db-lookups {
    fork-join-executor {
      parallelism-factor = 10.0
    }
  }
  expensive-db-lookups {
    fork-join-executor {
      parallelism-max = 4
    }
  }
  db-write-operations {
    fork-join-executor {
      parallelism-factor = 2.0
    }
  }
  expensive-cpu-operations {
    fork-join-executor {
      parallelism-max = 2
    }
  }
}
```

Then in your code, you would create `Futures` and pass the relevant `ExecutionContext` for the type of work that `Future` was doing.

> **Note**: The configuration namespace can be chosen freely, as long as it matches the dispatcher ID passed to `Akka.system.dispatchers.lookup`.

###少量特定线程池
This is a combination between the many specific thread pools and the highly synchronized profile. You would do most simple IO in the default execution context and set the number of threads there to be reasonably high (say 100), but then dispatch certain expensive operations to specific contexts, where you can limit the number of them that are done at one time.