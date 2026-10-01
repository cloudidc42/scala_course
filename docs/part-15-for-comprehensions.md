# Part 15: For Comprehensions เชิงลึก

## สารบัญ
1. [For Comprehension พื้นฐาน](#for-comprehension-พื้นฐาน)
2. [Desugaring](#desugaring)
3. [Monad Composition](#monad-composition)
4. [Custom Monads](#custom-monads)
5. [แบบฝึกหัด](#แบบฝึกหัด)

---

## For Comprehension พื้นฐาน

### Syntax และการใช้งาน

```scala
// For comprehension: syntactic sugar สำหรับ map/flatMap/filter

// Basic: เหมือน nested loops
val pairs = for
  x <- 1 to 3
  y <- 1 to 3
yield (x, y)

println(pairs.toList)
// List((1,1),(1,2),(1,3),(2,1),...,(3,3))

// กับ guard (if)
val evenPairs = for
  x <- 1 to 5
  y <- 1 to 5
  if x + y == 6
yield (x, y)

println(evenPairs.toList)
// List((1,5),(2,4),(3,3),(4,2),(5,1))

// กับ definitions
val result = for
  x <- 1 to 5
  squared = x * x    // definition (val, ไม่ใช่ generator)
  if squared > 5
yield s"$x^2 = $squared"

println(result.toList)
// List(3^2 = 9, 4^2 = 16, 5^2 = 25)
```

---

## Desugaring

### for comprehension → map/flatMap

```scala
// for comprehension แปลงเป็น:
// - yield → map
// - หลาย generators → flatMap
// - if → withFilter (หรือ filter)

// แบบนี้:
val r1 = for
  x <- List(1, 2, 3)
  y <- List(10, 20)
yield x + y

// เท่ากับแบบนี้:
val r2 = List(1, 2, 3).flatMap(x =>
  List(10, 20).map(y => x + y)
)

println(r1 == r2)  // true

// กับ guard:
val r3 = for
  x <- List(1, 2, 3, 4, 5)
  if x % 2 == 0
  y = x * x
yield y

// เท่ากับ:
val r4 = List(1, 2, 3, 4, 5)
  .withFilter(x => x % 2 == 0)
  .map(x => { val y = x * x; y })

println(r3 == r4)  // true

// for ไม่มี yield → foreach
for x <- List(1, 2, 3) do println(x)
// เท่ากับ
List(1, 2, 3).foreach(println)
```

### Nested for → flatMap Chain

```scala
// ยิ่งซ้อนกัน ยิ่งมี flatMap มาก

val nested = for
  a <- List(1, 2)
  b <- List(10, 20)
  c <- List(100, 200)
yield a + b + c

// เท่ากับ:
val nested2 = List(1, 2).flatMap { a =>
  List(10, 20).flatMap { b =>
    List(100, 200).map { c =>
      a + b + c
    }
  }
}

println(nested == nested2)  // true
println(nested.toList)
// List(111, 211, 121, 221, 112, 212, 122, 222)
```

---

## Monad Composition

### Option ใน for

```scala
case class Config(host: String, port: Int, path: String)

val configs = Map(
  "host" -> "localhost",
  "port" -> "8080",
  "path" -> "/api"
)

// ด้วย for comprehension
def loadConfig(map: Map[String, String]): Option[Config] =
  for
    host <- map.get("host")
    port <- map.get("port").flatMap(_.toIntOption)
    path <- map.get("path")
  yield Config(host, port, path)

println(loadConfig(configs))
// Some(Config(localhost,8080,/api))

println(loadConfig(Map("host" -> "localhost")))
// None (port and path missing)

println(loadConfig(Map("host" -> "localhost", "port" -> "invalid", "path" -> "/")))
// None (port not a number)
```

### Either ใน for

```scala
sealed trait ValidationError
case class MissingField(field: String) extends ValidationError
case class InvalidValue(field: String, reason: String) extends ValidationError

case class UserRegistration(
  username: String,
  email: String,
  age: Int,
  password: String
)

def validateUsername(u: String): Either[ValidationError, String] =
  if u.length < 3 then Left(InvalidValue("username", "Too short (min 3 chars)"))
  else if u.contains(" ") then Left(InvalidValue("username", "No spaces allowed"))
  else Right(u.toLowerCase)

def validateEmail(e: String): Either[ValidationError, String] =
  val emailRegex = """^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$"""
  if e.matches(emailRegex) then Right(e.toLowerCase)
  else Left(InvalidValue("email", "Invalid email format"))

def validateAge(a: String): Either[ValidationError, Int] =
  a.toIntOption match
    case None    => Left(InvalidValue("age", "Must be a number"))
    case Some(n) if n < 18 => Left(InvalidValue("age", "Must be 18 or older"))
    case Some(n) if n > 120 => Left(InvalidValue("age", "Unrealistic age"))
    case Some(n) => Right(n)

def validatePassword(p: String): Either[ValidationError, String] =
  if p.length < 8 then Left(InvalidValue("password", "Too short"))
  else if !p.exists(_.isUpper) then Left(InvalidValue("password", "Need uppercase"))
  else if !p.exists(_.isDigit) then Left(InvalidValue("password", "Need digit"))
  else Right(p)

def registerUser(
  username: String,
  email: String,
  age: String,
  password: String
): Either[ValidationError, UserRegistration] =
  for
    u <- validateUsername(username)
    e <- validateEmail(email)
    a <- validateAge(age)
    p <- validatePassword(password)
  yield UserRegistration(u, e, a, p)

// ทดสอบ
println(registerUser("alice", "alice@example.com", "25", "Password1"))
// Right(UserRegistration(alice,alice@example.com,25,Password1))

println(registerUser("ab", "invalid-email", "17", "weak"))
// Left(InvalidValue(username,Too short (min 3 chars)))
// short-circuits at first error
```

### Future ใน for

```scala
import scala.concurrent.{Future, ExecutionContext}
import scala.concurrent.ExecutionContext.Implicits.global

// Simulate async operations
def fetchUser(id: Int): Future[String] =
  Future {
    Thread.sleep(10)
    s"User$id"
  }

def fetchOrders(userId: String): Future[List[String]] =
  Future {
    Thread.sleep(10)
    List(s"Order1_$userId", s"Order2_$userId")
  }

def fetchTotal(orders: List[String]): Future[Double] =
  Future {
    Thread.sleep(10)
    orders.length * 100.0
  }

// Sequential with for comprehension
def getUserTotal(userId: Int): Future[String] =
  for
    user   <- fetchUser(userId)
    orders <- fetchOrders(user)
    total  <- fetchTotal(orders)
  yield s"$user: $total"

// Note: ใน for comprehension, futures run sequentially
// ถ้าต้องการ parallel ต้องสร้าง futures ก่อน

// Parallel futures
def getUserTotalParallel(userId: Int): Future[String] =
  val userFuture   = fetchUser(userId)          // starts immediately
  val orders2 = List("O1", "O2")
  val totalFuture  = fetchTotal(orders2)         // starts immediately (parallel!)
  for
    user  <- userFuture                          // wait for both
    total <- totalFuture
  yield s"$user: $total"
```

---

## Custom Monads

### สร้าง Monad ของตัวเอง

```scala
// Result monad ที่รวม Option และ Either

sealed trait Result[+A]:
  def map[B](f: A => B): Result[B] = this match
    case Ok(a)   => Ok(f(a))
    case Err(e)  => Err(e)
    case Empty   => Empty

  def flatMap[B](f: A => Result[B]): Result[B] = this match
    case Ok(a)  => f(a)
    case Err(e) => Err(e)
    case Empty  => Empty

  def withFilter(p: A => Boolean): Result[A] = this match
    case Ok(a) if !p(a) => Empty
    case other => other

  def getOrElse[B >: A](default: B): B = this match
    case Ok(a) => a
    case _ => default

  def orElse[B >: A](other: => Result[B]): Result[B] = this match
    case Empty | Err(_) => other
    case ok => ok

case class Ok[A](value: A) extends Result[A]
case class Err(message: String) extends Result[Nothing]
case object Empty extends Result[Nothing]

// ใช้ใน for comprehension
def divideResult(a: Int, b: Int): Result[Double] =
  if b == 0 then Err("Division by zero")
  else Ok(a.toDouble / b)

def sqrtResult(n: Double): Result[Double] =
  if n < 0 then Err("Negative number")
  else Ok(math.sqrt(n))

val computation = for
  x     <- divideResult(100, 4)
  root  <- sqrtResult(x)
  if root > 2.0
yield root

println(computation)  // Ok(5.0)

val fail1 = for
  x <- divideResult(100, 0)  // Err
  y <- sqrtResult(x)
yield y

println(fail1)  // Err(Division by zero)
```

### Writer Monad (Logging)

```scala
// Writer monad: value + log

case class Writer[W, A](value: A, log: List[W]):
  def map[B](f: A => B): Writer[W, B] =
    Writer(f(value), log)

  def flatMap[B](f: A => Writer[W, B]): Writer[W, B] =
    val Writer(b, newLog) = f(value)
    Writer(b, log ++ newLog)

  def withFilter(p: A => Boolean): Writer[W, A] = this  // simplified

object Writer:
  def pure[W, A](a: A): Writer[W, A] = Writer(a, Nil)
  def tell[W](msg: W): Writer[W, Unit] = Writer((), List(msg))

// ใช้งาน
def factorialLogged(n: Int): Writer[String, BigInt] =
  if n <= 0
  then Writer(1, List(s"factorial(0) = 1"))
  else
    for
      prev    <- factorialLogged(n - 1)
      result  =  n * prev
      _       <- Writer.tell(s"factorial($n) = $result")
    yield result

val result = factorialLogged(5)
println(s"Result: ${result.value}")
result.log.foreach(println)
// factorial(0) = 1
// factorial(1) = 1
// factorial(2) = 2
// factorial(3) = 6
// factorial(4) = 24
// factorial(5) = 120
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Pipeline กับ Custom Monad

```scala
// สร้าง IO-like monad สำหรับ deferred computations

case class IO[A](unsafeRun: () => A):
  def map[B](f: A => B): IO[B] = IO(() => f(unsafeRun()))
  def flatMap[B](f: A => IO[B]): IO[B] = IO(() => f(unsafeRun()).unsafeRun())

object IO:
  def pure[A](a: A): IO[A] = IO(() => a)
  def println(s: String): IO[Unit] = IO(() => Predef.println(s))
  def readLine(): IO[String] = IO(() => scala.io.StdIn.readLine())

// TODO: ใช้ for comprehension เพื่อสร้าง program
def program: IO[Unit] = for
  _    <- IO.println("Enter your name:")
  name <- IO.readLine()
  _    <- IO.println(s"Hello, $name!")
yield ()

// program.unsafeRun()  // Run program
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ For Comprehension syntax ครบถ้วน
- ✅ Desugaring: แปลงเป็น map/flatMap/withFilter
- ✅ Monad composition: Option, Either, Future
- ✅ Custom Monads: Result, Writer
- ✅ IO Monad pattern

---

*[← Part 14: Higher-Order Functions](part-14-higher-order-functions.md) | [Part 16: Option และ Either →](part-16-option-either.md)*
