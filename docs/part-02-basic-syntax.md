# Part 02: ไวยากรณ์พื้นฐาน (Basic Syntax)

## สารบัญ
1. [โครงสร้างโปรแกรม Scala](#โครงสร้างโปรแกรม-scala)
2. [Comments](#comments)
3. [Indentation-based Syntax (Scala 3)](#indentation-based-syntax)
4. [Expressions vs Statements](#expressions-vs-statements)
5. [Semicolons และ Newlines](#semicolons-และ-newlines)
6. [Identifiers และการตั้งชื่อ](#identifiers-และการตั้งชื่อ)
7. [Keywords ใน Scala](#keywords-ใน-scala)
8. [การแสดงผล (Output)](#การแสดงผล-output)
9. [String Interpolation](#string-interpolation)
10. [Packages และ Imports](#packages-และ-imports)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## โครงสร้างโปรแกรม Scala

### โปรแกรม Scala พื้นฐาน (Scala 3)

```scala
// ไฟล์: src/main/scala/Main.scala

// Package declaration (optional)
package com.example

// Import statements
import scala.collection.mutable.ArrayBuffer

// @main annotation สำหรับ entry point
@main def myProgram(): Unit =
  println("โปรแกรมเริ่มทำงาน")

  val greeting = "สวัสดี"
  val name = "Scala"

  println(s"$greeting, $name!")
```

### โครงสร้างไฟล์ Scala

ไฟล์ `.scala` สามารถประกอบด้วย:
1. **Package declaration** (บรรทัดแรก, ถ้ามี)
2. **Import statements**
3. **Definitions**: classes, objects, traits, functions, values

```scala
// 1. Package
package com.example.myapp

// 2. Imports
import scala.io.Source
import java.time.LocalDate

// 3. Definitions
case class User(name: String, email: String)

object UserService:
  def findUser(id: Int): Option[User] = ???

@main def run(): Unit =
  val user = User("Alice", "alice@example.com")
  println(user)
```

### Object declarations

```scala
// Object คือ singleton (มีแค่ instance เดียว)
object MyApp:
  val appName = "My Scala App"
  val version = "1.0.0"

  def greet(name: String): String =
    s"Hello, $name! Welcome to $appName v$version"

// เรียกใช้
@main def run(): Unit =
  println(MyApp.greet("Alice"))
  println(MyApp.appName)
```

---

## Comments

Scala มี comment 3 แบบ:

```scala
// นี่คือ single-line comment

/*
  นี่คือ
  multi-line comment
*/

/**
  * นี่คือ ScalaDoc comment
  * ใช้สำหรับสร้างเอกสาร API
  *
  * @param name ชื่อผู้ใช้
  * @return ข้อความทักทาย
  */
def greet(name: String): String = s"Hello, $name"
```

### Block Comments ซ้อนกันได้ใน Scala

```scala
/* นี่คือ outer comment
   /* และนี่คือ inner comment */
   ยังอยู่ใน outer comment
*/
```

นี่แตกต่างจาก Java ที่ block comments ซ้อนกันไม่ได้

### ScalaDoc

```scala
/**
  * คำนวณ factorial ของจำนวนเต็ม
  *
  * @param n จำนวนเต็มที่ต้องการหา factorial (ต้องมากกว่าหรือเท่ากับ 0)
  * @return factorial ของ n
  * @throws IllegalArgumentException ถ้า n < 0
  * @example
  * {{{
  * factorial(5) // returns 120
  * factorial(0) // returns 1
  * }}}
  */
def factorial(n: Int): Long =
  require(n >= 0, s"n must be >= 0, got $n")
  if n <= 1 then 1L else n * factorial(n - 1)
```

---

## Indentation-based Syntax

Scala 3 แนะนำ indentation-based syntax (เหมือน Python) ที่ไม่บังคับใช้ `{ }` เสมอไป

### แบบ Braces (ใช้ได้ทั้ง Scala 2 และ 3)

```scala
// แบบ braces - ยังใช้ได้ใน Scala 3
object BracesStyle {
  def greet(name: String): String = {
    val message = s"Hello, $name"
    message.toUpperCase()
  }

  def isAdult(age: Int): Boolean = {
    if (age >= 18) {
      true
    } else {
      false
    }
  }
}
```

### แบบ Indentation (Scala 3 แนะนำ)

```scala
// แบบ indentation - Scala 3 style (แนะนำ)
object IndentStyle:
  def greet(name: String): String =
    val message = s"Hello, $name"
    message.toUpperCase()

  def isAdult(age: Int): Boolean =
    if age >= 18 then
      true
    else
      false
```

### กฎของ Indentation

```scala
// ✅ ถูกต้อง - indent ด้วย spaces ที่สม่ำเสมอ
def example(): Unit =
  val x = 10
  val y = 20
  println(x + y)

// ✅ ถูกต้อง - body บรรทัดเดียวสามารถอยู่บรรทัดเดียวกับ def
def add(a: Int, b: Int): Int = a + b

// ✅ ถูกต้อง - ใช้ braces ก็ได้
def subtract(a: Int, b: Int): Int = {
  a - b
}

// ❌ ผิด - indent ไม่สม่ำเสมอ
def wrong(): Unit =
    val x = 10  // indent 4 spaces
  val y = 20    // indent 2 spaces - Error!
```

### if-else แบบต่างๆ

```scala
// แบบที่ 1: expression (คืนค่า)
val result = if x > 0 then "positive" else "non-positive"

// แบบที่ 2: multi-line
val message =
  if x > 100 then
    "very large"
  else if x > 0 then
    "positive"
  else if x == 0 then
    "zero"
  else
    "negative"

// แบบที่ 3: with braces (ทั้ง Scala 2 และ 3)
val value =
  if (x > 0) {
    x * 2
  } else {
    -x
  }
```

### for loops แบบต่างๆ

```scala
// แบบง่าย
for i <- 1 to 5 do
  println(i)

// แบบ multi-line body
for i <- 1 to 5 do
  val squared = i * i
  println(s"$i^2 = $squared")

// แบบ braces
for (i <- 1 to 5) {
  println(i)
}

// แบบ yield (for comprehension)
val squares = for i <- 1 to 5 yield i * i
```

### while loops

```scala
// Scala 3
var i = 0
while i < 5 do
  println(i)
  i += 1

// แบบ braces
var j = 0
while (j < 5) {
  println(j)
  j += 1
}
```

---

## Expressions vs Statements

### ใน Scala เกือบทุกอย่างเป็น Expression

Expression คือสิ่งที่มีค่า (value) ส่วน Statement คือคำสั่งที่ทำแต่ไม่คืนค่า

```scala
// Expression - มีค่า
val a = 5 + 3          // 8
val b = if true then 1 else 0  // 1
val c = { val x = 10; x * 2 } // 20

// Block เป็น expression ที่คืนค่าสุดท้าย
val result = {
  val x = 10
  val y = 20
  x + y  // นี่คือค่าที่ block คืนออกมา
}
println(result) // 30
```

```scala
// ทุก if-else ใน Scala คือ expression
val max = if a > b then a else b

// match expression
val description = a match
  case 1 => "one"
  case 2 => "two"
  case _ => "other"

// try-catch expression
val value = try
  Integer.parseInt("42")
catch
  case e: NumberFormatException => -1
```

### Unit type

`Unit` เหมือน `void` ใน Java แต่เป็น type จริง

```scala
// ฟังก์ชันที่ไม่คืนค่ามีประโยชน์ มี return type Unit
def printHello(): Unit =
  println("Hello")  // println คืน Unit

// Unit value คือ ()
val nothing: Unit = ()
println(nothing)  // ()

// สิ่งที่ไม่มี return type ที่ชัดเจน = Unit
def doSomething() =
  println("done")  // return type = Unit
```

---

## Semicolons และ Newlines

### ไม่ต้องใส่ Semicolon

```scala
// ✅ ไม่ต้องใส่ ;
val x = 10
val y = 20
println(x + y)

// ✅ ใส่ ; ก็ได้ (แต่ไม่จำเป็น)
val a = 1; val b = 2; println(a + b)

// ✅ หลาย statements บรรทัดเดียวต้องมี ;
val p = 1; val q = 2
```

### Line Continuation

```scala
// Scala รู้ว่าบรรทัดยังไม่จบถ้าจบด้วย operator
val result = 1 +
             2 +
             3  // = 6

// หรือถ้าเปิด ( หรือ [ แล้วยังไม่ปิด
val list = List(
  1, 2, 3,
  4, 5, 6
)

// Method chaining
val transformed = List(1, 2, 3, 4, 5)
  .filter(_ > 2)
  .map(_ * 2)
  .sum
```

---

## Identifiers และการตั้งชื่อ

### กฎการตั้งชื่อ

```scala
// ✅ ถูกต้อง - Alphanumeric identifiers
val myVariable = 10
val user_name = "Alice"  // snake_case (ไม่แนะนำใน Scala)
val userName = "Alice"   // camelCase (แนะนำ)
val MAX_SIZE = 100       // SCREAMING_SNAKE_CASE (constants)

// ✅ ถูกต้อง - สามารถขึ้นต้นด้วย _ หรือ letter
val _private = "hidden"
val `class` = "reserved word as identifier"  // backtick

// ❌ ผิด - ขึ้นต้นด้วยตัวเลข
val 1abc = 10  // Error!
```

### Naming Conventions ใน Scala

| สิ่ง | Convention | ตัวอย่าง |
|------|-----------|---------|
| Variables/Values | camelCase | `userName`, `maxAge` |
| Functions/Methods | camelCase | `calculateTotal()`, `getUserById()` |
| Classes/Traits/Objects | PascalCase | `UserService`, `HttpClient` |
| Constants | UPPER_SNAKE_CASE | `MAX_SIZE`, `DEFAULT_TIMEOUT` |
| Packages | lowercase | `com.example.myapp` |
| Type Parameters | UpperCase letter | `T`, `A`, `B`, `K`, `V` |

```scala
// ตัวอย่าง naming conventions
class UserRepository:
  val DEFAULT_PAGE_SIZE = 20

  def findById(userId: Int): Option[User] = ???
  def findByName(userName: String): List[User] = ???
  def saveUser(user: User): User = ???

object EmailValidator:
  def isValid(email: String): Boolean = ???

trait Serializable[A]:
  def serialize(value: A): String
  def deserialize(json: String): A
```

### Operator Identifiers

Scala อนุญาตให้ใช้ symbols เป็นชื่อ method:

```scala
class Vector(val x: Double, val y: Double):
  def +(other: Vector) = Vector(x + other.x, y + other.y)
  def -(other: Vector) = Vector(x - other.x, y - other.y)
  def *(scalar: Double) = Vector(x * scalar, y * scalar)
  def unary_- = Vector(-x, -y)

  override def toString = s"Vector($x, $y)"

@main def vectorDemo(): Unit =
  val v1 = Vector(1.0, 2.0)
  val v2 = Vector(3.0, 4.0)

  println(v1 + v2)     // Vector(4.0, 6.0)
  println(v1 - v2)     // Vector(-2.0, -2.0)
  println(v1 * 2.0)    // Vector(2.0, 4.0)
  println(-v1)         // Vector(-1.0, -2.0)
```

---

## Keywords ใน Scala

### Keywords ทั้งหมด

```
abstract  case     catch    class    def
do        else     enum     export   extends
false     final    finally  for      given
if        implicit import   lazy     match
new       null     object   override package
private   protected return  sealed   super
then      this     throw   trait    true
try       type     val      var      while
with      yield
```

### Soft Keywords (Scala 3)

Soft keywords มีความหมายพิเศษเฉพาะในบางบริบท:

```
as          derives   end       extension
infix       inline    opaque    open
transparent using
```

```scala
// ตัวอย่าง soft keywords
extension (s: String)
  def shout: String = s.toUpperCase + "!"

given Ordering[String] with
  def compare(x: String, y: String): Int = x.compareTo(y)

opaque type Meters = Double

open class Base:
  def method(): Unit = println("base")
```

---

## การแสดงผล (Output)

### println, print, printf

```scala
// println - แสดงและขึ้นบรรทัดใหม่
println("Hello, World!")
println(42)
println(3.14)
println(true)
println()  // แสดงบรรทัดว่าง

// print - แสดงโดยไม่ขึ้นบรรทัดใหม่
print("Hello, ")
print("World!")
println()  // ต้องขึ้นบรรทัดเองหลังจบ

// printf - format แบบ C-style
printf("Name: %s, Age: %d%n", "Alice", 30)
printf("Pi = %.4f%n", 3.14159265)
```

### f-strings (Scala's String Formatting)

```scala
val name = "Alice"
val age = 30
val score = 95.5

// f interpolator - type safe formatting
println(f"Name: $name, Age: $age%d, Score: $score%.1f%%")
// Output: Name: Alice, Age: 30, Score: 95.5%

// number formatting
println(f"$score%8.2f")    // "   95.50" (width 8, 2 decimals)
println(f"$age%-5d")       // "30   " (left aligned, width 5)
println(f"$age%05d")       // "00030" (zero padded, width 5)
```

### Console Output

```scala
import scala.Console.*

// Colored output (terminal ที่รองรับ)
println(RED + "Error message" + RESET)
println(GREEN + "Success!" + RESET)
println(YELLOW + "Warning..." + RESET)
println(BOLD + "Bold text" + RESET)
println(BLUE + "Info: " + RESET + "some info")
```

### Debug Output

```scala
// แสดง multiple values
val x = 10
val y = 20
val z = x + y

// แบบ basic
println(s"x=$x, y=$y, z=$z")

// แบบ pprint (ต้องเพิ่ม library com.lihaoyi::pprint ใน build.sbt)
// pprint.log(List(1, 2, 3))  // แสดง type และ value

// System.err สำหรับ error output
System.err.println("This is an error message")
```

---

## String Interpolation

### s Interpolator

```scala
val name = "Alice"
val age = 30

// s interpolator - แทรกค่าตัวแปร
println(s"Hello, $name!")
println(s"You are $age years old")
println(s"Next year you'll be ${age + 1}")  // expression ต้องใช้ ${}

// nested
val user = ("Alice", 30)
println(s"User: ${user._1}, Age: ${user._2}")
```

### f Interpolator (Type-safe Formatting)

```scala
val pi = 3.14159265358979
val count = 1000000

// f interpolator - format specifiers
println(f"Pi = $pi%.2f")          // Pi = 3.14
println(f"Pi = $pi%10.4f")        // Pi =     3.1416
println(f"Count = $count%,d")     // Count = 1,000,000
println(f"Count = $count%08d")    // Count = 01000000
println(f"Hex = $count%x")        // Hex = f4240
println(f"Sci = $pi%e")           // Sci = 3.141593e+00
```

### raw Interpolator

```scala
// raw - ไม่ interpret escape sequences
val path = raw"C:\Users\Alice\Documents"
println(path)  // C:\Users\Alice\Documents

// เปรียบเทียบกับ s
val s1 = s"Line1\nLine2"   // มี newline
val s2 = raw"Line1\nLine2"  // ไม่มี newline, \n เป็น literal
println(s1)
println("---")
println(s2)
```

### Custom Interpolators

```scala
// สร้าง custom interpolator
extension (sc: StringContext)
  def sql(args: Any*): String =
    val strings = sc.parts.iterator
    val expressions = args.iterator
    val buf = new StringBuilder(strings.next())
    while expressions.hasNext do
      val expr = expressions.next()
      // sanitize SQL input
      val sanitized = expr.toString.replace("'", "''")
      buf.append(s"'$sanitized'")
      buf.append(strings.next())
    buf.toString

// ใช้งาน
val tableName = "users; DROP TABLE users"
val query = sql"SELECT * FROM users WHERE name = ${tableName}"
println(query)
// SELECT * FROM users WHERE name = 'users; DROP TABLE users'
```

### Multiline Strings

```scala
// Triple-quoted strings
val multiline = """
  |Hello,
  |World!
  |This is multiline.
  """.stripMargin

println(multiline)

// กำหนด margin character
val html = """
  #<html>
  #  <body>
  #    <h1>Hello</h1>
  #  </body>
  #</html>
  """.stripMargin('#')

println(html)
```

---

## Packages และ Imports

### Package Declaration

```scala
// วิธีที่ 1: package ทั้งไฟล์
package com.example.myapp

class MyClass:
  def method(): Unit = ???

// วิธีที่ 2: package แบบ nested (Scala 2 style)
package com.example {
  package myapp {
    class MyClass2:
      def method(): Unit = ???
  }
}
```

### Import Statements

```scala
// Import class เดียว
import scala.collection.mutable.ArrayBuffer

// Import หลาย class
import scala.collection.mutable.{ArrayBuffer, HashMap, HashSet}

// Import ทั้ง package
import scala.collection.mutable.*  // Scala 3 (ใช้ * แทน _)
import scala.collection.mutable._  // Scala 2 (ยังใช้ได้ใน Scala 3)

// Import และ rename
import java.util.{Date => JavaDate}
import java.sql.{Date => SqlDate}

val jDate = JavaDate()
val sDate = SqlDate(0)

// Import และ hide
import java.util.{Date => _, *}  // import ทุกอย่างยกเว้น Date

// Import methods/values จาก object
import scala.math.{sqrt, pow, Pi}
println(sqrt(16))  // 4.0
println(pow(2, 10))  // 1024.0
println(Pi)  // 3.141592653589793
```

### Import ใน Scala 3

```scala
// Scala 3 รองรับ import anywhere
def processData(input: String): String =
  import scala.util.{Try, Success, Failure}

  Try(input.toInt) match
    case Success(n) => s"Number: $n"
    case Failure(e) => s"Error: ${e.getMessage}"
```

### Root Package

```scala
// ถ้า package ชนกัน ใช้ _root_ prefix
import _root_.java.util.Date
import _root_.scala.collection.mutable.Map
```

### Given Imports (Scala 3)

```scala
// Import given instances
import scala.math.Ordering.given
import mypackage.given  // import given ทั้งหมด

// Import given ที่เจาะจง type
import mypackage.{given Ordering[Int]}
```

---

## ตัวอย่างโปรแกรมครบ

```scala
// src/main/scala/BasicSyntaxDemo.scala
package demo

import scala.collection.mutable.ArrayBuffer

/**
 * Demo โปรแกรมแสดง basic syntax ของ Scala
 */
@main def basicSyntaxDemo(): Unit =

  // Variables
  val immutableVal = "ฉันเปลี่ยนแปลงไม่ได้"
  var mutableVar = "ฉันเปลี่ยนแปลงได้"
  mutableVar = "ฉันถูกเปลี่ยนแล้ว"

  // String interpolation
  val name = "Scala"
  val version = 3
  println(s"ยินดีต้อนรับสู่ $name $version!")
  println(f"Pi ประมาณ ${math.Pi}%.4f")

  // Block expressions
  val total = {
    val x = 10
    val y = 20
    val z = 30
    x + y + z  // ค่าที่คืนออกมา
  }
  println(s"Total = $total")

  // If-else as expression
  val score = 85
  val grade = if score >= 90 then "A"
              else if score >= 80 then "B"
              else if score >= 70 then "C"
              else if score >= 60 then "D"
              else "F"
  println(s"Score: $score, Grade: $grade")

  // For loop
  println("\nตาราง 3:")
  for i <- 1 to 10 do
    println(s"3 x $i = ${3 * i}")

  // For comprehension
  val evenSquares = for
    i <- 1 to 20
    if i % 2 == 0
  yield i * i

  println(s"\nกำลังสองของเลขคู่ 1-20: $evenSquares")

  // Collections
  val fruits = List("แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง")
  println(s"\nผลไม้: ${fruits.mkString(", ")}")

  val sortedFruits = fruits.sortBy(_.length)
  println(s"เรียงตามความยาว: ${sortedFruits.mkString(", ")}")

  // Pattern matching preview
  val x: Any = 42
  val description = x match
    case i: Int if i > 0 => s"จำนวนเต็มบวก: $i"
    case s: String       => s"String: $s"
    case _               => "อื่นๆ"
  println(s"\n$description")

  println("\nโปรแกรมทำงานเสร็จสิ้น!")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: String Interpolation

เขียนโปรแกรมที่รับข้อมูลและแสดงผลด้วย string interpolation:

```scala
@main def exercise1(): Unit =
  // TODO: กำหนดตัวแปรต่อไปนี้
  val firstName = "???"   // ชื่อจริงของคุณ
  val lastName = "???"    // นามสกุลของคุณ
  val birthYear = 0       // ปีเกิด
  val currentYear = 2024

  // TODO: แสดงผลต่อไปนี้:
  // 1. "ชื่อเต็ม: [ชื่อจริง] [นามสกุล]"
  // 2. "อายุโดยประมาณ: [อายุ] ปี"
  // 3. "อีก [จำนวนปี] ปีจะอายุ 100 ปี"
```

### แบบฝึกหัดที่ 2: Expressions

แก้ไขโค้ดต่อไปนี้ให้ถูกต้อง:

```scala
@main def exercise2(): Unit =
  // ผิด: if ไม่มี else สำหรับ expression
  // val result = if x > 0 then "positive"
  // แก้ไข: เพิ่ม else

  val x = 5
  val result = ??? // แก้ไขให้ถูกต้อง

  // ผิด: block ไม่คืนค่า
  // val computed = {
  //   val a = 10
  //   val b = 20
  //   println(a + b)  // Unit แทนที่จะเป็น Int
  // }
  // แก้ไข: ให้ block คืนค่า

  val computed = ??? // แก้ไขให้ถูกต้อง
```

### แบบฝึกหัดที่ 3: Multiline String

สร้างโปรแกรมที่แสดง ASCII art เพชร:

```
   *
  ***
 *****
*******
 *****
  ***
   *
```

```scala
@main def diamondArt(): Unit =
  val n = 4  // ขนาดครึ่งบน

  // TODO: ใช้ for loop สร้าง diamond
  // Hint: ใช้ " " * spaces + "*" * stars

  // แนวทาง:
  // ครึ่งบน: i จาก 1 ถึง n, spaces = n-i, stars = 2*i-1
  // ครึ่งล่าง: i จาก n-1 ถึง 1, spaces = n-i, stars = 2*i-1
```

### แบบฝึกหัดที่ 4: Imports

เขียนโปรแกรมที่ใช้ `import` เพื่อ:
1. คำนวณ sqrt, log, sin, cos ของตัวเลข
2. แสดงวันที่ปัจจุบัน
3. สร้าง random number

```scala
// TODO: import ที่จำเป็น

@main def importExercise(): Unit =
  // 1. Math functions
  val number = 144.0
  // println(s"sqrt($number) = ${sqrt(number)}")

  // 2. Current date
  // println(s"วันนี้คือ: ${LocalDate.now()}")

  // 3. Random number
  // val rand = new Random()
  // println(s"Random 1-100: ${rand.nextInt(100) + 1}")
```

**เฉลย แบบฝึกหัดที่ 3:**

```scala
@main def diamondArt(): Unit =
  val n = 4

  // ครึ่งบน
  for i <- 1 to n do
    val spaces = " " * (n - i)
    val stars = "*" * (2 * i - 1)
    println(spaces + stars)

  // ครึ่งล่าง
  for i <- (1 until n).reverse do
    val spaces = " " * (n - i)
    val stars = "*" * (2 * i - 1)
    println(spaces + stars)
```

**เฉลย แบบฝึกหัดที่ 4:**

```scala
import scala.math.{sqrt, log, sin, cos, Pi}
import java.time.LocalDate
import scala.util.Random

@main def importExercise(): Unit =
  val number = 144.0
  println(s"sqrt($number) = ${sqrt(number)}")
  println(f"log($number) = ${log(number)}%.4f")
  println(f"sin(Pi/2) = ${sin(Pi/2)}%.4f")
  println(f"cos(Pi) = ${cos(Pi)}%.4f")

  println(s"\nวันนี้คือ: ${LocalDate.now()}")

  val rand = new Random()
  val randomNum = rand.nextInt(100) + 1
  println(s"Random 1-100: $randomNum")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ โครงสร้างโปรแกรม Scala พื้นฐาน
- ✅ การเขียน Comments ทั้ง 3 แบบ
- ✅ Indentation-based syntax ของ Scala 3
- ✅ ความแตกต่างระหว่าง Expressions และ Statements
- ✅ Semicolons และการขึ้นบรรทัดใหม่
- ✅ กฎการตั้งชื่อ Identifiers
- ✅ Keywords ทั้งหมดใน Scala
- ✅ การแสดงผลด้วย println, print, printf
- ✅ String Interpolation (s, f, raw)
- ✅ Packages และ Imports

## ขั้นตอนถัดไป

ใน [Part 03: ประเภทข้อมูล](part-03-data-types.md) เราจะเรียนรู้:
- Numeric types (Int, Long, Double, Float, BigDecimal)
- Boolean
- Char และ String
- Unit และ Nothing
- Type inference
- Type casting และ conversion

---

*[← Part 01: แนะนำ Scala](part-01-introduction.md) | [Part 03: ประเภทข้อมูล →](part-03-data-types.md)*
