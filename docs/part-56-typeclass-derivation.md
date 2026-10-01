# ส่วนที่ 56: Typeclass Derivation ใน Scala 3

## สารบัญ

1. [Typeclass Derivation คืออะไร](#typeclass-derivation-คืออะไร)
2. [Mirror.Of สำหรับ Generic Programming](#mirrorof-สำหรับ-generic-programming)
3. [การ Derive Instances อัตโนมัติ](#การ-derive-instances-อัตโนมัติ)
4. [Auto-deriving Encoder/Decoder](#auto-deriving-encoderdecoder)
5. [Custom Derivation ด้วย Scala 3 Macros](#custom-derivation-ด้วย-scala-3-macros)
6. [Magnolia-style Derivation](#magnolia-style-derivation)
7. [Derivation สำหรับ ADTs](#derivation-สำหรับ-adts)
8. [Performance Considerations](#performance-considerations)
9. [Integration กับ Libraries](#integration-กับ-libraries)
10. [Best Practices](#best-practices)
11. [สรุป](#สรุป)

---

## Typeclass Derivation คืออะไร

Typeclass Derivation คือกระบวนการสร้าง typeclass instances โดยอัตโนมัติจาก structure ของ type โดยไม่ต้องเขียน boilerplate code ด้วยมือ

### ปัญหาที่ Derivation แก้ไข

```scala
// โดยไม่มี derivation - ต้องเขียน instance เองทุก type
case class User(name: String, age: Int)
case class Product(id: Int, price: Double, name: String)
case class Order(userId: Int, products: List[Product])

// ต้องเขียน Show instance สำหรับทุก type
given showUser: Show[User] = user =>
  s"User(name=${user.name}, age=${user.age})"

given showProduct: Show[Product] = product =>
  s"Product(id=${product.id}, price=${product.price}, name=${product.name})"

given showOrder: Show[Order] = order =>
  s"Order(userId=${order.userId}, products=[${order.products.map(showProduct.show).mkString(", ")}])"

// ด้วย derivation - derive อัตโนมัติ!
case class User(name: String, age: Int) derives Show
case class Product(id: Int, price: Double, name: String) derives Show
case class Order(userId: Int, products: List[Product]) derives Show
```

### ประโยชน์ของ Typeclass Derivation

1. **ลด boilerplate** - ไม่ต้องเขียน code ซ้ำๆ
2. **Type-safe** - compiler ตรวจสอบทุกอย่างในเวลา compile
3. **Maintainable** - เพิ่ม/ลด fields ใน case class โดยไม่ต้องแก้ instances
4. **Composable** - derive จาก derived instances ได้

### Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.typelevel"   %% "cats-core"     % "2.10.0",
  "io.circe"        %% "circe-core"    % "0.14.6",
  "io.circe"        %% "circe-generic" % "0.14.6",
  "io.circe"        %% "circe-parser"  % "0.14.6",
  "com.softwaremill.magnolia1_3" %% "magnolia" % "1.3.4"
)
```

---

## Mirror.Of สำหรับ Generic Programming

Scala 3 แนะนำ `Mirror` type class ที่เป็น foundation ของ typeclass derivation

### Mirror Types ใน Scala 3

```scala
// Mirror.Sum - สำหรับ sealed traits (sum types / ADTs)
// Mirror.Product - สำหรับ case classes (product types)

import scala.deriving.*
import scala.compiletime.*

// Mirror.Product มี:
//   MirroredType - ชนิดของ type
//   MirroredLabel - ชื่อของ type
//   MirroredElemTypes - tuple ของ element types
//   MirroredElemLabels - tuple ของ element labels

case class Person(name: String, age: Int, email: String)

// Compiler สร้าง Mirror.Product[Person] ให้อัตโนมัติ
// โดยมี:
//   MirroredType = Person
//   MirroredLabel = "Person"
//   MirroredElemTypes = (String, Int, String)
//   MirroredElemLabels = ("name", "age", "email")

// เรียกดู Mirror
val personMirror = summon[Mirror.Of[Person]]
```

### การใช้ Mirror สำหรับ Introspection

```scala
import scala.deriving.*
import scala.compiletime.*

// utility functions สำหรับ reflection
inline def typeLabel[T](using m: Mirror.Of[T]): String =
  constValue[m.MirroredLabel]

inline def fieldLabels[T](using m: Mirror.ProductOf[T]): List[String] =
  constValueTuple[m.MirroredElemLabels].toList.map(_.toString)

inline def sumVariants[T](using m: Mirror.SumOf[T]): List[String] =
  constValueTuple[m.MirroredElemLabels].toList.map(_.toString)

case class Book(title: String, author: String, pages: Int)
sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

@main def mirrorDemo(): Unit =
  println(s"Type label: ${typeLabel[Book]}")
  println(s"Field labels: ${fieldLabels[Book]}")
  println(s"Sum variants: ${sumVariants[Shape]}")
  // Output:
  // Type label: Book
  // Field labels: List(title, author, pages)
  // Sum variants: List(Circle, Rectangle, Triangle)
```

### Mirror-based Generic Operations

```scala
import scala.deriving.*
import scala.compiletime.*

// Generic arity (จำนวน fields)
inline def arity[T](using m: Mirror.ProductOf[T]): Int =
  constValue[Tuple.Size[m.MirroredElemTypes]]

// Generic equality
inline def genericEquals[T](using m: Mirror.ProductOf[T])(a: T, b: T): Boolean =
  val aElems = m.fromOrdinal  // ไม่มีใน Scala 3 โดยตรง
  // ใช้ Eq typeclass แทน
  a == b  // case class มี == อยู่แล้ว

// สร้าง instance จาก Tuple
case class Vec3(x: Double, y: Double, z: Double)

val mirror = summon[Mirror.ProductOf[Vec3]]
val tuple: (Double, Double, Double) = (1.0, 2.0, 3.0)
val vec: Vec3 = mirror.fromProduct(tuple)
println(vec)  // Vec3(1.0,2.0,3.0)
```

---

## การ Derive Instances อัตโนมัติ

### Simple Show Typeclass

```scala
import scala.deriving.*
import scala.compiletime.*

// กำหนด Show typeclass
trait Show[A]:
  def show(a: A): String

object Show:
  def apply[A](using s: Show[A]): Show[A] = s

  // Instances สำหรับ primitive types
  given Show[String]  = a => s""""$a""""
  given Show[Int]     = _.toString
  given Show[Double]  = a => f"$a%.2f"
  given Show[Boolean] = _.toString
  given [A](using Show[A]): Show[List[A]] =
    xs => xs.map(summon[Show[A]].show).mkString("[", ", ", "]")
  given [A](using Show[A]): Show[Option[A]] =
    _.fold("None")(a => s"Some(${summon[Show[A]].show(a)})")

  // Derivation สำหรับ Product types (case classes)
  inline def deriveProduct[T](using m: Mirror.ProductOf[T]): Show[T] =
    new Show[T]:
      def show(a: T): String =
        val label = constValue[m.MirroredLabel]
        val fields = summonAll[Tuple.Map[m.MirroredElemTypes, Show]]
        val values = a.asInstanceOf[Product].productIterator.toList
        val fieldLabels = constValueTuple[m.MirroredElemLabels].toList.map(_.toString)
        
        val fieldStrings = fieldLabels.zip(values).zip(fields.toList).map {
          case ((name, value), showInst) =>
            s"$name=${showInst.asInstanceOf[Show[Any]].show(value)}"
        }
        s"$label(${fieldStrings.mkString(", ")})"

  // Derivation สำหรับ Sum types (sealed traits)
  inline def deriveSum[T](using m: Mirror.SumOf[T]): Show[T] =
    new Show[T]:
      val instances = summonAll[Tuple.Map[m.MirroredElemTypes, Show]]
      def show(a: T): String =
        val ord = m.ordinal(a)
        instances.productElement(ord).asInstanceOf[Show[Any]].show(a)

  // Entry point สำหรับ derive
  inline given derived[T](using m: Mirror.Of[T]): Show[T] =
    inline m match
      case p: Mirror.ProductOf[T] => deriveProduct(using p)
      case s: Mirror.SumOf[T]     => deriveSum(using s)

// ใช้งาน
case class Address(street: String, city: String, zip: String) derives Show
case class Person(name: String, age: Int, address: Address) derives Show

sealed trait Status derives Show
case class Active(since: String) extends Status
case class Inactive(reason: String) extends Status
case object Pending extends Status

@main def showDemo(): Unit =
  val addr = Address("123 Main St", "Bangkok", "10110")
  val person = Person("Alice", 30, addr)
  
  println(Show[Person].show(person))
  println(Show[Status].show(Active("2024-01-01")))
  println(Show[Status].show(Inactive("Left company")))
```

### Derive Multiple Typeclasses พร้อมกัน

```scala
import scala.deriving.*

// กำหนดหลาย typeclasses ที่ต้องการ derive
trait Eq[A]:
  def eqv(a: A, b: A): Boolean

trait Hash[A]:
  def hash(a: A): Int

trait Default[A]:
  def default: A

// Derive ทุกอย่างพร้อมกัน
case class Config(
  host: String,
  port: Int,
  debug: Boolean
) derives Show, Eq, Hash

object Eq:
  given Eq[String]  = _ == _
  given Eq[Int]     = _ == _
  given Eq[Boolean] = _ == _
  
  inline given derived[T](using m: Mirror.ProductOf[T]): Eq[T] =
    new Eq[T]:
      def eqv(a: T, b: T): Boolean =
        val instances = summonAll[Tuple.Map[m.MirroredElemTypes, Eq]]
        val aValues = a.asInstanceOf[Product].productIterator.toList
        val bValues = b.asInstanceOf[Product].productIterator.toList
        aValues.zip(bValues).zip(instances.toList).forall {
          case ((av, bv), inst) =>
            inst.asInstanceOf[Eq[Any]].eqv(av, bv)
        }

object Hash:
  given Hash[String]  = _.hashCode
  given Hash[Int]     = identity
  given Hash[Boolean] = if _ then 1 else 0
  
  inline given derived[T](using m: Mirror.ProductOf[T]): Hash[T] =
    new Hash[T]:
      def hash(a: T): Int =
        val instances = summonAll[Tuple.Map[m.MirroredElemTypes, Hash]]
        val values = a.asInstanceOf[Product].productIterator.toList
        values.zip(instances.toList).foldLeft(17) {
          case (acc, (v, inst)) =>
            acc * 31 + inst.asInstanceOf[Hash[Any]].hash(v)
        }
```

---

## Auto-deriving Encoder/Decoder

### Custom JSON Encoder/Decoder

```scala
import scala.deriving.*
import scala.compiletime.*

// Simple JSON representation
enum Json:
  case JsonNull
  case JsonBool(value: Boolean)
  case JsonNumber(value: Double)
  case JsonString(value: String)
  case JsonArray(values: List[Json])
  case JsonObject(fields: Map[String, Json])

import Json.*

// Encoder typeclass
trait Encoder[A]:
  def encode(a: A): Json

object Encoder:
  def apply[A](using e: Encoder[A]): Encoder[A] = e

  // Primitive instances
  given Encoder[String]  = a => JsonString(a)
  given Encoder[Int]     = a => JsonNumber(a.toDouble)
  given Encoder[Double]  = a => JsonNumber(a)
  given Encoder[Boolean] = a => JsonBool(a)
  given Encoder[Unit]    = _ => JsonNull
  
  given [A](using Encoder[A]): Encoder[List[A]] =
    xs => JsonArray(xs.map(summon[Encoder[A]].encode))
  
  given [A](using Encoder[A]): Encoder[Option[A]] =
    _.fold(JsonNull)(summon[Encoder[A]].encode)
  
  given [A, B](using Encoder[A], Encoder[B]): Encoder[Map[A, B]] =
    m => JsonObject(
      m.map { (k, v) =>
        summon[Encoder[A]].encode(k).asInstanceOf[JsonString].value ->
        summon[Encoder[B]].encode(v)
      }
    )

  // Derivation
  inline given derived[T](using m: Mirror.Of[T]): Encoder[T] =
    inline m match
      case p: Mirror.ProductOf[T] => deriveProduct(using p)
      case s: Mirror.SumOf[T]     => deriveSum(using s)

  private inline def deriveProduct[T](using m: Mirror.ProductOf[T]): Encoder[T] =
    new Encoder[T]:
      val instances = summonAll[Tuple.Map[m.MirroredElemTypes, Encoder]]
      val labels    = constValueTuple[m.MirroredElemLabels].toList.map(_.toString)
      
      def encode(a: T): Json =
        val values = a.asInstanceOf[Product].productIterator.toList
        val fields = labels.zip(values).zip(instances.toList).map {
          case ((label, value), enc) =>
            label -> enc.asInstanceOf[Encoder[Any]].encode(value)
        }
        JsonObject(fields.toMap)

  private inline def deriveSum[T](using m: Mirror.SumOf[T]): Encoder[T] =
    new Encoder[T]:
      val instances = summonAll[Tuple.Map[m.MirroredElemTypes, Encoder]]
      val labels    = constValueTuple[m.MirroredElemLabels].toList.map(_.toString)
      
      def encode(a: T): Json =
        val ord = m.ordinal(a)
        val variantLabel = labels(ord)
        val encoded = instances.productElement(ord).asInstanceOf[Encoder[Any]].encode(a)
        JsonObject(Map("type" -> JsonString(variantLabel), "data" -> encoded))

// Decoder typeclass
trait Decoder[A]:
  def decode(json: Json): Either[String, A]

object Decoder:
  def apply[A](using d: Decoder[A]): Decoder[A] = d

  // Primitive instances
  given Decoder[String] = {
    case JsonString(s) => Right(s)
    case other => Left(s"Expected string, got: $other")
  }
  
  given Decoder[Int] = {
    case JsonNumber(n) => Right(n.toInt)
    case other => Left(s"Expected number, got: $other")
  }
  
  given Decoder[Double] = {
    case JsonNumber(n) => Right(n)
    case other => Left(s"Expected number, got: $other")
  }
  
  given Decoder[Boolean] = {
    case JsonBool(b) => Right(b)
    case other => Left(s"Expected boolean, got: $other")
  }
  
  given [A](using Decoder[A]): Decoder[List[A]] = {
    case JsonArray(values) =>
      values.foldRight(Right(Nil): Either[String, List[A]]) { (json, acc) =>
        for
          list <- acc
          item <- summon[Decoder[A]].decode(json)
        yield item :: list
      }
    case other => Left(s"Expected array, got: $other")
  }
  
  given [A](using Decoder[A]): Decoder[Option[A]] = {
    case JsonNull  => Right(None)
    case json      => summon[Decoder[A]].decode(json).map(Some(_))
  }

  // Derivation สำหรับ Product types
  inline given derived[T](using m: Mirror.ProductOf[T]): Decoder[T] =
    new Decoder[T]:
      val instances = summonAll[Tuple.Map[m.MirroredElemTypes, Decoder]]
      val labels    = constValueTuple[m.MirroredElemLabels].toList.map(_.toString)
      
      def decode(json: Json): Either[String, T] =
        json match
          case JsonObject(fields) =>
            val results = labels.zip(instances.toList).map { (label, dec) =>
              fields.get(label) match
                case Some(value) => dec.asInstanceOf[Decoder[Any]].decode(value)
                case None        => Left(s"Missing field: $label")
            }
            val errors = results.collect { case Left(err) => err }
            if errors.nonEmpty then Left(errors.mkString("; "))
            else
              val values = results.collect { case Right(v) => v }
              Right(m.fromProduct(Tuple.fromArray(values.toArray).asInstanceOf))
          case other => Left(s"Expected object, got: $other")

// ใช้งาน
case class Address(
  street: String,
  city: String,
  country: String
) derives Encoder, Decoder

case class User(
  id: Int,
  name: String,
  email: String,
  address: Address,
  active: Boolean
) derives Encoder, Decoder

@main def encoderDecoderDemo(): Unit =
  val user = User(
    id = 1,
    name = "Alice",
    email = "alice@example.com",
    address = Address("123 Main St", "Bangkok", "Thailand"),
    active = true
  )

  // Encode
  val json = Encoder[User].encode(user)
  println(s"Encoded: $json")

  // Decode
  val decoded = Decoder[User].decode(json)
  println(s"Decoded: $decoded")
  println(s"Round-trip success: ${decoded == Right(user)}")
```

---

## Custom Derivation ด้วย Scala 3 Macros

### Macro-based Derivation

```scala
import scala.quoted.*

// Typeclass สำหรับ validation
trait Validate[A]:
  def validate(a: A): List[String]  // List ของ error messages

object Validate:
  def apply[A](using v: Validate[A]): Validate[A] = v

  // Basic validators
  given Validate[String] = a =>
    if a.isEmpty then List("String must not be empty")
    else Nil

  given Validate[Int] = a =>
    if a < 0 then List(s"Int must be non-negative, got: $a")
    else Nil

  given Validate[Double] = a =>
    if a.isNaN || a.isInfinite then List(s"Double must be finite, got: $a")
    else Nil

  given [A](using Validate[A]): Validate[List[A]] = list =>
    list.zipWithIndex.flatMap { (item, idx) =>
      summon[Validate[A]].validate(item).map(err => s"[$idx] $err")
    }

  given [A](using Validate[A]): Validate[Option[A]] = {
    case None    => Nil
    case Some(a) => summon[Validate[A]].validate(a)
  }

// Macro สำหรับ derive Validate
object ValidateMacro:
  inline def derive[T]: Validate[T] = ${ deriveImpl[T] }

  def deriveImpl[T: Type](using Quotes): Expr[Validate[T]] =
    import quotes.reflect.*
    
    val tpe = TypeRepr.of[T]
    val sym = tpe.typeSymbol
    
    if sym.isClassDef && !sym.flags.is(Flags.Sealed) then
      deriveCaseClass[T]
    else
      report.errorAndAbort(s"Cannot derive Validate for ${sym.name}")

  def deriveCaseClass[T: Type](using Quotes): Expr[Validate[T]] =
    import quotes.reflect.*
    
    val tpe = TypeRepr.of[T]
    val sym = tpe.typeSymbol
    val fields = sym.primaryConstructor.paramSymss.flatten
    
    '{
      new Validate[T]:
        def validate(a: T): List[String] =
          ${
            val fieldValidations = fields.map { field =>
              val fieldType = tpe.memberType(field)
              // สร้าง validation expression สำหรับแต่ละ field
              '{ List.empty[String] }  // simplified
            }
            '{ ${Expr.ofList(fieldValidations)}.flatten }
          }
    }
```

### Annotation-driven Derivation

```scala
import scala.annotation.StaticAnnotation

// Custom annotations สำหรับ validation rules
class minLength(n: Int) extends StaticAnnotation
class maxLength(n: Int) extends StaticAnnotation
class min(n: Double) extends StaticAnnotation
class max(n: Double) extends StaticAnnotation
class email extends StaticAnnotation
class nonEmpty extends StaticAnnotation

// Case class ที่ใช้ annotations
case class RegistrationForm(
  @minLength(3) @maxLength(50) username: String,
  @email email: String,
  @minLength(8) password: String,
  @min(0) @max(150) age: Int
)

// Validator ที่ใช้ annotations
object AnnotationValidator:
  def validate(form: RegistrationForm): List[String] =
    val errors = scala.collection.mutable.ListBuffer[String]()
    
    // username validation
    if form.username.length < 3 then
      errors += "username must be at least 3 characters"
    if form.username.length > 50 then
      errors += "username must be at most 50 characters"
    
    // email validation
    if !form.email.contains("@") then
      errors += "email must be a valid email address"
    
    // password validation
    if form.password.length < 8 then
      errors += "password must be at least 8 characters"
    
    // age validation
    if form.age < 0 then errors += "age must be >= 0"
    if form.age > 150 then errors += "age must be <= 150"
    
    errors.toList

@main def annotationDemo(): Unit =
  val validForm = RegistrationForm("alice123", "alice@example.com", "secure123", 25)
  val invalidForm = RegistrationForm("al", "not-an-email", "short", -5)
  
  println("Valid form errors:")
  AnnotationValidator.validate(validForm).foreach(e => println(s"  - $e"))
  
  println("Invalid form errors:")
  AnnotationValidator.validate(invalidForm).foreach(e => println(s"  - $e"))
```

---

## Magnolia-style Derivation

### Magnolia TypeClass Derivation

```scala
import magnolia1.*

// Show typeclass ด้วย Magnolia
trait MagnoliaShow[A]:
  def show(a: A): String

object MagnoliaShow extends AutoDerivation[MagnoliaShow]:
  def join[T](ctx: CaseClass[MagnoliaShow, T]): MagnoliaShow[T] =
    new MagnoliaShow[T]:
      def show(a: T): String =
        val fields = ctx.params.map { param =>
          s"${param.label}=${param.typeclass.show(param.deref(a))}"
        }
        s"${ctx.typeInfo.short}(${fields.mkString(", ")})"

  def split[T](ctx: SealedTrait[MagnoliaShow, T]): MagnoliaShow[T] =
    new MagnoliaShow[T]:
      def show(a: T): String =
        ctx.choose(a) { sub =>
          sub.typeclass.show(sub.value)
        }

  // Primitive instances
  given MagnoliaShow[String]  = a => s""""$a""""
  given MagnoliaShow[Int]     = _.toString
  given MagnoliaShow[Double]  = a => f"$a%.2f"
  given MagnoliaShow[Boolean] = _.toString
  given [A](using MagnoliaShow[A]): MagnoliaShow[List[A]] =
    xs => xs.map(summon[MagnoliaShow[A]].show).mkString("[", ", ", "]")

// ใช้ Magnolia สำหรับ JSON encoding
trait JsonEncoder[A]:
  def encode(a: A): String

object JsonEncoder extends AutoDerivation[JsonEncoder]:
  def join[T](ctx: CaseClass[JsonEncoder, T]): JsonEncoder[T] =
    new JsonEncoder[T]:
      def encode(a: T): String =
        val fields = ctx.params.map { param =>
          s""""${param.label}": ${param.typeclass.encode(param.deref(a))}"""
        }
        s"{${fields.mkString(", ")}}"

  def split[T](ctx: SealedTrait[JsonEncoder, T]): JsonEncoder[T] =
    new JsonEncoder[T]:
      def encode(a: T): String =
        ctx.choose(a) { sub =>
          s"""{"type": "${sub.typeInfo.short}", "value": ${sub.typeclass.encode(sub.value)}}"""
        }

  given JsonEncoder[String]  = a => s""""$a""""
  given JsonEncoder[Int]     = _.toString
  given JsonEncoder[Double]  = _.toString
  given JsonEncoder[Boolean] = _.toString
  given JsonEncoder[Unit]    = _ => "null"
  given [A](using JsonEncoder[A]): JsonEncoder[List[A]] =
    xs => s"[${xs.map(summon[JsonEncoder[A]].encode).mkString(", ")}]"
  given [A](using JsonEncoder[A]): JsonEncoder[Option[A]] =
    _.fold("null")(summon[JsonEncoder[A]].encode)

// Domain models
case class Product(
  id: Int,
  name: String,
  price: Double,
  tags: List[String]
) derives MagnoliaShow, JsonEncoder

sealed trait PaymentMethod derives MagnoliaShow, JsonEncoder
case class CreditCard(number: String, expiry: String) extends PaymentMethod
case class BankTransfer(accountNo: String, bankCode: String) extends PaymentMethod
case object CashOnDelivery extends PaymentMethod

case class Order(
  orderId: String,
  product: Product,
  quantity: Int,
  payment: PaymentMethod,
  notes: Option[String]
) derives MagnoliaShow, JsonEncoder

@main def magnoliaDemo(): Unit =
  val product = Product(1, "Laptop", 29999.99, List("electronics", "computers"))
  val order = Order(
    orderId = "ORD-001",
    product = product,
    quantity = 2,
    payment = CreditCard("4111-xxxx-xxxx-1234", "12/26"),
    notes = Some("Please gift wrap")
  )

  println("=== MagnoliaShow ===")
  println(summon[MagnoliaShow[Order]].show(order))
  
  println("\n=== JsonEncoder ===")
  println(summon[JsonEncoder[Order]].encode(order))
```

---

## Derivation สำหรับ ADTs

### Recursive ADT Derivation

```scala
import scala.deriving.*
import scala.compiletime.*

// Recursive data structure
sealed trait Expr derives Show
case class Num(value: Int) extends Expr
case class Add(left: Expr, right: Expr) extends Expr
case class Mul(left: Expr, right: Expr) extends Expr
case class Var(name: String) extends Expr
case class Let(name: String, value: Expr, body: Expr) extends Expr

// Show instance จะทำงาน recursive ได้
object Expr:
  // Pretty printer
  def prettyPrint(expr: Expr, indent: Int = 0): String =
    val pad = "  " * indent
    expr match
      case Num(v)        => s"${pad}$v"
      case Var(n)        => s"${pad}$n"
      case Add(l, r)     =>
        s"${pad}(+\n${prettyPrint(l, indent+1)}\n${prettyPrint(r, indent+1)})"
      case Mul(l, r)     =>
        s"${pad}(*\n${prettyPrint(l, indent+1)}\n${prettyPrint(r, indent+1)})"
      case Let(n, v, b)  =>
        s"${pad}(let $n =\n${prettyPrint(v, indent+1)}\n${pad}in\n${prettyPrint(b, indent+1)})"

  // Evaluator
  def eval(expr: Expr, env: Map[String, Int] = Map.empty): Int =
    expr match
      case Num(v)        => v
      case Var(n)        => env.getOrElse(n, throw new RuntimeException(s"Undefined: $n"))
      case Add(l, r)     => eval(l, env) + eval(r, env)
      case Mul(l, r)     => eval(l, env) * eval(r, env)
      case Let(n, v, b)  => eval(b, env + (n -> eval(v, env)))

@main def adtDerivation(): Unit =
  // let x = 3 in (x + 2) * 5
  val program = Let("x", Num(3), Mul(Add(Var("x"), Num(2)), Num(5)))
  
  println("Expression tree:")
  println(Expr.prettyPrint(program))
  println(s"\nEvaluates to: ${Expr.eval(program)}")
  
  // Derived Show
  println(s"\nShow: ${summon[Show[Expr]].show(program)}")
```

### Generic Tree Derivation

```scala
import scala.deriving.*
import scala.compiletime.*

// Generic tree ที่ support derivation
sealed trait Tree[+A]
case class Leaf[A](value: A) extends Tree[A]
case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A]
case object Empty extends Tree[Nothing]

// Functor instance ที่ derive ได้
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[Tree] with
  def map[A, B](fa: Tree[A])(f: A => B): Tree[B] =
    fa match
      case Leaf(a)     => Leaf(f(a))
      case Node(l, r)  => Node(map(l)(f), map(r)(f))
      case Empty       => Empty

// Foldable instance
trait Foldable[F[_]]:
  def foldLeft[A, B](fa: F[A], b: B)(f: (B, A) => B): B

given Foldable[Tree] with
  def foldLeft[A, B](fa: Tree[A], b: B)(f: (B, A) => B): B =
    fa match
      case Leaf(a)     => f(b, a)
      case Node(l, r)  =>
        val lb = foldLeft(l, b)(f)
        foldLeft(r, lb)(f)
      case Empty       => b

// Traverse instance
trait Traverse[F[_]] extends Functor[F] with Foldable[F]:
  def traverse[G[_]: cats.Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]

@main def treeDerivation(): Unit =
  val tree: Tree[Int] = Node(
    Node(Leaf(1), Leaf(2)),
    Node(Leaf(3), Node(Leaf(4), Leaf(5)))
  )
  
  val doubled = summon[Functor[Tree]].map(tree)(_ * 2)
  val sum = summon[Foldable[Tree]].foldLeft(tree, 0)(_ + _)
  
  println(s"Original: $tree")
  println(s"Doubled: $doubled")
  println(s"Sum: $sum")
```

---

## Performance Considerations

### Compile-time vs Runtime Performance

```scala
import scala.deriving.*
import scala.compiletime.*

// การวัด performance ของ derived vs manual
object PerformanceComparison:

  // Manual instance (เร็วที่สุด)
  case class Point(x: Double, y: Double)
  
  val manualShow: Show[Point] = 
    p => s"Point(x=${p.x}, y=${p.y})"

  // Derived instance (สะดวกแต่อาจช้ากว่าเล็กน้อย)
  case class PointDerived(x: Double, y: Double) derives Show

  // วิธีเพิ่ม performance สำหรับ derived instances
  // 1. ใช้ inline เพื่อ eliminate overhead
  // 2. Cache instances ไว้ใน object
  // 3. ใช้ lazy val สำหรับ initialization

  // Lazy initialization ป้องกัน initialization order problems
  object ShowInstances:
    lazy val pointShow: Show[Point] = 
      summon[Show[PointDerived]].asInstanceOf[Show[Point]]

// Benchmark-style comparison
import scala.util.Try

def benchmark[A](name: String, n: Int)(block: => A): Unit =
  val start = System.nanoTime()
  var i = 0
  while i < n do
    block
    i += 1
  val end = System.nanoTime()
  println(f"$name: ${(end - start) / 1e6}%.2f ms for $n iterations")

@main def performanceDemo(): Unit =
  case class Data(a: Int, b: String, c: Double) derives Show
  
  val data = Data(42, "hello", 3.14)
  val n = 100000
  
  benchmark("Show.show", n):
    summon[Show[Data]].show(data)
  
  benchmark("toString", n):
    data.toString
  
  benchmark("Manual format", n):
    s"Data(a=${data.a}, b=${data.b}, c=${data.c})"
```

### การหลีกเลี่ยง Implicit Search Overhead

```scala
import scala.deriving.*

// ปัญหา: implicit search ช้าถ้ามี nested derivations มาก
// วิธีแก้: กำหนด derived instances ใน companion object

case class Address(street: String, city: String)
object Address:
  // กำหนด explicitly ใน companion object
  // จะถูกหาเจอก่อน implicit search
  given show: Show[Address] = Show.derived[Address]
  given encoder: Encoder[Address] = Encoder.derived[Address]

case class User(name: String, address: Address)
object User:
  given show: Show[User] = Show.derived[User]
  given encoder: Encoder[User] = Encoder.derived[User]

// Avoid re-derivation in every callsite
// Bad: summon[Show[User]] ทุกครั้ง (อาจ re-derive)
// Good: กำหนดไว้ใน companion object และ reference ตรงๆ

def printUser(user: User): Unit =
  println(User.show.show(user))  // ใช้ explicit instance
  // ดีกว่า:
  // println(summon[Show[User]].show(user))  // อาจ trigger re-derivation
```

### Derivation ที่มีประสิทธิภาพ

```scala
import scala.deriving.*
import scala.compiletime.*

// เทคนิคสำหรับ high-performance derivation

// 1. Cache field accessors
trait FastEncoder[A]:
  def encode(a: A): Array[Byte]

object FastEncoder:
  // ใช้ arrays แทน lists สำหรับ performance
  inline def derived[T](using m: Mirror.ProductOf[T]): FastEncoder[T] =
    new FastEncoder[T]:
      // Pre-compute field encoders ตอน initialization
      private val encoders =
        summonAll[Tuple.Map[m.MirroredElemTypes, FastEncoder]]
          .toList
          .toArray
      
      def encode(a: T): Array[Byte] =
        val product = a.asInstanceOf[Product]
        val parts = Array.tabulate(encoders.length) { i =>
          encoders(i).asInstanceOf[FastEncoder[Any]]
            .encode(product.productElement(i))
        }
        // Combine all parts
        parts.foldLeft(Array.empty[Byte])(_ ++ _)

  given FastEncoder[String]  = s => s.getBytes("UTF-8")
  given FastEncoder[Int]     = n => BigInt(n).toByteArray
  given FastEncoder[Double]  = n => java.lang.Double.doubleToLongBits(n)
                                      .toByte.toString.getBytes

// 2. Memoization ของ derived instances
import scala.collection.concurrent.TrieMap

object DerivedCache:
  private val cache = TrieMap.empty[Class[?], Any]
  
  def get[T](clazz: Class[T])(create: => Show[T]): Show[T] =
    cache.getOrElseUpdate(clazz, create).asInstanceOf[Show[T]]
```

---

## Integration กับ Libraries

### การใช้ circe สำหรับ JSON

```scala
import io.circe.*
import io.circe.generic.semiauto.*
import io.circe.parser.*
import io.circe.syntax.*

// circe auto derivation
case class Config(
  host: String,
  port: Int,
  database: String,
  poolSize: Int,
  timeout: Long
) derives Encoder.AsObject, Decoder

case class AppConfig(
  server: Config,
  cache: Config,
  features: Map[String, Boolean]
) derives Encoder.AsObject, Decoder

@main def circeDemo(): Unit =
  val config = AppConfig(
    server = Config("localhost", 8080, "mydb", 10, 30000L),
    cache = Config("redis-host", 6379, "0", 5, 5000L),
    features = Map("featureA" -> true, "featureB" -> false)
  )

  // Encode to JSON
  val json = config.asJson
  println("JSON:")
  println(json.spaces2)

  // Decode from JSON
  val jsonStr = json.noSpaces
  decode[AppConfig](jsonStr) match
    case Right(decoded) =>
      println(s"\nDecoded successfully: ${decoded == config}")
    case Left(err) =>
      println(s"\nDecode error: $err")
```

### การใช้ doobie สำหรับ Database

```scala
import doobie.*
import doobie.implicits.*
import cats.effect.*

// doobie ใช้ shapeless/magnolia-style derivation สำหรับ Read/Write

case class Employee(
  id: Int,
  name: String,
  department: String,
  salary: Double
)

// doobie derive Read และ Write instances อัตโนมัติ
// ไม่ต้องกำหนดเอง!

object EmployeeRepo:
  def findAll: ConnectionIO[List[Employee]] =
    sql"SELECT id, name, department, salary FROM employees"
      .query[Employee]  // auto-derive Read[Employee]
      .to[List]

  def insert(emp: Employee): ConnectionIO[Int] =
    sql"""
      INSERT INTO employees (id, name, department, salary)
      VALUES (${emp.id}, ${emp.name}, ${emp.department}, ${emp.salary})
    """.update.run

  def findByDepartment(dept: String): ConnectionIO[List[Employee]] =
    sql"SELECT id, name, department, salary FROM employees WHERE department = $dept"
      .query[Employee]
      .to[List]
```

---

## Best Practices

### แนวทางที่ดีสำหรับ Typeclass Derivation

```scala
// 1. กำหนด instances ใน companion objects
case class Order(id: Int, total: Double, status: String)
object Order:
  given orderShow: Show[Order] = Show.derived
  given orderEncoder: Encoder[Order] = Encoder.derived
  given orderDecoder: Decoder[Order] = Decoder.derived

// 2. ใช้ given instances ที่ explicit แทน derives keyword
// เมื่อต้องการควบคุมมากขึ้น
case class SensitiveData(userId: Int, token: String)
object SensitiveData:
  // Override default Show เพื่อ mask sensitive fields
  given Show[SensitiveData] = data =>
    s"SensitiveData(userId=${data.userId}, token=***)"

// 3. Test derived instances
class DerivedInstanceSpec extends munit.FunSuite:
  test("derived Show works for simple case class"):
    case class Foo(a: Int, b: String) derives Show
    val foo = Foo(42, "hello")
    val result = summon[Show[Foo]].show(foo)
    assert(result.contains("42"))
    assert(result.contains("hello"))

  test("derived Encoder round-trips with Decoder"):
    case class Bar(x: Double, y: Double) derives Encoder, Decoder
    val bar = Bar(1.5, 2.5)
    val encoded = summon[Encoder[Bar]].encode(bar)
    val decoded = summon[Decoder[Bar]].decode(encoded)
    assertEquals(decoded, Right(bar))

// 4. Handle derivation failures gracefully
object SafeDerivation:
  inline def showOrFallback[T](using m: Mirror.Of[T]): Show[T] =
    try Show.derived[T]
    catch case _: Exception =>
      _ => "<not showable>"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Typeclass Derivation ใน Scala 3:

### สิ่งที่ได้เรียนรู้

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Mirror.Of | Foundation ของ generic programming ใน Scala 3 |
| derives keyword | Auto-derive instances สำหรับ case classes/sealed traits |
| summonAll | รวบรวม instances สำหรับ tuple ของ types |
| Magnolia | Library สำหรับ derivation ที่ใช้งานง่าย |
| Macro derivation | Custom derivation ด้วย compiler macros |
| Performance | Cache instances, ใช้ arrays, กำหนดใน companion objects |

### เมื่อไหรควรใช้ Derivation

- ✅ เมื่อต้องการ JSON encoding/decoding สำหรับ data models
- ✅ เมื่อต้องการ Show/Pretty-print สำหรับ debugging
- ✅ เมื่อต้องการ validation สำหรับ input data
- ✅ เมื่อต้องการ database mapping
- ❌ เมื่อต้องการ custom behavior ที่ซับซ้อน (เขียน manual instance แทน)
- ❌ เมื่อ performance critical มาก (benchmark ก่อน)

### Tips

1. กำหนด derived instances ใน companion objects เสมอ
2. Test derived instances เพื่อตรวจสอบความถูกต้อง
3. Override derived instances เมื่อต้องการ custom behavior
4. ใช้ circe/magnolia สำหรับ production code (battle-tested)
5. เข้าใจ Mirror ก่อนเขียน custom derivation

---

*[← Part 55: Reactive Streams](part-55-reactive-streams.md) | [Part 57: Effect Patterns →](part-57-effect-patterns.md)*
