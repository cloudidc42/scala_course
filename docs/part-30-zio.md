# Part 30: ZIO - Functional Effects

## สารบัญ
1. [ZIO Overview](#zio-overview)
2. [ZIO Effects](#zio-effects)
3. [Error Handling](#error-handling)
4. [Resource Management](#resource-management)
5. [Concurrency](#concurrency)
6. [Layers (Dependency Injection)](#layers)

---

## ZIO Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "dev.zio" %% "zio"              % "2.0.19",
  "dev.zio" %% "zio-streams"      % "2.0.19",
  "dev.zio" %% "zio-test"         % "2.0.19" % Test,
  "dev.zio" %% "zio-test-sbt"     % "2.0.19" % Test,
  "dev.zio" %% "zio-http"         % "3.0.0-RC2",
  "dev.zio" %% "zio-json"         % "0.6.2",
  "dev.zio" %% "zio-logging"      % "2.1.14"
)
```

### ZIO Type Signature

```
ZIO[R, E, A]
- R: Environment (dependencies needed)
- E: Error type
- A: Success value type

Common aliases:
- Task[A]    = ZIO[Any, Throwable, A]  (any error)
- UIO[A]     = ZIO[Any, Nothing, A]   (can't fail)
- URIO[R, A] = ZIO[R, Nothing, A]     (can't fail, needs env)
- IO[E, A]   = ZIO[Any, E, A]         (no env needed)
- RIO[R, A]  = ZIO[R, Throwable, A]   (needs env)
```

---

## ZIO Effects

### Creating ZIO Effects

```scala
import zio.*

// Pure value
val pure: UIO[Int] = ZIO.succeed(42)

// Synchronous effect
val getTime: UIO[Long] = ZIO.succeed(System.currentTimeMillis())

// Potentially failing effect
val parseInt: IO[NumberFormatException, Int] =
  ZIO.attempt("42".toInt).refineToOrDie[NumberFormatException]

// From Future
import scala.concurrent.Future
val fromFuture: Task[Int] =
  ZIO.fromFuture(implicit ec => Future.successful(42))

// From Option
val fromOption: IO[None.type, Int] = ZIO.fromOption(Some(42))

// Fail explicitly
val fail: IO[String, Nothing] = ZIO.fail("Something went wrong")

// Die (defect, not recoverable)
val die: UIO[Nothing] = ZIO.die(new RuntimeException("Fatal error"))
```

### Running ZIO

```scala
import zio.*

object Main extends ZIOAppDefault:
  def run: ZIO[Any, Throwable, Unit] = for
    _ <- Console.printLine("Hello, ZIO!")
    _ <- Console.printLine("Processing...")
  yield ()

// Using ZIO.unsafeRun (for testing)
val result = Unsafe.unsafe { implicit unsafe =>
  Runtime.default.unsafe.run(ZIO.succeed(42)).getOrThrow()
}
```

### Basic Operations

```scala
import zio.*

// map, flatMap
val doubled = ZIO.succeed(21).map(_ * 2)
val chained = ZIO.succeed(5).flatMap(n => ZIO.succeed(n * 2))

// for comprehension
val program = for
  x <- ZIO.succeed(10)
  y <- ZIO.succeed(20)
  _ <- Console.printLine(s"Sum: ${x + y}")
yield x + y

// tap: run side effect without changing result
val withLog = ZIO.succeed(42)
  .tap(n => Console.printLine(s"Got: $n"))

// zip: combine two effects
val combined = ZIO.succeed(1) zip ZIO.succeed("hello")
// ZIO[Any, Nothing, (Int, String)]

// zipWith: combine with function
val summed = ZIO.succeed(3).zipWith(ZIO.succeed(4))(_ + _)
```

---

## Error Handling

### Error Operations

```scala
import zio.*

// catchAll: handle all errors
val recovered = ZIO.fail("error").catchAll(e => ZIO.succeed(s"Recovered: $e"))

// catchSome: handle specific errors
val handled = ZIO.attempt("abc".toInt).catchSome {
  case _: NumberFormatException => ZIO.succeed(-1)
}

// orElse: fallback
val withFallback = ZIO.fail("primary failed").orElse(ZIO.succeed("fallback"))

// fold: handle both success and failure
val folded = ZIO.succeed(42).fold(
  error => s"Error: $error",
  value => s"Value: $value"
)

// either: convert to Either
val asEither: UIO[Either[String, Int]] = ZIO.fail("oops").either

// mapError: transform error type
val mapped = ZIO.fail(new RuntimeException("error"))
  .mapError(_.getMessage)

// retrying
import Schedule.*
val withRetry = ZIO.fail("temp error")
  .retry(recurs(3) && spaced(1.second))

// timeout
val withTimeout = ZIO.succeed("slow").delay(5.seconds)
  .timeout(3.seconds)  // returns Option[A]
```

### Error Types

```scala
// Custom error hierarchy
sealed trait AppError
case class DatabaseError(msg: String) extends AppError
case class ValidationError(field: String, msg: String) extends AppError
case class NotFoundError(id: Long) extends AppError

// ZIO with custom error
def findUser(id: Long): IO[AppError, String] =
  if id > 0 then ZIO.succeed(s"User-$id")
  else ZIO.fail(NotFoundError(id))

// Pattern match on error
val result = findUser(-1).catchAll {
  case NotFoundError(id) => ZIO.succeed(s"Default for user $id")
  case other             => ZIO.fail(other)
}
```

---

## Resource Management

### ZIO.scoped and ZLayer

```scala
import zio.*

// Scoped resources (auto-closed when scope ends)
def openFile(path: String): ZIO[Scope, Throwable, java.io.BufferedReader] =
  ZIO.acquireRelease(
    ZIO.attempt(new java.io.BufferedReader(new java.io.FileReader(path)))
  )(reader => ZIO.succeed(reader.close()))

// Use scoped resource
val readLines = ZIO.scoped {
  for
    reader <- openFile("data.txt")
    lines  <- ZIO.attempt {
      Iterator.continually(reader.readLine())
        .takeWhile(_ != null)
        .toList
    }
  yield lines
}

// Database connection
trait Connection:
  def query(sql: String): Task[List[String]]
  def close(): UIO[Unit]

def makeConnection(url: String): ZIO[Scope, Throwable, Connection] =
  ZIO.acquireRelease(
    ZIO.attempt {
      new Connection:
        def query(sql: String): Task[List[String]] = ZIO.succeed(List("row1", "row2"))
        def close(): UIO[Unit] = ZIO.unit
    }
  )(conn => conn.close())
```

---

## Concurrency

### Fibers

```scala
import zio.*

// Fork: run in background
val background = ZIO.sleep(2.seconds) *> ZIO.succeed("background done")
val fiber: UIO[Fiber[Nothing, String]] = background.fork

// Join fiber
val result = for
  f      <- background.fork
  _      <- ZIO.sleep(1.second)
  result <- f.join
yield result

// Race: first to complete wins
val race = ZIO.sleep(2.seconds).as("slow") race ZIO.sleep(1.second).as("fast")

// Both: run in parallel
val both = ZIO.succeed(1) zipPar ZIO.succeed(2)
// runs both in parallel

// collectAllPar: parallel traverse
val tasks = List(ZIO.succeed(1), ZIO.succeed(2), ZIO.succeed(3))
val parallel = ZIO.collectAllPar(tasks)

// foreachPar: parallel map
val results = ZIO.foreachPar(List(1, 2, 3))(n => ZIO.succeed(n * 2))
```

### STM (Software Transactional Memory)

```scala
import zio.stm.*

// TRef: transactional reference
val counter = for
  ref    <- TRef.make(0).commit
  _      <- STM.foreach(1 to 100)(i => ref.update(_ + i)).commit
  result <- ref.get.commit
yield result

// TSemaphore: transactional semaphore
val sema = for
  semaphore <- TSemaphore.make(3).commit
  _         <- ZIO.foreachPar(1 to 10)(i =>
    semaphore.withPermit(
      ZIO.succeed(s"Task $i") *> ZIO.sleep(100.millis)
    ).commit
  )
yield ()

// TQueue
val queue = for
  q       <- TQueue.bounded[Int](10).commit
  producer = ZIO.foreach(1 to 5)(i => q.offer(i).commit).fork
  consumer = ZIO.foreach(1 to 5)(_ => q.take.commit).fork
  _       <- producer
  c       <- consumer
  items   <- c.join
yield items
```

---

## Layers

### Dependency Injection กับ ZLayer

```scala
import zio.*

// Service interfaces
trait UserRepository:
  def findById(id: Long): IO[String, Option[String]]
  def save(name: String): IO[String, Long]

trait EmailService:
  def send(to: String, msg: String): IO[String, Unit]

// Implementations
object UserRepositoryLive:
  val layer: ULayer[UserRepository] = ZLayer.succeed(
    new UserRepository:
      def findById(id: Long): IO[String, Option[String]] =
        ZIO.succeed(if id > 0 then Some(s"User-$id") else None)
      def save(name: String): IO[String, Long] =
        ZIO.succeed(name.hashCode.toLong.abs)
  )

object EmailServiceLive:
  val layer: ULayer[EmailService] = ZLayer.succeed(
    new EmailService:
      def send(to: String, msg: String): IO[String, Unit] =
        Console.printLine(s"Sending '$msg' to $to").orDie
  )

// Using services
def notifyUser(userId: Long, message: String): ZIO[UserRepository & EmailService, String, Unit] =
  for
    repo  <- ZIO.service[UserRepository]
    email <- ZIO.service[EmailService]
    user  <- repo.findById(userId)
    _     <- user match
      case Some(u) => email.send(u, message)
      case None    => ZIO.fail(s"User $userId not found")
  yield ()

// Compose layers
val appLayer = UserRepositoryLive.layer ++ EmailServiceLive.layer

// Run with layer
val program = notifyUser(1L, "Welcome!").provide(appLayer)

object App extends ZIOAppDefault:
  def run = program
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ ZIO type signature: ZIO[R, E, A]
- ✅ Creating ZIO effects
- ✅ Error handling: catchAll, retry, timeout
- ✅ Resource management: ZIO.scoped, acquireRelease
- ✅ Concurrency: fibers, race, parallel operations
- ✅ STM: transactional state
- ✅ ZLayer: dependency injection

---

*[← Part 29: Apache Spark](part-29-spark.md) | [Part 31: Scala Macros →](part-31-macros.md)*
