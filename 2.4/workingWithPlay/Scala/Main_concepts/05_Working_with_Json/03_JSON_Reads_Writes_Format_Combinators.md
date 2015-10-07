#JSON Reads/Writes/Format Combinators
[JSON basics](https://www.playframework.com/documentation/2.4.x/ScalaJson) introduced [`Reads`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html) and [`Writes`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Writes.html) converters which are used to convert between [`JsValue`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsValue.html) structures and other data types. This page covers in greater detail how to build these converters and how to use validation during conversion.

The examples on this page will use this `JsValue` structure and corresponding model:

```scala
import play.api.libs.json._

val json: JsValue = Json.parse("""
{
  "name" : "Watership Down",
  "location" : {
    "lat" : 51.235685,
    "long" : -1.309197
  },
  "residents" : [ {
    "name" : "Fiver",
    "age" : 4,
    "role" : null
  }, {
    "name" : "Bigwig",
    "age" : 6,
    "role" : "Owsla"
  } ]
}
""")
case class Location(lat: Double, long: Double)
case class Resident(name: String, age: Int, role: Option[String])
case class Place(name: String, location: Location, residents: Seq[Resident])
```


##JsPath
[`JsPath`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsPath.html) is a core building block for creating `Reads`/`Writes`. `JsPath` represents the location of data in a `JsValue` structure. You can use the `JsPath` object (root path) to define a `JsPath` child instance by using syntax similar to traversing `JsValue`:

```scala
import play.api.libs.json._

val json = { ... }

// Simple path
val latPath = JsPath \ "location" \ "lat"

// Recursive path
val namesPath = JsPath \\ "name"

// Indexed path
val firstResidentPath = (JsPath \ "residents")(0)
```

The [`play.api.libs.json`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/package.html) package defines an alias for `JsPath`: `__` (double underscore). You can use this if you prefer:

```scala
val longPath = __ \ "location" \ "long"
```


##Reads
[`Reads`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html) converters are used to convert from a `JsValue` to another type. You can combine and nest `Reads` to create more complex `Reads`.

You will require these imports to create `Reads`:

```scala
import play.api.libs.json._ // JSON library
import play.api.libs.json.Reads._ // Custom validation helpers
import play.api.libs.functional.syntax._ // Combinator syntax
```

###Path Reads
`JsPath` has methods to create special `Reads` that apply another `Reads` to a `JsValue` at a specified path:

* `JsPath.read[T](implicit r: Reads[T]): Reads[T]` - Creates a `Reads[T]` that will apply the implicit argument `r` to the `JsValue` at this path.
* `JsPath.readNullable[T](implicit r: Reads[T]): Reads[Option[T]]readNullable` - Use for paths that may be missing or can contain a null value.

> Note: The JSON library provides implicit `Reads` for basic types such as `String`, `Int`, `Double`, etc.

Defining an individual path `Reads` looks like this:

```scala
val nameReads: Reads[String] = (JsPath \ "name").read[String]
```

###Complex Reads
You can combine individual path `Reads` to form more complex `Reads` which can be used to convert to complex models.

For easier understanding, we’ll break down the combine functionality into two statements. First combine `Reads` objects using the `and` combinator:

```scala
val locationReadsBuilder =
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
```

This will yield a type of `FunctionalBuilder[Reads]#CanBuild2[Double, Double]`. This is an intermediary object and you don’t need to worry too much about it, just know that it’s used to create a complex `Reads`. 

Second call the `apply` method of `CanBuildX` with a function to translate individual values to your model, this will return your complex `Reads`. If you have a case class with a matching constructor signature, you can just use its `apply` method:

```scala
implicit val locationReads = locationReadsBuilder.apply(Location.apply _)
```

Here’s the same code in a single statement:

```scala
implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)
```

###Validation with Reads
The `JsValue.validate` method was introduced in [JSON basics](https://www.playframework.com/documentation/2.4.x/ScalaJson) as the preferred way to perform validation and conversion from a `JsValue` to another type. Here’s the basic pattern:

```scala
val json = { ... }

val nameReads: Reads[String] = (JsPath \ "name").read[String]

val nameResult: JsResult[String] = json.validate[String](nameReads)

nameResult match {
  case s: JsSuccess[String] => println("Name: " + s.get)
  case e: JsError => println("Errors: " + JsError.toFlatJson(e).toString())
}
```

Default validation for `Reads` is minimal, such as checking for type conversion errors. You can define custom validation rules by using `Reads` validation helpers. Here are some that are commonly used: 

* `Reads.email` - Validates a String has email format.
* `Reads.minLength(nb)` - Validates the minimum length of a String.
* `Reads.min` - Validates a minimum numeric value.
* `Reads.max` - Validates a maximum numeric value.
* `Reads[A] keepAnd Reads[B] => Reads[A]` - Operator that tries `Reads[A]` and `Reads[B]` but only keeps the result of `Reads[A]` (For those who know Scala parser combinators `keepAnd == <~` ).
* `Reads[A] andKeep Reads[B] => Reads[B]` - Operator that tries `Reads[A]` and `Reads[B]` but only keeps the result of `Reads[B]` (For those who know Scala parser combinators `andKeep == ~>` ).
* `Reads[A] or Reads[B] => Reads` - Operator that performs a logical OR and keeps the result of the last `Reads` checked.

To add validation, apply helpers as arguments to the `JsPath.read` method:

```scala
val improvedNameReads =
  (JsPath \ "name").read[String](minLength[String](2))
```

###Putting it all together
By using complex `Reads` and custom validation we can define a set of effective `Reads` for our example model and apply them:

```scala
import play.api.libs.json._
import play.api.libs.json.Reads._
import play.api.libs.functional.syntax._

implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").read[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply _)

implicit val residentReads: Reads[Resident] = (
  (JsPath \ "name").read[String](minLength[String](2)) and
  (JsPath \ "age").read[Int](min(0) keepAnd max(150)) and
  (JsPath \ "role").readNullable[String]
)(Resident.apply _)

implicit val placeReads: Reads[Place] = (
  (JsPath \ "name").read[String](minLength[String](2)) and
  (JsPath \ "location").read[Location] and
  (JsPath \ "residents").read[Seq[Resident]]
)(Place.apply _)


val json = { ... }

json.validate[Place] match {
  case s: JsSuccess[Place] => {
    val place: Place = s.get
    // do something with place
  }
  case e: JsError => {
    // error handling flow
  }
}
```

Note that complex `Reads` can be nested. In this case, `placeReads` uses the previously defined implicit `locationReads` and `residentReads` at specific paths in the structure.


##Writes
[`Writes`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Writes.html) converters are used to convert from some type to a `JsValue`.

You can build complex `Writes` using `JsPath` and combinators very similar to `Reads`. Here’s the `Writes` for our example model:

```scala
import play.api.libs.json._
import play.api.libs.functional.syntax._

implicit val locationWrites: Writes[Location] = (
  (JsPath \ "lat").write[Double] and
  (JsPath \ "long").write[Double]
)(unlift(Location.unapply))

implicit val residentWrites: Writes[Resident] = (
  (JsPath \ "name").write[String] and
  (JsPath \ "age").write[Int] and
  (JsPath \ "role").writeNullable[String]
)(unlift(Resident.unapply))

implicit val placeWrites: Writes[Place] = (
  (JsPath \ "name").write[String] and
  (JsPath \ "location").write[Location] and
  (JsPath \ "residents").write[Seq[Resident]]
)(unlift(Place.unapply))


val place = Place(
  "Watership Down",
  Location(51.235685, -1.309197),
  Seq(
    Resident("Fiver", 4, None),
    Resident("Bigwig", 6, Some("Owsla"))
  )
)

val json = Json.toJson(place)
```

There are a few differences between complex `Writes` and `Reads`:

* The individual path `Writes` are created using the `JsPath.write` method.
* There is no validation on conversion to `JsValue` which makes the structure simpler and you won’t need any validation helpers.
* The intermediary `FunctionalBuilder#CanBuildX` (created by `and` combinators) takes a function that translates a complex type `T` to a tuple matching the individual path `Writes`. Although this is symmetrical to the `Reads` case, the `unapply` method of a case class returns an `Option` of a tuple of properties and must be used with `unlift` to extract the tuple.


##Recursive Types
One special case that our example model doesn’t demonstrate is how to handle `Reads` and `Writes` for recursive types. `JsPath` provides `lazyRead` and `lazyWrite` methods that take call-by-name parameters to handle this:

```scala
case class User(name: String, friends: Seq[User])

implicit lazy val userReads: Reads[User] = (
  (__ \ "name").read[String] and
  (__ \ "friends").lazyRead(Reads.seq[User](userReads))
)(User)

implicit lazy val userWrites: Writes[User] = (
  (__ \ "name").write[String] and
  (__ \ "friends").lazyWrite(Writes.seq[User](userWrites))
)(unlift(User.unapply))
```


##Format
[`Format[T]`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Format.html) is just a mix of the `Reads` and `Writes` traits and can be used for implicit conversion in place of its components.

###Creating Format from Reads and Writes
You can define a `Format` by constructing it from `Reads` and `Writes` of the same type:

```scala
val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").read[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply _)

val locationWrites: Writes[Location] = (
  (JsPath \ "lat").write[Double] and
  (JsPath \ "long").write[Double]
)(unlift(Location.unapply))

implicit val locationFormat: Format[Location] =
  Format(locationReads, locationWrites)
```

###Creating Format using combinators
In the case where your `Reads` and `Writes` are symmetrical (which may not be the case in real applications), you can define a `Format` directly from combinators:

```scala
implicit val locationFormat: Format[Location] = (
  (JsPath \ "lat").format[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").format[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply, unlift(Location.unapply))
```