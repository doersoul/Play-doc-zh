#聚合（Aggregating）反向路由

在某些情况下，你可能想要在彼此不依赖的子项目之间共享反向路由。

例如, 你可能有一个`web` 子项目和一个`api` 子项目。这些子项目没有彼此依赖, 除了`web` 项目有链接指向`api` 项目 (为创建AJAX调用), 而`api` 项目想要链接到`web` (JSON资源的web链接)。在这种情况下, 使用反向路由是很方便的, 但因为这些项目不依赖对方，所以无法实现。

Play’s routes compiler offers a feature that allows a common dependency to generate the reverse routers for projects that depend on it so that the reverse routers can be shared between those projects. This is configured using the `aggregateReverseRoutes` sbt configuration item, like this:

```sbt
lazy val common: Project = (project in file("common"))
  .enablePlugins(PlayScala)
  .settings(
    aggregateReverseRoutes := Seq(api, web)
  )

lazy val api = (project in file("api"))
  .enablePlugins(PlayScala)
  .dependsOn(common)

lazy val web = (project in file("web"))
  .enablePlugins(PlayScala)
  .dependsOn(common)
```

In this setup, the reverse routers for `api` and `web` will be generated as part of the `common` project. Meanwhile, the forwards routers for `api` and `web` will still generate forwards routers, but not reverse routers, because their reverse routers have already been generated in the `common` project which they depend on, so they don’t need to generate them.

> Note that the `common` project has a type of `Project` explicitly declared. This is because there is a recursive reference between it and the `api` and `web` projects, through the `dependsOn` method and `aggregateReverseRoutes` setting, so the Scala type checker needs an explicit type somewhere in the chain of recursion.