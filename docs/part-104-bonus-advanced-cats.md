# ส่วนที่ 104 (BONUS): Advanced Cats ใน Scala

> **BONUS CONTENT** - เนื้อหาขั้นสูงเกี่ยวกับ Cats library สำหรับ Type Classes ขั้นสูงที่ไม่ค่อยเห็นในบทเรียนทั่วไป

---

## สารบัญ

1. [บทนำ: Cats Type Class Hierarchy](#บทนำ)
2. [Bifunctor และ Bitraverse](#bifunctor-และ-bitraverse)
3. [ContravariantMonoidal](#contravariantmonoidal)
4. [Parallel Type Class](#parallel-type-class)
5. [CommutativeApplicative](#commutativeapplicative)
6. [Ior: Both, Left, Right](#ior)
7. [Chain: Efficient Append](#chain)
8. [NonEmptyChain](#nonemptychain)
9. [Validated vs Either](#validated-vs-either)
10. [Complete Validation Pipeline](#validation-pipeline)
11. [FunctionK และ Natural Transformations](#functionk)
12. [สรุป](#สรุป)

---

## บทนำ

Cats (Typelevel Cats) มี type classes มากกว่าที่คนส่วนใหญ่รู้จัก บทนี้ครอบคลุม advanced type classes ที่มีประโยชน์มากในงานจริง

```scala
// dependencies
// libraryDependencies += "org.typelevel" %% "cats-core" % "2.10.0"
// libraryDependencies += "org.typelevel" %% "cats-effect" % "3.5.0"

import cats.*
import cats.data.*
import cats.implicits.*
import cats.syntax.all.*
```

### Cats Type Class Hierarchy (Simplified)

```
Functor
├── Apply (Functor + ap)
│   └── Applicative (Apply + pure)
│       └── Monad (Applicative + flatMap)
│           ├── MonadError
│           └── MonadThrow
├── ContravariantFunctor (contramap)
│   └── ContravariantMonoidal
├── Bifunctor (bimap)
│   └── Bitraverse
└── Traverse (Functor + traverse)
```

---

## Bifunctor และ Bitraverse

Bifunctor คือ type constructor ที่ map ทั้งสอง type parameters ได้

### Bifunctor

```scala
import cats.Bifunctor

// Bifunctor[F[_, _]]: map ทั้ง left และ right
// bimap: (A => C, B => D) => F[A, B] => F[C, D]

// Either เป็น Bifunctor
val result: Either[String, Int] = Right(42)

// bimap: transform both sides
val transformed = result.bimap(
  err => s"Error: $err",       // transform Left
  n   => n * 2                  // transform Right
)
println(transformed)  // Right(84)

// leftMap: transform only Left
val leftOnly: Either[Int, String] = Left("error")
val leftMapped = leftOnly.leftMap(_.toUpperCase)
println(leftMapped)  // Left(ERROR)

// map: transform only Right (from Functor)
val rightMapped = result.map(_ + 1)
println(rightMapped)  // Right(43)
```

### Custom Bifunctor

```scala
// สร้าง Bifunctor สำหรับ custom type
case class Validation[E, A](errors: List[E], value: A)

given Bifunctor[Validation] with
  def bimap[A, B, C, D](fab: Validation[A, B])(f: A => C)(g: B => D): Validation[C, D] =
    Validation(fab.errors.map(f), g(fab.value))

val v = Validation(List("err1", "err2"), 42)
val mapped = v.bimap(_.toUpperCase, _ * 2)
println(mapped)  // Validation(List(ERR1, ERR2),84)
```

### Bitraverse

```scala
import cats.Bitraverse

// Bitraverse: traverse ทั้งสอง sides
// bitraverse: (A => F[C], B => F[D]) => G[A, B] => F[G[C, D]]

// ตัวอย่าง: validate both sides of Either
def validateLeft(s: String): Either[String, Int] =
  s.toIntOption.toRight(s"'$s' is not a number")

def validateRight(n: Int): Either[String, String] =
  if n > 0 then Right(s"positive: $n") else Left("must be positive")

// Tuple2 เป็น Bitraversable
val pair: (String, Int) = ("42", 5)
val validated: Either[String, (Int, String)] = pair.bitraverse(validateLeft, validateRight)
println(validated)  // Right((42,positive: 5))

val badPair: (String, Int) = ("abc", -1)
val invalid = badPair.bitraverse(validateLeft, validateRight)
println(invalid)  // Left('abc' is not a number)
```

### ใช้ Bitraverse สำหรับ Error Handling

```scala
// แปลง (input, output) คู่โดย validate ทั้งสอง
case class RawInput(source: String, destination: String)
case class ValidatedRoute(from: Int, to: Int)

def validateNodeId(s: String): Either[String, Int] =
  s.toIntOption.toRight(s"Invalid node: '$s'")

def validateRoute(raw: RawInput): Either[String, ValidatedRoute] =
  (raw.source, raw.destination)
    .bitraverse(validateNodeId, validateNodeId)
    .map { case (from, to) => ValidatedRoute(from, to) }

println(validateRoute(RawInput("1", "5")))   // Right(ValidatedRoute(1,5))
println(validateRoute(RawInput("bad", "5"))) // Left(Invalid node: 'bad')
```

---

## ContravariantMonoidal

ContravariantMonoidal คือ Contravariant Functor ที่มี product operation

### นิยาม

```scala
// Contravariant[F]: contramap (map ย้อนทาง)
// ContravariantSemigroupal: product ของ contravariant functors

// ตัวอย่างที่ดีที่สุดคือ Decoder/Encoder
```

### Encoder เป็น ContravariantMonoidal

```scala
import cats.ContravariantSemigroupal

// Simple CSV Encoder
case class CSVEncoder[A](encode: A => List[String])

given ContravariantSemigroupal[CSVEncoder] with
  def contramap[A, B](fa: CSVEncoder[A])(f: B => A): CSVEncoder[B] =
    CSVEncoder(b => fa.encode(f(b)))
  
  def product[A, B](fa: CSVEncoder[A], fb: CSVEncoder[B]): CSVEncoder[(A, B)] =
    CSVEncoder((a, b) => fa.encode(a) ++ fb.encode(b))

// Primitive encoders
val intEncoder: CSVEncoder[Int] = CSVEncoder(n => List(n.toString))
val stringEncoder: CSVEncoder[String] = CSVEncoder(s => List(s))
val doubleEncoder: CSVEncoder[Double] = CSVEncoder(d => List(f"$d%.2f"))

// Derive encoder สำหรับ case class
case class Product(name: String, price: Double, qty: Int)

val productEncoder: CSVEncoder[Product] =
  (stringEncoder, doubleEncoder, intEncoder)
    .contramapN[Product](p => (p.name, p.price, p.qty))

val products = List(
  Product("Apple", 30.5, 100),
  Product("Banana", 15.0, 200)
)

products.foreach { p =>
  println(productEncoder.encode(p).mkString(","))
}
// Apple,30.50,100
// Banana,15.00,200
```

### Ordering เป็น Contravariant

```scala
// Ordering[A] เป็น Contravariant - เราสามารถ contramap ได้
import cats.instances.order.*

case class Person(name: String, age: Int)

val byAge: Order[Person] = Order[Int].contramap(_.age)
val byName: Order[Person] = Order[String].contramap(_.name)
val byAgeThenName: Order[Person] = byAge |+| byName  // Monoid on Order!

val people = List(
  Person("Charlie", 30),
  Person("Alice", 25),
  Person("Bob", 30),
  Person("Dave", 25)
)

println(people.sorted(byAge.toOrdering))
// List(Person(Alice,25), Person(Dave,25), Person(Charlie,30), Person(Bob,30))

println(people.sorted(byAgeThenName.toOrdering))
// List(Person(Alice,25), Person(Dave,25), Person(Bob,30), Person(Charlie,30))
```

---

## Parallel Type Class

Parallel คือ type class ที่แสดงความสัมพันธ์ระหว่าง Monad และ Applicative

### นิยาม

```scala
// Parallel[M, F]:
// M = Monad (sequential, fails fast)
// F = Applicative (parallel, accumulates)
// ทั้งคู่ต้องมี natural transformation ระหว่างกัน
```

### Either vs Validated

```scala
// Either: sequential, fails fast
def validateAge(n: Int): Either[String, Int] =
  if n > 0 then Right(n) else Left("Age must be positive")

def validateName(s: String): Either[String, String] =
  if s.nonEmpty then Right(s) else Left("Name cannot be empty")

// Sequential: หยุดที่ error แรก
val eitherResult = for
  age  <- validateAge(-1)
  name <- validateName("")   // ไม่ถูก evaluate
yield (age, name)
println(eitherResult)  // Left(Age must be positive) - เห็นแค่ error แรก

// Validated: parallel, accumulates all errors
import cats.data.Validated.*
import cats.syntax.validated.*

def validateAgeV(n: Int): Validated[List[String], Int] =
  if n > 0 then n.valid else List("Age must be positive").invalid

def validateNameV(s: String): Validated[List[String], String] =
  if s.nonEmpty then s.valid else List("Name cannot be empty").invalid

// Parallel: ดู errors ทั้งหมด
val validResult = (validateAgeV(-1), validateNameV("")).mapN((age, name) => (age, name))
println(validResult)
// Invalid(List(Age must be positive, Name cannot be empty))
```

### parMapN: Parallel Applicative

```scala
// parMapN ใช้ Parallel type class
// สำหรับ Either: ใช้ EitherT + Validated ภายใน
import cats.syntax.parallel.*

val e1: Either[List[String], Int] = Left(List("Error 1"))
val e2: Either[List[String], String] = Left(List("Error 2"))
val e3: Either[List[String], Boolean] = Right(true)

// parMapN สะสม errors จาก Either!
val result = (e1, e2, e3).parMapN((n, s, b) => s"$n, $s, $b")
println(result)  // Left(List(Error 1, Error 2))

// ถ้าทุกอย่าง valid
val ok1: Either[List[String], Int] = Right(42)
val ok2: Either[List[String], String] = Right("hello")
val ok3: Either[List[String], Boolean] = Right(true)

val okResult = (ok1, ok2, ok3).parMapN((n, s, b) => s"$n, $s, $b")
println(okResult)  // Right(42, hello, true)
```

### parTraverse: Parallel Traversal

```scala
import cats.effect.IO
import cats.syntax.parallel.*

// Sequential traverse (ทำทีละอัน)
def fetchUser(id: Int): IO[String] = IO(s"User$id")

val users1: IO[List[String]] = List(1, 2, 3).traverse(fetchUser)
// ทำทีละ id: 1 → 2 → 3

// Parallel traverse (ทำพร้อมกัน!)
val users2: IO[List[String]] = List(1, 2, 3).parTraverse(fetchUser)
// ทำพร้อมกัน: 1 || 2 || 3

// parSequence: ทำ List[IO[A]] => IO[List[A]] แบบ parallel
val ios: List[IO[String]] = List(1, 2, 3).map(fetchUser)
val parallel: IO[List[String]] = ios.parSequence
```

---

## CommutativeApplicative

CommutativeApplicative คือ Applicative ที่ argument order ไม่สำคัญ

```scala
import cats.CommutativeApplicative

// สำหรับ CommutativeApplicative:
// product(fa, fb) == product(fb, fa).map { case (b, a) => (a, b) }

// Option เป็น CommutativeApplicative
// Either[E, *] เป็น CommutativeApplicative (ถ้า E เป็น Commutative)

// ใช้งานกับ parallel validation
import cats.data.Validated
import cats.instances.list.*

def validateAll[A, E](validations: List[A => Validated[E, A]])(a: A): Validated[List[E], A] =
  validations.traverse_(_(a).leftMap(List(_))).as(a)
  // traverse_ ต้องการ CommutativeApplicative
```

---

## Ior

`Ior[A, B]` เป็น data type ที่แสดง "inclusive or" - สามารถมี Left, Right, หรือ Both

### นิยาม

```
Ior[A, B] = Left(a) | Right(b) | Both(a, b)
```

### การใช้งาน

```scala
import cats.data.Ior
import cats.data.Ior.*

val left: Ior[String, Int]  = Ior.left("error")
val right: Ior[String, Int] = Ior.right(42)
val both: Ior[String, Int]  = Ior.both("warning", 42)

// Fold/pattern match
def describe(ior: Ior[String, Int]): String = ior match
  case Left(e)    => s"Failed: $e"
  case Right(n)   => s"Success: $n"
  case Both(w, n) => s"Warning '$w' but got: $n"

println(describe(left))   // Failed: error
println(describe(right))  // Success: 42
println(describe(both))   // Warning 'warning' but got: 42
```

### IorNec: Ior กับ NonEmptyChain

```scala
import cats.data.IorNec
import cats.data.NonEmptyChain

// IorNec = Ior[NonEmptyChain[E], A]
// ใช้สำหรับ "partial success with warnings"
type Result[A] = IorNec[String, A]

def parseAge(s: String): Result[Int] =
  s.toIntOption match
    case Some(n) if n >= 0 && n <= 150 => Ior.right(n)
    case Some(n) => Ior.both(
      NonEmptyChain.one(s"Unusual age: $n"),
      n  // ยังคืนค่า แต่มี warning
    )
    case None => Ior.left(NonEmptyChain.one(s"'$s' is not a number"))

println(parseAge("25"))    // Right(25)
println(parseAge("200"))   // Both(Chain(Unusual age: 200), 200)
println(parseAge("abc"))   // Left(Chain('abc' is not a number))
```

### Ior Monad: สะสม Warnings

```scala
// Ior[W, A] เป็น Monad เมื่อ W เป็น Semigroup
// ทำให้สะสม warnings ระหว่างการ compute

type Warned[A] = Ior[List[String], A]

def warn(msg: String): Warned[Unit] = Ior.both(List(msg), ())
def pure[A](a: A): Warned[A] = Ior.right(a)

def safeDivide(a: Int, b: Int): Warned[Double] =
  if b == 0 then
    Ior.both(List("Division by zero, using Infinity"), Double.PositiveInfinity)
  else
    Ior.right(a.toDouble / b)

// Chain warnings
val computation: Warned[Double] = for
  _    <- warn("Starting computation")
  x    <- safeDivide(10, 2)    // Right(5.0)
  y    <- safeDivide(20, 0)    // Both(warning, Infinity)
  _    <- warn("Completed")
yield x + y

println(computation)
// Both(List(Starting computation, Division by zero..., Completed), Infinity)
```

### ตัวอย่างจริง: Data Import กับ Warnings

```scala
case class RawRecord(name: String, age: String, score: String)
case class CleanRecord(name: String, age: Int, score: Double)

type ImportResult[A] = IorNec[String, A]

def cleanRecord(raw: RawRecord): ImportResult[CleanRecord] =
  // Parse age
  val ageResult: ImportResult[Int] = raw.age.toIntOption match
    case Some(n) if n >= 0 => Ior.right(n)
    case Some(n) =>
      Ior.both(NonEmptyChain.one(s"Negative age ${n} for ${raw.name}, using 0"), 0)
    case None =>
      Ior.left(NonEmptyChain.one(s"Invalid age '${raw.age}' for ${raw.name}"))
  
  // Parse score
  val scoreResult: ImportResult[Double] = raw.score.toDoubleOption match
    case Some(s) if s >= 0 && s <= 100 => Ior.right(s)
    case Some(s) =>
      Ior.both(NonEmptyChain.one(s"Score $s out of range for ${raw.name}, clamping"), s.clamp(0, 100))
    case None =>
      Ior.left(NonEmptyChain.one(s"Invalid score '${raw.score}' for ${raw.name}"))
  
  // Combine
  (ageResult, scoreResult).mapN { (age, score) =>
    CleanRecord(raw.name, age, score)
  }

val records = List(
  RawRecord("Alice", "30", "85.5"),
  RawRecord("Bob", "-5", "90"),
  RawRecord("Carol", "abc", "150"),
  RawRecord("Dave", "25", "xyz")
)

val results = records.map(cleanRecord)
results.foreach { result =>
  result match
    case Ior.Right(r)    => println(s"OK: $r")
    case Ior.Both(w, r)  => println(s"WARN (${w.toList.mkString(", ")}): $r")
    case Ior.Left(e)     => println(s"ERROR: ${e.toList.mkString(", ")}")
}
```

---

## Chain

`Chain[A]` คือ data structure สำหรับ efficient append - O(1) prepend และ append

### ปัญหาของ List

```scala
// List: O(1) prepend, O(n) append
val list = List(1, 2, 3)
val appended = list :+ 4  // O(n) - ต้อง traverse ทั้ง list

// การ concat หลาย lists
val lists = List.fill(1000)(List(1, 2, 3))
val concat = lists.flatten  // O(n²) ในบางกรณี!
```

### Chain: O(1) สำหรับทุกอย่าง

```scala
import cats.data.Chain

// สร้าง Chain
val c1 = Chain(1, 2, 3)
val c2 = Chain(4, 5, 6)

// O(1) append
val appended = c1 ++ c2

// O(1) prepend
val prepended = 0 +: c1

// แปลงเป็น List
val list = appended.toList  // List(1, 2, 3, 4, 5, 6)
```

### Chain ใน Practice

```scala
import cats.data.Chain

// สะสม logs ระหว่าง computation
type Log = Chain[String]
type Logged[A] = (Log, A)

def log(msg: String): Log = Chain.one(msg)

def process(input: Int): Logged[Int] =
  val logs = log(s"Processing $input")
  val result = input * 2
  (logs, result)

// Combine logs efficiently
def processAll(inputs: List[Int]): Logged[List[Int]] =
  inputs.foldLeft((Chain.empty[String], List.empty[Int])) {
    case ((logs, results), input) =>
      val (newLogs, result) = process(input)
      (logs ++ newLogs, results :+ result)  // Chain ++ เป็น O(1)
  }

val (logs, results) = processAll(List(1, 2, 3, 4, 5))
println(logs.toList)    // List(Processing 1, Processing 2, ...)
println(results)        // List(2, 4, 6, 8, 10)
```

### Writer Monad กับ Chain

```scala
import cats.data.Writer

type Logged2[A] = Writer[Chain[String], A]

def processW(input: Int): Logged2[Int] =
  Writer(Chain.one(s"Processing $input"), input * 2)

def processAllW(inputs: List[Int]): Logged2[List[Int]] =
  inputs.traverse(processW)

val (logs2, results2) = processAllW(List(1, 2, 3)).run
println(logs2.toList)   // List(Processing 1, Processing 2, Processing 3)
println(results2)       // List(2, 4, 6)
```

---

## NonEmptyChain

`NonEmptyChain[A]` คือ Chain ที่รับประกันว่ามีอย่างน้อย 1 element

### การสร้างและใช้งาน

```scala
import cats.data.NonEmptyChain

// สร้าง NonEmptyChain
val nec1 = NonEmptyChain.one(1)
val nec2 = NonEmptyChain(1, 2, 3)
val nec3 = NonEmptyChain.fromSeq(List(1, 2, 3))  // Option[NonEmptyChain]

// Append - ยังคงเป็น NonEmptyChain
val combined = nec1 ++ NonEmptyChain(2, 3, 4)

// head ไม่ return Option!
println(nec2.head)  // 1 (ไม่ใช่ Option[Int])

// แปลงเป็น List
println(nec2.toList)  // List(1, 2, 3)
```

### Validated กับ NonEmptyChain

```scala
import cats.data.{Validated, NonEmptyChain}
import cats.data.ValidatedNec

// ValidatedNec = Validated[NonEmptyChain[E], A]
type ValidationResult[A] = ValidatedNec[String, A]

def validateAge(n: Int): ValidationResult[Int] =
  if n >= 0 && n <= 150 then Validated.valid(n)
  else Validated.invalidNec(s"Age $n is out of range")

def validateName(s: String): ValidationResult[String] =
  if s.length >= 2 && s.length <= 50 then Validated.valid(s)
  else Validated.invalidNec(s"Name '$s' must be 2-50 chars")

def validateEmail(s: String): ValidationResult[String] =
  if s.contains("@") && s.contains(".") then Validated.valid(s)
  else Validated.invalidNec(s"'$s' is not a valid email")

case class Registration(name: String, age: Int, email: String)

def validateRegistration(name: String, age: Int, email: String): ValidationResult[Registration] =
  (validateName(name), validateAge(age), validateEmail(email)).mapN(Registration.apply)

// Test
println(validateRegistration("Alice", 30, "alice@example.com"))
// Valid(Registration(Alice,30,alice@example.com))

println(validateRegistration("A", -5, "not-an-email"))
// Invalid(Chain(Name 'A' must be 2-50 chars, Age -5 is out of range, 'not-an-email' is not a valid email))
```

---

## Validated vs Either

### เมื่อไหร่ใช้อะไร

```scala
// Either: ใช้เมื่อ
// - ต้องการ fail fast (error แรกหยุดทันที)
// - computations ต้อง depend กัน
// - ใช้ใน for comprehension

val either: Either[String, Int] = for
  a <- Right(1)
  b <- Left("error")  // หยุดที่นี่
  c <- Right(3)       // ไม่ run
yield a + b + c

// Validated: ใช้เมื่อ
// - ต้องการ accumulate errors
// - validations เป็น independent
// - user input validation

// แปลงระหว่างกัน
val v: Validated[List[String], Int] = Validated.valid(42)
val e: Either[List[String], Int] = v.toEither
val back: Validated[List[String], Int] = e.toValidated

// ใช้ Parallel สำหรับ Either ที่สะสม errors
import cats.syntax.parallel.*

val r1: Either[List[String], Int] = Left(List("e1"))
val r2: Either[List[String], String] = Left(List("e2"))
val parallel = (r1, r2).parMapN((n, s) => s"$n $s")
println(parallel)  // Left(List(e1, e2))
```

---

## Validation Pipeline

ตัวอย่างสมบูรณ์: ระบบ User Registration Validation

### Domain Model

```scala
case class RawUserInput(
  username: String,
  email: String,
  password: String,
  age: String,
  phone: Option[String],
  referralCode: Option[String]
)

case class ValidatedUser(
  username: Username,
  email: Email,
  password: HashedPassword,
  age: Age,
  phone: Option[PhoneNumber],
  referralCode: Option[ReferralCode]
)

opaque type Username = String
opaque type Email = String
opaque type HashedPassword = String
opaque type Age = Int
opaque type PhoneNumber = String
opaque type ReferralCode = String

object Username:
  def apply(s: String): Username = s
  
object Email:
  def apply(s: String): Email = s

object HashedPassword:
  def apply(s: String): HashedPassword = s"hashed:$s"

object Age:
  def apply(n: Int): Age = n

object PhoneNumber:
  def apply(s: String): PhoneNumber = s

object ReferralCode:
  def apply(s: String): ReferralCode = s
```

### Validation Rules

```scala
import cats.data.ValidatedNec
import cats.syntax.validated.*

type V[A] = ValidatedNec[ValidationError, A]

sealed trait ValidationError:
  def message: String
  
case class TooShort(field: String, min: Int, actual: Int) extends ValidationError:
  def message = s"$field must be at least $min chars (got $actual)"
  
case class TooLong(field: String, max: Int, actual: Int) extends ValidationError:
  def message = s"$field must be at most $max chars (got $actual)"
  
case class InvalidFormat(field: String, expected: String) extends ValidationError:
  def message = s"$field has invalid format, expected: $expected"
  
case class OutOfRange(field: String, min: Int, max: Int, actual: Int) extends ValidationError:
  def message = s"$field ($actual) must be between $min and $max"

// Username validation
def validateUsername(s: String): V[Username] =
  val errors = List(
    Option.when(s.length < 3)(TooShort("username", 3, s.length)),
    Option.when(s.length > 20)(TooLong("username", 20, s.length)),
    Option.when(!s.matches("[a-zA-Z0-9_]+")):
      InvalidFormat("username", "alphanumeric and underscore only")
  ).flatten
  
  if errors.isEmpty then Username(s).valid
  else errors.foldLeft(Validated.validNec[ValidationError, Username](Username(s)))((_, e) =>
    Validated.invalidNec(e)
  )

// Email validation  
def validateEmail(s: String): V[Email] =
  val emailRegex = """^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$""".r
  if emailRegex.matches(s) then Email(s).valid
  else InvalidFormat("email", "valid email address").invalidNec

// Password validation
def validatePassword(s: String): V[HashedPassword] =
  val errors = List(
    Option.when(s.length < 8)(TooShort("password", 8, s.length)),
    Option.when(!s.exists(_.isUpper)):
      InvalidFormat("password", "at least one uppercase letter"),
    Option.when(!s.exists(_.isDigit)):
      InvalidFormat("password", "at least one digit"),
    Option.when(!s.exists(c => "!@#$%^&*".contains(c))):
      InvalidFormat("password", "at least one special character (!@#$%^&*)")
  ).flatten
  
  NonEmptyChain.fromSeq(errors) match
    case Some(nec) => Validated.invalid(nec)
    case None      => HashedPassword(s).valid

// Age validation
def validateAge(s: String): V[Age] =
  s.toIntOption match
    case None    => InvalidFormat("age", "a number").invalidNec
    case Some(n) =>
      if n < 13 || n > 120 then OutOfRange("age", 13, 120, n).invalidNec
      else Age(n).valid

// Phone validation (optional)
def validatePhone(s: String): V[PhoneNumber] =
  val phoneRegex = """^(\+66|0)[0-9]{8,9}$""".r
  if phoneRegex.matches(s) then PhoneNumber(s).valid
  else InvalidFormat("phone", "+66XXXXXXXXX or 0XXXXXXXXX").invalidNec

// Referral code (optional with warning via Ior)
def validateReferralCode(s: String): IorNec[String, ReferralCode] =
  if s.length == 8 && s.forall(_.isLetterOrDigit) then Ior.right(ReferralCode(s))
  else Ior.both(
    NonEmptyChain.one(s"Referral code '$s' format is unusual, proceeding anyway"),
    ReferralCode(s)
  )
```

### Combining Validations

```scala
def validateUserInput(input: RawUserInput): V[ValidatedUser] =
  (
    validateUsername(input.username),
    validateEmail(input.email),
    validatePassword(input.password),
    validateAge(input.age),
    input.phone.traverse(validatePhone),
    input.referralCode.traverse(code => validateReferralCode(code).toValidated.leftMap(
      nec => nec.map(msg => InvalidFormat("referral_code", msg))
    ))
  ).mapN(ValidatedUser.apply)
```

### Pipeline ด้วย Effect

```scala
import cats.effect.IO

// Business rules ที่ต้องเช็ค DB
def checkUsernameAvailable(username: Username): IO[V[Username]] =
  IO.pure {
    val taken = Set("admin", "root", "system")
    if taken.contains(username.asInstanceOf[String]) then
      InvalidFormat("username", s"'$username' is already taken").invalidNec
    else username.valid
  }

def checkEmailAvailable(email: Email): IO[V[Email]] =
  IO.pure(email.valid)  // simplified

// Full registration validation pipeline
def validateRegistration(input: RawUserInput): IO[V[ValidatedUser]] =
  for
    // Step 1: Format validation (pure)
    formatResult <- IO.pure(validateUserInput(input))
    
    // Step 2: Business rules (needs IO)
    finalResult <- formatResult.traverse { user =>
      (
        checkUsernameAvailable(user.username),
        checkEmailAvailable(user.email)
      ).parMapN((u, e) => (u, e).tupled)
        .map(_.as(user))
    }.map(_.flatten)
  yield finalResult

// Test the pipeline
val goodInput = RawUserInput(
  "alice_123", "alice@example.com", "SecureP@ss1",
  "25", Some("+66812345678"), Some("ABCD1234")
)

val badInput = RawUserInput(
  "a!", "not-email", "weak", "5",
  Some("bad-phone"), None
)

// Display results
def showResult(result: V[ValidatedUser]): String = result match
  case Validated.Valid(user) =>
    s"Registration successful: ${user.username}"
  case Validated.Invalid(errors) =>
    s"Registration failed:\n${errors.toList.map(e => s"  - ${e.message}").mkString("\n")}"

import cats.effect.unsafe.implicits.global

val program = for
  r1 <- validateRegistration(goodInput)
  r2 <- validateRegistration(badInput)
yield (r1, r2)

val (r1, r2) = program.unsafeRunSync()
println(showResult(r1))
println("---")
println(showResult(r2))
```

---

## FunctionK

`FunctionK[F, G]` (หรือ `F ~> G`) คือ natural transformation ระหว่าง type constructors

```scala
import cats.~>
import cats.Id

// แปลง Option เป็น List
val optionToList: Option ~> List = new (Option ~> List):
  def apply[A](fa: Option[A]): List[A] = fa.toList

// แปลง List เป็น Option (head)
val listToOption: List ~> Option = new (List ~> Option):
  def apply[A](fa: List[A]): Option[A] = fa.headOption

// Compose natural transformations
val identity: Option ~> Option = listToOption.compose(optionToList)

println(optionToList(Some(42)))       // List(42)
println(optionToList(None))           // List()
println(listToOption(List(1, 2, 3)))  // Some(1)
println(listToOption(Nil))            // None
```

### FunctionK ใน Interpreter Pattern

```scala
// ใช้ FunctionK เป็น interpreter สำหรับ algebraic effects
sealed trait Console[A]
case class ReadLine() extends Console[String]
case class WriteLine(msg: String) extends Console[Unit]

import cats.free.Free

type ConsoleIO[A] = Free[Console, A]

def readLine: ConsoleIO[String] = Free.liftF(ReadLine())
def writeLine(msg: String): ConsoleIO[Unit] = Free.liftF(WriteLine(msg))

// Interpreter: Console ~> IO
val consoleInterpreter: Console ~> IO = new (Console ~> IO):
  def apply[A](fa: Console[A]): IO[A] = fa match
    case ReadLine()      => IO(scala.io.StdIn.readLine())
    case WriteLine(msg)  => IO(println(msg))

// Test interpreter: Console ~> List (สำหรับ testing)
// ...

// Run program
val program2: ConsoleIO[Unit] = for
  _    <- writeLine("What's your name?")
  name <- readLine
  _    <- writeLine(s"Hello, $name!")
yield ()

// program2.foldMap(consoleInterpreter).unsafeRunSync()
```

---

## สรุป

### Advanced Cats Type Classes

| Type Class | Purpose | Example |
|------------|---------|---------|
| `Bifunctor[F[_, _]]` | map ทั้งสอง type params | `Either`, `(A, B)` |
| `Bitraverse[F[_, _]]` | traverse ทั้งสอง sides | Validate both sides |
| `ContravariantMonoidal` | product ของ contravariant | CSV Encoder, Ordering |
| `Parallel[M, F]` | Sequential vs Parallel | Either vs Validated |
| `CommutativeApplicative` | Order-independent apply | Option, Set |

### Cats Data Types

| Type | Description | Use When |
|------|-------------|---------|
| `Ior[A, B]` | Left, Right, หรือ Both | Partial success + warnings |
| `Chain[A]` | O(1) append/prepend | Accumulating logs |
| `NonEmptyChain[A]` | Non-empty Chain | Guaranteed non-empty errors |
| `ValidatedNec[E, A]` | Accumulate errors | User input validation |
| `IorNec[E, A]` | Warnings + result | Import with warnings |

### Parallel vs Sequential

```scala
// Sequential (Either/Monad): fails fast, can depend on previous results
for
  a <- parseA(input)   // ถ้า fail, หยุดเลย
  b <- parseB(a)       // ขึ้นอยู่กับ a
yield (a, b)

// Parallel (Validated/Applicative): accumulates all errors, independent
(parseA(input), parseB(input)).mapN { case (a, b) => (a, b) }

// Parallel Either: ใช้ parMapN
(parseA(input), parseB(input)).parMapN { case (a, b) => (a, b) }
```

---

*[← BONUS ส่วนที่ 103: Optics กับ Monocle](part-103-bonus-optics.md) | [BONUS ส่วนที่ 105: Metaprogramming →](part-105-bonus-metaprogramming.md)*
