#JSON 与 HTTP
通过HTTP API与JSON库的结合，Play 支持带Content-Type为JSON的HTTP请求和响应。

> 关于Controllers,、Actions和routing的详细介绍请查阅 [HTTP 编程](https://www.playframework.com/documentation/2.4.x/ScalaActions)。

我们通过设计一个简单的RESTful web服务来说明一些必要的概念，通过GET获取实体列表，接收POSTs来创建新实体。对于所有数据，服务都会使用JSON内容类型。

这里是我们的服务要使用的模型:

```scala
case class Location(lat: Double, long: Double)

case class Place(name: String, location: Location)

object Place {
    
  var list: List[Place] = {
    List(
      Place(
        "Sandleford",
        Location(51.377797, -1.318965)
      ),
      Place(
        "Watership Down",
        Location(51.235685, -1.309197)
      )
    )
  }
    
  def save(place: Place) = {
    list = list ::: List(place)
  }
}
```


##以JSON格式提供实体列表
我们首先导入必要的东西到controller。

```scala
import play.api.mvc._
import play.api.libs.json._
import play.api.libs.functional.syntax._

object Application extends Controller {
  
}
```

在写`Action`之前, 我们需要建一个转换模型到`JsValue` 表示的管道。通过定义一个隐式`Writes[Place]`即可。

```scala
implicit val locationWrites: Writes[Location] = (
  (JsPath \ "lat").write[Double] and
  (JsPath \ "long").write[Double]
)(unlift(Location.unapply))

implicit val placeWrites: Writes[Place] = (
  (JsPath \ "name").write[String] and
  (JsPath \ "location").write[Location]
)(unlift(Place.unapply))
```

下一步我们写`Action`:

```scala
def listPlaces = Action {
  val json = Json.toJson(Place.list)
  Ok(json)
}
```

`Action` 接收一个`Place` 对象列表, 使用`Json.toJson` 和隐式`Writes[Place]` 转换他们到一个`JsValue`，然后作为result的body返回。Play会识别result为JSON，并为响应设置适当的`Content-Type` 标头和主体。 

最后一步是在`conf/routes`文件中，为我们的`Action` 添加路由:

```
GET   /places               controllers.Application.listPlaces
```

我们可以通过浏览器或HTTP工具来发送请求进行测试。下面的例子通过使用unix命令行工具[cURL](http://curl.haxx.se/)。

```shell
curl --include http://localhost:9000/places
```

响应:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 141

[{"name":"Sandleford","location":{"lat":51.377797,"long":-1.318965}},{"name":"Watership Down","location":{"lat":51.235685,"long":-1.309197}}]
```


##从JSON格式创建新实体实例
为这个`Action` 我们需要定义一个隐式`Reads[Place]` 来转换`JsValue` 到模型。

```scala
implicit val locationReads: Reads[Location] = (
  (JsPath \ "lat").read[Double] and
  (JsPath \ "long").read[Double]
)(Location.apply _)

implicit val placeReads: Reads[Place] = (
  (JsPath \ "name").read[String] and
  (JsPath \ "location").read[Location]
)(Place.apply _)
```

下一步我们定义`Action`。

```scala
def savePlace = Action(BodyParsers.parse.json) { request =>
  val placeResult = request.body.validate[Place]
  placeResult.fold(
    errors => {
      BadRequest(Json.obj("status" ->"KO", "message" -> JsError.toFlatJson(errors)))
    },
    place => { 
      Place.save(place)
      Ok(Json.obj("status" ->"OK", "message" -> ("Place '"+place.name+"' saved.") ))  
    }
  )
}
```

这个`Action` 比前面那个列表示例更复杂。需要注意以下几点:

* 该`Action` 期望接收的请求的`Content-Type` 标头需要是`text/json` 或`application/json` ，并且body包含一个创建的实体的 JSON表示。
* 它使用一个JSON 特定的`BodyParser` 解析请求和提供`request.body` 作为`JsValue`。
* 我们使用`validate` 方法来转换，它依赖于隐式`Reads[Place]`。
* 要处理验证结果, 我们使用带错误和成功处理的`fold` 。这种模式也类似于使用[表单提交](../04_Form_submission_and_validation/01_Handling_form_submission.md).
* `Action` 发送的响应也是JSON格式。

最后我们在`conf/routes`文件中添加路由:

```
POST  /places               controllers.Application.savePlace
```

我们使用有效和无效的请求来测试这个action，验证成功及错误的工作流。 

用有效数据测试action:

```shell
curl --include
  --request POST
  --header "Content-type: application/json" 
  --data '{"name":"Nuthanger Farm","location":{"lat" : 51.244031,"long" : -1.263224}}' 
  http://localhost:9000/places
```

响应:

```shell
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 57

{"status":"OK","message":"Place 'Nuthanger Farm' saved."}
```

用无效数据测试action, 丢失 “name” 字段:

```shell
curl --include
  --request POST
  --header "Content-type: application/json"
  --data '{"location":{"lat" : 51.244031,"long" : -1.263224}}' 
  http://localhost:9000/places
```

响应:

```shell
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Content-Length: 79

{"status":"KO","message":{"obj.name":[{"msg":"error.path.missing","args":[]}]}}
```

用无效数据测试action, “lat”数据类型错误:

```shell
curl --include
  --request POST
  --header "Content-type: application/json" 
  --data '{"name":"Nuthanger Farm","location":{"lat" : "xxx","long" : -1.263224}}' 
  http://localhost:9000/places
```

响应:

```shell
HTTP/1.1 400 Bad Request
Content-Type: application/json; charset=utf-8
Content-Length: 92

{"status":"KO","message":{"obj.location.lat":[{"msg":"error.expected.jsnumber","args":[]}]}}
```


##小结
Play 天生设计为支持REST 和JSON，开发这些服务是相当简单直观的。大部分的工作就是为你的模型写`Reads` 和`Writes` , 下一节我们详细介绍。