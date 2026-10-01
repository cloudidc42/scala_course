# Part 47: Testing Strategies

## สารบัญ
1. [Testing Philosophy](#testing-philosophy)
2. [Unit Testing Patterns](#unit-testing)
3. [Integration Testing](#integration-testing)
4. [Property-Based Testing](#property-based)
5. [Contract Testing](#contract-testing)
6. [Performance Testing](#performance-testing)

---

## Testing Philosophy

### Testing Pyramid

```
Testing Pyramid:
         /\
        /  \
       / E2E\ ← few, expensive, slow
      /------\
     / Integration \ ← moderate, test boundaries
    /--------------\
   /    Unit Tests   \ ← many, cheap, fast
  /------------------\

Scala Testing Ecosystem:
- ScalaTest:  most popular, multiple styles
- ScalaCheck: property-based testing
- Weaver:     Cats Effect compatible
- MUnit:      lightweight, fast
- Munit-Cats-Effect: MUnit + IO
```

---

## Unit Testing Patterns

### Pure Function Testing

```scala
import org.scalatest.funspec.AnyFunSpec
import org.scalatest.matchers.should.Matchers

// Testing pure functions: easy, deterministic
class MoneySpec extends AnyFunSpec with Matchers:
  case class Money(amount: BigDecimal, currency: String):
    def +(other: Money): Money =
      require(currency == other.currency)
      Money(amount + other.amount, currency)

  val usd = (n: BigDecimal) => Money(n, "USD")
  val thb = (n: BigDecimal) => Money(n, "THB")

  describe("Money"):
    describe("+"):
      it("adds two amounts of same currency") {
        val total = usd(100) + usd(50)
        total.amount shouldEqual BigDecimal(150)
      }

      it("fails for different currencies") {
        assertThrows[IllegalArgumentException] {
          usd(100) + thb(200)
        }
      }

    describe("properties"):
      it("is commutative") {
        val a = usd(30)
        val b = usd(70)
        a + b shouldEqual b + a
      }

      it("has identity element zero") {
        val m = usd(100)
        m + usd(0) shouldEqual m
      }

// Testing ADTs
class ShapeSpec extends AnyFunSpec with Matchers:
  sealed trait Shape
  case class Circle(radius: Double) extends Shape
  case class Rectangle(w: Double, h: Double) extends Shape

  def area(s: Shape): Double = s match
    case Circle(r)    => math.Pi * r * r
    case Rectangle(w, h) => w * h

  def perimeter(s: Shape): Double = s match
    case Circle(r)    => 2 * math.Pi * r
    case Rectangle(w, h) => 2 * (w + h)

  describe("area"):
    it("computes circle area") {
      area(Circle(5.0)) shouldEqual math.Pi * 25.0 +- 0.001
    }

    it("computes rectangle area") {
      area(Rectangle(4.0, 6.0)) shouldEqual 24.0
    }
```

### Testing with Mocks

```scala
import org.scalatest.funspec.AnyFunSpec
import org.scalatest.matchers.should.Matchers
import org.scalatestplus.mockito.MockitoSugar
import org.mockito.Mockito.*
import org.mockito.ArgumentMatchers.*
import cats.effect.IO
import cats.effect.unsafe.implicits.global

class UserServiceSpec extends AnyFunSpec with Matchers with MockitoSugar:
  trait UserRepository:
    def findById(id: Long): IO[Option[String]]
    def save(name: String): IO[Long]

  class UserService(repo: UserRepository):
    def register(name: String): IO[Either[String, Long]] =
      if name.trim.isEmpty then IO.pure(Left("Name cannot be empty"))
      else repo.save(name).map(Right(_))

    def getOrCreate(id: Long, defaultName: String): IO[String] =
      repo.findById(id).flatMap {
        case Some(user) => IO.pure(user)
        case None       => repo.save(defaultName).as(defaultName)
      }

  describe("UserService"):
    describe("register"):
      it("saves user with valid name") {
        val repo = mock[UserRepository]
        when(repo.save(any[String])).thenReturn(IO.pure(42L))

        val svc = UserService(repo)
        val result = svc.register("Alice").unsafeRunSync()

        result shouldEqual Right(42L)
        verify(repo).save("Alice")
      }

      it("rejects empty name without calling repository") {
        val repo = mock[UserRepository]
        val svc = UserService(repo)

        val result = svc.register("").unsafeRunSync()

        result shouldEqual Left("Name cannot be empty")
        verify(repo, never()).save(any())
      }
```

---

## Integration Testing

### Database Integration Tests

```scala
import cats.effect.IO
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.wordspec.AsyncWordSpec
import doobie.*
import doobie.implicits.*

class UserRepositoryIntegrationSpec extends AsyncWordSpec with AsyncIOSpec with Matchers:
  // Use testcontainers for real DB
  // testcontainers-scala: "com.dimafeng" %% "testcontainers-scala" % "0.41.0"
  import com.dimafeng.testcontainers.PostgreSQLContainer
  import com.dimafeng.testcontainers.scalatest.TestContainerForAll
  import org.testcontainers.utility.DockerImageName

  // In a real test, you'd extend TestContainerForAll
  // Here we show the pattern with H2

  lazy val xa = Transactor.fromDriverManager[IO](
    driver   = "org.h2.Driver",
    url      = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;MODE=PostgreSQL",
    user     = "sa",
    password = ""
  )

  val setup: ConnectionIO[Unit] = for
    _ <- sql"""
      CREATE TABLE IF NOT EXISTS users (
        id BIGINT AUTO_INCREMENT PRIMARY KEY,
        name VARCHAR(255) NOT NULL,
        email VARCHAR(255) UNIQUE NOT NULL
      )
    """.update.run
  yield ()

  def runSetup: IO[Unit] = setup.transact(xa)

  "UserRepository" should {
    "insert and retrieve user" in {
      val test = for
        _  <- runSetup
        id <- sql"INSERT INTO users (name, email) VALUES ('Alice', 'alice@test.com')"
                .update.withUniqueGeneratedKeys[Long]("id").transact(xa)
        u  <- sql"SELECT name FROM users WHERE id = $id"
                .query[String].unique.transact(xa)
      yield u

      test.asserting(_ shouldEqual "Alice")
    }
  }

// Testcontainers example (for real DB)
/*
class PostgresIntegrationSpec
    extends AsyncWordSpec
    with AsyncIOSpec
    with Matchers
    with TestContainerForAll:

  override val containerDef = PostgreSQLContainer.Def(
    dockerImageName = DockerImageName.parse("postgres:16-alpine"),
    databaseName    = "testdb",
    username        = "test",
    password        = "test"
  )

  "Database" should {
    "be reachable" in withContainers { pg =>
      val xa = Transactor.fromDriverManager[IO](
        "org.postgresql.Driver",
        pg.jdbcUrl, pg.username, pg.password
      )
      sql"SELECT 1".query[Int].unique.transact(xa)
        .asserting(_ shouldEqual 1)
    }
  }
*/
```

---

## Property-Based Testing

### ScalaCheck Advanced

```scala
import org.scalatest.propspec.AnyPropSpec
import org.scalatest.matchers.should.Matchers
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks
import org.scalacheck.{Gen, Arbitrary, Prop}

class SortingPropertiesSpec extends AnyPropSpec with ScalaCheckPropertyChecks with Matchers:
  // Custom generators
  val sortedListGen: Gen[List[Int]] =
    Gen.listOf(Gen.choose(-1000, 1000)).map(_.sorted)

  val nonEmptyListGen: Gen[List[Int]] =
    Gen.nonEmptyListOf(Gen.choose(0, 100))

  // Properties of a correct sort algorithm
  def sort(list: List[Int]): List[Int] = list.sorted  // implementation to test

  property("sorted list has same length as input") {
    forAll { (list: List[Int]) =>
      sort(list).length shouldEqual list.length
    }
  }

  property("sorted list is monotonically non-decreasing") {
    forAll { (list: List[Int]) =>
      val sorted = sort(list)
      sorted.zip(sorted.tail).forall { case (a, b) => a <= b }
    }
  }

  property("sorted list contains same elements as input") {
    forAll { (list: List[Int]) =>
      sort(list).sorted shouldEqual list.sorted
    }
  }

  property("idempotent: sorting a sorted list is the same") {
    forAll { (list: List[Int]) =>
      val once  = sort(list)
      val twice = sort(once)
      once shouldEqual twice
    }
  }

  property("reverse then sort equals sort") {
    forAll { (list: List[Int]) =>
      sort(list.reverse) shouldEqual sort(list)
    }
  }

// Generator composition
val emailGen: Gen[String] =
  for
    user   <- Gen.alphaStr.suchThat(_.nonEmpty)
    domain <- Gen.alphaStr.suchThat(_.nonEmpty)
    tld    <- Gen.oneOf("com", "net", "org", "io")
  yield s"$user@$domain.$tld"

val personGen: Gen[(String, String, Int)] =
  for
    name  <- Gen.alphaStr.suchThat(_.length > 1)
    email <- emailGen
    age   <- Gen.choose(1, 120)
  yield (name, email, age)

// Stateful property testing (command pattern)
object StatefulTest:
  import org.scalacheck.commands.Commands
  import scala.util.Try

  object StackCommands extends Commands:
    type State = List[Int]
    type Sut = scala.collection.mutable.Stack[Int]

    def newSut(state: State): Sut = new scala.collection.mutable.Stack[Int]()
    def destroySut(sut: Sut): Unit = ()
    def initialPreCondition(state: State): Boolean = true
    def genInitialState = Gen.const(List.empty[Int])
    def canCreateNewSut(s: State, i: Iterable[State], d: Iterable[Sut]): Boolean = true

    def genCommand(state: State): Gen[Command] = Gen.oneOf(
      Gen.choose(-100, 100).map(Push),
      Gen.const(Pop)
    )

    case class Push(n: Int) extends UnitCommand:
      def run(sut: Sut): Unit = sut.push(n)
      def nextState(state: State): State = n :: state
      def preCondition(state: State): Boolean = true
      def postCondition(state: State, success: Boolean): Prop = success

    case object Pop extends Command:
      type Result = Option[Int]
      def run(sut: Sut): Option[Int] =
        if sut.isEmpty then None else Some(sut.pop())
      def nextState(state: State): State = if state.isEmpty then state else state.tail
      def preCondition(state: State): Boolean = true
      def postCondition(state: State, result: Try[Option[Int]]): Prop =
        result.map(_ == state.headOption).getOrElse(false)
```

---

## Contract Testing

### Consumer-Driven Contract Testing

```scala
// Using Pact for contract testing
// "au.com.dius.pact.consumer" %% "scalatest" % "4.6.6"

// Consumer test
import au.com.dius.pact.consumer.PactConsumerTestExt
import au.com.dius.pact.consumer.dsl.*
import au.com.dius.pact.consumer.junit5.PactTestFor
import au.com.dius.pact.core.model.RequestResponsePact
import au.com.dius.pact.core.model.annotations.*
import org.scalatest.funspec.AnyFunSpec

class UserApiContractSpec extends AnyFunSpec:
  // Define what the consumer expects from the provider
  @Pact(consumer = "order-service", provider = "user-service")
  def getUserPact(builder: PactDslWithProvider): RequestResponsePact =
    builder
      .given("user 1 exists")
      .uponReceiving("a request to get user 1")
        .path("/users/1")
        .method("GET")
      .willRespondWith()
        .status(200)
        .body(PactDslJsonBody()
          .numberValue("id", 1)
          .stringValue("name", "Alice")
          .stringValue("email", "alice@example.com")
        )
      .toPact()

  // The consumer tests against a mock provider
  // The pact file is then verified against the real provider
```

---

## Performance Testing

### Load Testing with Gatling

```scala
// build.sbt
libraryDependencies += "io.gatling" % "gatling-core" % "3.9.5" % Test
addSbtPlugin("io.gatling" % "gatling-sbt" % "4.7.0")
enablePlugins(GatlingPlugin)

// Gatling simulation
import io.gatling.core.Predef.*
import io.gatling.http.Predef.*
import scala.concurrent.duration.*

class UserApiLoadTest extends Simulation:
  val httpConfig = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")

  val getUser = exec(
    http("Get User")
      .get("/users/1")
      .check(status.is(200))
      .check(jsonPath("$.name").is("Alice"))
  )

  val createUser = exec(
    http("Create User")
      .post("/users")
      .body(StringBody("""{"name":"LoadTest","email":"load@test.com"}"""))
      .check(status.is(201))
  )

  val scenario1 = scenario("Read-heavy load")
    .exec(getUser)
    .pause(100.millis)

  val scenario2 = scenario("Write operations")
    .exec(createUser)
    .pause(500.millis)

  setUp(
    scenario1.inject(
      rampUsersPerSec(1).to(50).during(30.seconds),
      constantUsersPerSec(50).during(60.seconds)
    ),
    scenario2.inject(
      rampUsersPerSec(1).to(10).during(30.seconds),
      constantUsersPerSec(10).during(60.seconds)
    )
  )
  .protocols(httpConfig)
  .assertions(
    global.responseTime.percentile(95).lt(500),   // p95 < 500ms
    global.successfulRequests.percent.gt(99.0),   // 99%+ success
    global.requestsPerSec.gt(50.0)                // 50+ rps
  )
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Unit testing: pure functions, mocking
- ✅ Integration testing: databases, testcontainers
- ✅ Property-based testing: ScalaCheck generators and properties
- ✅ Stateful property testing: command pattern
- ✅ Contract testing: Pact consumer-driven contracts
- ✅ Performance/load testing: Gatling simulations

---

*[← Part 46: Docker](part-46-docker.md) | [Part 48: Real-World Project: E-Commerce →](part-48-ecommerce.md)*
