# Part 11: Traits

## สารบัญ
1. [Trait พื้นฐาน](#trait-พื้นฐาน)
2. [Mixin Composition](#mixin-composition)
3. [Linearization](#linearization)
4. [Self Types](#self-types)
5. [Type Class Pattern](#type-class-pattern)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Trait พื้นฐาน

### Trait Definition

```scala
// Trait คือ interface ที่มี concrete implementation ได้ด้วย
trait Greeting:
  def hello(name: String): String = s"Hello, $name!"
  def goodbye(name: String): String = s"Goodbye, $name!"

// Abstract methods ใน trait
trait Shape:
  def area: Double         // abstract - ต้อง override
  def perimeter: Double    // abstract
  def name: String         // abstract

  // Concrete method ที่ใช้ abstract methods
  def describe: String =
    f"$name: area=${area%.2f}, perimeter=${perimeter%.2f}"

// Implementation
class Circle(radius: Double) extends Shape:
  def area: Double      = math.Pi * radius * radius
  def perimeter: Double = 2 * math.Pi * radius
  def name: String      = "Circle"

class Rectangle(width: Double, height: Double) extends Shape:
  def area: Double      = width * height
  def perimeter: Double = 2 * (width + height)
  def name: String      = "Rectangle"

val shapes: List[Shape] = List(Circle(5), Rectangle(3, 4))
shapes.foreach(s => println(s.describe))
// Circle: area=78.54, perimeter=31.42
// Rectangle: area=12.00, perimeter=14.00
```

### Traits กับ Fields

```scala
trait Logger:
  val prefix: String = "[LOG]"
  var count: Int = 0

  def log(msg: String): Unit =
    count += 1
    println(s"$prefix[$count] $msg")

trait Timestamps:
  def timestamp: String =
    java.time.LocalDateTime.now().toString

class Service extends Logger, Timestamps:
  def process(data: String): Unit =
    log(s"Processing at ${timestamp}: $data")

val svc = Service()
svc.process("order-123")
// [LOG][1] Processing at 2024-01-15T10:30:00: order-123
```

---

## Mixin Composition

### Multiple Traits

```scala
trait Flyable:
  def fly(): String = "I can fly!"
  def altitude: Int = 1000

trait Swimmable:
  def swim(): String = "I can swim!"
  def depth: Int = 10

trait Runnable:
  def run(): String = "I can run!"
  def speed: Int = 30

// Duck ทำได้ทุกอย่าง
class Duck extends Flyable, Swimmable, Runnable:
  override def altitude: Int = 100
  override def speed: Int = 15

val duck = Duck()
println(duck.fly())   // I can fly!
println(duck.swim())  // I can swim!
println(duck.run())   // I can run!

// Mix traits กับ with
trait Persistable:
  def save(): Unit = println("Saving...")
  def load(): Unit = println("Loading...")

class UserService extends Logger, Persistable:
  def createUser(name: String): Unit =
    log(s"Creating user: $name")
    save()
```

### Override ใน Traits

```scala
trait Base:
  def greet: String = "Hello from Base"

trait UpperCase extends Base:
  override def greet: String = super.greet.toUpperCase

trait Exclaim extends Base:
  override def greet: String = super.greet + "!!!"

// Mixin order matters
class C1 extends UpperCase, Exclaim
class C2 extends Exclaim, UpperCase

println(C1().greet)  // HELLO FROM BASE!!!
println(C2().greet)  // HELLO FROM BASE!!!
// linearization order กำหนดผล
```

### Stackable Trait Pattern

```scala
abstract class Queue[A]:
  def get(): A
  def put(a: A): Unit

// Concrete Queue implementation
class BasicQueue[A] extends Queue[A]:
  private val buf = collection.mutable.ArrayBuffer[A]()
  def get(): A = buf.remove(0)
  def put(a: A): Unit = buf += a

// Stackable modifications
trait Doubling extends Queue[Int]:
  abstract override def put(x: Int): Unit = super.put(x * 2)

trait Incrementing extends Queue[Int]:
  abstract override def put(x: Int): Unit = super.put(x + 1)

trait Filtering extends Queue[Int]:
  abstract override def put(x: Int): Unit =
    if x >= 0 then super.put(x)

// ใช้งาน
val q1 = new BasicQueue[Int] with Doubling with Incrementing
q1.put(5)  // put(5+1=6), then put(6*2=12)

val q2 = new BasicQueue[Int] with Incrementing with Doubling
q2.put(5)  // put(5*2=10), then put(10+1=11)

// Note: linearization ทำให้ right-most mixin ทำงานก่อน
```

---

## Linearization

### C3 Linearization Algorithm

```scala
// Scala ใช้ C3 Linearization ในการแก้ Multiple Inheritance

trait A:
  def m: String = "A"

trait B extends A:
  override def m: String = s"B(${super.m})"

trait C extends A:
  override def m: String = s"C(${super.m})"

class D extends B, C:
  override def m: String = s"D(${super.m})"

// Linearization of D:
// D -> C -> B -> A -> AnyRef -> Any
println(D().m)  // D(C(B(A)))

// ลำดับ linearization:
// 1. คลาสเอง (D)
// 2. traits จากขวาไปซ้าย (C, B)
// 3. แต่ละ trait เป็น linearization ของตัวเอง
// 4. กำจัด duplicates (เก็บท้ายสุด)
```

---

## Self Types

### Self Type Annotation

```scala
// Self type บอกว่า trait ต้องถูก mix กับ type ที่กำหนด
trait UserRepository:
  def findUser(id: Int): Option[String]

trait EmailService:
  def sendEmail(to: String, msg: String): Unit

// UserNotificationService ต้องการ UserRepository และ EmailService
trait UserNotificationService:
  this: UserRepository with EmailService => // self type

  def notifyUser(userId: Int, message: String): Unit =
    findUser(userId) match  // ใช้ method จาก UserRepository
      case Some(email) => sendEmail(email, message)
      case None => println(s"User $userId not found")

// Concrete implementation
class UserService extends UserRepository, EmailService, UserNotificationService:
  def findUser(id: Int): Option[String] =
    if id == 1 then Some("alice@example.com") else None
  def sendEmail(to: String, msg: String): Unit =
    println(s"Sending '$msg' to $to")

val service = UserService()
service.notifyUser(1, "Welcome!")
// Sending 'Welcome!' to alice@example.com
service.notifyUser(99, "Hello")
// User 99 not found
```

### Dependency Injection กับ Self Types (Cake Pattern)

```scala
// Cake Pattern สำหรับ Dependency Injection

// Component traits
trait DatabaseComponent:
  trait Database:
    def query(sql: String): List[String]
    def execute(sql: String): Int

  val db: Database

trait LoggingComponent:
  trait Logger:
    def info(msg: String): Unit
    def error(msg: String): Unit

  val logger: Logger

trait UserRepositoryComponent:
  this: DatabaseComponent with LoggingComponent =>

  class UserRepository:
    def findById(id: Int): Option[String] =
      logger.info(s"Finding user $id")
      db.query(s"SELECT name FROM users WHERE id=$id").headOption

    def save(name: String): Int =
      logger.info(s"Saving user $name")
      db.execute(s"INSERT INTO users VALUES ('$name')")

  val userRepository: UserRepository

// Wiring everything together
trait AppComponent extends DatabaseComponent
    with LoggingComponent
    with UserRepositoryComponent:

  val db: Database = new Database:
    def query(sql: String): List[String] =
      println(s"DB Query: $sql")
      List("Alice", "Bob")
    def execute(sql: String): Int =
      println(s"DB Execute: $sql")
      1

  val logger: Logger = new Logger:
    def info(msg: String): Unit = println(s"INFO: $msg")
    def error(msg: String): Unit = println(s"ERROR: $msg")

  val userRepository: UserRepository = new UserRepository

object App extends AppComponent:
  def main(): Unit =
    val user = userRepository.findById(1)
    println(s"Found: $user")

App.main()
// INFO: Finding user 1
// DB Query: SELECT name FROM users WHERE id=1
// Found: Some(Alice)
```

---

## Type Class Pattern

### Type Class พื้นฐาน

```scala
// Type class = trait + given instances
trait Printable[A]:
  def print(a: A): String

// Instances
given Printable[Int] with
  def print(n: Int): String = s"Int($n)"

given Printable[String] with
  def print(s: String): String = s"String($s)"

given Printable[Boolean] with
  def print(b: Boolean): String = if b then "yes" else "no"

// Generic function ที่ต้องการ Printable
def printValue[A](a: A)(using p: Printable[A]): String = p.print(a)

// Extension syntax (Scala 3)
extension [A: Printable](a: A)
  def prettyPrint: String = summon[Printable[A]].print(a)

println(printValue(42))        // Int(42)
println(printValue("hello"))   // String(hello)
println(42.prettyPrint)        // Int(42)
```

### Comparable Type Class

```scala
trait Ord[A]:
  def compare(x: A, y: A): Int

  def lt(x: A, y: A): Boolean = compare(x, y) < 0
  def gt(x: A, y: A): Boolean = compare(x, y) > 0
  def eq(x: A, y: A): Boolean = compare(x, y) == 0

given Ord[Int] with
  def compare(x: Int, y: Int): Int = x - y

given Ord[String] with
  def compare(x: String, y: String): Int = x.compareTo(y)

given [A: Ord]: Ord[List[A]] with
  def compare(xs: List[A], ys: List[A]): Int =
    (xs, ys) match
      case (Nil, Nil)      => 0
      case (Nil, _)        => -1
      case (_, Nil)        => 1
      case (x :: xt, y :: yt) =>
        val c = summon[Ord[A]].compare(x, y)
        if c != 0 then c else compare(xt, yt)

def sort[A: Ord](list: List[A]): List[A] =
  val ord = summon[Ord[A]]
  list.sortWith((a, b) => ord.lt(a, b))

println(sort(List(3, 1, 4, 1, 5, 9)))         // List(1, 1, 3, 4, 5, 9)
println(sort(List("banana", "apple", "cherry"))) // List(apple, banana, cherry)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Observer Pattern

```scala
trait Observer[A]:
  def update(event: A): Unit

trait Observable[A]:
  private val observers = collection.mutable.ListBuffer[Observer[A]]()

  def subscribe(observer: Observer[A]): Unit =
    observers += observer

  def unsubscribe(observer: Observer[A]): Unit =
    observers -= observer

  protected def notify(event: A): Unit =
    observers.foreach(_.update(event))

// ใช้งาน
case class StockEvent(symbol: String, price: Double)

class StockMarket extends Observable[StockEvent]:
  private var prices = Map[String, Double]()

  def updatePrice(symbol: String, price: Double): Unit =
    prices = prices.updated(symbol, price)
    notify(StockEvent(symbol, price))

// TODO: สร้าง Observer ที่:
// 1. พิมพ์ราคาทุกครั้งที่เปลี่ยน
// 2. แจ้งเตือนเมื่อราคาลดลงมากกว่า 5%
// 3. บันทึกประวัติราคา
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Trait พื้นฐาน: abstract methods, concrete methods, fields
- ✅ Mixin Composition: หลาย traits
- ✅ Stackable Trait Pattern
- ✅ C3 Linearization
- ✅ Self Types และ Cake Pattern
- ✅ Type Class Pattern

---

*[← Part 10: Case Classes](part-10-case-classes.md) | [Part 12: Generics →](part-12-generics.md)*
