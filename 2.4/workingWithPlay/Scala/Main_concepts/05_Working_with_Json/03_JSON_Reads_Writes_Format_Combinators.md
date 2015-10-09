#JSON Reads/Writes/Format 组合子（Combinators）
[JSON 基础](01_JSON_basics.md) 中介绍了[`Reads`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html) 和[`Writes`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Writes.html) 转换器，可以在[`JsValue`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsValue.html) 结构和其它数据类型之间转换。本节更详细地介绍如何构建这些转换器，以及在转换过程中如何进行验证。

本节示例会用到这个`JsValue` 结构和相应的模型:

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
[`JsPath`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/JsPath.html) 是构建`Reads`/`Writes` 的核心。`JsPath` 表明了数据在`JsValue` 结构中的位置。你可以使用`JsPath` 对象(在根路径) 来定义一个`JsPath` 子实例，语法类似于遍历`JsValue`:

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

[`play.api.libs.json`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/package.html) 包中为`JsPath` 定义了一个别名: `__` (两个下划线)。如果你喜欢也可以使用它:

```scala
val longPath = __ \ "location" \ "long"
```


##Reads
[`Reads`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Reads.html) 转换器用于将`JsValue` 转换到其它类型。你可以组合与嵌套`Reads` 来构造更复杂的`Reads`。

你需要导入这些内容以创建`Reads`:

```scala
import play.api.libs.json._ // JSON 库
import play.api.libs.json.Reads._ // 自定义 验证助手
import play.api.libs.functional.syntax._ // Combinator 语法
```

###Path Reads
`JsPath` 包含方法来创建特定的`Reads`，它应用另一个`Reads` 到特定路径的`JsValue` :

* `JsPath.read[T](implicit r: Reads[T]): Reads[T]` - 创建一个`Reads[T]`，它将应用隐式参数`r` 到该路径的`JsValue` 。
* `JsPath.readNullable[T](implicit r: Reads[T]): Reads[Option[T]]readNullable` - 该路径可能缺失，或包含空值时使用。

> 注意: JSON库为基本类型提供了隐式`Reads` ，如`String`, `Int`, `Double`, 等。

定义一个具体路径的`Reads` 如下:

```scala
val nameReads: Reads[String] = (JsPath \ "name").read[String]
```

###复合 Reads
你可以组合单个路径`Reads` 成复合`Reads` ，这样可以用来转换复杂模型。

为容易理解, 我们先分解成两条语句。首先使用`and` 组合子来组合`Reads` 对象:

```scala
val locationReadsBuilder =
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
```

上面产生的结果类型为`FunctionalBuilder[Reads]#CanBuild2[Double, Double]`。这是一个中间对象，你不需要担心太多，只需要知道它会被用来创建一个复合`Reads`。 

第二步是调用`CanBuildX`的`apply` 方法，它有一个功能是转换单个值到你的模型, 这会返回你的复合`Reads`。如果你有一个带有构造器签名的样例类, 你可以只使用它的`apply` 方法:

```scala
implicit val locationReads = locationReadsBuilder.apply(Location.apply _)
```

上述代码合成一条语句:

```scala
implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)
```

###验证 Reads
`JsValue.validate` 方法在 [JSON 基础](01_JSON_basics.md)中介绍过， 推荐用它进行验证和转换`JsValue` 到其它类型。这里是基本模式:

```scala
val json = { ... }

val nameReads: Reads[String] = (JsPath \ "name").read[String]

val nameResult: JsResult[String] = json.validate[String](nameReads)

nameResult match {
  case s: JsSuccess[String] => println("Name: " + s.get)
  case e: JsError => println("Errors: " + JsError.toFlatJson(e).toString())
}
```

`Reads` 的默认验证是最简单的, 如检查类型转换错误。你可以通过使用`Reads` 验证助手定义自定义验证规则。这里是一些常用的: 

* `Reads.email` - 验证字符串是否电子邮箱格式。
* `Reads.minLength(nb)` - 验证一个字符串的最小长度。
* `Reads.min` - 验证最小数值。
* `Reads.max` - 验证最大数值。
* `Reads[A] keepAnd Reads[B] => Reads[A]` - 尝试`Reads[A]` 和`Reads[B]` ，但只保留`Reads[A]`结果的运算符 (如果你知道Scala 解析组命子 `keepAnd == <~` )。
* `Reads[A] andKeep Reads[B] => Reads[B]` - 尝试`Reads[A]` 和`Reads[B]` ，但只保留`Reads[B]`结果的运算符 (如果你知道Scala 解析组合子`andKeep == ~>` )。
* `Reads[A] or Reads[B] => Reads` - 执行逻辑或，并保留最后选中的`Reads` 的结果的运算符。

要添加验证, 应用助手作为`JsPath.read` 方法的参数:

```scala
val improvedNameReads =
  (JsPath \ "name").read[String](minLength[String](2))
```

###全部合并到一起
通过使用复合`Reads` 和自定义验证，我们可以为示例模型定义一组有效的`Reads` 并应用他们:

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

注意复合`Reads` 可以嵌套。在本例, `placeReads` 使用前面定义的隐式`locationReads` 和`residentReads` 在结果的特定路径。


##Writes
[`Writes`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Writes.html) 用于转换一些类型到`JsValue`。

你可以使用和`Reads`非常类似的`JsPath` 和组合子构建复合`Writes`。这里是我们示例模型的`Writes`:

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

在复合`Writes` 和`Reads`之间有一点点不同:

* 单个路径`Writes` 是使用`JsPath.write` 方法创建。
* 转换到`JsValue`无需验证，这让结构简单些，并且也不需要任何验证助手。
* 中间结果`FunctionalBuilder#CanBuildX` (由`and` 组合子创建) 接收一个函数为参数，该函数转换复合类型`T`到一个元组，该元组与单个路径`Writes`匹配。虽然看起来和`Reads` 对称, 样例类的`unapply` 方法返回的是属性元组的`Option`类型，必须使用`unlift` 方法将元组提取出来。


##递归类型
有一种特殊情况是上面的例子未讲到的，是如何处理递归类型的`Reads` 和`Writes`。`JsPath` 提供`lazyRead` 和`lazyWrite` 方法，带有call-by-name 参数来处理这种情况:

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
[`Format[T]`](https://www.playframework.com/documentation/2.4.x/api/scala/play/api/libs/json/Format.html) 只是一个`Reads` 和`Writes` 混合的特质，可以代替这二个进行隐式转换。

###从Reads和Writes创建Format
你可以通过`Reads` and `Writes`为同一类型构建它的`Format` :

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

###使用组合子创建 Format 
对于`Reads` 和`Writes` 对称的情况(真实应用程序中不一定是这样), 你可以直接从组合子定义一个`Format` :

```scala
implicit val locationFormat: Format[Location] = (
  (JsPath \ "lat").format[Double](min(-90.0) keepAnd max(90.0)) and
  (JsPath \ "long").format[Double](min(-180.0) keepAnd max(180.0))
)(Location.apply, unlift(Location.unapply))
```