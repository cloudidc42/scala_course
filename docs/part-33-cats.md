# Part 33: Cats Library

## สารบัญ
1. [Cats Overview](#cats-overview)
2. [Type Classes in Cats](#type-classes-in-cats)
3. [Functor, Applicative, Monad](#functor-applicative-monad)
4. [Validated](#validated)
5. [NonEmptyList](#nonemptylist)
6. [State and Writer](#state-and-writer)

---

## Cats Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.typelevel" %% "cats-core"   % "2.10.0",
  "org.typelevel" %% "cats-effect" % "3.5.2",
  "org.typelevel" %% "cats-free"   % "2.10.0"
)

// Add compiler plugin for better type inference
addCompilerPlugin("org.typelevel" % "kind-projector" % "0.13.3" cross CrossVersion.full)
```

### Core Concepts

```
Cats (Category Theory abstractions):
- Functor:      map   F[A] => (A => B) => F[B]
- Apply:        ap    F[A => B] => F[A] => F[B]
- Applicative:  pure  A => F[A]
- FlatMap:      flatMap F[A] => (A => F[B]) => F[B]
- Monad:        pure + flatMap
- Foldable:     fold, foldLeft, foldRight
- Traverse:     traverse, sequence

Type classes for values:
- Eq:           type-safe equality
- Order:        type-safe ordering
- Show:         string representation
- Monoid:       combine + empty
- Semigroup:    combine only
```

---

## Type Classes in Cats

### Eq, Show, Order

```scala
import cats.*
import cats.syntax.all.*

// Eq: type-safe equality
case class Point(x: Int, y: Int)

given Eq[Point] = Eq.fromUniversalEquals  // uses ==
// or
given Eq[Point] = Eq.instance((a, b) => a.x == b.x && a.y == b.y)

val p1 = Point(1, 2)
val p2 = Point(1, 2)
val p3 = Point(3, 4)

println(p1 === p2)  // true  (type-safe)
println(p1 =!= p3) // true
// println(p1 == "hello")  // won't compile with strict mode

// Show: display
given Show[Point] = Show.show(p => s"Point(${p.x}, ${p.y})")

println(p1.show)  // Point(1, 2)

// Monoid: combine elements with identity
given Monoid[Point] = new Monoid[Point]:
  def empty: Point = Point(0, 0)
  def combine(a: Point, b: Point): Point = Point(a.x + b.x, a.y + b.y)

val points = List(Point(1,2), Point(3,4), Point(5,6))
println(points.combineAll)  // Point(9, 12)

// Semigroup
"Hello" |+| ", " |+| "World"  // uses String Semigroup
```

---

## Functor, Applicative, Monad

### Functor

```scala
import cats.*
import cats.syntax.functor.*

// Functor: can map over
trait Functor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]

// Cats has instances for Option, List, Either, Future, etc.
val optResult = 42.some.map(_ * 2)   // Some(84)
val lstResult = List(1, 2, 3).map(_ + 10)  // List(11, 12, 13)

// Compose functors
val nested: Option[List[Int]] = Some(List(1, 2, 3))
val mappedNested = nested.map(_.map(_ * 2))  // Some(List(2, 4, 6))

// Functor for custom type
case class Box[A](value: A)

given Functor[Box] with
  def map[A, B](box: Box[A])(f: A => B): Box[B] = Box(f(box.value))

Box(42).map(_ + 1)  // Box(43)
```

### Applicative

```scala
import cats.*
import cats.syntax.applicative.*
import cats.syntax.apply.*

// Applicative: pure + ap
// Pure: lift A into F[A]
val pure42: Option[Int] = 42.pure[Option]
val pureList: List[String] = "hello".pure[List]

// mapN: combine multiple F[A] with a function
def validateAge(age: Int): Option[Int] =
  if age >= 0 && age <= 150 then Some(age) else None

def validateName(name: String): Option[String] =
  if name.nonEmpty then Some(name) else None

val validPerson = (validateName("Alice"), validateAge(30)).mapN { (name, age) =>
  s"$name, $age years old"
}
// Some(Alice, 30 years old)

val invalidPerson = (validateName(""), validateAge(30)).mapN { (name, age) =>
  s"$name, $age years old"
}
// None

// Apply *> and <*
val result = List(1, 2, 3) *> List("a", "b")
// List(a, b, a, b, a, b) - keeps right, applies to all combinations

// Parallel applicative (runs independently)
import cats.syntax.parallel.*
val par = (Option(1), Option(2), Option(3)).parMapN(_ + _ + _)
// Some(6)
```

### Monad

```scala
import cats.*
import cats.syntax.flatMap.*
import cats.syntax.monad.*

// Monad: pure + flatMap
// Sequential operations with flatMap
val program = for
  x <- List(1, 2, 3)
  y <- List(10, 20)
yield x * y
// List(10, 20, 20, 40, 30, 60)

// Kleisli: composable A => F[B] functions
import cats.data.Kleisli

val parseAge: Kleisli[Option, String, Int] =
  Kleisli(s => s.toIntOption)

val validateAge: Kleisli[Option, Int, Int] =
  Kleisli(n => if n > 0 then Some(n) else None)

val processAge = parseAge andThen validateAge

processAge.run("30")  // Some(30)
processAge.run("-5")  // None
processAge.run("abc") // None

// ReaderT / Kleisli for dependency injection
type Config = Map[String, String]
type ConfigReader[A] = Kleisli[Option, Config, A]

def getConfig(key: String): ConfigReader[String] =
  Kleisli(cfg => cfg.get(key))

def getPort: ConfigReader[Int] =
  getConfig("port").andThen(s => Kleisli(_ => s.toIntOption))

val config = Map("port" -> "8080", "host" -> "localhost")
println(getPort.run(config))  // Some(8080)
```

---

## Validated

### Accumulating Errors

```scala
import cats.data.{Validated, ValidatedNel, NonEmptyList}
import cats.syntax.validated.*
import cats.syntax.apply.*

// Validated: Either-like but accumulates errors
// ValidatedNel[E, A] = Validated[NonEmptyList[E], A]
type ValidationResult[A] = ValidatedNel[String, A]

def validateName(name: String): ValidationResult[String] =
  if name.trim.nonEmpty then name.trim.validNel
  else "Name cannot be empty".invalidNel

def validateEmail(email: String): ValidationResult[String] =
  if email.contains("@") && email.contains(".")
  then email.validNel
  else s"Invalid email: $email".invalidNel

def validateAge(age: Int): ValidationResult[Int] =
  if age >= 0 && age <= 150 then age.validNel
  else s"Age must be between 0 and 150, got: $age".invalidNel

case class User(name: String, email: String, age: Int)

// mapN: combines validations and ACCUMULATES errors
def createUser(
  name: String,
  email: String,
  age: Int
): ValidationResult[User] =
  (
    validateName(name),
    validateEmail(email),
    validateAge(age)
  ).mapN(User.apply)

// All valid
println(createUser("Alice", "alice@example.com", 30))
// Valid(User(Alice, alice@example.com, 30))

// Multiple errors accumulated
println(createUser("", "bad-email", 200))
// Invalid(NonEmptyList(Name cannot be empty,
//                      Invalid email: bad-email,
//                      Age must be between 0 and 150, got: 200))

// With Either for fail-fast
import cats.syntax.either.*

def createUserFast(
  name: String,
  email: String,
  age: Int
): Either[String, User] =
  for
    n <- validateName(name).toEither.left.map(_.head)
    e <- validateEmail(email).toEither.left.map(_.head)
    a <- validateAge(age).toEither.left.map(_.head)
  yield User(n, e, a)
```

---

## NonEmptyList

### กับ List ที่ไม่ว่างเปล่า

```scala
import cats.data.NonEmptyList

// NonEmptyList: guaranteed non-empty at type level
val nel: NonEmptyList[Int] = NonEmptyList.of(1, 2, 3, 4, 5)

// Head is always safe
println(nel.head)   // 1
println(nel.tail)   // List(2, 3, 4, 5)
println(nel.last)   // 5

// From List: returns Option
val fromList: Option[NonEmptyList[Int]] = NonEmptyList.fromList(List(1, 2, 3))

// Operations
val doubled = nel.map(_ * 2)         // NonEmptyList(2, 4, 6, 8, 10)
val sorted  = nel.sorted             // NonEmptyList(1, 2, 3, 4, 5)
val reduced = nel.reduce(_ + _)      // 15 (safe, no empty case)

// Append
val extended = nel :+ 6              // NonEmptyList(1, 2, 3, 4, 5, 6)
val prepended = 0 +: nel             // NonEmptyList(0, 1, 2, 3, 4, 5)

// Convert to List
val lst: List[Int] = nel.toList

// Real-world usage: return at least one error
type Errors = NonEmptyList[String]

def validate(input: String): Either[Errors, String] =
  val errors = List(
    Option.when(input.isEmpty)("Input is empty"),
    Option.when(input.length > 100)("Input too long"),
    Option.when(!input.matches("[a-zA-Z]+"))("Only letters allowed")
  ).flatten

  NonEmptyList.fromList(errors).toLeft(input)
```

---

## State and Writer

### State Monad

```scala
import cats.data.State

// State[S, A]: computation that reads/modifies state S, produces A
type Stack[A] = State[List[Int], A]

val push: Int => Stack[Unit] = n =>
  State.modify(n :: _)

val pop: Stack[Option[Int]] =
  State { stack =>
    stack match
      case h :: t => (t, Some(h))
      case Nil    => (Nil, None)
  }

val peek: Stack[Option[Int]] =
  State.inspect(_.headOption)

// Stack operations
val program = for
  _  <- push(1)
  _  <- push(2)
  _  <- push(3)
  v1 <- pop
  v2 <- peek
  _  <- push(10)
yield (v1, v2)

val (finalStack, result) = program.run(List.empty).value
println(finalStack)  // List(10, 2, 1)
println(result)      // (Some(3), Some(2))
```

### Writer Monad

```scala
import cats.data.Writer
import cats.syntax.writer.*

// Writer[L, A]: computation that produces a value A and logs L
type Logged[A] = Writer[List[String], A]

def loggedSquare(n: Int): Logged[Int] =
  Writer(List(s"Squaring $n"), n * n)

def loggedAdd(a: Int, b: Int): Logged[Int] =
  Writer(List(s"Adding $a + $b = ${a + b}"), a + b)

val computation = for
  sq  <- loggedSquare(3)
  sum <- loggedAdd(sq, 5)
yield sum

val (logs, result) = computation.run
println(logs.mkString("\n"))
// Squaring 3
// Adding 9 + 5 = 14
println(result)  // 14

// Tell: just log something
def withAudit[A](action: String, value: A): Writer[List[String], A] =
  value.writer(List(s"[AUDIT] $action: $value"))

val audited = for
  _   <- ().writer(List("[START] Transaction"))
  v   <- withAudit("compute", 42)
  res <- withAudit("double", v * 2)
yield res

val (auditLog, finalVal) = audited.run
auditLog.foreach(println)
println(s"Final: $finalVal")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Cats type classes: Eq, Show, Monoid, Semigroup
- ✅ Functor, Applicative, Monad สำหรับ F[_]
- ✅ Kleisli: composable monadic functions
- ✅ Validated: error accumulation
- ✅ NonEmptyList: guaranteed non-empty list
- ✅ State Monad: stateful computation
- ✅ Writer Monad: logging

---

*[← Part 32: Advanced Type System](part-32-type-system-advanced.md) | [Part 34: fs2 Streams →](part-34-fs2.md)*
