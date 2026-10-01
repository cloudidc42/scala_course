# Part 63: ZIO - Functional Effect System

## สารบัญ
1. [ZIO คืออะไร](#zio-คืออะไร)
2. [ZIO vs Cats Effect](#zio-vs-cats-effect)
3. [ZIO[R, E, A] - Environment, Error, Value](#zior-e-a---environment-error-value)
4. [ZLayer สำหรับ Dependency Injection](#zlayer-สำหรับ-dependency-injection)
5. [ZIO Streams](#zio-streams)
6. [ZIO Schedule สำหรับ Retry](#zio-schedule-สำหรับ-retry)
7. [ZIO Test](#zio-test)
8. [ตัวอย่างแอปพลิเคชันครบถ้วน](#ตัวอย่างแอปพลิเคชันครบถ้วน)
9. [สรุป](#สรุป)

---

## ZIO คืออะไร

ZIO คือ functional effect library สำหรับ Scala ที่ออกแบบมาเพื่อทำให้การเขียนโปรแกรม concurrent, asynchronous, และ resilient เป็นเรื่องง่ายและ type-safe

### ZIO Type Signature

```
ZIO[R, E, A]
    │  │  │
    │  │  └─── A: ประเภทของค่าที่ได้เมื่อสำเร็จ (value)
    │  └────── E: ประเภทของ error เมื่อล้มเหลว (error)
    └───────── R: สิ่งที่ต้องการจาก environment (requirement)

Aliases ที่ใช้บ่อย:
Task[A]     = ZIO[Any, Throwable, A]  ← ใช้บ่อยที่สุด
IO[E, A]    = ZIO[Any, E, A]
UIO[A]      = ZIO[Any, Nothing, A]   ← ไม่มี error
URIO[R, A]  = ZIO[R, Nothing, A]
RIO[R, A]   = ZIO[R, Throwable, A]
```

### ติดตั้ง Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "dev.zio" %% "zio"         % "2.1.6",
  "dev.zio" %% "zio-streams" % "2.1.6",
  "dev.zio" %% "zio-test"    % "2.1.6" % Test,
  "dev.zio" %% "zio-test-sbt"% "2.1.6" % Test,
  // Database
  "dev.zio" %% "zio-jdbc"    % "0.1.2",
  // HTTP
  "dev.zio" %% "zio-http"    % "3.0.0",
  // JSON
  "dev.zio" %% "zio-json"    % "0.7.1",
  // Config
  "dev.zio" %% "zio-config"  % "4.0.2",
)
```

---

## ZIO vs Cats Effect

### เปรียบเทียบ Syntax

```scala
// === Cats Effect ===
import cats.effect.IO
import cats.effect.unsafe.implicits.global

val catsProgram: IO[Unit] = for
  _ <- IO.println("Hello from Cats Effect!")
  n <- IO(42)
  _ <- IO.println(s"Number: $n")
yield ()

catsProgram.unsafeRunSync()

// === ZIO ===
import zio.*
import zio.Console.*

val zioProgram: Task[Unit] = for
  _ <- printLine("Hello from ZIO!")
  n <- ZIO.succeed(42)
  _ <- printLine(s"Number: $n")
yield ()

Unsafe.unsafe { implicit u =>
  Runtime.default.unsafe.run(zioProgram).getOrThrowFiberFailure()
}
```

### เปรียบเทียบ Error Handling

```scala
// === Cats Effect ===
import cats.effect.IO

val catsWithError: IO[Int] =
  IO.raiseError[Int](new RuntimeException("oops"))
    .handleErrorWith { ex =>
      IO(println(s"Caught: ${ex.getMessage}")) *> IO.pure(-1)
    }

// ปัญหา: IO[Int] ไม่บอกว่า error type คืออะไร
// ต้องใช้ EitherT สำหรับ typed errors

// === ZIO ===
import zio.*

val zioWithError: IO[String, Int] =
  ZIO.fail("Custom error")
    .catchAll { err =>
      ZIO.logError(s"Caught: $err") *> ZIO.succeed(-1)
    }

// ZIO[Any, String, Int] บอก error type ชัดเจน!
// ไม่จำเป็นต้องใช้ EitherT
```

### เปรียบเทียบ Concurrency

```scala
// === Cats Effect ===
import cats.effect.{IO, Fiber}
import cats.effect.std.Queue
import cats.effect.implicits.*

val catsConcurrent: IO[Unit] = for
  fiber1 <- IO.sleep(100.millis).start
  fiber2 <- IO.sleep(200.millis).start
  _      <- fiber1.join
  _      <- fiber2.join
yield ()

// === ZIO ===
import zio.*

val zioConcurrent: Task[Unit] = for
  fiber1 <- ZIO.sleep(100.millis).fork
  fiber2 <- ZIO.sleep(200.millis).fork
  _      <- fiber1.join
  _      <- fiber2.join
yield ()

// ZIO มี fiber, Ref, Queue built-in โดยตรง
// ไม่ต้อง import เพิ่ม
```

### ความแตกต่างหลัก

```
Feature            | Cats Effect         | ZIO
-------------------|--------------------|-----------------
Error Type         | Always Throwable   | Typed E
Environment        | ReaderT pattern    | ZIO[R, E, A]
Dependency Inject  | tagless final      | ZLayer
Fiber model        | Fiber[F, E, A]     | Fiber[E, A]
Ref/Queue          | Separate lib       | Built-in
Stream             | FS2                | ZStream
Testing            | cats-effect-testing| ZIO Test
Learning Curve     | Steeper            | Gentler
Ecosystem          | Larger             | Growing fast
```

---

## ZIO[R, E, A] - Environment, Error, Value

### ZIO.succeed และ ZIO.fail

```scala
import zio.*

// สร้าง ZIO effects
val succeed: UIO[Int]    = ZIO.succeed(42)
val fail: IO[String, Nothing] = ZIO.fail("Something went wrong")
val failWith: Task[Int]  = ZIO.attempt(throw new RuntimeException("boom"))

// จาก Option/Either
val fromOption: IO[String, Int] =
  ZIO.fromOption(Some(42)).mapError(_ => "No value")

val fromEither: IO[String, Int] =
  ZIO.fromEither(Right(42))

val fromFuture: Task[String] =
  ZIO.fromFuture { implicit ec =>
    scala.concurrent.Future.successful("from future")
  }
```

### การ Compose Effects

```scala
import zio.*

// map, flatMap, for-comprehension
val computation: Task[String] = for
  x <- ZIO.succeed(10)
  y <- ZIO.succeed(20)
  sum = x + y
  result <- ZIO.succeed(s"Sum: $sum")
yield result

// zip: รัน 2 effects แล้ว tuple ผล
val zipped: Task[(Int, String)] =
  ZIO.succeed(42) zip ZIO.succeed("hello")

// zipPar: รัน 2 effects พร้อมกัน
val parallel: Task[(Int, String)] =
  ZIO.succeed(42) <&> ZIO.succeed("hello")

// orElse: ใช้ fallback ถ้า fail
val withFallback: IO[String, Int] =
  ZIO.fail("primary failed") orElse ZIO.succeed(0)
```

### Error Handling

```scala
import zio.*

// catchAll: จัดการทุก error
val handled: UIO[Int] =
  ZIO.fail("error")
    .catchAll(err => ZIO.succeed(-1))

// catchSome: จัดการบาง error
val partialHandle: IO[String, Int] =
  ZIO.fail("timeout")
    .catchSome {
      case "timeout" => ZIO.succeed(0)
      // "other errors" ยังคง propagate
    }

// mapError: แปลง error type
val mappedError: IO[AppError, Int] =
  ZIO.fail("raw error").mapError(AppError.apply)

case class AppError(message: String)

// orDie: แปลง error เป็น defect (ไม่ recover ได้)
val orDie: UIO[Int] =
  ZIO.fail(new RuntimeException("fatal")).orDie

// either: แปลงเป็น ZIO[R, Nothing, Either[E, A]]
val asEither: UIO[Either[String, Int]] =
  ZIO.fail("oops").either

// fold: จัดการทั้ง success และ failure
val folded: UIO[String] =
  ZIO.fail("error").fold(
    onFailure = err => s"Failed: $err",
    onSuccess = value => s"Success: $value"
  )
```

### Concurrency และ Fibers

```scala
import zio.*

// Fork: รัน effect ใน background fiber
val concurrent: Task[Unit] = for
  fiber1 <- ZIO.sleep(200.millis).as("result-1").fork
  fiber2 <- ZIO.sleep(100.millis).as("result-2").fork
  r2     <- fiber2.join  // รอ fiber2 ก่อน
  r1     <- fiber1.join
  _      <- ZIO.log(s"Got: $r1, $r2")
yield ()

// foreachPar: map แบบ parallel
val parallelMap: Task[List[Int]] =
  ZIO.foreachPar(1 to 10) { n =>
    ZIO.succeed(n * 2).delay(100.millis)
  }

// foreachPar with limit
val limitedParallel: Task[List[Int]] =
  ZIO.foreachParN(4)(1 to 100) { n =>
    ZIO.succeed(n * n)
  }

// collectAllPar: รัน list of effects พร้อมกัน
val effects: List[Task[Int]] = (1 to 5).map(n => ZIO.succeed(n * 10)).toList
val allResults: Task[List[Int]] = ZIO.collectAllPar(effects)

// race: รัน 2 effects พร้อมกัน เอาที่เสร็จก่อน
val fastest: Task[String] =
  ZIO.sleep(200.millis).as("slow") race ZIO.sleep(50.millis).as("fast")
  // ได้ "fast"
```

### Ref - Concurrent State

```scala
import zio.*

object CounterExample:
  def program: Task[Unit] =
    for
      counter <- Ref.make(0)  // mutable, thread-safe state

      // increment พร้อมกัน 1000 ครั้ง
      _ <- ZIO.foreachParDiscard(1 to 1000) { _ =>
        counter.update(_ + 1)
      }

      value <- counter.get
      _     <- ZIO.log(s"Final count: $value")  // 1000 เสมอ
    yield ()

// Queue - async queue
object QueueExample:
  def program: Task[Unit] =
    for
      queue <- Queue.bounded[String](100)

      // Producer
      producer = ZIO.foreach(1 to 5) { n =>
        queue.offer(s"item-$n") *> ZIO.sleep(100.millis)
      }

      // Consumer
      consumer = ZIO.foreach(1 to 5) { _ =>
        queue.take.flatMap(item => ZIO.log(s"Consumed: $item"))
      }

      // รัน producer และ consumer พร้อมกัน
      _ <- producer zipPar consumer
    yield ()
```

---

## ZLayer สำหรับ Dependency Injection

### ZLayer คืออะไร

```
ZLayer[RIn, E, ROut]:
  RIn  = dependencies ที่ต้องการ
  E    = error ที่อาจเกิดขึ้นระหว่าง construction
  ROut = service ที่ provide ออกไป

ZLayer.succeed(value)    = ZLayer[Any, Nothing, MyService]
ZLayer.fromZIO(effect)   = ZLayer[Any, E, MyService]
ZLayer(acquire)(release) = Scoped layer (managed resource)
```

### สร้าง Services ด้วย ZLayer

```scala
import zio.*

// 1. กำหนด interface
trait UserRepository:
  def findById(id: Long): Task[Option[User]]
  def create(user: User): Task[User]
  def update(user: User): Task[User]
  def delete(id: Long): Task[Unit]

case class User(id: Long, name: String, email: String, active: Boolean = true)

// 2. สร้าง implementation
case class InMemoryUserRepository(users: Ref[Map[Long, User]]) extends UserRepository:
  def findById(id: Long): Task[Option[User]] =
    users.get.map(_.get(id))

  def create(user: User): Task[User] =
    users.update(_ + (user.id -> user)).as(user)

  def update(user: User): Task[User] =
    users.update(_.updated(user.id, user)).as(user)

  def delete(id: Long): Task[Unit] =
    users.update(_ - id)

object InMemoryUserRepository:
  // ZLayer สำหรับ UserRepository
  val layer: ULayer[UserRepository] =
    ZLayer.fromZIO {
      Ref.make(Map.empty[Long, User])
        .map(InMemoryUserRepository.apply)
    }

// 3. สร้าง Service ที่ depend on UserRepository
trait UserService:
  def getUser(id: Long): Task[User]
  def registerUser(name: String, email: String): Task[User]
  def deactivateUser(id: Long): Task[Unit]

case class UserServiceLive(
  userRepo: UserRepository,
  idGen: IdGenerator
) extends UserService:

  def getUser(id: Long): Task[User] =
    userRepo.findById(id).flatMap {
      case Some(user) => ZIO.succeed(user)
      case None       => ZIO.fail(new NoSuchElementException(s"User $id not found"))
    }

  def registerUser(name: String, email: String): Task[User] =
    for
      id   <- idGen.nextId
      user  = User(id, name, email)
      saved <- userRepo.create(user)
      _     <- ZIO.log(s"Registered user: $id")
    yield saved

  def deactivateUser(id: Long): Task[Unit] =
    for
      user    <- getUser(id)
      updated  = user.copy(active = false)
      _       <- userRepo.update(updated)
      _       <- ZIO.log(s"Deactivated user: $id")
    yield ()

object UserServiceLive:
  val layer: URLayer[UserRepository & IdGenerator, UserService] =
    ZLayer.fromFunction(UserServiceLive.apply)

// IdGenerator service
trait IdGenerator:
  def nextId: Task[Long]

object IdGenerator:
  val layer: ULayer[IdGenerator] =
    ZLayer.fromZIO {
      Ref.make(0L).map { counter =>
        new IdGenerator:
          def nextId: Task[Long] =
            counter.updateAndGet(_ + 1)
      }
    }
```

### การใช้งาน ZLayer

```scala
import zio.*

object Main extends ZIOAppDefault:
  def run: Task[Unit] =
    program.provide(
      // ZLayer composition
      UserServiceLive.layer,
      InMemoryUserRepository.layer,
      IdGenerator.layer
    )

  val program: ZIO[UserService, Throwable, Unit] =
    for
      service  <- ZIO.service[UserService]
      user1    <- service.registerUser("Alice", "alice@example.com")
      user2    <- service.registerUser("Bob", "bob@example.com")
      _        <- ZIO.log(s"Created users: ${user1.id}, ${user2.id}")
      found    <- service.getUser(user1.id)
      _        <- ZIO.log(s"Found: ${found.name}")
      _        <- service.deactivateUser(user2.id)
    yield ()
```

### ZLayer Composition

```scala
import zio.*

// Horizontal composition: combine services
val combined: URLayer[Any, UserRepository & IdGenerator] =
  InMemoryUserRepository.layer ++ IdGenerator.layer

// Vertical composition: chain dependencies
val full: URLayer[Any, UserService] =
  combined >>> UserServiceLive.layer

// provide ทั้งหมดในครั้งเดียว
program.provideLayer(full)

// Memoization: layer ถูกสร้างครั้งเดียว แชร์กันทุก consumer
val memoized = someLayer.memoize
```

### Database Layer ด้วย ZIO JDBC

```scala
import zio.*
import zio.jdbc.*

// Database configuration
val dbConfig = JdbcConfig(
  driver = "org.postgresql.Driver",
  url    = "jdbc:postgresql://localhost/mydb",
  user   = "postgres",
  password = "secret"
)

object UserRepo:
  def findById(id: Long): ZIO[ZConnectionPool, Throwable, Option[User]] =
    transaction {
      sql"SELECT id, name, email FROM users WHERE id = $id"
        .query[(Long, String, String)]
        .selectOne
        .map(_.map((id, name, email) => User(id, name, email)))
    }

  def insert(user: User): ZIO[ZConnectionPool, Throwable, Long] =
    transaction {
      sql"INSERT INTO users (name, email) VALUES (${user.name}, ${user.email})"
        .insert
    }

  val layer: URLayer[ZConnectionPool, UserRepository] =
    ZLayer.succeed {
      new UserRepository:
        def findById(id: Long) = UserRepo.findById(id)
        def create(user: User) = UserRepo.insert(user).as(user)
        def update(user: User) = ??? // implement
        def delete(id: Long)   = ??? // implement
    }
```

---

## ZIO Streams

### ZStream พื้นฐาน

```scala
import zio.*
import zio.stream.*

// สร้าง ZStream
val s1: ZStream[Any, Nothing, Int] = ZStream(1, 2, 3, 4, 5)
val s2: ZStream[Any, Nothing, Int] = ZStream.fromIterable(1 to 100)
val s3: ZStream[Any, Nothing, Long] =
  ZStream.tick(1.second).zipWithIndex.map(_._2)

// Transformations
val processed: ZStream[Any, Nothing, String] =
  ZStream(1 to 10: _*)
    .filter(_ % 2 == 0)
    .map(n => s"Even: $n")
    .take(3)

// Running stream
val result: Task[List[String]] =
  processed.runCollect.map(_.toList)

// runForeach
val printing: Task[Unit] =
  ZStream("a", "b", "c").runForeach(ZIO.log(_))

// fold/aggregate
val sum: Task[Int] =
  ZStream(1 to 10: _*).runFold(0)(_ + _)
```

### ZStream advanced

```scala
import zio.*
import zio.stream.*

// mapZIO: async transformation
val asyncProcessed: ZStream[Any, Throwable, String] =
  ZStream(1 to 5: _*)
    .mapZIO { n =>
      ZIO.sleep(100.millis) *> ZIO.succeed(s"processed-$n")
    }

// mapZIOPar: parallel async transformation
val parallelProcessed: ZStream[Any, Throwable, String] =
  ZStream(1 to 20: _*)
    .mapZIOPar(4) { n =>
      ZIO.sleep((100 * scala.util.Random.nextInt(5)).millis)
        .as(s"result-$n")
    }

// groupedWithin: batch by time or count
val batched: ZStream[Any, Nothing, Chunk[Int]] =
  ZStream.fromIterable(1 to 100)
    .groupedWithin(10, 1.second)

// merge: รวม 2 streams
val merged: ZStream[Any, Nothing, Int] =
  ZStream(1, 2, 3).merge(ZStream(10, 20, 30))

// flatMap
val flatMapped: ZStream[Any, Nothing, Int] =
  ZStream(1, 2, 3).flatMap { n =>
    ZStream.fromIterable(List.fill(n)(n))
  }
// 1, 2, 2, 3, 3, 3

// ZStream.fromQueue
val queueStream: ZIO[Any, Throwable, Unit] = for
  queue  <- Queue.bounded[String](100)
  stream  = ZStream.fromQueue(queue)
  fiber  <- stream.runForeach(ZIO.log(_)).fork
  _      <- queue.offer("hello")
  _      <- queue.offer("world")
  _      <- ZIO.sleep(100.millis)
  _      <- fiber.interrupt
yield ()
```

### ZSink

```scala
import zio.*
import zio.stream.*

// Sinks
val collectSink: ZSink[Any, Nothing, Int, Nothing, Chunk[Int]] =
  ZSink.collectAll[Int]

val sumSink: ZSink[Any, Nothing, Int, Nothing, Int] =
  ZSink.sum[Int]

val headSink: ZSink[Any, Nothing, Int, Nothing, Option[Int]] =
  ZSink.head[Int]

// Custom sink
val countEven: ZSink[Any, Nothing, Int, Nothing, Int] =
  ZSink.foldLeft(0) { (count, n) =>
    if n % 2 == 0 then count + 1 else count
  }

// run stream into sink
val evenCount: Task[Int] =
  ZStream(1 to 100: _*).run(countEven)
```

### ZPipeline

```scala
import zio.*
import zio.stream.*

// ZPipeline คือ reusable Flow
val upperPipeline: ZPipeline[Any, Nothing, String, String] =
  ZPipeline.map(_.toUpperCase)

val filterPipeline: ZPipeline[Any, Nothing, String, String] =
  ZPipeline.filter(_.nonEmpty)

// Compose pipelines
val combined: ZPipeline[Any, Nothing, String, String] =
  filterPipeline >>> upperPipeline

// สำหรับ file processing
val linePipeline: ZPipeline[Any, Throwable, Byte, String] =
  ZPipeline.utf8Decode >>> ZPipeline.splitLines

// File reading
val fileContent: ZStream[Any, Throwable, String] =
  ZStream.fromFile(java.nio.file.Paths.get("data.txt").toFile)
    .via(linePipeline)
```

---

## ZIO Schedule สำหรับ Retry

### Schedule พื้นฐาน

```scala
import zio.*

// Retry ด้วย Schedule
val retrySchedule = Schedule.recurs(3)  // retry 3 ครั้ง
val expBackoff = Schedule.exponential(100.millis, factor = 2.0)
val fixedDelay = Schedule.fixed(1.second)
val spaced     = Schedule.spaced(500.millis)

// retry effect
val effect: Task[String] =
  ZIO.attempt {
    if scala.util.Random.nextBoolean()
    then throw new RuntimeException("Transient error")
    else "Success!"
  }

val withRetry: Task[String] =
  effect.retry(Schedule.recurs(5) && Schedule.exponential(100.millis))
```

### Complex Schedules

```scala
import zio.*

// retry สูงสุด 5 ครั้ง ด้วย exponential backoff สูงสุด 30 วินาที
val robustRetry: Schedule[Any, Throwable, Unit] =
  Schedule.recurs(5) &&
  Schedule.exponential(200.millis)
    .jittered  // เพิ่ม random jitter
    .upTo(30.seconds)

// Retry เฉพาะ transient errors
val selectiveRetry: Schedule[Any, Throwable, Unit] =
  Schedule.recurWhile[Throwable] {
    case _: java.net.ConnectException    => true
    case _: java.net.SocketTimeoutException => true
    case _                               => false
  }

// Capped exponential backoff
val cappedBackoff: Schedule[Any, Any, (Long, Duration)] =
  (Schedule.exponential(100.millis) && Schedule.recurs(10))
    .tapOutput { case (attempt, delay) =>
      ZIO.log(s"Attempt ${attempt + 1}, next delay: ${delay.toMillis}ms")
    }

// Retry กับ timeout รวม
val withTimeout: Task[String] =
  effect
    .retry(Schedule.exponential(100.millis) && Schedule.recurs(10))
    .timeoutFail(new RuntimeException("Overall timeout"))(30.seconds)
```

### Schedule สำหรับ Polling

```scala
import zio.*

// Poll จนกว่าจะได้ค่าที่ต้องการ
def pollUntilReady(jobId: String): Task[String] =
  checkJobStatus(jobId)
    .repeat(
      Schedule.recurUntil[JobStatus](_ == JobStatus.Completed) &&
      Schedule.spaced(1.second)
    )
    .map(_.result.getOrElse(""))
    .timeoutFail(new RuntimeException("Job timeout"))(5.minutes)

def checkJobStatus(jobId: String): Task[JobStatus] =
  ZIO.succeed {
    // Simulate polling
    if scala.util.Random.nextInt(5) == 0
    then JobStatus.Completed
    else JobStatus.Running
  }

enum JobStatus:
  case Running, Completed, Failed
  val result: Option[String] = if this == Completed then Some("done") else None
```

---

## ZIO Test

### ตั้งค่า Test

```scala
// build.sbt
testFrameworks += new TestFramework("zio.test.sbt.ZTestFramework")
```

### เขียน Tests

```scala
import zio.*
import zio.test.*
import zio.test.Assertion.*

// ตัวอย่าง service ที่จะทดสอบ
trait MathService:
  def add(a: Int, b: Int): Task[Int]
  def divide(a: Int, b: Int): IO[ArithmeticException, Int]

case class MathServiceLive() extends MathService:
  def add(a: Int, b: Int): Task[Int] = ZIO.succeed(a + b)
  def divide(a: Int, b: Int): IO[ArithmeticException, Int] =
    if b == 0 then ZIO.fail(new ArithmeticException("/ by zero"))
    else ZIO.succeed(a / b)

object MathServiceSpec extends ZIOSpecDefault:
  def spec = suite("MathService")(
    suite("add")(
      test("adds two positive numbers") {
        for
          svc    <- ZIO.service[MathService]
          result <- svc.add(3, 4)
        yield assert(result)(equalTo(7))
      },
      test("adds negative numbers") {
        for
          svc    <- ZIO.service[MathService]
          result <- svc.add(-1, -2)
        yield assert(result)(equalTo(-3))
      }
    ),
    suite("divide")(
      test("divides normally") {
        for
          svc    <- ZIO.service[MathService]
          result <- svc.divide(10, 2)
        yield assert(result)(equalTo(5))
      },
      test("fails on division by zero") {
        for
          svc    <- ZIO.service[MathService]
          result <- svc.divide(10, 0).exit
        yield assert(result)(
          failsWithA[ArithmeticException]
        )
      }
    )
  ).provide(ZLayer.succeed(MathServiceLive()))
```

### Property-Based Testing

```scala
import zio.*
import zio.test.*
import zio.test.Gen.*

object PropertySpec extends ZIOSpecDefault:
  def spec = suite("Property Tests")(
    test("reverse of reverse is identity") {
      check(listOf(int)) { list =>
        val reversed = list.reverse.reverse
        assert(reversed)(equalTo(list))
      }
    },

    test("sorting maintains length") {
      check(listOf(int)) { list =>
        assert(list.sorted.length)(equalTo(list.length))
      }
    },

    test("string concat is associative") {
      check(string, string, string) { (a, b, c) =>
        assert((a + b) + c)(equalTo(a + (b + c)))
      }
    }
  )
```

### Mock Services

```scala
import zio.*
import zio.test.*
import zio.test.mock.*

// สร้าง Mock
@mockable[UserRepository]
object MockUserRepository

// ใช้ Mock ใน tests
object UserServiceSpec extends ZIOSpecDefault:
  def spec = suite("UserService")(
    test("getUser returns user when found") {
      val mockLayer = MockUserRepository.FindById(
        assertion = Assertion.equalTo(1L),
        result    = Expectation.value(Some(User(1L, "Alice", "alice@example.com")))
      )

      for
        service <- ZIO.service[UserService]
        user    <- service.getUser(1L)
      yield assert(user.name)(equalTo("Alice"))
    }.provide(UserServiceLive.layer, mockLayer, IdGenerator.layer)
  )
```

### ZIO Test Utilities

```scala
import zio.*
import zio.test.*

object ClockSpec extends ZIOSpecDefault:
  def spec = suite("Clock Tests")(
    test("schedule fires after delay") {
      for
        ref    <- Ref.make(0)
        fiber  <- ref.update(_ + 1).delay(1.second).fork
        _      <- TestClock.adjust(1.second)  // advance virtual time
        _      <- fiber.join
        count  <- ref.get
      yield assert(count)(equalTo(1))
    },

    test("retry with exponential backoff") {
      for
        count  <- Ref.make(0)
        result <- count.update(_ + 1)
                    .flatMap(_ => count.get)
                    .flatMap { n =>
                      if n < 3 then ZIO.fail("not ready")
                      else ZIO.succeed("done")
                    }
                    .retry(Schedule.exponential(1.second))
                    .race(TestClock.adjust(10.seconds).fork.flatMap(_.join.as("timeout")))
      yield assert(result)(equalTo("done"))
    }
  )
```

---

## ตัวอย่างแอปพลิเคชันครบถ้วน

### REST API ด้วย ZIO HTTP

```scala
package com.example.api

import zio.*
import zio.http.*
import zio.json.*

// Domain
case class Todo(id: Long, title: String, done: Boolean = false)
object Todo:
  given JsonEncoder[Todo] = DeriveJsonEncoder.gen
  given JsonDecoder[Todo] = DeriveJsonDecoder.gen

case class CreateTodo(title: String)
object CreateTodo:
  given JsonDecoder[CreateTodo] = DeriveJsonDecoder.gen

// Repository
trait TodoRepository:
  def findAll(): Task[List[Todo]]
  def findById(id: Long): Task[Option[Todo]]
  def create(title: String): Task[Todo]
  def complete(id: Long): Task[Option[Todo]]
  def delete(id: Long): Task[Boolean]

object TodoRepositoryLive:
  val layer: ULayer[TodoRepository] =
    ZLayer.fromZIO {
      for
        ref    <- Ref.make(Map.empty[Long, Todo])
        counter <- Ref.make(0L)
      yield new TodoRepository:
        def findAll(): Task[List[Todo]] =
          ref.get.map(_.values.toList.sortBy(_.id))

        def findById(id: Long): Task[Option[Todo]] =
          ref.get.map(_.get(id))

        def create(title: String): Task[Todo] =
          for
            id  <- counter.updateAndGet(_ + 1)
            todo = Todo(id, title)
            _   <- ref.update(_ + (id -> todo))
          yield todo

        def complete(id: Long): Task[Option[Todo]] =
          ref.modify { todos =>
            todos.get(id) match
              case Some(todo) =>
                val updated = todo.copy(done = true)
                (Some(updated), todos + (id -> updated))
              case None =>
                (None, todos)
          }

        def delete(id: Long): Task[Boolean] =
          ref.modify { todos =>
            (todos.contains(id), todos - id)
          }
    }

// HTTP Routes
object TodoRoutes:
  def routes(repo: TodoRepository): Routes[Any, Response] =
    Routes(
      // GET /todos
      Method.GET / "todos" -> handler {
        repo.findAll()
          .map(todos => Response.json(todos.toJson))
          .catchAll(ex => ZIO.succeed(Response.internalServerError(ex.getMessage)))
      },

      // GET /todos/:id
      Method.GET / "todos" / long("id") -> handler { (id: Long, _: Request) =>
        repo.findById(id).map {
          case Some(todo) => Response.json(todo.toJson)
          case None       => Response.notFound
        }
      },

      // POST /todos
      Method.POST / "todos" -> handler { (req: Request) =>
        req.body.asString.flatMap { body =>
          body.fromJson[CreateTodo] match
            case Left(err) =>
              ZIO.succeed(Response.badRequest(s"Invalid JSON: $err"))
            case Right(create) =>
              repo.create(create.title)
                .map(todo => Response.json(todo.toJson).status(Status.Created))
        }
      },

      // PATCH /todos/:id/complete
      Method.PATCH / "todos" / long("id") / "complete" -> handler { (id: Long, _: Request) =>
        repo.complete(id).map {
          case Some(todo) => Response.json(todo.toJson)
          case None       => Response.notFound
        }
      },

      // DELETE /todos/:id
      Method.DELETE / "todos" / long("id") -> handler { (id: Long, _: Request) =>
        repo.delete(id).map { deleted =>
          if deleted then Response.noContent
          else Response.notFound
        }
      }
    )

// Main App
object TodoApp extends ZIOAppDefault:
  def run: Task[Unit] =
    (for
      repo   <- ZIO.service[TodoRepository]
      server  = Server.serve(TodoRoutes.routes(repo))
      _      <- ZIO.log("Starting Todo API on :8080")
      _      <- server
    yield ())
    .provide(
      TodoRepositoryLive.layer,
      Server.defaultWithPort(8080)
    )
```

---

## สรุป

ZIO เป็น effect system ที่ทรงพลังสำหรับ Scala โดยมีจุดเด่นที่:

- ✅ **Type-safe errors**: `ZIO[R, E, A]` บอก error type ที่ compile time
- ✅ **Dependency Injection**: `ZLayer` เป็น first-class DI ที่ composable
- ✅ **Built-in concurrency**: Fiber, Ref, Queue, STM ใน core library
- ✅ **ZIO Streams**: streaming pipeline ที่ integrate กับ ZIO
- ✅ **ZIO Schedule**: retry/repeat logic ที่ declarative
- ✅ **ZIO Test**: test framework ที่ powerful พร้อม property testing

เมื่อเปรียบกับ Cats Effect:
- ZIO มี error typing ที่ดีกว่า
- ZIO มี DI built-in ผ่าน ZLayer
- Cats Effect มี ecosystem ที่ใหญ่กว่าเล็กน้อย
- ทั้งคู่เหมาะกับ production-grade applications

---

*[← Part 62: Akka Streams](part-62-akka-streams.md) | [Part 64: Monix →](part-64-monix.md)*
