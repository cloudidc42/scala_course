# Part 06: ฟังก์ชัน (Functions)

## สารบัญ
1. [Function Definitions](#function-definitions)
2. [Default Parameters](#default-parameters)
3. [Named Arguments](#named-arguments)
4. [Varargs](#varargs)
5. [Return Types และ Unit](#return-types)
6. [Nested Functions](#nested-functions)
7. [First-Class Functions](#first-class-functions)
8. [Anonymous Functions (Lambdas)](#anonymous-functions)
9. [Higher-Order Functions](#higher-order-functions)
10. [Closures](#closures)
11. [Partial Application](#partial-application)
12. [Currying](#currying)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Function Definitions

### def Syntax

```scala
// รูปแบบพื้นฐาน
def functionName(param1: Type1, param2: Type2): ReturnType =
  body

// ตัวอย่างง่าย
def add(a: Int, b: Int): Int = a + b
def greet(name: String): Unit = println(s"Hello, $name!")
def square(x: Double): Double = x * x

// Multi-line function
def factorial(n: Int): Long =
  if n <= 1 then 1L
  else n * factorial(n - 1)

// Return type inference (สำหรับ non-recursive)
def double(x: Int) = x * 2  // Scala infers: Int
```

### Function Body

```scala
// Single expression body
def max(a: Int, b: Int): Int = if a > b then a else b

// Block body (ค่าสุดท้ายเป็น return value)
def describe(n: Int): String =
  val absolute = math.abs(n)
  val parity = if n % 2 == 0 then "even" else "odd"
  s"$n is $parity (absolute: $absolute)"  // return value

// การใช้ return (ไม่แนะนำ ยกเว้นจำเป็น)
def findFirst(list: List[Int], target: Int): Int =
  for item <- list do
    if item == target then return item  // early return
  -1  // ไม่พบ

// Recursive
def sum(n: Int): Long =
  if n <= 0 then 0L
  else n + sum(n - 1)
```

### Procedure Syntax (ล้าสมัยใน Scala 3)

```scala
// Scala 2: procedure syntax (ไม่มี = สำหรับ Unit)
// def printHello() { println("Hello") }  // ล้าสมัย!

// Scala 3: ต้องระบุ Unit อย่างชัดเจน
def printHello(): Unit = println("Hello")
def printHello2(): Unit =
  println("Hello")
  println("World")
```

### Method ใน Class/Object

```scala
class Calculator:
  // Instance method
  def add(a: Int, b: Int): Int = a + b

  // Method กับ state
  private var memory: Double = 0.0

  def storeInMemory(value: Double): Unit =
    memory = value

  def recallMemory(): Double = memory

object MathUtils:
  // Static-like method ใน object
  def circleArea(r: Double): Double = math.Pi * r * r
  def hypotenuse(a: Double, b: Double): Double =
    math.sqrt(a * a + b * b)
```

---

## Default Parameters

```scala
// Default parameter values
def greet(name: String, greeting: String = "Hello"): String =
  s"$greeting, $name!"

println(greet("Alice"))             // Hello, Alice!
println(greet("Bob", "Hi"))         // Hi, Bob!
println(greet("Charlie", "Howdy"))  // Howdy, Charlie!

// หลาย default parameters
def createUser(
  name: String,
  role: String = "user",
  active: Boolean = true,
  maxSessions: Int = 3
): String =
  s"User($name, $role, active=$active, sessions=$maxSessions)"

println(createUser("Alice"))
// User(Alice, user, active=true, sessions=3)

println(createUser("Admin", role = "admin", maxSessions = 10))
// User(Admin, admin, active=true, sessions=10)

// Default ที่เป็น expression
def makeList[A](elem: A, size: Int = 10): List[A] =
  List.fill(size)(elem)

println(makeList("x"))     // List(x, x, x, x, x, x, x, x, x, x)
println(makeList("y", 3))  // List(y, y, y)
```

### Default Parameters กับ Overloading

```scala
// Default parameters ช่วยลด overloading
// ❌ แบบ Java ที่ต้องมี overloaded methods
// def log(message: String): Unit = log(message, "INFO")
// def log(message: String, level: String): Unit = ...

// ✅ แบบ Scala
def log(
  message: String,
  level: String = "INFO",
  timestamp: Boolean = true
): Unit =
  val ts = if timestamp then s"[${java.time.LocalTime.now()}] " else ""
  println(s"$ts[$level] $message")

log("Server started")
log("Error occurred", "ERROR")
log("Debug info", "DEBUG", false)
```

---

## Named Arguments

```scala
def connect(
  host: String,
  port: Int,
  timeout: Int,
  ssl: Boolean
): String =
  s"Connecting to $host:$port (timeout=${timeout}s, ssl=$ssl)"

// Named arguments ช่วยให้โค้ดชัดเจน
println(connect(
  host = "localhost",
  port = 5432,
  timeout = 30,
  ssl = false
))

// เปลี่ยนลำดับได้เมื่อใช้ named arguments
println(connect(
  ssl = true,
  host = "production.db.com",
  timeout = 60,
  port = 5433
))

// ผสม positional กับ named (positional ต้องมาก่อน)
println(connect("localhost", 5432, ssl = false, timeout = 30))
```

### Named Arguments กับ Case Classes

```scala
case class Config(
  host: String = "localhost",
  port: Int = 8080,
  debug: Boolean = false,
  maxConnections: Int = 100
)

// สร้าง config โดยใช้ named arguments
val devConfig = Config(debug = true)
val prodConfig = Config(
  host = "api.example.com",
  port = 443,
  maxConnections = 1000
)

println(devConfig)   // Config(localhost,8080,true,100)
println(prodConfig)  // Config(api.example.com,443,false,1000)
```

---

## Varargs

```scala
// Varargs: รับ arguments จำนวนไม่กำหนด
def sum(numbers: Int*): Int =
  numbers.sum  // numbers เป็น Seq[Int]

println(sum())           // 0
println(sum(1))          // 1
println(sum(1, 2, 3))    // 6
println(sum(1, 2, 3, 4, 5))  // 15

// ส่ง collection เป็น varargs ด้วย :_*  (Scala 2)
val nums = List(1, 2, 3, 4)
println(sum(nums*))      // Scala 3: nums*
// println(sum(nums: _*)) // Scala 2 style

// Varargs กับ parameter อื่น (varargs ต้องเป็นตัวสุดท้าย)
def log(level: String, messages: String*): Unit =
  messages.foreach(msg => println(s"[$level] $msg"))

log("INFO", "Server started", "Listening on port 8080")
log("ERROR", "Connection failed")
log("DEBUG")  // ไม่มี messages ก็ได้

// Format function
def format(template: String, values: Any*): String =
  values.foldLeft(template) { (acc, value) =>
    acc.replaceFirst("%s", value.toString)
  }

println(format("Hello, %s! You have %s messages.", "Alice", 5))
// Hello, Alice! You have 5 messages.
```

---

## Return Types

### Explicit vs Inferred Return Types

```scala
// Explicit return type (แนะนำสำหรับ public API)
def divide(a: Double, b: Double): Option[Double] =
  if b == 0 then None else Some(a / b)

// Inferred return type (ดีสำหรับ internal/simple functions)
def double(x: Int) = x * 2           // Int
def greet(name: String) = s"Hi $name" // String

// Recursive ต้องระบุ return type เสมอ
def fibonacci(n: Int): BigInt =
  if n <= 1 then n
  else fibonacci(n - 1) + fibonacci(n - 2)

// Unit return type
def printInfo(msg: String): Unit = println(msg)
def printInfo2(msg: String) = println(msg)  // inferred Unit ก็ได้
```

### Never Return (Nothing type)

```scala
// ฟังก์ชันที่ throw เสมอ return type คือ Nothing
def fail(msg: String): Nothing =
  throw RuntimeException(msg)

// ใช้งาน
def validatePositive(n: Int): Int =
  if n > 0 then n
  else fail(s"Expected positive, got $n")

// ??? stub
def notImplemented(): Int = ???  // throws NotImplementedError
```

---

## Nested Functions

```scala
def processData(data: List[Int]): List[String] =
  // Inner function - เข้าถึงได้เฉพาะใน processData
  def format(n: Int): String =
    if n > 0 then s"+$n"
    else if n < 0 then s"$n"
    else "zero"

  def validate(n: Int): Boolean =
    n != Int.MinValue && n != Int.MaxValue

  data.filter(validate).map(format)

println(processData(List(-3, 0, Int.MinValue, 5, Int.MaxValue, -1)))
// List(-3, zero, +5, -1)

// Nested function สามารถเรียก outer function ได้
def outerFn(x: Int): Int =
  def helper(n: Int): Int =
    if n <= 0 then 0
    else n + helper(n - 1)

  helper(x) * 2

println(outerFn(5))  // 30 (1+2+3+4+5=15, *2=30)
```

---

## First-Class Functions

### Functions เป็น Values

```scala
// ฟังก์ชันเป็น value ที่เก็บในตัวแปรได้
val double: Int => Int = x => x * 2
val addOne: Int => Int = (x: Int) => x + 1
val greet: String => String = name => s"Hello, $name!"

println(double(5))    // 10
println(addOne(4))    // 5
println(greet("Alice"))  // Hello, Alice!

// Function type syntax
// Input => Output
val square: Int => Int = x => x * x
val add: (Int, Int) => Int = (a, b) => a + b
val printMsg: String => Unit = msg => println(msg)

// Function กับหลาย inputs
val multiAdd: (Int, Int, Int) => Int = (a, b, c) => a + b + c
```

### Method to Function Conversion

```scala
// Method (def) สามารถแปลงเป็น function value ได้
def multiply(a: Int, b: Int): Int = a * b

// Eta expansion: แปลง method เป็น function
val multiplyFn: (Int, Int) => Int = multiply _  // Scala 2
val multiplyFn2: (Int, Int) => Int = multiply    // Scala 3

// หรือใช้ directly เป็น function argument
List(1, 2, 3).map(double)     // ใช้ val function
List(1, 2, 3).map(x => x + 1) // lambda
```

### Function Types

```scala
// ประเภทของ Function
// Function0[R]: () => R
// Function1[A, R]: A => R
// Function2[A, B, R]: (A, B) => R
// ... ถึง Function22

val noArgs: () => Int = () => 42
val oneArg: Int => Int = x => x + 1
val twoArgs: (Int, String) => String = (n, s) => s"$n: $s"

// เรียกใช้
println(noArgs())           // 42
println(oneArg(5))          // 6
println(twoArgs(3, "hello")) // 3: hello

// apply method
println(noArgs.apply())         // 42 (เทียบเท่า noArgs())
println(oneArg.apply(5))        // 6
```

---

## Anonymous Functions (Lambdas)

### Lambda Syntax

```scala
// รูปแบบเต็ม
val double1 = (x: Int) => x * 2

// ละ type ได้เมื่อ inferred
val double2: Int => Int = x => x * 2

// Placeholder syntax: _ แทน single parameter
val double3: Int => Int = _ * 2

// Multiple parameters
val add1 = (a: Int, b: Int) => a + b
val add2: (Int, Int) => Int = (a, b) => a + b
val add3: (Int, Int) => Int = _ + _  // placeholder

// Multi-line lambda
val process: List[Int] => String = list =>
  val filtered = list.filter(_ > 0)
  val doubled = filtered.map(_ * 2)
  doubled.mkString(", ")
```

### Placeholder Syntax

```scala
// _ แทน parameter ที่ใช้ครั้งเดียว

// หนึ่ง parameter
List(1, 2, 3).map(_ * 2)        // x => x * 2
List(1, 2, 3).filter(_ > 1)     // x => x > 1
List(1, 2, 3).foreach(println)  // x => println(x)

// สอง parameters
List(1, 2, 3).reduce(_ + _)  // (a, b) => a + b
(1 to 10).foldLeft(0)(_ + _) // (acc, x) => acc + x

// ระวัง: _ ใช้ได้เฉพาะเมื่อแต่ละ _ แทน parameter ต่างกัน
// และแต่ละ _ ต้องปรากฏแค่ครั้งเดียว

// ✅ ถูกต้อง
List(1, 2, 3).map(_ + 1)         // x => x + 1
List(1, 2, 3).reduce(_ + _)      // (a, b) => a + b

// ❌ ผิด - ใช้ parameter เดิมสองครั้ง
// List(1, 2, 3).map(_ + _)  // Error! แต่ละ _ เป็นคนละ parameter

// ✅ ต้องเขียนแบบนี้แทน
List(1, 2, 3).map(x => x + x)    // = x * 2
```

### Lambda ที่ซับซ้อน

```scala
// Lambda ที่มีหลายบรรทัด
val processUser: (String, Int) => String = (name, age) =>
  val cleanName = name.trim.capitalize
  val ageGroup = if age < 18 then "minor"
                 else if age < 65 then "adult"
                 else "senior"
  s"$cleanName ($ageGroup)"

println(processUser("  alice  ", 25))  // Alice (adult)
println(processUser("bob", 15))        // Bob (minor)

// Lambda กับ pattern matching
val describeOption: Option[Int] => String =
  case Some(n) => s"Has value: $n"
  case None    => "Empty"

println(describeOption(Some(42)))  // Has value: 42
println(describeOption(None))      // Empty
```

---

## Higher-Order Functions

### Functions as Arguments

```scala
// Higher-order function: รับ function เป็น parameter
def applyTwice(f: Int => Int, x: Int): Int = f(f(x))

println(applyTwice(_ * 2, 3))   // 12 (3*2=6, 6*2=12)
println(applyTwice(_ + 1, 5))   // 7 (5+1=6, 6+1=7)
println(applyTwice(x => x * x, 2))  // 16 (2^2=4, 4^2=16)

// ใช้ฟังก์ชันชื่อ
def addOne(x: Int): Int = x + 1
println(applyTwice(addOne, 0))  // 2
```

### Functions as Return Values

```scala
// Higher-order function: คืน function
def multiplier(factor: Int): Int => Int =
  x => x * factor

val double = multiplier(2)
val triple = multiplier(3)
val times10 = multiplier(10)

println(double(5))   // 10
println(triple(5))   // 15
println(times10(5))  // 50

// Function factory
def greeter(greeting: String): String => String =
  name => s"$greeting, $name!"

val hello = greeter("Hello")
val hi = greeter("Hi")
val sawatdi = greeter("สวัสดี")

println(hello("Alice"))    // Hello, Alice!
println(hi("Bob"))         // Hi, Bob!
println(sawatdi("Charlie")) // สวัสดี, Charlie!
```

### Standard Higher-Order Functions

```scala
val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// map: แปลงทุก element
println(numbers.map(_ * 2))     // List(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
println(numbers.map(_.toString)) // List(1, 2, 3, ...)

// filter: เลือก element ที่ตรงเงื่อนไข
println(numbers.filter(_ % 2 == 0))  // List(2, 4, 6, 8, 10)

// reduce: รวม elements เป็นค่าเดียว (ต้องมีอย่างน้อย 1 element)
println(numbers.reduce(_ + _))   // 55
println(numbers.reduce(_ max _)) // 10

// fold: เหมือน reduce แต่มีค่าเริ่มต้น
println(numbers.foldLeft(0)(_ + _))   // 55
println(numbers.foldRight(0)(_ + _))  // 55

// foldLeft สร้าง stack ทางซ้าย
// foldRight สร้าง stack ทางขวา
val reversed = numbers.foldLeft(List[Int]())((acc, x) => x :: acc)
println(reversed)  // List(10, 9, 8, 7, 6, 5, 4, 3, 2, 1)

// flatMap: map แล้ว flatten
val words = List("hello world", "scala is great")
println(words.flatMap(_.split(" ")))
// List(hello, world, scala, is, great)

// forEach: ทำ side effects
numbers.take(3).foreach(n => println(s"Number: $n"))

// partition: แยกเป็นสอง lists
val (evens, odds) = numbers.partition(_ % 2 == 0)
println(evens)  // List(2, 4, 6, 8, 10)
println(odds)   // List(1, 3, 5, 7, 9)

// groupBy: จัดกลุ่ม
val grouped = numbers.groupBy(_ % 3)
println(grouped)  // Map(0 -> List(3,6,9), 1 -> List(1,4,7,10), 2 -> List(2,5,8))

// sortBy
val words2 = List("banana", "apple", "cherry", "date")
println(words2.sortBy(_.length))  // List(date, apple, banana, cherry)
println(words2.sorted)            // List(apple, banana, cherry, date)
println(words2.sortWith(_ > _))   // descending: List(date, cherry, banana, apple)
```

### Custom Higher-Order Functions

```scala
// สร้าง HOF เอง
def pipeline[A](value: A)(transforms: (A => A)*): A =
  transforms.foldLeft(value)((v, f) => f(v))

val result = pipeline(5)(
  _ * 2,    // 10
  _ + 3,    // 13
  _ * _ - 1 // ❌ ไม่ได้เพราะ _ แทน parameter สองตัว
)
// ต้องเขียนเป็น:
val result2 = pipeline(5)(
  x => x * 2,     // 10
  x => x + 3,     // 13
  x => x * x - 1  // 168
)
println(result2)  // 168

// Composing functions
def compose[A, B, C](f: B => C, g: A => B): A => C =
  x => f(g(x))

val doubleAndSquare = compose((x: Int) => x * x, (x: Int) => x * 2)
println(doubleAndSquare(3))  // (3*2)^2 = 36

// andThen: f andThen g = g(f(x))
val double = (x: Int) => x * 2
val square = (x: Int) => x * x

val doubleThenSquare = double andThen square  // square(double(x))
val squareThenDouble = double compose square  // double(square(x))

println(doubleThenSquare(3))  // (3*2)^2 = 36
println(squareThenDouble(3))  // (3^2)*2 = 18
```

---

## Closures

### Closure พื้นฐาน

```scala
// Closure: function ที่ capture ตัวแปรจาก outer scope
def makeCounter(): () => Int =
  var count = 0
  () => {
    count += 1
    count
  }

val counter1 = makeCounter()
val counter2 = makeCounter()  // counter ใหม่ที่เป็นอิสระ

println(counter1())  // 1
println(counter1())  // 2
println(counter1())  // 3
println(counter2())  // 1 (เริ่มใหม่)
println(counter1())  // 4
```

### Closure กับ Immutable Values

```scala
// Closure capture val
val base = 100
val addBase: Int => Int = x => x + base

println(addBase(5))   // 105
println(addBase(20))  // 120

// base ถูก capture by reference (ไม่ใช่ copy)
// สำหรับ val ไม่มีปัญหา เพราะเปลี่ยนไม่ได้
```

### Closure Pitfalls (กับ var)

```scala
// ⚠️ ระวัง: closure capture var by reference
var multiplier = 2
val multiply = (x: Int) => x * multiplier

println(multiply(5))  // 10

multiplier = 3  // เปลี่ยน multiplier
println(multiply(5))  // 15 (เปลี่ยนตาม!)

// ตัวอย่างที่อาจสร้างความสับสน
val funcs = (1 to 5).map { i =>
  () => i  // capture i (val จาก for comprehension)
}
// นี่ปลอดภัยเพราะ i เป็น val ใหม่แต่ละ iteration
funcs.foreach(f => print(s"${f()} "))  // 1 2 3 4 5

// เทียบกับ Java ที่มีปัญหากับ var ใน closures
```

---

## Partial Application

```scala
// Partial application: ใส่ argument บางส่วน สร้าง function ใหม่
def add(x: Int, y: Int): Int = x + y

// Partial application ด้วย _ (Scala 2 style)
val add5 = add(5, _: Int)  // fix x=5
println(add5(3))   // 8
println(add5(10))  // 15

// ใน Scala 3 สามารถใช้ underscore ได้เช่นกัน
val add10 = add(10, _: Int)
println(add10(5))   // 15

// Partial application ด้วย higher-order
def multiply(x: Int)(y: Int): Int = x * y  // curried

val triple = multiply(3) _  // Scala 2
val triple2 = multiply(3)   // Scala 3 (ถ้า method มีหลาย parameter list)
println(triple(4))   // 12
println(triple2(4))  // 12

// ตัวอย่างจริง: partial application สำหรับ config
def createEmail(
  from: String,
  subject: String,
  body: String
): String = s"From: $from\nSubject: $subject\n\n$body"

val systemEmail = createEmail("system@example.com", _: String, _: String)
val alertEmail = systemEmail("ALERT", _: String)

println(alertEmail("Database connection failed!"))
```

---

## Currying

### Curried Functions

```scala
// Currying: แปลง f(a, b) เป็น f(a)(b)
// Scala รองรับ currying โดยตรงด้วย multiple parameter lists

// Non-curried
def addNormal(x: Int, y: Int): Int = x + y

// Curried
def addCurried(x: Int)(y: Int): Int = x + y

println(addCurried(3)(4))  // 7

// Partial application ง่ายขึ้น
val add3 = addCurried(3)    // function ที่รับ y และ return x+3
val add10 = addCurried(10)

println(add3(7))   // 10
println(add10(5))  // 15
```

### Currying กับ Type Parameters

```scala
def map[A, B](list: List[A])(f: A => B): List[B] =
  list.map(f)

// Partial application
val doubleAll = map[Int, Int](_: List[Int])(_ * 2)
// หรือ
val doubleList = (list: List[Int]) => map(list)(_ * 2)

println(doubleList(List(1, 2, 3)))  // List(2, 4, 6)
```

### ประโยชน์ของ Currying

```scala
// 1. Configuration ที่ต้องการ context
def withRetry(maxRetries: Int)(operation: () => Int): Int =
  var attempts = 0
  var result = 0
  while attempts < maxRetries do
    try
      result = operation()
      attempts = maxRetries  // success, exit loop
    catch
      case e: Exception =>
        attempts += 1
        if attempts >= maxRetries then throw e
  result

val retryThrice = withRetry(3)

// ใช้งาน
val result = retryThrice { () =>
  if math.random() > 0.5 then 42
  else throw RuntimeException("Random failure")
}

// 2. DSL-like syntax
def repeat(times: Int)(action: => Unit): Unit =
  for _ <- 1 to times do action

repeat(3) {
  println("Hello!")
}

// 3. By-name parameters กับ currying
def time[T](label: String)(block: => T): T =
  val start = System.currentTimeMillis()
  val result = block
  val elapsed = System.currentTimeMillis() - start
  println(s"$label took ${elapsed}ms")
  result

val result2 = time("Sort operation") {
  List.fill(100000)(math.random()).sorted
}
```

### Function Composition กับ Currying

```scala
// Point-free style
val isEven: Int => Boolean = _ % 2 == 0
val square: Int => Int = x => x * x
val toString: Int => String = _.toString

// compose
val evenSquareString: Int => String =
  isEven.andThen(b => if b then "even" else "odd")
  // ไม่ได้แล้ว เพราะ type ไม่ตรง

// ต้องใช้ andThen/compose อย่างถูกต้อง
val processNumber: Int => String =
  ((n: Int) => n * n)
    .andThen(n => if n % 2 == 0 then s"$n is even square" else s"$n is odd square")

println(processNumber(4))  // 16 is even square
println(processNumber(3))  // 9 is odd square

// Lifted functions
def liftToOption[A, B](f: A => B): Option[A] => Option[B] =
  _.map(f)

val safeDouble = liftToOption[Int, Int](_ * 2)
println(safeDouble(Some(5)))  // Some(10)
println(safeDouble(None))     // None
```

---

## ตัวอย่างโปรแกรมครบ: Functional Pipeline

```scala
import scala.util.Try

case class Product(name: String, price: Double, category: String, inStock: Boolean)

@main def functionalPipeline(): Unit =
  val products = List(
    Product("Apple MacBook Pro", 2499.99, "Laptops", true),
    Product("Samsung Galaxy S24", 999.99, "Phones", true),
    Product("Sony WH-1000XM5", 349.99, "Headphones", false),
    Product("iPad Pro", 1099.99, "Tablets", true),
    Product("AirPods Pro", 249.99, "Headphones", true),
    Product("Dell XPS 15", 1799.99, "Laptops", false),
    Product("Google Pixel 8", 699.99, "Phones", true)
  )

  // Pipeline: filter → sort → group → summarize

  // 1. Filter in-stock products
  val inStock = products.filter(_.inStock)

  // 2. Functions for transformations
  val formatPrice: Double => String = p => f"$$${p}%.2f"
  val formatProduct: Product => String =
    p => s"${p.name} (${formatPrice(p.price)})"

  // 3. Group by category
  val byCategory = inStock.groupBy(_.category)

  // 4. Summary function
  def categorySummary(category: String, prods: List[Product]): String =
    val sorted = prods.sortBy(_.price)
    val prices = prods.map(_.price)
    val avg = prices.sum / prices.length
    s"""$category:
       |  Count: ${prods.length}
       |  Avg Price: ${formatPrice(avg)}
       |  Cheapest: ${formatProduct(sorted.head)}
       |  Most Expensive: ${formatProduct(sorted.last)}""".stripMargin

  // 5. Display
  println("=== Product Catalog (In Stock) ===\n")

  byCategory.toList
    .sortBy(_._1)
    .foreach { case (category, prods) =>
      println(categorySummary(category, prods))
      println()
    }

  // 6. Overall stats
  val allPrices = inStock.map(_.price)
  println(s"Total in-stock products: ${inStock.length}")
  println(f"Total inventory value: ${formatPrice(allPrices.sum)}")
  println(f"Average price: ${formatPrice(allPrices.sum / allPrices.length)}")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Higher-Order Functions

```scala
// implement ฟังก์ชันต่อไปนี้ด้วย higher-order functions
// (ห้ามใช้ loop!)

def sumOfSquaresOfEvens(numbers: List[Int]): Int = ???
// ผลรวมของกำลังสองของเลขคู่

def longestWord(sentence: String): String = ???
// คำที่ยาวที่สุดใน sentence

def countByFirstLetter(words: List[String]): Map[Char, Int] = ???
// นับจำนวนคำที่ขึ้นต้นด้วยตัวอักษรแต่ละตัว

def transformAndFilter[A, B](
  list: List[A],
  transform: A => B,
  predicate: B => Boolean
): List[B] = ???
// transform แล้ว filter
```

### แบบฝึกหัดที่ 2: Function Composition

```scala
// สร้าง text processing pipeline

val steps: List[String => String] = List(
  _.trim,                           // ตัด whitespace
  _.toLowerCase,                    // lowercase
  _.replaceAll("[^a-z0-9 ]", ""),  // เอา special chars ออก
  _.replaceAll("\\s+", " ")         // normalize spaces
)

// TODO: compose ทุก step เป็น function เดียว
def normalize: String => String = ???

println(normalize("  Hello, World!  "))  // "hello world"
println(normalize("  Scala 3.3.1 is AWESOME!!!  "))  // "scala 331 is awesome"
```

### แบบฝึกหัดที่ 3: Currying

```scala
// สร้าง curried validation functions

def validate[A](
  value: A
)(
  predicate: A => Boolean
)(
  errorMessage: => String
): Either[String, A] = ???

// ใช้งาน:
val validateAge = validate[Int](_)(age => age > 0 && age < 150)("Invalid age")
val validateName = validate[String](_)(_.nonEmpty)("Name cannot be empty")

// TODO: compose validations
case class UserInput(name: String, age: Int)

def validateUser(name: String, age: Int): Either[String, UserInput] = ???
// ควร return Right(UserInput) ถ้าทั้งสอง validate ผ่าน
// หรือ Left(error) ถ้าไม่ผ่าน
```

**เฉลย แบบฝึกหัดที่ 1:**

```scala
def sumOfSquaresOfEvens(numbers: List[Int]): Int =
  numbers.filter(_ % 2 == 0).map(n => n * n).sum

def longestWord(sentence: String): String =
  sentence.split("\\s+").maxBy(_.length)

def countByFirstLetter(words: List[String]): Map[Char, Int] =
  words.filter(_.nonEmpty)
    .groupBy(_.head.toLower)
    .view.mapValues(_.length)
    .toMap

def transformAndFilter[A, B](
  list: List[A],
  transform: A => B,
  predicate: B => Boolean
): List[B] =
  list.map(transform).filter(predicate)
```

**เฉลย แบบฝึกหัดที่ 2:**

```scala
def normalize: String => String =
  steps.reduce(_ andThen _)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Function definition รูปแบบต่างๆ
- ✅ Default parameters และ Named arguments
- ✅ Varargs (\*)
- ✅ Return types และ Unit
- ✅ Nested functions
- ✅ First-class functions (ฟังก์ชันเป็น value)
- ✅ Anonymous functions (lambdas) และ placeholder syntax
- ✅ Higher-order functions (map, filter, fold, etc.)
- ✅ Closures และ variable capture
- ✅ Partial application
- ✅ Currying และ multiple parameter lists

## ขั้นตอนถัดไป

ใน [Part 07: Collections พื้นฐาน](part-07-collections-basic.md) เราจะเรียนรู้:
- List, Set, Map ทั้งหมด
- Mutable vs Immutable collections
- Collection operations ครบถ้วน
- Performance considerations

---

*[← Part 05: การควบคุมโปรแกรม](part-05-control-flow.md) | [Part 07: Collections พื้นฐาน →](part-07-collections-basic.md)*
