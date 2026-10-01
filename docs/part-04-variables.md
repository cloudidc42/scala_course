# Part 04: ตัวแปรและค่าคงที่ (Variables and Values)

## สารบัญ
1. [val vs var](#val-vs-var)
2. [Lazy val](#lazy-val)
3. [Type Annotations](#type-annotations)
4. [Scope และ Visibility](#scope-และ-visibility)
5. [Immutability หลักการสำคัญ](#immutability-หลักการสำคัญ)
6. [Variable Shadowing](#variable-shadowing)
7. [Constants และ Pattern Matching](#constants-และ-pattern-matching)
8. [Destructuring Assignment](#destructuring-assignment)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## val vs var

### val: Immutable Value

```scala
// val คือค่าที่เปลี่ยนแปลงไม่ได้ (immutable)
val name = "Alice"
val age = 30
val pi = 3.14159

// ❌ ไม่สามารถ reassign
// name = "Bob"  // Error: reassignment to val

// val สร้างได้หลายแบบ
val x: Int = 10           // พร้อม type annotation
val y = 20                // type inference
val z: Double = x + y     // expression
```

### var: Mutable Variable

```scala
// var คือตัวแปรที่เปลี่ยนแปลงได้ (mutable)
var counter = 0
var message = "Hello"

// ✅ สามารถ reassign ได้
counter = counter + 1    // = 1
counter += 1             // = 2
counter -= 1             // = 1
counter *= 2             // = 2
counter /= 2             // = 1
counter %= 1             // = 0

message = "World"
message += "!"  // "World!"

// var ต้องมีค่าเริ่มต้น (ยกเว้นใน class)
var count: Int = 0     // ✅
// var empty: Int      // ❌ Error: needs to be initialized
```

### เมื่อไหร่ควรใช้ var?

```scala
// ❌ หลีกเลี่ยง var เมื่อเป็นไปได้
var sum = 0
for i <- 1 to 100 do
  sum += i
println(sum)  // 5050

// ✅ ใช้ functional approach แทน
val sum2 = (1 to 100).sum
println(sum2)  // 5050

// ❌ var ใน accumulator pattern
var result = List[Int]()
for i <- 1 to 10 do
  result = result :+ (i * i)
println(result)

// ✅ ใช้ for comprehension แทน
val result2 = for i <- 1 to 10 yield i * i
println(result2)

// กรณีที่ var จำเป็น:
// 1. Performance critical code
// 2. สถานะที่ต้องเปลี่ยนแปลง (state machine)
// 3. Loop counters ในบางกรณี
// 4. Interop กับ Java mutable APIs
```

### val กับ Mutable Collections

```scala
// val ป้องกันการ reassign แต่ไม่ป้องกัน mutation ของ mutable objects!
import scala.collection.mutable.ArrayBuffer

val list = ArrayBuffer(1, 2, 3)  // val แต่ mutable!
list.append(4)      // ✅ สามารถ mutate ได้
list += 5           // ✅
println(list)       // ArrayBuffer(1, 2, 3, 4, 5)

// list = ArrayBuffer(10, 20)  // ❌ ไม่สามารถ reassign ได้

// ถ้าต้องการ immutable จริงๆ ใช้ immutable collections
val immutableList = List(1, 2, 3)
// immutableList.append(4)  // ❌ List ไม่มี append method
val newList = immutableList :+ 4  // สร้าง list ใหม่ [1,2,3,4]
```

---

## Lazy val

```scala
// lazy val: คำนวณค่าเมื่อถูกเรียกใช้ครั้งแรก (ไม่คำนวณทันที)

// ปกติ: คำนวณทันทีเมื่อ define
val eager = {
  println("Computing eager value...")
  42
}
// Output: Computing eager value...  (ทันทีเมื่อ define)

println("Before accessing lazy")

// lazy: คำนวณตอนที่เรียกใช้ครั้งแรก
lazy val deferred = {
  println("Computing lazy value...")
  84
}
// ยังไม่มี output

println("Accessing lazy value...")
println(deferred)  // ตอนนี้จึง compute
// Output:
// Computing lazy value...
// 84

println(deferred)  // ครั้งที่ 2 ไม่ compute ใหม่ (cached)
// Output: 84  (เฉพาะค่า ไม่มี "Computing...")
```

### ประโยชน์ของ Lazy val

```scala
// 1. Expensive computation ที่อาจไม่ได้ใช้
lazy val config = loadConfigFromFile("config.json")
lazy val database = initializeDatabase()

def processIfNeeded(useDatabase: Boolean): Unit =
  if useDatabase then
    database.query("SELECT * FROM users")
  // ถ้า useDatabase = false, database ไม่ถูกสร้าง

// 2. Circular dependencies
class A:
  lazy val b: B = new B(this)

class B(val a: A):
  val greeting = "Hello from B"

// ถ้าไม่ใช้ lazy จะเกิด initialization order problem

// 3. Infinite structures
lazy val ones: LazyList[Int] = 1 #:: ones
println(ones.take(5).toList)  // List(1, 1, 1, 1, 1)

lazy val naturals: LazyList[Int] = LazyList.from(1)
println(naturals.take(10).toList)  // List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// 4. Initialization order control
object DatabaseConfig:
  val host = "localhost"
  val port = 5432
  lazy val connectionString = s"postgresql://$host:$port/mydb"
  // connectionString จะถูก evaluate หลัง host และ port
```

### lazy val Thread Safety

```scala
// lazy val ใน Scala ปลอดภัยสำหรับ multi-threading
// Scala รับประกันว่า block จะถูกรันแค่ครั้งเดียว

class ExpensiveResource:
  lazy val resource = {
    Thread.sleep(100)  // simulate expensive init
    "resource initialized"
  }

// สามารถเรียกจาก multiple threads โดยปลอดภัย
val obj = ExpensiveResource()
val threads = (1 to 10).map { _ =>
  new Thread(() => println(obj.resource))
}
threads.foreach(_.start())
threads.foreach(_.join())
// "resource initialized" จะถูก compute แค่ครั้งเดียว
```

---

## Type Annotations

### เมื่อไหรควรใส่ Type Annotation

```scala
// 1. Public API - ควรใส่เสมอ
def getUserById(id: Int): Option[User] = ???

// 2. เมื่อ inference ให้ type ที่ไม่ต้องการ
val numbers = List(1, 2, 3)        // List[Int] ✅
val mixed: List[Any] = List(1, "2", 3.0)  // ต้องระบุ

// 3. เมื่อต้องการ type ที่กว้างกว่า
val specific = 42          // Int
val general: AnyVal = 42  // AnyVal

// 4. เพื่อ documentation
val maxRetries: Int = 3  // ชัดเจนกว่า val maxRetries = 3

// 5. ป้องกัน surprises
val result: Double = 10 / 3    // 3.0 ไม่ใช่ 3.333...
val correct: Double = 10.0 / 3 // 3.3333...
```

### Type Ascription

```scala
// Type ascription: ระบุ type ให้ expression
val x = (42: Int)
val y = ("hello": String)
val z = (42: Any)  // Int แต่ view เป็น Any

// ใช้กับ polymorphic methods
def process[A](value: A): String = value.toString
process(42)          // inferred as Int
process(42: Any)     // explicitly Any

// Upcast
val s: String = "hello"
val a: Any = s: Any  // upcast to Any
```

---

## Scope และ Visibility

### Local Variables

```scala
@main def scopeDemo(): Unit =
  val x = 10  // local to this function

  def innerFunction(): Unit =
    val y = 20  // local to innerFunction
    println(x + y)  // สามารถเข้าถึง x จาก outer scope ได้
    // val z = 30

  innerFunction()
  // println(y)  // ❌ Error: y ไม่อยู่ใน scope
```

### Block Scope

```scala
@main def blockScope(): Unit =
  val result = {
    val a = 10
    val b = 20
    a + b  // ค่าที่คืนออกมา
  }
  // println(a)  // ❌ a ไม่อยู่ใน scope แล้ว
  println(result)  // 30

  // if-else blocks
  val x = 5
  val message = if x > 3 then
    val doubleX = x * 2
    s"$x is large, double is $doubleX"  // doubleX อยู่ใน scope นี้
  else
    s"$x is small"
  // println(doubleX)  // ❌ ไม่อยู่ใน scope
```

### Access Modifiers สำหรับ class members

```scala
class Person(
  val name: String,          // public (default)
  private val age: Int,      // เข้าถึงได้เฉพาะใน class
  protected val id: Int      // เข้าถึงได้ใน class และ subclass
):
  private def validate(): Boolean = age > 0
  protected def getInfo(): String = s"$name, $age"

  def greeting(): String =
    if validate() then s"Hello, I'm $name"
    else "Invalid person"

// Package-private
package myapp

class Internal:
  private[myapp] val secret = "only in myapp package"
  private[this] val exclusive = "only in this instance"
```

### Top-level Definitions (Scala 3)

```scala
// Scala 3 อนุญาตให้ define ที่ top-level ได้
// ไม่ต้องอยู่ใน object หรือ class

// ไฟล์: myapp/Utils.scala
package myapp

val DefaultTimeout = 30  // top-level val
var globalCounter = 0    // top-level var (หลีกเลี่ยงถ้าเป็นไปได้)

def formatDate(date: java.time.LocalDate): String =
  date.toString  // top-level function

case class Config(host: String, port: Int)  // top-level class
```

---

## Immutability หลักการสำคัญ

### ทำไม Immutability ถึงสำคัญ?

```scala
// 1. Thread Safety
// Immutable objects ปลอดภัยสำหรับ concurrent access

// ❌ Mutable shared state - อันตราย!
var sharedCounter = 0

import java.util.concurrent.Executors
val executor = Executors.newFixedThreadPool(10)

for _ <- 1 to 1000 do
  executor.submit(() => sharedCounter += 1)

Thread.sleep(1000)
println(sharedCounter)  // อาจไม่ใช่ 1000 เพราะ race condition!

// ✅ ใช้ immutable + functional approach
import java.util.concurrent.atomic.AtomicInteger
val atomicCounter = new AtomicInteger(0)

for _ <- 1 to 1000 do
  executor.submit(() => atomicCounter.incrementAndGet())

Thread.sleep(1000)
println(atomicCounter.get())  // 1000 เสมอ

// 2. Easier to reason about
// เมื่อ val เปลี่ยนไม่ได้ เราแน่ใจได้ว่าค่าคงที่ตลอด

val config = Map("host" -> "localhost", "port" -> "5432")
// สามารถส่ง config ไปให้ function ใดก็ได้ โดยมั่นใจว่าไม่เปลี่ยน

// 3. Pure Functions
// ฟังก์ชันที่ไม่มี side effects ทดสอบได้ง่าย

def calculateTax(income: Double, rate: Double): Double =
  income * rate  // pure function - ไม่มี side effects

// 4. Caching/Memoization ปลอดภัย
lazy val expensiveResult = veryExpensiveComputation()
// ปลอดภัยเพราะ lazy val เป็น immutable หลัง compute แล้ว
```

### Immutable Data Structures

```scala
// Scala collections เป็น immutable by default
val list = List(1, 2, 3)
val newList = list :+ 4   // สร้าง list ใหม่
println(list)     // List(1, 2, 3) - ไม่เปลี่ยน
println(newList)  // List(1, 2, 3, 4)

val map = Map("a" -> 1, "b" -> 2)
val newMap = map + ("c" -> 3)  // สร้าง map ใหม่
println(map)     // Map(a -> 1, b -> 2)
println(newMap)  // Map(a -> 1, b -> 2, c -> 3)

// Copy with modification สำหรับ case class
case class Person(name: String, age: Int, email: String)

val alice = Person("Alice", 30, "alice@example.com")
val olderAlice = alice.copy(age = 31)  // สร้าง object ใหม่
val newEmail = alice.copy(email = "newalice@example.com")

println(alice)      // Person(Alice,30,alice@example.com)
println(olderAlice) // Person(Alice,31,alice@example.com)
```

### Immutable OOP Pattern

```scala
// Builder pattern กับ immutable objects
case class ServerConfig private (
  host: String,
  port: Int,
  maxConnections: Int,
  timeout: Int
):
  def withHost(h: String) = copy(host = h)
  def withPort(p: Int) = copy(port = p)
  def withMaxConnections(mc: Int) = copy(maxConnections = mc)
  def withTimeout(t: Int) = copy(timeout = t)

object ServerConfig:
  val default = ServerConfig(
    host = "localhost",
    port = 8080,
    maxConnections = 100,
    timeout = 30
  )

// ใช้งาน
val config = ServerConfig.default
  .withHost("production.example.com")
  .withPort(443)
  .withMaxConnections(1000)

println(config)
// ServerConfig(production.example.com,443,1000,30)
```

---

## Variable Shadowing

### ตัวอย่าง Shadowing

```scala
val x = 10  // outer x

{
  val x = 20  // shadows outer x
  println(x)  // 20 (inner x)
}

println(x)  // 10 (outer x ยังอยู่)

// Shadowing ใน for loops
for x <- 1 to 5 do
  println(x)  // 1, 2, 3, 4, 5 (shadows outer x)

// Shadowing ใน pattern matching
val value = 42
value match
  case x if x > 0 =>
    val result = x * 2  // x shadows outer value in this scope
    println(result)
  case _ =>
    println("non-positive")
```

### เมื่อไหร่ Shadowing มีประโยชน์?

```scala
// 1. Pipeline transformations - ทำให้โค้ดกระชับ
val data = List(1, 2, 3, 4, 5)

val result = data
  .filter(_ > 2)      // data: List[Int]
  .map(_ * 2)         // still List[Int]
  .take(3)            // still List[Int]

// 2. Refining types
def process(input: Any): Unit =
  input match
    case input: String =>  // shadows outer 'input' with narrowed type
      println(input.toUpperCase)  // รู้ว่าเป็น String
    case input: Int =>
      println(input * 2)  // รู้ว่าเป็น Int

// 3. Avoiding long variable names
class UserProcessor:
  def process(user: User): ProcessedUser =
    val user = validateAndEnrich(user)  // shadows parameter
    // ณ จุดนี้ user เป็น enriched version
    convert(user)
```

### ระวัง: Shadowing ที่สร้างความสับสน

```scala
// ❌ อย่าทำแบบนี้ - สับสนมาก
val data = List(1, 2, 3)
val result = data.map { data =>   // data shadows outer data
  data * 2                         // data ตรงนี้คือ element ไม่ใช่ list
}

// ✅ ใช้ชื่อที่ชัดเจน
val numbers = List(1, 2, 3)
val doubled = numbers.map { n => n * 2 }
```

---

## Constants และ Pattern Matching

### Constants ใน Scala

```scala
// ใน object: uppercase names
object Constants:
  val MAX_SIZE = 100
  val DEFAULT_TIMEOUT = 30
  val APP_VERSION = "1.0.0"
  val SUPPORTED_FORMATS = Set("json", "xml", "csv")

// Val ใน class
class Config:
  final val MAX_RETRY = 3  // final ป้องกัน override

// Pattern matching กับ constants
import Constants.*

def processWithLimit(size: Int): String =
  if size > MAX_SIZE then s"Too large (max: $MAX_SIZE)"
  else s"Processing $size items"
```

### Constants ใน Pattern Matching

```scala
object HttpStatus:
  val OK = 200
  val NOT_FOUND = 404
  val SERVER_ERROR = 500

// Pattern matching กับ constants - ต้องใช้ backtick หรือ uppercase!
def handleStatus(code: Int): String =
  code match
    case HttpStatus.OK           => "Success"
    case HttpStatus.NOT_FOUND    => "Not Found"
    case HttpStatus.SERVER_ERROR => "Server Error"
    case _                        => s"Unknown: $code"

// ระวัง: lowercase val ใน pattern matching
val notFound = 404

// ❌ นี่ไม่ใช่การ match กับ notFound!
def wrong(code: Int) = code match
  case notFound => "this always matches!"  // notFound เป็น new binding!
  case _ => "never reached"

// ✅ ใช้ backtick เพื่อ reference val
def correct(code: Int) = code match
  case `notFound` => "404 Not Found"
  case _          => "other"

// ✅ หรือใช้ uppercase สำหรับ stable identifiers
val NotFound = 404
def correct2(code: Int) = code match
  case NotFound => "404 Not Found"
  case _        => "other"
```

---

## Destructuring Assignment

### Tuple Destructuring

```scala
// แกะค่าจาก Tuple
val pair = (42, "hello")
val (num, str) = pair
println(num)  // 42
println(str)  // hello

// Triple
val triple = (1, 2, 3)
val (a, b, c) = triple

// Ignore ค่าที่ไม่ต้องการด้วย _
val (first, _, third) = triple
println(first)  // 1
println(third)  // 3

// ใน for loop
val pairs = List((1, "one"), (2, "two"), (3, "three"))
for (n, s) <- pairs do
  println(s"$n = $s")
```

### Case Class Destructuring

```scala
case class Point(x: Int, y: Int)
case class Circle(center: Point, radius: Double)

val p = Point(10, 20)
val Point(x, y) = p
println(s"x=$x, y=$y")  // x=10, y=20

val circle = Circle(Point(0, 0), 5.0)
val Circle(Point(cx, cy), r) = circle
println(s"center=($cx,$cy), radius=$r")  // center=(0,0), radius=5.0

// ใน function parameters
def distance(Point(x1, y1): Point, Point(x2, y2): Point): Double =
  math.sqrt(math.pow(x2 - x1, 2) + math.pow(y2 - y1, 2))

println(distance(Point(0, 0), Point(3, 4)))  // 5.0
```

### List Destructuring

```scala
val list = List(1, 2, 3, 4, 5)

// head :: tail pattern
val head :: tail = list
println(head)  // 1
println(tail)  // List(2, 3, 4, 5)

// หลาย elements
val first :: second :: rest = list
println(first)   // 1
println(second)  // 2
println(rest)    // List(3, 4, 5)

// ใน match
def sumList(list: List[Int]): Int = list match
  case Nil          => 0
  case head :: tail => head + sumList(tail)

println(sumList(List(1, 2, 3, 4, 5)))  // 15
```

### Map Destructuring

```scala
val config = Map("host" -> "localhost", "port" -> "5432")

// ด้วย for comprehension
for (key, value) <- config do
  println(s"$key = $value")

// ดึงค่าโดยตรง
val Map("host" -> host, "port" -> port) = config
println(s"Connecting to $host:$port")

// Pattern matching กับ Map
def configToString(config: Map[String, String]): String =
  config match
    case m if m.contains("host") =>
      s"Server at ${m("host")}:${m.getOrElse("port", "80")}"
    case _ => "Unknown config"
```

### Pattern Matching ใน val

```scala
// Partial match - ระวัง MatchError!
val Some(x) = Option(42)  // ✅ works
println(x)  // 42

// ❌ อันตราย!
val Some(y) = Option.empty[Int]  // MatchError: None

// ✅ ปลอดภัยกว่า
val optValue = Option(42)
val result = optValue match
  case Some(v) => v
  case None    => 0
```

---

## ตัวอย่างโปรแกรมประยุกต์

### โปรแกรมจัดการ Student Records

```scala
case class Student(
  id: Int,
  name: String,
  scores: Map[String, Double]
):
  lazy val average: Double =
    if scores.isEmpty then 0.0
    else scores.values.sum / scores.size

  lazy val grade: String =
    average match
      case avg if avg >= 90 => "A"
      case avg if avg >= 80 => "B"
      case avg if avg >= 70 => "C"
      case avg if avg >= 60 => "D"
      case _                => "F"

  def addScore(subject: String, score: Double): Student =
    copy(scores = scores + (subject -> score))

  override def toString: String =
    f"[$id] $name - Average: $average%.1f ($grade)"

@main def studentDemo(): Unit =
  // สร้างนักเรียน
  var alice = Student(1, "Alice", Map.empty)
  alice = alice.addScore("Math", 92.0)
  alice = alice.addScore("Science", 88.0)
  alice = alice.addScore("English", 95.0)

  val bob = Student(2, "Bob",
    Map("Math" -> 75.0, "Science" -> 82.0, "English" -> 68.0)
  )

  val students = List(alice, bob)

  // แสดงผล
  println("=== Student Report ===")
  students.foreach(println)

  // Statistics
  val (topScore, topStudent) = students
    .map(s => (s.average, s.name))
    .maxBy(_._1)

  println(f"\nTop Student: $topStudent (avg: $topScore%.1f)")

  // Destructuring ใน for loop
  println("\nDetailed Scores:")
  for
    student <- students
    (subject, score) <- student.scores
  do
    println(f"  ${student.name}: $subject = $score%.1f")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: val vs var

เขียนโค้ดด้วย val ให้ได้ผลเหมือนกับ:

```scala
// แบบ var (อย่าใช้แบบนี้)
var total = 0
var count = 0
for price <- List(10, 20, 30, 40, 50) do
  total += price
  count += 1
val average = total.toDouble / count

// TODO: เขียนใหม่ด้วย val และ functional approach
```

### แบบฝึกหัดที่ 2: Lazy val

```scala
// สร้าง class ที่มี lazy val สำหรับ:
// 1. prime numbers list (คำนวณแพง)
// 2. connection string (ขึ้นกับ host และ port)

class DatabaseConfig(host: String, port: Int, dbName: String):
  // TODO: สร้าง lazy val สำหรับ connection string
  // format: "jdbc:postgresql://host:port/dbName"

  // TODO: สร้าง lazy val สำหรับ URL
  // format: "http://host:port"

  // TODO: สร้าง lazy val เพื่อ validate (ตรวจว่า port อยู่ใน 1-65535)
```

### แบบฝึกหัดที่ 3: Destructuring

```scala
// ใช้ destructuring assignment
val coordinates = List((0, 0), (1, 2), (3, 4), (-1, -1))

// TODO:
// 1. แกะแต่ละ coordinate เป็น (x, y)
// 2. คำนวณระยะทางจาก origin (0,0)
// 3. หา coordinate ที่ไกลที่สุด
// 4. แสดงผล "Farthest point: (x, y) at distance d"
```

### แบบฝึกหัดที่ 4: Immutable Update

```scala
case class ShoppingCart(items: Map[String, Int], discount: Double = 0.0):
  def addItem(name: String, qty: Int): ShoppingCart = ???
  def removeItem(name: String): ShoppingCart = ???
  def applyDiscount(pct: Double): ShoppingCart = ???
  def total(prices: Map[String, Double]): Double = ???

// TODO: implement methods
// ทุก method ต้อง return ShoppingCart ใหม่ (ไม่ mutate!)

@main def cartDemo(): Unit =
  val prices = Map("apple" -> 0.5, "banana" -> 0.3, "milk" -> 1.5)

  val cart = ShoppingCart(Map.empty)
    .addItem("apple", 3)
    .addItem("banana", 2)
    .addItem("milk", 1)
    .applyDiscount(0.1)  // 10% discount

  println(f"Total: $$${cart.total(prices)}%.2f")
```

**เฉลย แบบฝึกหัดที่ 1:**

```scala
@main def exercise1Answer(): Unit =
  val prices = List(10, 20, 30, 40, 50)

  val total = prices.sum
  val count = prices.length
  val average = total.toDouble / count

  println(s"Total: $total")
  println(s"Count: $count")
  println(f"Average: $average%.2f")
```

**เฉลย แบบฝึกหัดที่ 3:**

```scala
@main def exercise3Answer(): Unit =
  val coordinates = List((0, 0), (1, 2), (3, 4), (-1, -1))

  val distances = coordinates.map { case (x, y) =>
    val dist = math.sqrt(x * x + y * y)
    ((x, y), dist)
  }

  val ((fx, fy), maxDist) = distances.maxBy(_._2)

  println(f"Farthest point: ($fx, $fy) at distance $maxDist%.2f")
```

**เฉลย แบบฝึกหัดที่ 4:**

```scala
case class ShoppingCart(items: Map[String, Int], discount: Double = 0.0):
  def addItem(name: String, qty: Int): ShoppingCart =
    val newQty = items.getOrElse(name, 0) + qty
    copy(items = items + (name -> newQty))

  def removeItem(name: String): ShoppingCart =
    copy(items = items - name)

  def applyDiscount(pct: Double): ShoppingCart =
    copy(discount = pct)

  def total(prices: Map[String, Double]): Double =
    val subtotal = items.map { case (name, qty) =>
      prices.getOrElse(name, 0.0) * qty
    }.sum
    subtotal * (1 - discount)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ `val` (immutable) vs `var` (mutable) - และเหตุผลที่ควรเลือก val
- ✅ `lazy val` - การ defer computation และประโยชน์ในหลายกรณี
- ✅ Type annotations - เมื่อไหรควรใส่และเมื่อไหรปล่อยให้ inference
- ✅ Scope และ visibility rules
- ✅ หลักการ Immutability และทำไมถึงสำคัญ
- ✅ Variable shadowing - ข้อดี ข้อเสีย และกรณีที่ควรระวัง
- ✅ Constants และการใช้งานใน pattern matching
- ✅ Destructuring assignment สำหรับ Tuples, Case Classes, Lists

## ขั้นตอนถัดไป

ใน [Part 05: การควบคุมโปรแกรม](part-05-control-flow.md) เราจะเรียนรู้:
- if-else expressions
- while และ do-while loops
- for loops และ for comprehensions
- Pattern matching เบื้องต้น
- throw และ try-catch-finally

---

*[← Part 03: ประเภทข้อมูล](part-03-data-types.md) | [Part 05: การควบคุมโปรแกรม →](part-05-control-flow.md)*
