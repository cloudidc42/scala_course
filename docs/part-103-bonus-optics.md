# ส่วนที่ 103 (BONUS): Optics กับ Monocle ใน Scala

> **BONUS CONTENT** - เนื้อหาขั้นสูงเกี่ยวกับ Optics สำหรับการเข้าถึงและแก้ไข nested data structures อย่างสง่างาม

---

## สารบัญ

1. [บทนำ: ปัญหาของ Nested Data](#บทนำ)
2. [Lens: get และ set nested fields](#lens)
3. [Prism: focus ใน sum types](#prism)
4. [Optional: partial lens](#optional)
5. [Traversal: multiple focuses](#traversal)
6. [Iso: bijection ระหว่าง types](#iso)
7. [Fold และ Getter](#fold-และ-getter)
8. [Setter](#setter)
9. [การ compose optics](#การ-compose-optics)
10. [ตัวอย่างสมบูรณ์: Data Transformation](#data-transformation)
11. [Optics กับ State Monad](#optics-กับ-state-monad)
12. [สรุป](#สรุป)

---

## บทนำ

Optics คือ composable abstraction สำหรับการ access และ modify data structures แบบ immutable

### ปัญหาของ Nested Immutable Data

```scala
// Deep nested case classes
case class Street(name: String, number: Int)
case class Address(street: Street, city: String, country: String)
case class Person(name: String, age: Int, address: Address)

val alice = Person(
  "Alice",
  30,
  Address(
    Street("Main St", 42),
    "Bangkok",
    "Thailand"
  )
)

// การ update nested field ใน vanilla Scala
// อ่านยาก, เขียนยาก, error-prone
val aliceNewStreet = alice.copy(
  address = alice.address.copy(
    street = alice.address.street.copy(
      name = "New St"
    )
  )
)

// ด้วย Optics (Monocle) - สะอาดกว่ามาก!
import monocle.Lens
import monocle.macros.GenLens

val streetName = GenLens[Person](_.address.street.name)
val aliceNewStreet2 = streetName.replace("New St")(alice)
```

### Monocle Library

```scala
// build.sbt
// libraryDependencies += "dev.optics" %% "monocle-core" % "3.2.0"
// libraryDependencies += "dev.optics" %% "monocle-macro" % "3.2.0"

import monocle.*
import monocle.syntax.all.*
```

---

## Lens

Lens คือ optic สำหรับ focus ใน field หนึ่งของ product type (case class)

### นิยามของ Lens

```
Lens[S, A] where:
  S = source type (whole structure)
  A = focus type (part we care about)

Methods:
  get: S => A          // extract focus
  replace: A => S => S // set focus to new value
  modify: (A => A) => S => S // update focus with function
```

### สร้าง Lens

```scala
import monocle.Lens

case class Point(x: Double, y: Double)

// Manual Lens
val xLens: Lens[Point, Double] = Lens[Point, Double](_.x)(newX => p => p.copy(x = newX))
val yLens: Lens[Point, Double] = Lens[Point, Double](_.y)(newY => p => p.copy(y = newY))

// ใช้งาน
val p = Point(1.0, 2.0)
println(xLens.get(p))           // 1.0
println(xLens.replace(5.0)(p))  // Point(5.0, 2.0)
println(xLens.modify(_ + 1)(p)) // Point(2.0, 2.0)
```

### สร้าง Lens ด้วย Macro

```scala
import monocle.macros.GenLens

case class User(name: String, age: Int, address: Address)
case class Address(street: String, city: String, zip: String)

// Auto-generate Lens
val nameLens: Lens[User, String]    = GenLens[User](_.name)
val ageLens: Lens[User, Int]        = GenLens[User](_.age)
val addressLens: Lens[User, Address] = GenLens[User](_.address)
val streetLens: Lens[Address, String] = GenLens[Address](_.street)

// สร้าง nested Lens ด้วย macro
val userStreetLens: Lens[User, String] = GenLens[User](_.address.street)

// Test
val user = User("Alice", 30, Address("123 Main St", "Bangkok", "10110"))
println(userStreetLens.get(user))            // 123 Main St
println(userStreetLens.replace("456 New Ave")(user))
// User(Alice,30,Address(456 New Ave,Bangkok,10110))
```

### Lens Laws

```scala
// Laws ที่ Lens ต้องทำตาม:
// 1. get-set: ถ้า set แล้ว get กลับมาต้องได้ค่าเดิม
//    lens.get(lens.replace(a)(s)) == a
// 2. set-get: ถ้า get แล้ว set กลับต้องได้ structure เดิม
//    lens.replace(lens.get(s))(s) == s
// 3. set-set: set สองครั้งต้องเหมือนกับ set ครั้งเดียว
//    lens.replace(a2)(lens.replace(a1)(s)) == lens.replace(a2)(s)

def verifyLensLaws[S, A](lens: Lens[S, A], s: S, a: A): Unit =
  assert(lens.get(lens.replace(a)(s)) == a, "get-set law failed")
  assert(lens.replace(lens.get(s))(s) == s, "set-get law failed")
  println("Lens laws verified!")

verifyLensLaws(xLens, Point(1.0, 2.0), 5.0)
```

### Lens Syntax

```scala
import monocle.syntax.all.*

val user = User("Alice", 30, Address("123 Main St", "Bangkok", "10110"))

// Focus syntax
val updated = user
  .focus(_.address.street).replace("456 Oak Ave")
  .focus(_.age).modify(_ + 1)
  .focus(_.name).replace("Alice Smith")

println(updated)
// User(Alice Smith,31,Address(456 Oak Ave,Bangkok,10110))
```

---

## Prism

Prism คือ optic สำหรับ focus ใน constructor ของ sum type (sealed trait)

### นิยามของ Prism

```
Prism[S, A] where:
  S = source type (sum type)
  A = focus type (one variant)

Methods:
  getOption: S => Option[A]  // extract if matching
  reverseGet: A => S         // construct S from A
  modify: (A => A) => S => S // modify if matching, else identity
```

### สร้าง Prism

```scala
import monocle.Prism

sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

// Manual Prism
val circlePrism: Prism[Shape, Double] = Prism[Shape, Double] {
  case Circle(r) => Some(r)
  case _         => None
}(Circle.apply)

// ใช้งาน
val circle: Shape = Circle(5.0)
val rect: Shape = Rectangle(4.0, 6.0)

println(circlePrism.getOption(circle))  // Some(5.0)
println(circlePrism.getOption(rect))    // None
println(circlePrism.reverseGet(3.0))    // Circle(3.0)
println(circlePrism.modify(_ * 2)(circle))   // Circle(10.0)
println(circlePrism.modify(_ * 2)(rect))     // Rectangle(4.0, 6.0) - unchanged
```

### สร้าง Prism ด้วย Macro

```scala
import monocle.macros.GenPrism

val circlePrism2: Prism[Shape, Circle] = GenPrism[Shape, Circle]
val rectPrism: Prism[Shape, Rectangle] = GenPrism[Shape, Rectangle]

// ใช้งาน
val shapes: List[Shape] = List(Circle(1.0), Rectangle(2.0, 3.0), Circle(4.0))
val circleRadii: List[Double] = shapes.flatMap(circlePrism2.getOption(_).map(_.radius))
println(circleRadii)  // List(1.0, 4.0)
```

### Prism สำหรับ Option

```scala
// Option เป็น sum type: Some(a) | None
val some: Prism[Option[Int], Int] = monocle.std.option.some[Int]
val none: Prism[Option[Int], Unit] = monocle.std.option.none[Int]

val opt: Option[Int] = Some(42)
println(some.getOption(opt))        // Some(42)
println(some.modify(_ + 1)(opt))    // Some(43)
println(some.getOption(None))       // None
```

### Prism สำหรับ Either

```scala
val right: Prism[Either[String, Int], Int] = monocle.std.either.stdRight[String, Int]
val left: Prism[Either[String, Int], String] = monocle.std.either.stdLeft[String, Int]

val e: Either[String, Int] = Right(42)
println(right.getOption(e))     // Some(42)
println(left.getOption(e))      // None
```

---

## Optional

Optional คือ generalization ของ Lens และ Prism - ใช้เมื่อ focus อาจไม่มีอยู่

### นิยาม

```
Optional[S, A] where:
  S = source type
  A = focus type (might not exist)

Methods:
  getOption: S => Option[A]
  replace: A => S => S
  modify: (A => A) => S => S  // identity if focus doesn't exist
```

### สร้าง Optional

```scala
import monocle.Optional

case class Config(
  database: Option[DatabaseConfig],
  cache: Option[CacheConfig]
)
case class DatabaseConfig(host: String, port: Int, name: String)
case class CacheConfig(host: String, ttl: Int)

// Optional สำหรับ database field
val dbConfig: Optional[Config, DatabaseConfig] =
  Optional[Config, DatabaseConfig](_.database)(db => cfg => cfg.copy(database = Some(db)))

// Optional สำหรับ database host
val dbHost: Optional[DatabaseConfig, String] =
  Optional[DatabaseConfig, String](cfg => Some(cfg.host))(h => cfg => cfg.copy(host = h))

// หรือใช้ Lens (เพราะ DatabaseConfig.host เป็น required field)
val dbHostLens = GenLens[DatabaseConfig](_.host)

// Compose Optional กับ Lens
val configDbHost: Optional[Config, String] = dbConfig andThen dbHostLens

// ใช้งาน
val config1 = Config(Some(DatabaseConfig("localhost", 5432, "mydb")), None)
val config2 = Config(None, None)

println(configDbHost.getOption(config1))  // Some("localhost")
println(configDbHost.getOption(config2))  // None
println(configDbHost.replace("db.prod.com")(config1))
// Config(Some(DatabaseConfig("db.prod.com",5432,mydb)),None)
println(configDbHost.replace("db.prod.com")(config2))
// Config(None,None) - no change
```

---

## Traversal

Traversal คือ optic ที่ focus บน elements หลายตัวพร้อมกัน

### นิยาม

```
Traversal[S, A] where:
  S = source type
  A = type of each element

Methods:
  getAll: S => List[A]
  modify: (A => A) => S => S  // modify all elements
  replace: A => S => S        // replace all elements
```

### สร้าง Traversal

```scala
import monocle.Traversal

// Traversal สำหรับ List
val listTraversal: Traversal[List[Int], Int] =
  Traversal.fromTraverse[List, Int]

// ใช้งาน
val nums = List(1, 2, 3, 4, 5)
println(listTraversal.getAll(nums))      // List(1, 2, 3, 4, 5)
println(listTraversal.modify(_ * 2)(nums))  // List(2, 4, 6, 8, 10)
println(listTraversal.replace(0)(nums))    // List(0, 0, 0, 0, 0)
```

### Traversal สำหรับ Product Type

```scala
// Traversal ที่ focus บน fields หลาย fields
case class RGB(r: Int, g: Int, b: Int)

// Manual Traversal
val rgbTraversal: Traversal[RGB, Int] = new Traversal[RGB, Int]:
  def modifyA[F[_]: cats.Applicative](f: Int => F[Int])(s: RGB): F[RGB] =
    import cats.syntax.apply.*
    (f(s.r), f(s.g), f(s.b)).mapN(RGB.apply)

// ใช้งาน
val color = RGB(100, 150, 200)
println(rgbTraversal.getAll(color))        // List(100, 150, 200)
println(rgbTraversal.modify(_ / 2)(color)) // RGB(50, 75, 100)

// Clamp values to 0-255
println(rgbTraversal.modify(_.clamp(0, 255))(RGB(300, -10, 128)))
// RGB(255, 0, 128)
```

### Each: List Traversal ด้วย Monocle

```scala
import monocle.syntax.all.*

case class Order(
  id: String,
  items: List[OrderItem],
  status: String
)
case class OrderItem(name: String, price: Double, qty: Int)

val orders = List(
  Order("O1", List(OrderItem("Apple", 30.0, 2), OrderItem("Banana", 20.0, 3)), "pending"),
  Order("O2", List(OrderItem("Cherry", 150.0, 1)), "shipped")
)

// Update all prices
val updatedOrders = orders.focus().each.focus(_.items).each.focus(_.price).modify(_ * 1.1)
```

---

## Iso

Iso คือ optic ที่แสดง bijection (1-to-1 correspondence) ระหว่าง 2 types

### นิยาม

```
Iso[S, A] where:
  S = source type
  A = target type (isomorphic to S)

Methods:
  get: S => A
  reverseGet: A => S
```

### สร้าง Iso

```scala
import monocle.Iso

// Iso ระหว่าง String และ List[Char]
val stringToChars: Iso[String, List[Char]] =
  Iso[String, List[Char]](_.toList)(_.mkString)

// ใช้งาน
val str = "Hello"
println(stringToChars.get(str))                // List(H, e, l, l, o)
println(stringToChars.reverseGet(List('W', 'o', 'r', 'l', 'd')))  // World

// Modify ผ่าน Iso
println(stringToChars.modify(_.reverse)(str))  // olleH
println(stringToChars.modify(_.map(_.toUpper))(str))  // HELLO
```

### Iso สำหรับ Newtype

```scala
// Newtype pattern ด้วย Iso
opaque type Celsius = Double
opaque type Fahrenheit = Double

object Celsius:
  def apply(d: Double): Celsius = d
  
object Fahrenheit:
  def apply(d: Double): Fahrenheit = d

val celsiusToFahrenheit: Iso[Celsius, Fahrenheit] =
  Iso[Celsius, Fahrenheit](c => Fahrenheit(c * 9.0 / 5.0 + 32))(
    f => Celsius((f - 32) * 5.0 / 9.0)
  )

val boiling: Celsius = Celsius(100.0)
println(celsiusToFahrenheit.get(boiling))  // 212.0 (Fahrenheit)

// Reverse
val body: Fahrenheit = Fahrenheit(98.6)
println(celsiusToFahrenheit.reverseGet(body))  // 37.0 (Celsius)
```

### Iso กับ Case Classes

```scala
// Iso ระหว่าง case class และ tuple
case class Point2D(x: Double, y: Double)

val pointToTuple: Iso[Point2D, (Double, Double)] =
  Iso[Point2D, (Double, Double)](p => (p.x, p.y))(t => Point2D(t._1, t._2))

val p = Point2D(3.0, 4.0)
val tuple = pointToTuple.get(p)  // (3.0, 4.0)
val back = pointToTuple.reverseGet(tuple)  // Point2D(3.0, 4.0)
```

---

## Fold และ Getter

### Getter

```scala
import monocle.Getter

// Getter: read-only Lens
val lengthGetter: Getter[String, Int] = Getter(_.length)
val headGetter: Getter[List[Int], Option[Int]] = Getter(_.headOption)

println(lengthGetter.get("Hello"))  // 5
println(headGetter.get(List(1, 2, 3)))  // Some(1)
```

### Fold

```scala
import monocle.Fold

// Fold: read multiple values
val listFold: Fold[List[Int], Int] = Fold.fromFoldable[List, Int]

val nums = List(1, 2, 3, 4, 5)
println(listFold.getAll(nums))      // List(1, 2, 3, 4, 5)
println(listFold.exist(_ > 3)(nums))  // true
println(listFold.forall(_ > 0)(nums)) // true
println(listFold.length(nums))        // 5
```

---

## Setter

```scala
import monocle.Setter

// Setter: write-only
val listSetter: Setter[List[Int], Int] = Setter[List[Int], Int](f => lst => lst.map(f))

val nums = List(1, 2, 3, 4, 5)
println(listSetter.modify(_ * 10)(nums))  // List(10, 20, 30, 40, 50)
println(listSetter.replace(0)(nums))       // List(0, 0, 0, 0, 0)
```

---

## การ Compose Optics

Optics สามารถ compose กันได้ - นี่คือ superpower ของ Optics!

### Optic Hierarchy

```
Iso > Lens > Optional > Traversal > Fold > Getter
     > Prism > Optional
```

### Composition Rules

```
Lens + Lens = Lens
Lens + Prism = Optional
Lens + Optional = Optional
Lens + Traversal = Traversal
Prism + Prism = Prism
Prism + Lens = Optional
Optional + Optional = Optional
... etc
```

### ตัวอย่าง Composition

```scala
case class Company(
  name: String,
  employees: List[Employee]
)
case class Employee(
  name: String,
  department: String,
  salary: Salary,
  contact: Contact
)
case class Salary(base: Double, bonus: Option[Double])
case class Contact(email: String, phone: Option[String])

// Lens chain
val companyEmployees: Lens[Company, List[Employee]] = GenLens[Company](_.employees)
val employeeSalary: Lens[Employee, Salary] = GenLens[Employee](_.salary)
val salaryBase: Lens[Salary, Double] = GenLens[Salary](_.base)
val salaryBonus: Optional[Salary, Double] = 
  monocle.std.option.some compose GenLens[Salary](_.bonus) // เราต้องใช้วิธีอื่น

// ใช้ macro สำหรับ nested path
val employeeBaseSalary: Lens[Employee, Double] = GenLens[Employee](_.salary.base)

// Traversal: access all employees' salaries
val allSalaries: Traversal[Company, Double] =
  companyEmployees
    .andThen(Traversal.fromTraverse[List, Employee])
    .andThen(employeeSalary)
    .andThen(salaryBase)

// ใช้งาน
val company = Company(
  "TechCorp",
  List(
    Employee("Alice", "Eng", Salary(100000, Some(20000)), Contact("alice@tech.com", None)),
    Employee("Bob", "Eng", Salary(90000, Some(15000)), Contact("bob@tech.com", Some("081-xxx"))),
    Employee("Carol", "HR", Salary(80000, None), Contact("carol@tech.com", None))
  )
)

// Get all base salaries
println(allSalaries.getAll(company))  // List(100000.0, 90000.0, 80000.0)

// Give 10% raise to everyone
val afterRaise = allSalaries.modify(_ * 1.1)(company)
println(afterRaise.employees.map(_.salary.base))  // List(110000.0, 99000.0, 88000.0)
```

### Compose Prism + Lens

```scala
sealed trait Payment
case class CreditCard(number: String, cvv: String, expiry: String) extends Payment
case class BankTransfer(accountNo: String, bankCode: String) extends Payment
case class Cash(amount: Double) extends Payment

val creditCardPrism: Prism[Payment, CreditCard] = GenPrism[Payment, CreditCard]
val ccNumber: Lens[CreditCard, String] = GenLens[CreditCard](_.number)

// Compose: Optional[Payment, String] = Prism + Lens
val paymentCCNumber: Optional[Payment, String] = creditCardPrism.andThen(ccNumber)

val p1: Payment = CreditCard("1234-5678-9012-3456", "123", "12/25")
val p2: Payment = Cash(500.0)

println(paymentCCNumber.getOption(p1))  // Some(1234-5678-9012-3456)
println(paymentCCNumber.getOption(p2))  // None

// Mask credit card number
def maskCC(s: String): String = 
  s.replaceAll("[0-9](?=[0-9]{4})", "*")

println(paymentCCNumber.modify(maskCC)(p1))
// CreditCard(****-****-****-3456,123,12/25)
```

---

## Data Transformation

ตัวอย่างสมบูรณ์: ระบบ E-commerce Data Transformation

### Domain Model

```scala
// Source data (from external API)
case class ApiProduct(
  product_id: String,
  product_name: String,
  price_thb: Double,
  stock_qty: Int,
  category_path: String,  // "Electronics/Phones/Smartphone"
  images: List[String],
  discount_pct: Option[Double]
)

// Target data (our domain model)
case class Product(
  id: ProductId,
  name: ProductName,
  pricing: Pricing,
  inventory: Inventory,
  category: Category,
  media: Media
)

opaque type ProductId = String
opaque type ProductName = String

case class Pricing(
  base: Money,
  discount: Option[Percentage],
  effective: Money
)
case class Money(amount: BigDecimal, currency: String)
case class Percentage(value: Double)

case class Inventory(quantity: Int, available: Boolean)
case class Category(path: List[String], primary: String)
case class Media(images: List[String], primary: Option[String])

object ProductId:
  def apply(s: String): ProductId = s
object ProductName:
  def apply(s: String): ProductName = s
```

### Optics สำหรับ Transformation

```scala
// Optics สำหรับ nested access
val productPricing: Lens[Product, Pricing] = GenLens[Product](_.pricing)
val pricingBase: Lens[Pricing, Money] = GenLens[Pricing](_.base)
val moneyAmount: Lens[Money, BigDecimal] = GenLens[Money](_.amount)
val pricingEffective: Lens[Pricing, Money] = GenLens[Pricing](_.effective)

val productBaseAmount: Lens[Product, BigDecimal] =
  productPricing andThen pricingBase andThen moneyAmount

val productInventory: Lens[Product, Inventory] = GenLens[Product](_.inventory)
val inventoryQty: Lens[Inventory, Int] = GenLens[Inventory](_.quantity)

val productQty: Lens[Product, Int] = productInventory andThen inventoryQty

// Convert ApiProduct to Product
def apiToProduct(api: ApiProduct): Product =
  val baseMoney = Money(BigDecimal(api.price_thb), "THB")
  val effectiveMoney = api.discount_pct match
    case Some(pct) => Money(BigDecimal(api.price_thb * (1 - pct / 100)), "THB")
    case None      => baseMoney
  
  val categoryPath = api.category_path.split("/").toList
  
  Product(
    id = ProductId(api.product_id),
    name = ProductName(api.product_name),
    pricing = Pricing(
      base = baseMoney,
      discount = api.discount_pct.map(d => Percentage(d)),
      effective = effectiveMoney
    ),
    inventory = Inventory(api.stock_qty, api.stock_qty > 0),
    category = Category(categoryPath, categoryPath.headOption.getOrElse("")),
    media = Media(api.images, api.images.headOption)
  )
```

### Bulk Operations ด้วย Traversal

```scala
// สร้าง Traversal สำหรับ List[Product]
val allProducts: Traversal[List[Product], Product] =
  Traversal.fromTraverse[List, Product]

val allPrices: Traversal[List[Product], BigDecimal] =
  allProducts andThen productPricing andThen pricingBase andThen moneyAmount

// Apply global discount
def applyGlobalDiscount(pct: Double)(products: List[Product]): List[Product] =
  allPrices.modify(price => price * BigDecimal(1 - pct / 100))(products)

// Update effective prices after discount
def recalcEffective(products: List[Product]): List[Product] =
  allProducts.modify { product =>
    val base = productBaseAmount.get(product)
    val discount = product.pricing.discount.map(_.value).getOrElse(0.0)
    val effective = base * BigDecimal(1 - discount / 100)
    productPricing.andThen(pricingEffective).andThen(moneyAmount)
      .replace(effective)(product)
  }(products)

// ตัวอย่าง batch operation
val products = List(
  apiToProduct(ApiProduct("P1", "iPhone", 35000, 10, "Electronics/Phones", List("img1.jpg"), Some(10.0))),
  apiToProduct(ApiProduct("P2", "Samsung", 28000, 5, "Electronics/Phones", List("img2.jpg"), None)),
  apiToProduct(ApiProduct("P3", "AirPods", 8500, 20, "Electronics/Audio", List("img3.jpg"), Some(5.0)))
)

// Apply 5% flash sale discount
val onSale = applyGlobalDiscount(5.0)(products)
println(onSale.map(p => productBaseAmount.get(p)))
```

### Validation ด้วย Optics

```scala
import cats.data.Validated
import cats.data.Validated.*
import cats.syntax.validated.*

// Validate fields ด้วย optics
def validateProduct(product: Product): Validated[List[String], Product] =
  val errors = collection.mutable.ListBuffer[String]()
  
  if productBaseAmount.get(product) <= 0 then
    errors += "Price must be positive"
  
  if productQty.get(product) < 0 then
    errors += "Quantity cannot be negative"
  
  val name: String = product.name.asInstanceOf[String]
  if name.isBlank then
    errors += "Name cannot be blank"
  
  if errors.isEmpty then product.valid
  else errors.toList.invalid

// Validate list of products
def validateAll(products: List[Product]): Validated[List[String], List[Product]] =
  import cats.syntax.traverse.*
  products.traverse(validateProduct).leftMap(_.flatten)
```

---

## Optics กับ State Monad

Optics ทำงานร่วมกับ State Monad ได้ดีมาก

```scala
import cats.data.State
import monocle.Lens

// State operations ด้วย Lens
def gets[S, A](lens: Lens[S, A]): State[S, A] =
  State.inspect(lens.get)

def puts[S, A](lens: Lens[S, A])(value: A): State[S, Unit] =
  State.modify(lens.replace(value))

def modifies[S, A](lens: Lens[S, A])(f: A => A): State[S, Unit] =
  State.modify(lens.modify(f))

// Game State ตัวอย่าง
case class GameState(
  player: Player,
  enemies: List[Enemy],
  score: Int,
  level: Int
)
case class Player(hp: Int, mp: Int, position: Point)
case class Enemy(id: String, hp: Int, position: Point)
case class Point(x: Int, y: Int)

val playerLens: Lens[GameState, Player] = GenLens[GameState](_.player)
val playerHp: Lens[GameState, Int] = playerLens andThen GenLens[Player](_.hp)
val scoreLens: Lens[GameState, Int] = GenLens[GameState](_.score)
val levelLens: Lens[GameState, Int] = GenLens[GameState](_.level)

// Game actions as State
type Game[A] = State[GameState, A]

def healPlayer(amount: Int): Game[Unit] =
  modifies(playerHp)(_ + amount)

def addScore(points: Int): Game[Unit] =
  modifies(scoreLens)(_ + points)

def nextLevel: Game[Unit] =
  modifies(levelLens)(_ + 1)

def getScore: Game[Int] =
  gets(scoreLens)

def attack(damage: Int): Game[String] =
  for
    hp   <- gets(playerHp)
    _    <- modifies(playerHp)(_ - damage)
    newHp <- gets(playerHp)
  yield if newHp <= 0 then "Player died!" else s"HP: $hp -> $newHp"

// Run game
val gameProgram: Game[String] = for
  _     <- healPlayer(50)
  _     <- addScore(100)
  _     <- nextLevel
  msg   <- attack(30)
  score <- getScore
yield s"$msg, Score: $score"

val initialState = GameState(
  Player(100, 50, Point(0, 0)),
  Nil,
  0,
  1
)

val (finalState, message) = gameProgram.run(initialState).value
println(message)  // HP: 150 -> 120, Score: 100
println(finalState.level)  // 2
```

---

## สรุป

### Optic Hierarchy และการใช้งาน

```
Iso              - 1-to-1 bijection (ใช้ swap types)
├── Lens         - focus field ใน product type (required)
│   └── Optional - focus ที่อาจไม่มีอยู่
│       └── Traversal - focus หลาย elements
│           └── Fold - read-only Traversal
│               └── Getter - read-only Lens
└── Prism        - focus constructor ใน sum type
    └── Optional
```

### Composition Table

| + | Iso | Lens | Prism | Optional | Traversal |
|---|-----|------|-------|----------|-----------|
| **Iso** | Iso | Lens | Prism | Optional | Traversal |
| **Lens** | Lens | Lens | Optional | Optional | Traversal |
| **Prism** | Prism | Optional | Prism | Optional | Traversal |
| **Optional** | Optional | Optional | Optional | Optional | Traversal |
| **Traversal** | Traversal | Traversal | Traversal | Traversal | Traversal |

### Best Practices

1. **ใช้ macro** (`GenLens`, `GenPrism`) แทนการสร้าง manual เมื่อเป็นไปได้
2. **Compose optics** สำหรับ deep nesting แทนการ copy nested หลายชั้น
3. **Traversal + modify** สำหรับ bulk updates บน collections
4. **Laws**: ตรวจสอบ laws เสมอ (Lens 3 laws, Prism 2 laws)
5. **State Monad integration**: ใช้ optics กับ State สำหรับ stateful programs

---

*[← BONUS ส่วนที่ 102: Recursion Schemes](part-102-bonus-recursion-schemes.md) | [BONUS ส่วนที่ 104: Advanced Cats →](part-104-bonus-advanced-cats.md)*
