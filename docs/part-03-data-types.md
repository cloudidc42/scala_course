# Part 03: ประเภทข้อมูล (Data Types)

## สารบัญ
1. [Type System ของ Scala](#type-system-ของ-scala)
2. [Numeric Types](#numeric-types)
3. [Boolean](#boolean)
4. [Char](#char)
5. [String](#string)
6. [Unit และ Null](#unit-และ-null)
7. [Nothing และ Any](#nothing-และ-any)
8. [Type Inference](#type-inference)
9. [Type Conversion](#type-conversion)
10. [Literal Types (Scala 3)](#literal-types)
11. [Union Types (Scala 3)](#union-types)
12. [Intersection Types (Scala 3)](#intersection-types)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Type System ของ Scala

### Scala Type Hierarchy

```
              Any
             /   \
           AnyVal  AnyRef (= java.lang.Object)
          /  |  \      \
       Int Double Boolean ... (Java/Scala Classes)
         \   |  /              \
          Nothing            Null
```

**รายละเอียด:**

```
Any
├── AnyVal (value types - ไม่ใช่ object บน heap)
│   ├── Int       (32-bit integer)
│   ├── Long      (64-bit integer)
│   ├── Short     (16-bit integer)
│   ├── Byte      (8-bit integer)
│   ├── Float     (32-bit floating point)
│   ├── Double    (64-bit floating point)
│   ├── Char      (16-bit Unicode character)
│   ├── Boolean   (true/false)
│   └── Unit      (เหมือน void)
└── AnyRef (reference types - เป็น object บน heap)
    ├── String
    ├── List
    ├── Option
    └── ... (ทุก class อื่นๆ)

Nothing  = subtype ของทุก type (ไม่มีค่า)
Null     = subtype ของทุก AnyRef type (มีค่าเดียวคือ null)
```

### ทำไม Type System ถึงสำคัญ?

```scala
// Type safety ป้องกัน bug ตั้งแต่ compile time
val x: Int = "hello"  // ❌ Error: String ไม่ใช่ Int

// แต่ Scala มี implicit conversion ในบางกรณี
val bigNumber: Long = 1000000000000L  // ✅

// Type ช่วยให้โค้ดชัดเจน
def processAge(age: Int): String = ???
def processAge(age: String): String = ???  // คนละ function
```

---

## Numeric Types

### Int (ใช้บ่อยที่สุด)

```scala
// Int: 32-bit signed integer
// ช่วงค่า: -2,147,483,648 ถึง 2,147,483,647

val a: Int = 42
val b: Int = -100
val c: Int = 0

// Literals
val decimal = 1000000      // ทศนิยม
val hex = 0xFF             // hexadecimal = 255
val octal = 0o17           // octal (Scala 3) = 15
val binary = 0b1010        // binary = 10

// Underscore สำหรับ readability
val million = 1_000_000
val hexColor = 0xFF_AA_BB

println(Int.MaxValue)      // 2147483647
println(Int.MinValue)      // -2147483648
```

### Long

```scala
// Long: 64-bit signed integer
// ช่วงค่า: -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807

val bigNum: Long = 9_223_372_036_854_775_807L  // ต้องมี L ท้าย
val population: Long = 8_000_000_000L

println(Long.MaxValue)     // 9223372036854775807
println(Long.MinValue)     // -9223372036854775808

// Int ไม่พอ? ใช้ Long
val intOverflow = Int.MaxValue + 1   // -2147483648 (overflow!)
val longSafe: Long = Int.MaxValue.toLong + 1L  // 2147483648
```

### Short และ Byte

```scala
// Short: 16-bit signed integer (-32,768 ถึง 32,767)
val s: Short = 1000
println(Short.MaxValue)  // 32767

// Byte: 8-bit signed integer (-128 ถึง 127)
val b: Byte = 127
println(Byte.MaxValue)   // 127
println(Byte.MinValue)   // -128

// ใช้กับ binary data หรือ protocol buffers
val header: Array[Byte] = Array(0x48, 0x65, 0x6C, 0x6C, 0x6F)
println(new String(header))  // Hello
```

### Double (floating point)

```scala
// Double: 64-bit IEEE 754 floating point (แนะนำสำหรับ decimal)
val pi: Double = 3.14159265358979
val e = 2.71828182845905  // type inference ให้ Double

// Scientific notation
val avogadro = 6.022e23
val electronCharge = 1.6e-19

println(Double.MaxValue)   // 1.7976931348623157E308
println(Double.MinValue)   // 5.0E-324 (ค่าบวกน้อยที่สุด)
println(Double.PositiveInfinity)  // Infinity
println(Double.NegativeInfinity)  // -Infinity
println(Double.NaN)               // NaN (Not a Number)

// Special values
println(1.0 / 0.0)    // Infinity
println(-1.0 / 0.0)   // -Infinity
println(0.0 / 0.0)    // NaN

// ระวัง floating point precision
println(0.1 + 0.2)        // 0.30000000000000004 (ไม่ใช่ 0.3!)
println(0.1 + 0.2 == 0.3) // false!
```

### Float

```scala
// Float: 32-bit IEEE 754 floating point (ความแม่นยำน้อยกว่า Double)
val f: Float = 3.14f  // ต้องมี f ท้าย
println(Float.MaxValue)  // 3.4028235E38

// เมื่อไหรใช้ Float แทน Double?
// - ประหยัด memory เมื่อมีข้อมูลจำนวนมาก
// - graphics programming (OpenGL, GPU)
// - machine learning weights
```

### BigInt และ BigDecimal

```scala
// BigInt: integer ที่ใหญ่ไม่จำกัด
val factorial100 = BigInt(1).to(100).product
println(factorial100)
// 9332621544394415268169923885626670049071596826438162146859296389521759999322991560894146397615651828625369792082722375825118521091686400000000000000000000000

// เปรียบเทียบ
val n: BigInt = 123456789012345678901234567890L  // Error: too large for Long
val m: BigInt = BigInt("123456789012345678901234567890")  // ✅

// BigDecimal: decimal ที่แม่นยำสูง (สำคัญมากสำหรับการเงิน!)
val price = BigDecimal("19.99")
val tax = BigDecimal("0.07")
val total = price * (1 + tax)
println(total)  // 21.3893

// ระวัง: อย่าสร้าง BigDecimal จาก Double!
val wrong = BigDecimal(0.1 + 0.2)
println(wrong)  // 0.30000000000000004 (ยังผิด!)

val correct = BigDecimal("0.1") + BigDecimal("0.2")
println(correct)  // 0.3 (ถูกต้อง!)

// สำหรับระบบการเงิน
val amount = BigDecimal("1234567.89")
val interestRate = BigDecimal("0.045")
val interest = (amount * interestRate).setScale(2, BigDecimal.RoundingMode.HALF_UP)
println(f"Interest: $$${interest}")
```

### Numeric Operations

```scala
// Arithmetic operators
val a = 10
val b = 3

println(a + b)   // 13
println(a - b)   // 7
println(a * b)   // 30
println(a / b)   // 3 (integer division)
println(a % b)   // 1 (remainder)

// Integer division truncates
println(7 / 2)   // 3 (ไม่ใช่ 3.5!)
println(7.0 / 2) // 3.5

// Comparison
println(a > b)   // true
println(a < b)   // false
println(a >= b)  // true
println(a <= b)  // false
println(a == b)  // false
println(a != b)  // true

// Bitwise operators
println(a & b)   // 2  (AND)
println(a | b)   // 11 (OR)
println(a ^ b)   // 9  (XOR)
println(~a)      // -11 (NOT)
println(a << 1)  // 20 (left shift)
println(a >> 1)  // 5  (right shift)
println(a >>> 1) // 5  (unsigned right shift)

// Math functions
import scala.math.*
println(abs(-5))     // 5
println(max(3, 7))   // 7
println(min(3, 7))   // 3
println(pow(2, 10))  // 1024.0
println(sqrt(144))   // 12.0
println(floor(3.7))  // 3.0
println(ceil(3.2))   // 4.0
println(round(3.5))  // 4
println(log(math.E)) // 1.0
println(log10(100))  // 2.0
```

---

## Boolean

```scala
// Boolean: true หรือ false
val isActive: Boolean = true
val isDeleted = false  // type inference

// Logical operators
val a = true
val b = false

println(a && b)  // false (AND)
println(a || b)  // true  (OR)
println(!a)      // false (NOT)

// Short-circuit evaluation
def expensive(): Boolean =
  println("computing...")
  true

val result1 = false && expensive()  // ไม่เรียก expensive()
val result2 = true || expensive()   // ไม่เรียก expensive()
val result3 = true && expensive()   // เรียก expensive()

// ใช้ Boolean
val temperature = 38.5
val hasSymptoms = true

val needsDoctor = temperature > 37.5 && hasSymptoms
println(s"Should see doctor: $needsDoctor")  // true

// Boolean expressions
val score = 85
val isPassing = score >= 60
val isHighScore = score >= 90
val grade = if isHighScore then "A" else if isPassing then "Pass" else "Fail"
```

---

## Char

```scala
// Char: single Unicode character (16-bit)
val c1: Char = 'A'
val c2: Char = '0'
val c3: Char = 'ก'  // Thai character
val c4: Char = '\n' // newline
val c5: Char = '\t' // tab
val c6: Char = '\\'  // backslash
val c7: Char = '\''  // single quote
val c8: Char = '\"'  // double quote

// Char เป็น number ได้
val a: Char = 'A'
println(a.toInt)   // 65
println((a + 1).toChar)  // B

// Unicode
val thai: Char = 'ก'  // ก
val heart: Char = '♥'  // ♥

// Char operations
val letter = 'h'
println(letter.toUpper)        // H
println(letter.isLetter)       // true
println(letter.isDigit)        // false
println(letter.isLowerCase)    // true
println(letter.isUpperCase)    // false
println(letter.isWhitespace)   // false

// แปลง Char เป็น String
val charAsString = c1.toString  // "A"
val charToStr = s"$c1"          // "A"
```

---

## String

### String Basics

```scala
// String: immutable sequence of characters
val s1 = "Hello, World!"
val s2: String = "สวัสดีโลก"

// String length
println(s1.length)   // 13

// Character access
println(s1(0))       // H
println(s1.charAt(0)) // H

// Substring
println(s1.substring(7, 12))  // World
println(s1.slice(7, 12))      // World

// Case conversion
println(s1.toUpperCase)  // HELLO, WORLD!
println(s1.toLowerCase)  // hello, world!
```

### String Operations

```scala
val s = "Hello, World!"

// Search
println(s.contains("World"))    // true
println(s.startsWith("Hello"))  // true
println(s.endsWith("!"))        // true
println(s.indexOf("o"))         // 4 (first occurrence)
println(s.lastIndexOf("o"))     // 8 (last occurrence)

// Trim
val padded = "   hello   "
println(padded.trim)      // "hello"
println(padded.strip)     // "hello" (Unicode-aware, Java 11+)
println(padded.stripLeading)  // "hello   "
println(padded.stripTrailing) // "   hello"

// Split and Join
val csv = "Alice,30,Engineer"
val parts = csv.split(",")
println(parts.mkString(" | "))  // Alice | 30 | Engineer

val words = Array("Scala", "is", "awesome")
println(words.mkString(" "))   // Scala is awesome

// Replace
val text = "Hello World Hello"
println(text.replace("Hello", "Hi"))         // Hi World Hi
println(text.replaceFirst("Hello", "Hi"))    // Hi World Hello
println(text.replaceAll("l", "L"))           // HeLLo WorLd HeLLo

// Regex replace
println("abc123def456".replaceAll("[0-9]+", "#"))  // abc#def#
```

### String Comparison

```scala
val s1 = "Hello"
val s2 = "Hello"
val s3 = "hello"

// == ใน Scala เปรียบเทียบค่า (ต่างจาก Java)
println(s1 == s2)    // true (value equality)
println(s1 == s3)    // false (case sensitive)

// Case-insensitive comparison
println(s1.equalsIgnoreCase(s3))  // true

// Comparison for sorting
println(s1.compareTo(s3))         // negative (H < h)
println(s1.compareToIgnoreCase(s3))  // 0 (equal)

// eq: reference equality (rarely needed)
val a = new String("Hello")
val b = new String("Hello")
println(a == b)    // true (value equality)
println(a eq b)    // false (different objects)
```

### String Formatting

```scala
// String.format (Java style)
val formatted = "Name: %s, Age: %d, Score: %.2f".format("Alice", 30, 95.5)
println(formatted)  // Name: Alice, Age: 30, Score: 95.50

// f-string (type-safe)
val name = "Alice"
val age = 30
val score = 95.5
println(f"Name: $name, Age: $age%d, Score: $score%.2f")

// Printf-style format specifiers
println(f"$age%5d")    // "   30" (right-align, width 5)
println(f"$age%-5d")   // "30   " (left-align, width 5)
println(f"$age%05d")   // "00030" (zero-pad, width 5)
println(f"$score%8.2f") // "   95.50"
```

### String Conversion

```scala
// String → Numeric
val numStr = "42"
val num: Int = numStr.toInt
val bigNum: Long = "9876543210".toLong
val decimal: Double = "3.14".toDouble
val flag: Boolean = "true".toBoolean

// ระวัง: อาจ throw exception
try
  val bad = "hello".toInt  // NumberFormatException!
catch
  case e: NumberFormatException =>
    println(s"Cannot convert: ${e.getMessage}")

// Safe conversion ด้วย Try
import scala.util.{Try, Success, Failure}
val result = Try("hello".toInt) match
  case Success(n) => n
  case Failure(_) => 0

println(result)  // 0

// Numeric → String
val n = 42
println(n.toString)        // "42"
println(n.toString(16))    // "2a" (hex)
println(n.toString(2))     // "101010" (binary)
println(s"$n")             // "42"
println(f"$n%d")           // "42"
```

### StringBuilder

```scala
// StringBuilder สำหรับ string ที่ต้องสร้างแบบ dynamic
val sb = new StringBuilder()

// Append
sb.append("Hello")
sb.append(", ")
sb.append("World")
sb.append("!")

println(sb.toString)  // Hello, World!

// Method chaining
val result = new StringBuilder()
  .append("Scala ")
  .append("is ")
  .append("awesome!")
  .toString

println(result)  // Scala is awesome!

// Insert, Delete, Replace
val sb2 = new StringBuilder("Hello World")
sb2.insert(5, ",")       // Hello, World
sb2.delete(5, 6)          // Hello World (ลบ comma)
sb2.replace(6, 11, "Scala")  // Hello Scala

println(sb2)  // Hello Scala
```

---

## Unit และ Null

### Unit

```scala
// Unit เหมือน void แต่เป็น type จริงๆ
def printHello(): Unit =
  println("Hello")  // ไม่มี return value

val result: Unit = printHello()
println(result)  // ()

// Unit value คือ ()
val u: Unit = ()
println(u)  // ()

// ฟังก์ชันที่ return type ไม่ระบุ มักคือ Unit
def doWork() =
  println("working...")
  // ไม่มี return value → Unit

// Unit ใน collections
val units: List[Unit] = List((), (), ())
println(units.length)  // 3
```

### Null (หลีกเลี่ยงถ้าเป็นไปได้!)

```scala
// Null เป็น subtype ของ AnyRef
// ใช้ null ได้กับ reference types แต่ไม่แนะนำ

var name: String = null  // ✅ compile ผ่าน แต่ไม่ดี!

// NullPointerException - ศัตรูตัวร้าย
// name.length  // ❌ NullPointerException!

// วิธีที่ดีกว่า: ใช้ Option
val safeName: Option[String] = None  // แทน null
val anotherName: Option[String] = Some("Alice")

// ตรวจสอบ null (ถ้าจำเป็นต้องใช้ null เช่น Java interop)
val javaResult: String = javaMethod()  // อาจ return null
val safeResult = Option(javaResult).getOrElse("default")

// ใน Scala 3 มี -Yexplicit-nulls flag ที่ทำให้ null ต้องระบุชัดเจน
```

---

## Nothing และ Any

### Nothing

```scala
// Nothing เป็น subtype ของทุก type
// ไม่มี instance ของ Nothing
// ใช้กับ:
// 1. ฟังก์ชันที่ throw exception เสมอ
// 2. ฟังก์ชันที่ loop ไม่จบ
// 3. ??? (stub implementation)

def fail(msg: String): Nothing =
  throw new RuntimeException(msg)

def infiniteLoop(): Nothing =
  while true do ()
  throw new AssertionError("unreachable")

// ??? คือ stub implementation ที่ return Nothing
def notImplementedYet(): Int = ???
// throw scala.NotImplementedError

// ประโยชน์ของ Nothing ใน type system
val list: List[Int] = List.empty  // List[Nothing] แล้ว subtype เป็น List[Int]
val opt: Option[String] = None    // None เป็น Option[Nothing]
```

### Any

```scala
// Any เป็น supertype ของทุก type
val anything: Any = 42
val anything2: Any = "hello"
val anything3: Any = List(1, 2, 3)
val anything4: Any = true

// AnyVal: supertype ของ value types
val value: AnyVal = 42
val value2: AnyVal = true
val value3: AnyVal = 3.14

// AnyRef: supertype ของ reference types (= java.lang.Object)
val ref: AnyRef = "hello"
val ref2: AnyRef = List(1, 2, 3)

// Pattern matching กับ Any
def describe(x: Any): String = x match
  case i: Int    => s"Integer: $i"
  case s: String => s"String: $s"
  case b: Boolean => s"Boolean: $b"
  case _         => s"Unknown: $x"

println(describe(42))       // Integer: 42
println(describe("hello"))  // String: hello
println(describe(true))     // Boolean: true
println(describe(3.14))     // Unknown: 3.14
```

---

## Type Inference

```scala
// Scala อนุมาน type โดยอัตโนมัติ
val x = 42          // Int
val y = 3.14        // Double
val s = "hello"     // String
val b = true        // Boolean

// Complex inference
val list = List(1, 2, 3)        // List[Int]
val mixed = List(1, "two", 3.0)  // List[Any] (เพราะต่าง type)

val map = Map("a" -> 1, "b" -> 2)  // Map[String, Int]

// Function return type inference
def double(x: Int) = x * 2      // return type: Int
def first[A](list: List[A]) = list.head  // return type: A

// ควร annotate type เมื่อ:
// 1. เพื่อความชัดเจน (public API)
// 2. เมื่อ inference ให้ type ที่ไม่ต้องการ
def processData(input: String): Option[Int] =  // ระบุ return type ชัด
  if input.forall(_.isDigit) then Some(input.toInt)
  else None

// Type annotation ช่วย catch bugs
val result: Double = 1 / 2  // ⚠️ = 0.0 ไม่ใช่ 0.5!
// เพราะ 1/2 เป็น Int division แล้วแปลงเป็น Double
val correct: Double = 1.0 / 2  // = 0.5 ✅
```

---

## Type Conversion

### Implicit Widening Conversion (ไม่มีใน Scala!)

```scala
// ต่างจาก Java, Scala ไม่มี implicit widening
val i: Int = 42
// val l: Long = i  // ❌ Error ใน Scala! (Java OK)
val l: Long = i.toLong  // ✅ ต้อง explicit

// Explicit conversions
val byte: Byte = 127
val short: Short = byte.toShort
val int: Int = short.toInt
val long: Long = int.toLong
val float: Float = long.toFloat
val double: Double = float.toDouble
val bigInt: BigInt = BigInt(long)
val bigDec: BigDecimal = BigDecimal(double)
```

### Numeric Conversion Methods

```scala
val n = 42

// toXxx methods
println(n.toByte)     // 42
println(n.toShort)    // 42
println(n.toInt)      // 42
println(n.toLong)     // 42L
println(n.toFloat)    // 42.0
println(n.toDouble)   // 42.0
println(n.toChar)     // '*' (ASCII 42)
println(n.toString)   // "42"

// String to number
println("42".toInt)
println("3.14".toDouble)
println("true".toBoolean)

// Overflow!
println(Int.MaxValue.toByte)  // -1 (overflow!)
println(300.toByte)           // 44 (overflow!)
```

### asInstanceOf และ isInstanceOf

```scala
// การ cast แบบ Java style (หลีกเลี่ยงถ้าเป็นไปได้)
val obj: Any = "Hello"

// isInstanceOf: ตรวจสอบ type
if obj.isInstanceOf[String] then
  val s = obj.asInstanceOf[String]  // cast
  println(s.toUpperCase)

// แนะนำ: ใช้ pattern matching แทน
obj match
  case s: String => println(s.toUpperCase)
  case _         => println("not a string")
```

---

## Literal Types

Scala 3 รองรับ Literal Types ที่แม่นยำมาก:

```scala
// Literal type คือ type ที่มีค่าเดียว
val one: 1 = 1
val hello: "hello" = "hello"
val flag: true = true

// ใช้กับ type aliases
type EnabledFlag = true
type DisabledFlag = false

// ใช้กับ function signature
def process(flag: true): Unit =
  println("Processing with flag enabled")

// Useful กับ singleton types
object Config:
  val maxRetries: 3 = 3  // type คือ literal 3 ไม่ใช่ Int

// Union ของ literal types
type Direction = "up" | "down" | "left" | "right"

def move(dir: Direction): Unit =
  println(s"Moving $dir")

move("up")    // ✅
move("down")  // ✅
// move("diagonal")  // ❌ compile error!
```

---

## Union Types

Scala 3 เพิ่ม Union Types (`A | B`):

```scala
// Union type: ค่าอาจเป็น type ใดก็ได้จาก union
def formatInput(input: Int | String): String =
  input match
    case i: Int    => s"Integer: $i"
    case s: String => s"String: $s"

println(formatInput(42))       // Integer: 42
println(formatInput("hello"))  // String: hello

// ใช้กับ Option-like patterns
type MaybeInt = Int | Null
def parseInt(s: String): MaybeInt =
  try s.toInt
  catch case _ => null

// Union กับ error handling
type Result[T] = T | Error

// Union types ใน practice
case class ApiError(code: Int, message: String)
case class NetworkError(cause: String)

type ServiceError = ApiError | NetworkError

def callService(): String | ServiceError =
  // ลองเรียก service
  if math.random() > 0.5 then "success"
  else ApiError(404, "Not found")

// Process result
callService() match
  case result: String         => println(s"Got: $result")
  case ApiError(code, msg)    => println(s"API Error $code: $msg")
  case NetworkError(cause)    => println(s"Network Error: $cause")
```

---

## Intersection Types

Scala 3 รองรับ Intersection Types (`A & B`):

```scala
// Intersection type: ต้องเป็นทั้ง type A และ type B
trait Printable:
  def print(): Unit

trait Serializable:
  def serialize(): String

// type ที่ต้องเป็นทั้ง Printable และ Serializable
def processItem(item: Printable & Serializable): Unit =
  item.print()
  println(item.serialize())

// สร้าง class ที่ implement ทั้งสอง trait
class Document(content: String) extends Printable, Serializable:
  def print(): Unit = println(content)
  def serialize(): String = s"""{"content": "$content"}"""

val doc = Document("Hello World")
processItem(doc)
// Hello World
// {"content": "Hello World"}

// Intersection กับ type aliases
type FullAccess = Readable & Writable & Executable

trait Readable:
  def read(): String
trait Writable:
  def write(data: String): Unit
trait Executable:
  def execute(): Unit
```

---

## Type Aliases

```scala
// ตั้งชื่อ type ให้กระชับ
type UserId = Int
type UserName = String
type UserMap = Map[UserId, UserName]

val users: UserMap = Map(
  1 -> "Alice",
  2 -> "Bob",
  3 -> "Charlie"
)

// Opaque Types (Scala 3) - เหมือน type alias แต่ type-safe กว่า
opaque type Meters = Double
opaque type Kilograms = Double

object Meters:
  def apply(v: Double): Meters = v
  extension (m: Meters) def value: Double = m

object Kilograms:
  def apply(v: Double): Kilograms = v
  extension (kg: Kilograms) def value: Double = kg

val distance = Meters(100.0)
val weight = Kilograms(70.0)

// ไม่สามารถสับสนระหว่าง Meters และ Kilograms
// val wrong: Meters = weight  // ❌ Error!
val d: Meters = distance  // ✅
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Numeric Types

```scala
@main def numericExercise(): Unit =
  // 1. หา factorial ของ 20 ด้วย Long
  // TODO: คำนวณ 20! ด้วย Long

  // 2. คำนวณ BMI
  val weightKg = 70.0
  val heightM = 1.75
  // TODO: คำนวณ BMI = weight / (height * height)
  // แสดงผล f"BMI = $bmi%.2f"

  // 3. แสดงเลขในฐานต่างๆ
  val num = 255
  // TODO: แสดง 255 ในฐาน 2, 8, 10, 16
```

### แบบฝึกหัดที่ 2: String Operations

```scala
@main def stringExercise(): Unit =
  val text = "The quick brown fox jumps over the lazy dog"

  // 1. นับจำนวนคำ
  // TODO: split ด้วย space และนับ

  // 2. แสดงคำที่ยาวที่สุด
  // TODO: หาคำที่ยาวที่สุด

  // 3. แปลงเป็น Title Case
  // TODO: ทำให้ตัวอักษรแรกของทุกคำเป็น uppercase

  // 4. นับจำนวนตัวอักษรแต่ละตัว (histogram)
  // TODO: นับแต่ละ character ที่ไม่ใช่ space
```

### แบบฝึกหัดที่ 3: Type System

```scala
@main def typeExercise(): Unit =
  // 1. สร้าง function ที่รับ Int | String และ format ให้เป็น String
  // TODO: def format(value: Int | String): String = ???

  // 2. ทดสอบ type inference
  val a = 5
  val b = 2.0
  // val c = a + b  // c มี type อะไร?

  // 3. BigDecimal arithmetic
  // คำนวณ: (1.1 + 2.2) vs BigDecimal("1.1") + BigDecimal("2.2")
  // TODO: แสดงความแตกต่าง
```

**เฉลย แบบฝึกหัดที่ 1:**

```scala
@main def numericExercise(): Unit =
  // 1. Factorial 20
  val factorial20 = (1L to 20L).product
  println(s"20! = $factorial20")  // 2432902008176640000

  // 2. BMI
  val weightKg = 70.0
  val heightM = 1.75
  val bmi = weightKg / (heightM * heightM)
  println(f"BMI = $bmi%.2f")  // BMI = 22.86

  val bmiCategory = if bmi < 18.5 then "Underweight"
                    else if bmi < 25 then "Normal"
                    else if bmi < 30 then "Overweight"
                    else "Obese"
  println(s"Category: $bmiCategory")

  // 3. Number bases
  val num = 255
  println(s"Decimal: $num")
  println(s"Binary: ${num.toBinaryString}")    // 11111111
  println(s"Octal: ${num.toOctalString}")      // 377
  println(s"Hex: ${num.toHexString}")          // ff
  println(s"Hex upper: ${num.toHexString.toUpperCase}")  // FF
```

**เฉลย แบบฝึกหัดที่ 2:**

```scala
@main def stringExercise(): Unit =
  val text = "The quick brown fox jumps over the lazy dog"

  // 1. นับคำ
  val words = text.split(" ")
  println(s"จำนวนคำ: ${words.length}")  // 9

  // 2. คำที่ยาวที่สุด
  val longestWord = words.maxBy(_.length)
  println(s"คำที่ยาวที่สุด: $longestWord (${longestWord.length} ตัว)")  // jumps (5)

  // 3. Title Case
  val titleCase = words.map(w => w.capitalize).mkString(" ")
  println(s"Title Case: $titleCase")

  // 4. Character histogram
  val histogram = text.filterNot(_ == ' ')
    .groupBy(identity)
    .view.mapValues(_.length)
    .toList
    .sortBy(_._1)

  println("Character histogram:")
  histogram.foreach { case (char, count) =>
    println(f"'$char': $count")
  }
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Type hierarchy ของ Scala (Any, AnyVal, AnyRef, Nothing, Null)
- ✅ Numeric types: Int, Long, Short, Byte, Float, Double, BigInt, BigDecimal
- ✅ Boolean และ Logical operators
- ✅ Char และ Unicode
- ✅ String operations ทั้งหมด
- ✅ Unit และ Null
- ✅ Nothing และ Any
- ✅ Type inference
- ✅ Type conversion (explicit)
- ✅ Literal types, Union types, Intersection types (Scala 3)
- ✅ Type aliases และ Opaque types

## ขั้นตอนถัดไป

ใน [Part 04: ตัวแปรและค่าคงที่](part-04-variables.md) เราจะเรียนรู้:
- `val` vs `var`
- Lazy val
- Local variables vs fields
- Immutability และ why it matters
- Variable shadowing

---

*[← Part 02: ไวยากรณ์พื้นฐาน](part-02-basic-syntax.md) | [Part 04: ตัวแปรและค่าคงที่ →](part-04-variables.md)*
