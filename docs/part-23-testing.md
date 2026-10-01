# Part 23: Testing in Scala

## สารบัญ
1. [ScalaTest](#scalatest)
2. [Property-Based Testing กับ ScalaCheck](#property-based-testing)
3. [Mocking กับ Mockito](#mocking)
4. [Testing Akka Actors](#testing-akka-actors)
5. [Testing Futures](#testing-futures)

---

## ScalaTest

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.scalatest"  %% "scalatest"           % "3.2.17"  % Test,
  "org.scalacheck" %% "scalacheck"          % "1.17.0"  % Test,
  "org.mockito"    %% "mockito-scala"       % "1.17.30" % Test,
  "org.scalamock"  %% "scalamock"           % "5.2.0"   % Test
)
```

### FunSuite Style

```scala
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers

class MathSpec extends AnyFunSuite with Matchers:

  test("addition") {
    val result = 2 + 2
    result shouldEqual 4
    result should be(4)
    result shouldBe 4
  }

  test("string operations") {
    val str = "Hello, World!"
    str should startWith("Hello")
    str should endWith("World!")
    str should include("World")
    str.length shouldBe 13
  }

  test("collections") {
    val list = List(1, 2, 3, 4, 5)
    list should have length 5
    list should contain(3)
    list should contain allOf(1, 3, 5)
    list shouldBe sorted
    list.sum shouldEqual 15
  }

  test("exceptions") {
    an[ArithmeticException] should be thrownBy {
      val x = 1 / 0
    }

    val ex = the[IllegalArgumentException] thrownBy {
      throw new IllegalArgumentException("bad argument")
    }
    ex.getMessage shouldBe "bad argument"
  }
```

### FlatSpec Style (BDD)

```scala
import org.scalatest.flatspec.AnyFlatSpec

class StackSpec extends AnyFlatSpec with Matchers:

  class Stack[A]:
    private var items = List.empty[A]
    def push(a: A): Unit = items = a :: items
    def pop(): Option[A] = items match
      case Nil => None
      case h :: t => items = t; Some(h)
    def peek: Option[A] = items.headOption
    def isEmpty: Boolean = items.isEmpty
    def size: Int = items.length

  "A Stack" should "be empty when created" in {
    val stack = new Stack[Int]()
    stack.isEmpty shouldBe true
    stack.size shouldEqual 0
  }

  it should "push and pop elements" in {
    val stack = new Stack[Int]()
    stack.push(1)
    stack.push(2)
    stack.push(3)
    stack.pop() shouldBe Some(3)
    stack.pop() shouldBe Some(2)
    stack.size shouldBe 1
  }

  it should "return None when popping empty stack" in {
    val stack = new Stack[Int]()
    stack.pop() shouldBe None
  }
```

### WordSpec Style

```scala
import org.scalatest.wordspec.AnyWordSpec

class UserServiceSpec extends AnyWordSpec with Matchers:

  case class User(id: Int, name: String, email: String)

  class UserService:
    private var users = Map[Int, User]()
    def create(name: String, email: String): User =
      val id = users.size + 1
      val user = User(id, name, email)
      users = users.updated(id, user)
      user
    def findById(id: Int): Option[User] = users.get(id)
    def delete(id: Int): Boolean =
      if users.contains(id) then
        users = users.removed(id)
        true
      else false

  "UserService" when {
    "creating a user" should {
      "return the created user" in {
        val service = new UserService()
        val user = service.create("Alice", "alice@example.com")
        user.name shouldBe "Alice"
        user.email shouldBe "alice@example.com"
        user.id should be > 0
      }
    }

    "finding a user" should {
      "return Some(user) when user exists" in {
        val service = new UserService()
        val created = service.create("Bob", "bob@example.com")
        service.findById(created.id) shouldBe Some(created)
      }

      "return None when user doesn't exist" in {
        val service = new UserService()
        service.findById(999) shouldBe None
      }
    }
  }
```

### Shared Fixtures

```scala
import org.scalatest.*
import org.scalatest.flatspec.AnyFlatSpec

trait DatabaseFixture extends BeforeAndAfterEach:
  this: Suite =>

  var db: FakeDatabase = _

  override protected def beforeEach(): Unit =
    db = new FakeDatabase()
    db.connect()

  override protected def afterEach(): Unit =
    db.disconnect()

class FakeDatabase:
  var connected = false
  def connect(): Unit = connected = true
  def disconnect(): Unit = connected = false

class UserRepositorySpec extends AnyFlatSpec with DatabaseFixture with Matchers:
  "UserRepository" should "query users" in {
    db.connected shouldBe true
    // test code using db
  }
```

---

## Property-Based Testing

### ScalaCheck

```scala
import org.scalacheck.*
import org.scalacheck.Prop.*

// Properties
object MathProperties extends Properties("Math"):

  property("addition is commutative") = forAll { (a: Int, b: Int) =>
    a + b == b + a
  }

  property("addition is associative") = forAll { (a: Int, b: Int, c: Int) =>
    (a + b) + c == a + (b + c)
  }

  property("multiplication distributes over addition") = forAll { (a: Int, b: Int, c: Int) =>
    a * (b + c) == (a * b) + (a * c)
  }

// String properties
object StringProperties extends Properties("String"):

  property("reverse twice is identity") = forAll { (s: String) =>
    s.reverse.reverse == s
  }

  property("length after concat") = forAll { (s1: String, s2: String) =>
    (s1 + s2).length == s1.length + s2.length
  }

// Custom generators
val positiveInt: Gen[Int] = Gen.posNum[Int]
val nonEmptyString: Gen[String] = Gen.alphaStr.suchThat(_.nonEmpty)
val emailGen: Gen[String] = for
  user   <- Gen.alphaStr.suchThat(_.nonEmpty)
  domain <- Gen.alphaStr.suchThat(_.nonEmpty)
  tld    <- Gen.oneOf("com", "org", "net")
yield s"$user@$domain.$tld"

property("email contains @") = forAll(emailGen) { email =>
  email.contains("@")
}

// Run properties (in ScalaTest integration)
import org.scalatest.propspec.AnyPropSpec
import org.scalatestplus.scalacheck.ScalaCheckPropertyChecks

class ListSpec extends AnyPropSpec with ScalaCheckPropertyChecks with Matchers:
  property("sorted list is non-decreasing") {
    forAll { (list: List[Int]) =>
      val sorted = list.sorted
      sorted.sliding(2).forall {
        case List(a, b) => a <= b
        case _ => true
      } shouldBe true
    }
  }
```

---

## Mocking

### Mockito Scala

```scala
import org.mockito.MockitoSugar
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

trait UserRepository:
  def findById(id: Int): Option[String]
  def save(name: String): Int
  def delete(id: Int): Boolean

class UserService(repo: UserRepository):
  def getUser(id: Int): String =
    repo.findById(id).getOrElse("Unknown")

  def createUser(name: String): Either[String, Int] =
    if name.isEmpty then Left("Name required")
    else Right(repo.save(name))

class UserServiceSpec extends AnyFlatSpec with Matchers with MockitoSugar:

  "UserService.getUser" should "return user name when found" in {
    val mockRepo = mock[UserRepository]
    when(mockRepo.findById(1)) thenReturn Some("Alice")

    val service = UserService(mockRepo)
    service.getUser(1) shouldBe "Alice"
    verify(mockRepo).findById(1)
  }

  it should "return 'Unknown' when user not found" in {
    val mockRepo = mock[UserRepository]
    when(mockRepo.findById(99)) thenReturn None

    val service = UserService(mockRepo)
    service.getUser(99) shouldBe "Unknown"
  }

  "UserService.createUser" should "return Right(id) for valid name" in {
    val mockRepo = mock[UserRepository]
    when(mockRepo.save("Alice")) thenReturn 1

    val service = UserService(mockRepo)
    service.createUser("Alice") shouldBe Right(1)
    verify(mockRepo).save("Alice")
  }

  it should "return Left(error) for empty name" in {
    val mockRepo = mock[UserRepository]
    val service = UserService(mockRepo)
    service.createUser("") shouldBe Left("Name required")
    verifyNoInteractions(mockRepo)
  }
```

---

## Testing Akka Actors

```scala
import akka.actor.testkit.typed.scaladsl.*
import org.scalatest.flatspec.AnyFlatSpec
import org.scalatest.matchers.should.Matchers

// Simple counter actor for testing
object Counter:
  sealed trait Command
  case object Increment extends Command
  case class GetCount(replyTo: akka.actor.typed.ActorRef[Int]) extends Command

  import akka.actor.typed.Behavior
  import akka.actor.typed.scaladsl.Behaviors
  def apply(count: Int = 0): Behavior[Command] =
    Behaviors.receiveMessage {
      case Increment => apply(count + 1)
      case GetCount(replyTo) =>
        replyTo ! count
        Behaviors.same
    }

class CounterSpec extends AnyFlatSpec with Matchers:
  import Counter.*

  val testKit = ActorTestKit()

  "Counter" should "start at 0" in {
    val counter = testKit.spawn(Counter())
    val probe = testKit.createTestProbe[Int]()
    counter ! GetCount(probe.ref)
    probe.expectMessage(0)
  }

  it should "increment correctly" in {
    val counter = testKit.spawn(Counter())
    val probe = testKit.createTestProbe[Int]()

    counter ! Increment
    counter ! Increment
    counter ! Increment
    counter ! GetCount(probe.ref)

    probe.expectMessage(3)
  }

  override def afterAll(): Unit = testKit.shutdownTestKit()
```

---

## Testing Futures

```scala
import org.scalatest.flatspec.AsyncFlatSpec
import org.scalatest.matchers.should.Matchers
import scala.concurrent.Future

class AsyncSpec extends AsyncFlatSpec with Matchers:
  // AsyncFlatSpec: tests return Future[Assertion]

  "Future computation" should "complete with correct value" in {
    val future = Future {
      Thread.sleep(10)
      42
    }
    future.map { result =>
      result shouldBe 42
    }
  }

  it should "handle failures" in {
    val failed = Future.failed(new RuntimeException("oops"))
    recoverToExceptionIf[RuntimeException](failed).map { ex =>
      ex.getMessage shouldBe "oops"
    }
  }

  it should "compose futures correctly" in {
    def double(n: Int): Future[Int] = Future(n * 2)
    def addOne(n: Int): Future[Int] = Future(n + 1)

    val result = for
      d <- double(5)
      a <- addOne(d)
    yield a

    result.map(_ shouldBe 11)
  }
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ ScalaTest: FunSuite, FlatSpec, WordSpec styles
- ✅ Matchers: shouldEqual, should contain, shouldBe
- ✅ Fixtures: BeforeAndAfterEach
- ✅ Property-Based Testing กับ ScalaCheck
- ✅ Mocking ด้วย Mockito Scala
- ✅ Testing Akka Typed Actors
- ✅ Testing Futures กับ AsyncFlatSpec

---

*[← Part 22: Akka Streams](part-22-akka-streams.md) | [Part 24: SBT Build Tool →](part-24-sbt.md)*
