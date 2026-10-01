# ตอนที่ 87: Testing with Cats Effect

## สารบัญ

1. [cats-effect-testing Library](#cats-effect-testing-library)
2. [IOSpec และ AsyncIOSpec](#iospec-และ-asynciospec)
3. [Mocking IO Effects](#mocking-io-effects)
4. [TestClock และ TestConsole](#testclock-และ-testconsole)
5. [Property-Based Testing กับ IO](#property-based-testing-กับ-io)
6. [Integration Tests กับ Resource](#integration-tests-กับ-resource)
7. [Complete Test Suite Example](#complete-test-suite-example)
8. [สรุป](#สรุป)

---

## cats-effect-testing Library

cats-effect-testing ช่วยให้เขียน Tests สำหรับ code ที่ใช้ Cats Effect ได้อย่างสะดวก

### การตั้งค่า build.sbt

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.1"

lazy val root = (project in file("."))
  .settings(
    name := "cats-effect-testing-example",
    libraryDependencies ++= Seq(
      // Cats Effect
      "org.typelevel" %% "cats-effect"          % "3.5.2",
      
      // Testing frameworks
      "org.typelevel" %% "cats-effect-testing-scalatest" % "1.5.0" % Test,
      "org.typelevel" %% "munit-cats-effect"    % "2.0.0"   % Test,
      "org.scalatest" %% "scalatest"             % "3.2.17"  % Test,
      "org.scalameta" %% "munit"                 % "1.0.0"   % Test,
      
      // Mocking
      "org.mockito"   %% "mockito-scala"         % "1.17.14" % Test,
      
      // Property-based testing
      "org.typelevel" %% "scalacheck-effect-munit" % "2.0.0" % Test,
      "org.scalacheck" %% "scalacheck"           % "1.17.0"  % Test,
      
      // Test resources
      "org.typelevel" %% "cats-effect-testkit"   % "3.5.2"   % Test
    ),
    
    testFrameworks += new TestFramework("munit.Framework")
  )
```

---

## IOSpec และ AsyncIOSpec

### CatsEffectSuite กับ MUnit

```scala
package com.example.test

import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import cats.syntax.all.*
import munit.CatsEffectSuite
import org.scalatest.matchers.should.Matchers
import org.scalatest.freespec.AsyncFreeSpec

// ============================================================
// MUnit style (แนะนำ)
// ============================================================

class BasicIoSuite extends CatsEffectSuite:
  
  // Test ที่ return IO[Unit]
  test("IO should compute value"):
    val io = IO.pure(42)
    assertIO(io, 42)
  
  test("IO should sequence operations"):
    val io = for
      a <- IO.pure(10)
      b <- IO.pure(20)
      c = a + b
    yield c
    assertIO(io, 30)
  
  test("IO should handle side effects"):
    var sideEffect = 0
    val io = IO { sideEffect += 1 } >> IO { sideEffect += 1 }
    io.map: _ =>
      assertEquals(sideEffect, 2)
  
  test("IO should handle errors"):
    val failing = IO.raiseError[Int](new RuntimeException("test error"))
    val recovered = failing.handleError(_ => -1)
    assertIO(recovered, -1)
  
  test("IO.defer should be lazy"):
    var count = 0
    val io = IO.defer { count += 1; IO.pure(count) }
    // Not yet executed
    assertEquals(count, 0)
    
    for
      result <- io
      _ = assertEquals(result, 1)
      _ = assertEquals(count, 1)
    yield ()
  
  // Test ที่ใช้ Resources
  test("Resource should be acquired and released"):
    var acquired = false
    var released = false
    
    val resource = Resource.make(
      IO { acquired = true; "resource" }
    )(r => IO { released = true })
    
    resource.use: r =>
      IO:
        assertEquals(acquired, true)
        assertEquals(released, false)
        assertEquals(r, "resource")
    .flatMap: _ =>
      IO(assertEquals(released, true))
```

### AsyncFreeSpec กับ ScalaTest

```scala
class AsyncIOSpecExample extends AsyncFreeSpec with AsyncIOSpec with Matchers:
  
  "IO" - {
    "should handle async operations" in {
      val io = IO.async_[String]: cb =>
        // Simulate async operation
        cb(Right("async result"))
      
      io.asserting(_ shouldBe "async result")
    }
    
    "should work with flatMap" in {
      val io = IO.pure(1)
        .flatMap(n => IO.pure(n * 2))
        .flatMap(n => IO.pure(n + 1))
      
      io.asserting(_ shouldBe 3)
    }
    
    "should handle concurrent operations" in {
      import cats.effect.syntax.all.*
      
      val io1 = IO.sleep(100.milliseconds) >> IO.pure("first")
      val io2 = IO.sleep(50.milliseconds) >> IO.pure("second")
      
      (io1, io2).parTupled.asserting: (r1, r2) =>
        r1 shouldBe "first"
        r2 shouldBe "second"
    }
    
    "should compose multiple effects" in {
      def fetchUser(id: Int): IO[String] =
        if id > 0 then IO.pure(s"User$id")
        else IO.raiseError(new IllegalArgumentException("Invalid ID"))
      
      val result = for
        u1 <- fetchUser(1)
        u2 <- fetchUser(2)
        u3 <- fetchUser(3)
      yield List(u1, u2, u3)
      
      result.asserting(_ shouldBe List("User1", "User2", "User3"))
    }
  }
```

### Testing Fibers

```scala
class FiberTestSuite extends CatsEffectSuite:
  
  test("fiber should cancel"):
    import scala.concurrent.duration.*
    
    for
      fiber  <- IO.never.start
      _      <- fiber.cancel
      result <- fiber.join
    yield
      result match
        case Outcome.Canceled() => () // ✅ Expected
        case other              => fail(s"Expected cancelled, got: $other")
  
  test("fiber should complete with value"):
    for
      fiber  <- IO.pure(42).start
      result <- fiber.join
    yield
      result match
        case Outcome.Succeeded(io) => assertIO(io, 42)
        case other                 => fail(s"Expected success, got: $other")
  
  test("parallel execution should be faster"):
    import scala.concurrent.duration.*
    
    val slowOp = IO.sleep(100.millis) >> IO.pure(1)
    val start  = System.currentTimeMillis()
    
    for
      (r1, r2) <- (slowOp, slowOp).parTupled
      elapsed  = System.currentTimeMillis() - start
      _ = assert(elapsed < 180, s"Should run in parallel but took ${elapsed}ms")
    yield
      assertEquals(r1 + r2, 2)
```

---

## Mocking IO Effects

### Trait-Based Mocking

```scala
package com.example.test

import cats.effect.*
import munit.CatsEffectSuite

// =============================================================
// Production code
// =============================================================

trait UserRepository[F[_]]:
  def findById(id: String): F[Option[User]]
  def save(user: User): F[User]
  def delete(id: String): F[Boolean]

trait EmailService[F[_]]:
  def sendWelcomeEmail(email: String, name: String): F[Unit]
  def sendPasswordReset(email: String, token: String): F[Unit]

case class User(id: String, name: String, email: String)
case class CreateUserRequest(name: String, email: String)

class UserService[F[_]: Monad](
  userRepo: UserRepository[F],
  emailSvc: EmailService[F]
)(using F: MonadError[F, Throwable]):
  
  def createUser(req: CreateUserRequest): F[User] =
    val user = User(
      id = java.util.UUID.randomUUID().toString,
      name = req.name,
      email = req.email
    )
    for
      saved <- userRepo.save(user)
      _     <- emailSvc.sendWelcomeEmail(saved.email, saved.name)
    yield saved
  
  def getUser(id: String): F[User] =
    userRepo.findById(id).flatMap:
      case Some(user) => F.pure(user)
      case None       => F.raiseError(new NoSuchElementException(s"User $id not found"))
  
  def deleteUser(id: String): F[Unit] =
    userRepo.delete(id).flatMap:
      case true  => F.pure(())
      case false => F.raiseError(new NoSuchElementException(s"User $id not found"))

// =============================================================
// Test implementations (Manual mocks)
// =============================================================

class InMemoryUserRepository extends UserRepository[IO]:
  private var users: Map[String, User] = Map.empty
  
  def findById(id: String): IO[Option[User]] =
    IO.pure(users.get(id))
  
  def save(user: User): IO[User] =
    IO { users = users + (user.id -> user); user }
  
  def delete(id: String): IO[Boolean] =
    IO { 
      val exists = users.contains(id)
      users = users - id
      exists
    }
  
  def all: IO[List[User]] = IO.pure(users.values.toList)

class SpyEmailService extends EmailService[IO]:
  private var sentEmails: List[(String, String, String)] = List.empty
  
  def sendWelcomeEmail(email: String, name: String): IO[Unit] =
    IO { sentEmails = sentEmails :+ ("welcome", email, name) }
  
  def sendPasswordReset(email: String, token: String): IO[Unit] =
    IO { sentEmails = sentEmails :+ ("reset", email, token) }
  
  def getSentEmails: List[(String, String, String)] = sentEmails
  def welcomeEmailsSent: Int = sentEmails.count(_._1 == "welcome")
  def resetEmailsSent: Int = sentEmails.count(_._1 == "reset")

// =============================================================
// Tests using manual mocks
// =============================================================

class UserServiceSuite extends CatsEffectSuite:
  
  test("createUser should save user and send welcome email"):
    val repo     = new InMemoryUserRepository()
    val emailSpy = new SpyEmailService()
    val service  = UserService[IO](repo, emailSpy)
    
    for
      user <- service.createUser(CreateUserRequest("Alice", "alice@example.com"))
      _    = assertEquals(user.name, "Alice")
      _    = assertEquals(user.email, "alice@example.com")
      _    = assertEquals(emailSpy.welcomeEmailsSent, 1)
      savedUser <- repo.findById(user.id)
      _ = assertEquals(savedUser, Some(user))
    yield ()
  
  test("getUser should return user if exists"):
    val repo    = new InMemoryUserRepository()
    val emailSvc = new SpyEmailService()
    val service = UserService[IO](repo, emailSvc)
    
    for
      created <- service.createUser(CreateUserRequest("Bob", "bob@example.com"))
      found   <- service.getUser(created.id)
      _ = assertEquals(found.name, "Bob")
    yield ()
  
  test("getUser should raise error for unknown user"):
    val repo    = new InMemoryUserRepository()
    val emailSvc = new SpyEmailService()
    val service = UserService[IO](repo, emailSvc)
    
    val result = service.getUser("nonexistent-id").attempt
    assertIO(result.map(_.isLeft), true)
  
  test("deleteUser should remove user"):
    val repo    = new InMemoryUserRepository()
    val emailSvc = new SpyEmailService()
    val service = UserService[IO](repo, emailSvc)
    
    for
      user   <- service.createUser(CreateUserRequest("Charlie", "charlie@example.com"))
      _      <- service.deleteUser(user.id)
      found  <- repo.findById(user.id)
      _ = assertEquals(found, None)
    yield ()
  
  test("createUser should fail if email service fails"):
    val repo = new InMemoryUserRepository()
    val failingEmailSvc = new EmailService[IO]:
      def sendWelcomeEmail(email: String, name: String): IO[Unit] =
        IO.raiseError(new RuntimeException("Email service unavailable"))
      def sendPasswordReset(email: String, token: String): IO[Unit] =
        IO.raiseError(new RuntimeException("Email service unavailable"))
    
    val service = UserService[IO](repo, failingEmailSvc)
    
    val result = service.createUser(CreateUserRequest("Dave", "dave@example.com")).attempt
    assertIO(result.map(_.isLeft), true)
```

### Ref-Based State Mocking

```scala
class RefBasedMockSuite extends CatsEffectSuite:
  
  test("should count invocations with Ref"):
    for
      callCount <- Ref.of[IO, Int](0)
      
      mockRepo = new UserRepository[IO]:
        def findById(id: String): IO[Option[User]] =
          callCount.update(_ + 1) >> IO.pure(Some(User(id, "Test", "test@example.com")))
        
        def save(user: User): IO[User] =
          callCount.update(_ + 1) >> IO.pure(user)
        
        def delete(id: String): IO[Boolean] =
          callCount.update(_ + 1) >> IO.pure(true)
      
      _ <- mockRepo.findById("123")
      _ <- mockRepo.findById("456")
      count <- callCount.get
      _ = assertEquals(count, 2)
    yield ()
  
  test("should capture all calls with Ref[List]"):
    for
      calls <- Ref.of[IO, List[String]](List.empty)
      
      mockSvc = new EmailService[IO]:
        def sendWelcomeEmail(email: String, name: String): IO[Unit] =
          calls.update(_ :+ s"welcome:$email")
        
        def sendPasswordReset(email: String, token: String): IO[Unit] =
          calls.update(_ :+ s"reset:$email")
      
      _ <- mockSvc.sendWelcomeEmail("alice@example.com", "Alice")
      _ <- mockSvc.sendPasswordReset("bob@example.com", "token123")
      allCalls <- calls.get
      _ = assertEquals(allCalls.length, 2)
      _ = assert(allCalls.exists(_.startsWith("welcome:")))
      _ = assert(allCalls.exists(_.startsWith("reset:")))
    yield ()
```

---

## TestClock และ TestConsole

### TestClock

```scala
package com.example.test

import cats.effect.*
import cats.effect.testkit.*
import munit.CatsEffectSuite
import scala.concurrent.duration.*

class TestClockSuite extends CatsEffectSuite:
  
  test("should advance time with TestClock"):
    TestControl.executeEmbed:
      for
        start  <- Clock[IO].realTime
        _      <- IO.sleep(1.hour)
        end    <- Clock[IO].realTime
        elapsed = end - start
        _ = assertEquals(elapsed.toSeconds, 3600L)
      yield ()
  
  test("should test time-dependent logic"):
    // ระบบที่ expire session หลังจาก 30 นาที
    def isSessionExpired(createdAt: FiniteDuration, now: FiniteDuration): Boolean =
      (now - createdAt) >= 30.minutes
    
    TestControl.executeEmbed:
      for
        createdAt <- Clock[IO].realTime
        
        // ตรวจสอบทันที - ยังไม่หมดอายุ
        now1      <- Clock[IO].realTime
        _         = assert(!isSessionExpired(createdAt, now1))
        
        // รอ 29 นาที - ยังไม่หมดอายุ
        _ <- IO.sleep(29.minutes)
        now2 <- Clock[IO].realTime
        _ = assert(!isSessionExpired(createdAt, now2))
        
        // รอเพิ่มอีก 2 นาที - หมดอายุแล้ว
        _ <- IO.sleep(2.minutes)
        now3 <- Clock[IO].realTime
        _ = assert(isSessionExpired(createdAt, now3))
      yield ()
  
  test("should test retry with exponential backoff"):
    import cats.effect.std.Queue
    
    TestControl.executeEmbed:
      for
        attempts <- Ref.of[IO, Int](0)
        
        retryWithBackoff = {
          def attempt(): IO[String] =
            attempts.updateAndGet(_ + 1).flatMap: n =>
              if n >= 3 then IO.pure("success")
              else IO.raiseError(new RuntimeException(s"Attempt $n failed"))
          
          def retry(n: Int, delay: FiniteDuration): IO[String] =
            attempt().handleErrorWith: _ =>
              if n <= 0 then IO.raiseError(new RuntimeException("Max retries"))
              else IO.sleep(delay) >> retry(n - 1, delay * 2)
          
          retry(3, 1.second)
        }
        
        result <- retryWithBackoff
        _  = assertEquals(result, "success")
        n  <- attempts.get
        _  = assertEquals(n, 3)
      yield ()
```

### Deferred และ Ref ใน Tests

```scala
class DeferredRefSuite extends CatsEffectSuite:
  
  test("Deferred should synchronize concurrent operations"):
    for
      deferred <- Deferred[IO, Int]
      
      // Consumer fiber รอค่า
      consumer <- (deferred.get.map(_ * 2)).start
      
      // Producer เขียนค่า
      _ <- deferred.complete(21)
      
      result <- consumer.join.flatMap(_.embedError)
      _ = assertEquals(result, 42)
    yield ()
  
  test("Ref should be safe for concurrent updates"):
    val iterations = 1000
    val increment = (ref: Ref[IO, Int]) => ref.update(_ + 1)
    
    for
      counter <- Ref.of[IO, Int](0)
      _       <- List.fill(iterations)(increment(counter)).sequence.void
      result  <- counter.get
      _ = assertEquals(result, iterations)
    yield ()
  
  test("Semaphore should limit concurrent access"):
    import cats.effect.std.Semaphore
    
    for
      sem     <- Semaphore[IO](3) // สูงสุด 3 concurrent
      active  <- Ref.of[IO, Int](0)
      maxSeen <- Ref.of[IO, Int](0)
      
      task = sem.permit.surround:
        active.updateAndGet(_ + 1).flatMap: current =>
          maxSeen.update(math.max(_, current)) >>
          IO.sleep(10.milliseconds) >>
          active.update(_ - 1)
      
      _ <- List.fill(10)(task).parSequence
      max <- maxSeen.get
      _ = assert(max <= 3, s"Max concurrent should be ≤ 3, but was $max")
    yield ()
```

---

## Property-Based Testing กับ IO

### ScalaCheck กับ Cats Effect

```scala
package com.example.test

import cats.effect.*
import munit.CatsEffectSuite
import org.scalacheck.{Gen, Prop, Properties}
import org.scalacheck.effect.PropF
import cats.syntax.all.*

class PropertyBasedSuite extends CatsEffectSuite:
  
  // Generator สำหรับ User
  val userGen: Gen[CreateUserRequest] = for
    name  <- Gen.alphaStr.suchThat(_.nonEmpty)
    email <- Gen.alphaStr.suchThat(_.nonEmpty).map(_ + "@example.com")
  yield CreateUserRequest(name, email)
  
  test("createUser should always succeed with valid input"):
    PropF.forAllF(userGen): req =>
      val repo    = new InMemoryUserRepository()
      val emailSvc = new SpyEmailService()
      val service = UserService[IO](repo, emailSvc)
      
      for
        user  <- service.createUser(req)
        found <- repo.findById(user.id)
      yield
        assertEquals(user.name, req.name)
        assertEquals(user.email, req.email)
        assertEquals(found, Some(user))
  
  test("user ID should be unique for each creation"):
    val requests = Gen.listOfN(100, userGen)
    
    PropF.forAllF(requests): reqs =>
      val repo    = new InMemoryUserRepository()
      val emailSvc = new SpyEmailService()
      val service = UserService[IO](repo, emailSvc)
      
      reqs.traverse(service.createUser).map: users =>
        val ids = users.map(_.id)
        assertEquals(ids.distinct.length, ids.length, "All IDs should be unique")
  
  test("delete then get should fail"):
    PropF.forAllF(userGen): req =>
      val repo    = new InMemoryUserRepository()
      val emailSvc = new SpyEmailService()
      val service = UserService[IO](repo, emailSvc)
      
      for
        user   <- service.createUser(req)
        _      <- service.deleteUser(user.id)
        result <- service.getUser(user.id).attempt
      yield
        assert(result.isLeft, "Should fail after deletion")

// Property ที่ซับซ้อนขึ้น
class ListOperationsSuite extends CatsEffectSuite:
  
  test("parallel map should give same result as sequential"):
    val gen = Gen.listOfN(20, Gen.choose(1, 100))
    
    PropF.forAllF(gen): numbers =>
      val transform = (n: Int) => IO(n * 2 + 1)
      
      for
        sequential <- numbers.traverse(transform)
        parallel   <- numbers.parTraverse(transform)
      yield
        assertEquals(sequential.toSet, parallel.toSet)
  
  test("IO operations should be referentially transparent"):
    PropF.forAllF(Gen.choose(1, 1000)): n =>
      val computation = IO.pure(n).map(_ * 2)
      
      for
        result1 <- computation
        result2 <- computation
      yield
        assertEquals(result1, result2)
  
  test("error handling should be consistent"):
    val gen = Gen.either(Gen.alphaStr, Gen.alphaStr)
    
    PropF.forAllF(gen): input =>
      val io: IO[String] = input match
        case Right(s) => IO.pure(s)
        case Left(e)  => IO.raiseError(new RuntimeException(e))
      
      io.attempt.map: result =>
        assertEquals(result.isRight, input.isRight)
```

---

## Integration Tests กับ Resource

### Database Integration Tests

```scala
package com.example.test

import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import munit.CatsEffectSuite
import org.scalatest.matchers.should.Matchers

// =============================================================
// Test Database เชื่อมต่อ
// =============================================================

case class DbConfig(url: String, user: String, password: String)

class TestDatabase(config: DbConfig):
  
  def setup(): IO[Unit] =
    IO.println(s"Setting up test database at ${config.url}")
  
  def teardown(): IO[Unit] =
    IO.println("Tearing down test database")
  
  def executeQuery(sql: String): IO[List[Map[String, String]]] =
    IO.pure(List.empty) // Mock implementation
  
  def execute(sql: String): IO[Int] =
    IO.pure(0) // Mock implementation

object TestDatabase:
  
  def resource(config: DbConfig): Resource[IO, TestDatabase] =
    Resource.make(
      IO { new TestDatabase(config) }.flatTap(_.setup())
    )(db => db.teardown())
  
  def withTestDb[A](test: TestDatabase => IO[A]): IO[A] =
    val config = DbConfig(
      url      = "jdbc:postgresql://localhost:5432/test_db",
      user     = "test_user",
      password = "test_pass"
    )
    resource(config).use(test)

// =============================================================
// Integration Test Suite
// =============================================================

class UserRepositoryIntegrationSuite extends CatsEffectSuite:
  
  // Shared resource สำหรับทุก tests ใน suite
  val dbFixture = ResourceFunFixture(
    TestDatabase.resource(DbConfig(
      url      = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1",
      user     = "sa",
      password = ""
    ))
  )
  
  dbFixture.test("should create and retrieve user"):
    db =>
      for
        // Setup test data
        _ <- db.execute("INSERT INTO users VALUES ('1', 'Alice', 'alice@example.com')")
        
        // Query
        rows <- db.executeQuery("SELECT * FROM users WHERE id = '1'")
        
        // Assertions
        _ = assert(rows.nonEmpty)
      yield ()
  
  dbFixture.test("should handle concurrent writes"):
    db =>
      val writes = List.fill(10)(
        db.execute("INSERT INTO users VALUES (gen_random_uuid(), 'User', 'user@example.com')")
      )
      
      writes.parSequence.map: results =>
        assertEquals(results.length, 10)

// =============================================================
// HTTP Integration Tests
// =============================================================

import org.http4s.*
import org.http4s.client.*
import org.http4s.ember.client.*
import org.http4s.circe.*
import io.circe.generic.auto.*

class HttpIntegrationSuite extends CatsEffectSuite:
  
  val httpClientFixture = ResourceFunFixture(
    EmberClientBuilder.default[IO].build
  )
  
  httpClientFixture.test("should make HTTP requests"):
    client =>
      val uri = uri"https://httpbin.org/get"
      
      client.expect[String](uri).map: response =>
        assert(response.nonEmpty)

// =============================================================
// Full Integration Test กับ Layered Resources
// =============================================================

case class AppContext(
  db: TestDatabase,
  emailSvc: SpyEmailService,
  userRepo: UserRepository[IO],
  userSvc: UserService[IO]
)

object AppContext:
  
  def resource(): Resource[IO, AppContext] =
    for
      db       <- TestDatabase.resource(DbConfig("test", "test", "test"))
      emailSpy = new SpyEmailService()
      userRepo = new InMemoryUserRepository()
      userSvc  = UserService[IO](userRepo, emailSpy)
    yield AppContext(db, emailSpy, userRepo, userSvc)

class IntegrationSuite extends CatsEffectSuite:
  
  val appCtxFixture = ResourceFunFixture(AppContext.resource())
  
  appCtxFixture.test("full user flow should work"):
    ctx =>
      for
        // Create user
        user1 <- ctx.userSvc.createUser(CreateUserRequest("Alice", "alice@example.com"))
        user2 <- ctx.userSvc.createUser(CreateUserRequest("Bob", "bob@example.com"))
        
        // Verify welcome emails sent
        _ = assertEquals(ctx.emailSvc.welcomeEmailsSent, 2)
        
        // Get users
        found1 <- ctx.userSvc.getUser(user1.id)
        found2 <- ctx.userSvc.getUser(user2.id)
        
        _ = assertEquals(found1.name, "Alice")
        _ = assertEquals(found2.name, "Bob")
        
        // Delete user
        _ <- ctx.userSvc.deleteUser(user1.id)
        
        // Verify deletion
        deleteResult <- ctx.userSvc.getUser(user1.id).attempt
        _ = assert(deleteResult.isLeft)
        
        // Other user still exists
        stillExists <- ctx.userSvc.getUser(user2.id)
        _ = assertEquals(stillExists.name, "Bob")
      yield ()
```

---

## Complete Test Suite Example

### E-Commerce Test Suite

```scala
package com.example.test.ecommerce

import cats.effect.*
import cats.syntax.all.*
import munit.CatsEffectSuite
import org.scalacheck.Gen
import org.scalacheck.effect.PropF

// =============================================================
// Domain Models
// =============================================================

case class Product(id: String, name: String, price: BigDecimal, stock: Int)
case class CartItem(product: Product, quantity: Int)
case class Cart(userId: String, items: List[CartItem]):
  def total: BigDecimal = items.map(i => i.product.price * i.quantity).sum
  def isEmpty: Boolean = items.isEmpty

case class Order(id: String, userId: String, items: List[CartItem], total: BigDecimal)

// =============================================================
// Services
// =============================================================

trait ProductService[F[_]]:
  def findById(id: String): F[Option[Product]]
  def updateStock(id: String, delta: Int): F[Either[String, Product]]
  def listAll(): F[List[Product]]

trait CartService[F[_]]:
  def getCart(userId: String): F[Cart]
  def addItem(userId: String, productId: String, qty: Int): F[Either[String, Cart]]
  def removeItem(userId: String, productId: String): F[Cart]
  def clearCart(userId: String): F[Unit]

trait OrderService[F[_]]:
  def checkout(cart: Cart): F[Either[String, Order]]
  def getOrder(orderId: String): F[Option[Order]]

// =============================================================
// In-Memory Implementations for Testing
// =============================================================

class InMemoryProductService(initial: List[Product]) extends ProductService[IO]:
  private val state: Ref[IO, Map[String, Product]] =
    Ref.unsafe(initial.map(p => p.id -> p).toMap)
  
  def findById(id: String): IO[Option[Product]] =
    state.get.map(_.get(id))
  
  def updateStock(id: String, delta: Int): IO[Either[String, Product]] =
    state.modify: products =>
      products.get(id) match
        case None =>
          (products, Left(s"Product $id not found"))
        case Some(p) =>
          val newStock = p.stock + delta
          if newStock < 0 then
            (products, Left(s"Insufficient stock for ${p.name}"))
          else
            val updated = p.copy(stock = newStock)
            (products.updated(id, updated), Right(updated))
  
  def listAll(): IO[List[Product]] =
    state.get.map(_.values.toList)

class InMemoryCartService extends CartService[IO]:
  private val carts: Ref[IO, Map[String, Cart]] = Ref.unsafe(Map.empty)
  
  def getCart(userId: String): IO[Cart] =
    carts.get.map(_.getOrElse(userId, Cart(userId, List.empty)))
  
  def addItem(userId: String, productId: String, qty: Int): IO[Either[String, Cart]] =
    // Simplified - always succeeds
    carts.modify: state =>
      val current = state.getOrElse(userId, Cart(userId, List.empty))
      val mockProduct = Product(productId, s"Product $productId", 10.0, 100)
      val item = CartItem(mockProduct, qty)
      val updated = current.copy(items = current.items :+ item)
      (state.updated(userId, updated), Right(updated))
  
  def removeItem(userId: String, productId: String): IO[Cart] =
    carts.modify: state =>
      val current = state.getOrElse(userId, Cart(userId, List.empty))
      val updated = current.copy(items = current.items.filterNot(_.product.id == productId))
      (state.updated(userId, updated), updated)
  
  def clearCart(userId: String): IO[Unit] =
    carts.update(_ - userId)

class InMemoryOrderService(productSvc: ProductService[IO]) extends OrderService[IO]:
  private val orders: Ref[IO, Map[String, Order]] = Ref.unsafe(Map.empty)
  
  def checkout(cart: Cart): IO[Either[String, Order]] =
    if cart.isEmpty then
      IO.pure(Left("Cannot checkout empty cart"))
    else
      val order = Order(
        id = java.util.UUID.randomUUID().toString,
        userId = cart.userId,
        items = cart.items,
        total = cart.total
      )
      orders.update(_ + (order.id -> order)).as(Right(order))
  
  def getOrder(orderId: String): IO[Option[Order]] =
    orders.get.map(_.get(orderId))

// =============================================================
// Test Suites
// =============================================================

class ProductServiceSuite extends CatsEffectSuite:
  
  val testProducts = List(
    Product("p1", "Laptop", 999.99, 10),
    Product("p2", "Mouse", 29.99, 50),
    Product("p3", "Keyboard", 79.99, 30)
  )
  
  def makeProductService() = new InMemoryProductService(testProducts)
  
  test("findById should return product if exists"):
    val svc = makeProductService()
    assertIO(svc.findById("p1").map(_.map(_.name)), Some("Laptop"))
  
  test("findById should return None for unknown product"):
    val svc = makeProductService()
    assertIO(svc.findById("unknown"), None)
  
  test("updateStock should decrease stock"):
    val svc = makeProductService()
    for
      result <- svc.updateStock("p1", -3)
      _ = result match
        case Right(p) => assertEquals(p.stock, 7)
        case Left(e)  => fail(e)
    yield ()
  
  test("updateStock should fail if insufficient stock"):
    val svc = makeProductService()
    for
      result <- svc.updateStock("p2", -100)  // Only 50 in stock
      _ = assert(result.isLeft)
    yield ()
  
  test("listAll should return all products"):
    val svc = makeProductService()
    svc.listAll().map: products =>
      assertEquals(products.length, 3)

class CartServiceSuite extends CatsEffectSuite:
  
  test("getCart should return empty cart for new user"):
    val svc = new InMemoryCartService()
    for
      cart <- svc.getCart("user1")
      _ = assert(cart.isEmpty)
    yield ()
  
  test("addItem should add product to cart"):
    val svc = new InMemoryCartService()
    for
      result <- svc.addItem("user1", "p1", 2)
      _ = result match
        case Right(cart) => assertEquals(cart.items.length, 1)
        case Left(e)     => fail(e)
    yield ()
  
  test("removeItem should remove product from cart"):
    val svc = new InMemoryCartService()
    for
      _    <- svc.addItem("user1", "p1", 1)
      _    <- svc.addItem("user1", "p2", 1)
      cart <- svc.removeItem("user1", "p1")
      _ = assertEquals(cart.items.length, 1)
      _ = assert(cart.items.forall(_.product.id != "p1"))
    yield ()

class CheckoutSuite extends CatsEffectSuite:
  
  test("checkout should create order from cart"):
    val productSvc = new InMemoryProductService(List(
      Product("p1", "Laptop", 999.99, 10)
    ))
    val cartSvc  = new InMemoryCartService()
    val orderSvc = new InMemoryOrderService(productSvc)
    
    for
      _      <- cartSvc.addItem("user1", "p1", 2)
      cart   <- cartSvc.getCart("user1")
      result <- orderSvc.checkout(cart)
      _ = result match
        case Right(order) =>
          assertEquals(order.userId, "user1")
          assertEquals(order.items.length, 1)
        case Left(e) => fail(s"Checkout failed: $e")
    yield ()
  
  test("checkout should fail with empty cart"):
    val productSvc = new InMemoryProductService(List.empty)
    val orderSvc = new InMemoryOrderService(productSvc)
    val emptyCart = Cart("user1", List.empty)
    
    orderSvc.checkout(emptyCart).map: result =>
      assert(result.isLeft)
  
  test("property: checkout total should equal cart total"):
    val gen = for
      price <- Gen.choose(1.0, 1000.0)
      qty   <- Gen.choose(1, 10)
    yield (price, qty)
    
    PropF.forAllF(gen): (price, qty) =>
      val product = Product("p1", "Test Product", BigDecimal(price), 100)
      val productSvc = new InMemoryProductService(List(product))
      val cartSvc  = new InMemoryCartService()
      val orderSvc = new InMemoryOrderService(productSvc)
      
      for
        _      <- cartSvc.addItem("user1", "p1", qty)
        cart   <- cartSvc.getCart("user1")
        result <- orderSvc.checkout(cart)
      yield
        result match
          case Right(order) =>
            val expectedTotal = BigDecimal(price) * qty
            assertEquals(order.total, expectedTotal)
          case Left(e) => fail(s"Checkout failed: $e")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **cats-effect-testing**: Library สำหรับ test code ที่ใช้ Cats Effect
2. **CatsEffectSuite**: MUnit-based test suite สำหรับ IO operations
3. **AsyncIOSpec**: ScalaTest integration สำหรับ async tests
4. **Manual Mocks**: สร้าง in-memory implementations สำหรับ testing
5. **Ref-based Mocks**: ใช้ Ref สำหรับ stateful mocks
6. **TestClock**: Control time ใน tests
7. **Property-Based Testing**: ใช้ PropF สำหรับ property tests กับ IO
8. **Integration Tests**: ใช้ Resource สำหรับ test infrastructure
9. **Complete Suite**: ระบบ E-commerce ที่ test ได้ทุก layer

### Best Practices

- **Test ด้วย IO** แทน Future เพื่อควบคุม effects ได้ดีขึ้น
- **ใช้ Resource** สำหรับ test infrastructure เพื่อ cleanup อัตโนมัติ
- **Ref สำหรับ State**: ใช้ `Ref` แทน `var` ใน test fixtures
- **Property Tests**: ใช้ PropF ตรวจสอบ invariants
- **Test Layers**: แยก unit tests, integration tests และ e2e tests

---

*[← ตอนที่ 86: Type-Driven Development](part-86-type-driven.md) | [ตอนที่ 88: Web Scraping →](part-88-web-scraping.md)*
