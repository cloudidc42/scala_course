# Part 12: Generics และ Type System

## สารบัญ
1. [Generic Classes และ Methods](#generic-classes-และ-methods)
2. [Variance](#variance)
3. [Type Bounds](#type-bounds)
4. [Higher-Kinded Types](#higher-kinded-types)
5. [Type Members](#type-members)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Generic Classes และ Methods

### Generic Class พื้นฐาน

```scala
// Generic class
class Box[A](val value: A):
  def map[B](f: A => B): Box[B] = Box(f(value))
  def flatMap[B](f: A => Box[B]): Box[B] = f(value)
  def getOrElse[B >: A](default: B): B = value
  override def toString: String = s"Box($value)"

val intBox    = Box(42)
val strBox    = Box("hello")
val doubleBox = intBox.map(_.toDouble)

println(intBox)      // Box(42)
println(strBox)      // Box(hello)
println(doubleBox)   // Box(42.0)

// Generic method
def identity[A](a: A): A = a
def swap[A, B](pair: (A, B)): (B, A) = (pair._2, pair._1)
def first[A](list: List[A]): Option[A] = list.headOption
def repeat[A](a: A, n: Int): List[A] = List.fill(n)(a)

println(identity(42))               // 42
println(swap(("hello", 42)))        // (42,hello)
println(first(List(1, 2, 3)))       // Some(1)
println(repeat("scala", 3))         // List(scala, scala, scala)
```

### Generic Data Structures

```scala
// Generic Stack
class Stack[A]:
  private var items: List[A] = Nil

  def push(a: A): Stack[A] =
    items = a :: items
    this

  def pop(): Option[A] =
    items match
      case head :: tail =>
        items = tail
        Some(head)
      case Nil => None

  def peek: Option[A] = items.headOption
  def size: Int = items.size
  def isEmpty: Boolean = items.isEmpty
  override def toString: String = s"Stack(${items.mkString(", ")})"

val stack = Stack[Int]()
stack.push(1).push(2).push(3)
println(stack)          // Stack(3, 2, 1)
println(stack.pop())    // Some(3)
println(stack.peek)     // Some(2)

// Generic Pair
case class Pair[A, B](first: A, second: B):
  def swap: Pair[B, A] = Pair(second, first)
  def mapFirst[C](f: A => C): Pair[C, B] = Pair(f(first), second)
  def mapSecond[C](f: B => C): Pair[A, C] = Pair(first, f(second))
  def map[C, D](fa: A => C, fb: B => D): Pair[C, D] = Pair(fa(first), fb(second))

val pair = Pair("hello", 42)
println(pair.swap)           // Pair(42,hello)
println(pair.mapFirst(_.toUpperCase))  // Pair(HELLO,42)
```

---

## Variance

### Covariance (+A)

```scala
// Covariant: ถ้า B extends A แล้ว Container[B] extends Container[A]
sealed trait Animal
class Dog extends Animal
class Cat extends Animal

// Covariant container (read-only)
class ReadOnly[+A](val value: A):
  def get: A = value

val dog: ReadOnly[Dog] = ReadOnly(new Dog)
val animal: ReadOnly[Animal] = dog  // OK เพราะ ReadOnly เป็น covariant

// List เป็น covariant
val dogs: List[Dog] = List(new Dog, new Dog)
val animals: List[Animal] = dogs  // OK

// ทำไม covariant container ต้องเป็น read-only
// ถ้า ReadOnly[+A] มี def set(a: A): Unit ก็จะ error
// เพราะ:
//   val catContainer: ReadOnly[Cat] = ReadOnly(new Cat)
//   val animalContainer: ReadOnly[Animal] = catContainer
//   animalContainer.set(new Dog)  // ❌ กำลัง set Dog เข้า Cat container!
```

### Contravariance (-A)

```scala
// Contravariant: ถ้า B extends A แล้ว Container[A] extends Container[B]
// ใช้กับ "consumer" หรือ "input" positions

trait Serializer[-A]:
  def serialize(a: A): String

// Serializer สำหรับ Animal ใช้ได้กับ Dog
class AnimalSerializer extends Serializer[Animal]:
  def serialize(a: Animal): String = s"Animal: ${a.getClass.getSimpleName}"

val animalSer: Serializer[Animal] = new AnimalSerializer
val dogSer: Serializer[Dog] = animalSer  // OK เพราะ contravariant

// Function types เป็น contravariant ใน input, covariant ใน output
// Function1[-A, +B]
val animalToString: Animal => String = _.getClass.getSimpleName
val dogToString: Dog => String = animalToString  // OK
```

### Invariance (ไม่มี + หรือ -)

```scala
// Invariant: Container[A] ไม่ใช่ subtype ของ Container[B] เลย
class Mutable[A](var value: A)

val dogBox: Mutable[Dog] = Mutable(new Dog)
// val animalBox: Mutable[Animal] = dogBox  // ERROR! Invariant

// เหตุผล: ถ้า Mutable เป็น covariant
//   val animalBox: Mutable[Animal] = dogBox
//   animalBox.value = new Cat  // ❌ ใส่ Cat เข้า Dog box!

// Array ใน Scala เป็น invariant (ต่างจาก Java ที่ covariant)
val dogs2: Array[Dog] = Array(new Dog)
// val animals2: Array[Animal] = dogs2  // ERROR
```

---

## Type Bounds

### Upper Bounds (<:)

```scala
// A <: B หมายความว่า A ต้องเป็น subtype ของ B
def findMax[A <: Comparable[A]](list: List[A]): Option[A] =
  list.reduceOption((a, b) => if a.compareTo(b) >= 0 then a else b)

// กับ Ordered (มี compare)
def findMinMax[A <: Ordered[A]](list: List[A]): Option[(A, A)] =
  if list.isEmpty then None
  else Some((list.min, list.max))

// Numeric upper bound
def sum[A <: Numeric[A]](list: List[A])(using n: Numeric[A]): A =
  list.foldLeft(n.zero)(n.plus)

// Upper bound กับ case class
trait Named:
  def name: String

def sortByName[A <: Named](items: List[A]): List[A] =
  items.sortBy(_.name)

case class Product(name: String, price: Double) extends Named
case class Person(name: String, age: Int) extends Named

println(sortByName(List(Product("Zebra", 10), Product("Apple", 5))))
```

### Lower Bounds (>:)

```scala
// A >: B หมายความว่า A ต้องเป็น supertype ของ B
// ใช้บ่อยกับ covariant types

class Tree[+A]:
  def add[B >: A](elem: B): Tree[B] = ???  // B ต้องเป็น supertype ของ A

// prepend ของ List ใช้ lower bound
// def ::[B >: A](elem: B): List[B]
val ints: List[Int] = List(1, 2, 3)
val mixed: List[Any] = "hello" :: ints  // String >: Int? No, but Any >: both
```

### Context Bounds

```scala
// Context bound [A: TypeClass] เท่ากับ (using TypeClass[A])
def show[A: Show](a: A): String = summon[Show[A]].show(a)

// หลาย context bounds
def process[A: Ordering: Show](items: List[A]): String =
  val sorted = items.sorted
  sorted.map(summon[Show[A]].show).mkString(", ")

// Type Class instance
trait Show[A]:
  def show(a: A): String

given Show[Int] with
  def show(n: Int): String = n.toString

given Show[String] with
  def show(s: String): String = s""""$s""""

given [A: Show]: Show[List[A]] with
  def show(list: List[A]): String =
    list.map(summon[Show[A]].show).mkString("[", ", ", "]")

println(show(42))                    // 42
println(show("hello"))               // "hello"
println(show(List(1, 2, 3)))         // [1, 2, 3]
println(show(List("a", "b")))        // ["a", "b"]
```

---

## Higher-Kinded Types

### Type Constructors

```scala
// Type constructor: F[_] คือ type ที่รับ type parameter
// เช่น List[_], Option[_], Future[_], Either[_, _]

// Functor type class
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

// Generic function ที่ทำงานกับ Functor ใดๆ
def double[F[_]: Functor](container: F[Int]): F[Int] =
  summon[Functor[F]].map(container)(_ * 2)

println(double(List(1, 2, 3)))     // List(2, 4, 6)
println(double(Option(5)))          // Some(10)
println(double(None: Option[Int]))  // None
```

### Monad Pattern

```scala
trait Monad[F[_]] extends Functor[F]:
  def pure[A](a: A): F[A]
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  def map[A, B](fa: F[A])(f: A => B): F[B] =
    flatMap(fa)(a => pure(f(a)))

given Monad[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def flatMap[A, B](fa: Option[A])(f: A => Option[B]): Option[B] = fa.flatMap(f)

given Monad[List] with
  def pure[A](a: A): List[A] = List(a)
  def flatMap[A, B](fa: List[A])(f: A => List[B]): List[B] = fa.flatMap(f)

// Generic computation ที่ทำงานกับ Monad ใดๆ
def safeDiv[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
  val m = summon[Monad[F]]
  m.flatMap(a) { x =>
    m.flatMap(b) { y =>
      if y == 0 then m.pure(-1)  // simplified error handling
      else m.pure(x / y)
    }
  }
```

---

## Type Members

### Abstract Type Members

```scala
trait Container:
  type Element  // Abstract type member
  def empty: Element
  def add(e: Element): Container

class StringContainer extends Container:
  type Element = String
  private var items: List[String] = Nil
  def empty: String = ""
  def add(e: String): StringContainer =
    items = e :: items
    this

// Path-dependent types
class Graph:
  class Node(val label: String)
  class Edge(val from: Node, val to: Node)

val g1 = new Graph
val g2 = new Graph
val n1 = new g1.Node("A")
val n2 = new g1.Node("B")
// val e = new g2.Edge(n1, n2)  // ERROR! n1 is g1.Node, not g2.Node
val e = new g1.Edge(n1, n2)   // OK
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Generic Either-like Type

```scala
// สร้าง Validated type ที่สะสม errors แทน short-circuit
sealed trait Validated[+E, +A]:
  def map[B](f: A => B): Validated[E, B]
  def flatMap[EE >: E, B](f: A => Validated[EE, B]): Validated[EE, B]
  def mapError[F](f: E => F): Validated[F, A]

case class Valid[A](value: A) extends Validated[Nothing, A]:
  def map[B](f: A => B): Validated[Nothing, B] = Valid(f(value))
  def flatMap[EE, B](f: A => Validated[EE, B]): Validated[EE, B] = f(value)
  def mapError[F](f: Nothing => F): Validated[F, A] = this

case class Invalid[E](errors: List[E]) extends Validated[E, Nothing]:
  def map[B](f: Nothing => B): Validated[E, B] = this
  def flatMap[EE >: E, B](f: Nothing => Validated[EE, B]): Validated[EE, B] = this
  def mapError[F](f: E => F): Validated[F, Nothing] = Invalid(errors.map(f))

// TODO: implement combine (สะสม errors จากทั้งสอง side)
def combine[E, A, B, C](
  va: Validated[E, A],
  vb: Validated[E, B]
)(f: (A, B) => C): Validated[E, C] = ???
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Generic classes และ methods
- ✅ Variance: Covariant, Contravariant, Invariant
- ✅ Type Bounds: Upper (<:) และ Lower (>:) Bounds
- ✅ Context Bounds
- ✅ Higher-Kinded Types
- ✅ Type Members

---

*[← Part 11: Traits](part-11-traits.md) | [Part 13: Functional Programming →](part-13-functional-programming.md)*
