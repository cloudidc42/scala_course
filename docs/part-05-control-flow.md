# Part 05: การควบคุมโปรแกรม (Control Flow)

## สารบัญ
1. [if-else Expressions](#if-else-expressions)
2. [while Loops](#while-loops)
3. [for Loops](#for-loops)
4. [for Comprehensions](#for-comprehensions)
5. [match Expression](#match-expression)
6. [try-catch-finally](#try-catch-finally)
7. [break และ continue (ไม่มีใน Scala!)](#break-และ-continue)
8. [Tail Recursion](#tail-recursion)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## if-else Expressions

### if-else พื้นฐาน

```scala
// if-else เป็น expression (คืนค่า)
val x = 10

// แบบง่าย
if x > 0 then println("positive")

// if-else
if x > 0 then
  println("positive")
else
  println("non-positive")

// if-else เป็น expression
val sign = if x > 0 then 1 else if x < 0 then -1 else 0
println(sign)  // 1

// ใช้ใน assignment
val message = if x > 0 then "positive" else "non-positive"
```

### if-else แบบ Multi-line

```scala
val score = 85

val grade =
  if score >= 90 then "A"
  else if score >= 80 then "B"
  else if score >= 70 then "C"
  else if score >= 60 then "D"
  else "F"

println(s"Grade: $grade")

// แบบมี body หลายบรรทัด
val feedback =
  if score >= 90 then
    val congrats = "Excellent!"
    val detail = "You scored in the top tier"
    s"$congrats $detail"
  else if score >= 60 then
    s"Passing grade: $score"
  else
    val msg = "You need to improve"
    val hint = "Consider reviewing the material"
    s"$msg. $hint"
```

### if-else กับ Unit

```scala
// ถ้า if ไม่มี else และ body เป็น Unit
val flag = false

if flag then println("flag is true")
// ✅ ไม่ต้องมี else ถ้า return type เป็น Unit

// แต่ถ้าเป็น expression ที่คืนค่า ต้องมี else
// val result = if flag then 42  // ❌ Error! (ต้องมี else)
val result = if flag then 42 else 0  // ✅
```

### Nested if-else

```scala
def classifyNumber(n: Int): String =
  if n == 0 then
    "zero"
  else if n > 0 then
    if n % 2 == 0 then
      "positive even"
    else
      "positive odd"
  else
    if n % 2 == 0 then
      "negative even"
    else
      "negative odd"

// ทดสอบ
List(-3, -2, 0, 1, 4).foreach { n =>
  println(s"$n is ${classifyNumber(n)}")
}
```

### if-else กับ Side Effects

```scala
// หลีกเลี่ยง if-else ที่มีแต่ side effects
// ❌ แบบนี้ไม่ดี
def processUser(user: Option[String]): Unit =
  if user.isDefined then
    println(s"Processing: ${user.get}")
  else
    println("No user")

// ✅ ดีกว่า: ใช้ pattern matching
def processUser2(user: Option[String]): Unit =
  user match
    case Some(name) => println(s"Processing: $name")
    case None       => println("No user")

// ✅ หรือใช้ fold
def processUser3(user: Option[String]): Unit =
  user.fold(println("No user"))(name => println(s"Processing: $name"))
```

---

## while Loops

### while Loop พื้นฐาน

```scala
// while loop ใน Scala
var i = 0
while i < 5 do
  println(i)
  i += 1
// Output: 0, 1, 2, 3, 4

// แบบ braces
var j = 0
while (j < 5) {
  println(j)
  j += 1
}
```

### do-while (Scala 3)

```scala
// Scala 3 มี do-while โดย syntax ใหม่
var count = 0
do
  println(s"Count: $count")
  count += 1
while count < 3
// Output: Count: 0, Count: 1, Count: 2

// Scala 2 style (ยังใช้ได้ใน Scala 3)
var n = 0
do {
  println(n)
  n += 1
} while (n < 3)
```

### while Loop ที่ใช้บ่อย

```scala
// อ่านข้อมูลจนครบ
import scala.io.Source

def readLines(filename: String): List[String] =
  val source = Source.fromFile(filename)
  val buffer = scala.collection.mutable.ArrayBuffer[String]()
  val iter = source.getLines()
  while iter.hasNext do
    buffer += iter.next()
  source.close()
  buffer.toList

// นับถอยหลัง
var countdown = 10
while countdown > 0 do
  println(s"$countdown...")
  countdown -= 1
println("Blast off!")

// ค้นหาใน array
def linearSearch(arr: Array[Int], target: Int): Int =
  var i = 0
  var found = -1
  while i < arr.length && found == -1 do
    if arr(i) == target then found = i
    i += 1
  found
```

### while vs Recursive vs Functional

```scala
// คำนวณ GCD สามแบบ

// 1. while loop
def gcdWhile(a: Int, b: Int): Int =
  var x = a
  var y = b
  while y != 0 do
    val temp = y
    y = x % y
    x = temp
  x

// 2. Recursive
def gcdRecursive(a: Int, b: Int): Int =
  if b == 0 then a
  else gcdRecursive(b, a % b)

// 3. Functional (scala standard)
def gcdFunctional(a: Int, b: Int): Int =
  BigInt(a).gcd(BigInt(b)).toInt

println(gcdWhile(48, 18))       // 6
println(gcdRecursive(48, 18))   // 6
println(gcdFunctional(48, 18))  // 6
```

---

## for Loops

### for Loop พื้นฐาน

```scala
// for loop กับ Range
for i <- 1 to 5 do
  println(i)  // 1, 2, 3, 4, 5

// to = inclusive, until = exclusive
for i <- 1 until 5 do
  println(i)  // 1, 2, 3, 4

// ก้าวทีละ n
for i <- 1 to 10 by 2 do
  print(s"$i ")  // 1 3 5 7 9

// ถอยหลัง
for i <- 10 to 1 by -1 do
  print(s"$i ")  // 10 9 8 7 6 5 4 3 2 1

println()
```

### for Loop กับ Collections

```scala
val fruits = List("apple", "banana", "cherry")

// iterate
for fruit <- fruits do
  println(fruit.capitalize)

// with index
for (fruit, index) <- fruits.zipWithIndex do
  println(s"${index + 1}. $fruit")

// nested loops
for
  i <- 1 to 3
  j <- 1 to 3
do
  println(s"($i, $j)")

// การสร้างตาราง
println("\nMultiplication Table:")
for i <- 1 to 5 do
  for j <- 1 to 5 do
    print(f"${i * j}%4d")
  println()
```

### for Loop กับ Guards (Filters)

```scala
// guards: ใช้ if เพื่อกรอง
for
  i <- 1 to 20
  if i % 2 == 0  // เฉพาะเลขคู่
do
  print(s"$i ")  // 2 4 6 8 10 12 14 16 18 20

println()

// หลาย guards
for
  i <- 1 to 100
  if i % 3 == 0   // หาร 3 ลงตัว
  if i % 5 == 0   // หาร 5 ลงตัว
do
  print(s"$i ")   // 15 30 45 60 75 90

println()

// FizzBuzz
for i <- 1 to 20 do
  if i % 15 == 0 then println("FizzBuzz")
  else if i % 3 == 0 then println("Fizz")
  else if i % 5 == 0 then println("Buzz")
  else println(i)
```

### for Loop กับ Multiple Generators

```scala
// Cartesian product
val colors = List("red", "green", "blue")
val sizes = List("S", "M", "L", "XL")

println("All combinations:")
for
  color <- colors
  size <- sizes
do
  println(s"  $color-$size")

// Nested collections
val matrix = List(List(1, 2, 3), List(4, 5, 6), List(7, 8, 9))

println("\nMatrix elements:")
for
  row <- matrix
  elem <- row
do
  print(s"$elem ")

println()

// ด้วย index
for
  (row, rowIdx) <- matrix.zipWithIndex
  (elem, colIdx) <- row.zipWithIndex
do
  print(s"[$rowIdx,$colIdx]=$elem ")
```

---

## for Comprehensions

### yield: สร้าง Collection ใหม่

```scala
// for comprehension ด้วย yield
val squares = for i <- 1 to 10 yield i * i
println(squares)  // Vector(1, 4, 9, 16, 25, 36, 49, 64, 81, 100)

// กับ filter
val evenSquares = for
  i <- 1 to 20
  if i % 2 == 0
yield i * i
println(evenSquares)  // Vector(4, 16, 36, 64, 100, 144, 196, 256, 324, 400)

// Nested generators
val pairs = for
  x <- 1 to 3
  y <- 1 to 3
  if x != y
yield (x, y)
println(pairs)  // Vector((1,2),(1,3),(2,1),(2,3),(3,1),(3,2))
```

### for Comprehension กับ List

```scala
val names = List("Alice", "Bob", "Charlie", "Diana")
val scores = List(85, 92, 78, 95)

// zip and transform
val results = for
  (name, score) <- names.zip(scores)
  if score >= 90
yield s"$name: $score (Excellent!)"

println(results)
// List(Bob: 92 (Excellent!), Diana: 95 (Excellent!))
```

### for Comprehension กับ Option

```scala
def findUser(id: Int): Option[String] =
  if id == 1 then Some("Alice")
  else if id == 2 then Some("Bob")
  else None

def findEmail(name: String): Option[String] =
  if name == "Alice" then Some("alice@example.com")
  else None

// Monadic composition ด้วย for comprehension
val emailResult = for
  name  <- findUser(1)
  email <- findEmail(name)
yield s"$name: $email"

println(emailResult)  // Some(Alice: alice@example.com)

val failResult = for
  name  <- findUser(999)  // None
  email <- findEmail(name)
yield s"$name: $email"

println(failResult)  // None (short-circuits)
```

### for Comprehension กับ Future

```scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

def fetchUser(id: Int): Future[String] =
  Future { s"User$id" }

def fetchOrders(userId: String): Future[List[String]] =
  Future { List(s"Order1 for $userId", s"Order2 for $userId") }

// Compose Futures ด้วย for comprehension
val result = for
  user   <- fetchUser(42)
  orders <- fetchOrders(user)
yield s"$user has ${orders.length} orders: ${orders.mkString(", ")}"

// รอผล
import scala.concurrent.Await
import scala.concurrent.duration.*
println(Await.result(result, 5.seconds))
```

### for Comprehension กับ Either

```scala
def parseAge(s: String): Either[String, Int] =
  s.toIntOption match
    case Some(n) if n > 0 => Right(n)
    case Some(_)           => Left("Age must be positive")
    case None              => Left(s"'$s' is not a number")

def validateAge(age: Int): Either[String, Int] =
  if age < 18 then Left("Must be 18 or older")
  else Right(age)

// Chain validations
def processAge(input: String): Either[String, String] =
  for
    parsed    <- parseAge(input)
    validated <- validateAge(parsed)
  yield s"Valid age: $validated"

println(processAge("25"))    // Right(Valid age: 25)
println(processAge("15"))    // Left(Must be 18 or older)
println(processAge("-5"))    // Left(Age must be positive)
println(processAge("abc"))   // Left('abc' is not a number)
```

### ความเข้าใจเชิงลึก: for เป็น syntactic sugar

```scala
// for comprehension แปลงเป็น map/flatMap/filter
val result1 = for
  x <- List(1, 2, 3)
  y <- List(10, 20)
  if x + y > 15
yield x + y

// เทียบเท่ากับ:
val result2 = List(1, 2, 3)
  .flatMap(x =>
    List(10, 20)
      .filter(y => x + y > 15)
      .map(y => x + y)
  )

println(result1 == result2)  // true
```

---

## match Expression

### Pattern Matching พื้นฐาน

```scala
// match เป็น expression
val day = 3

val dayName = day match
  case 1 => "Monday"
  case 2 => "Tuesday"
  case 3 => "Wednesday"
  case 4 => "Thursday"
  case 5 => "Friday"
  case 6 => "Saturday"
  case 7 => "Sunday"
  case _ => "Unknown"  // default case

println(dayName)  // Wednesday
```

### Pattern Matching แบบต่างๆ

```scala
// 1. Literal patterns
def describe(x: Any): String = x match
  case 0         => "zero"
  case 1         => "one"
  case true      => "true"
  case false     => "false"
  case "hello"   => "greeting"
  case _         => s"something else: $x"

// 2. Type patterns
def typeOf(x: Any): String = x match
  case _: Int    => "Int"
  case _: String => "String"
  case _: Double => "Double"
  case _: List[_] => "List"
  case _         => "Unknown"

// 3. Guards (ทำงานร่วมกับ if)
def classify(n: Int): String = n match
  case n if n < 0  => "negative"
  case 0           => "zero"
  case n if n < 10 => "small positive"
  case n if n < 100 => "medium positive"
  case _           => "large positive"

// 4. Multiple patterns (OR)
def isWeekend(day: Int): Boolean = day match
  case 6 | 7 => true
  case _     => false
```

### Pattern Matching กับ Case Classes

```scala
sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

def area(shape: Shape): Double = shape match
  case Circle(r)         => math.Pi * r * r
  case Rectangle(w, h)   => w * h
  case Triangle(b, h)    => 0.5 * b * h

def describe(shape: Shape): String = shape match
  case Circle(r) if r > 10    => s"Large circle (r=$r)"
  case Circle(r)               => s"Small circle (r=$r)"
  case Rectangle(w, h) if w == h => s"Square (side=$w)"
  case Rectangle(w, h)         => s"Rectangle ${w}x${h}"
  case Triangle(b, h)          => s"Triangle (b=$b, h=$h)"

// ทดสอบ
val shapes = List(
  Circle(5.0),
  Circle(15.0),
  Rectangle(4.0, 4.0),
  Rectangle(3.0, 5.0),
  Triangle(6.0, 4.0)
)

for shape <- shapes do
  println(f"${describe(shape)}: area = ${area(shape)}%.2f")
```

### Nested Pattern Matching

```scala
case class Address(city: String, country: String)
case class Person(name: String, age: Int, address: Address)

def greeting(person: Person): String = person match
  case Person(name, age, Address(_, "Thailand")) if age < 18 =>
    s"สวัสดี $name! (เยาวชนไทย)"
  case Person(name, _, Address(_, "Thailand")) =>
    s"สวัสดี $name! (ผู้ใหญ่ไทย)"
  case Person(name, _, Address(city, country)) =>
    s"Hello $name from $city, $country!"

val people = List(
  Person("Alice", 25, Address("Bangkok", "Thailand")),
  Person("Bob", 15, Address("Chiang Mai", "Thailand")),
  Person("Charlie", 30, Address("New York", "USA"))
)

people.foreach(p => println(greeting(p)))
```

### Pattern Matching กับ Collections

```scala
// List patterns
def describeList(list: List[Int]): String = list match
  case Nil           => "empty"
  case x :: Nil      => s"single element: $x"
  case x :: y :: Nil => s"two elements: $x and $y"
  case x :: rest     => s"starts with $x, then ${rest.length} more"

println(describeList(List()))          // empty
println(describeList(List(1)))         // single element: 1
println(describeList(List(1, 2)))      // two elements: 1 and 2
println(describeList(List(1, 2, 3)))   // starts with 1, then 2 more

// Tuple patterns
def processTuple(t: (Int, String)): String = t match
  case (0, msg) => s"Zero with message: $msg"
  case (n, msg) if n > 0 => s"Positive $n: $msg"
  case (n, msg) => s"Negative $n: $msg"
```

---

## try-catch-finally

### Exception Handling พื้นฐาน

```scala
// try-catch-finally
def divide(a: Int, b: Int): Int =
  try
    a / b
  catch
    case e: ArithmeticException =>
      println(s"Error: ${e.getMessage}")
      0
  finally
    println("Division attempted")

println(divide(10, 2))   // 5
println(divide(10, 0))   // Error: / by zero, then 0

// try เป็น expression
val result = try
  "42".toInt
catch
  case _: NumberFormatException => 0

println(result)  // 42
```

### Multiple Exception Types

```scala
import java.io.*

def readFile(filename: String): String =
  try
    val source = scala.io.Source.fromFile(filename)
    val content = source.mkString
    source.close()
    content
  catch
    case e: FileNotFoundException =>
      s"File not found: $filename"
    case e: IOException =>
      s"IO Error: ${e.getMessage}"
    case e: Exception =>
      s"Unexpected error: ${e.getMessage}"

// การจัดการ exception หลายอย่างพร้อมกัน
def parseInput(input: String): Int =
  try
    input.trim.toInt
  catch
    case _: NumberFormatException | _: NullPointerException =>
      throw new IllegalArgumentException(s"Invalid input: '$input'")
```

### Custom Exceptions

```scala
// สร้าง custom exceptions
class ValidationException(message: String, val field: String)
    extends Exception(message)

class DatabaseException(message: String, val query: String, cause: Throwable)
    extends Exception(message, cause)

// ใช้ custom exceptions
def validateAge(age: Int): Int =
  if age < 0 then
    throw ValidationException(s"Age cannot be negative: $age", "age")
  else if age > 150 then
    throw ValidationException(s"Age seems too high: $age", "age")
  else
    age

try
  validateAge(-5)
catch
  case e: ValidationException =>
    println(s"Validation failed on field '${e.field}': ${e.getMessage}")
```

### Try, Success, Failure

```scala
import scala.util.{Try, Success, Failure}

// Try เป็น functional error handling ที่ดีกว่า try-catch
def safeDivide(a: Int, b: Int): Try[Int] =
  Try(a / b)

println(safeDivide(10, 2))  // Success(5)
println(safeDivide(10, 0))  // Failure(java.lang.ArithmeticException: / by zero)

// การใช้งาน
safeDivide(10, 2) match
  case Success(result) => println(s"Result: $result")
  case Failure(e)      => println(s"Error: ${e.getMessage}")

// Chaining ด้วย map/flatMap
val result = Try("42".toInt)
  .map(_ * 2)
  .map(n => s"Result: $n")
  .getOrElse("Invalid number")

println(result)  // Result: 84

// recover
val recovered = Try("not a number".toInt)
  .recover {
    case _: NumberFormatException => 0
  }

println(recovered)  // Success(0)

// recoverWith
val result2 = Try("not a number".toInt)
  .recoverWith {
    case _: NumberFormatException => Try(0)
  }
```

### finally block

```scala
// finally รันเสมอไม่ว่าจะ success หรือ failure
def withResource[R, T](resource: R)(use: R => T)(close: R => Unit): T =
  try
    use(resource)
  finally
    close(resource)

// ตัวอย่างการใช้
def processFile(filename: String): Unit =
  val source = scala.io.Source.fromFile(filename)
  try
    source.getLines().foreach(println)
  finally
    source.close()  // รันเสมอ แม้จะ throw exception

// Using pattern (ดีกว่า)
// Scala มี scala.util.Using ตั้งแต่ 2.13
import scala.util.Using

def processFile2(filename: String): Try[Unit] =
  Using(scala.io.Source.fromFile(filename)) { source =>
    source.getLines().foreach(println)
  }
```

---

## break และ continue (ไม่มีใน Scala!)

### Scala ไม่มี break/continue

```scala
// Scala ไม่มี break หรือ continue แบบ Java/C
// แต่มีทางเลือกที่ดีกว่า:

// 1. ใช้ return (เฉพาะใน def)
def findFirst(list: List[Int], target: Int): Int =
  for item <- list do
    if item == target then return item
  -1

// 2. ใช้ takeWhile/dropWhile
val numbers = List(1, 3, 5, 2, 8, 4, 7)
val beforeEven = numbers.takeWhile(_ % 2 != 0)
println(beforeEven)  // List(1, 3, 5)

// 3. ใช้ find สำหรับ early exit
val firstEven = numbers.find(_ % 2 == 0)
println(firstEven)  // Some(2)

// 4. ใช้ recursion แทน loop
def findTarget(list: List[Int], target: Int): Boolean = list match
  case Nil => false
  case head :: _ if head == target => true
  case _ :: tail => findTarget(tail, target)

// 5. Breakable (ถ้าจำเป็นจริงๆ)
import scala.util.control.Breaks.*

var found = false
breakable {
  for i <- 1 to 100 do
    if i * i > 50 then
      println(s"Found: $i")
      found = true
      break()
}
```

### Pattern แทน break/continue

```scala
// แทน continue: ใช้ filter
val data = List(1, -2, 3, -4, 5)

// แบบ continue (ไม่ทำบรรทัดนี้ถ้าลบ)
// for item <- data do
//   if item < 0 then continue  // ❌ ไม่มีใน Scala
//   println(item * 2)

// ✅ ใช้ filter แทน
for item <- data.filter(_ > 0) do
  println(item * 2)

// หรือ withFilter (lazy, ไม่สร้าง collection กลาง)
for
  item <- data
  if item > 0
do
  println(item * 2)

// แทน break: ใช้ functional operations
val items = (1 to 1000).toList

// ❌ แบบ imperative (ต้องการ break)
// var sum = 0
// for i <- items do
//   sum += i
//   if sum > 100 then break

// ✅ แบบ functional
val partialSum = items
  .scanLeft(0)(_ + _)    // running sum
  .takeWhile(_ <= 100)
  .lastOption
  .getOrElse(0)

println(partialSum)  // 91 (sum ก่อนเกิน 100)
```

---

## Tail Recursion

### ปัญหาของ Recursion ปกติ

```scala
// ❌ ปัญหา: Stack Overflow สำหรับ input ขนาดใหญ่
def sum(n: Int): Long =
  if n <= 0 then 0L
  else n + sum(n - 1)

// sum(10000)    // StackOverflowError!
```

### Tail Recursion: ทางออก

```scala
import scala.annotation.tailrec

// ✅ Tail recursive: Scala แปลงเป็น loop อัตโนมัติ
@tailrec
def sumTail(n: Int, acc: Long = 0): Long =
  if n <= 0 then acc
  else sumTail(n - 1, acc + n)

println(sumTail(1000000))  // 500000500000 (ไม่ overflow!)

// @tailrec annotation ช่วย verify ว่า recursive จริงๆ
// @tailrec
// def notTailRecursive(n: Int): Int =
//   if n <= 0 then 0
//   else 1 + notTailRecursive(n - 1)  // ❌ Error: not tail recursive
```

### ตัวอย่าง Tail Recursion

```scala
import scala.annotation.tailrec

// Fibonacci แบบ tail recursive
@tailrec
def fibTail(n: Int, a: BigInt = 0, b: BigInt = 1): BigInt =
  if n == 0 then a
  else fibTail(n - 1, b, a + b)

println(fibTail(100))   // 354224848179261915075
println(fibTail(1000))  // (ตัวเลขขนาดใหญ่มาก)

// Factorial
@tailrec
def factorial(n: Int, acc: BigInt = 1): BigInt =
  if n <= 1 then acc
  else factorial(n - 1, acc * n)

println(factorial(50))  // 30414093201713378043612608166979581188299763898377856000000000000

// Reverse list
@tailrec
def reverseList[A](list: List[A], acc: List[A] = Nil): List[A] = list match
  case Nil          => acc
  case head :: tail => reverseList(tail, head :: acc)

println(reverseList(List(1, 2, 3, 4, 5)))  // List(5, 4, 3, 2, 1)

// Binary search
@tailrec
def binarySearch(arr: Array[Int], target: Int, low: Int = 0, high: Int = -1): Int =
  val h = if high == -1 then arr.length - 1 else high
  if low > h then -1
  else
    val mid = (low + h) / 2
    if arr(mid) == target then mid
    else if arr(mid) < target then binarySearch(arr, target, mid + 1, h)
    else binarySearch(arr, target, low, mid - 1)

val sorted = Array(1, 3, 5, 7, 9, 11, 13, 15, 17, 19)
println(binarySearch(sorted, 13))  // 6
println(binarySearch(sorted, 8))   // -1
```

### Mutual Recursion

```scala
// ฟังก์ชันที่เรียกกันไปมา
def isEven(n: Int): Boolean =
  if n == 0 then true
  else isOdd(n - 1)

def isOdd(n: Int): Boolean =
  if n == 0 then false
  else isEven(n - 1)

// ❌ อันตราย: stack overflow สำหรับ n ใหญ่
// println(isEven(10000))

// ✅ ใช้ trampolining
import scala.util.control.TailCalls.*

def isEvenT(n: Int): TailRec[Boolean] =
  if n == 0 then done(true)
  else tailcall(isOddT(n - 1))

def isOddT(n: Int): TailRec[Boolean] =
  if n == 0 then done(false)
  else tailcall(isEvenT(n - 1))

println(isEvenT(100000).result)  // true
println(isOddT(100001).result)   // true
```

---

## ตัวอย่างโปรแกรมครบ: Text Analysis

```scala
@main def textAnalysis(): Unit =
  val text = """
    Scala is a powerful programming language that combines
    object-oriented and functional programming paradigms.
    It runs on the JVM and is fully interoperable with Java.
    Scala code is concise, expressive, and type-safe.
    Many companies use Scala for big data processing and backend services.
  """.trim

  // Count words
  val words = text.split("\\s+").toList
  val wordCount = words.length
  println(s"Word count: $wordCount")

  // Word frequency
  val frequency = words
    .map(_.toLowerCase.replaceAll("[^a-z]", ""))
    .filterNot(_.isEmpty)
    .groupBy(identity)
    .view.mapValues(_.length)
    .toMap
    .toList
    .sortBy(-_._2)  // sort by frequency descending

  println("\nTop 10 words:")
  for (word, count) <- frequency.take(10) do
    println(f"  $word%-15s: $count")

  // Find sentences
  val sentences = text.split("[.!?]+").map(_.trim).filter(_.nonEmpty)
  println(s"\nSentence count: ${sentences.length}")

  // Average sentence length
  val avgLength = sentences.map(_.split("\\s+").length).sum.toDouble / sentences.length
  println(f"Average sentence length: $avgLength%.1f words")

  // Pattern matching for statistics
  val stats = Map(
    "Total words" -> wordCount,
    "Unique words" -> frequency.length,
    "Sentences" -> sentences.length
  )

  println("\nStatistics:")
  for (label, value) <- stats do
    println(f"  $label%-20s: $value")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: if-else

```scala
// เขียน function ที่ตรวจสอบว่า year เป็น leap year หรือไม่
// กฎ:
// - หาร 4 ลงตัว → leap year
// - หาร 100 ลงตัว → ไม่ใช่ leap year
// - หาร 400 ลงตัว → leap year
def isLeapYear(year: Int): Boolean = ???

// ทดสอบ
val years = List(2000, 1900, 2024, 2023, 1600)
years.foreach(y => println(s"$y: ${if isLeapYear(y) then "leap" else "not leap"}"))
// 2000: leap, 1900: not leap, 2024: leap, 2023: not leap, 1600: leap
```

### แบบฝึกหัดที่ 2: for Comprehension

```scala
// หาเลข Pythagorean triple ทั้งหมด ที่ a, b, c <= 50
// (a^2 + b^2 = c^2 และ a <= b <= c)
val triples = for
  // TODO: generators และ guards
yield (???, ???, ???)

println(triples.take(5))
// List((3,4,5), (5,12,13), (6,8,10), (7,24,25), (8,15,17))
```

### แบบฝึกหัดที่ 3: Pattern Matching

```scala
sealed trait Expr
case class Num(value: Double) extends Expr
case class Add(left: Expr, right: Expr) extends Expr
case class Sub(left: Expr, right: Expr) extends Expr
case class Mul(left: Expr, right: Expr) extends Expr
case class Div(left: Expr, right: Expr) extends Expr

// TODO: implement eval
def eval(expr: Expr): Double = expr match
  case Num(v)        => ???
  case Add(l, r)     => ???
  case Sub(l, r)     => ???
  case Mul(l, r)     => ???
  case Div(l, r)     => ???

// ทดสอบ: (3 + 4) * (10 - 2) / 4
val expr = Div(
  Mul(Add(Num(3), Num(4)), Sub(Num(10), Num(2))),
  Num(4)
)
println(eval(expr))  // 14.0
```

### แบบฝึกหัดที่ 4: Tail Recursion

```scala
// เขียน tail recursive function สำหรับ:

// 1. Power (a^n)
@tailrec
def power(base: Double, exp: Int, acc: Double = 1.0): Double = ???

// 2. Flatten nested list
def flatten[A](list: List[Any]): List[A] = ???

// ทดสอบ
println(power(2, 10))     // 1024.0
println(power(3, 5))      // 243.0
println(flatten[Int](List(1, List(2, 3), List(4, List(5, 6)))))
// List(1, 2, 3, 4, 5, 6)
```

**เฉลย แบบฝึกหัดที่ 1:**

```scala
def isLeapYear(year: Int): Boolean =
  (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0)
```

**เฉลย แบบฝึกหัดที่ 2:**

```scala
val triples = for
  a <- 1 to 50
  b <- a to 50
  c <- b to 50
  if a * a + b * b == c * c
yield (a, b, c)

println(triples.take(5))
```

**เฉลย แบบฝึกหัดที่ 3:**

```scala
def eval(expr: Expr): Double = expr match
  case Num(v)    => v
  case Add(l, r) => eval(l) + eval(r)
  case Sub(l, r) => eval(l) - eval(r)
  case Mul(l, r) => eval(l) * eval(r)
  case Div(l, r) =>
    val divisor = eval(r)
    if divisor == 0 then throw ArithmeticException("Division by zero")
    else eval(l) / divisor
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ if-else เป็น expression ที่คืนค่า
- ✅ while และ do-while loops
- ✅ for loops กับ Range, Collections, Guards
- ✅ for comprehensions ด้วย yield และ monadic composition
- ✅ match expression สำหรับ pattern matching
- ✅ try-catch-finally และ Try[T]
- ✅ ทำไม Scala ไม่มี break/continue และทางเลือกที่ดีกว่า
- ✅ Tail recursion กับ @tailrec annotation

## ขั้นตอนถัดไป

ใน [Part 06: ฟังก์ชัน](part-06-functions.md) เราจะเรียนรู้:
- Function definitions ครบทุกรูปแบบ
- Default และ named parameters
- Varargs
- Higher-order functions เบื้องต้น
- Closures
- Partial application และ currying

---

*[← Part 04: ตัวแปรและค่าคงที่](part-04-variables.md) | [Part 06: ฟังก์ชัน →](part-06-functions.md)*
