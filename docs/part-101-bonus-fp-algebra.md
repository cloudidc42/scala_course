# ส่วนที่ 101 (BONUS): Abstract Algebra และ Category Theory ใน Scala

> **BONUS CONTENT** - เนื้อหาขั้นสูงสำหรับผู้ที่ต้องการเข้าใจรากฐานทางคณิตศาสตร์ของ Functional Programming

---

## สารบัญ

1. [บทนำ: ทำไมต้องเรียน Algebra?](#บทนำ)
2. [Semigroup](#semigroup)
3. [Monoid](#monoid)
4. [Group](#group)
5. [Functor](#functor)
6. [Applicative Functor](#applicative-functor)
7. [Monad](#monad)
8. [Natural Transformations](#natural-transformations)
9. [Profunctors](#profunctors)
10. [Adjunctions](#adjunctions)
11. [การประยุกต์ใช้จริงใน Scala](#การประยุกต์ใช้จริง)
12. [สรุป](#สรุป)

---

## บทนำ

Abstract Algebra และ Category Theory เป็นรากฐานทางคณิตศาสตร์ที่อยู่เบื้องหลัง Functional Programming หลายแนวคิดใน libraries อย่าง Cats และ Scalaz มาจากทฤษฎีเหล่านี้โดยตรง

```scala
// เราใช้ Scala 3 และ Cats ในตัวอย่างทั้งหมด
import cats.*
import cats.implicits.*
import cats.kernel.*
```

### ทำไม Category Theory ถึงสำคัญ?

```scala
// โค้ดที่ดูเหมือนแตกต่างกัน แต่มีโครงสร้างเดียวกัน
val sumInts: List[Int] => Int = _.sum
val concatStrings: List[String] => String = _.mkString
val andBooleans: List[Boolean] => Boolean = _.forall(identity)

// ทั้งหมดนี้คือ Monoid fold!
// เมื่อเข้าใจ Monoid เราเข้าใจทั้งสามพร้อมกัน
def monoFold[A: Monoid](list: List[A]): A =
  list.foldLeft(Monoid[A].empty)(Monoid[A].combine)
```

---

## Semigroup

Semigroup คือ set ที่มี binary operation ที่ associative

### กฎของ Semigroup

```
combine(combine(a, b), c) == combine(a, combine(b, c))  // Associativity
```

### การนิยาม Semigroup ใน Scala 3

```scala
// Type class definition
trait Semigroup[A]:
  def combine(x: A, y: A): A
  
  // Extension method สำหรับ syntax ที่สวยงาม
  extension (x: A)
    def |+|(y: A): A = combine(x, y)

// Instances
given Semigroup[Int] with
  def combine(x: Int, y: Int): Int = x + y

given Semigroup[String] with
  def combine(x: String, y: String): String = x + y

given [A]: Semigroup[List[A]] with
  def combine(x: List[A], y: List[A]): List[A] = x ++ y
```

### ตัวอย่างการใช้งาน

```scala
import cats.Semigroup
import cats.syntax.semigroup.*

// Basic usage
val sum = 3 |+| 4 |+| 5  // 12
val concat = "Hello" |+| " " |+| "World"  // "Hello World"
val merged = List(1, 2) |+| List(3, 4)  // List(1, 2, 3, 4)

// Semigroup สำหรับ Option
given [A: Semigroup]: Semigroup[Option[A]] with
  def combine(x: Option[A], y: Option[A]): Option[A] =
    (x, y) match
      case (Some(a), Some(b)) => Some(a |+| b)
      case (Some(a), None)    => Some(a)
      case (None, Some(b))    => Some(b)
      case (None, None)       => None

val optSum = Some(3) |+| Some(4)  // Some(7)
val optLeft = Some("hi") |+| None  // Some("hi")
```

### Semigroup สำหรับ Map

```scala
// Map Semigroup รวม values ด้วย value's Semigroup
given [K, V: Semigroup]: Semigroup[Map[K, V]] with
  def combine(x: Map[K, V], y: Map[K, V]): Map[K, V] =
    y.foldLeft(x) { case (acc, (k, v)) =>
      acc.updated(k, acc.get(k).fold(v)(_ |+| v))
    }

val map1 = Map("a" -> 1, "b" -> 2)
val map2 = Map("b" -> 3, "c" -> 4)
val merged = map1 |+| map2  // Map("a" -> 1, "b" -> 5, "c" -> 4)
```

### การพิสูจน์ Law ด้วย ScalaCheck

```scala
import org.scalacheck.*
import org.scalacheck.Prop.*

def semigroupLaws[A: Semigroup: Arbitrary]: Properties =
  new Properties("Semigroup"):
    property("associativity") = forAll { (a: A, b: A, c: A) =>
      (a |+| b) |+| c == a |+| (b |+| c)
    }
```

---

## Monoid

Monoid คือ Semigroup ที่มี identity element (empty)

### กฎของ Monoid

```
combine(empty, a) == a      // Left identity
combine(a, empty) == a      // Right identity
combine(combine(a, b), c) == combine(a, combine(b, c))  // Associativity (inherited)
```

### การนิยาม Monoid

```scala
trait Monoid[A] extends Semigroup[A]:
  def empty: A

// Instances
given Monoid[Int] with
  def empty: Int = 0
  def combine(x: Int, y: Int): Int = x + y

given Monoid[String] with
  def empty: String = ""
  def combine(x: String, y: String): String = x + y

given [A]: Monoid[List[A]] with
  def empty: List[A] = Nil
  def combine(x: List[A], y: List[A]): List[A] = x ++ y

// Product Monoid
given [A: Monoid, B: Monoid]: Monoid[(A, B)] with
  def empty: (A, B) = (Monoid[A].empty, Monoid[B].empty)
  def combine(x: (A, B), y: (A, B)): (A, B) =
    (x._1 |+| y._1, x._2 |+| y._2)
```

### การใช้ Monoid สำหรับ fold

```scala
import cats.Monoid
import cats.syntax.monoid.*

def fold[A: Monoid](list: List[A]): A =
  list.foldLeft(Monoid[A].empty)(_ |+| _)

// หรือใช้ combineAll จาก Cats
val sumAll = List(1, 2, 3, 4, 5).combineAll  // 15
val concatAll = List("a", "b", "c").combineAll  // "abc"

// Monoid ช่วยให้เราสามารถรวม parallel computations
case class Stats(count: Int, sum: Double, sumSq: Double):
  def mean: Double = if count == 0 then 0 else sum / count
  def variance: Double = if count == 0 then 0 else sumSq / count - mean * mean

given Monoid[Stats] with
  def empty: Stats = Stats(0, 0.0, 0.0)
  def combine(x: Stats, y: Stats): Stats =
    Stats(x.count + y.count, x.sum + y.sum, x.sumSq + y.sumSq)

def toStats(x: Double): Stats = Stats(1, x, x * x)

val data = List(1.0, 2.0, 3.0, 4.0, 5.0)
val stats = data.map(toStats).combineAll
println(s"Mean: ${stats.mean}, Variance: ${stats.variance}")
```

### Monoid ใน Parallel Computation

```scala
import cats.effect.IO
import cats.effect.implicits.*

// Map-Reduce ด้วย Monoid
def mapReduce[A, B: Monoid](data: List[A])(f: A => B): B =
  data.map(f).combineAll

// Parallel version
def parallelMapReduce[A, B: Monoid](data: List[A])(f: A => IO[B]): IO[B] =
  data.parTraverse(f).map(_.combineAll)
```

---

## Group

Group คือ Monoid ที่ทุก element มี inverse

### กฎของ Group

```
combine(a, inverse(a)) == empty  // Right inverse
combine(inverse(a), a) == empty  // Left inverse
```

### การนิยาม Group

```scala
trait Group[A] extends Monoid[A]:
  def inverse(a: A): A
  
  def remove(x: A, y: A): A = combine(x, inverse(y))

// Int addition forms a group
given Group[Int] with
  def empty: Int = 0
  def combine(x: Int, y: Int): Int = x + y
  def inverse(a: Int): Int = -a

// Double multiplication (non-zero) forms a group
// But we need to handle 0 carefully
opaque type NonZeroDouble = Double
object NonZeroDouble:
  def apply(d: Double): Option[NonZeroDouble] =
    if d != 0.0 then Some(d) else None

given Group[NonZeroDouble] with
  def empty: NonZeroDouble = 1.0
  def combine(x: NonZeroDouble, y: NonZeroDouble): NonZeroDouble = x * y
  def inverse(a: NonZeroDouble): NonZeroDouble = 1.0 / a
```

---

## Functor

Functor คือ structure ที่สามารถ map ได้ - นิยามในภาษา Category Theory ว่าเป็น morphism ระหว่าง categories

### กฎของ Functor

```
map(fa)(identity) == fa                    // Identity
map(map(fa)(f))(g) == map(fa)(f andThen g) // Composition
```

### การนิยาม Functor

```scala
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]
  
  // Derived operations
  def void[A](fa: F[A]): F[Unit] = map(fa)(_ => ())
  def as[A, B](fa: F[A])(b: B): F[B] = map(fa)(_ => b)
  def fmap[A, B](f: A => B): F[A] => F[B] = fa => map(fa)(f)
  
  // Lift function into Functor
  def lift[A, B](f: A => B): F[A] => F[B] = fmap(f)

// Instances
given Functor[List] with
  def map[A, B](fa: List[A])(f: A => B): List[B] = fa.map(f)

given Functor[Option] with
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)

given [E]: Functor[Either[E, *]] with
  def map[A, B](fa: Either[E, A])(f: A => B): Either[E, B] = fa.map(f)

// Function Functor (contravariant in input, covariant in output)
given [R]: Functor[Function1[R, *]] with
  def map[A, B](fa: R => A)(f: A => B): R => B = fa andThen f
```

### Covariant vs Contravariant Functor

```scala
// Covariant Functor: map ไปข้างหน้า
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

// Contravariant Functor: map ย้อนกลับ
trait Contravariant[F[_]]:
  def contramap[A, B](fa: F[A])(f: B => A): F[B]

// ตัวอย่าง: Ordering (Comparator)
given Contravariant[Ordering] with
  def contramap[A, B](fa: Ordering[A])(f: B => A): Ordering[B] =
    (x, y) => fa.compare(f(x), f(y))

// ใช้ contramap เพื่อ derive Ordering
val personByAge: Ordering[Person] =
  Ordering.Int.contramap(_.age)

val personByName: Ordering[Person] =
  Ordering.String.contramap(_.name)
```

### Invariant Functor

```scala
// Invariant Functor: ต้องการทั้ง A => B และ B => A
trait Invariant[F[_]]:
  def imap[A, B](fa: F[A])(f: A => B)(g: B => A): F[B]

// Codec เป็นตัวอย่างของ Invariant Functor
case class Codec[A](encode: A => String, decode: String => A)

given Invariant[Codec] with
  def imap[A, B](fa: Codec[A])(f: A => B)(g: B => A): Codec[B] =
    Codec(
      encode = b => fa.encode(g(b)),
      decode = s => f(fa.decode(s))
    )

val intCodec: Codec[Int] = Codec(_.toString, _.toInt)
val doubleCodec: Codec[Double] = intCodec.imap(_.toDouble)(_.toInt)
```

---

## Applicative Functor

Applicative เป็น Functor ที่สามารถ apply function ที่อยู่ใน context ได้

### กฎของ Applicative

```
ap(pure(id))(fa) == fa                                    // Identity
ap(ap(ap(pure(compose))(fab))(fbc))(fa) == ap(fbc)(ap(fab)(fa))  // Composition
ap(pure(f))(pure(a)) == pure(f(a))                       // Homomorphism
ap(ff)(pure(a)) == ap(pure(f => f(a)))(ff)              // Interchange
```

### การนิยาม Applicative

```scala
trait Applicative[F[_]] extends Functor[F]:
  def pure[A](a: A): F[A]
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]
  
  // map สามารถ derive ได้จาก pure และ ap
  override def map[A, B](fa: F[A])(f: A => B): F[B] =
    ap(pure(f))(fa)
  
  // Derived operations
  def map2[A, B, C](fa: F[A], fb: F[B])(f: (A, B) => C): F[C] =
    ap(map(fa)(a => (b: B) => f(a, b)))(fb)
  
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)] =
    map2(fa, fb)((a, b) => (a, b))
  
  def *>[A, B](fa: F[A])(fb: F[B]): F[B] = map2(fa, fb)((_, b) => b)
  def <*[A, B](fa: F[A])(fb: F[B]): F[A] = map2(fa, fb)((a, _) => a)

given Applicative[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def ap[A, B](ff: Option[A => B])(fa: Option[A]): Option[B] =
    (ff, fa) match
      case (Some(f), Some(a)) => Some(f(a))
      case _                  => None
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)
```

### Validation ด้วย Applicative

```scala
import cats.data.Validated
import cats.data.Validated.*
import cats.syntax.validated.*
import cats.syntax.apply.*

type ValidationResult[A] = Validated[List[String], A]

def validateAge(age: Int): ValidationResult[Int] =
  if age >= 0 && age <= 150 then age.valid
  else List(s"Invalid age: $age").invalid

def validateName(name: String): ValidationResult[String] =
  if name.nonEmpty then name.valid
  else List("Name cannot be empty").invalid

def validateEmail(email: String): ValidationResult[String] =
  if email.contains("@") then email.valid
  else List(s"Invalid email: $email").invalid

case class User(name: String, age: Int, email: String)

// Applicative validation สะสม errors ทั้งหมด (ไม่หยุดที่ error แรก)
def createUser(name: String, age: Int, email: String): ValidationResult[User] =
  (validateName(name), validateAge(age), validateEmail(email)).mapN(User.apply)

// Test
println(createUser("Alice", 30, "alice@example.com"))
// Valid(User(Alice,30,alice@example.com))

println(createUser("", -1, "not-an-email"))
// Invalid(List(Name cannot be empty, Invalid age: -1, Invalid email: not-an-email))
```

---

## Monad

Monad คือ Applicative ที่มี flatMap (bind) - ช่วยให้ sequence computations ที่ depend on กันได้

### กฎของ Monad

```
flatMap(pure(a))(f) == f(a)                             // Left identity
flatMap(ma)(pure) == ma                                 // Right identity
flatMap(flatMap(ma)(f))(g) == flatMap(ma)(a => flatMap(f(a))(g))  // Associativity
```

### การนิยาม Monad

```scala
trait Monad[F[_]] extends Applicative[F]:
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  
  // derived
  override def ap[A, B](ff: F[A => B])(fa: F[A]): F[B] =
    flatMap(ff)(f => map(fa)(f))
  
  def flatten[A](ffa: F[F[A]]): F[A] = flatMap(ffa)(identity)
  
  // Kleisli composition
  def compose[A, B, C](f: A => F[B], g: B => F[C]): A => F[C] =
    a => flatMap(f(a))(g)

given Monad[Option] with
  def pure[A](a: A): Option[A] = Some(a)
  def flatMap[A, B](fa: Option[A])(f: A => Option[B]): Option[B] = fa.flatMap(f)
  def map[A, B](fa: Option[A])(f: A => B): Option[B] = fa.map(f)
```

### Kleisli Arrow

```scala
import cats.data.Kleisli

// Kleisli[F, A, B] คือ A => F[B]
type Kleisli[F[_], A, B] = A => F[B]

// Composition ของ Kleisli arrows
val parseAge: String => Option[Int] = s => s.toIntOption
val validateAge: Int => Option[Int] = n => if n > 0 then Some(n) else None
val toAdultAge: Int => Option[String] = n => if n >= 18 then Some(s"Adult: $n") else None

// Kleisli composition
val processAge = Kleisli(parseAge) andThen Kleisli(validateAge) andThen Kleisli(toAdultAge)

println(processAge("25"))  // Some(Adult: 25)
println(processAge("-5"))  // None
println(processAge("abc")) // None
```

### Reader Monad

```scala
// Reader Monad = Kleisli[Id, R, A] = R => A
type Reader[R, A] = Kleisli[cats.Id, R, A]

case class Config(dbUrl: String, port: Int, debug: Boolean)

type AppReader[A] = Reader[Config, A]

val getDbUrl: AppReader[String] = Kleisli(_.dbUrl)
val getPort: AppReader[Int] = Kleisli(_.port)
val isDebug: AppReader[Boolean] = Kleisli(_.debug)

val appInfo: AppReader[String] = for
  url   <- getDbUrl
  port  <- getPort
  debug <- isDebug
yield s"DB: $url, Port: $port, Debug: $debug"

val config = Config("localhost:5432", 8080, true)
println(appInfo(config))
// DB: localhost:5432, Port: 8080, Debug: true
```

### State Monad

```scala
import cats.data.State

type Stack[A] = State[List[Int], A]

val push: Int => Stack[Unit] = n =>
  State.modify(stack => n :: stack)

val pop: Stack[Option[Int]] =
  State.get[List[Int]].flatMap {
    case Nil       => State.pure(None)
    case h :: rest => State.set(rest).as(Some(h))
  }

val stackOps: Stack[List[Option[Int]]] = for
  _  <- push(1)
  _  <- push(2)
  _  <- push(3)
  a  <- pop
  b  <- pop
  _  <- push(10)
  c  <- pop
yield List(a, b, c)

val (finalStack, results) = stackOps.run(Nil).value
println(s"Stack: $finalStack, Results: $results")
// Stack: List(1), Results: List(Some(3), Some(2), Some(10))
```

---

## Natural Transformations

Natural Transformation คือ morphism ระหว่าง Functors ที่รักษา structure

### นิยาม

```
// สำหรับทุก f: A => B, diagram ต้องสมมาตร:
// nat.transform(fa.map(f)) == nat.transform(fa).map(f)
```

### การนิยามใน Scala

```scala
// Natural Transformation F ~> G
trait FunctionK[F[_], G[_]]:
  def apply[A](fa: F[A]): G[A]

// Alias
type ~>[F[_], G[_]] = FunctionK[F, G]

// ตัวอย่าง: Option ~> List
val optionToList: Option ~> List = new (Option ~> List):
  def apply[A](fa: Option[A]): List[A] = fa.toList

// List ~> Option
val listToOption: List ~> Option = new (List ~> Option):
  def apply[A](fa: List[A]): Option[A] = fa.headOption

// ตัวอย่างการใช้
val opt: Option[Int] = Some(42)
val list: List[Int] = optionToList(opt)  // List(42)

val lst: List[String] = List("a", "b", "c")
val maybeFirst: Option[String] = listToOption(lst)  // Some("a")
```

### Composition ของ Natural Transformations

```scala
// Natural transformations compose!
val optToList: Option ~> List = optionToList
val listToOpt: List ~> Option = listToOption

// Compose: Option ~> Option (round trip)
val roundTrip: Option ~> Option = new (Option ~> Option):
  def apply[A](fa: Option[A]): Option[A] =
    listToOpt(optToList(fa))

// Identity Natural Transformation
val idNat: Option ~> Option = new (Option ~> Option):
  def apply[A](fa: Option[A]): Option[A] = fa
```

### ใน Cats

```scala
import cats.~>

val safeHead: List ~> Option = new (List ~> Option):
  def apply[A](fa: List[A]): Option[A] = fa.headOption

// ใช้กับ Free Monad หรือ Interpreter Pattern
// Natural Transformation มักใช้เป็น interpreter
```

---

## Profunctors

Profunctor คือ type constructor ที่ contravariant ใน argument แรกและ covariant ใน argument ที่สอง

### นิยาม

```scala
trait Profunctor[F[_, _]]:
  def dimap[A, B, C, D](fab: F[A, B])(f: C => A)(g: B => D): F[C, D]
  
  // Derived
  def lmap[A, B, C](fab: F[A, B])(f: C => A): F[C, B] =
    dimap(fab)(f)(identity)
  
  def rmap[A, B, D](fab: F[A, B])(g: B => D): F[A, D] =
    dimap(fab)(identity)(g)

// Function1 เป็น Profunctor
given Profunctor[Function1] with
  def dimap[A, B, C, D](fab: A => B)(f: C => A)(g: B => D): C => D =
    f andThen fab andThen g
```

### Strong Profunctor

```scala
trait Strong[F[_, _]] extends Profunctor[F]:
  def first[A, B, C](fab: F[A, B]): F[(A, C), (B, C)]
  def second[A, B, C](fab: F[A, B]): F[(C, A), (C, B)]

// Function1 เป็น Strong Profunctor
given Strong[Function1] with
  def dimap[A, B, C, D](fab: A => B)(f: C => A)(g: B => D): C => D =
    f andThen fab andThen g
  
  def first[A, B, C](fab: A => B): ((A, C)) => (B, C) =
    (a, c) => (fab(a), c)
  
  def second[A, B, C](fab: A => B): ((C, A)) => (C, B) =
    (c, a) => (c, fab(a))
```

### Lens เป็น Profunctor (Preview)

```scala
// Lens คือ Strong Profunctor!
// นี่คือรากฐานของ Optics
case class Lens[S, A](get: S => A, set: S => A => S)

// dimap: เปลี่ยน type ของ source และ focus
def dimapLens[S, T, A, B](lab: Lens[S, A])(f: T => S)(g: A => B): Lens[T, B] =
  Lens(
    get = t => g(lab.get(f(t))),
    set = t => b => f(???) // นี่ไม่สมบูรณ์ - Lens ต้องการ Iso ในการ dimap เต็มรูปแบบ
  )
```

---

## Adjunctions

Adjunction คือความสัมพันธ์พิเศษระหว่าง Functors สองตัว

### นิยาม

F ⊣ G (F is left adjoint to G) ถ้า:
```
Hom(F(A), B) ≅ Hom(A, G(B))
```

ใน Scala หมายความว่า:
```
F[A] => B  ≅  A => G[B]
```

### การนิยายใน Scala

```scala
trait Adjunction[F[_], G[_]]:
  def unit[A](a: A): G[F[A]]
  def counit[A](fga: F[G[A]]): A
  
  // Derived
  def leftAdjunct[A, B](f: F[A] => B): A => G[B] =
    a => unit(a).map(fa => f(fa)) // ต้องการ Functor[G]
  
  def rightAdjunct[A, B](f: A => G[B]): F[A] => B =
    fa => counit(fa.map(ga => ???)) // ซับซ้อนกว่านี้
```

### ตัวอย่างจริง: Curry/Uncurry

```scala
// (A, B) => C  ≅  A => (B => C)
// นี่คือ Adjunction ระหว่าง Product และ Function!

def curry[A, B, C](f: (A, B) => C): A => B => C = a => b => f(a, b)
def uncurry[A, B, C](f: A => B => C): (A, B) => C = (a, b) => f(a)(b)

// Monad สามารถ derive ได้จาก Adjunction!
// ถ้า F ⊣ G แล้ว G ∘ F เป็น Monad

// State Monad มาจาก Adjunction
// State[S, A] = S => (A, S) = Reader[S, (A, S)] = Reader[S] ∘ Writer[S]
```

### Free/Forgetful Adjunction

```scala
// List เป็น free Monoid
// Forgetful functor ลืม Monoid structure
// Free ∘ Forgetful = List (เกือบ)

// ทุก Monad มาจาก Adjunction
// Free Monad มาจาก Forgetful Adjunction
```

---

## การประยุกต์ใช้จริง

### Algebra สำหรับ Domain Model

```scala
import cats.*
import cats.implicits.*

// Domain: Shopping Cart
case class Money(amount: BigDecimal, currency: String)

given Semigroup[Money] with
  def combine(x: Money, y: Money): Money =
    require(x.currency == y.currency, "Cannot add different currencies")
    Money(x.amount + y.amount, x.currency)

given Monoid[Money] with
  def empty: Money = Money(0, "THB")
  def combine(x: Money, y: Money): Money =
    if x.amount == 0 then y
    else if y.amount == 0 then x
    else
      require(x.currency == y.currency, "Cannot add different currencies")
      Money(x.amount + y.amount, x.currency)

case class CartItem(product: String, price: Money, quantity: Int):
  def total: Money = Money(price.amount * quantity, price.currency)

case class Cart(items: List[CartItem]):
  def total: Money = items.map(_.total).combineAll

// ใช้งาน
val cart = Cart(List(
  CartItem("Apple", Money(30, "THB"), 3),
  CartItem("Banana", Money(20, "THB"), 5),
  CartItem("Cherry", Money(150, "THB"), 1)
))

println(s"Total: ${cart.total}")  // Total: Money(390,THB)
```

### Functor สำหรับ Transformation Pipeline

```scala
// Pipeline ที่ใช้ Functor
case class Pipeline[F[_]: Functor, A](value: F[A]):
  def transform[B](f: A => B): Pipeline[F, B] =
    Pipeline(value.map(f))
  
  def filter(pred: A => Boolean)(using Functor[F], Alternative[F]): Pipeline[F, A] =
    Pipeline(value) // simplified

// Data transformation pipeline
val rawData: List[String] = List("1", "2", "bad", "3", "4")
val parsed: List[Option[Int]] = rawData.map(_.toIntOption)
val validated: List[Option[Int]] = parsed.map(_.filter(_ > 0))

// ใช้ traverse เพื่อ invert List[Option[A]] => Option[List[A]]
val allValid: Option[List[Int]] = parsed.sequence
// None เพราะมี "bad"

val withDefaults: List[Int] = parsed.map(_.getOrElse(0))
// List(1, 2, 0, 3, 4)
```

### Monad Transformer Stack

```scala
import cats.data.*
import cats.effect.IO

// สร้าง effect stack ด้วย Monad Transformers
type AppError = String
type AppState = Map[String, Int]

// EitherT[StateT[IO, AppState, *], AppError, A]
type App[A] = EitherT[StateT[IO, AppState, *], AppError, A]

def getCounter(key: String): App[Int] =
  EitherT.liftF(StateT.get[IO, AppState].map(_.getOrElse(key, 0)))

def incrementCounter(key: String): App[Unit] =
  EitherT.liftF(StateT.modify[IO, AppState](state =>
    state.updated(key, state.getOrElse(key, 0) + 1)
  ))

def failIfTooHigh(key: String, max: Int): App[Unit] =
  for
    count <- getCounter(key)
    _     <- if count > max
              then EitherT.leftT[StateT[IO, AppState, *], Unit](s"$key exceeded $max")
              else EitherT.pure[StateT[IO, AppState, *], AppError](())
  yield ()
```

### Category Theory Pattern: Free Monad Interpreter

```scala
import cats.free.Free
import cats.free.Free.*

// Define algebra as ADT
sealed trait KVStoreA[A]
case class Put[T](key: String, value: T) extends KVStoreA[Unit]
case class Get[T](key: String) extends KVStoreA[Option[T]]
case class Delete(key: String) extends KVStoreA[Unit]

type KVStore[A] = Free[KVStoreA, A]

// Smart constructors
def put[T](key: String, value: T): KVStore[Unit] =
  liftF[KVStoreA, Unit](Put[T](key, value))

def get[T](key: String): KVStore[Option[T]] =
  liftF[KVStoreA, Option[T]](Get[T](key))

def delete(key: String): KVStore[Unit] =
  liftF[KVStoreA, Unit](Delete(key))

// Program ที่ไม่ผูกติดกับ implementation
val program: KVStore[Option[Int]] = for
  _   <- put("age", 25)
  _   <- put("score", 100)
  age <- get[Int]("age")
  _   <- delete("score")
yield age

// Interpreter 1: In-memory
import cats.~>
import scala.collection.mutable

val impureInterpreter: KVStoreA ~> cats.Id = new (KVStoreA ~> cats.Id):
  val store = mutable.Map.empty[String, Any]
  
  def apply[A](fa: KVStoreA[A]): A = fa match
    case Put(key, value) => store(key) = value
    case Get(key)        => store.get(key).map(_.asInstanceOf[A])
    case Delete(key)     => store.remove(key)
  
  // Workaround: ต้องการ cast เพิ่มเติม
  override def apply[A](fa: KVStoreA[A]): cats.Id[A] = fa match
    case p: Put[t]  => store(p.key) = p.value
    case g: Get[t]  => store.get(g.key).asInstanceOf[A]
    case d: Delete  => store.remove(d.key); ().asInstanceOf[A]

// Run program
val result = program.foldMap(impureInterpreter)
println(result)  // Some(25)
```

---

## สรุป

| แนวคิด | กฎ | ใน Cats | ใช้งาน |
|--------|-----|---------|--------|
| Semigroup | Associativity | `Semigroup[A]` | รวม values |
| Monoid | Semigroup + Identity | `Monoid[A]` | fold, reduce |
| Group | Monoid + Inverse | `Group[A]` | undo operations |
| Functor | map identity/composition | `Functor[F]` | transform values |
| Applicative | Functor + pure + ap | `Applicative[F]` | independent effects |
| Monad | Applicative + flatMap | `Monad[F]` | sequential effects |
| Natural Transform | Morphism between functors | `F ~> G` | interpreters |
| Profunctor | Contravariant + Covariant | `Profunctor[F]` | optics, arrows |
| Adjunction | F ⊣ G | - | theory foundation |

### Key Insights

1. **Laws matter**: type class laws ทำให้โค้ด predictable และ composable
2. **Abstraction pays off**: เมื่อ abstract ถูกต้อง โค้ดจะ reuse ได้มาก
3. **Category Theory เป็นภาษา**: ช่วยให้สื่อสาร patterns ได้ชัดเจน
4. **Monad comes from Adjunction**: ทุก Monad มีรากฐานใน Adjunction
5. **Profunctors เป็นพื้นฐานของ Optics**: Lens, Prism ล้วนเป็น Profunctors

---

*[← ส่วนที่ 94: Best Practices](part-94-best-practices.md) | [BONUS ส่วนที่ 102: Recursion Schemes →](part-102-bonus-recursion-schemes.md)*
