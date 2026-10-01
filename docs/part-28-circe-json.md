# Part 28: Circe JSON

## สารบัญ
1. [Circe Overview](#circe-overview)
2. [Encoding/Decoding](#encoding-decoding)
3. [Custom Codecs](#custom-codecs)
4. [JSON Manipulation](#json-manipulation)
5. [Optics](#optics)

---

## Circe Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "io.circe" %% "circe-core"    % "0.14.6",
  "io.circe" %% "circe-generic" % "0.14.6",
  "io.circe" %% "circe-parser" % "0.14.6",
  "io.circe" %% "circe-optics" % "0.14.1"
)
```

### Basic Usage

```scala
import io.circe.*
import io.circe.syntax.*
import io.circe.parser.*
import io.circe.generic.auto.*

// Case class to JSON
case class User(id: Long, name: String, email: String, active: Boolean)

val user = User(1L, "Alice", "alice@example.com", true)

// Encode
val json: Json = user.asJson
println(json.spaces2)
// {
//   "id" : 1,
//   "name" : "Alice",
//   "email" : "alice@example.com",
//   "active" : true
// }

// Decode
val jsonStr = """{"id":1,"name":"Alice","email":"alice@example.com","active":true}"""
val decoded: Either[Error, User] = decode[User](jsonStr)
println(decoded)  // Right(User(1,Alice,alice@example.com,true))

// Parse JSON string to Json
val parsed: Either[ParsingFailure, Json] = parse(jsonStr)
```

---

## Encoding/Decoding

### Automatic Derivation

```scala
import io.circe.*
import io.circe.generic.auto.*
import io.circe.syntax.*
import io.circe.parser.*

// Nested case classes
case class Address(
  street: String,
  city: String,
  country: String,
  postalCode: String
)

case class Person(
  id: Long,
  name: String,
  age: Int,
  address: Address,
  hobbies: List[String],
  metadata: Map[String, String]
)

val person = Person(
  1L,
  "Alice",
  30,
  Address("123 Main St", "Bangkok", "Thailand", "10110"),
  List("coding", "reading"),
  Map("department" -> "Engineering", "level" -> "Senior")
)

println(person.asJson.spaces2)

// Sealed trait hierarchy
sealed trait Shape derives Encoder.AsObject, Decoder
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

val shapes: List[Shape] = List(Circle(5), Rectangle(3, 4), Triangle(6, 8))
println(shapes.asJson.noSpaces)
```

### Semi-automatic Derivation

```scala
import io.circe.*
import io.circe.generic.semiauto.*

case class Product(id: Long, name: String, price: BigDecimal, tags: Set[String])

// Explicit derivation
given Encoder[Product] = deriveEncoder[Product]
given Decoder[Product] = deriveDecoder[Product]

val product = Product(1L, "Laptop", BigDecimal("1299.99"), Set("electronics", "computers"))
println(product.asJson.noSpaces)
```

---

## Custom Codecs

### Custom Encoder

```scala
import io.circe.*
import io.circe.syntax.*

// Custom Encoder for special types
case class EmailAddress(value: String)

given Encoder[EmailAddress] = Encoder[String].contramap(_.value)
given Decoder[EmailAddress] = Decoder[String].map(EmailAddress.apply)

// Custom Encoder for sealed trait (no type discriminator)
sealed trait Status
case object Active   extends Status
case object Inactive extends Status
case object Pending  extends Status

given Encoder[Status] = Encoder[String].contramap {
  case Active   => "active"
  case Inactive => "inactive"
  case Pending  => "pending"
}

given Decoder[Status] = Decoder[String].emap {
  case "active"   => Right(Active)
  case "inactive" => Right(Inactive)
  case "pending"  => Right(Pending)
  case s          => Left(s"Unknown status: $s")
}

// Custom Encoder with different JSON structure
case class Money(amount: BigDecimal, currency: String)

given Encoder[Money] = Encoder.instance { m =>
  Json.obj(
    "amount"   -> m.amount.asJson,
    "currency" -> m.currency.asJson,
    "formatted" -> s"${m.currency} ${m.amount}".asJson
  )
}

given Decoder[Money] = Decoder.instance { cursor =>
  for
    amount   <- cursor.downField("amount").as[BigDecimal]
    currency <- cursor.downField("currency").as[String]
  yield Money(amount, currency)
}

val money = Money(BigDecimal("1299.99"), "USD")
println(money.asJson.spaces2)
// {
//   "amount" : 1299.99,
//   "currency" : "USD",
//   "formatted" : "USD 1299.99"
// }
```

### Configuration-based Derivation

```scala
import io.circe.*
import io.circe.generic.extras.*
import io.circe.generic.extras.semiauto.*

@ConfiguredJsonCodec
case class ApiResponse(
  statusCode: Int,
  errorMessage: Option[String],
  responseData: Option[Json]
)

object ApiResponse:
  given Configuration = Configuration.default
    .withSnakeCaseMemberNames  // statusCode -> status_code
    .withDefaults

// snake_case JSON
println(ApiResponse(200, None, Some(Json.obj("key" -> "value".asJson))).asJson.spaces2)
// {
//   "status_code" : 200,
//   "response_data" : {"key":"value"}
// }
```

---

## JSON Manipulation

### Cursor API

```scala
import io.circe.*
import io.circe.parser.*

val json = parse("""
{
  "user": {
    "id": 1,
    "name": "Alice",
    "address": {
      "city": "Bangkok",
      "country": "Thailand"
    },
    "scores": [95, 87, 92, 88]
  }
}
""").getOrElse(Json.Null)

// Navigate with cursor
val cursor = json.hcursor

// Get nested value
val city = cursor.downField("user").downField("address").downField("city").as[String]
println(city)  // Right(Bangkok)

// Get array element
val firstScore = cursor.downField("user").downField("scores").downN(0).as[Int]
println(firstScore)  // Right(95)

// Focus on subtree
val userCursor = cursor.downField("user")
val name = userCursor.get[String]("name")
println(name)  // Right(Alice)

// Modify value
val updatedJson = cursor
  .downField("user")
  .downField("name")
  .set(Json.fromString("Bob"))
  .top.getOrElse(Json.Null)

println(updatedJson.spaces2)
```

### Json Building

```scala
import io.circe.*
import io.circe.syntax.*

// Manual JSON construction
val jsonObj = Json.obj(
  "name"    -> "Alice".asJson,
  "age"     -> 30.asJson,
  "scores"  -> List(95, 87, 92).asJson,
  "address" -> Json.obj(
    "city"    -> "Bangkok".asJson,
    "country" -> "Thailand".asJson
  )
)

// JSON from template
def apiResponse[A: Encoder](data: A, status: Int = 200): Json =
  Json.obj(
    "status"    -> status.asJson,
    "data"      -> data.asJson,
    "timestamp" -> java.time.Instant.now().toString.asJson
  )

// Merge JSON objects
val base = Json.obj("a" -> 1.asJson, "b" -> 2.asJson)
val extra = Json.obj("c" -> 3.asJson, "a" -> 99.asJson)  // 'a' will be overwritten
val merged = base.deepMerge(extra)
println(merged)  // {"a":99,"b":2,"c":3}
```

---

## Optics

```scala
import io.circe.*
import io.circe.optics.JsonPath.*
import io.circe.parser.*

val json = parse("""
{
  "users": [
    {"id": 1, "name": "Alice", "active": true},
    {"id": 2, "name": "Bob", "active": false},
    {"id": 3, "name": "Charlie", "active": true}
  ]
}
""").getOrElse(Json.Null)

// Navigate with optics
val usersPath = root.users.each.name.string
val names = usersPath.getAll(json)
println(names)  // List(Alice, Bob, Charlie)

// Modify with optics
val deactivateAll = root.users.each.active.bool.set(false)
val updated = deactivateAll(json)

// Conditional modification
val activateFirst = root.users.index(0).active.bool.set(true)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Circe setup และ automatic derivation
- ✅ Encoding/Decoding case classes
- ✅ Custom Encoders/Decoders
- ✅ Configuration-based derivation (snake_case)
- ✅ JSON manipulation กับ Cursor API
- ✅ JSON building
- ✅ Optics สำหรับ nested JSON operations

---

*[← Part 27: Doobie](part-27-doobie.md) | [Part 29: Apache Spark →](part-29-spark.md)*
