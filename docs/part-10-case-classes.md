# Part 10: Case Classes และ Pattern Matching

## สารบัญ
1. [Case Classes](#case-classes)
2. [Pattern Matching ขั้นสูง](#pattern-matching-ขั้นสูง)
3. [Sealed Traits และ ADTs](#sealed-traits-และ-adts)
4. [Extractors](#extractors)
5. [Pattern Matching กับ Collections](#pattern-matching-กับ-collections)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Case Classes

### Case Class พื้นฐาน

```scala
// Case class สร้าง boilerplate ให้อัตโนมัติ:
// - equals และ hashCode
// - toString
// - copy method
// - companion object กับ apply/unapply
// - serializable

case class Person(name: String, age: Int, email: String)

// ไม่ต้องใช้ new
val alice = Person("Alice", 30, "alice@example.com")

// toString อัตโนมัติ
println(alice)  // Person(Alice,30,alice@example.com)

// equals อัตโนมัติ (value-based)
val alice2 = Person("Alice", 30, "alice@example.com")
println(alice == alice2)  // true

// copy method
val olderAlice = alice.copy(age = 31)
val newEmail = alice.copy(email = "newalice@example.com")
println(olderAlice)  // Person(Alice,31,alice@example.com)
println(newEmail)    // Person(Alice,30,newalice@example.com)

// Pattern matching อัตโนมัติ
alice match
  case Person(name, age, email) =>
    println(s"$name is $age years old")
```

### Case Class กับ Default Values

```scala
case class Config(
  host: String = "localhost",
  port: Int = 8080,
  debug: Boolean = false,
  maxConnections: Int = 100,
  timeout: Int = 30
)

val dev = Config(debug = true)
val prod = Config(
  host = "api.example.com",
  port = 443,
  maxConnections = 1000
)

println(dev)   // Config(localhost,8080,true,100,30)
println(prod)  // Config(api.example.com,443,false,1000,30)
```

### Case Class Methods

```scala
case class BoundingBox(minX: Double, minY: Double, maxX: Double, maxY: Double):
  def width: Double = maxX - minX
  def height: Double = maxY - minY
  def area: Double = width * height
  def center: (Double, Double) = ((minX + maxX) / 2, (minY + maxY) / 2)

  def intersects(other: BoundingBox): Boolean =
    minX < other.maxX && maxX > other.minX &&
    minY < other.maxY && maxY > other.minY

  def contains(x: Double, y: Double): Boolean =
    x >= minX && x <= maxX && y >= minY && y <= maxY

  def union(other: BoundingBox): BoundingBox =
    BoundingBox(
      math.min(minX, other.minX),
      math.min(minY, other.minY),
      math.max(maxX, other.maxX),
      math.max(maxY, other.maxY)
    )

val box1 = BoundingBox(0, 0, 10, 5)
val box2 = BoundingBox(5, 2, 15, 8)

println(f"Box1 area: ${box1.area}%.1f")         // 50.0
println(s"Intersects: ${box1.intersects(box2)}")  // true
println(s"Union: ${box1.union(box2)}")            // BoundingBox(0.0,0.0,15.0,8.0)
```

### Nested Case Classes

```scala
case class Address(
  street: String,
  city: String,
  country: String,
  postalCode: String
)

case class ContactInfo(
  phone: Option[String] = None,
  email: Option[String] = None,
  website: Option[String] = None
)

case class Company(
  name: String,
  address: Address,
  contact: ContactInfo,
  employeeCount: Int
)

val company = Company(
  name = "TechCorp",
  address = Address("123 Tech St", "Bangkok", "Thailand", "10110"),
  contact = ContactInfo(
    phone = Some("+66-2-000-0000"),
    email = Some("info@techcorp.com")
  ),
  employeeCount = 500
)

// Nested pattern matching
company match
  case Company(
    name,
    Address(_, city, country, _),
    ContactInfo(_, Some(email), _),
    size
  ) =>
    println(s"$name in $city, $country")
    println(s"Email: $email, Size: $size")
```

### Case Class Inheritance

```scala
// Abstract base (ไม่ใช่ case class เพื่อหลีกเลี่ยง equality issues)
abstract class Event:
  def timestamp: Long = System.currentTimeMillis()

// Case classes ที่ extend abstract class
case class UserCreated(userId: Int, name: String) extends Event
case class UserUpdated(userId: Int, field: String, newValue: String) extends Event
case class UserDeleted(userId: Int) extends Event

def handleEvent(event: Event): String = event match
  case UserCreated(id, name)          => s"New user: $name (ID: $id)"
  case UserUpdated(id, field, value)  => s"User $id updated $field to $value"
  case UserDeleted(id)                => s"User $id deleted"

val events = List(
  UserCreated(1, "Alice"),
  UserUpdated(1, "email", "newalice@example.com"),
  UserDeleted(2)
)

events.foreach(e => println(handleEvent(e)))
```

---

## Pattern Matching ขั้นสูง

### Guard Conditions

```scala
def classify(n: Int): String = n match
  case 0                => "zero"
  case n if n < 0       => s"negative: $n"
  case n if n % 2 == 0  => s"positive even: $n"
  case n                => s"positive odd: $n"

println(classify(0))    // zero
println(classify(-5))   // negative: -5
println(classify(4))    // positive even: 4
println(classify(7))    // positive odd: 7
```

### Type Patterns

```scala
def processAny(value: Any): String = value match
  case n: Int             => s"Int: $n (doubled: ${n * 2})"
  case s: String          => s"String: '$s' (length: ${s.length})"
  case d: Double          => f"Double: $d%.4f"
  case b: Boolean         => s"Boolean: $b"
  case list: List[_]      => s"List with ${list.size} elements"
  case Some(v)            => s"Some($v)"
  case None               => "None"
  case null               => "null"
  case _                  => s"Unknown: ${value.getClass.getSimpleName}"

println(processAny(42))          // Int: 42 (doubled: 84)
println(processAny("hello"))     // String: 'hello' (length: 5)
println(processAny(3.14))        // Double: 3.1400
println(processAny(List(1,2,3))) // List with 3 elements
println(processAny(Some("hi")))  // Some(hi)
println(processAny(None))        // None
```

### Tuple Patterns

```scala
def describePair(pair: (Any, Any)): String = pair match
  case (0, 0)             => "origin"
  case (x: Int, y: Int)   => s"int pair: ($x, $y)"
  case (s: String, n: Int) => s"string-int: ($s, $n)"
  case (a, b)              => s"generic pair: ($a, $b)"

println(describePair((0, 0)))       // origin
println(describePair((3, 4)))       // int pair: (3, 4)
println(describePair(("hi", 5)))    // string-int: (hi, 5)
println(describePair((1.0, "x")))   // generic pair: (1.0, x)

// Triple matching
def describeTriple(t: (Int, Int, Int)): String = t match
  case (0, 0, 0) => "zero point"
  case (x, 0, 0) => s"on x-axis: $x"
  case (0, y, 0) => s"on y-axis: $y"
  case (0, 0, z) => s"on z-axis: $z"
  case (x, y, z) => s"point: ($x, $y, $z)"
```

### Complex Nested Patterns

```scala
case class Order(id: Int, customer: Customer, items: List[OrderItem], status: OrderStatus)
case class Customer(id: Int, name: String, tier: CustomerTier)
case class OrderItem(productId: Int, quantity: Int, price: Double)

sealed trait CustomerTier
case object Bronze extends CustomerTier
case object Silver extends CustomerTier
case object Gold extends CustomerTier

sealed trait OrderStatus
case object Pending extends OrderStatus
case object Processing extends OrderStatus
case class Shipped(trackingNumber: String) extends OrderStatus
case object Delivered extends OrderStatus
case class Cancelled(reason: String) extends OrderStatus

def processOrder(order: Order): String = order match
  case Order(id, Customer(_, name, Gold), items, Delivered) =>
    s"[PREMIUM] Order #$id for $name DELIVERED with ${items.length} items"

  case Order(id, Customer(_, name, _), _, Cancelled(reason)) =>
    s"Order #$id for $name CANCELLED: $reason"

  case Order(id, _, items, Shipped(tracking)) if items.length > 10 =>
    s"Large order #$id SHIPPED - Tracking: $tracking"

  case Order(id, Customer(_, name, _), _, status) =>
    s"Order #$id for $name: $status"

// ทดสอบ
val orders = List(
  Order(1,
    Customer(1, "Alice", Gold),
    List(OrderItem(1, 2, 10.0)),
    Delivered),
  Order(2,
    Customer(2, "Bob", Bronze),
    List(OrderItem(2, 1, 5.0)),
    Cancelled("Out of stock")),
  Order(3,
    Customer(3, "Charlie", Silver),
    List.fill(15)(OrderItem(1, 1, 1.0)),
    Shipped("TRK123456"))
)

orders.foreach(o => println(processOrder(o)))
```

### OR Patterns

```scala
def isHoliday(day: String): Boolean = day.toLowerCase match
  case "saturday" | "sunday" => true
  case "monday" | "tuesday" | "wednesday" | "thursday" | "friday" => false
  case _ => false

println(isHoliday("Saturday"))  // true
println(isHoliday("Monday"))    // false

// OR patterns กับ case classes
case class Response(status: Int, body: String)

def isError(resp: Response): Boolean = resp.status match
  case 400 | 401 | 403 | 404 => true  // Client errors
  case s if s >= 500 => true           // Server errors
  case _ => false
```

---

## Sealed Traits และ ADTs

### Algebraic Data Types (ADTs)

ADT (Algebraic Data Type) เป็น pattern ที่ใช้ sealed trait กับ case classes สร้าง type-safe data structures

```scala
// Sum Type (Either A or B)
sealed trait Result[+A]
case class Success[A](value: A) extends Result[A]
case class Failure(error: String) extends Result[Nothing]

// Product Type (A and B)
case class Pair[A, B](first: A, second: B)

// ตัวอย่าง: HTTP Response
sealed trait HttpResponse
case class Ok(body: String, headers: Map[String, String] = Map.empty) extends HttpResponse
case class NotFound(path: String) extends HttpResponse
case class BadRequest(message: String) extends HttpResponse
case class ServerError(message: String, cause: Option[Throwable] = None) extends HttpResponse

def handleResponse(resp: HttpResponse): String = resp match
  case Ok(body, headers)    => s"200 OK: $body"
  case NotFound(path)       => s"404 Not Found: $path"
  case BadRequest(msg)      => s"400 Bad Request: $msg"
  case ServerError(msg, _)  => s"500 Error: $msg"
```

### Expression Problem กับ ADTs

```scala
// AST (Abstract Syntax Tree) สำหรับ expression language
sealed trait Expr
case class Lit(value: Double) extends Expr
case class Var(name: String) extends Expr
case class Add(left: Expr, right: Expr) extends Expr
case class Mul(left: Expr, right: Expr) extends Expr
case class Neg(expr: Expr) extends Expr
case class Div(left: Expr, right: Expr) extends Expr
case class Let(name: String, value: Expr, body: Expr) extends Expr
case class If(cond: Expr, thenBranch: Expr, elseBranch: Expr) extends Expr

type Env = Map[String, Double]

def eval(expr: Expr, env: Env = Map.empty): Double = expr match
  case Lit(v)      => v
  case Var(name)   => env.getOrElse(name, throw new RuntimeException(s"Undefined: $name"))
  case Add(l, r)   => eval(l, env) + eval(r, env)
  case Mul(l, r)   => eval(l, env) * eval(r, env)
  case Neg(e)      => -eval(e, env)
  case Div(l, r)   =>
    val divisor = eval(r, env)
    if divisor == 0 then throw new ArithmeticException("Division by zero")
    eval(l, env) / divisor
  case Let(name, value, body) =>
    val v = eval(value, env)
    eval(body, env + (name -> v))
  case If(cond, t, e) =>
    if eval(cond, env) != 0 then eval(t, env)
    else eval(e, env)

def prettyPrint(expr: Expr): String = expr match
  case Lit(v)      => v.toString
  case Var(name)   => name
  case Add(l, r)   => s"(${prettyPrint(l)} + ${prettyPrint(r)})"
  case Mul(l, r)   => s"(${prettyPrint(l)} * ${prettyPrint(r)})"
  case Neg(e)      => s"(-${prettyPrint(e)})"
  case Div(l, r)   => s"(${prettyPrint(l)} / ${prettyPrint(r)})"
  case Let(n, v, b) => s"let $n = ${prettyPrint(v)} in ${prettyPrint(b)}"
  case If(c, t, e) => s"if ${prettyPrint(c)} then ${prettyPrint(t)} else ${prettyPrint(e)}"

// ทดสอบ
// let x = 5 in (x + 3) * 2
val program = Let("x", Lit(5), Mul(Add(Var("x"), Lit(3)), Lit(2)))

println(prettyPrint(program))
// let x = 5 in ((x + 3) * 2)

println(eval(program))
// 16.0
```

### Enums (Scala 3 style)

```scala
// Scala 3 enum syntax
enum Color:
  case Red, Green, Blue, Yellow, Purple

enum Direction:
  case North, South, East, West

// Enum กับ parameters
enum Shape:
  case Circle(radius: Double)
  case Rectangle(width: Double, height: Double)
  case Triangle(base: Double, height: Double)

  def area: Double = this match
    case Circle(r)        => math.Pi * r * r
    case Rectangle(w, h)  => w * h
    case Triangle(b, h)   => 0.5 * b * h

// Enum กับ methods
enum Planet(val mass: Double, val radius: Double):
  case Mercury extends Planet(3.303e+23, 2.4397e6)
  case Venus   extends Planet(4.869e+24, 6.0518e6)
  case Earth   extends Planet(5.976e+24, 6.37814e6)
  case Mars    extends Planet(6.421e+23, 3.3972e6)

  val surfaceGravity: Double =
    6.67300E-11 * mass / (radius * radius)

  def surfaceWeight(otherMass: Double): Double =
    otherMass * surfaceGravity

// ทดสอบ
println(Shape.Circle(5).area)      // 78.53...
println(Shape.Rectangle(4, 3).area) // 12.0

val earthWeight = 75.0  // kg
Planet.values.foreach { planet =>
  println(f"${planet.name}: ${planet.surfaceWeight(earthWeight)}%.2f N")
}
```

### Recursive ADTs

```scala
// Tree data structure
sealed trait Tree[+A]
case object Leaf extends Tree[Nothing]
case class Branch[A](value: A, left: Tree[A], right: Tree[A]) extends Tree[A]

def size[A](tree: Tree[A]): Int = tree match
  case Leaf => 0
  case Branch(_, left, right) => 1 + size(left) + size(right)

def depth[A](tree: Tree[A]): Int = tree match
  case Leaf => 0
  case Branch(_, left, right) => 1 + math.max(depth(left), depth(right))

def sum(tree: Tree[Int]): Int = tree match
  case Leaf => 0
  case Branch(v, left, right) => v + sum(left) + sum(right)

def map[A, B](tree: Tree[A])(f: A => B): Tree[B] = tree match
  case Leaf => Leaf
  case Branch(v, left, right) => Branch(f(v), map(left)(f), map(right)(f))

def toList[A](tree: Tree[A]): List[A] = tree match
  case Leaf => Nil
  case Branch(v, left, right) => toList(left) ++ List(v) ++ toList(right)

// สร้าง Binary Search Tree
val bst =
  Branch(5,
    Branch(3,
      Branch(1, Leaf, Leaf),
      Branch(4, Leaf, Leaf)
    ),
    Branch(8,
      Branch(7, Leaf, Leaf),
      Branch(9, Leaf, Leaf)
    )
  )

println(s"Size: ${size(bst)}")    // 7
println(s"Depth: ${depth(bst)}")  // 3
println(s"Sum: ${sum(bst)}")      // 37
println(s"InOrder: ${toList(bst)}") // List(1, 3, 4, 5, 7, 8, 9)
println(map(bst)(_ * 2))
```

---

## Extractors

### Custom Extractors (unapply)

```scala
// Extractor: object กับ unapply method
object EvenNumber:
  def unapply(n: Int): Option[Int] =
    if n % 2 == 0 then Some(n) else None

object OddNumber:
  def unapply(n: Int): Option[Int] =
    if n % 2 != 0 then Some(n) else None

// ใช้งาน
10 match
  case EvenNumber(n) => println(s"$n is even")
  case OddNumber(n)  => println(s"$n is odd")

// Pattern matching
val numbers = List(1, 2, 3, 4, 5, 6)
for number <- numbers do
  number match
    case EvenNumber(n) => println(s"Even: $n")
    case OddNumber(n)  => println(s"Odd: $n")
```

### Extractor กับ String Patterns

```scala
// Email extractor
object Email:
  def unapply(s: String): Option[(String, String)] =
    s.split("@") match
      case Array(local, domain) if domain.contains(".") =>
        Some((local, domain))
      case _ => None

// URL extractor
object Url:
  def unapply(s: String): Option[(String, String)] =
    if s.startsWith("https://") then Some(("https", s.drop(8)))
    else if s.startsWith("http://") then Some(("http", s.drop(7)))
    else None

// ใช้งาน
def processInput(input: String): String = input match
  case Email(local, domain) => s"Email: $local at $domain"
  case Url("https", path)   => s"Secure URL: $path"
  case Url("http", path)    => s"Insecure URL: $path"
  case s                    => s"Unknown: $s"

println(processInput("alice@example.com"))
// Email: alice at example.com
println(processInput("https://www.scala-lang.org"))
// Secure URL: www.scala-lang.org
println(processInput("hello world"))
// Unknown: hello world
```

### Boolean Extractors (unapply กับ Boolean)

```scala
// unapply ที่ return Boolean (ไม่ extract ค่า)
object UpperCase:
  def unapply(s: String): Boolean = s == s.toUpperCase

object Positive:
  def unapply(n: Int): Boolean = n > 0

"HELLO" match
  case UpperCase() => println("All uppercase!")
  case _ => println("Mixed case")

5 match
  case Positive() => println("Positive!")
  case _ => println("Non-positive")
```

### unapplySeq สำหรับ Variable-length Matching

```scala
object WordList:
  def unapplySeq(s: String): Option[Seq[String]] =
    Some(s.split("\\s+").toSeq)

"hello world scala" match
  case WordList(first, rest*) =>
    println(s"First: $first, Rest: ${rest.mkString(", ")}")
  case _ =>

"hello world scala" match
  case WordList(a, b, c) =>
    println(s"Exactly three words: $a, $b, $c")
  case WordList(words*) =>
    println(s"${words.length} words: ${words.mkString(", ")}")
```

---

## Pattern Matching กับ Collections

### List Patterns

```scala
def describeList[A](list: List[A]): String = list match
  case Nil               => "empty"
  case _ :: Nil          => "single"
  case _ :: _ :: Nil     => "pair"
  case head :: tail      => s"head=${head}, ${tail.length} more"

println(describeList(List()))        // empty
println(describeList(List(1)))       // single
println(describeList(List(1, 2)))    // pair
println(describeList(List(1, 2, 3))) // head=1, 2 more

// Pattern matching ใน recursive algorithms
def sum(list: List[Int]): Int = list match
  case Nil          => 0
  case head :: tail => head + sum(tail)

def merge(a: List[Int], b: List[Int]): List[Int] = (a, b) match
  case (Nil, bs)                    => bs
  case (as, Nil)                    => as
  case (ah :: at, bh :: _) if ah <= bh => ah :: merge(at, b)
  case (_, bh :: bt)                => bh :: merge(a, bt)

println(merge(List(1,3,5), List(2,4,6)))  // List(1,2,3,4,5,6)
```

### Option Patterns

```scala
def processOption(opt: Option[Int]): String = opt match
  case Some(n) if n > 0 => s"Positive: $n"
  case Some(0)           => "Zero"
  case Some(n)           => s"Negative: $n"
  case None              => "Nothing"

// Nested Options
def getUsername(userId: Option[Int], users: Map[Int, String]): String =
  userId match
    case Some(id) => users.get(id) match
      case Some(name) => name
      case None       => s"User $id not found"
    case None => "No user specified"

// สวยกว่าด้วย for comprehension
def getUsernameClean(userId: Option[Int], users: Map[Int, String]): Option[String] =
  for
    id   <- userId
    name <- users.get(id)
  yield name
```

### Map Patterns

```scala
def extractConfig(config: Map[String, String]): String =
  config match
    case m if m.contains("host") && m.contains("port") =>
      s"Server: ${m("host")}:${m("port")}"
    case m if m.contains("host") =>
      s"Server: ${m("host")} (default port)"
    case _ =>
      "No server config"

println(extractConfig(Map("host" -> "localhost", "port" -> "8080")))
// Server: localhost:8080
```

---

## ตัวอย่างโปรแกรมครบ: JSON-like Data Model

```scala
// JSON-like Value type
sealed trait JsonValue
case object JsonNull extends JsonValue
case class JsonBool(value: Boolean) extends JsonValue
case class JsonNumber(value: Double) extends JsonValue
case class JsonString(value: String) extends JsonValue
case class JsonArray(elements: List[JsonValue]) extends JsonValue
case class JsonObject(fields: Map[String, JsonValue]) extends JsonValue

// Pretty printer
def prettyPrint(json: JsonValue, indent: Int = 0): String =
  val spaces = "  " * indent
  val innerSpaces = "  " * (indent + 1)

  json match
    case JsonNull     => "null"
    case JsonBool(b)  => b.toString
    case JsonNumber(n) =>
      if n == n.toLong then n.toLong.toString else n.toString
    case JsonString(s) => s""""$s""""
    case JsonArray(elements) if elements.isEmpty => "[]"
    case JsonArray(elements) =>
      val items = elements.map(e => s"$innerSpaces${prettyPrint(e, indent + 1)}")
      s"[\n${items.mkString(",\n")}\n$spaces]"
    case JsonObject(fields) if fields.isEmpty => "{}"
    case JsonObject(fields) =>
      val items = fields.map { case (k, v) =>
        s"""$innerSpaces"$k": ${prettyPrint(v, indent + 1)}"""
      }
      s"{\n${items.mkString(",\n")}\n$spaces}"

// JSON accessor
def get(json: JsonValue, path: String*): Option[JsonValue] =
  path.foldLeft(Option(json)) { (current, key) =>
    current.flatMap {
      case JsonObject(fields) => fields.get(key)
      case JsonArray(elems)   => key.toIntOption.flatMap(i =>
        if i >= 0 && i < elems.length then Some(elems(i)) else None)
      case _ => None
    }
  }

def getString(json: JsonValue, path: String*): Option[String] =
  get(json, path*).collect { case JsonString(s) => s }

def getNumber(json: JsonValue, path: String*): Option[Double] =
  get(json, path*).collect { case JsonNumber(n) => n }

// ทดสอบ
val userData = JsonObject(Map(
  "id"      -> JsonNumber(1),
  "name"    -> JsonString("Alice"),
  "active"  -> JsonBool(true),
  "age"     -> JsonNumber(30),
  "address" -> JsonObject(Map(
    "city"    -> JsonString("Bangkok"),
    "country" -> JsonString("Thailand"),
    "zip"     -> JsonString("10110")
  )),
  "tags"    -> JsonArray(List(
    JsonString("developer"),
    JsonString("scala"),
    JsonString("functional")
  )),
  "metadata" -> JsonNull
))

println(prettyPrint(userData))
println()
println(s"Name: ${getString(userData, "name")}")
println(s"City: ${getString(userData, "address", "city")}")
println(s"First tag: ${getString(userData, "tags", "0")}")
println(s"Age: ${getNumber(userData, "age")}")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ADT สำหรับ Calculator

```scala
// สร้าง ADT สำหรับ expression language พร้อม functions

sealed trait Expr:
  def +(other: Expr): Expr = Add(this, other)
  def -(other: Expr): Expr = Sub(this, other)
  def *(other: Expr): Expr = Mul(this, other)
  def /(other: Expr): Expr = Div(this, other)

case class Num(n: Double) extends Expr
case class Var(name: String) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Sub(l: Expr, r: Expr) extends Expr
case class Mul(l: Expr, r: Expr) extends Expr
case class Div(l: Expr, r: Expr) extends Expr
case class Pow(base: Expr, exp: Expr) extends Expr
case class Sqrt(expr: Expr) extends Expr
case class Abs(expr: Expr) extends Expr

type Env = Map[String, Double]

// TODO: implement
def eval(expr: Expr, env: Env = Map.empty): Either[String, Double] = ???
def simplify(expr: Expr): Expr = ???
def prettyPrint(expr: Expr): String = ???

// ทดสอบ
// val expr = Num(2) * (Var("x") + Num(3)) / Num(4)
// println(prettyPrint(expr))        // "((2 * (x + 3)) / 4)"
// println(eval(expr, Map("x" -> 5))) // Right(4.0)
```

### แบบฝึกหัดที่ 2: Extractor สำหรับ Parsing

```scala
// สร้าง extractors สำหรับ parsing

// DateString: "2024-01-15" -> (2024, 1, 15)
object DateString:
  def unapply(s: String): Option[(Int, Int, Int)] = ???

// TimeString: "10:30:45" -> (10, 30, 45)
object TimeString:
  def unapply(s: String): Option[(Int, Int, Int)] = ???

// IPv4: "192.168.1.1" -> (192, 168, 1, 1)
object IPv4:
  def unapply(s: String): Option[(Int, Int, Int, Int)] = ???

// ทดสอบ
def parseInput(input: String): String = input match
  case DateString(y, m, d)   => s"Date: Year=$y, Month=$m, Day=$d"
  case TimeString(h, m, s)   => s"Time: $h:$m:$s"
  case IPv4(a, b, c, d)      => s"IP: $a.$b.$c.$d"
  case _                     => s"Unknown: $input"

println(parseInput("2024-01-15"))  // Date: Year=2024, Month=1, Day=15
println(parseInput("10:30:45"))    // Time: 10:30:45
println(parseInput("192.168.1.1")) // IP: 192.168.1.1
println(parseInput("hello"))       // Unknown: hello
```

### แบบฝึกหัดที่ 3: State Machine

```scala
// สร้าง state machine สำหรับ traffic light

sealed trait TrafficLight
case object Red extends TrafficLight
case object Yellow extends TrafficLight
case object Green extends TrafficLight

sealed trait Action
case object TimerExpired extends Action
case object EmergencyVehicle extends Action
case object PowerOutage extends Action

// TODO: implement state transitions
def nextState(current: TrafficLight, action: Action): TrafficLight = ???

// Expected:
// Red + TimerExpired -> Green
// Green + TimerExpired -> Yellow
// Yellow + TimerExpired -> Red
// Any + EmergencyVehicle -> Red
// Any + PowerOutage -> Red (flashing, represented as Red here)

// Simulate traffic light
val actions = List(
  TimerExpired, TimerExpired, TimerExpired, TimerExpired,
  EmergencyVehicle, TimerExpired, TimerExpired
)

// TODO: simulate and print each state
```

**เฉลย แบบฝึกหัดที่ 1 (บางส่วน):**

```scala
def eval(expr: Expr, env: Env = Map.empty): Either[String, Double] = expr match
  case Num(n)      => Right(n)
  case Var(name)   => env.get(name).toRight(s"Undefined variable: $name")
  case Add(l, r)   => for { lv <- eval(l, env); rv <- eval(r, env) } yield lv + rv
  case Sub(l, r)   => for { lv <- eval(l, env); rv <- eval(r, env) } yield lv - rv
  case Mul(l, r)   => for { lv <- eval(l, env); rv <- eval(r, env) } yield lv * rv
  case Div(l, r)   =>
    for
      lv <- eval(l, env)
      rv <- eval(r, env)
      result <- if rv == 0 then Left("Division by zero") else Right(lv / rv)
    yield result
  case Pow(b, e)   => for { bv <- eval(b, env); ev <- eval(e, env) } yield math.pow(bv, ev)
  case Sqrt(e)     => eval(e, env).flatMap(v =>
    if v < 0 then Left("Sqrt of negative number") else Right(math.sqrt(v)))
  case Abs(e)      => eval(e, env).map(math.abs)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Case classes และ features ทั้งหมด (equals, hashCode, copy, toString)
- ✅ Pattern matching ขั้นสูง (guards, type patterns, nested patterns)
- ✅ Sealed traits และ Algebraic Data Types (ADTs)
- ✅ Enums ใน Scala 3
- ✅ Custom extractors (unapply, unapplySeq)
- ✅ Pattern matching กับ collections, Options
- ✅ Expression problem กับ ADTs

## ขั้นตอนถัดไป

ใน [Part 11: Traits](part-11-traits.md) เราจะเรียนรู้:
- Trait definitions
- Mixin composition
- Linearization
- Self types
- Abstract vs concrete members

---

*[← Part 09: Classes และ Objects](part-09-classes-objects.md) | [Part 11: Traits →](part-11-traits.md)*
