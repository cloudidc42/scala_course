# Part 09: Classes และ Objects

## สารบัญ
1. [Class Definitions](#class-definitions)
2. [Constructors](#constructors)
3. [Fields และ Methods](#fields-และ-methods)
4. [Access Modifiers](#access-modifiers)
5. [Companion Objects](#companion-objects)
6. [Inheritance](#inheritance)
7. [Abstract Classes](#abstract-classes)
8. [Override](#override)
9. [Object Equality](#object-equality)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Class Definitions

### Class พื้นฐาน

```scala
// Class definition พื้นฐาน
class Person(val name: String, val age: Int):
  def greeting(): String = s"Hello, I'm $name and I'm $age years old."

// สร้าง instance
val alice = new Person("Alice", 30)
val bob = Person("Bob", 25)  // Scala 3: ไม่ต้องใช้ new

println(alice.greeting())
println(alice.name)  // Alice
println(alice.age)   // 30
```

### Class Parameter Types

```scala
class Example(
  val readOnly: Int,      // val: สร้าง getter เท่านั้น
  var mutable: String,    // var: สร้าง getter และ setter
  private: Double,        // private: ไม่สร้าง getter/setter
  private val hidden: Boolean  // private val: getter เฉพาะภายใน
):
  // private ไม่มี val/var เป็น constructor parameter ธรรมดา
  // ไม่มี getter/setter, เข้าถึงได้เฉพาะใน constructor body

  def show(): Unit =
    println(s"readOnly=$readOnly, mutable=$mutable, hidden=$hidden")
    // println(private)  // ❌ ไม่เข้าถึงได้ หลัง constructor body

val e = Example(1, "hello", 3.14, true)
println(e.readOnly)   // 1
println(e.mutable)    // hello
e.mutable = "world"   // ✅ สามารถเปลี่ยนได้
// println(e.hidden)  // ❌ private
// println(e.private) // ❌ private
```

### Class Body

```scala
class BankAccount(val owner: String, initialBalance: Double):
  // Fields (initialized in constructor body)
  private var balance: Double = initialBalance
  val accountNumber: String = java.util.UUID.randomUUID().toString

  // Initialization code
  println(s"Account created for $owner")

  // Methods
  def deposit(amount: Double): Unit =
    require(amount > 0, "Deposit amount must be positive")
    balance += amount
    println(s"Deposited $$${amount}. Balance: $$${balance}")

  def withdraw(amount: Double): Boolean =
    if amount > balance then
      println("Insufficient funds")
      false
    else
      balance -= amount
      println(s"Withdrew $$${amount}. Balance: $$${balance}")
      true

  def getBalance: Double = balance

  override def toString: String =
    f"BankAccount($owner, balance=$$${balance}%.2f)"

// ใช้งาน
val account = new BankAccount("Alice", 1000.0)
account.deposit(500.0)
account.withdraw(200.0)
account.withdraw(2000.0)
println(account)
println(s"Balance: ${account.getBalance}")
```

---

## Constructors

### Primary Constructor

```scala
// Primary constructor คือ parameter list หลัง class name
class Point(val x: Double, val y: Double):
  // Constructor body (statements here run during construction)
  println(s"Created Point($x, $y)")

  val magnitude: Double = math.sqrt(x * x + y * y)

  def distanceTo(other: Point): Double =
    math.sqrt(math.pow(x - other.x, 2) + math.pow(y - other.y, 2))
```

### Auxiliary Constructors

```scala
class Rectangle private (val width: Double, val height: Double):
  val area: Double = width * height
  val perimeter: Double = 2 * (width + height)
  val isSquare: Boolean = width == height

  override def toString: String =
    s"Rectangle(${width}x${height}, area=${area})"

// Auxiliary constructors ด้วย this(...)
object Rectangle:
  def apply(width: Double, height: Double): Rectangle =
    new Rectangle(width, height)

  def apply(side: Double): Rectangle =
    new Rectangle(side, side)  // square

  def apply(width: Int, height: Int): Rectangle =
    new Rectangle(width.toDouble, height.toDouble)

// ใช้งาน
val r1 = Rectangle(4.0, 3.0)
val r2 = Rectangle(5.0)  // square
val r3 = Rectangle(3, 4)  // from Int

println(r1)  // Rectangle(4.0x3.0, area=12.0)
println(r2)  // Rectangle(5.0x5.0, area=25.0)
println(r2.isSquare)  // true
```

### Constructor Validation

```scala
class Person private (val name: String, val age: Int):
  override def toString = s"Person($name, $age)"

object Person:
  def apply(name: String, age: Int): Either[String, Person] =
    if name.isBlank then Left("Name cannot be blank")
    else if age < 0 || age > 150 then Left(s"Invalid age: $age")
    else Right(new Person(name.trim, age))

  // Factory method ที่ throw exception
  def create(name: String, age: Int): Person =
    apply(name, age) match
      case Right(p) => p
      case Left(err) => throw IllegalArgumentException(err)

// ใช้งาน
Person("Alice", 30) match
  case Right(p) => println(p)
  case Left(err) => println(s"Error: $err")

Person("", 30) match
  case Right(p) => println(p)
  case Left(err) => println(s"Error: $err")  // Error: Name cannot be blank

Person("Bob", -5) match
  case Right(p) => println(p)
  case Left(err) => println(s"Error: $err")  // Error: Invalid age: -5
```

---

## Fields และ Methods

### Instance Fields

```scala
class Circle(val radius: Double):
  // Lazy fields (computed on first access)
  lazy val area: Double =
    println("Computing area...")  // จะเห็น message แค่ครั้งแรก
    math.Pi * radius * radius

  lazy val circumference: Double = 2 * math.Pi * radius

  // Derived properties
  def scale(factor: Double): Circle = Circle(radius * factor)
  def contains(x: Double, y: Double): Boolean =
    x * x + y * y <= radius * radius

val c = Circle(5.0)
println(c.area)          // Computing area... 78.53981633974483
println(c.area)          // 78.53981633974483 (no computation)
println(c.circumference) // Computing circumference... 31.41592653589793
```

### Getters และ Setters

```scala
class Temperature:
  private var _celsius: Double = 0.0

  // Getter
  def celsius: Double = _celsius

  // Setter
  def celsius_=(value: Double): Unit =
    require(value >= -273.15, s"Temperature below absolute zero: $value")
    _celsius = value

  // Derived
  def fahrenheit: Double = _celsius * 9.0 / 5.0 + 32
  def kelvin: Double = _celsius + 273.15

// ใช้งาน
val temp = Temperature()
temp.celsius = 100.0
println(f"${temp.celsius}°C = ${temp.fahrenheit}°F = ${temp.kelvin}K")
// 100.0°C = 212.0°F = 373.15K

// temp.celsius = -300  // ❌ IllegalArgumentException
```

### Static-like Members ด้วย Companion Object

```scala
class Counter private (val count: Int):
  def increment: Counter = Counter(count + 1)
  def decrement: Counter = if count > 0 then Counter(count - 1) else this
  def reset: Counter = Counter.zero

  override def toString = s"Counter($count)"

object Counter:
  val zero: Counter = Counter(0)
  val one: Counter = Counter(1)

  def apply(n: Int): Counter = new Counter(n)

  // Shared state (ระวัง thread safety!)
  private var instanceCount = 0
  def getInstanceCount: Int = instanceCount

val c1 = Counter.zero
val c2 = c1.increment.increment.increment
println(c2)  // Counter(3)
println(c2.decrement)  // Counter(2)
```

---

## Access Modifiers

```scala
class MyClass:
  val publicField = "accessible everywhere"
  private val privateField = "accessible in this class only"
  protected val protectedField = "accessible in subclasses"

  private[this] val instancePrivate = "accessible in this instance only"
  private[mypackage] val packagePrivate = "accessible in mypackage"

  // Methods
  def publicMethod(): Unit = println(publicField)
  private def privateMethod(): Unit = println(privateField)
  protected def protectedMethod(): Unit = println(protectedField)

class SubClass extends MyClass:
  def demo(): Unit =
    println(publicField)    // ✅
    // println(privateField)   // ❌
    println(protectedField) // ✅

  override def protectedMethod(): Unit =
    println(s"Overridden: $protectedField")
```

### Sealed Classes

```scala
// sealed: subclasses ต้องอยู่ใน same file
sealed abstract class Shape:
  def area: Double
  def perimeter: Double

class Circle(val r: Double) extends Shape:
  def area: Double = math.Pi * r * r
  def perimeter: Double = 2 * math.Pi * r

class Rectangle(val w: Double, val h: Double) extends Shape:
  def area: Double = w * h
  def perimeter: Double = 2 * (w + h)

class Triangle(val a: Double, val b: Double, val c: Double) extends Shape:
  def area: Double =
    val s = (a + b + c) / 2
    math.sqrt(s * (s-a) * (s-b) * (s-c))
  def perimeter: Double = a + b + c

// Pattern matching กับ sealed class (exhaustive!)
def describe(shape: Shape): String = shape match
  case Circle(r)        => f"Circle with radius $r (area=${shape.area}%.2f)"
  case Rectangle(w, h)  => f"Rectangle ${w}x${h} (area=${shape.area}%.2f)"
  case Triangle(a,b,c)  => f"Triangle $a-$b-$c (area=${shape.area}%.2f)"
// ถ้าลืม case ใด compiler จะ warn!
```

---

## Companion Objects

### Companion Object คืออะไร

```scala
// Companion object อยู่ใน same file มีชื่อเดียวกับ class
// เข้าถึง private members ของ class ได้

class User private (val id: Int, val name: String, val email: String):
  private var _active: Boolean = true

  def deactivate(): Unit = _active = false
  def isActive: Boolean = _active

  override def toString = s"User($id, $name, $email, active=$_active)"

object User:
  private var nextId = 1

  def apply(name: String, email: String): User =
    val id = nextId
    nextId += 1
    new User(id, name, email)

  def createAdmin(name: String): User =
    val user = apply(name, s"${name.toLowerCase}@admin.com")
    user  // Could do special setup here

  // Factory from Map
  def fromMap(data: Map[String, String]): Option[User] =
    for
      name  <- data.get("name")
      email <- data.get("email")
    yield apply(name, email)

// ใช้งาน
val u1 = User("Alice", "alice@example.com")
val u2 = User("Bob", "bob@example.com")
val admin = User.createAdmin("Admin")
val fromMap = User.fromMap(Map("name" -> "Charlie", "email" -> "charlie@example.com"))

println(u1)       // User(1, Alice, alice@example.com, active=true)
println(u2)       // User(2, Bob, bob@example.com, active=true)
println(admin)    // User(3, Admin, admin@admin.com, active=true)
println(fromMap)  // Some(User(4, Charlie, charlie@example.com, active=true))
```

### apply และ unapply

```scala
class Email private (val value: String):
  override def toString = value

object Email:
  // apply: สร้าง Email จาก String
  def apply(s: String): Option[Email] =
    if s.contains("@") && s.contains(".") then
      Some(new Email(s.toLowerCase.trim))
    else
      None

  // unapply: แกะ Email กลับเป็น String (สำหรับ pattern matching)
  def unapply(email: Email): Option[String] = Some(email.value)

// ใช้งาน
val maybeEmail = Email("alice@example.com")
println(maybeEmail)  // Some(alice@example.com)

Email("invalid") match
  case None => println("Invalid email")
  case Some(e) => println(s"Valid: $e")

// Pattern matching กับ unapply
val email = Email("bob@test.com").get
email match
  case Email(addr) => println(s"Email address: $addr")

// unapply กับ case class (อัตโนมัติ)
case class Point(x: Int, y: Int)
val p = Point(3, 4)
val Point(x, y) = p
println(s"x=$x, y=$y")
```

---

## Inheritance

### Class Inheritance

```scala
// Base class
class Animal(val name: String, val sound: String):
  def speak(): String = s"$name says: $sound"
  def describe(): String = s"I am $name"

// Subclass
class Dog(name: String, val breed: String) extends Animal(name, "Woof"):
  override def describe(): String =
    s"${super.describe()}, a $breed dog"

  def fetch(): String = s"$name fetches the ball!"

class Cat(name: String) extends Animal(name, "Meow"):
  def purr(): String = s"$name purrs..."

// ใช้งาน
val dog = Dog("Rex", "German Shepherd")
val cat = Cat("Whiskers")

println(dog.speak())     // Rex says: Woof
println(dog.describe())  // I am Rex, a German Shepherd dog
println(dog.fetch())     // Rex fetches the ball!
println(cat.speak())     // Whiskers says: Meow
println(cat.purr())      // Whiskers purrs...

// Polymorphism
val animals: List[Animal] = List(dog, cat, Dog("Buddy", "Poodle"))
animals.foreach(a => println(a.speak()))
```

### final Classes และ Methods

```scala
// final class ไม่สามารถ extend ได้
final class ImmutablePoint(val x: Double, val y: Double):
  def distanceTo(other: ImmutablePoint): Double =
    math.sqrt(math.pow(x - other.x, 2) + math.pow(y - other.y, 2))

// class Point3D extends ImmutablePoint(0, 0)  // ❌ Error!

// final method ไม่สามารถ override ได้
class Base:
  final def importantMethod(): String = "cannot override"
  def regularMethod(): String = "can override"

class Derived extends Base:
  // override def importantMethod() = "overridden"  // ❌ Error!
  override def regularMethod(): String = "overridden"
```

### Mixin Inheritance

```scala
// Scala รองรับ multiple inheritance ผ่าน traits (จะเรียนใน Part 11)
// แต่ class สืบทอดได้จาก class เดียวเท่านั้น

class Vehicle(val make: String, val model: String, val year: Int):
  def info(): String = s"$year $make $model"

class ElectricCar(
  make: String,
  model: String,
  year: Int,
  val batteryCapacity: Double  // kWh
) extends Vehicle(make, model, year):
  def chargeTime(hours: Double): Double = batteryCapacity * hours
  override def info(): String = s"${super.info()} (Electric, ${batteryCapacity}kWh)"

class HybridCar(
  make: String,
  model: String,
  year: Int,
  val engineCC: Int,
  val batteryKWh: Double
) extends Vehicle(make, model, year):
  override def info(): String =
    s"${super.info()} (Hybrid, ${engineCC}cc + ${batteryKWh}kWh)"

val tesla = ElectricCar("Tesla", "Model 3", 2023, 75.0)
val prius = HybridCar("Toyota", "Prius", 2023, 1800, 8.8)

println(tesla.info())   // 2023 Tesla Model 3 (Electric, 75.0kWh)
println(prius.info())   // 2023 Toyota Prius (Hybrid, 1800cc + 8.8kWh)

// Polymorphism
val fleet: List[Vehicle] = List(tesla, prius)
fleet.foreach(v => println(v.info()))
```

---

## Abstract Classes

```scala
// abstract class: มี abstract members ที่ต้อง implement
abstract class Shape:
  // Abstract methods (ไม่มี implementation)
  def area: Double
  def perimeter: Double

  // Concrete methods (มี implementation)
  def describe(): String =
    f"Shape with area=${area}%.2f and perimeter=${perimeter}%.2f"

  def scale(factor: Double): Shape

class Circle(val radius: Double) extends Shape:
  def area: Double = math.Pi * radius * radius
  def perimeter: Double = 2 * math.Pi * radius
  def scale(factor: Double): Shape = Circle(radius * factor)

class Rectangle(val width: Double, val height: Double) extends Shape:
  def area: Double = width * height
  def perimeter: Double = 2 * (width + height)
  def scale(factor: Double): Shape = Rectangle(width * factor, height * factor)

// ไม่สามารถสร้าง instance ของ abstract class
// val s = Shape()  // ❌ Error!

val shapes = List(Circle(5), Rectangle(4, 3))
shapes.foreach(s => println(s.describe()))
```

### Abstract Class กับ Template Method Pattern

```scala
abstract class DataProcessor[T, R]:
  // Template method
  final def process(data: List[T]): List[R] =
    val validated = validate(data)
    val transformed = transform(validated)
    postProcess(transformed)

  // Abstract methods ที่ subclass ต้อง implement
  protected def validate(data: List[T]): List[T]
  protected def transform(data: List[T]): List[R]

  // Optional hook method
  protected def postProcess(data: List[R]): List[R] = data

// Implementation
class NumberProcessor extends DataProcessor[String, Int]:
  protected def validate(data: List[String]): List[String] =
    data.filter(s => s.trim.nonEmpty && s.trim.forall(_.isDigit))

  protected def transform(data: List[String]): List[Int] =
    data.map(_.trim.toInt)

  override protected def postProcess(data: List[Int]): List[Int] =
    data.sorted.distinct

// ใช้งาน
val processor = NumberProcessor()
val input = List("5", "3", "", "abc", "3", "1", "  8  ", "5")
val result = processor.process(input)
println(result)  // List(1, 3, 5, 8)
```

---

## Override

### Override Rules

```scala
class Base:
  val x: Int = 10           // val
  var y: Int = 20           // var
  def method(): String = "base"

class Derived extends Base:
  override val x: Int = 100         // ✅ override val
  // override var y: Int = 200     // ❌ ไม่สามารถ override var ด้วย val

  override def method(): String = s"derived (base was ${super.method()})"

  // เพิ่ม method ใหม่
  def extra(): String = "extra in derived"

val d = Derived()
println(d.x)        // 100
println(d.method()) // derived (base was base)
```

### Override กับ Generics

```scala
abstract class Container[A]:
  def get: A
  def map[B](f: A => B): Container[B]

class Box[A](private val value: A) extends Container[A]:
  def get: A = value

  def map[B](f: A => B): Container[B] = Box(f(value))

  override def toString = s"Box($value)"

val intBox = Box(42)
val strBox = intBox.map(_.toString)
val doubleBox = intBox.map(_ * 2.0)

println(intBox)     // Box(42)
println(strBox)     // Box(42)
println(doubleBox)  // Box(84.0)
```

---

## Object Equality

### equals และ hashCode

```scala
class Person(val name: String, val age: Int):
  // ต้อง override ทั้ง equals และ hashCode พร้อมกัน!

  override def equals(other: Any): Boolean =
    other match
      case p: Person => name == p.name && age == p.age
      case _ => false

  override def hashCode(): Int =
    val prime = 31
    var result = 1
    result = prime * result + name.hashCode
    result = prime * result + age
    result

  override def toString = s"Person($name, $age)"

val p1 = Person("Alice", 30)
val p2 = Person("Alice", 30)
val p3 = Person("Bob", 25)

println(p1 == p2)   // true (value equality)
println(p1 == p3)   // false
println(p1 eq p2)   // false (different objects)
println(p1.hashCode == p2.hashCode)  // true (เพราะ equals จริง)

// ใช้ใน Map/Set ได้ถูกต้อง
val set = Set(p1, p2, p3)  // Set(Person(Alice,30), Person(Bob,25))
println(set.size)  // 2 (ไม่ใช่ 3)
```

### Case Classes กับ Equality

```scala
// Case classes implement equals และ hashCode อัตโนมัติ!
case class Point(x: Int, y: Int)

val p1 = Point(1, 2)
val p2 = Point(1, 2)
val p3 = Point(3, 4)

println(p1 == p2)   // true (structural equality)
println(p1 == p3)   // false
println(p1.eq(p2))  // false (different objects)

// ใช้ใน collections ได้ถูกต้อง
val points = Set(Point(1,2), Point(1,2), Point(3,4))
println(points.size)  // 2

// copy method (ด้วย case class)
val moved = p1.copy(x = 5)
println(moved)  // Point(5, 2)
```

---

## ตัวอย่างโปรแกรมครบ: Library Management System

```scala
import java.time.LocalDate

// Domain classes
sealed abstract class LibraryItem(
  val id: String,
  val title: String,
  val author: String
):
  def description: String
  def isAvailable: Boolean

class Book(
  id: String,
  title: String,
  author: String,
  val isbn: String,
  val pages: Int,
  private var _available: Boolean = true
) extends LibraryItem(id, title, author):
  def description: String = s"Book: '$title' by $author ($pages pages)"
  def isAvailable: Boolean = _available
  def checkout(): Boolean =
    if _available then { _available = false; true }
    else false
  def returnBook(): Unit = _available = true

class DVD(
  id: String,
  title: String,
  author: String,
  val duration: Int,  // minutes
  private var _available: Boolean = true
) extends LibraryItem(id, title, author):
  def description: String = s"DVD: '$title' by $author (${duration}min)"
  def isAvailable: Boolean = _available
  def checkout(): Boolean =
    if _available then { _available = false; true }
    else false
  def returnDVD(): Unit = _available = true

// Library service
class Library(val name: String):
  private val items = scala.collection.mutable.HashMap[String, LibraryItem]()
  private val checkouts = scala.collection.mutable.HashMap[String, (String, LocalDate)]()

  def addItem(item: LibraryItem): Unit = items(item.id) = item

  def checkout(itemId: String, memberId: String): Either[String, String] =
    items.get(itemId) match
      case None => Left(s"Item $itemId not found")
      case Some(item) if !item.isAvailable => Left(s"'${item.title}' is not available")
      case Some(book: Book) =>
        book.checkout()
        checkouts(itemId) = (memberId, LocalDate.now())
        Right(s"Checked out '${book.title}'")
      case Some(dvd: DVD) =>
        dvd.checkout()
        checkouts(itemId) = (memberId, LocalDate.now())
        Right(s"Checked out '${dvd.title}'")
      case _ => Left("Unknown item type")

  def returnItem(itemId: String): Either[String, String] =
    items.get(itemId) match
      case None => Left(s"Item $itemId not found")
      case Some(book: Book) =>
        book.returnBook()
        checkouts.remove(itemId)
        Right(s"Returned '${book.title}'")
      case Some(dvd: DVD) =>
        dvd.returnDVD()
        checkouts.remove(itemId)
        Right(s"Returned '${dvd.title}'")
      case _ => Left("Unknown item type")

  def availableItems: List[LibraryItem] =
    items.values.filter(_.isAvailable).toList.sortBy(_.title)

  def checkedOutItems: List[(LibraryItem, String)] =
    checkouts.toList.flatMap { case (itemId, (memberId, _)) =>
      items.get(itemId).map(item => (item, memberId))
    }

  def searchByTitle(query: String): List[LibraryItem] =
    items.values
      .filter(_.title.toLowerCase.contains(query.toLowerCase))
      .toList
      .sortBy(_.title)

  def report(): Unit =
    println(s"\n=== $name Library Report ===")
    println(s"Total items: ${items.size}")
    println(s"Available: ${availableItems.length}")
    println(s"Checked out: ${checkouts.size}")

    println("\nAvailable items:")
    availableItems.foreach(item => println(s"  [${item.id}] ${item.description}"))

    if checkouts.nonEmpty then
      println("\nChecked out items:")
      checkedOutItems.foreach { case (item, member) =>
        println(s"  [${item.id}] ${item.title} → Member: $member")
      }

@main def libraryDemo(): Unit =
  val library = Library("City Central Library")

  // เพิ่ม items
  library.addItem(Book("B001", "Scala Programming", "Odersky", "978-0981531601", 852))
  library.addItem(Book("B002", "Functional Programming in Scala", "Chiusano", "978-1617290657", 320))
  library.addItem(Book("B003", "Clean Code", "Martin", "978-0132350884", 464))
  library.addItem(DVD("D001", "Introduction to Scala", "Coursera", 120))
  library.addItem(DVD("D002", "Functional Programming Principles", "EPFL", 180))

  // Checkout
  library.checkout("B001", "M001") match
    case Right(msg) => println(s"✅ $msg")
    case Left(err) => println(s"❌ $err")

  library.checkout("B001", "M002") match
    case Right(msg) => println(s"✅ $msg")
    case Left(err) => println(s"❌ $err")  // Already checked out

  library.checkout("D001", "M001") match
    case Right(msg) => println(s"✅ $msg")
    case Left(err) => println(s"❌ $err")

  // Search
  println("\nSearch 'Scala':")
  library.searchByTitle("scala").foreach(item => println(s"  ${item.description}"))

  // Report
  library.report()

  // Return
  library.returnItem("B001") match
    case Right(msg) => println(s"\n✅ $msg")
    case Left(err) => println(s"❌ $err")

  library.report()
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Class Design

```scala
// สร้าง Matrix class
class Matrix private (private val data: Array[Array[Double]]):
  val rows: Int = data.length
  val cols: Int = if data.nonEmpty then data(0).length else 0

  def apply(row: Int, col: Int): Double = ???

  def +(other: Matrix): Matrix = ???
  def *(scalar: Double): Matrix = ???
  def *(other: Matrix): Matrix = ???  // matrix multiplication

  def transpose: Matrix = ???

  override def toString: String = ???

object Matrix:
  def apply(data: Array[Array[Double]]): Matrix = ???
  def identity(size: Int): Matrix = ???
  def zeros(rows: Int, cols: Int): Matrix = ???

// ทดสอบ
// val m = Matrix(Array(Array(1,2,3), Array(4,5,6)))
// println(m(0, 1))  // 2
// println(m.transpose)
```

### แบบฝึกหัดที่ 2: Inheritance Hierarchy

```scala
// สร้าง hierarchy สำหรับ geometric shapes
abstract class Shape2D:
  def area: Double
  def perimeter: Double
  def contains(x: Double, y: Double): Boolean
  def boundingBox: (Double, Double, Double, Double)  // (minX, minY, maxX, maxY)

// TODO: Implement
// - Circle(centerX, centerY, radius)
// - Rectangle(x, y, width, height)
// - Triangle(x1,y1, x2,y2, x3,y3)

// ทดสอบ
// val shapes: List[Shape2D] = List(
//   Circle(0, 0, 5),
//   Rectangle(0, 0, 10, 8),
//   Triangle(0, 0, 3, 4, 6, 0)
// )
// shapes.foreach(s => println(f"Area: ${s.area}%.2f, Perimeter: ${s.perimeter}%.2f"))
```

### แบบฝึกหัดที่ 3: Companion Object

```scala
// สร้าง Color class พร้อม companion object
// Color ประกอบด้วย r, g, b (0-255)

class Color private (val r: Int, val g: Int, val b: Int):
  // TODO: implement
  // - hex: String property (e.g., "#FF5733")
  // - toGrayscale: Color
  // - blend(other: Color, ratio: Double): Color

object Color:
  // TODO: implement
  // - apply(r, g, b): Color (with validation)
  // - fromHex(hex: String): Option[Color]
  // - Constants: Red, Green, Blue, White, Black, etc.
```

**เฉลย แบบฝึกหัดที่ 1 (บางส่วน):**

```scala
class Matrix private (private val data: Array[Array[Double]]):
  val rows: Int = data.length
  val cols: Int = if data.nonEmpty then data(0).length else 0

  def apply(row: Int, col: Int): Double = data(row)(col)

  def +(other: Matrix): Matrix =
    require(rows == other.rows && cols == other.cols, "Matrix dimensions must match")
    Matrix(
      Array.tabulate(rows, cols)((i, j) => data(i)(j) + other.data(i)(j))
    )

  def *(scalar: Double): Matrix =
    Matrix(Array.tabulate(rows, cols)((i, j) => data(i)(j) * scalar))

  def transpose: Matrix =
    Matrix(Array.tabulate(cols, rows)((i, j) => data(j)(i)))

  override def toString: String =
    data.map(row => row.map(v => f"$v%6.2f").mkString(" ")).mkString("\n")

object Matrix:
  def apply(data: Array[Array[Double]]): Matrix = new Matrix(data)

  def identity(size: Int): Matrix =
    Matrix(Array.tabulate(size, size)((i, j) => if i == j then 1.0 else 0.0))

  def zeros(rows: Int, cols: Int): Matrix =
    Matrix(Array.fill(rows, cols)(0.0))
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Class definitions และ constructor parameters
- ✅ Primary constructor และ auxiliary constructors
- ✅ Fields, methods, getters, setters
- ✅ Access modifiers (public, private, protected)
- ✅ Companion objects และ factory methods
- ✅ apply/unapply methods
- ✅ Inheritance และ method overriding
- ✅ Abstract classes และ Template Method pattern
- ✅ Object equality (equals/hashCode)
- ✅ Sealed classes

## ขั้นตอนถัดไป

ใน [Part 10: Case Classes และ Pattern Matching](part-10-case-classes.md) เราจะเรียนรู้:
- Case classes อย่างละเอียด
- Pattern matching ขั้นสูง
- Sealed traits และ ADTs
- Extractors

---

*[← Part 08: การจัดการ String](part-08-strings.md) | [Part 10: Case Classes →](part-10-case-classes.md)*
