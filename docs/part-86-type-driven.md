# ตอนที่ 86: Type-Driven Development

## สารบัญ

1. [การใช้ Types ป้องกัน Bugs](#การใช้-types-ป้องกัน-bugs)
2. [Phantom Types สำหรับ State Machines](#phantom-types-สำหรับ-state-machines)
3. [Refined Types กับ newtype library](#refined-types-กับ-newtype-library)
4. [Type-Level Computations](#type-level-computations)
5. [Type-Safe Builder Pattern](#type-safe-builder-pattern)
6. [Dependent Types Simulation](#dependent-types-simulation)
7. [Complete Type-Driven Design](#complete-type-driven-design)
8. [สรุป](#สรุป)

---

## การใช้ Types ป้องกัน Bugs

Type-Driven Development (TDD) คือแนวทางที่ใช้ Type System ของภาษาเป็นเครื่องมือในการ encode business rules และ constraints ทำให้ตรวจจับ bugs ได้ตั้งแต่ compile time

### ปัญหาของ Primitive Types

```scala
// ❌ แบบนี้ไม่ปลอดภัย - ส่งผิด parameter ได้ง่าย
def transferMoney(
  fromAccountId: String,
  toAccountId: String,
  amount: Double
): Unit = ???

// เรียกแบบนี้ผิด แต่ compile ผ่าน!
transferMoney(toAccountId, fromAccountId, -100.0)
```

### Newtype เพื่อสร้าง Semantic Types

```scala
// ✅ ใช้ Newtype ทำให้ type-safe
opaque type AccountId = String
opaque type Amount = BigDecimal
opaque type UserId = String
opaque type Email = String

object AccountId:
  def apply(s: String): Either[String, AccountId] =
    if s.matches("[A-Z]{2}[0-9]{10}") then Right(s)
    else Left(s"Invalid account ID format: $s")
  
  extension (id: AccountId)
    def value: String = id

object Amount:
  def apply(n: BigDecimal): Either[String, Amount] =
    if n > 0 then Right(n)
    else Left(s"Amount must be positive, got: $n")
  
  extension (a: Amount)
    def value: BigDecimal = a

// ตอนนี้ compiler จะจับ errors ให้
def transferMoney(
  from: AccountId,
  to: AccountId,
  amount: Amount
): Unit = ???

// ✅ ถูกต้อง
val from = AccountId("TH0123456789").getOrElse(???)
val to = AccountId("TH9876543210").getOrElse(???)
val amount = Amount(1000).getOrElse(???)
transferMoney(from, to, amount)

// ❌ Compile error! ไม่สามารถสลับ AccountId กับ UserId ได้
val userId: UserId = ???
// transferMoney(userId, to, amount)  // Type mismatch!
```

### Tagged Types สำหรับ Measurement Units

```scala
// สร้าง Tag types สำหรับ units
sealed trait Meters
sealed trait Kilometers
sealed trait Celsius
sealed trait Fahrenheit

opaque type Measurement[U] = Double

object Measurement:
  def apply[U](value: Double): Measurement[U] = value
  
  extension [U](m: Measurement[U])
    def value: Double = m
    def +(other: Measurement[U]): Measurement[U] = m + other
    def *(factor: Double): Measurement[U] = m * factor

type Meters_ = Measurement[Meters]
type Km_ = Measurement[Kilometers]
type Celsius_ = Measurement[Celsius]
type Fahrenheit_ = Measurement[Fahrenheit]

// Conversion functions
def metersToKm(m: Meters_): Km_ =
  Measurement[Kilometers](m.value / 1000.0)

def celsiusToFahrenheit(c: Celsius_): Fahrenheit_ =
  Measurement[Fahrenheit](c.value * 9.0 / 5.0 + 32)

// Usage
val distance: Meters_ = Measurement[Meters](5000)
val distanceKm: Km_ = metersToKm(distance)

val temp: Celsius_ = Measurement[Celsius](100)
val tempF: Fahrenheit_ = celsiusToFahrenheit(temp)

// ❌ Compile error! ไม่สามารถบวก Meters กับ Km ได้โดยตรง
// distance + distanceKm  // Type mismatch!
```

---

## Phantom Types สำหรับ State Machines

Phantom Types คือ Type parameters ที่ไม่ปรากฏอยู่ใน runtime value แต่ใช้ encode ข้อมูลที่ compile time

### Door State Machine

```scala
// Phantom type markers
sealed trait Open
sealed trait Closed
sealed trait Locked

// Door ที่มี phantom type S แสดง state
case class Door[S] private (id: String)

object Door:
  // สร้าง door ใหม่ในสถานะ Closed
  def create(id: String): Door[Closed] = Door(id)

// Operations ที่ valid ตาม state
extension [S](door: Door[Closed])
  def open(): Door[Open] =
    println(s"Opening door ${door.id}")
    Door(door.id)

extension (door: Door[Open])
  def close(): Door[Closed] =
    println(s"Closing door ${door.id}")
    Door(door.id)

extension (door: Door[Closed])
  def lock(): Door[Locked] =
    println(s"Locking door ${door.id}")
    Door(door.id)

extension (door: Door[Locked])
  def unlock(): Door[Closed] =
    println(s"Unlocking door ${door.id}")
    Door(door.id)

// Test
val door = Door.create("main-door")
val opened = door.open()
val closed = opened.close()
val locked = closed.lock()
val unlocked = locked.unlock()

// ❌ เหล่านี้จะเป็น Compile error:
// door.close()          // Can't close a closed door
// opened.lock()         // Can't lock an open door
// locked.open()         // Can't open a locked door
```

### HTTP Request State Machine

```scala
// State markers
sealed trait Uninitialized
sealed trait WithMethod
sealed trait WithUrl
sealed trait WithBody
sealed trait Ready

// HTTP Request builder ที่ type-safe
case class HttpRequestBuilder[S] private (
  method: Option[String] = None,
  url: Option[String] = None,
  headers: Map[String, String] = Map.empty,
  body: Option[String] = None
)

object HttpRequestBuilder:
  def apply(): HttpRequestBuilder[Uninitialized] = new HttpRequestBuilder()

extension (builder: HttpRequestBuilder[Uninitialized])
  def get(url: String): HttpRequestBuilder[Ready] =
    builder.copy(method = Some("GET"), url = Some(url))
    .asInstanceOf[HttpRequestBuilder[Ready]]
  
  def post(url: String): HttpRequestBuilder[WithUrl] =
    builder.copy(method = Some("POST"), url = Some(url))
    .asInstanceOf[HttpRequestBuilder[WithUrl]]
  
  def put(url: String): HttpRequestBuilder[WithUrl] =
    builder.copy(method = Some("PUT"), url = Some(url))
    .asInstanceOf[HttpRequestBuilder[WithUrl]]

extension (builder: HttpRequestBuilder[WithUrl])
  def withBody(body: String): HttpRequestBuilder[Ready] =
    builder.copy(body = Some(body))
    .asInstanceOf[HttpRequestBuilder[Ready]]

extension [S](builder: HttpRequestBuilder[S])
  def withHeader(key: String, value: String): HttpRequestBuilder[S] =
    builder.copy(headers = builder.headers + (key -> value))

extension (builder: HttpRequestBuilder[Ready])
  def build(): HttpRequest =
    HttpRequest(
      method  = builder.method.get,
      url     = builder.url.get,
      headers = builder.headers,
      body    = builder.body
    )

case class HttpRequest(method: String, url: String, headers: Map[String, String], body: Option[String])

// Usage
val getRequest = HttpRequestBuilder()
  .get("https://api.example.com/users")
  .withHeader("Authorization", "Bearer token123")
  .build()

val postRequest = HttpRequestBuilder()
  .post("https://api.example.com/users")
  .withHeader("Content-Type", "application/json")
  .withBody("""{"name": "Alice"}""")
  .build()

// ❌ Compile errors:
// HttpRequestBuilder().build()  // Uninitialized can't build
// HttpRequestBuilder().post("url").build()  // Missing body!
```

### Traffic Light State Machine

```scala
sealed trait Red
sealed trait Yellow
sealed trait Green

class TrafficLight[S] private (val color: String)

object TrafficLight:
  def start(): TrafficLight[Red] = new TrafficLight("Red")

extension (light: TrafficLight[Red])
  def toGreen(): TrafficLight[Green] =
    println("Red -> Green")
    new TrafficLight("Green")

extension (light: TrafficLight[Green])
  def toYellow(): TrafficLight[Yellow] =
    println("Green -> Yellow")
    new TrafficLight("Yellow")

extension (light: TrafficLight[Yellow])
  def toRed(): TrafficLight[Red] =
    println("Yellow -> Red")
    new TrafficLight("Red")

// ✅ Valid sequence
val light = TrafficLight.start()
  .toGreen()
  .toYellow()
  .toRed()
  .toGreen()

// ❌ Invalid sequences won't compile:
// TrafficLight.start().toYellow()   // Red can't go to Yellow directly
// TrafficLight.start().toGreen().toRed()  // Green can't go to Red directly
```

---

## Refined Types กับ newtype library

Refined types ช่วยเพิ่ม constraints ให้กับ types ที่มีอยู่แล้ว

### การ Implement Refined Types ด้วย Scala 3

```scala
// Refined type implementation
opaque type Refined[A, P] = A

object Refined:
  def apply[A, P](value: A)(predicate: A => Boolean, errorMsg: A => String): Either[String, Refined[A, P]] =
    if predicate(value) then Right(value)
    else Left(errorMsg(value))
  
  extension [A, P](r: Refined[A, P])
    def value: A = r

// Predicates
sealed trait Positive
sealed trait NonEmpty
sealed trait ValidEmail
sealed trait MinLength[N <: Int]
sealed trait MaxLength[N <: Int]
sealed trait Url

// Type aliases สำหรับ common refined types
type PositiveInt = Refined[Int, Positive]
type NonEmptyString = Refined[String, NonEmpty]
type ValidEmailStr = Refined[String, ValidEmail]

// Smart constructors
object PositiveInt:
  def apply(n: Int): Either[String, PositiveInt] =
    Refined[Int, Positive](n)(
      _ > 0,
      n => s"Expected positive integer, got: $n"
    )
  
  def unsafe(n: Int): PositiveInt =
    apply(n).getOrElse(throw IllegalArgumentException(s"$n is not positive"))

object NonEmptyString:
  def apply(s: String): Either[String, NonEmptyString] =
    Refined[String, NonEmpty](s)(
      _.nonEmpty,
      _ => "String cannot be empty"
    )

object ValidEmailStr:
  private val emailRegex = """^[^\s@]+@[^\s@]+\.[^\s@]+$""".r
  
  def apply(s: String): Either[String, ValidEmailStr] =
    Refined[String, ValidEmail](s)(
      emailRegex.matches,
      s => s"Invalid email: $s"
    )

// Example usage
case class UserProfile(
  name: NonEmptyString,
  email: ValidEmailStr,
  age: PositiveInt
)

def createProfile(name: String, email: String, age: Int): Either[String, UserProfile] =
  for
    validName  <- NonEmptyString(name)
    validEmail <- ValidEmailStr(email)
    validAge   <- PositiveInt(age)
  yield UserProfile(validName, validEmail, validAge)

// Test
createProfile("Alice", "alice@example.com", 30)
// Right(UserProfile(...))

createProfile("", "invalid-email", -5)
// Left("String cannot be empty")
```

### Numeric Constraints

```scala
// Type-level integers (simplified)
sealed trait Nat
sealed trait Zero extends Nat
sealed trait Succ[N <: Nat] extends Nat

type _0 = Zero
type _1 = Succ[_0]
type _2 = Succ[_1]
type _5 = Succ[Succ[Succ[Succ[Succ[_0]]]]]

// Bounded types
opaque type BoundedInt[Min <: Int & Singleton, Max <: Int & Singleton] = Int

object BoundedInt:
  def apply[Min <: Int & Singleton, Max <: Int & Singleton](
    value: Int
  )(using min: ValueOf[Min], max: ValueOf[Max]): Either[String, BoundedInt[Min, Max]] =
    if value >= min.value && value <= max.value then Right(value)
    else Left(s"Value $value out of range [${min.value}, ${max.value}]")
  
  extension [Min <: Int & Singleton, Max <: Int & Singleton](b: BoundedInt[Min, Max])
    def value: Int = b

// Specific bounded types
type Percentage = BoundedInt[0, 100]
type DayOfMonth = BoundedInt[1, 31]
type HourOfDay = BoundedInt[0, 23]

// Usage
val pct: Either[String, Percentage] = BoundedInt[0, 100](75)
val day: Either[String, DayOfMonth] = BoundedInt[1, 31](15)
val hour: Either[String, HourOfDay] = BoundedInt[0, 23](14)

// ❌ Invalid values
val badPct = BoundedInt[0, 100](150)  // Left("Value 150 out of range [0, 100]")
```

---

## Type-Level Computations

### HList - Heterogeneous Lists

```scala
// Type-level list
sealed trait HList
sealed trait HNil extends HList
case class ::[H, T <: HList](head: H, tail: T) extends HList

val hnil: HNil = new HNil {}
val list1 = 42 :: "hello" :: true :: hnil

// Type-safe access
def head[H, T <: HList](l: H :: T): H = l.head
def tail[H, T <: HList](l: H :: T): T = l.tail

val n: Int = head(list1)       // 42
val rest = tail(list1)          // "hello" :: true :: HNil
val s: String = head(rest)      // "hello"
```

### Type-Level Booleans

```scala
// Type-level booleans
sealed trait True
sealed trait False

type And[A, B] = (A, B) match
  case (True, True)   => True
  case _              => False

type Or[A, B] = (A, B) match
  case (False, False) => False
  case _              => True

type Not[A] = A match
  case True  => False
  case False => True

// Type-safe conditional
trait If[Cond, Then, Else]:
  type Result

object If:
  given [Then, Else]: If[True, Then, Else] with
    type Result = Then
  
  given [Then, Else]: If[False, Then, Else] with
    type Result = Else
```

### Type-Level Natural Numbers

```scala
// Peano numbers at type level
sealed trait Nat
case object Zero extends Nat
case class Succ[N <: Nat](n: N) extends Nat

type Nat0 = Zero.type
type Nat1 = Succ[Nat0]
type Nat2 = Succ[Nat1]
type Nat3 = Succ[Nat2]

// Fixed-size vector
case class Vec[N <: Nat, A](toList: List[A]):
  def head(using ev: N =:= Succ[?]): A = toList.head
  def tail(using ev: N =:= Succ[?]): Vec[?, A] = Vec(toList.tail)
  def length: Int = toList.length

object Vec:
  def empty[A]: Vec[Nat0, A] = Vec(List.empty)
  def one[A](a: A): Vec[Nat1, A] = Vec(List(a))
  def two[A](a: A, b: A): Vec[Nat2, A] = Vec(List(a, b))

// Sized collections
case class Matrix[Rows <: Nat, Cols <: Nat, A](
  data: Vec[Rows, Vec[Cols, A]]
)
```

---

## Type-Safe Builder Pattern

### Form Builder

```scala
// Type-level record
sealed trait RequiredField
sealed trait OptionalField
sealed trait ProvidedField

// Form field descriptor
case class Field[Name, Type, Status](value: Option[Type] = None)

// Form type with all required fields
trait FormShape

// Type-safe form builder
class FormBuilder[F <: FormShape, Fields <: HList](fields: Map[String, Any]):
  
  def set[T](name: String, value: T): FormBuilder[F, Fields] =
    new FormBuilder(fields + (name -> value))
  
  def build(using complete: FormComplete[F, Fields]): Form[F] =
    Form(fields)

case class Form[F](fields: Map[String, Any]):
  def get[T](key: String): Option[T] = fields.get(key).map(_.asInstanceOf[T])

// Evidence that form is complete
trait FormComplete[F, Fields]
```

### Type-Safe SQL Query Builder

```scala
// SQL query builder ที่ type-safe
sealed trait Select
sealed trait From
sealed trait Where
sealed trait Ordered
sealed trait Limited
sealed trait Complete

case class QueryBuilder[State](
  private val selectCols: List[String] = List("*"),
  private val fromTable: Option[String] = None,
  private val whereClause: Option[String] = None,
  private val orderByCol: Option[String] = None,
  private val limitVal: Option[Int] = None
)

object QueryBuilder:
  def apply(): QueryBuilder[Select] = new QueryBuilder()

extension (qb: QueryBuilder[Select])
  def select(cols: String*): QueryBuilder[From] =
    qb.copy(selectCols = cols.toList).asInstanceOf[QueryBuilder[From]]

extension (qb: QueryBuilder[From])
  def from(table: String): QueryBuilder[Where] =
    qb.copy(fromTable = Some(table)).asInstanceOf[QueryBuilder[Where]]

extension (qb: QueryBuilder[Where])
  def where(condition: String): QueryBuilder[Ordered] =
    qb.copy(whereClause = Some(condition)).asInstanceOf[QueryBuilder[Ordered]]
  
  def orderBy(col: String): QueryBuilder[Limited] =
    qb.copy(orderByCol = Some(col)).asInstanceOf[QueryBuilder[Limited]]
  
  def limit(n: Int): QueryBuilder[Complete] =
    qb.copy(limitVal = Some(n)).asInstanceOf[QueryBuilder[Complete]]

extension (qb: QueryBuilder[Ordered])
  def orderBy(col: String): QueryBuilder[Limited] =
    qb.copy(orderByCol = Some(col)).asInstanceOf[QueryBuilder[Limited]]

extension (qb: QueryBuilder[Limited])
  def limit(n: Int): QueryBuilder[Complete] =
    qb.copy(limitVal = Some(n)).asInstanceOf[QueryBuilder[Complete]]

extension [S >: Complete <: Complete](qb: QueryBuilder[Complete])
  def build(): String =
    val sb = StringBuilder()
    sb.append(s"SELECT ${qb.selectCols.mkString(", ")}")
    qb.fromTable.foreach(t => sb.append(s" FROM $t"))
    qb.whereClause.foreach(w => sb.append(s" WHERE $w"))
    qb.orderByCol.foreach(o => sb.append(s" ORDER BY $o"))
    qb.limitVal.foreach(l => sb.append(s" LIMIT $l"))
    sb.toString()

// Simplified builder ที่ใช้ง่ายขึ้น
class SqlBuilder private (private val parts: List[String]):
  
  def toSql: String = parts.mkString(" ")
  
  override def toString: String = toSql

object SqlBuilder:
  
  def select(cols: String*): SelectBuilder =
    new SelectBuilder(cols.toList)
  
  class SelectBuilder(cols: List[String]):
    def from(table: String): FromBuilder = new FromBuilder(cols, table)
  
  class FromBuilder(cols: List[String], table: String):
    def where(condition: String): WhereBuilder = new WhereBuilder(cols, table, Some(condition))
    def orderBy(col: String): OrderBuilder = new OrderBuilder(cols, table, None, Some(col))
    def limit(n: Int): LimitBuilder = new LimitBuilder(cols, table, None, None, Some(n))
    def build(): String = s"SELECT ${cols.mkString(", ")} FROM $table"
  
  class WhereBuilder(cols: List[String], table: String, where: Option[String]):
    def orderBy(col: String): OrderBuilder = new OrderBuilder(cols, table, where, Some(col))
    def limit(n: Int): LimitBuilder = new LimitBuilder(cols, table, where, None, Some(n))
    def build(): String =
      val w = where.map(c => s" WHERE $c").getOrElse("")
      s"SELECT ${cols.mkString(", ")} FROM $table$w"
  
  class OrderBuilder(cols: List[String], table: String, where: Option[String], order: Option[String]):
    def limit(n: Int): LimitBuilder = new LimitBuilder(cols, table, where, order, Some(n))
    def build(): String =
      val w = where.map(c => s" WHERE $c").getOrElse("")
      val o = order.map(c => s" ORDER BY $c").getOrElse("")
      s"SELECT ${cols.mkString(", ")} FROM $table$w$o"
  
  class LimitBuilder(cols: List[String], table: String, where: Option[String], 
                     order: Option[String], limit: Option[Int]):
    def build(): String =
      val w = where.map(c => s" WHERE $c").getOrElse("")
      val o = order.map(c => s" ORDER BY $c").getOrElse("")
      val l = limit.map(n => s" LIMIT $n").getOrElse("")
      s"SELECT ${cols.mkString(", ")} FROM $table$w$o$l"

// Usage
val query = SqlBuilder
  .select("id", "name", "email")
  .from("users")
  .where("age > 18")
  .orderBy("name")
  .limit(10)
  .build()
// "SELECT id, name, email FROM users WHERE age > 18 ORDER BY name LIMIT 10"
```

---

## Dependent Types Simulation

### Record Types

```scala
import scala.compiletime.*

// Type-safe record (row type)
sealed trait Row
sealed trait Empty extends Row
sealed trait Field[K <: String, V, Rest <: Row] extends Row

// Type-level lookup
type Lookup[R <: Row, K <: String] <: Any = R match
  case Field[K, v, ?]     => v
  case Field[?, ?, rest]  => Lookup[rest, K]

// Record value
case class Record[R <: Row](private val fields: Map[String, Any]):
  
  def get[K <: String & Singleton](key: K)(
    using ValueOf[K]
  ): Lookup[R, K] =
    fields(key).asInstanceOf[Lookup[R, K]]
  
  def set[K <: String & Singleton, V](key: K, value: V)(
    using ValueOf[K]
  ): Record[Field[K, V, R]] =
    Record(fields + (key -> value))

object Record:
  def empty: Record[Empty] = Record(Map.empty)

// Usage
val record = Record.empty
  .set("name", "Alice")
  .set("age", 30)
  .set("email", "alice@example.com")

val name: String = record.get("name")
val age: Int = record.get("age")
```

### Sized Collections

```scala
import scala.compiletime.ops.int.*

// Compile-time integer operations
type Min[A <: Int, B <: Int] <: Int = (A < B) match
  case true  => A
  case false => B

// Type-safe indexed access
opaque type BoundedIndex[N <: Int] = Int

object BoundedIndex:
  def apply[N <: Int](i: Int)(using n: ValueOf[N]): Option[BoundedIndex[N]] =
    if i >= 0 && i < n.value then Some(i) else None
  
  extension [N <: Int](idx: BoundedIndex[N])
    def value: Int = idx

// Fixed-size array
class SizedArray[N <: Int, A] private (private val data: Array[A]):
  
  def apply(i: BoundedIndex[N]): A = data(i.value)
  
  def updated(i: BoundedIndex[N], value: A): SizedArray[N, A] =
    val newData = data.clone()
    newData(i.value) = value
    new SizedArray(newData)
  
  def length: Int = data.length
  def toArray: Array[A] = data.clone()

object SizedArray:
  def apply[N <: Int: ValueOf, A: reflect.ClassTag](elements: A*): Option[SizedArray[N, A]] =
    val n = summon[ValueOf[N]].value
    if elements.length == n then Some(new SizedArray(elements.toArray))
    else None
```

---

## Complete Type-Driven Design

### E-commerce Order System

```scala
package com.example.typedrivenorder

import scala.util.Try

// =============================================================
// Domain Types
// =============================================================

opaque type OrderId = String
opaque type CustomerId = String
opaque type ProductId = String
opaque type Money = BigDecimal
opaque type Quantity = Int

object OrderId:
  def apply(s: String): Either[String, OrderId] =
    if s.nonEmpty && s.startsWith("ORD-") then Right(s)
    else Left(s"Invalid OrderId: $s")
  
  extension (id: OrderId) def value: String = id

object CustomerId:
  def apply(s: String): Either[String, CustomerId] =
    if s.nonEmpty then Right(s)
    else Left("CustomerId cannot be empty")
  
  extension (id: CustomerId) def value: String = id

object Money:
  def apply(amount: BigDecimal): Either[String, Money] =
    if amount >= 0 then Right(amount)
    else Left(s"Money cannot be negative: $amount")
  
  def zero: Money = BigDecimal(0)
  
  extension (m: Money)
    def value: BigDecimal = m
    def +(other: Money): Money = m + other
    def *(n: Int): Money = m * n
    def >(other: Money): Boolean = m > other

object Quantity:
  def apply(n: Int): Either[String, Quantity] =
    if n > 0 then Right(n)
    else Left(s"Quantity must be positive: $n")
  
  extension (q: Quantity)
    def value: Int = q

// =============================================================
// Order States (Phantom Types)
// =============================================================

sealed trait Draft
sealed trait Submitted
sealed trait Paid
sealed trait Shipped
sealed trait Delivered
sealed trait Cancelled

// =============================================================
// Product
// =============================================================

case class Product(
  id: ProductId,
  name: String,
  price: Money,
  stockQuantity: Int
)

// =============================================================
// Order Line Item
// =============================================================

case class OrderItem(
  productId: ProductId,
  productName: String,
  quantity: Quantity,
  unitPrice: Money
):
  def subtotal: Money = unitPrice * quantity.value

// =============================================================
// Order State Machine
// =============================================================

sealed class Order[S] private (
  val id: OrderId,
  val customerId: CustomerId,
  val items: List[OrderItem],
  val total: Money,
  val createdAt: Long,
  val notes: Option[String]
):
  override def toString: String =
    s"Order(${id.value}, customer=${customerId.value}, items=${items.length}, total=${total.value})"

object Order:
  // สร้าง order ใหม่
  def create(
    id: OrderId,
    customerId: CustomerId
  ): Order[Draft] =
    new Order(
      id = id,
      customerId = customerId,
      items = List.empty,
      total = Money.zero,
      createdAt = System.currentTimeMillis(),
      notes = None
    )

// Operations บน Draft orders
extension (order: Order[Draft])
  
  def addItem(item: OrderItem): Order[Draft] =
    val newItems = order.items :+ item
    val newTotal = newItems.foldLeft(Money.zero)(_ + _.subtotal)
    new Order(order.id, order.customerId, newItems, newTotal, order.createdAt, order.notes)
  
  def removeItem(productId: ProductId): Order[Draft] =
    val newItems = order.items.filterNot(_.productId == productId)
    val newTotal = newItems.foldLeft(Money.zero)(_ + _.subtotal)
    new Order(order.id, order.customerId, newItems, newTotal, order.createdAt, order.notes)
  
  def withNote(note: String): Order[Draft] =
    new Order(order.id, order.customerId, order.items, order.total, order.createdAt, Some(note))
  
  def submit(): Either[String, Order[Submitted]] =
    if order.items.isEmpty then Left("Cannot submit empty order")
    else Right(new Order(order.id, order.customerId, order.items, order.total, order.createdAt, order.notes))

// Operations บน Submitted orders
extension (order: Order[Submitted])
  def pay(paymentConfirmation: String): Order[Paid] =
    println(s"Payment confirmed: $paymentConfirmation for order ${order.id.value}")
    new Order(order.id, order.customerId, order.items, order.total, order.createdAt, order.notes)
  
  def cancel(reason: String): Order[Cancelled] =
    println(s"Order ${order.id.value} cancelled: $reason")
    new Order(order.id, order.customerId, order.items, order.total, order.createdAt, Some(reason))

// Operations บน Paid orders
extension (order: Order[Paid])
  def ship(trackingNumber: String): Order[Shipped] =
    println(s"Order ${order.id.value} shipped with tracking: $trackingNumber")
    new Order(order.id, order.customerId, order.items, order.total, order.createdAt, order.notes)

// Operations บน Shipped orders
extension (order: Order[Shipped])
  def deliver(): Order[Delivered] =
    println(s"Order ${order.id.value} delivered")
    new Order(order.id, order.customerId, order.items, order.total, order.createdAt, order.notes)

// =============================================================
// Order Service
// =============================================================

class OrderService(productCatalog: Map[ProductId, Product]):
  
  def createOrder(customerId: CustomerId): Either[String, Order[Draft]] =
    for
      orderId <- OrderId(s"ORD-${java.util.UUID.randomUUID().toString.take(8).toUpperCase}")
    yield Order.create(orderId, customerId)
  
  def addProductToOrder(
    order: Order[Draft],
    productId: ProductId,
    qty: Int
  ): Either[String, Order[Draft]] =
    for
      product  <- productCatalog.get(productId).toRight(s"Product $productId not found")
      quantity <- Quantity(qty)
      _        <- if product.stockQuantity >= qty then Right(())
                  else Left(s"Insufficient stock for ${product.name}")
      item = OrderItem(
        productId = productId,
        productName = product.name,
        quantity = quantity,
        unitPrice = product.price
      )
    yield order.addItem(item)
  
  def submitOrder(order: Order[Draft]): Either[String, Order[Submitted]] =
    order.submit()
  
  def processPayment(
    order: Order[Submitted],
    amount: Money,
    paymentMethod: String
  ): Either[String, Order[Paid]] =
    if amount > order.total then Left("Payment amount exceeds order total")
    else if amount.value < order.total.value then Left(s"Insufficient payment: ${amount.value} < ${order.total.value}")
    else Right(order.pay(s"PAY-${System.currentTimeMillis()}"))

// =============================================================
// Example Usage
// =============================================================

@main def runOrderExample(): Unit =
  // Setup
  val productCatalog: Map[ProductId, Product] = Map.empty // Simplified

  // This demonstrates the type-safe state machine
  println("Type-Driven Order System")
  println("========================")
  
  // Demonstrate compile-time safety
  val orderId = OrderId("ORD-12345678").getOrElse(???)
  val customerId = CustomerId("CUST-001").getOrElse(???)
  
  val draft: Order[Draft] = Order.create(orderId, customerId)
  println(s"Created draft order: $draft")
  
  // Must submit before paying - enforced at compile time
  val submitResult: Either[String, Order[Submitted]] = draft.submit()
  
  submitResult match
    case Right(submitted) =>
      val paid: Order[Paid] = submitted.pay("PAY-CONFIRM-001")
      val shipped: Order[Shipped] = paid.ship("TRACK-123")
      val delivered: Order[Delivered] = shipped.deliver()
      println(s"Order journey complete: $delivered")
    
    case Left(err) =>
      println(s"Submit failed: $err")
  
  // ❌ These would be COMPILE ERRORS:
  // draft.pay("...")        // Can't pay a draft
  // draft.ship("...")       // Can't ship a draft
  // submitted.deliver()     // Can't deliver without shipping
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Primitive Type Problems**: ปัญหาของการใช้ String, Int ตรงๆ
2. **Newtype/Opaque Types**: สร้าง domain-specific types ที่ไม่มี runtime overhead
3. **Phantom Types**: Encode state ที่ type level โดยไม่มี runtime data
4. **State Machines**: ใช้ phantom types สร้าง safe state transitions
5. **Refined Types**: เพิ่ม validation constraints ที่ type level
6. **Type-Level Computations**: Compute ที่ compile time
7. **Type-Safe Builders**: Build complex objects อย่าง safe
8. **Complete System**: ระบบ Order ที่ใช้ type-driven design ทั้งหมด

### หลักการสำคัญ

> "Make illegal states unrepresentable" - Yaron Minsky

การออกแบบแบบ Type-Driven ช่วยให้:
- Bugs ถูกจับได้ที่ Compile time แทน Runtime
- Code เป็น Documentation ในตัวเอง
- Refactoring ง่ายขึ้นเพราะ Compiler ช่วย guide

---

*[← ตอนที่ 85: Serverless Scala](part-85-serverless.md) | [ตอนที่ 87: Testing with Cats Effect →](part-87-cats-effect-testing.md)*
