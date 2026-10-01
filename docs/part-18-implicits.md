# Part 18: Implicits และ Given/Using (Scala 3)

## สารบัญ
1. [Given และ Using (Scala 3)](#given-และ-using-scala-3)
2. [Extension Methods](#extension-methods)
3. [Type Classes ด้วย Given](#type-classes-ด้วย-given)
4. [Context Functions](#context-functions)
5. [Implicit Conversions](#implicit-conversions)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Given และ Using (Scala 3)

### given Instances

```scala
// Scala 3: ใช้ given แทน implicit val/def
// ใช้ using แทน implicit parameter

// กำหนด type class
trait Encoder[A]:
  def encode(a: A): String

// given instances
given Encoder[Int] with
  def encode(n: Int): String = n.toString

given Encoder[String] with
  def encode(s: String): String = s""""$s""""

given Encoder[Boolean] with
  def encode(b: Boolean): String = if b then "true" else "false"

given [A: Encoder]: Encoder[List[A]] with
  def encode(list: List[A]): String =
    val enc = summon[Encoder[A]]
    list.map(enc.encode).mkString("[", ",", "]")

// using: รับ given instance
def encode[A](a: A)(using enc: Encoder[A]): String = enc.encode(a)

println(encode(42))               // 42
println(encode("hello"))          // "hello"
println(encode(true))             // true
println(encode(List(1, 2, 3)))    // [1,2,3]
```

### summon vs implicitly

```scala
// summon[T] คือ Scala 3 version ของ implicitly[T]

given ordering: Ordering[String] = Ordering.String

val ord = summon[Ordering[String]]  // Scala 3
// val ord = implicitly[Ordering[String]]  // Scala 2

// using หลาย givens
def processItems[A](items: List[A])(
  using ord: Ordering[A],
  enc: Encoder[A]
): String =
  items.sorted(ord).map(enc.encode).mkString(", ")

// Context bound syntax (shorthand)
def processItems2[A: Ordering: Encoder](items: List[A]): String =
  items.sorted.map(encode(_)).mkString(", ")

println(processItems2(List("banana", "apple", "cherry")))
// "apple", "banana", "cherry"
```

### Derived Instances

```scala
// Auto-deriving instances

given [A: Encoder, B: Encoder]: Encoder[(A, B)] with
  def encode(pair: (A, B)): String =
    s"(${encode(pair._1)},${encode(pair._2)})"

given [A: Encoder]: Encoder[Option[A]] with
  def encode(opt: Option[A]): String = opt match
    case Some(a) => s"some(${encode(a)})"
    case None    => "null"

given [L: Encoder, R: Encoder]: Encoder[Either[L, R]] with
  def encode(either: Either[L, R]): String = either match
    case Right(r) => s"ok(${encode(r)})"
    case Left(l)  => s"err(${encode(l)})"

println(encode((1, "hello")))          // (1,"hello")
println(encode(Option(42)))            // some(42)
println(encode(None: Option[Int]))     // null
println(encode(Right(42): Either[String, Int]))  // ok(42)
```

---

## Extension Methods

### Extension Method Syntax

```scala
// Extension methods: เพิ่ม methods ให้ existing types

extension (n: Int)
  def squared: Int = n * n
  def isEven: Boolean = n % 2 == 0
  def times(f: => Unit): Unit =
    var i = 0
    while i < n do
      f
      i += 1

println(5.squared)     // 25
println(4.isEven)      // true
3.times { print("! ") }
// ! ! !

// Extension กับ Generic Type
extension [A](list: List[A])
  def second: Option[A] = list.drop(1).headOption
  def penultimate: Option[A] = if list.length >= 2 then Some(list(list.length - 2)) else None
  def splitWhen(pred: A => Boolean): (List[A], List[A]) =
    list.span(!pred(_))

println(List(1, 2, 3, 4).second)       // Some(2)
println(List(1, 2, 3, 4).penultimate)  // Some(3)

// Extension กับ Type Class constraint
extension [A: Ordering](list: List[A])
  def median: Option[A] =
    if list.isEmpty then None
    else Some(list.sorted.apply(list.length / 2))

  def isSorted: Boolean =
    list.sliding(2).forall { case List(a, b) => summon[Ordering[A]].lteq(a, b); case _ => true }

println(List(3, 1, 4, 1, 5).median)    // Some(3)
println(List(1, 2, 3, 4).isSorted)     // true
println(List(4, 3, 2, 1).isSorted)     // false
```

### Extension กับ String

```scala
extension (s: String)
  def toSnakeCase: String =
    s.replaceAll("([A-Z])", "_$1").toLowerCase.dropWhile(_ == '_')

  def toCamelCase: String =
    val parts = s.split("[_\\s]+")
    parts.head + parts.tail.map(_.capitalize).mkString

  def truncate(maxLength: Int, suffix: String = "..."): String =
    if s.length <= maxLength then s
    else s.take(maxLength - suffix.length) + suffix

  def isPalindrome: Boolean =
    val clean = s.toLowerCase.filter(_.isLetterOrDigit)
    clean == clean.reverse

println("HelloWorld".toSnakeCase)           // hello_world
println("hello_world_scala".toCamelCase)    // helloWorldScala
println("Hello, World!".truncate(10))       // Hello, W...
println("racecar".isPalindrome)             // true
println("A man a plan a canal Panama".isPalindrome)  // true
```

---

## Type Classes ด้วย Given

### Monoid Type Class

```scala
trait Monoid[A]:
  def empty: A
  def combine(x: A, y: A): A

given Monoid[Int] with
  def empty: Int = 0
  def combine(x: Int, y: Int): Int = x + y

given Monoid[String] with
  def empty: String = ""
  def combine(x: String, y: String): String = x + y

given [A]: Monoid[List[A]] with
  def empty: List[A] = Nil
  def combine(x: List[A], y: List[A]): List[A] = x ++ y

given [A, B](using ma: Monoid[A], mb: Monoid[B]): Monoid[(A, B)] with
  def empty: (A, B) = (ma.empty, mb.empty)
  def combine(x: (A, B), y: (A, B)): (A, B) =
    (ma.combine(x._1, y._1), mb.combine(x._2, y._2))

// Generic fold using Monoid
def fold[A: Monoid](list: List[A]): A =
  val m = summon[Monoid[A]]
  list.foldLeft(m.empty)(m.combine)

println(fold(List(1, 2, 3, 4, 5)))           // 15
println(fold(List("a", "b", "c")))           // abc
println(fold(List(List(1,2), List(3,4))))    // List(1, 2, 3, 4)
println(fold(List((1, "a"), (2, "b"))))      // (3,ab)

// mconcat
def mconcat[A: Monoid](items: A*): A = fold(items.toList)
println(mconcat(1, 2, 3))        // 6
println(mconcat("x", "y", "z"))  // xyz
```

### Eq Type Class

```scala
trait Eq[A]:
  def eqv(x: A, y: A): Boolean
  def neqv(x: A, y: A): Boolean = !eqv(x, y)

extension [A: Eq](x: A)
  def ===(y: A): Boolean = summon[Eq[A]].eqv(x, y)
  def =!=(y: A): Boolean = summon[Eq[A]].neqv(x, y)

given Eq[Int] with
  def eqv(x: Int, y: Int): Boolean = x == y

given Eq[String] with
  def eqv(x: String, y: String): Boolean = x == y

given [A: Eq]: Eq[Option[A]] with
  def eqv(x: Option[A], y: Option[A]): Boolean = (x, y) match
    case (None, None)         => true
    case (Some(a), Some(b))   => a === b
    case _                    => false

println(1 === 1)          // true
println(1 === 2)          // false
println("hi" === "hi")    // true
println(Some(1) === Some(1))   // true
println(Some(1) === Some(2))   // false
println(None === None)         // true
```

---

## Context Functions

### Context Function Types

```scala
// Context function: A ?=> B
// ฟังก์ชันที่รับ context parameter โดยนัย

type Logged[A] = List[String] ?=> A

def withLog[A](f: List[String] ?=> A): A =
  given List[String] = List.empty
  f

def log(msg: String)(using logs: List[String]): Unit =
  println(s"LOG: $msg")

// Context Function ใช้ใน DSL
class HtmlBuilder:
  private val content = StringBuilder()

  def element(tag: String)(body: HtmlBuilder ?=> Unit): String =
    given HtmlBuilder = this
    body
    s"<$tag>${content.toString}</$tag>"

  def text(s: String)(using builder: HtmlBuilder): Unit =
    builder.content.append(s)

// Reader Monad pattern
type Reader[Env, A] = Env ?=> A

case class AppConfig(host: String, port: Int, debug: Boolean)

def getHost: Reader[AppConfig, String] = summon[AppConfig].host
def getPort: Reader[AppConfig, Int] = summon[AppConfig].port

def connectionString: Reader[AppConfig, String] =
  val host = getHost
  val port = getPort
  s"$host:$port"

given AppConfig = AppConfig("localhost", 8080, true)
println(connectionString)  // localhost:8080
```

---

## Implicit Conversions

### Scala 3 Implicit Conversions

```scala
import scala.language.implicitConversions

// Scala 3: ใช้ given Conversion[A, B]
given Conversion[String, Int] = _.length
given Conversion[Int, String] = _.toString

val n: Int = "hello"  // implicit conversion
println(n)  // 5

// Practical: RichInt-like class
class Meters(val value: Double):
  def toFeet: Feet = Feet(value * 3.28084)
  override def toString = s"${value}m"

class Feet(val value: Double):
  def toMeters: Meters = Meters(value / 3.28084)
  override def toString = s"${value}ft"

given Conversion[Double, Meters] = Meters(_)
given Conversion[Int, Meters] = n => Meters(n.toDouble)

val distance: Meters = 5.5
println(distance)         // 5.5m
println(distance.toFeet)  // 18.044620000000003ft

// Extension methods อาจดีกว่า implicit conversions
extension (d: Double)
  def meters: Meters = Meters(d)
  def feet: Feet = Feet(d)

println(10.0.meters.toFeet)  // 32.8084ft
println(100.0.feet.toMeters) // 30.48000000000000...m
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Serialization Type Class

```scala
// สร้าง JSON serialization type class

trait JsonSerializer[A]:
  def toJson(a: A): String

object JsonSerializer:
  def apply[A: JsonSerializer]: JsonSerializer[A] = summon[JsonSerializer[A]]

extension [A: JsonSerializer](a: A)
  def toJson: String = summon[JsonSerializer[A]].toJson(a)

// TODO: implement instances
given JsonSerializer[Int] with
  def toJson(n: Int): String = n.toString

given JsonSerializer[String] with
  def toJson(s: String): String = ???  // wrap in quotes, escape special chars

given JsonSerializer[Boolean] with
  def toJson(b: Boolean): String = ???

given [A: JsonSerializer]: JsonSerializer[List[A]] with
  def toJson(list: List[A]): String = ???

given [A: JsonSerializer]: JsonSerializer[Option[A]] with
  def toJson(opt: Option[A]): String = ???

// Test
// println(42.toJson)                       // 42
// println("hello \"world\"".toJson)        // "hello \"world\""
// println(List(1, 2, 3).toJson)            // [1,2,3]
// println(Some("test").toJson)             // "test"
// println((None: Option[Int]).toJson)      // null
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Given และ Using (Scala 3)
- ✅ summon และ Context Bounds
- ✅ Extension Methods
- ✅ Type Classes: Monoid, Eq
- ✅ Context Functions
- ✅ Implicit Conversions (Scala 3 style)

---

*[← Part 17: Collections Advanced](part-17-collections-advanced.md) | [Part 19: Type Classes Pattern →](part-19-type-classes.md)*
