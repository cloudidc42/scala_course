# ส่วนที่ 95: Advanced Scala Type System

## สารบัญ

- [1. Dependent Types Simulation](#1-dependent-types-simulation)
- [2. Higher-Kinded Types เชิงลึก](#2-higher-kinded-types-เชิงลึก)
- [3. Type Class Coherence](#3-type-class-coherence)
- [4. Implicit Resolution อย่างละเอียด](#4-implicit-resolution-อย่างละเอียด)
- [5. Type Projections](#5-type-projections)
- [6. Existential Types](#6-existential-types)
- [7. Type-Level Programming ตัวอย่างสมบูรณ์](#7-type-level-programming-ตัวอย่างสมบูรณ์)
- [สรุป](#สรุป)

---

## 1. Dependent Types Simulation

ใน Scala 3 เราสามารถจำลอง dependent types ได้โดยใช้ path-dependent types และ singleton types

### Singleton Types และ Literal Types

```scala
// Literal types ใน Scala 3
val x: 42 = 42
val name: "Scala" = "Scala"
val flag: true = true

// Singleton type จาก value
def singletonType[T <: Singleton](value: T): T = value
val result = singletonType(42) // result: 42
```

### Path-Dependent Types

```scala
// Path-dependent types - type ขึ้นกับ instance
trait Container:
  type Element
  def get: Element
  def put(e: Element): Container

class IntContainer(value: Int) extends Container:
  type Element = Int
  def get: Int = value
  def put(e: Int): Container = IntContainer(e)

class StringContainer(value: String) extends Container:
  type Element = String
  def get: String = value
  def put(e: String): Container = StringContainer(e)

def processContainer(c: Container)(element: c.Element): c.Element =
  // element ต้องเป็น type เดียวกับที่ container นั้นๆ กำหนด
  element

val intC = IntContainer(10)
val strC = StringContainer("hello")

val intResult = processContainer(intC)(42)    // OK
val strResult = processContainer(strC)("world") // OK
// processContainer(intC)("wrong") // Error! String ไม่ใช่ intC.Element
```

### Dependent Function Types

```scala
// Dependent function type (Scala 3 feature)
type F = (c: Container) => c.Element

val getElement: F = c => c.get

// Dependent method type
def identify[C <: Container](c: C, e: c.Element): c.Element = e

// Type-safe heterogeneous list ใช้ dependent types
sealed trait HList:
  type Head
  type Tail <: HList

case class HCons[H, T <: HList](head: H, tail: T) extends HList:
  type Head = H
  type Tail = T

case object HNil extends HList:
  type Head = Nothing
  type Tail = HNil.type

// ใช้งาน
val hlist = HCons(1, HCons("hello", HCons(true, HNil)))
```

### Simulating Dependent Pairs (Sigma Types)

```scala
// Sigma type: มี value พร้อม type ที่ขึ้นกับ value นั้น
case class Sigma[A, B[_]](value: A, evidence: B[A])

// ตัวอย่าง: Vector ที่มี size เป็น type parameter
sealed trait Nat
case object Zero extends Nat
case class Succ[N <: Nat](n: N) extends Nat

class Vec[N <: Nat, A](private val data: List[A]):
  def size: Int = data.size
  def toList: List[A] = data

object Vec:
  def empty[A]: Vec[Zero.type, A] = Vec(List.empty)
  def cons[N <: Nat, A](head: A, tail: Vec[N, A]): Vec[Succ[N], A] =
    Vec(head :: tail.toList)

// สร้าง vector ที่มี size เป็น type
val empty = Vec.empty[Int]           // Vec[Zero.type, Int]
val one   = Vec.cons(1, empty)       // Vec[Succ[Zero.type], Int]
val two   = Vec.cons(2, one)         // Vec[Succ[Succ[Zero.type]], Int]
```

### Refined Types Pattern

```scala
// Simulating refined types ด้วย opaque types + smart constructors
opaque type PositiveInt = Int
opaque type Email = String
opaque type NonEmptyString = String

object PositiveInt:
  def apply(n: Int): Option[PositiveInt] =
    if n > 0 then Some(n) else None
  def unsafe(n: Int): PositiveInt =
    require(n > 0, s"$n must be positive")
    n
  extension (n: PositiveInt) def value: Int = n

object Email:
  private val pattern = "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$".r
  def apply(s: String): Option[Email] =
    if pattern.matches(s) then Some(s) else None
  extension (e: Email) def value: String = e

object NonEmptyString:
  def apply(s: String): Option[NonEmptyString] =
    if s.nonEmpty then Some(s) else None
  extension (s: NonEmptyString) def value: String = s

// ใช้งาน
case class User(
  name: NonEmptyString,
  email: Email,
  age: PositiveInt
)

def createUser(name: String, email: String, age: Int): Option[User] =
  for
    n <- NonEmptyString(name)
    e <- Email(email)
    a <- PositiveInt(age)
  yield User(n, e, a)

val user = createUser("Alice", "alice@example.com", 25)
```

---

## 2. Higher-Kinded Types เชิงลึก

Higher-kinded types คือ types ที่รับ type constructor เป็น parameter

### พื้นฐาน HKT

```scala
// Type ธรรมดา: Int, String, List[Int]
// Type constructor: List[_], Option[_], Either[String, _]
// Higher-kinded type: รับ type constructor เป็น parameter

// F[_] คือ type parameter ที่รับ type constructor
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

// Higher-order type class
trait Monad[F[_]] extends Functor[F]:
  def pure[A](a: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  
  override def map[A, B](fa: F[A])(f: A => B): F[B] =
    flatMap(fa)(a => pure(f(a)))
```

### Higher-Kinded Types ระดับสูง (HKT ของ HKT)

```scala
// Type constructor order 2: รับ type constructor ที่รับ type constructor
trait NaturalTransformation[F[_], G[_]]:
  def transform[A](fa: F[A]): G[A]

// หรือเขียนเป็น ~>
type ~>[F[_], G[_]] = NaturalTransformation[F, G]

// ตัวอย่าง: แปลง Option เป็น List
val optionToList: Option ~> List = new NaturalTransformation[Option, List]:
  def transform[A](fa: Option[A]): List[A] = fa.toList

// Monad Transformer - HKT ที่ใช้ HKT สร้าง HKT ใหม่
case class OptionT[F[_], A](value: F[Option[A]])

given [F[_]: Monad]: Monad[OptionT[F, *]] with
  def pure[A](a: A): OptionT[F, A] =
    OptionT(summon[Monad[F]].pure(Some(a)))
    
  def flatMap[A, B](fa: OptionT[F, A])(f: A => OptionT[F, B]): OptionT[F, B] =
    val F = summon[Monad[F]]
    OptionT(F.flatMap(fa.value) {
      case None    => F.pure(None)
      case Some(a) => f(a).value
    })
```

### Bifunctor และ Profunctor

```scala
// Bifunctor: map บน type parameter สองตัว
trait Bifunctor[F[_, _]]:
  def bimap[A, B, C, D](fab: F[A, B])(f: A => C, g: B => D): F[C, D]
  def leftMap[A, B, C](fab: F[A, B])(f: A => C): F[C, B] =
    bimap(fab)(f, identity)
  def rightMap[A, B, D](fab: F[A, B])(g: B => D): F[A, D] =
    bimap(fab)(identity, g)

given Bifunctor[Either] with
  def bimap[A, B, C, D](fab: Either[A, B])(f: A => C, g: B => D): Either[C, D] =
    fab.fold(a => Left(f(a)), b => Right(g(b)))

given Bifunctor[Tuple2] with
  def bimap[A, B, C, D](fab: (A, B))(f: A => C, g: B => D): (C, D) =
    (f(fab._1), g(fab._2))

// Profunctor: contramap บน input, map บน output
trait Profunctor[F[_, _]]:
  def dimap[A, B, C, D](fab: F[A, B])(f: C => A, g: B => D): F[C, D]

// Function1 เป็น Profunctor
given Profunctor[Function1] with
  def dimap[A, B, C, D](fab: A => B)(f: C => A, g: B => D): C => D =
    c => g(fab(f(c)))
```

### Kind Polymorphism

```scala
// Scala 3 มี kind polymorphism ผ่าน type lambda และ polymorphic functions
// Type Lambda syntax
type MyEither[E] = [A] =>> Either[E, A]
type MyReader[R] = [A] =>> R => A

// ใช้กับ type class
def functorForEither[E]: Functor[Either[E, *]] = new Functor[Either[E, *]]:
  def map[A, B](fa: Either[E, A])(f: A => B): Either[E, B] = fa.map(f)

// Polymorphic function ที่ทำงานกับ type ใดก็ได้
val identity: [A] => A => A = [A] => (a: A) => a

// Higher rank polymorphism
def applyPolymorphic[A, B](f: [T] => T => T, a: A, b: B): (A, B) =
  (f(a), f(b))
```

---

## 3. Type Class Coherence

Type class coherence หมายความว่า สำหรับ type ใดๆ ต้องมี instance เดียวเท่านั้น

### ปัญหา Incoherence

```scala
// ปัญหา: มี instance สองอันสำหรับ type เดียวกัน
trait Ordering[A]:
  def compare(x: A, y: A): Int

// instance แรก
given naturalOrdering: Ordering[Int] with
  def compare(x: Int, y: Int): Int = x.compareTo(y)

// instance ที่สอง (สมมติ)
// given reverseOrdering: Ordering[Int] with
//   def compare(x: Int, y: Int): Int = y.compareTo(x)
// นี่จะทำให้ ambiguous!

// วิธีแก้: ใช้ newtype pattern
opaque type Reversed[A] = A
object Reversed:
  def apply[A](a: A): Reversed[A] = a
  extension [A](r: Reversed[A]) def value: A = r

given [A: Ordering]: Ordering[Reversed[A]] with
  def compare(x: Reversed[A], y: Reversed[A]): Int =
    -summon[Ordering[A]].compare(x, y)
```

### Orphan Instances และ วิธีจัดการ

```scala
// Orphan instance: instance ที่อยู่นอก package ของ type หรือ type class
// Scala 3 แจ้งเตือนเรื่อง orphan instances

// วิธีที่ 1: ใส่ instance ใน companion object ของ type
case class Money(amount: BigDecimal, currency: String)
object Money:
  given Ordering[Money] with
    def compare(x: Money, y: Money): Int =
      x.amount.compareTo(y.amount)

// วิธีที่ 2: ใส่ instance ใน companion object ของ type class
trait Pretty[A]:
  def pretty(a: A): String

object Pretty:
  given Pretty[Int] with
    def pretty(n: Int): String = n.toString
  given Pretty[String] with
    def pretty(s: String): String = s""""$s""""
  given [A: Pretty]: Pretty[List[A]] with
    def pretty(list: List[A]): String =
      list.map(summon[Pretty[A]].pretty).mkString("[", ", ", "]")

// วิธีที่ 3: มี package สำหรับ instances
package instances:
  given Pretty[java.time.LocalDate] with
    def pretty(d: java.time.LocalDate): String = d.toString
```

### Superclasses และ Subclasses ใน Type Classes

```scala
// Hierarchy ของ type class
trait Semigroup[A]:
  def combine(x: A, y: A): A
  
  // กฎ: associativity
  // combine(combine(a, b), c) == combine(a, combine(b, c))

trait Monoid[A] extends Semigroup[A]:
  def empty: A
  
  // กฎ: identity
  // combine(empty, a) == a == combine(a, empty)

trait Group[A] extends Monoid[A]:
  def inverse(a: A): A
  
  // กฎ: inverse
  // combine(a, inverse(a)) == empty == combine(inverse(a), a)

// Instances
given Semigroup[String] with
  def combine(x: String, y: String): String = x + y

given Monoid[String] with
  def empty: String = ""
  def combine(x: String, y: String): String = x + y

given Semigroup[Int] with
  def combine(x: Int, y: Int): Int = x + y

given Monoid[Int] with
  def empty: Int = 0
  def combine(x: Int, y: Int): Int = x + y

// Derived instances
given [A: Monoid, B: Monoid]: Monoid[(A, B)] with
  def empty: (A, B) = (summon[Monoid[A]].empty, summon[Monoid[B]].empty)
  def combine(x: (A, B), y: (A, B)): (A, B) =
    (summon[Monoid[A]].combine(x._1, y._1),
     summon[Monoid[B]].combine(x._2, y._2))
```

---

## 4. Implicit Resolution อย่างละเอียด

### ลำดับการค้นหา Implicit

```scala
// Scala 3 ค้นหา given instances ตามลำดับ:
// 1. Local scope (ประกาศในบล็อกปัจจุบัน)
// 2. Import scope (ที่ import เข้ามา)
// 3. Outer scope (scope รอบนอก)
// 4. Companion objects ของ type ที่เกี่ยวข้อง

trait Show[A]:
  def show(a: A): String

// ใน companion object
case class Point(x: Int, y: Int)
object Point:
  given Show[Point] with
    def show(p: Point): String = s"(${p.x}, ${p.y})"

// การใช้งาน
def printIt[A: Show](a: A): Unit =
  println(summon[Show[A]].show(a))

printIt(Point(1, 2)) // ค้นหา given Show[Point] จาก Point companion
```

### Priority และ Ambiguity

```scala
// แก้ ambiguity ด้วย Priority trait
trait LowPriority:
  given [A]: Show[List[A]] with
    def show(list: List[A]): String = list.mkString("[", ", ", "]")

object Show extends LowPriority:
  // instance นี้มี priority สูงกว่า (override LowPriority)
  given [A: Show]: Show[List[A]] with
    def show(list: List[A]): String =
      list.map(summon[Show[A]].show).mkString("[", ", ", "]")
```

### Using Clauses และ Context Functions

```scala
// Context function type: (Using clause) => ReturnType
type Configured[A] = Config ?=> A

case class Config(debug: Boolean, maxRetries: Int)

def withConfig(config: Config): Configured[Unit] =
  // config ถูก inject เป็น given
  val c = summon[Config]
  if c.debug then println("Debug mode on")

// Context function เป็น first-class value
val process: Configured[String] = "processed"
val result = process(using Config(debug = true, maxRetries = 3))

// Composing context functions
def enriched[A](f: Configured[A]): Configured[A] =
  val config = summon[Config]
  println(s"Running with config: $config")
  f
```

### Implicit Conversion (ควรระวัง)

```scala
// Scala 3 ต้องการ import scala.language.implicitConversions อย่างชัดเจน
import scala.language.implicitConversions

given Conversion[String, Int] = _.length
val n: Int = "hello" // implicit conversion

// ทางที่ดีกว่า: ใช้ extension methods แทน
extension (s: String)
  def toIntLength: Int = s.length

// หรือใช้ explicit conversion
case class Wrapper[A](value: A)
given [A]: Conversion[A, Wrapper[A]] = Wrapper(_)
```

---

## 5. Type Projections

### Inner Types และ Type Projection

```scala
// Type projection: เข้าถึง inner type ด้วย #
class Outer:
  class Inner
  type InnerAlias = Inner

// Type projection syntax
type InnerType = Outer#Inner

def createInner(o: Outer): o.Inner = new o.Inner

// แตกต่างกัน:
// o.Inner = path-dependent type, specific to instance o
// Outer#Inner = type projection, ไม่ขึ้นกับ instance
```

### Type Members vs Type Parameters

```scala
// Type Member
trait Container1:
  type Element
  def empty: Element
  def add(e: Element): Container1

// Type Parameter
trait Container2[Element]:
  def empty: Element
  def add(e: Element): Container2[Element]

// เมื่อไรใช้ type member?
// - เมื่อ type ควรถูก inferred
// - เมื่อต้องการ encapsulation
// - เมื่อ type ขึ้นกับ instance

// เมื่อไรใช้ type parameter?
// - เมื่อต้องการ parametric polymorphism
// - เมื่อ type ควรปรากฏใน interface
// - เมื่อต้องการ variance annotations

// Existential ผ่าน type member
def printContainer(c: Container1): Unit =
  // ไม่รู้ว่า c.Element คืออะไร แต่ทำงานได้
  println(c.empty)
```

### Abstract Type Members และ Refinements

```scala
// Abstract type member
abstract class Graph:
  type Node
  type Edge
  
  def nodes: Set[Node]
  def edges: Set[Edge]
  def adjacent(n1: Node, n2: Node): Boolean

// Refinement type
type IntGraph = Graph { type Node = Int; type Edge = (Int, Int) }

// Structural type (duck typing)
type Closeable = { def close(): Unit }

def withResource[A <: Closeable, B](resource: A)(f: A => B): B =
  try f(resource)
  finally resource.close()
```

---

## 6. Existential Types

### Wildcard Types

```scala
// Existential type: ไม่รู้ type parameter แต่รู้ว่ามันมีอยู่
// ใน Scala 3 ใช้ wildcard _ หรือ ?

val lists: List[List[?]] = List(List(1, 2), List("a", "b"), List(true))

// การทำงานกับ existential type ต้องระวัง
def sizeOf(list: List[?]): Int = list.size

// ไม่สามารถทำสิ่งที่ต้องการ type ได้
// def head(list: List[?]): ? = list.head // ไม่ compile
def headAsAny(list: List[?]): Any = list.head // OK ด้วย Any
```

### Existential Type ด้วย Type Class

```scala
// Existential + type class: ใช้ type class กับ existential type
trait Showable:
  type A
  val value: A
  val show: Show[A]
  
  def showValue: String = show.show(value)

object Showable:
  def apply[T: Show](v: T): Showable = new Showable:
    type A = T
    val value: T = v
    val show: Show[T] = summon[Show[T]]

given Show[Int] with
  def show(n: Int): String = n.toString

given Show[String] with
  def show(s: String): String = s""""$s""""

val showables: List[Showable] = List(
  Showable(42),
  Showable("hello"),
  Showable(100)
)

showables.foreach(s => println(s.showValue))
```

### Existential Types ใน Practice

```scala
// GADTs ใน Scala 3 (Generalized Algebraic Data Types)
enum Expr[A]:
  case Num(n: Int)    extends Expr[Int]
  case Bool(b: Boolean) extends Expr[Boolean]
  case Add(l: Expr[Int], r: Expr[Int]) extends Expr[Int]
  case If[A](cond: Expr[Boolean], t: Expr[A], f: Expr[A]) extends Expr[A]
  case Eq[A](l: Expr[A], r: Expr[A]) extends Expr[Boolean]

def eval[A](expr: Expr[A]): A = expr match
  case Expr.Num(n)     => n
  case Expr.Bool(b)    => b
  case Expr.Add(l, r)  => eval(l) + eval(r)
  case Expr.If(c, t, f) => if eval(c) then eval(t) else eval(f)
  case Expr.Eq(l, r)   => eval(l) == eval(r)

// ใช้งาน - type safe!
val prog: Expr[Int] = Expr.If(
  Expr.Eq(Expr.Num(1), Expr.Num(1)),
  Expr.Add(Expr.Num(10), Expr.Num(20)),
  Expr.Num(0)
)
val result = eval(prog) // result: Int = 30
```

---

## 7. Type-Level Programming ตัวอย่างสมบูรณ์

### Natural Numbers ที่ Type Level

```scala
// Peano numbers ที่ type level
sealed trait Nat
sealed trait Zero extends Nat
sealed trait Succ[N <: Nat] extends Nat

// Type-level aliases
type One   = Succ[Zero]
type Two   = Succ[One]
type Three = Succ[Two]
type Four  = Succ[Three]
type Five  = Succ[Four]

// Type-level addition
type Add[A <: Nat, B <: Nat] <: Nat
// ใน Scala 3 ใช้ match types
type Add2[A <: Nat, B <: Nat] <: Nat = A match
  case Zero    => B
  case Succ[n] => Succ[Add2[n, B]]

// Sized vector ด้วย type-level nat
class Vec[N <: Nat, +A](val toList: List[A]):
  def ::[B >: A](head: B): Vec[Succ[N], B] = Vec(head :: toList)

object Vec:
  def empty[A]: Vec[Zero, A] = Vec(Nil)

def append[N <: Nat, M <: Nat, A](
  v1: Vec[N, A], v2: Vec[M, A]
): Vec[Add2[N, M], A] =
  Vec(v1.toList ++ v2.toList)
```

### Match Types

```scala
// Match types: type-level pattern matching (Scala 3)
type Head[X] = X match
  case h *: _ => h

type Tail[X] = X match
  case _ *: t => t

type Length[X <: Tuple] <: Int = X match
  case EmptyTuple => 0
  case _ *: t     => S[Length[t]]

// Tuple operations ด้วย match types
type Concat[X <: Tuple, Y <: Tuple] <: Tuple = X match
  case EmptyTuple => Y
  case h *: t     => h *: Concat[t, Y]

// ทดสอบ
val t1: (Int, String) = (1, "a")
val t2: (Boolean, Double) = (true, 3.14)

type T1 = Int *: String *: EmptyTuple
type T2 = Boolean *: Double *: EmptyTuple
type T3 = Concat[T1, T2] // Int *: String *: Boolean *: Double *: EmptyTuple
```

### Type-Level State Machine

```scala
// State machine ที่ encode state ใน type
sealed trait State
sealed trait Locked extends State
sealed trait Unlocked extends State

class Door[S <: State] private:
  def this() = this()

object Door:
  def locked: Door[Locked] = Door()
  
  extension (d: Door[Locked])
    def unlock(key: String): Door[Unlocked] =
      if key == "secret" then Door()
      else throw new Exception("Wrong key!")
  
  extension (d: Door[Unlocked])
    def lock: Door[Locked] = Door()
    def enter: String = "Welcome!"
    def exit: String = "Goodbye!"

// ใช้งาน
val door = Door.locked
// door.enter // Compile error! Cannot enter locked door
val unlocked = door.unlock("secret")
println(unlocked.enter) // OK
val locked = unlocked.lock
// locked.enter // Compile error again!
```

### Type-Level JSON Schema

```scala
// สร้าง JSON schema ที่ type safe ด้วย type-level programming
import scala.deriving.*
import scala.compiletime.*

// Type class สำหรับ JSON encoding
trait JsonEncoder[A]:
  def encode(a: A): String

given JsonEncoder[Int] with
  def encode(n: Int): String = n.toString

given JsonEncoder[String] with
  def encode(s: String): String = s""""$s""""

given JsonEncoder[Boolean] with
  def encode(b: Boolean): String = b.toString

given [A: JsonEncoder]: JsonEncoder[List[A]] with
  def encode(list: List[A]): String =
    list.map(summon[JsonEncoder[A]].encode).mkString("[", ",", "]")

given [A: JsonEncoder]: JsonEncoder[Option[A]] with
  def encode(opt: Option[A]): String = opt match
    case None    => "null"
    case Some(a) => summon[JsonEncoder[A]].encode(a)

// Automatic derivation สำหรับ case classes
inline def summonAll[T <: Tuple]: List[JsonEncoder[?]] =
  inline erasedValue[T] match
    case _: EmptyTuple => Nil
    case _: (t *: ts) => summonInline[JsonEncoder[t]] :: summonAll[ts]

inline given [A <: Product](using mirror: Mirror.ProductOf[A]): JsonEncoder[A] with
  def encode(a: A): String =
    val encoders = summonAll[mirror.MirroredElemTypes]
    val labels   = constValueTuple[mirror.MirroredElemLabels].toList.map(_.toString)
    val values   = a.productIterator.toList
    
    val fields = labels.zip(values).zip(encoders).map:
      case ((label, value), encoder) =>
        val encoded = encoder.asInstanceOf[JsonEncoder[Any]].encode(value)
        s""""$label":$encoded"""
    
    fields.mkString("{", ",", "}")

// ทดสอบ
case class Person(name: String, age: Int, active: Boolean)
case class Company(name: String, employees: List[Person])

val person = Person("Alice", 30, true)
val company = Company("Acme", List(Person("Bob", 25, true), Person("Charlie", 35, false)))

println(summon[JsonEncoder[Person]].encode(person))
println(summon[JsonEncoder[Company]].encode(company))
```

### Complete Type-Level Computation Example

```scala
// สร้าง type-safe query builder
sealed trait Column[A]:
  def name: String

case class IntColumn(name: String) extends Column[Int]
case class StringColumn(name: String) extends Column[String]
case class BoolColumn(name: String) extends Column[Boolean]

// Predicate ที่ type safe
sealed trait Predicate
case class Eq[A](col: Column[A], value: A) extends Predicate
case class Gt(col: Column[Int], value: Int) extends Predicate
case class And(l: Predicate, r: Predicate) extends Predicate
case class Or(l: Predicate, r: Predicate) extends Predicate

// Query builder
case class Query[Cols <: Tuple](
  table: String,
  columns: List[Column[?]],
  predicate: Option[Predicate] = None,
  limit: Option[Int] = None
):
  def where(pred: Predicate): Query[Cols] =
    copy(predicate = Some(pred))
  
  def limit(n: Int): Query[Cols] =
    copy(limit = Some(n))
  
  def toSQL: String =
    val cols = if columns.isEmpty then "*" else columns.map(_.name).mkString(", ")
    val where = predicate.map(p => s" WHERE ${renderPredicate(p)}").getOrElse("")
    val lim = limit.map(l => s" LIMIT $l").getOrElse("")
    s"SELECT $cols FROM $table$where$lim"

def renderPredicate(pred: Predicate): String = pred match
  case Eq(col, value) => s"${col.name} = ${renderValue(value)}"
  case Gt(col, value) => s"${col.name} > $value"
  case And(l, r)      => s"(${renderPredicate(l)} AND ${renderPredicate(r)})"
  case Or(l, r)       => s"(${renderPredicate(l)} OR ${renderPredicate(r)})"

def renderValue(value: Any): String = value match
  case s: String  => s""""$s""""
  case n: Int     => n.toString
  case b: Boolean => b.toString
  case other      => other.toString

// ใช้งาน
val nameCol = StringColumn("name")
val ageCol  = IntColumn("age")
val activeCol = BoolColumn("active")

val query = Query[(String, Int, Boolean)](
  table = "users",
  columns = List(nameCol, ageCol, activeCol)
).where(
  And(Gt(ageCol, 18), Eq(activeCol, true))
).limit(10)

println(query.toSQL)
// SELECT name, age, active FROM users WHERE (age > 18 AND active = true) LIMIT 10
```

---

## สรุป

Advanced Type System ของ Scala 3 ให้เครื่องมือที่ทรงพลัง:

| เทคนิค | ใช้เมื่อ |
|--------|---------|
| Dependent Types | Type ต้องขึ้นกับ runtime value |
| Higher-Kinded Types | Abstract เหนือ type constructors |
| Type Class Coherence | รับประกัน instance เดียวต่อ type |
| Implicit Resolution | เข้าใจลำดับการค้นหา given |
| Type Projections | Inner types และ type members |
| Existential Types | ทำงานกับ unknown type parameters |
| Type-Level Programming | Computation ที่ compile time |

### ข้อควรระวัง

- Type-level programming ทำให้ compile time นานขึ้น
- Error messages จาก HKT อาจอ่านยาก
- ใช้ความซับซ้อนเท่าที่จำเป็น อย่า over-engineer
- ทดสอบ type-level code ด้วย compile-time assertions

---

*[← Part 94: Best Practices](part-94-best-practices.md) | [Part 96: Concurrent Data Structures →](part-96-concurrent-data.md)*
