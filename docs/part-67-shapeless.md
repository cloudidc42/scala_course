# Part 67: Shapeless และ Generic Programming

## สารบัญ

1. [แนะนำ Shapeless](#1-แนะนำ-shapeless)
2. [HList: Heterogeneous Lists](#2-hlist-heterogeneous-lists)
3. [Coproduct: Sum Types](#3-coproduct-sum-types)
4. [Generic สำหรับ Case Classes](#4-generic-สำหรับ-case-classes)
5. [LabelledGeneric กับ Field Names](#5-labelledgeneric-กับ-field-names)
6. [Poly Functions](#6-poly-functions)
7. [Type Class Derivation อัตโนมัติ](#7-type-class-derivation-อัตโนมัติ)
8. [Practical Use Cases](#8-practical-use-cases)
9. [Shapeless ใน Scala 3](#9-shapeless-ใน-scala-3)
10. [สรุป](#10-สรุป)

---

## 1. แนะนำ Shapeless

Shapeless เป็น library สำหรับ generic programming ใน Scala ที่ช่วยให้เขียนโค้ดที่ทำงานกับ types ต่างๆ ได้โดยอัตโนมัติ

### ทำไมต้องใช้ Shapeless?

```scala
// ปัญหา: ต้องเขียน boilerplate ซ้ำๆ สำหรับแต่ละ case class
case class User(id: Long, name: String, email: String)
case class Product(id: Long, name: String, price: Double)
case class Order(id: Long, userId: Long, total: Double)

// ต้องเขียน CSV encoder สำหรับแต่ละ type
def userToCsv(u: User): String      = s"${u.id},${u.name},${u.email}"
def productToCsv(p: Product): String = s"${p.id},${p.name},${p.price}"
def orderToCsv(o: Order): String     = s"${o.id},${o.userId},${o.total}"

// ด้วย Shapeless: เขียนครั้งเดียว ใช้กับทุก case class!
def toCsv[A](a: A)(using CsvEncoder[A]): String = summon[CsvEncoder[A]].encode(a)
```

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  // Shapeless สำหรับ Scala 2
  "com.chuusai" %% "shapeless" % "2.3.10",
  
  // Shapeless 3 สำหรับ Scala 3 (แนะนำ)
  "org.typelevel" %% "shapeless3-deriving" % "3.3.0",
  "org.typelevel" %% "shapeless3-typeable"  % "3.3.0"
)
```

---

## 2. HList: Heterogeneous Lists

HList คือ list ที่แต่ละ element มี type ต่างกัน และ type system รู้ type ของแต่ละตำแหน่ง

### Basic HList Operations

```scala
import shapeless.*
import shapeless.HList.*

// สร้าง HList
val hlist1: Int :: String :: Boolean :: HNil =
  42 :: "hello" :: true :: HNil

// แตกต่างจาก List[Any] ตรงที่ type-safe
val list1: List[Any] = List(42, "hello", true)
// list1.head เป็น Any (ไม่ type-safe)
// hlist1.head เป็น Int (type-safe!)

// Access elements
val firstInt: Int     = hlist1.head          // 42
val rest: String :: Boolean :: HNil = hlist1.tail
val secondStr: String = hlist1.tail.head     // "hello"
```

### HList Operations

```scala
import shapeless.*
import shapeless.ops.hlist.*

val h = 1 :: "two" :: 3.0 :: true :: HNil

// Length
val len = h.runtimeLength   // 4

// Map over HList (ต้องใช้ Poly function)
object toStringPoly extends Poly1:
  given Case.Aux[Int, String]     = at(_.toString)
  given Case.Aux[String, String]  = at(s => s"str: $s")
  given Case.Aux[Double, String]  = at(d => s"${d}f")
  given Case.Aux[Boolean, String] = at(_.toString)

val mapped = h.map(toStringPoly)
// "1" :: "str: two" :: "3.0f" :: "true" :: HNil

// FlatMap
val nested = (1 :: HNil) :: ("two" :: HNil) :: HNil
// val flat = nested.flatten

// Prepend / Append
val prepended = 0 :: h     // 0 :: 1 :: "two" :: 3.0 :: true :: HNil
val appended  = h :+ false  // 1 :: "two" :: 3.0 :: true :: false :: HNil

// Take and Drop
val taken    = h.take[Nat._2]     // 1 :: "two" :: HNil
val dropped  = h.drop[Nat._2]     // 3.0 :: true :: HNil

// Split
val (left, right) = h.split[Nat._2]
// left:  1 :: "two" :: HNil
// right: 3.0 :: true :: HNil

// Reverse
val reversed = h.reverse   // true :: 3.0 :: "two" :: 1 :: HNil

// Select by type
val strVal: String = h.select[String]  // "two"

// Filter by type
val strings = h.filter[String]  // "two" :: HNil
```

### Converting HList

```scala
// HList เป็น Tuple
val tuple: (Int, String, Double, Boolean) = h.tupled
// (1, "two", 3.0, true)

// Tuple เป็น HList
val fromTuple = (1, "hello", true).productElements
// 1 :: "hello" :: true :: HNil

// HList เป็น List[Any]
val anyList: List[Any] = h.toList
// List(1, "two", 3.0, true)

// Typed HList operations
def sumHList(h: Int :: Int :: Int :: HNil): Int =
  h.head + h.tail.head + h.tail.tail.head

val result = sumHList(1 :: 2 :: 3 :: HNil)  // 6
```

### Zip and Unzip

```scala
val names  = "Alice" :: "Bob" :: "Charlie" :: HNil
val scores = 95 :: 87 :: 92 :: HNil

// Zip สอง HLists
val zipped = names.zip(scores)
// ("Alice", 95) :: ("Bob", 87) :: ("Charlie", 92) :: HNil

// Unzip
val (unzippedNames, unzippedScores) = zipped.unzip
```

---

## 3. Coproduct: Sum Types

Coproduct คือ type สำหรับ "one of these types" คล้าย sealed trait

### Basic Coproduct

```scala
import shapeless.*

// Coproduct ที่รวม Int | String | Boolean
type IntOrStringOrBool = Int :+: String :+: Boolean :+: CNil

// สร้าง values
val intVal:    IntOrStringOrBool = Inl(42)           // Int case
val strVal:    IntOrStringOrBool = Inr(Inl("hello")) // String case
val boolVal:   IntOrStringOrBool = Inr(Inr(Inl(true))) // Boolean case

// Pattern matching
def processValue(c: IntOrStringOrBool): String =
  c match
    case Inl(i)         => s"Int: $i"
    case Inr(Inl(s))    => s"String: $s"
    case Inr(Inr(Inl(b))) => s"Boolean: $b"
    case Inr(Inr(Inr(_))) => "impossible - CNil"
```

### Coproduct Operations

```scala
import shapeless.*
import shapeless.ops.coproduct.*

type Shape = Circle :+: Rectangle :+: Triangle :+: CNil

case class Circle(radius: Double)
case class Rectangle(width: Double, height: Double)
case class Triangle(base: Double, height: Double)

// Inject value
val circle: Shape = Coproduct[Shape](Circle(5.0))

// Select
circle.select[Circle]     // Some(Circle(5.0))
circle.select[Rectangle]  // None

// Map over Coproduct
object areaCalc extends Poly1:
  given Case.Aux[Circle, Double]    = at(c => Math.PI * c.radius * c.radius)
  given Case.Aux[Rectangle, Double] = at(r => r.width * r.height)
  given Case.Aux[Triangle, Double]  = at(t => 0.5 * t.base * t.height)

val area = circle.map(areaCalc)  // Double Coproduct -> Double
```

---

## 4. Generic สำหรับ Case Classes

`Generic[A]` แปลง case class เป็น HList และกลับ

### Basic Generic

```scala
import shapeless.*

case class User(id: Long, name: String, email: String)

// ได้ Generic instance สำหรับ User
val gen = Generic[User]

// Case class -> HList
val user = User(1L, "Alice", "alice@example.com")
val hlist: Long :: String :: String :: HNil = gen.to(user)
// 1L :: "Alice" :: "alice@example.com" :: HNil

// HList -> Case class
val restored: User = gen.from(hlist)
// User(1, "Alice", "alice@example.com")
```

### Generic สำหรับ Sealed Traits

```scala
sealed trait Shape
case class Circle(radius: Double)    extends Shape
case class Rectangle(w: Double, h: Double) extends Shape
case class Triangle(b: Double, h: Double)  extends Shape

// Generic[Shape] จะสร้าง Coproduct
val gen = Generic[Shape]

type ShapeRepr = Circle :+: Rectangle :+: Triangle :+: CNil

val circle: Shape = Circle(5.0)
val repr: ShapeRepr = gen.to(circle)
// Inl(Circle(5.0))
```

### Using Generic for Type Class Derivation

```scala
// สร้าง CSV encoder ที่ทำงานกับทุก case class
trait CsvEncoder[A]:
  def encode(value: A): String

// Encoder สำหรับ primitive types
given CsvEncoder[Int]    = v => v.toString
given CsvEncoder[Long]   = v => v.toString
given CsvEncoder[Double] = v => v.toString
given CsvEncoder[String] = v => s""""$v""""
given CsvEncoder[Boolean] = v => v.toString

// Encoder สำหรับ HList
given CsvEncoder[HNil] = _ => ""

given [H, T <: HList](using
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = { case h :: t =>
  val tStr = tEncoder.encode(t)
  if tStr.isEmpty then hEncoder.encode(h)
  else s"${hEncoder.encode(h)},$tStr"
}

// Encoder ที่ derive จาก Generic
given [A, Repr <: HList](using
  gen: Generic.Aux[A, Repr],
  enc: CsvEncoder[Repr]
): CsvEncoder[A] = a => enc.encode(gen.to(a))

// ใช้งาน
case class User(id: Long, name: String, email: String)
case class Product(id: Long, name: String, price: Double)

val user    = User(1L, "Alice", "alice@example.com")
val product = Product(42L, "Widget", 9.99)

println(summon[CsvEncoder[User]].encode(user))
// 1,"Alice","alice@example.com"

println(summon[CsvEncoder[Product]].encode(product))
// 42,"Widget",9.99
```

---

## 5. LabelledGeneric กับ Field Names

`LabelledGeneric` เก็บ field names ไว้เป็น type-level information

### Basic LabelledGeneric

```scala
import shapeless.*
import shapeless.record.*

case class User(id: Long, name: String, email: String)

val lgen = LabelledGeneric[User]

val user = User(1L, "Alice", "alice@example.com")
val record = lgen.to(user)

// record มี type:
// Long with KeyTag[Symbol with Tagged["id"], Long] ::
// String with KeyTag[Symbol with Tagged["name"], String] ::
// String with KeyTag[Symbol with Tagged["email"], String] ::
// HNil

// Access by field name
val id:    Long   = record("id")
val name:  String = record("name")
val email: String = record("email")

println(s"Name: $name, Email: $email")
```

### Record Operations

```scala
import shapeless.*
import shapeless.record.*
import shapeless.syntax.singleton.*

// สร้าง Record โดยตรง
val person = ("name"  ->> "Alice") ::
             ("age"   ->> 30) ::
             ("email" ->> "alice@example.com") ::
             HNil

// Access fields
val personName: String = person("name")
val personAge:  Int    = person("age")

// Update field
val updatedPerson = person.updated("age", 31)

// Remove field
val withoutEmail = person.remove("email")

// Rename field
val renamed = person.rename("email", "contact")

// Keys
val keys = person.keys
// "name" :: "age" :: "email" :: HNil

// Values
val values = person.values
// "Alice" :: 30 :: "alice@example.com" :: HNil
```

### JSON Encoder ด้วย LabelledGeneric

```scala
import shapeless.*
import shapeless.record.*
import shapeless.labelled.*

// JSON value type
sealed trait JValue
case class JString(value: String)  extends JValue
case class JNumber(value: Double)  extends JValue
case class JBoolean(value: Boolean) extends JValue
case class JObject(fields: Map[String, JValue]) extends JValue
case object JNull extends JValue

// Type class
trait JsonEncoder[A]:
  def encode(value: A): JValue

// Primitive encoders
given JsonEncoder[String]  = v => JString(v)
given JsonEncoder[Int]     = v => JNumber(v.toDouble)
given JsonEncoder[Long]    = v => JNumber(v.toDouble)
given JsonEncoder[Double]  = v => JNumber(v)
given JsonEncoder[Boolean] = v => JBoolean(v)

// HNil encoder
given JsonEncoder[HNil] with
  def encode(h: HNil): JValue = JObject(Map.empty)

// Labelled HList encoder
given [K <: Symbol, V, T <: HList](using
  witness: Witness.Aux[K],
  vEncoder: Lazy[JsonEncoder[V]],
  tEncoder: JsonEncoder[T]
): JsonEncoder[FieldType[K, V] :: T] with
  def encode(hlist: FieldType[K, V] :: T): JValue =
    val fieldName  = witness.value.name
    val fieldValue = vEncoder.value.encode(hlist.head)
    val rest       = tEncoder.encode(hlist.tail)
    rest match
      case JObject(fields) => JObject(fields + (fieldName -> fieldValue))
      case _               => JObject(Map(fieldName -> fieldValue))

// Derive encoder for any case class
given [A, Repr <: HList](using
  lgen: LabelledGeneric.Aux[A, Repr],
  enc: Lazy[JsonEncoder[Repr]]
): JsonEncoder[A] with
  def encode(value: A): JValue =
    enc.value.encode(lgen.to(value))

// ใช้งาน
case class User(id: Long, name: String, email: String, active: Boolean)

val user = User(1L, "Alice", "alice@example.com", true)
val json = summon[JsonEncoder[User]].encode(user)
// JObject(Map("id" -> JNumber(1.0), "name" -> JString("Alice"), ...))
```

---

## 6. Poly Functions

Poly functions คือ polymorphic functions ที่ทำงานกับหลาย types

### Basic Poly1

```scala
import shapeless.*

// Poly function ที่แปลงทุก type เป็น String
object stringify extends Poly1:
  given Case.Aux[Int, String]     = at(i => s"int:$i")
  given Case.Aux[String, String]  = at(s => s"str:$s")
  given Case.Aux[Double, String]  = at(d => s"dbl:$d")
  given Case.Aux[Boolean, String] = at(b => s"bool:$b")

val h = 42 :: "hello" :: 3.14 :: true :: HNil
val result = h.map(stringify)
// "int:42" :: "str:hello" :: "dbl:3.14" :: "bool:true" :: HNil
```

### Poly2 (Binary Poly Functions)

```scala
import shapeless.*

// Poly2 สำหรับ fold operation
object addToList extends Poly2:
  given Case.Aux[List[String], Int, List[String]] =
    at((acc, i) => acc :+ i.toString)
  given Case.Aux[List[String], String, List[String]] =
    at((acc, s) => acc :+ s)
  given Case.Aux[List[String], Boolean, List[String]] =
    at((acc, b) => acc :+ b.toString)

val h = 1 :: "two" :: true :: HNil
val list = h.foldLeft(List.empty[String])(addToList)
// List("1", "two", "true")
```

### Poly Functions กับ Constraints

```scala
import shapeless.*

// Poly function ที่ทำงานกับทุก type ที่มี Numeric
object doubleIt extends Poly1:
  given [A: Numeric]: Case.Aux[A, A] = at(a => Numeric[A].plus(a, a))

val nums = 1 :: 2.5 :: 3L :: HNil
val doubled = nums.map(doubleIt)
// 2 :: 5.0 :: 6L :: HNil

// Poly function ที่ทำงานกับ Option
object liftOption extends Poly1:
  given [A]: Case.Aux[A, Option[A]] = at(Some(_))

val lifted = (1 :: "hello" :: true :: HNil).map(liftOption)
// Some(1) :: Some("hello") :: Some(true) :: HNil
```

---

## 7. Type Class Derivation อัตโนมัติ

### Eq Type Class

```scala
import shapeless.*

// Type class
trait Eq[A]:
  def eqv(a1: A, a2: A): Boolean

object Eq:
  // Primitive instances
  given Eq[Int]    = (a, b) => a == b
  given Eq[String] = (a, b) => a == b
  given Eq[Boolean] = (a, b) => a == b
  given Eq[Long]   = (a, b) => a == b
  given Eq[Double] = (a, b) => Math.abs(a - b) < 1e-10

  // HNil: always equal
  given Eq[HNil] = (_, _) => true

  // HCons: both head and tail must be equal
  given [H, T <: HList](using
    eqH: Eq[H],
    eqT: Eq[T]
  ): Eq[H :: T] = (h1 :: t1, h2 :: t2) =>
    eqH.eqv(h1, h2) && eqT.eqv(t1, t2)

  // Derive for any case class via Generic
  given [A, Repr <: HList](using
    gen: Generic.Aux[A, Repr],
    eqRepr: Eq[Repr]
  ): Eq[A] = (a1, a2) =>
    eqRepr.eqv(gen.to(a1), gen.to(a2))

// ใช้งาน
case class Point(x: Double, y: Double)
case class Circle(center: Point, radius: Double)

val p1 = Point(1.0, 2.0)
val p2 = Point(1.0, 2.0)
val p3 = Point(1.0, 3.0)

println(summon[Eq[Point]].eqv(p1, p2))  // true
println(summon[Eq[Point]].eqv(p1, p3))  // false
```

### Functor สำหรับ Product Types

```scala
// Show type class ที่ auto-derive
trait Show[A]:
  def show(a: A): String

object Show:
  given Show[Int]    = _.toString
  given Show[String] = s => s""""$s""""
  given Show[Double] = _.toString
  given Show[Boolean] = _.toString

  given Show[HNil] = _ => ""

  given [H, T <: HList](using
    showH: Show[H],
    showT: Show[T]
  ): Show[H :: T] = (h :: t) =>
    val tStr = showT.show(t)
    if tStr.isEmpty then showH.show(h)
    else s"${showH.show(h)}, $tStr"

  // LabelledGeneric version with field names
  given Show[HNil] = _ => ""

  given [K <: Symbol, V, T <: HList](using
    witness: Witness.Aux[K],
    showV: Show[V],
    showT: Show[T]
  ): Show[FieldType[K, V] :: T] = (hlist) =>
    val key   = witness.value.name
    val value = showV.show(hlist.head)
    val rest  = showT.show(hlist.tail)
    if rest.isEmpty then s"$key = $value"
    else s"$key = $value, $rest"

  given [A, Repr <: HList](using
    lgen: LabelledGeneric.Aux[A, Repr],
    showRepr: Show[Repr]
  ): Show[A] = a =>
    val typeName = a.getClass.getSimpleName
    s"$typeName(${showRepr.show(lgen.to(a))})"

// ใช้งาน
case class Person(name: String, age: Int, email: String)

val alice = Person("Alice", 30, "alice@example.com")
println(summon[Show[Person]].show(alice))
// Person(name = "Alice", age = 30, email = "alice@example.com")
```

---

## 8. Practical Use Cases

### Generic Merge

```scala
import shapeless.*
import shapeless.record.*
import shapeless.ops.record.*

// Merge two records
def mergeRecords[A, B, Out <: HList](a: A, b: B)(using
  lgenA: LabelledGeneric.Aux[A, ?],
  lgenB: LabelledGeneric.Aux[B, ?],
  merger: Merger.Aux[lgenA.Repr, lgenB.Repr, Out]
): Out = merger(lgenA.to(a), lgenB.to(b))
```

### Generic Lens

```scala
import shapeless.*

// Lens สำหรับเข้าถึงและแก้ไข nested fields
case class Address(street: String, city: String, country: String)
case class Person(name: String, age: Int, address: Address)

// สร้าง lens ด้วย Shapeless
val addressLens  = lens[Person] >> Symbol("address")
val cityLens     = lens[Person] >> Symbol("address") >> Symbol("city")

val alice = Person("Alice", 30, Address("123 Main St", "Bangkok", "Thailand"))

// Get
val city: String = cityLens.get(alice)     // "Bangkok"

// Set
val updated = cityLens.set(alice)("Chiang Mai")
// Person("Alice", 30, Address("123 Main St", "Chiang Mai", "Thailand"))

// Modify
val modified = cityLens.modify(alice)(_.toUpperCase)
// Person("Alice", 30, Address("123 Main St", "BANGKOK", "Thailand"))
```

### Generic Diff

```scala
import shapeless.*
import shapeless.labelled.*

// Find differences between two instances of same type
trait Diff[A]:
  def diff(a1: A, a2: A): Map[String, (Any, Any)]

object Diff:
  given [A: Eq]: Diff[A] = (a1, a2) =>
    if summon[Eq[A]].eqv(a1, a2) then Map.empty
    else Map("value" -> (a1, a2))

  given Diff[HNil] = (_, _) => Map.empty

  given [K <: Symbol, V, T <: HList](using
    witness: Witness.Aux[K],
    diffV: Diff[V],
    diffT: Diff[T]
  ): Diff[FieldType[K, V] :: T] = (h1, h2) =>
    val fieldDiffs = diffV.diff(h1.head, h2.head)
      .map { case (k, v) => s"${witness.value.name}.$k" -> v }
      .toMap ++ Map.empty  // if no sub-diff, add field itself
    val tailDiffs  = diffT.diff(h1.tail, h2.tail)
    fieldDiffs ++ tailDiffs

  given [A, Repr <: HList](using
    lgen: LabelledGeneric.Aux[A, Repr],
    diffRepr: Diff[Repr]
  ): Diff[A] = (a1, a2) =>
    diffRepr.diff(lgen.to(a1), lgen.to(a2))

// ใช้งาน
case class Config(host: String, port: Int, debug: Boolean)

val old   = Config("localhost", 8080, false)
val newer = Config("production.com", 443, true)

val diffs = summon[Diff[Config]].diff(old, newer)
// Map("host" -> ("localhost", "production.com"),
//     "port" -> (8080, 443),
//     "debug" -> (false, true))
```

### Generic Validator

```scala
import shapeless.*
import shapeless.labelled.*

trait Validator[A]:
  def validate(a: A): List[String]  // List of error messages

// สร้าง validation rules
object Validator:
  def instance[A](f: A => List[String]): Validator[A] = a => f(a)

  given Validator[HNil] = _ => Nil

  given [K <: Symbol, V, T <: HList](using
    witness: Witness.Aux[K],
    validV: Validator[V],
    validT: Validator[T]
  ): Validator[FieldType[K, V] :: T] = hlist =>
    val fieldName   = witness.value.name
    val headErrors  = validV.validate(hlist.head).map(e => s"$fieldName: $e")
    val tailErrors  = validT.validate(hlist.tail)
    headErrors ++ tailErrors

  given [A, Repr <: HList](using
    lgen: LabelledGeneric.Aux[A, Repr],
    validRepr: Validator[Repr]
  ): Validator[A] = a => validRepr.validate(lgen.to(a))

// Rules สำหรับ specific fields
given Validator[String] = s =>
  List.empty[String]
    .prependedAll(Option.when(s.isEmpty)("must not be empty"))
    .prependedAll(Option.when(s.length > 100)("too long"))

given Validator[Int] = i =>
  List.empty[String]
    .prependedAll(Option.when(i < 0)("must be non-negative"))

// ใช้งาน
case class Registration(name: String, age: Int, email: String)

val invalid = Registration("", -5, "invalid")
val errors  = summon[Validator[Registration]].validate(invalid)
// List("name: must not be empty", "age: must be non-negative")
```

---

## 9. Shapeless ใน Scala 3

Scala 3 มี built-in support สำหรับ generic programming ที่ดีกว่า ลด dependency บน Shapeless

### Mirror API (Scala 3)

```scala
// Scala 3 built-in generic programming
import scala.deriving.*
import scala.compiletime.*

// Auto-derive Show ด้วย Mirror
trait Show[A]:
  def show(a: A): String

object Show:
  given Show[Int]    = _.toString
  given Show[String] = s => s""""$s""""
  given Show[Double] = _.toString

  // Auto-derive สำหรับ Product types (case classes)
  inline given [A <: Product](using m: Mirror.ProductOf[A]): Show[A] =
    a =>
      val typeName = a.getClass.getSimpleName
      val elems    = a.productIterator.toList
      val labels   = m.asInstanceOf[{ def _2: Any }] match
        case _ => Nil  // simplified
      s"$typeName(${elems.mkString(", ")})"

// Shapeless 3 สำหรับ Scala 3
import shapeless3.deriving.*

// Derive type class instances
case class Point(x: Double, y: Double) derives Show, Eq
```

### shapeless3-deriving

```scala
import shapeless3.deriving.*

// CsvEncoder ด้วย shapeless3
trait CsvEncoder[A]:
  def encode(a: A): String

object CsvEncoder:
  given CsvEncoder[Int]    = _.toString
  given CsvEncoder[String] = s => s""""$s""""
  given CsvEncoder[Double] = _.toString
  given CsvEncoder[Boolean] = _.toString

  // Generic derivation
  given [A](using inst: K0.ProductInstances[CsvEncoder, A]): CsvEncoder[A] =
    a =>
      inst.foldLeft(a)((acc: List[String], enc: CsvEncoder[?], field: Any) =>
        acc :+ enc.asInstanceOf[CsvEncoder[Any]].encode(field)
      )(Nil).mkString(",")

// Derive automatically
case class User(id: Long, name: String, email: String) derives CsvEncoder
case class Product(id: Long, name: String, price: Double) derives CsvEncoder

val user    = User(1L, "Alice", "alice@example.com")
val csv     = summon[CsvEncoder[User]].encode(user)
```

---

## 10. สรุป

Shapeless เป็น library ที่ทรงพลังสำหรับ generic programming ใน Scala:

### สิ่งที่เรียนรู้

| Concept | ใช้สำหรับ |
|---------|-----------|
| HList | Type-safe heterogeneous collections |
| Coproduct | Sum types (one of N types) |
| Generic | Case class <-> HList conversion |
| LabelledGeneric | รักษา field names ไว้ |
| Poly Functions | Polymorphic operations บน HList |
| Type Class Derivation | Auto-generate instances |

### เมื่อใช้ Shapeless

- **Auto-derive type class instances** - ไม่ต้องเขียน boilerplate
- **Generic transformations** - แปลง data structures
- **Type-safe operations** - operations ที่ checked ที่ compile time
- **Code generation** - ลด repetitive code

### ข้อควรระวัง

```scala
// Shapeless อาจทำให้ compile time ช้า
// ถ้า type class hierarchy ลึกมาก

// ✅ ใช้ Lazy เพื่อหลีกเลี่ยง implicit divergence
given [H, T <: HList](using
  showH: Lazy[Show[H]],  // Lazy ป้องกัน divergence
  showT: Show[T]
): Show[H :: T] = ???

// ✅ Scala 3 Mirror API เป็นทางเลือกที่ดีกว่าสำหรับโปรเจกต์ใหม่
```

### Shapeless vs Scala 3 Mirrors

| Feature | Shapeless 2 | Shapeless 3 | Scala 3 Mirror |
|---------|-------------|-------------|----------------|
| Complexity | High | Medium | Low |
| Performance | Good | Good | Best |
| Flexibility | Highest | High | Medium |
| Learning Curve | Steep | Moderate | Gentle |

---

*[← Part 66: Slick Database](part-66-slick.md) | [Part 68: Cats MTL →](part-68-cats-mtl.md)*
