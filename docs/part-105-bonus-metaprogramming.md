# ส่วนที่ 105 (BONUS): Metaprogramming ใน Scala 3

> **BONUS CONTENT** - เนื้อหาขั้นสูงเกี่ยวกับ Metaprogramming, Macros, และ Compile-time Programming ใน Scala 3

---

## สารบัญ

1. [บทนำ: Metaprogramming คืออะไร?](#บทนำ)
2. [Inline และ Compile-time Computation](#inline)
3. [Quotes และ Splices](#quotes-และ-splices)
4. [Staging: Multi-Stage Programming](#staging)
5. [Type-safe printf](#type-safe-printf)
6. [Compile-time Checks](#compile-time-checks)
7. [Macros สำหรับ Type Class Derivation](#type-class-derivation)
8. [Mirror: Automatic Derivation](#mirror)
9. [Match Types](#match-types)
10. [Compile-time Strings](#compile-time-strings)
11. [ตัวอย่างสมบูรณ์](#ตัวอย่างสมบูรณ์)
12. [สรุป](#สรุป)

---

## บทนำ

Metaprogramming คือการเขียนโปรแกรมที่สร้างหรือจัดการโปรแกรมอื่น Scala 3 มีระบบ metaprogramming ที่ทรงพลังและ type-safe ด้วย Quotes และ Splices

### สามระดับของ Metaprogramming ใน Scala 3

```
Level 0: Runtime computation  (ปกติ)
Level 1: Compile-time macro   (inline, quotes/splices)
Level 2: Staging              (multi-stage programming)
```

### ทำไม Metaprogramming ถึงสำคัญ?

```scala
// ปัญหา: Performance - กำลังคำนวณซ้ำๆ ที่ compile time รู้อยู่แล้ว
def isPrime(n: Int): Boolean =
  n > 1 && (2 to Math.sqrt(n).toInt).forall(n % _ != 0)

// ด้วย inline: ตรวจสอบที่ compile time
inline def inlinePrime(n: Int): Boolean =
  ${ inlinePrimeImpl('n) }  // macro จะ expand ที่ compile time

// ปัญหา: Boilerplate - ต้องเขียน toString, equals ซ้ำๆ
// Metaprogramming ช่วยสร้าง code อัตโนมัติ
```

---

## Inline และ Compile-time Computation

`inline` บอก compiler ให้ expand ที่ call site

### Inline Def

```scala
// inline def: expand at call site
inline def log(msg: String): Unit =
  println(s"[${java.time.LocalTime.now()}] $msg")

// Inline if: ตัดสินที่ compile time
inline def greet(inline isEnglish: Boolean): String =
  inline if isEnglish then "Hello!" else "สวัสดี!"

// Compile-time constant
println(greet(true))   // "Hello!" - ไม่มี branch ใน bytecode
println(greet(false))  // "สวัสดี!" - ไม่มี branch ใน bytecode

// Inline match: pattern match ที่ compile time
inline def describe[T](inline x: T): String =
  inline x match
    case _: Int    => "integer"
    case _: String => "string"
    case _: Double => "double"
    case _         => "unknown"

println(describe(42))     // "integer"
println(describe("hi"))   // "string"
```

### Transparent Inline

```scala
// transparent inline: return type ถูก infer จาก implementation
transparent inline def toNumber(s: String): Any =
  s.toIntOption match
    case Some(n) => n
    case None    => s.toDoubleOption.getOrElse(s)

// Compiler รู้ว่า return type คือ Int หรือ Double หรือ String
val n = toNumber("42")      // type: Int
val d = toNumber("3.14")    // type: Double
val s = toNumber("hello")   // type: String
```

### Summon Inline

```scala
// summonInline: summon type class ที่ compile time
inline def showAll[T <: Tuple](t: T): String =
  inline t match
    case _: EmptyTuple => ""
    case h *: rest =>
      val shown = summonInline[cats.Show[h.type]].show(h)
      shown + " " + showAll(rest)

// Compile-time error ถ้าไม่มี Show instance
showAll((1, "hello", true))
```

---

## Quotes และ Splices

Quotes และ Splices เป็นระบบ core ของ macros ใน Scala 3

### นิยาม

```
'{ expr }  = Quote:  "code as data" (lift expression to Expr[T])
${ expr }  = Splice: "data as code" (run Expr[T] as expression)
```

### Hello World Macro

```scala
import scala.quoted.*

// Macro definition: ต้องอยู่ใน object
object Macros:
  
  // Simple macro: log with file and line info
  inline def logWithLocation(msg: String): Unit =
    ${ logWithLocationImpl('msg) }
  
  def logWithLocationImpl(msg: Expr[String])(using Quotes): Expr[Unit] =
    import quotes.reflect.*
    
    val location = Position.ofMacroExpansion
    val file = location.sourceFile.path
    val line = location.startLine + 1
    
    '{ println(s"[$file:$line] " + $msg) }

// ใช้งาน
import Macros.*

logWithLocation("Hello from macro!")
// Output: [/path/to/Main.scala:42] Hello from macro!
```

### Inspect AST

```scala
object ASTMacros:
  
  // พิมพ์ AST ของ expression
  inline def printAST[T](inline x: T): T =
    ${ printASTImpl('x) }
  
  def printASTImpl[T](x: Expr[T])(using Quotes): Expr[T] =
    import quotes.reflect.*
    
    // ดู tree structure
    println(x.asTerm.show)
    println(x.asTerm.showAnsiColored)
    
    x  // คืน expression เดิม

// ใช้งาน
val result = printAST(1 + 2 * 3)
// Prints the AST of 1 + 2 * 3
```

### Expr Pattern Matching

```scala
object ExprMacros:
  
  inline def optimize(inline n: Int): Int =
    ${ optimizeImpl('n) }
  
  def optimizeImpl(n: Expr[Int])(using Quotes): Expr[Int] =
    n match
      // Constant folding
      case Expr(x) => Expr(x)  // constant
      case '{ $a + $b } =>
        (a, b) match
          case (Expr(x), Expr(y)) => Expr(x + y)
          case (Expr(0), b)       => b
          case (a, Expr(0))       => a
          case _                  => '{ $a + $b }
      case '{ $a * $b } =>
        (a, b) match
          case (Expr(x), Expr(y)) => Expr(x * y)
          case (Expr(0), _)       => Expr(0)
          case (_, Expr(0))       => Expr(0)
          case (Expr(1), b)       => b
          case (a, Expr(1))       => a
          case _                  => '{ $a * $b }
      case _ => n

// ใช้งาน
val x = optimize(2 + 3)     // ได้ 5 ที่ compile time
val y = optimize(0 * 1000)  // ได้ 0 ที่ compile time
```

---

## Staging: Multi-Stage Programming

Multi-Stage Programming (MSP) คือการแยก computation เป็นหลาย stages

### Stage 0 และ Stage 1

```scala
import scala.quoted.*

// Stage 0: รู้ที่ compile time
// Stage 1: รู้ที่ runtime (ใน Quote)

// ตัวอย่าง: Power function ที่ optimize ตาม exponent
object Power:
  
  inline def power(base: Double, inline exp: Int): Double =
    ${ powerImpl('base, exp) }
  
  def powerImpl(base: Expr[Double], exp: Int)(using Quotes): Expr[Double] =
    if exp == 0 then '{ 1.0 }
    else if exp == 1 then base
    else if exp % 2 == 0 then
      '{ 
        val half = ${ powerImpl(base, exp / 2) }
        half * half
      }
    else
      '{
        val half = ${ powerImpl(base, exp / 2) }
        half * half * $base
      }

// power(x, 8) จะ expand เป็น:
// val h1 = x * x       (x²)
// val h2 = h1 * h1     (x⁴)
// val h3 = h2 * h2     (x⁸)
// h3

// test
val result = Power.power(2.0, 8)  // 256.0 - optimized at compile time
```

### Staged Computation

```scala
// ตัวอย่าง: Staged dot product
object StagedDotProduct:
  
  // Stage: สร้าง specialized dot product สำหรับ vector length ที่รู้ล่วงหน้า
  inline def dotProduct(v1: Array[Double], v2: Array[Double], inline n: Int): Double =
    ${ dotProductImpl('v1, 'v2, n) }
  
  def dotProductImpl(
    v1: Expr[Array[Double]],
    v2: Expr[Array[Double]],
    n: Int
  )(using Quotes): Expr[Double] =
    // Unroll loop ที่ compile time
    val terms = (0 until n).map { i =>
      '{ $v1(${ Expr(i) }) * $v2(${ Expr(i) }) }
    }
    
    if terms.isEmpty then '{ 0.0 }
    else terms.reduce((a, b) => '{ $a + $b })
  
  // dotProduct(v1, v2, 4) expands to:
  // v1(0)*v2(0) + v1(1)*v2(1) + v1(2)*v2(2) + v1(3)*v2(3)

// ใช้งาน
val v1 = Array(1.0, 2.0, 3.0, 4.0)
val v2 = Array(5.0, 6.0, 7.0, 8.0)
val dot = StagedDotProduct.dotProduct(v1, v2, 4)  // 70.0
```

---

## Type-safe printf

ตัวอย่างคลาสสิกของ metaprogramming: printf ที่ type-safe

### Simple Printf

```scala
import scala.quoted.*

object TypeSafePrintf:
  
  // Parse format string ที่ compile time
  sealed trait FmtPart
  case class Literal(s: String) extends FmtPart
  case class FormatSpec(typ: Char) extends FmtPart
  
  def parseFmt(fmt: String): List[FmtPart] =
    val parts = collection.mutable.ListBuffer[FmtPart]()
    var i = 0
    var literal = new StringBuilder
    
    while i < fmt.length do
      if fmt(i) == '%' && i + 1 < fmt.length then
        if literal.nonEmpty then
          parts += Literal(literal.toString)
          literal.clear()
        fmt(i + 1) match
          case 'd' | 's' | 'f' | 'b' =>
            parts += FormatSpec(fmt(i + 1))
          case '%' =>
            literal += '%'
        i += 2
      else
        literal += fmt(i)
        i += 1
    
    if literal.nonEmpty then parts += Literal(literal.toString)
    parts.toList
  
  // Type ที่สอดคล้องกับ format spec
  type FmtType[C <: Char] <: Any = C match
    case 'd' => Int
    case 's' => String
    case 'f' => Double
    case 'b' => Boolean
  
  // สร้าง function type จาก format specs
  // %d %s %f => Int => String => Double => String
  
  inline def printf(inline fmt: String): Any =
    ${ printfImpl(fmt) }
  
  def printfImpl(fmt: String)(using Quotes): Expr[Any] =
    import quotes.reflect.*
    
    val parts = parseFmt(fmt)
    val specs = parts.collect { case FormatSpec(c) => c }
    
    if specs.isEmpty then
      val literal = parts.collect { case Literal(s) => s }.mkString
      Expr(literal)
    else
      // Build curried function
      buildPrintf(parts, specs)
  
  def buildPrintf(parts: List[FmtPart], specs: List[Char])(using Quotes): Expr[Any] =
    // Simplified implementation
    specs match
      case Nil =>
        val literal = parts.collect { case Literal(s) => s }.mkString
        Expr(literal)
      case 'd' :: rest =>
        '{ (n: Int) => ${ buildRestPrintf(parts, rest, Map('d' -> '{ n })) } }
      case 's' :: rest =>
        '{ (s: String) => ${ buildRestPrintf(parts, rest, Map('s' -> '{ s })) } }
      case _ =>
        '{ "unsupported" }
  
  def buildRestPrintf(
    parts: List[FmtPart],
    remaining: List[Char],
    args: Map[Char, Expr[Any]]
  )(using Quotes): Expr[String] = ???  // simplified
```

### ตัวอย่างจริง: printf ด้วย HList

```scala
// Type-level HList สำหรับ arguments
type HList = Any  // simplified

// Printf ที่ type-safe ด้วยแนวทางต่างกัน
object SafePrintf:
  
  // ใช้ type classes แทน macros
  sealed trait Format[A]:
    def apply(a: A, fmt: String): String
  
  given Format[Int] with
    def apply(n: Int, fmt: String): String = fmt match
      case "%d" => n.toString
      case _    => throw new IllegalArgumentException(s"Invalid format $fmt for Int")
  
  given Format[String] with
    def apply(s: String, fmt: String): String = fmt match
      case "%s" => s
      case _    => throw new IllegalArgumentException(s"Invalid format $fmt for String")
  
  given Format[Double] with
    def apply(d: Double, fmt: String): String = fmt match
      case f if f.matches("%\\.\\d+f") =>
        val decimals = f.substring(2, f.length - 1).toInt
        s"%.${decimals}f".format(d)
      case "%f" => d.toString
      case _    => throw new IllegalArgumentException(s"Invalid format $fmt for Double")
  
  // Type-safe print function
  def format[A: Format](fmt: String)(a: A): String =
    summon[Format[A]].apply(a, fmt)

// ใช้งาน
println(SafePrintf.format[Int]("%d")(42))        // 42
println(SafePrintf.format[String]("%s")("hi"))   // hi
println(SafePrintf.format[Double]("%.2f")(3.14)) // 3.14
```

---

## Compile-time Checks

Macros สำหรับ validation ที่ compile time

### Regex ที่ Validate ที่ Compile Time

```scala
import scala.quoted.*
import scala.util.matching.Regex

object RegexMacro:
  
  // Compile-time regex validation
  inline def r(inline pattern: String): Regex =
    ${ rImpl(pattern) }
  
  def rImpl(pattern: String)(using Quotes): Expr[Regex] =
    import quotes.reflect.*
    
    try
      new Regex(pattern)  // validate pattern
      '{ new scala.util.matching.Regex(${ Expr(pattern) }) }
    catch
      case e: java.util.regex.PatternSyntaxException =>
        report.error(s"Invalid regex: ${e.getMessage}")
        '{ ??? }

// ใช้งาน
val emailRegex = r("""^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$""")
// r("[invalid") // compile error!
```

### SQL ที่ Validate ที่ Compile Time

```scala
object SQLMacro:
  
  // Parse และ validate SQL ที่ compile time
  inline def sql(inline query: String): String =
    ${ sqlImpl(query) }
  
  def sqlImpl(query: String)(using Quotes): Expr[String] =
    import quotes.reflect.*
    
    // Validate SQL syntax (simplified)
    val upperQuery = query.trim.toUpperCase
    if !upperQuery.startsWith("SELECT") &&
       !upperQuery.startsWith("INSERT") &&
       !upperQuery.startsWith("UPDATE") &&
       !upperQuery.startsWith("DELETE") then
      report.error(s"Query must start with SELECT/INSERT/UPDATE/DELETE: $query")
    
    // Check for common SQL injection patterns
    val dangerousPatterns = List("--", ";--", "/*", "*/", "xp_")
    dangerousPatterns.foreach { p =>
      if upperQuery.contains(p) then
        report.warning(s"Potentially dangerous SQL pattern: $p")
    }
    
    Expr(query)

// ใช้งาน
val query = sql("SELECT * FROM users WHERE id = ?")
// val bad = sql("NOT_SQL")  // compile error!
```

### Positive Number ที่ Compile Time

```scala
object PositiveMacro:
  
  inline def positive(inline n: Int): Int =
    ${ positiveImpl('n) }
  
  def positiveImpl(n: Expr[Int])(using Quotes): Expr[Int] =
    import quotes.reflect.*
    
    n.value match
      case Some(v) if v <= 0 =>
        report.error(s"Expected positive number, got $v")
        n
      case Some(_) => n
      case None =>
        // ไม่รู้ค่าที่ compile time - ต้องตรวจสอบที่ runtime
        '{
          val v = $n
          require(v > 0, s"Expected positive number, got $v")
          v
        }

// ใช้งาน  
val p = positive(42)   // OK
// val bad = positive(-1)  // compile error: Expected positive number, got -1

// Unknown at compile time: adds runtime check
def posFromInput(n: Int) = positive(n)
```

---

## Type Class Derivation

Macros สำหรับ auto-derive type class instances

### Show Derivation

```scala
import scala.quoted.*
import scala.deriving.*

// Show type class
trait Show[A]:
  def show(a: A): String

object Show:
  
  inline def derived[A](using m: Mirror.Of[A]): Show[A] =
    inline m match
      case s: Mirror.SumOf[A]     => derivedSum[A](using s)
      case p: Mirror.ProductOf[A] => derivedProduct[A](using p)
  
  inline def derivedProduct[A](using m: Mirror.ProductOf[A]): Show[A] =
    new Show[A]:
      def show(a: A): String =
        val name = getTypeName[A]
        val elems = getProductElements[A, m.MirroredElemTypes](a)
        s"$name(${elems.mkString(", ")})"
  
  inline def derivedSum[A](using m: Mirror.SumOf[A]): Show[A] =
    new Show[A]:
      def show(a: A): String =
        getSumVariant[A, m.MirroredElemTypes](a)
  
  // Helper macros
  inline def getTypeName[A]: String =
    ${ getTypeNameImpl[A] }
  
  def getTypeNameImpl[A](using Quotes, Type[A]): Expr[String] =
    import quotes.reflect.*
    Expr(TypeRepr.of[A].typeSymbol.name)
  
  inline def getProductElements[A, T <: Tuple](a: A): List[String] =
    ${ getProductElementsImpl[A, T]('a) }
  
  def getProductElementsImpl[A, T <: Tuple](
    a: Expr[A]
  )(using Quotes, Type[A], Type[T]): Expr[List[String]] =
    import quotes.reflect.*
    val tpe = TypeRepr.of[T]
    '{ List("...") }  // simplified
  
  inline def getSumVariant[A, T <: Tuple](a: A): String = "..."  // simplified

// ใช้งาน
case class Person(name: String, age: Int) derives Show

val alice = Person("Alice", 30)
println(summon[Show[Person]].show(alice))  // Person(Alice, 30)
```

---

## Mirror: Automatic Derivation

`Mirror` เป็น Scala 3 built-in สำหรับ automatic type class derivation

### Mirror Types

```scala
// Mirror.ProductOf[T]: สำหรับ case classes
// Mirror.SumOf[T]: สำหรับ sealed traits
// Mirror.Singleton[T]: สำหรับ case objects

import scala.deriving.*

// ดู structure ของ type
case class Point(x: Int, y: Int)

// Mirror สำหรับ Point
summon[Mirror.ProductOf[Point]] // gives us Mirror.Product
```

### JSON Encoder Derivation

```scala
import scala.quoted.*
import scala.deriving.*

// Simple JSON ADT
sealed trait Json
case class JNull() extends Json
case class JBool(b: Boolean) extends Json
case class JNum(n: Double) extends Json
case class JStr(s: String) extends Json
case class JArr(elems: List[Json]) extends Json
case class JObj(fields: List[(String, Json)]) extends Json

// JsonEncoder type class
trait JsonEncoder[A]:
  def encode(a: A): Json

object JsonEncoder:
  
  // Primitive instances
  given JsonEncoder[Boolean] with
    def encode(b: Boolean): Json = JBool(b)
  
  given JsonEncoder[Int] with
    def encode(n: Int): Json = JNum(n.toDouble)
  
  given JsonEncoder[Double] with
    def encode(d: Double): Json = JNum(d)
  
  given JsonEncoder[String] with
    def encode(s: String): Json = JStr(s)
  
  given [A: JsonEncoder]: JsonEncoder[Option[A]] with
    def encode(opt: Option[A]): Json = opt match
      case None    => JNull()
      case Some(a) => summon[JsonEncoder[A]].encode(a)
  
  given [A: JsonEncoder]: JsonEncoder[List[A]] with
    def encode(list: List[A]): Json =
      JArr(list.map(summon[JsonEncoder[A]].encode))
  
  // Derived instances using Mirror
  inline def derived[A](using m: Mirror.Of[A]): JsonEncoder[A] =
    inline m match
      case s: Mirror.SumOf[A]     => derivedSum[A](s)
      case p: Mirror.ProductOf[A] => derivedProduct[A](p)
  
  def derivedProduct[A](m: Mirror.ProductOf[A])(using
    elemLabels: ValueOf[Tuple.Map[m.MirroredElemLabels, [x] =>> x]]
  ): JsonEncoder[A] =
    new JsonEncoder[A]:
      def encode(a: A): Json =
        val labels = elemLabels.value.toList.asInstanceOf[List[String]]
        val values = a.asInstanceOf[Product].productIterator.toList
        // simplified: ต้องการ encoder สำหรับแต่ละ element
        JObj(labels.zip(values).map((k, v) => k -> JStr(v.toString)))
  
  def derivedSum[A](m: Mirror.SumOf[A]): JsonEncoder[A] =
    new JsonEncoder[A]:
      def encode(a: A): Json =
        val label = a.getClass.getSimpleName
        JObj(List("type" -> JStr(label)))  // simplified

// ใช้งาน derives
case class Address(street: String, city: String) derives JsonEncoder
case class User(name: String, age: Int, address: Address) derives JsonEncoder

def toJsonString(json: Json): String = json match
  case JNull()        => "null"
  case JBool(b)       => b.toString
  case JNum(n)        => if n == n.toLong then n.toLong.toString else n.toString
  case JStr(s)        => s""""$s""""
  case JArr(elems)    => elems.map(toJsonString).mkString("[", ",", "]")
  case JObj(fields)   => fields.map((k,v) => s""""$k":${toJsonString(v)}""").mkString("{", ",", "}")

val user = User("Alice", 30, Address("123 Main", "Bangkok"))
println(toJsonString(summon[JsonEncoder[User]].encode(user)))
```

---

## Match Types

Match Types คือ type-level computation ด้วย pattern matching

### นิยามและการใช้งาน

```scala
// Match Type: คำนวณ type ด้วย pattern matching
type Head[X <: Tuple] = X match
  case h *: _ => h
  case _      => Nothing

type Tail[X <: Tuple] = X match
  case _ *: t => t
  case _      => EmptyTuple

// ทดสอบ
type H = Head[(Int, String, Boolean)]  // Int
type T = Tail[(Int, String, Boolean)]  // (String, Boolean)

// ใช้งาน
def head[T <: NonEmptyTuple](t: T): Head[T] = t.head.asInstanceOf[Head[T]]
def tail[T <: NonEmptyTuple](t: T): Tail[T] = t.tail.asInstanceOf[Tail[T]]

val tup = (1, "hello", true)
val h = head(tup)  // 1 : Int
val t = tail(tup)  // ("hello", true) : (String, Boolean)
```

### Complex Match Types

```scala
// Flatten nested tuples
type Flatten[T <: Tuple] <: Tuple = T match
  case EmptyTuple => EmptyTuple
  case (h *: t) *: rest => h *: Flatten[t *: rest]
  case h *: t => h *: Flatten[t]

// Map type function over tuple
type Map[T <: Tuple, F[_]] <: Tuple = T match
  case EmptyTuple => EmptyTuple
  case h *: t     => F[h] *: Map[t, F]

type Optionalized = Map[(Int, String, Boolean), Option]
// (Option[Int], Option[String], Option[Boolean])

// Element type at index
type Elem[T <: Tuple, N <: Int] = T match
  case h *: _ => N match
    case 0 => h
    case S[n] => Elem[T, n]
  case _ => Nothing

// Type-level length
type Length[T <: Tuple] <: Int = T match
  case EmptyTuple => 0
  case _ *: t     => S[Length[t]]  // S = Successor type
```

### Recursive Match Types

```scala
// Type-level Nat ด้วย Match Types
type Nat = 0 | S[_]

// type-level addition
type Add[A <: Int, B <: Int] <: Int = A match
  case 0      => B
  case S[a]   => S[Add[a, B]]

// ใช้ใน function
def repeatN[A](a: A, n: Int): List[A] = 
  List.fill(n)(a)
```

---

## Compile-time Strings

การ manipulate strings ที่ compile time

### String Context และ Macros

```scala
import scala.quoted.*

// Custom string interpolator ที่ validate ที่ compile time
extension (sc: StringContext)
  inline def url(args: Any*): java.net.URL =
    ${ urlImpl('sc, 'args) }

def urlImpl(sc: Expr[StringContext], args: Expr[Seq[Any]])(using Quotes): Expr[java.net.URL] =
  import quotes.reflect.*
  
  // Get the literal parts
  val parts = sc match
    case '{ StringContext(${ Varargs(parts) }*) } =>
      parts.collect { case Expr(s: String) => s }
    case _ =>
      report.error("Cannot extract string parts")
      return '{ java.net.URI("").toURL() }
  
  // Reconstruct URL template
  val urlTemplate = parts.mkString("{arg}")
  
  // Basic URL validation
  if !urlTemplate.startsWith("http://") && !urlTemplate.startsWith("https://") then
    report.error(s"URL must start with http:// or https://: $urlTemplate")
  
  '{ java.net.URI(${ Expr(urlTemplate) }).toURL() }

// ใช้งาน
// val apiUrl = url"https://api.example.com/users"  // OK
// val badUrl = url"ftp://bad.url"  // compile error!
```

### SQL Builder ที่ Type-safe

```scala
// Type-safe SQL builder ด้วย macros
object TypeSafeSQL:
  
  // Phantom types สำหรับ query state
  sealed trait QueryState
  sealed trait NoTable extends QueryState
  sealed trait HasTable extends QueryState
  sealed trait HasWhere extends QueryState
  
  // Query builder
  case class Query[S <: QueryState](parts: List[String]):
    def toSQL: String = parts.mkString(" ")
  
  def select(cols: String*): Query[NoTable] =
    Query(List(s"SELECT ${cols.mkString(", ")}"))
  
  extension [S <: NoTable](q: Query[S])
    def from(table: String): Query[HasTable] =
      Query(q.parts :+ s"FROM $table")
  
  extension [S <: HasTable](q: Query[S])
    def where(condition: String): Query[HasWhere] =
      Query(q.parts :+ s"WHERE $condition")
  
  extension [S <: HasTable | HasWhere](q: Query[S])
    def limit(n: Int): Query[S] =
      Query(q.parts :+ s"LIMIT $n")

// ใช้งาน - type system บังคับ order
val q = TypeSafeSQL.select("id", "name")
        .from("users")
        .where("age > 18")
        .limit(10)
        .toSQL

println(q)  // SELECT id, name FROM users WHERE age > 18 LIMIT 10

// compile error ถ้า order ผิด:
// TypeSafeSQL.select("id").where("age > 18")  // ERROR: cannot call where before from
```

---

## ตัวอย่างสมบูรณ์

### Type-safe Configuration System

```scala
import scala.quoted.*

// Configuration system ที่ validate ที่ compile time
object Config:
  
  // Type class สำหรับ config values
  trait ConfigReader[A]:
    def read(key: String, env: Map[String, String]): Either[String, A]
  
  given ConfigReader[String] with
    def read(key: String, env: Map[String, String]): Either[String, String] =
      env.get(key).toRight(s"Missing config key: $key")
  
  given ConfigReader[Int] with
    def read(key: String, env: Map[String, String]): Either[String, Int] =
      env.get(key).toRight(s"Missing config key: $key")
        .flatMap(s => s.toIntOption.toRight(s"'$s' is not an Int for key: $key"))
  
  given ConfigReader[Boolean] with
    def read(key: String, env: Map[String, String]): Either[String, Boolean] =
      env.get(key).toRight(s"Missing config key: $key")
        .flatMap {
          case "true" | "1" | "yes" => Right(true)
          case "false" | "0" | "no" => Right(false)
          case s => Left(s"'$s' is not a Boolean for key: $key")
        }
  
  // Macro สำหรับ validate config keys ที่ compile time
  inline def require[A: ConfigReader](inline key: String): Either[String, A] =
    ${ requireImpl[A](key) }
  
  def requireImpl[A: Type](key: String)(using Quotes): Expr[Either[String, A]] =
    import quotes.reflect.*
    
    // ตรวจสอบว่า key ไม่ว่าง
    if key.isBlank then
      report.error("Config key cannot be blank")
    
    // ตรวจสอบ naming convention
    if key.contains(" ") then
      report.error(s"Config key '$key' should not contain spaces, use '_' or '.' instead")
    
    '{ summon[Config.ConfigReader[A]].read(${ Expr(key) }, sys.env) }

// ใช้งาน
object AppConfig:
  val dbHost: Either[String, String] = Config.require[String]("DB_HOST")
  val dbPort: Either[String, Int] = Config.require[Int]("DB_PORT")
  val debug: Either[String, Boolean] = Config.require[Boolean]("DEBUG")
  
  // Compile error:
  // val bad = Config.require[String]("has spaces")  // ERROR
  // val empty = Config.require[String]("")           // ERROR

// Load all config
def loadConfig: Either[List[String], (String, Int, Boolean)] =
  import cats.syntax.either.*
  
  val results = List(
    AppConfig.dbHost.leftMap(List(_)),
    AppConfig.dbPort.leftMap(List(_)),
    AppConfig.debug.leftMap(List(_))
  )
  
  results.sequence.map {
    case List(host: String, port: Int, debug: Boolean) => (host, port, debug)
    case _ => throw new Exception("Unreachable")
  }
```

### Code Generator

```scala
import scala.quoted.*

// Macro ที่ generate boilerplate code
object Codegen:
  
  // Generate toString สำหรับ case class
  inline def genToString[A]: String => A => String = macroImpl
  
  def macroImpl[A: Type](using Quotes): Expr[String => A => String] =
    import quotes.reflect.*
    
    val sym = TypeRepr.of[A].typeSymbol
    val fields = sym.primaryConstructor.paramSymss.flatten.filter(_.isTerm)
    val className = sym.name
    
    '{ prefix => a =>
      val fieldValues = ${ 
        Expr.ofList(fields.map { field =>
          '{ s"${${ Expr(field.name) }}=${${
            Select('{a}.asTerm, field).asExprOf[Any]
          }}" }
        })
      }
      s"${ ${ Expr(className) } }(${ fieldValues.mkString(", ") })"
    }
```

### Performance Benchmark Macro

```scala
import scala.quoted.*

// Macro สำหรับ micro-benchmarking
object Bench:
  
  inline def time[A](inline name: String)(inline block: A): A =
    ${ timeImpl(name, 'block) }
  
  def timeImpl[A: Type](name: String, block: Expr[A])(using Quotes): Expr[A] =
    '{
      val start = System.nanoTime()
      val result = $block
      val elapsed = System.nanoTime() - start
      println(s"[${ ${ Expr(name) } }] ${elapsed / 1_000_000.0}ms")
      result
    }
  
  // Warm-up + benchmark
  inline def benchmark[A](inline name: String, inline iterations: Int)(inline block: A): Unit =
    ${ benchmarkImpl(name, iterations, 'block) }
  
  def benchmarkImpl[A: Type](
    name: String,
    iterations: Int,
    block: Expr[A]
  )(using Quotes): Expr[Unit] =
    '{
      // Warm up
      for _ <- 1 to 5 do $block
      
      // Measure
      val start = System.nanoTime()
      for _ <- 1 to ${ Expr(iterations) } do $block
      val elapsed = System.nanoTime() - start
      
      val avg = elapsed.toDouble / ${ Expr(iterations) } / 1_000.0
      println(s"[${ ${ Expr(name) } }] avg: ${avg}μs (${${ Expr(iterations) }} iterations)")
    }

// ใช้งาน
val result = Bench.time("sort 1000 elements"):
  (1 to 1000).toList.sorted

Bench.benchmark("fibonacci", 100000):
  def fib(n: Int): Long = if n <= 1 then n else fib(n-1) + fib(n-2)
  fib(30)
```

---

## สรุป

### Metaprogramming Tools ใน Scala 3

| Tool | Level | Use Case |
|------|-------|---------|
| `inline def` | Source | Constant folding, inlining |
| `inline if/match` | Source | Compile-time branching |
| `Quotes/Splices` | Macro | AST manipulation |
| `Mirror` | Type | Automatic derivation |
| `Match Types` | Type | Type-level computation |
| `transparent inline` | Source | Return type refinement |

### เมื่อไหร่ใช้ Metaprogramming

```
Use inline when:
  ✓ ต้องการ constant folding
  ✓ ต้องการ optimize hot paths
  ✓ Type class dispatch ที่ compile time

Use Macros when:
  ✓ ต้องการ compile-time validation
  ✓ Code generation จาก type structure
  ✓ DSL ที่ type-safe
  ✓ Performance-critical code

Use Mirror when:
  ✓ Auto-derive type class instances
  ✓ Generic programming
  ✓ Serialization/Deserialization

Avoid macros when:
  ✗ Logic สามารถเขียนด้วย runtime polymorphism ได้
  ✗ Team ยังไม่คุ้นเคยกับ macros
  ✗ Error messages ต้องการความชัดเจน
```

### Best Practices

1. **เริ่มจาก inline ก่อน** - ง่ายกว่า full macros
2. **ให้ error messages ดี** - ใช้ `report.error` แทน throwing
3. **Test macros อย่างละเอียด** - ทั้ง compile-time และ runtime behavior
4. **Document ให้ชัด** - macros มักเข้าใจยาก
5. **ใช้ Mirror สำหรับ derivation** - ดีกว่า custom macros ในหลายกรณี

---

*[← BONUS ส่วนที่ 104: Advanced Cats](part-104-bonus-advanced-cats.md)*
