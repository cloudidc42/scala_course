# Part 13: Functional Programming

## สารบัญ
1. [Immutability และ Pure Functions](#immutability-และ-pure-functions)
2. [Referential Transparency](#referential-transparency)
3. [Function Composition](#function-composition)
4. [Functor, Monad, Applicative](#functor-monad-applicative)
5. [Error Handling แบบ Functional](#error-handling-แบบ-functional)
6. [State Monad](#state-monad)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Immutability และ Pure Functions

### Pure Functions

```scala
// Pure function: output ขึ้นอยู่กับ input เท่านั้น ไม่มี side effects

// Pure ✓
def add(a: Int, b: Int): Int = a + b
def toUpperCase(s: String): String = s.toUpperCase
def factorial(n: Int): BigInt =
  if n <= 1 then 1 else n * factorial(n - 1)

// Impure ✗ - มี side effects
var counter = 0
def incrementCounter(): Int =
  counter += 1  // side effect: modifies external state
  counter

def readLine(): String = scala.io.StdIn.readLine()  // IO side effect
def getCurrentTime(): Long = System.currentTimeMillis()  // time dependency
```

### ประโยชน์ของ Pure Functions

```scala
// 1. Testable
def calculateTax(amount: Double, rate: Double): Double = amount * rate
assert(calculateTax(100, 0.07) == 7.0)  // ทดสอบง่าย

// 2. Memoizable
def memoize[A, B](f: A => B): A => B =
  val cache = collection.mutable.HashMap[A, B]()
  (a: A) =>
    cache.getOrElseUpdate(a, f(a))

val expensiveComputation: Int => Long = memoize { n =>
  Thread.sleep(100)  // simulate expensive work
  n.toLong * n.toLong
}

// 3. Parallelizable
val numbers = (1 to 1000000).toList
val result = numbers.par.map(n => n * n).sum  // parallel safe

// 4. Composable
val processName: String => String =
  (_.trim)
    .andThen(_.toLowerCase)
    .andThen(_.capitalize)
    .andThen(s => s.replace(" ", "_"))

println(processName("  HELLO WORLD  "))  // hello_world
```

---

## Referential Transparency

### RT หมายถึงอะไร

```scala
// Referential Transparency: expression สามารถแทนที่ด้วยค่าของมันได้โดยไม่เปลี่ยน behavior

// RT: String.concat เป็น pure
val s = "Hello"
val r1 = s + " " + "World"  // Hello World
val r2 = "Hello" + " " + "World"  // Hello World - เหมือนกัน ✓

// Not RT: มี side effects
var total = 0
def addToTotal(n: Int): Int =
  total += n
  total

val a = addToTotal(5)  // total = 5, returns 5
val b = addToTotal(5)  // total = 10, returns 10
// a != b แม้ arguments เหมือนกัน ✗

// ทำให้ RT โดยใช้ immutable state
case class Counter(value: Int):
  def increment: Counter = Counter(value + 1)
  def add(n: Int): Counter = Counter(value + n)

val c1 = Counter(0).add(5)  // Counter(5)
val c2 = Counter(0).add(5)  // Counter(5) - เหมือนกันเสมอ ✓
```

---

## Function Composition

### andThen และ compose

```scala
val addOne: Int => Int = _ + 1
val double: Int => Int = _ * 2
val square: Int => Int = n => n * n

// andThen: ทำงานซ้ายไปขวา
val addOneThenDouble = addOne andThen double
println(addOneThenDouble(3))  // (3+1)*2 = 8

// compose: ทำงานขวาไปซ้าย
val doubleAfterAddOne = double compose addOne
println(doubleAfterAddOne(3))  // (3+1)*2 = 8 (เหมือนกัน)

// Pipeline ยาวๆ
val process: Int => String =
  ((n: Int) => n * n)
    .andThen(_ + 1)
    .andThen(_ * 2)
    .andThen(n => s"Result: $n")

println(process(4))  // Result: 34  (4^2=16, 16+1=17, 17*2=34)
```

### Function Composition Utilities

```scala
// Generic compose
def compose[A, B, C](f: B => C, g: A => B): A => C =
  a => f(g(a))

def pipe[A, B, C](f: A => B, g: B => C): A => C =
  a => g(f(a))

// Lifting pure function to work in container
def lift[A, B](f: A => B): Option[A] => Option[B] = _.map(f)

val optDouble = lift((n: Int) => n * 2)
println(optDouble(Some(5)))  // Some(10)
println(optDouble(None))     // None

// Kleisli composition (composing A => F[B] functions)
type Kleisli[F[_], A, B] = A => F[B]

def kleisliCompose[F[_], A, B, C](
  f: A => Option[B],
  g: B => Option[C]
): A => Option[C] = a => f(a).flatMap(g)

val parseAge: String => Option[Int] = s =>
  s.toIntOption.filter(_ >= 0)

val ageCategory: Int => Option[String] = age =>
  if age < 18 then Some("minor")
  else if age < 65 then Some("adult")
  else if age >= 65 then Some("senior")
  else None

val getCategory = kleisliCompose(parseAge, ageCategory)

println(getCategory("25"))   // Some(adult)
println(getCategory("-5"))   // None
println(getCategory("abc"))  // None
```

---

## Functor, Monad, Applicative

### Functor

```scala
// Functor laws:
// 1. Identity: fa.map(id) == fa
// 2. Composition: fa.map(f).map(g) == fa.map(f andThen g)

trait MyFunctor[F[_]]:
  extension [A](fa: F[A])
    def fmap[B](f: A => B): F[B]

given MyFunctor[Option] with
  extension [A](fa: Option[A])
    def fmap[B](f: A => B): Option[B] = fa.map(f)

given MyFunctor[List] with
  extension [A](fa: List[A])
    def fmap[B](f: A => B): List[B] = fa.map(f)

// ทดสอบ functor laws
val opt = Some(5)
assert(opt.fmap(identity) == opt)            // identity
assert(opt.fmap(_ + 1).fmap(_ * 2) ==
       opt.fmap(n => (n + 1) * 2))            // composition
```

### Monad

```scala
// Monad laws:
// 1. Left identity: pure(a).flatMap(f) == f(a)
// 2. Right identity: m.flatMap(pure) == m
// 3. Associativity: m.flatMap(f).flatMap(g) == m.flatMap(a => f(a).flatMap(g))

// Either monad สำหรับ error handling
type Error = String
type Result[A] = Either[Error, A]

def parseInt(s: String): Result[Int] =
  s.toIntOption.toRight(s"Not a number: $s")

def validatePositive(n: Int): Result[Int] =
  if n > 0 then Right(n) else Left(s"Not positive: $n")

def sqrt(n: Int): Result[Double] =
  if n >= 0 then Right(math.sqrt(n)) else Left("Negative number")

// Monadic chain
def safeSqrt(input: String): Result[Double] =
  for
    n    <- parseInt(input)
    pos  <- validatePositive(n)
    root <- sqrt(pos)
  yield root

println(safeSqrt("25"))   // Right(5.0)
println(safeSqrt("-4"))   // Left(Not positive: -4)
println(safeSqrt("abc"))  // Left(Not a number: abc)
```

### Applicative

```scala
// Applicative: apply function inside F to value inside F
// f: F[A => B], a: F[A] => F[B]

// กรณีที่ต้องการ combine ผลลัพธ์หลายอย่าง
case class Address(street: String, city: String, zip: String)

def validateStreet(s: String): Either[String, String] =
  if s.nonEmpty then Right(s) else Left("Street required")

def validateCity(c: String): Either[String, String] =
  if c.nonEmpty then Right(c) else Left("City required")

def validateZip(z: String): Either[String, String] =
  if z.matches("\\d{5}") then Right(z) else Left("Invalid zip")

// Applicative style (independent validations)
def validateAddress(street: String, city: String, zip: String) =
  for
    s <- validateStreet(street)
    c <- validateCity(city)
    z <- validateZip(zip)
  yield Address(s, c, z)

println(validateAddress("123 Main St", "Boston", "02101"))
// Right(Address(123 Main St,Boston,02101))
println(validateAddress("", "", "invalid"))
// Left(Street required) - short circuits at first error
```

---

## Error Handling แบบ Functional

### Option

```scala
// Option: Some(value) หรือ None
// ใช้เมื่อ value อาจไม่มี

case class User(id: Int, name: String, email: Option[String])
case class UserProfile(userId: Int, bio: Option[String])

val users = Map(
  1 -> User(1, "Alice", Some("alice@example.com")),
  2 -> User(2, "Bob", None)
)

val profiles = Map(
  1 -> UserProfile(1, Some("Scala developer")),
  2 -> UserProfile(2, None)
)

def getUserEmail(userId: Int): Option[String] =
  for
    user  <- users.get(userId)
    email <- user.email
  yield email

def getUserBio(userId: Int): Option[String] =
  for
    profile <- profiles.get(userId)
    bio     <- profile.bio
  yield bio

// Operations
println(getUserEmail(1))  // Some(alice@example.com)
println(getUserEmail(2))  // None
println(getUserEmail(3))  // None

// fold/getOrElse
val email = getUserEmail(1).getOrElse("no email")
val bio = getUserBio(2).fold("No bio")(b => s"Bio: $b")
```

### Either

```scala
// Either[E, A]: Left(error) หรือ Right(value)
// Right-biased: map/flatMap ทำงานกับ Right

sealed trait AppError
case class ValidationError(field: String, message: String) extends AppError
case class DatabaseError(message: String) extends AppError
case class NotFoundError(id: Int) extends AppError

def validateName(name: String): Either[AppError, String] =
  if name.trim.isEmpty then Left(ValidationError("name", "Name cannot be empty"))
  else if name.length < 2 then Left(ValidationError("name", "Name too short"))
  else Right(name.trim)

def validateAge(age: Int): Either[AppError, Int] =
  if age < 0 then Left(ValidationError("age", "Age cannot be negative"))
  else if age > 150 then Left(ValidationError("age", "Age unrealistic"))
  else Right(age)

case class NewUser(name: String, age: Int)

def createUser(name: String, age: Int): Either[AppError, NewUser] =
  for
    validName <- validateName(name)
    validAge  <- validateAge(age)
  yield NewUser(validName, validAge)

println(createUser("Alice", 30))   // Right(NewUser(Alice,30))
println(createUser("", 25))        // Left(ValidationError(name,Name cannot be empty))
println(createUser("Alice", -1))   // Left(ValidationError(age,Age cannot be negative))
```

### Try

```scala
import scala.util.{Try, Success, Failure}

def readFile(path: String): Try[String] =
  Try(scala.io.Source.fromFile(path).mkString)

def parseJson(json: String): Try[Map[String, Any]] =
  Try {
    // Simplified - ในความจริงใช้ library
    if json.startsWith("{") then Map("data" -> json)
    else throw new RuntimeException("Invalid JSON")
  }

def processFile(path: String): Try[String] =
  for
    content <- readFile(path)
    json    <- parseJson(content)
  yield s"Processed: ${json.size} fields"

processFile("config.json") match
  case Success(result) => println(result)
  case Failure(ex)     => println(s"Error: ${ex.getMessage}")

// recover
val result = Try(1 / 0).recover {
  case _: ArithmeticException => 0
}
println(result)  // Success(0)
```

---

## State Monad

### Pure Stateful Computations

```scala
// State monad: S => (A, S)
// ทำให้ stateful computation เป็น pure

case class State[S, A](run: S => (A, S)):
  def map[B](f: A => B): State[S, B] =
    State { s =>
      val (a, ns) = run(s)
      (f(a), ns)
    }

  def flatMap[B](f: A => State[S, B]): State[S, B] =
    State { s =>
      val (a, ns) = run(s)
      f(a).run(ns)
    }

object State:
  def pure[S, A](a: A): State[S, A] = State(s => (a, s))
  def get[S]: State[S, S] = State(s => (s, s))
  def set[S](s: S): State[S, Unit] = State(_ => ((), s))
  def modify[S](f: S => S): State[S, Unit] = State(s => ((), f(s)))

// ตัวอย่าง: Random Number Generator
case class RNG(seed: Long):
  def nextInt: (Int, RNG) =
    val newSeed = (seed * 6364136223846793005L + 1442695040888963407L)
    val n = ((newSeed >>> 16) & Int.MaxValue).toInt
    (n, RNG(newSeed))

type RandState[A] = State[RNG, A]

def nextInt: RandState[Int] = State(rng => rng.nextInt)

def nextDouble: RandState[Double] =
  nextInt.map(n => n.toDouble / Int.MaxValue)

def nextBool: RandState[Boolean] =
  nextInt.map(_ % 2 == 0)

def roll: RandState[Int] =
  nextInt.map(n => (n % 6).abs + 1)

// Combine multiple random values
val threeRolls: RandState[List[Int]] =
  for
    r1 <- roll
    r2 <- roll
    r3 <- roll
  yield List(r1, r2, r3)

val (rolls, finalRng) = threeRolls.run(RNG(42))
println(s"Rolls: $rolls")  // Rolls: List(some, random, numbers)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Function Composition Pipeline

```scala
// สร้าง data processing pipeline ด้วย function composition

case class RawData(timestamp: String, value: String, unit: String)
case class Measurement(timestamp: Long, value: Double, unit: String)
case class NormalizedMeasurement(timestamp: Long, valueInSI: Double)

// TODO: implement each step as a pure function
def parseTimestamp(s: String): Either[String, Long] = ???
def parseValue(s: String): Either[String, Double] = ???
def toSI(value: Double, unit: String): Either[String, Double] = ???

def processMeasurement(raw: RawData): Either[String, NormalizedMeasurement] = ???

// Test data
val testData = List(
  RawData("1705312200", "100", "km"),
  RawData("1705312201", "1.5", "miles"),
  RawData("invalid", "100", "km"),
  RawData("1705312202", "abc", "km")
)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Pure functions และ immutability
- ✅ Referential Transparency
- ✅ Function Composition (andThen, compose)
- ✅ Functor, Monad, Applicative concepts
- ✅ Error handling: Option, Either, Try
- ✅ State Monad

---

*[← Part 12: Generics](part-12-generics.md) | [Part 14: Higher-Order Functions →](part-14-higher-order-functions.md)*
