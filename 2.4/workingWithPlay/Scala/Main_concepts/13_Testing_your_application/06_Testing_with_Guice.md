#Testing with Guice

If you’re using Guice for [dependency injection](https://playframework.com/documentation/2.4.x/ScalaDependencyInjection) then you can directly configure how components and applications are created for tests. This includes adding extra bindings or overriding existing bindings.


##GuiceApplicationBuilder
[GuiceApplicationBuilder](https://playframework.com/documentation/2.4.x/api/scala/play/api/inject/guice/GuiceApplicationBuilder.html) provides a builder API for configuring the dependency injection and creation of an [Application](https://playframework.com/documentation/2.4.x/api/scala/play/api/Application.html).

###Environment
The [Environment](https://playframework.com/documentation/2.4.x/api/scala/play/api/Environment.html), or parts of the environment such as the root path, mode, or class loader for an application, can be specified. The configured environment will be used for loading the application configuration, it will be used when loading modules and passed when deriving bindings from Play modules, and it will be injectable into other components.

```scala
import play.api.inject.guice.GuiceApplicationBuilder
val application = new GuiceApplicationBuilder()
  .in(Environment(new File("path/to/app"), classLoader, Mode.Test))
  .build
val application = new GuiceApplicationBuilder()
  .in(new File("path/to/app"))
  .in(Mode.Test)
  .in(classLoader)
  .build
```

###Configuration
Additional configuration can be added. This configuration will always be in addition to the configuration loaded automatically for the application. When existing keys are used the new configuration will be preferred.

```scala
val application = new GuiceApplicationBuilder()
  .configure(Configuration("a" -> 1))
  .configure(Map("b" -> 2, "c" -> "three"))
  .configure("d" -> 4, "e" -> "five")
  .build
```

The automatic loading of configuration from the application environment can also be overridden. This will completely replace the application configuration. For example:

```scala
val application = new GuiceApplicationBuilder()
  .loadConfig(env => Configuration.load(env))
  .build
```

###Bindings and Modules
The bindings used for dependency injection are completely configurable. The builder methods support [Play Modules and Bindings](https://playframework.com/documentation/2.4.x/ScalaDependencyInjection) and also Guice Modules.

####Additional bindings
Additional bindings, via Play modules, Play bindings, or Guice modules, can be added:

```scala
import play.api.inject.bind
val injector = new GuiceApplicationBuilder()
  .bindings(new ComponentModule)
  .bindings(bind[Component].to[DefaultComponent])
  .injector
```

####Override bindings
Bindings can be overridden using Play bindings, or modules that provide bindings. For example:

```scala
val application = new GuiceApplicationBuilder()
  .overrides(bind[Component].to[MockComponent])
  .build
```

####Disable modules
Any loaded modules can be disabled by class name:

```scala
val injector = new GuiceApplicationBuilder()
  .disable[ComponentModule]
  .injector
```

####Loaded modules
Modules are automatically loaded from the classpath based on the `play.modules.enabled` configuration. This default loading of modules can be overridden. For example:

```scala
val injector = new GuiceApplicationBuilder()
  .load(
    new play.api.inject.BuiltinModule,
    bind[Component].to[DefaultComponent]
  ).injector
```


##GuiceInjectorBuilder
[GuiceInjectorBuilder](https://playframework.com/documentation/2.4.x/api/scala/play/api/inject/guice/GuiceInjectorBuilder.html) provides a builder API for configuring Guice dependency injection more generally. This builder does not load configuration or modules automatically from the environment like `GuiceApplicationBuilder`, but provides a completely clean state for adding configuration and bindings. The common interface for both builders can be found in [GuiceBuilder](https://playframework.com/documentation/2.4.x/api/scala/play/api/inject/guice/GuiceBuilder.html). A Play [Injector](https://playframework.com/documentation/2.4.x/api/scala/play/api/inject/Injector.html) is created. Here’s an example of instantiating a component using the injector builder:

```scala
import play.api.inject.guice.GuiceInjectorBuilder
import play.api.inject.bind
val injector = new GuiceInjectorBuilder()
  .configure("key" -> "value")
  .bindings(new ComponentModule)
  .overrides(bind[Component].to[MockComponent])
  .injector

val component = injector.instanceOf[Component]
```


##Overriding bindings in a functional test
Here is a full example of replacing a component with a mock component for testing. Let’s start with a component, that has a default implementation and a mock implementation for testing:

```scala
trait Component {
  def hello: String
}

class DefaultComponent extends Component {
  def hello = "default"
}

class MockComponent extends Component {
  def hello = "mock"
}
```

This component is loaded automatically using a module:

```scala
import play.api.{ Environment, Configuration }
import play.api.inject.Module

class ComponentModule extends Module {
  def bindings(env: Environment, conf: Configuration) = Seq(
    bind[Component].to[DefaultComponent]
  )
}
```

And the component is used in a controller:

```scala
import play.api.mvc._
import javax.inject.Inject

class Application @Inject() (component: Component) extends Controller {
  def index() = Action {
    Ok(component.hello)
  }
}
```

To build an `Application` to use in functional tests we can simply override the binding for the component:

```scala
import play.api.inject.guice.GuiceApplicationBuilder
import play.api.inject.bind
val application = new GuiceApplicationBuilder()
  .overrides(bind[Component].to[MockComponent])
  .build
```

The created application can be used with the functional testing helpers for [Specs2](https://playframework.com/documentation/2.4.x/ScalaFunctionalTestingWithSpecs2) and [ScalaTest](https://playframework.com/documentation/2.4.x/ScalaFunctionalTestingWithScalaTest).