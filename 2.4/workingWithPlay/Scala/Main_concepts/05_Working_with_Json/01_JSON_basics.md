#JSON基础
现代web应用程序经常需要解析和生成JSON (JavaScript Object Notation) 格式的数据。Play通过它的[JSON 库](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/package.html)提供支持。

JSON是一个轻量级的数据交换格式，如下面这个:

```json
{
  "name" : "Watership Down",
  "location" : {
    "lat" : 51.235685,
    "long" : -1.309197
  },
  "residents" : [ 
    {
      "name" : "Fiver",
      "age" : 4,
      "role" : null
    }, 
    {
      "name" : "Bigwig",
      "age" : 6,
      "role" : "Owsla"
    } 
  ]
}
```

> 要学习更多JSON知识, 查阅[json.org](http://json.org/) .


##Play的 JSON库
[`play.api.libs.json`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/package.html) 包中包含表示JSON数据的数据结构，以及这些数据结构和其它数据表示之前转换的实用工具。有用的类型有:

####[`JsValue`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsValue.html)
这是一个表示任何JSON值的特质。JSON库有一个样例类扩展`JsValue`，以表示每个有效的JSON类型:

* [`JsString`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsString.html)
* [`JsNumber`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsNumber.html)
* [`JsBoolean`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsBoolean.html)
* [`JsObject`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsObject.html)
* [`JsArray`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsArray.html)
* [`JsNull`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsNull$.html)

使用各种`JsValue` 类型, 你可以构造任何JSON结构。

####[`Json`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Json$.html)
`Json` 对象提供实用工具, 主要用于转换到`JsValue` 结构或反向转换。

####[`JsPath`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsPath.html)
表示`JsValue` 结构内部结构的路径, 类似于XML的XPath。可以用来析取`JsValue` 结构，并为隐式转换进行模式匹配。


##转换到`JsValue`

###使用字符串解析

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
```

###使用类构造器

```scala
import play.api.libs.json._
val json: JsValue = JsObject(Seq(
  "name" -> JsString("Watership Down"),
  "location" -> JsObject(Seq("lat" -> JsNumber(51.235685), "long" -> JsNumber(-1.309197))),
  "residents" -> JsArray(Seq(
    JsObject(Seq(
      "name" -> JsString("Fiver"),
      "age" -> JsNumber(4),
      "role" -> JsNull
    )),
    JsObject(Seq(
      "name" -> JsString("Bigwig"),
      "age" -> JsNumber(6),
      "role" -> JsString("Owsla")
    ))
  ))
))
```

用`Json.obj` 和`Json.arr` 构造可能更简单。注意多数值不需要通过JsValue类显式封装, 工厂方法会使用隐式转换(下面会讲到)。

```scala
import play.api.libs.json.{JsNull,Json,JsString,JsValue}

val json: JsValue = Json.obj(
  "name" -> "Watership Down",
  "location" -> Json.obj("lat" -> 51.235685, "long" -> -1.309197),
  "residents" -> Json.arr(
    Json.obj(
      "name" -> "Fiver",
      "age" -> 4,
      "role" -> JsNull
    ),
    Json.obj(
      "name" -> "Bigwig",
      "age" -> 6,
      "role" -> "Owsla"
    )
  )
)
```

###使用Writes转换器
执行Scala到`JsValue`的转换是通过`Json.toJson[T](T)(implicit writes: Writes[T])` 工具方法。这个功能依赖一个[`Writes[T]`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Writes.html)类型转换器，它能将`T` 类型转换到`JsValue`。 

Play的JSON API为多数基本类型提供隐式`Writes`, 如`Int`, `Double`, `String`, 和 `Boolean`。它也支持包含任何上述类型的集合的`Writes` 转换器。 

```scala
import play.api.libs.json._

// 基本类型
val jsonString = Json.toJson("Fiver")
val jsonNumber = Json.toJson(4)
val jsonBoolean = Json.toJson(false)

// 基本类型集合
val jsonArrayOfInts = Json.toJson(Seq(1, 2, 3, 4))
val jsonArrayOfStrings = Json.toJson(List("Fiver", "Bigwig"))
```

要转换自己的模型为`JsValues`, 你必须定义隐式`Writes` 转换器并将其引入作用域。

```scala
case class Location(lat: Double, long: Double)
case class Resident(name: String, age: Int, role: Option[String])
case class Place(name: String, location: Location, residents: Seq[Resident])

import play.api.libs.json._

implicit val locationWrites = new Writes[Location] {
  def writes(location: Location) = Json.obj(
    "lat" -> location.lat,
    "long" -> location.long
  )
}

implicit val residentWrites = new Writes[Resident] {
  def writes(resident: Resident) = Json.obj(
    "name" -> resident.name,
    "age" -> resident.age,
    "role" -> resident.role
  )
}

implicit val placeWrites = new Writes[Place] {
  def writes(place: Place) = Json.obj(
    "name" -> place.name,
    "location" -> place.location,
    "residents" -> place.residents)
}

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

或者, 佻可以使用配合模式来定义`Writes` :

> 注意: 关于配合模式会在[JSON Reads/Writes/Formats Combinators](https://www.playframework.com/documentation/2.4.x/ScalaJsonCombinators)一节详细说明。

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
```


##析取JsValue结构
你可以析取一个`JsValue` 结构并读取特定的值。语法和功能类似于Scala的XML处理。

> 注意: 下面示例应用在前面创建的JsValue结构上。

###简单路径 `\`
应用`\` 操作符到一个`JsValue` 会返回和相关字段对应的属性, 假设这是个`JsObject`。
 
```scala
val lat = (json \ "location" \ "lat").get
// returns JsNumber(51.235685)
```

###递归路径 `\\`
应用`\\` 操作符会递归查找当前对象和所有对象中的字段。

```scala
val names = json \\ "name"
// returns Seq(JsString("Watership Down"), JsString("Fiver"), JsString("Bigwig"))
```

###索引查找(对于JsArrays)
你可以使用一个应用操作符和索引数获取`JsArray` 中的值。

```scala
val bigwig = (json \ "residents")(1)
// returns {"name":"Bigwig","age":6,"role":"Owsla"}
```


##从JsValue中转换

###使用字符串实用工具
微型:

```scala
val minifiedString: String = Json.stringify(json)
{"name":"Watership Down","location":{"lat":51.235685,"long":-1.309197},"residents":[{"name":"Fiver","age":4,"role":null},{"name":"Bigwig","age":6,"role":"Owsla"}]}
```

可读的:

```scala
val readableString: String = Json.prettyPrint(json)
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
```

###使用 JsValue.as/asOpt
转换`JsValue`到另一种类型的简单方式是使用`JsValue.as[T](implicit fjs: Reads[T]): T`。这个需要一个[`Reads[T]`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html)类型的隐式转换，来转换 `JsValue` 到`T` (和`Writes[T]` 相反)。跟`Writes`一样, JSON API为基本类型提供了`Reads` 转换器。

```scala
val name = (json \ "name").as[String]
// "Watership Down"

val names = (json \\ "name").map(_.as[String])
// Seq("Watership Down", "Fiver", "Bigwig")
```

如果路径不存在或者转换失败，`as` 方法会抛出一个`JsResultException` 。一个安全的方法是使用`JsValue.asOpt[T](implicit fjs: Reads[T]): Option[T]`。

```scala
val nameOption = (json \ "name").asOpt[String]
// Some("Watership Down")

val bogusOption = (json \ "bogus").asOpt[String]
// None
```

虽然`asOpt` 方法更安全, 但是错误信息会丢失。

###使用验证
推荐使用`JsValue` 的`validate` 方法将其转换到另一个类型(这个方法带有一个`Reads` 类型的参数)。这个方法即执行验证也执行转换操作, 并返回一个[`JsResult`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsResult.html)类型的结果。`JsResult` 通过两个类实现:

* [`JsSuccess`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsSuccess.html): 表示成功验证/转换，并封装结果。
* [`JsError`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsError.html): 表示验证/转换不成功，并包含一个验证错误列表。

你可以采用多种模式来处理验证结果:

```scala
val json = { ... }

val nameResult: JsResult[String] = (json \ "name").validate[String]

// 模式匹配
nameResult match {
  case s: JsSuccess[String] => println("Name: " + s.get)
  case e: JsError => println("Errors: " + JsError.toFlatJson(e).toString())
}

// Fallback value
val nameOrFallback = nameResult.getOrElse("Undefined")

// map
val nameUpperResult: JsResult[String] = nameResult.map(_.toUpperCase())

// fold
val nameOption: Option[String] = nameResult.fold(
  invalid = {
    fieldErrors => fieldErrors.foreach(x => {
      println("field: " + x._1 + ", errors: " + x._2)
    })
    None
  },
  valid = {
    name => Some(name)
  }
)
```

###JsValue转换到模型
要从JsValue转换到模型, 你必段定义一个隐式`Reads[T]` ，其中`T` 是你的模型的类型。

> 注意: 用于实现`Reads` 的模式和自定义验证的细节请查阅 [JSON Reads/Writes/Formats Combinators](https://www.playframework.com/documentation/2.4.x/ScalaJsonCombinators).

```scala
case class Location(lat: Double, long: Double)
case class Resident(name: String, age: Int, role: Option[String])
case class Place(name: String, location: Location, residents: Seq[Resident])
import play.api.libs.json._
import play.api.libs.functional.syntax._

implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)

implicit val residentReads: Reads[Resident] = (
  (JsPath \ "name").read[String] and
  (JsPath \ "age").read[Int] and
  (JsPath \ "role").readNullable[String]
)(Resident.apply _)

implicit val placeReads: Reads[Place] = (
  (JsPath \ "name").read[String] and
  (JsPath \ "location").read[Location] and
  (JsPath \ "residents").read[Seq[Resident]]
)(Place.apply _)


val json = { ... }

val placeResult: JsResult[Place] = json.validate[Place]
// JsSuccess(Place(...),)

val residentResult: JsResult[Resident] = (json \ "residents")(1).validate[Resident]
// JsSuccess(Resident(Bigwig,6,Some(Owsla)),)
```