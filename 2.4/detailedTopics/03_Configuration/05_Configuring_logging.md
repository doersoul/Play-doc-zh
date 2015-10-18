#配置日志

Play 使用 [Logback](http://logback.qos.ch/) 作为它的日志引擎, 参阅 [Logback 文档](http://logback.qos.ch/manual/configuration.html) 了解更多详细配置。


##默认配置
Play在生产模式使用以下默认配置:

```xml
<!--
  ~ Copyright (C) 2009-2015 Typesafe Inc. <http://www.typesafe.com>
  -->
<!-- The default logback configuration that Play uses if no other configuration is provided -->
<configuration>
    
  <conversionRule conversionWord="coloredLevel" converterClass="play.api.Logger$ColoredLevel" />

  <appender name="FILE" class="ch.qos.logback.core.FileAppender">
     <file>${application.home}/logs/application.log</file>
     <encoder>
       <pattern>%date [%level] from %logger in %thread - %message%n%xException</pattern>
     </encoder>
  </appender>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%coloredLevel %logger{15} - %message%n%xException{10}</pattern>
    </encoder>
  </appender>

  <appender name="ASYNCFILE" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="FILE" />
  </appender>

  <appender name="ASYNCSTDOUT" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="STDOUT" />
  </appender>

  <logger name="play" level="INFO" />
  <logger name="application" level="DEBUG" />
  
  <!-- Off these ones as they are annoying, and anyway we manage configuration ourself -->
  <logger name="com.avaje.ebean.config.PropertyMapLoader" level="OFF" />
  <logger name="com.avaje.ebeaninternal.server.core.XmlConfigLoader" level="OFF" />
  <logger name="com.avaje.ebeaninternal.server.lib.BackgroundThread" level="OFF" />
  <logger name="com.gargoylesoftware.htmlunit.javascript" level="OFF" />

  <root level="WARN">
    <appender-ref ref="ASYNCFILE" />
    <appender-ref ref="ASYNCSTDOUT" />
  </root>
  
</configuration>
```

有关此配置的要注意几点:

* 这个指定了一个文件输出到`logs/application.log`。
* 这个文件记录完整的异常堆栈跟踪, 而控制台日志只记录10行的异常堆栈跟踪。
* Play的级别消息默认使用 ANSI 颜色代码。
* Play 将控制台和文件日志放在logback [AsyncAppender](http://logback.qos.ch/manual/appenders.html#AsyncAppender)后面。要了解这对性能影响的详细信息，参阅[博客文章](https://blog.takipi.com/how-to-instantly-improve-your-java-logging-with-7-logback-tweaks/)。


##自定义配置
对于任何自定义配置, 你需要指定你自己的 Logback 配置文件。

###从项目源码使用配置文件
你可以通过提供一个`conf/logback.xml` 文件来提供默认日志配置。

###使用外部配置文件
你也可以能过系统属性来指定一个配置文件。这在生产环境是特别有用的，配置文件可以在应用程序源文件之外管理。

> 注意: 对于由系统属性来指定配置文件，日志系统是给予最高优先级的, 其次是在`conf` 目录中的文件, 最后才是默认的。这允许你自定义你的应用程序的日志配置，并且还可为特定环境或开发配置而重写它。

####使用 `-Dlogger.resource`
指定一个配置文件以从类路径加载:

```shell
$ start -Dlogger.resource=prod-logger.xml
```

####使用 `-Dlogger.file`
指定一个配置文件以从文件系统加载:

```shell
$ start -Dlogger.file=/opt/prod/logger.xml
```

###示例
这里是一个配置示例，使用RollingFileAppender，以及分离的输出和访问记录:

```xml
<configuration>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${user.dir}/web/logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- Daily rollover with compression -->
            <fileNamePattern>application-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 30 days worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date{yyyy-MM-dd HH:mm:ss ZZZZ} [%level] from %logger in %thread - %message%n%xException</pattern>
        </encoder>
    </appender>
    
    <appender name="ACCESS_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${user.dir}/web/logs/access.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover with compression -->
            <fileNamePattern>access-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 1 week worth of history -->
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date{yyyy-MM-dd HH:mm:ss ZZZZ} %message%n</pattern>
            <!-- this quadruples logging throughput -->
            <immediateFlush>false</immediateFlush>
        </encoder>
    </appender>

    <!-- additivity=false ensures access log data only goes to the access log -->
    <logger name="access" level="INFO" additivity="false">
        <appender-ref ref="ACCESS_FILE" />
    </logger>
    
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>

</configuration>
```

这展示了几个有用的功能:
- 使用`RollingFileAppender` ，可以帮助管理越来越多的日志文件。
- 它把日志文件写到应用程序外部目录，因此他们不受升级的影响，等等。
- `FILE` 输出使用扩展消息格式，可以被第三方日志分析提供者解析，如Sumo Logic。
- `access` 记录路由到一个单独的日志文件，使用`ACCESS_FILE_APPENDER`。
- 所有记录设置到一个 `INFO` 阈值，这是生产日志的一个公共选择。


##Akka 日志配置
Akka 有它自己的日志系统，可能用也可能不用 Play底层的日志引擎，这取决于如何配置。

默认情况下, Akka会忽略Play的日志配置，并使用自己的格式打印日志消息到 STDOUT。你可以在`application.conf` 中配置日志级别:

```scala
akka {
  loglevel="INFO"
}
```

要让Akka直接使用Play的日志引擎, 你需要一些小心配置。首先添加下面的配置到`application.conf`:

```scala
akka {
  loggers = ["akka.event.slf4j.Slf4jLogger"]
  loglevel="DEBUG"
}
```

有几个注意事项:

* 设置 `akka.loggers` 到 `["akka.event.slf4j.Slf4jLogger"]` 会引致 Akka 使用Play的底层日志引擎。
* `akka.loglevel` 属性设置阈值到Akka会控制将记录请求转发到日志引擎但不控制日志输出。一旦记录请求被转发, Logback 配置控制日志级别并作为正常输出。你应该设置`akka.loglevel` 到最低阈值，这样可以为你的Akka组件使用Logback配置。

然后, 在Logback配置中优化你的 Akka 日志设置:

```xml
<!-- Set logging for all Akka library classes to INFO -->
<logger name="akka" level="INFO" />
<!-- Set a specific actor to DEBUG -->
<logger name="actors.MyActor" level="DEBUG" />
```

你也可能希望为Akka日志配置一个输出源，包括有用的属性，如线程和actor地址。

关于配置Akka日志的配置信息, 包括Logback和Slf4j集成的细节，请参阅[Akka 文档](http://doc.akka.io/docs/akka/current/scala/logging.html)。