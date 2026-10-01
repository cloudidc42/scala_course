# Part 32: Advanced Type System

## สารบัญ
1. [GADTs](#gadts)
2. [Path-Dependent Types](#path-dependent-types)
3. [Singleton Types](#singleton-types)
4. [Type Lambdas](#type-lambdas)
5. [Match Types](#match-types)
6. [Opaque Types](#opaque-types)
7. [Union and Intersection Types](#union-and-intersection-types)

---

## GADTs

### Generalized Algebraic Data Types

```scala
// GADT: type parameter carries meaning about the data
sealed trait Expr[A]
case class Num(value: Int)                               extends Expr[Int]
case class Bool(value: Boolean)                          extends Expr[Boolean]
case class Add(left: Expr[Int], right: Expr[Int])        extends Expr[Int]
case class If[A](cond: Expr[Boolean], t: Expr[A], f: Expr[A]) extends Expr[A]
case class Equal[A](left: Expr[A], right: Expr[A])       extends Expr[Boolean]

// Type-safe evaluator
def eval[A](expr: Expr[A]): A = expr match
  case Num(n)         => n
  case Bool(b)        => b
  case Add(l, r)      => eval(l) + eval(r)
  case If(cond, t, f) => if eval(cond) then eval(t) else eval(f)
  case Equal(l, r)    => eval(l) == eval(r)

// ตัวอย่าง
val program = If(
  Equal(Add(Num(2), Num(3)), Num(5)),
  Num(100),
  Num(0)
)

println(eval(program))  // 100
// ผลลัพธ์เป็น Int เสมอ ตามชนิด Expr[Int]

// Type-safe pretty printer
def pretty[A](expr: Expr[A]): String = expr match
  case Num(n)         => n.toString
  case Bool(b)        => b.toString
  case Add(l, r)      => s"(${pretty(l)} + ${pretty(r)})"
  case If(c, t, f)    => s"if ${pretty(c)} then ${pretty(t)} else ${pretty(f)}"
  case Equal(l, r)    => s"(${pretty(l)} == ${pretty(r)})"
```

### Type-Safe Heterogeneous List

```scala
// HList: list with type-level length and element types
sealed trait HList
case object HNil extends HList
case class ::[H, T <: HList](head: H, tail: T) extends HList

type HNil = HNil.type

// Type alias for cleaner syntax
type ::[H, T <: HList] = ::[H, T]

// Smart constructor
def hnil: HNil = HNil
def hcons[H, T <: HList](h: H, t: T): ::[H, T] = new ::(h, t)

// Usage
val myList = hcons(1, hcons("hello", hcons(true, hnil)))
// Type: ::[Int, ::[String, ::[Boolean, HNil]]]

// Type-safe head extraction
def head[H, T <: HList](hlist: ::[H, T]): H = hlist.head

val firstElem: Int = head(myList)  // type-safe!
```

---

## Path-Dependent Types

### Type Members

```scala
// Types as members of an instance
trait Container:
  type Item
  def get: Item
  def set(item: Item): Container

class IntContainer(value: Int) extends Container:
  type Item = Int
  def get: Int = value
  def set(item: Int): IntContainer = IntContainer(item)

class StringContainer(value: String) extends Container:
  type Item = String
  def get: String = value
  def set(item: String): StringContainer = StringContainer(item)

// Path-dependent type: the type depends on a specific instance
val ic: IntContainer = IntContainer(42)
val sc: StringContainer = StringContainer("hello")

val x: ic.Item = ic.get  // ic.Item = Int
val y: sc.Item = sc.get  // sc.Item = String

// Type safety: can't mix
// val bad: ic.Item = sc.get  // ERROR!
```

### Database Abstraction with Type Members

```scala
// Type-safe DB abstraction
trait Database:
  type Record
  type Query

  def execute(q: Query): List[Record]
  def insert(r: Record): Unit

class UserDatabase extends Database:
  case class User(id: Long, name: String)
  case class UserQuery(minAge: Int, maxAge: Int)

  type Record = User
  type Query  = UserQuery

  private var users = List(User(1L, "Alice"), User(2L, "Bob"))

  def execute(q: UserQuery): List[User] = users  // simplified
  def insert(u: User): Unit = users = users :+ u

// Type-safe usage
val db = UserDatabase()
val users: List[db.Record] = db.execute(db.UserQuery(18, 65))
```

---

## Singleton Types

### Literal Singleton Types

```scala
// Singleton type: exactly one value
val x: 42 = 42
val y: "hello" = "hello"
val z: true = true

// Type-level string operations
import compiletime.ops.string.*

type Greeting = "Hello, " + "World!"
val greeting: Greeting = "Hello, World!"

// Literal types in type parameters
def requirePositive(n: Int): Unit =
  require(n > 0, s"Expected positive, got $n")

// Refined types (manual)
opaque type PositiveInt = Int
object PositiveInt:
  def apply(n: Int): Option[PositiveInt] =
    if n > 0 then Some(n.asInstanceOf[PositiveInt]) else None
  def unsafeApply(n: Int): PositiveInt =
    if n > 0 then n.asInstanceOf[PositiveInt]
    else throw IllegalArgumentException(s"Not positive: $n")

// ValueOf for singleton type evidence
def theValue[T](using v: ValueOf[T]): T = v.value

val forty2 = theValue[42]  // 42: Int
```

---

## Type Lambdas

### Higher-Kinded Type Aliases

```scala
// Type lambda: inline higher-kinded type
// Instead of: type EitherString[A] = Either[String, A]
// Use type lambda:
type EitherString = [A] =>> Either[String, A]

// Usage
def mapRight[F[_], A, B](fa: F[A])(f: A => B): F[B] = ???

// Type lambda as argument
def processEither[A](x: EitherString[A]): String =
  x.fold(err => s"Error: $err", v => s"Value: $v")

// Partial application of type constructors
type MapK[K] = [V] =>> Map[K, V]

type StringMap = MapK[String]
// StringMap[Int] = Map[String, Int]

// Functor for Either with fixed left type
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

// Using type lambda to create Functor for Either[String, *]
given Functor[[A] =>> Either[String, A]] with
  def map[A, B](fa: Either[String, A])(f: A => B): Either[String, B] =
    fa.map(f)

// Kind projector style (alternative)
// given Functor[Either[String, *]] with  // Scala 2 style
```

---

## Match Types

### Type-Level Pattern Matching

```scala
// Match type: select type based on type pattern
type Elem[X] = X match
  case Array[t]  => t
  case List[t]   => t
  case Option[t] => t
  case String    => Char
  case _         => X

// Test it
val a: Elem[Array[Int]]     = 1     // Int
val b: Elem[List[String]]   = "hi"  // String
val c: Elem[Option[Double]] = 3.14  // Double
val d: Elem[String]         = 'x'   // Char
val e: Elem[Boolean]        = true  // Boolean

// Type-safe element extraction
def getFirst[C](container: C): Elem[C] = container match
  case arr: Array[_]  => arr(0).asInstanceOf[Elem[C]]
  case lst: List[_]   => lst.head.asInstanceOf[Elem[C]]
  case opt: Option[_] => opt.get.asInstanceOf[Elem[C]]
  case str: String    => str(0).asInstanceOf[Elem[C]]

// Recursive match type
type Flattened[T] = T match
  case List[t] => Flattened[t]
  case _       => T

// type Flattened[List[List[Int]]] = Int

// Tuple element type
type TupleElem[T <: Tuple, N <: Int] = N match
  case 0 => T match
    case h *: t => h
  case S[n] => T match
    case h *: t => TupleElem[t, n]
```

---

## Opaque Types

### Type-Safe Newtypes

```scala
// Opaque type: alias with hidden implementation
opaque type Meters = Double
opaque type Kilograms = Double
opaque type Seconds = Double

object Meters:
  def apply(d: Double): Meters = d
  def value(m: Meters): Double = m
  extension (m: Meters)
    def toFeet: Double = m * 3.28084
    def +(other: Meters): Meters = m + other  // allowed
    def *(factor: Double): Meters = m * factor

object Kilograms:
  def apply(d: Double): Kilograms = d
  extension (k: Kilograms)
    def toPounds: Double = k * 2.20462
    def +(other: Kilograms): Kilograms = k + other

// Can't mix types
val dist = Meters(100.0)
val mass = Kilograms(70.0)
// val bad = dist + mass  // ERROR! Different types

// Phantom types for validation
opaque type Validated[A] = A
opaque type Unvalidated[A] = A

object Validated:
  def apply[A](a: A): Validated[A] = a
  extension [A](v: Validated[A]) def value: A = v

def validateEmail(raw: String): Either[String, Validated[String]] =
  if raw.contains("@") && raw.contains(".")
  then Right(Validated(raw))
  else Left(s"Invalid email: $raw")

def sendEmail(to: Validated[String], msg: String): Unit =
  println(s"Sending '$msg' to ${to.value}")

// Compile-time guarantee: must validate before using
val result = for
  email <- validateEmail("user@example.com")
  _     = sendEmail(email, "Hello!")
yield email
```

---

## Union and Intersection Types

### Union Types (A | B)

```scala
// Union type: value is one of the types
type StringOrInt = String | Int

def process(value: StringOrInt): String = value match
  case s: String => s"String: $s"
  case n: Int    => s"Int: $n"

process("hello")  // String: hello
process(42)       // Int: 42

// Union with null (Scala 3)
type Nullable[A] = A | Null

def safeDivide(a: Int, b: Int): Int | String =
  if b == 0 then "Division by zero"
  else a / b

// Pattern matching
safeDivide(10, 2) match
  case n: Int    => println(s"Result: $n")
  case s: String => println(s"Error: $s")

// Union type for errors
type DatabaseError   = "connection_failed" | "query_failed" | "timeout"
type ValidationError = "invalid_email" | "too_short" | "too_long"
type AppError        = DatabaseError | ValidationError

def handleError(e: AppError): String = e match
  case "connection_failed" => "Cannot connect to DB"
  case "query_failed"      => "Query execution failed"
  case "timeout"           => "Request timed out"
  case "invalid_email"     => "Email format is invalid"
  case "too_short"         => "Input is too short"
  case "too_long"          => "Input is too long"
```

### Intersection Types (A & B)

```scala
// Intersection type: must satisfy both constraints
trait HasName:
  def name: String

trait HasAge:
  def age: Int

trait Printable:
  def print(): Unit

// Intersection type
type Person = HasName & HasAge

def greet(p: HasName & HasAge): String =
  s"Hello, ${p.name}! You are ${p.age} years old."

// Anonymous class satisfying intersection
val alice = new HasName with HasAge:
  val name = "Alice"
  val age  = 30

greet(alice)  // Hello, Alice! You are 30 years old.

// Used in type bounds
def printAll[A <: HasName & HasAge & Printable](items: List[A]): Unit =
  items.foreach(_.print())

// Intersection for mixins
trait Logging:
  def log(msg: String): Unit = println(s"[LOG] $msg")

trait Metrics:
  def record(key: String, value: Double): Unit = println(s"[METRIC] $key=$value")

class Service extends Logging & Metrics:
  def doWork(): Unit =
    log("Starting work")
    record("work.started", 1.0)
    log("Work done")
    record("work.duration", 0.5)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ GADTs: type-safe evaluators
- ✅ Path-Dependent Types: type as object member
- ✅ Singleton Types: literal types
- ✅ Type Lambdas: `[A] =>> F[A]`
- ✅ Match Types: `T match { case ... }`
- ✅ Opaque Types: newtype pattern
- ✅ Union Types `A | B` and Intersection Types `A & B`

---

*[← Part 31: Macros](part-31-macros.md) | [Part 33: Cats Library →](part-33-cats.md)*
