# ส่วนที่ 94: Scala Best Practices

## สารบัญ

1. [Code Style Guide (Scalafmt)](#code-style-guide-scalafmt)
2. [Package Organization](#package-organization)
3. [Error Handling Conventions](#error-handling-conventions)
4. [Null Safety: Option vs null](#null-safety-option-vs-null)
5. [Exception vs Typed Errors](#exception-vs-typed-errors)
6. [Performance Anti-patterns](#performance-anti-patterns)
7. [Code Review Checklist](#code-review-checklist)
8. [สรุป](#สรุป)

---

## Code Style Guide (Scalafmt)

Scalafmt เป็น opinionated code formatter สำหรับ Scala ช่วยให้ทีมมี consistent style

### Installation และ Configuration

```scala
// project/plugins.sbt
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.5.2")
```

```hocon
// .scalafmt.conf
version = "3.8.1"
runner.dialect = scala3

maxColumn = 100

indent.main = 2
indent.significant = 2

align.preset = most
align.tokens."+" = [
  { code = "=",  owner = "Defn.Val" }
  { code = "=",  owner = "Defn.Var" }
  { code = "//", owner = ".*" }
]

newlines.source = keep
newlines.afterCurlyLambdaParams = squash
newlines.beforeCurlyLambdaParams = multilineWithCaseOnly

danglingParentheses.preset = true

rewrite.rules = [RedundantBraces, SortModifiers, PreferCurlyFors]
rewrite.redundantBraces.stringInterpolation = true
rewrite.sortModifiers.order = [
  "implicit", "final", "sealed", "abstract",
  "override", "private", "protected", "lazy"
]
```

### Naming Conventions

```scala
// Classes and Objects: PascalCase
class UserRepository
object DatabaseConfig
trait EventPublisher

// Methods and values: camelCase
def findUserById(id: UserId): Option[User]
val defaultTimeout = 30.seconds
var mutableCounter = 0

// Constants: ALL_CAPS หรือ PascalCase ก็ได้ใน Scala
val MaxRetries = 3
val DEFAULT_PAGE_SIZE = 20

// Type parameters: single uppercase letter หรือ descriptive
class Container[A]
def transform[F[_], A, B](fa: F[A])(f: A => B): F[B]

// ❌ Bad naming
def fn(x: Int): Int = x + 1  // ชื่อไม่สื่อความหมาย
class d               // ย่อเกินไป
var Temp = 0          // ขึ้นต้นด้วยตัวใหญ่ทั้งที่เป็น variable

// ✅ Good naming
def increment(count: Int): Int = count + 1
class DatabaseConnection
var temporaryBuffer = 0
```

### Formatting Guidelines

```scala
// ✅ Method definitions - align parameters
def createUser(
  name:      String,
  email:     String,
  role:      UserRole,
  createdAt: Instant = Instant.now()
): IO[User]

// ✅ Case class - one field per line for long classes
case class OrderRequest(
  customerId: CustomerId,
  items:      List[OrderItem],
  shippingAddress: Address,
  paymentMethod:   PaymentMethod,
  notes:      Option[String] = None
)

// ✅ For comprehension - indent consistently
val result =
  for
    user    <- userRepo.findById(userId)
    profile <- profileRepo.findByUserId(user.id)
    _       <- analyticsService.track(UserViewed(user.id))
  yield UserWithProfile(user, profile)

// ✅ Pattern matching - align cases
x match
  case Success(value)      => handleSuccess(value)
  case Failure(exception)  => handleError(exception)
  case _                   => handleUnknown()

// ❌ Inconsistent formatting
val r = for { u <- userRepo.findById(userId); p <- profileRepo.findByUserId(u.id) } yield (u,p)
```

---

## Package Organization

### Layered Package Structure

```
com.example.myapp/
├── domain/
│   ├── model/          # Entities, Value Objects
│   │   ├── User.scala
│   │   ├── Order.scala
│   │   └── package.scala
│   ├── repository/     # Repository traits (ports)
│   │   └── UserRepository.scala
│   ├── service/        # Domain services
│   │   └── UserDomainService.scala
│   └── error/          # Domain errors
│       └── DomainError.scala
├── application/
│   ├── usecase/        # Application use cases
│   │   ├── CreateUser.scala
│   │   └── GetUser.scala
│   └── dto/            # Data Transfer Objects
│       └── UserDto.scala
├── infrastructure/
│   ├── database/       # DB implementations
│   │   └── DoobieUserRepo.scala
│   ├── http/           # HTTP client implementations
│   │   └── EmailServiceImpl.scala
│   └── config/         # Config loading
│       └── AppConfig.scala
└── api/
    ├── routes/         # HTTP route definitions
    │   └── UserRoutes.scala
    ├── middleware/     # Auth, logging, etc.
    │   └── AuthMiddleware.scala
    └── Main.scala      # Application entry point
```

### Package Object Best Practices

```scala
// domain/model/package.scala
package com.example.myapp.domain

package object model:
  // Type aliases for clarity
  type UserId    = java.util.UUID
  type Email     = String
  type NonEmptyString = String

  // Common utilities
  def generateId(): UserId = java.util.UUID.randomUUID()

// ✅ Prefer opaque types over type aliases for safety
package com.example.myapp.domain.model

opaque type UserId = java.util.UUID
object UserId:
  def apply(value: java.util.UUID): UserId = value
  def generate(): UserId = java.util.UUID.randomUUID()
  def fromString(s: String): Either[String, UserId] =
    scala.util.Try(java.util.UUID.fromString(s))
      .toEither
      .left.map(_ => s"Invalid UUID: $s")
  extension (id: UserId)
    def value: java.util.UUID = id
    def show: String = id.toString
```

### Module Boundaries

```scala
// ✅ Enforce boundaries via access modifiers
package com.example.myapp.domain.model

// Public: accessible everywhere
case class User(id: UserId, name: String, email: Email)

// Package-private: only accessible within domain
private[domain] case class UserAuditLog(userId: UserId, action: String)

// Protected within hierarchy
sealed abstract class BaseEntity protected (val id: java.util.UUID)

// ✅ Use companion objects to control construction
case class Email private (value: String)
object Email:
  def apply(value: String): Either[String, Email] =
    if value.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$") then
      Right(new Email(value))
    else
      Left(s"'$value' is not a valid email address")

  // unsafe for use in tests only
  def unsafeApply(value: String): Email = new Email(value)
```

---

## Error Handling Conventions

### Error Type Hierarchy

```scala
// ✅ Define a clear error hierarchy
sealed trait AppError derives CanEqual

// Domain errors (business logic violations)
sealed trait DomainError extends AppError
object DomainError:
  case class NotFound(entityType: String, id: String) extends DomainError
  case class AlreadyExists(entityType: String, field: String, value: String) extends DomainError
  case class ValidationError(field: String, message: String) extends DomainError
  case class BusinessRuleViolation(rule: String, context: Map[String, String] = Map.empty) extends DomainError

// Infrastructure errors (technical failures)
sealed trait InfraError extends AppError
object InfraError:
  case class DatabaseError(cause: Throwable, query: Option[String] = None) extends InfraError
  case class NetworkError(url: String, statusCode: Option[Int] = None, cause: Throwable) extends InfraError
  case class SerializationError(message: String, cause: Throwable) extends InfraError
  case class TimeoutError(operation: String, durationMs: Long) extends InfraError

// Application errors (orchestration failures)
sealed trait ServiceError extends AppError
object ServiceError:
  case class Unauthorized(reason: String) extends ServiceError
  case class Forbidden(resource: String, action: String) extends ServiceError
  case class RateLimitExceeded(clientId: String, limit: Int) extends ServiceError
```

### Error Handling Patterns

```scala
import cats.data.{EitherT, ValidatedNel}
import cats.syntax.all.*

// ✅ Pattern 1: EitherT for sequential computation with early failure
type Result[A] = EitherT[IO, AppError, A]

def processOrder(request: OrderRequest): Result[Order] =
  for
    user     <- EitherT(findUser(request.userId))
    product  <- EitherT(findProduct(request.productId))
    _        <- EitherT.fromEither[IO](validateStock(product, request.quantity))
    order    <- EitherT(createOrder(user, product, request.quantity))
    _        <- EitherT(chargePayment(user, order.total))
  yield order

// ✅ Pattern 2: Validated for accumulating multiple errors
def validateRegistration(
  name:     String,
  email:    String,
  password: String
): ValidatedNel[String, RegistrationData] =
  (
    validateName(name).toValidatedNel,
    validateEmail(email).toValidatedNel,
    validatePassword(password).toValidatedNel
  ).mapN(RegistrationData.apply)

def validateName(name: String): Either[String, String] =
  if name.trim.length >= 2 then Right(name.trim)
  else Left("Name must be at least 2 characters")

def validateEmail(email: String): Either[String, String] =
  if email.contains("@") then Right(email.toLowerCase)
  else Left("Invalid email format")

def validatePassword(password: String): Either[String, String] =
  if password.length >= 8 then Right(password)
  else Left("Password must be at least 8 characters")

// Usage: collects all errors
validateRegistration("A", "notanemail", "short") match
  case Valid(data)    => println(s"Valid: $data")
  case Invalid(errors) => println(s"Errors: ${errors.toList.mkString(", ")}")
  // Errors: Name must be at least 2 characters, Invalid email format, Password must be at least 8 characters

// ✅ Pattern 3: Error recovery
def fetchWithFallback(url: String): IO[String] =
  httpClient.get(url).handleErrorWith:
    case _: java.net.ConnectException =>
      IO.println(s"Primary failed, trying fallback") *>
        httpClient.get(fallbackUrl(url))
    case e =>
      IO.raiseError(e)  // re-raise unexpected errors
```

---

## Null Safety: Option vs null

### ทำไมต้องหลีกเลี่ยง null

```scala
// ❌ Java-style null - causes NullPointerException
def findUser(id: Int): User = null  // ไม่บอกว่า user อาจไม่มี

val user = findUser(999)
val name = user.name  // NPE! ไม่รู้ว่า user เป็น null

// ✅ Scala-style Option - explicit about absence
def findUser(id: Int): Option[User] = None  // ชัดเจนว่าอาจไม่มี

val user = findUser(999)
val name = user.map(_.name)           // Option[String]
val nameOrDefault = user.map(_.name).getOrElse("Anonymous")

// ✅ Chaining safely
def getCity(userId: Int): Option[String] =
  findUser(userId)
    .flatMap(_.address)
    .map(_.city)

// ✅ Convert Java nulls at boundaries
import java.util.{Map => JMap}

def fromJava(javaMap: JMap[String, String], key: String): Option[String] =
  Option(javaMap.get(key))  // wraps null in Option

// ✅ Pattern matching for clarity
findUser(1) match
  case Some(user) => println(s"Found: ${user.name}")
  case None       => println("User not found")
```

### Option Best Practices

```scala
// ✅ Use for-comprehension for chained Options
def getDisplayInfo(userId: Int): Option[String] =
  for
    user    <- findUser(userId)
    profile <- findProfile(user.id)
    address <- profile.primaryAddress
  yield s"${user.name} lives at ${address.street}"

// ✅ orElse for fallback Options
def findConfig(key: String): Option[String] =
  localConfig.get(key)
    .orElse(envConfig.get(key))
    .orElse(defaultConfig.get(key))

// ✅ Use fold instead of map + getOrElse
val greeting = findUser(1).fold("Hello, Guest")(u => s"Hello, ${u.name}")

// ❌ Avoid get - throws NoSuchElementException
val user = findUser(1).get  // dangerous!

// ❌ Avoid isDefined + get pattern
if findUser(1).isDefined then findUser(1).get.name  // wasteful, still unsafe

// ✅ Use exists, forall, contains for predicates
val isAdmin = findUser(1).exists(_.role == Role.Admin)
val allConfirmed = users.forall(_.isEmailConfirmed)

// ✅ Convert between Option and Either
def requireUser(id: Int): Either[String, User] =
  findUser(id).toRight(s"User $id not found")

def optionalEmail(id: Int): Option[String] =
  getUserEmail(id).toOption  // Either -> Option
```

---

## Exception vs Typed Errors

### เมื่อไหรควรใช้อะไร

```scala
// ✅ Use typed errors (Either/IO/ZIO) for expected failures
def divide(a: Int, b: Int): Either[String, Double] =
  if b == 0 then Left("Division by zero")
  else Right(a.toDouble / b)

// ✅ Use IO.raiseError for unrecoverable/unexpected failures
def loadConfig(path: String): IO[Config] =
  IO.blocking(scala.io.Source.fromFile(path).mkString)
    .flatMap: content =>
      IO.fromEither(parseConfig(content))
    .adaptError: case e: java.io.IOException =>
      new RuntimeException(s"Config file not found: $path", e)

// ✅ Typed errors for domain logic
def transferMoney(
  fromId: AccountId,
  toId:   AccountId,
  amount: Money
): IO[Either[TransferError, Transfer]] =
  for
    from  <- accountRepo.findById(fromId)
    to    <- accountRepo.findById(toId)
    result = (from, to) match
      case (None, _)    => Left(TransferError.SourceNotFound(fromId))
      case (_, None)    => Left(TransferError.DestinationNotFound(toId))
      case (Some(f), Some(t)) =>
        if f.balance < amount then Left(TransferError.InsufficientFunds(f.balance, amount))
        else Right(performTransfer(f, t, amount))
  yield result

// ❌ Don't use exceptions for control flow
def badDivide(a: Int, b: Int): Double =
  try a.toDouble / b
  catch case _: ArithmeticException => 0.0  // hiding errors

// ✅ Exceptions OK for: programming errors, system errors
def getElement(arr: Array[Int], idx: Int): Int =
  if idx < 0 || idx >= arr.length then
    throw new IndexOutOfBoundsException(s"Index $idx out of bounds for length ${arr.length}")
  arr(idx)
```

### Error Boundary Pattern

```scala
// ✅ Catch all exceptions at system boundaries (HTTP handlers, message consumers)
def handleRequest(req: Request): IO[Response] =
  businessLogic(req)
    .handleError: e =>
      // Log unexpected exceptions
      logger.error("Unexpected error", e) *>
        IO.pure(Response.internalServerError("An unexpected error occurred"))
    .flatMap:
      case Left(domainError) =>
        IO.pure(domainErrorToResponse(domainError))
      case Right(result) =>
        IO.pure(Response.ok(result))

// ✅ ZIO error handling
val program: ZIO[Any, AppError, Result] =
  businessLogic
    .mapError:
      case e: DatabaseException => AppError.Infrastructure(e)
      case e: ValidationException => AppError.Domain(e)
    .tapError(e => ZIO.logError(s"Error: $e"))
    .retry(Schedule.recurs(3) && Schedule.exponential(100.millis))
```

---

## Performance Anti-patterns

### 1. Collection Performance

```scala
// ❌ Repeated concatenation - O(n²)
def buildString(items: List[String]): String =
  items.foldLeft(""):
    case (acc, item) => acc + ", " + item  // creates new string each time

// ✅ Use StringBuilder or mkString
def buildStringGood(items: List[String]): String =
  items.mkString(", ")

// ✅ Use StringBuilder for complex cases
def buildStringBuilder(items: List[String]): String =
  val sb = new StringBuilder
  items.foreach: item =>
    if sb.nonEmpty then sb.append(", ")
    sb.append(item)
  sb.toString

// ❌ Converting to List repeatedly for lookup
def findItems(ids: Set[Int], items: List[Item]): List[Item] =
  items.filter(i => ids.contains(i.id))  // OK, but...
  // If called in loop, ids.contains is O(1) which is fine

// ❌ But this is bad - O(n) lookup in loop
def findItemsBad(id: Int, items: List[Item]): Option[Item] =
  items.find(_.id == id)  // O(n) - use Map instead

// ✅ Convert to Map once
def processItems(itemIds: List[Int], items: List[Item]): List[Option[Item]] =
  val itemMap = items.map(i => i.id -> i).toMap  // build once O(n)
  itemIds.map(id => itemMap.get(id))             // O(1) per lookup
```

### 2. Lazy vs Strict Collections

```scala
// ❌ Eager evaluation - materializes entire collection
val result = (1 to 1_000_000)
  .filter(_ % 2 == 0)
  .map(_ * 3)
  .take(10)
  .toList  // creates intermediate collections

// ✅ Lazy evaluation with LazyList or view
val resultLazy = LazyList.from(1)
  .filter(_ % 2 == 0)
  .map(_ * 3)
  .take(10)
  .toList  // only processes until 10 elements found

// ✅ Using .view for collections
val resultView = (1 to 1_000_000).view
  .filter(_ % 2 == 0)
  .map(_ * 3)
  .take(10)
  .toList  // no intermediate collections
```

### 3. Memory Leaks

```scala
// ❌ Holding references in closures
class Cache:
  val store = scala.collection.mutable.Map.empty[String, Array[Byte]]
  def add(key: String, data: Array[Byte]): Unit = store.put(key, data)
  // No eviction = unbounded memory growth

// ✅ Use bounded cache with eviction
import java.util.concurrent.{ConcurrentHashMap, LinkedHashMap}

class BoundedCache[K, V](maxSize: Int):
  private val store = java.util.Collections.synchronizedMap(
    new java.util.LinkedHashMap[K, V](maxSize, 0.75f, true):
      override def removeEldestEntry(eldest: java.util.Map.Entry[K, V]): Boolean =
        size() > maxSize
  )
  def put(key: K, value: V): Unit = store.put(key, value)
  def get(key: K): Option[V] = Option(store.get(key))

// ❌ Accumulating all data in memory
def processEvents(events: LazyList[Event]): Map[String, Int] =
  events.foldLeft(Map.empty[String, Int]): // fine if bounded
    case (acc, e) => acc + (e.userId -> (acc.getOrElse(e.userId, 0) + 1))

// ✅ For unbounded streams, use windowing or streaming aggregation
import fs2.Stream
import cats.effect.IO

def processEventStream(events: Stream[IO, Event]): Stream[IO, Map[String, Int]] =
  events
    .groupWithin(1000, 10.seconds)  // process in time-bounded windows
    .map(chunk => chunk.toList.groupBy(_.userId).view.mapValues(_.size).toMap)
```

### 4. Avoid Boxing

```scala
// ❌ Using generic collections with primitives causes boxing
val ints: List[Int] = List(1, 2, 3)  // Int is boxed to java.lang.Integer

// ✅ Use specialized collections for performance-critical code
import java.util

val intArray: Array[Int] = Array(1, 2, 3)  // unboxed primitives
val intVector = scala.collection.immutable.ArraySeq(1, 2, 3)  // value-specialized

// For number-crunching, consider using primitive arrays directly
def sumArray(arr: Array[Int]): Int =
  var sum = 0
  var i   = 0
  while i < arr.length do
    sum += arr(i)
    i += 1
  sum  // much faster than functional style for tight loops
```

### 5. String Operations

```scala
// ❌ Regex compilation in hot path
def isValidEmail(email: String): Boolean =
  email.matches("[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}")  // compiles regex every call

// ✅ Compile once
object Validators:
  private val emailRegex = "[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}".r

  def isValidEmail(email: String): Boolean =
    emailRegex.matches(email)

// ❌ String interpolation in logger (always evaluates arguments)
logger.debug(s"Processing user ${user.toDetailedString}")  // expensive even if DEBUG disabled

// ✅ Use lazy string evaluation
logger.debug(s"Processing user ${user.id}")  // cheap: just ID
// Or use parameterized logging
logger.debug("Processing user {}", user.id)  // SLF4J style - doesn't format if disabled
```

---

## Code Review Checklist

### Domain Model

```scala
// Checklist: Domain Model
// ✅ Value objects ใช้ opaque types หรือ case classes
// ✅ Entities มี identity ที่ชัดเจน (ID)
// ✅ Validation อยู่ใน constructor/factory method
// ✅ Business logic อยู่ใน domain objects ไม่ใช่ service
// ✅ Side effects ไม่มีใน pure domain model
// ✅ toString มีความหมาย (case class auto-generate)
// ✅ Equals/hashCode ถูกต้อง (case class auto-generate)

// Example of well-structured domain object
case class Money private (amount: BigDecimal, currency: Currency):
  require(amount >= 0, "Amount cannot be negative")

  def +(other: Money): Either[String, Money] =
    if currency != other.currency then Left(s"Currency mismatch: $currency vs ${other.currency}")
    else Right(Money.unsafe(amount + other.amount, currency))

  def *(factor: BigDecimal): Money =
    Money.unsafe(amount * factor, currency)

  override def toString: String = s"${currency.symbol}${amount.setScale(2)}"

object Money:
  def apply(amount: BigDecimal, currency: Currency): Either[String, Money] =
    if amount < 0 then Left(s"Amount cannot be negative: $amount")
    else Right(new Money(amount, currency))

  def unsafe(amount: BigDecimal, currency: Currency): Money =
    new Money(amount, currency)

  val zero: Money = new Money(0, Currency.USD)
```

### API Design

```scala
// Checklist: API Design
// ✅ Return types ระบุ failure explicitly (Either/Option/IO)
// ✅ Input validation ก่อน business logic
// ✅ Error messages มีความหมาย สำหรับ user
// ✅ API ไม่ leak internal details (no internal exceptions)
// ✅ Pagination สำหรับ list endpoints
// ✅ Idempotency สำหรับ mutations (PUT, DELETE)
// ✅ Rate limiting documentation

// ✅ Good API design
trait UserService[F[_]]:
  // Clear return type - fails if user not found or validation fails
  def updateEmail(userId: UserId, newEmail: String): F[Either[UpdateEmailError, User]]

  // Pagination built-in
  def listUsers(page: PageRequest): F[Page[User]]

  // Returns what was deleted, or error if not found
  def deleteUser(userId: UserId): F[Either[UserNotFoundError, UserId]]

sealed trait UpdateEmailError
case class UserNotFound(id: UserId) extends UpdateEmailError
case class EmailInUse(email: String) extends UpdateEmailError
case class InvalidEmail(input: String, reason: String) extends UpdateEmailError

case class PageRequest(page: Int, size: Int, sortBy: String = "createdAt", ascending: Boolean = false):
  require(page >= 0, "Page must be non-negative")
  require(size > 0 && size <= 100, "Page size must be between 1 and 100")

case class Page[A](items: List[A], total: Long, page: Int, pageSize: Int):
  def totalPages: Long = math.ceil(total.toDouble / pageSize).toLong
  def hasNext: Boolean = (page + 1) * pageSize < total
  def hasPrev: Boolean = page > 0
```

### Testing

```scala
// Checklist: Testing
// ✅ Unit tests สำหรับ pure functions และ domain logic
// ✅ Integration tests สำหรับ repository และ HTTP
// ✅ Property-based tests สำหรับ complex invariants
// ✅ Test coverage สำหรับ edge cases (empty, max, min)
// ✅ Tests ไม่ depend on order หรือ external state
// ✅ Test names อธิบาย behavior ไม่ใช่แค่ method name

import org.scalatest.matchers.should.Matchers
import org.scalatest.wordspec.AnyWordSpec
import org.scalacheck.{Gen, Prop}
import org.scalacheck.Prop.forAll

class MoneySpec extends AnyWordSpec with Matchers:

  "Money" should {
    "not allow negative amounts" in {
      Money(-1, Currency.USD) shouldBe Left("Amount cannot be negative: -1")
    }

    "correctly add same currencies" in {
      val result =
        for
          a <- Money(10, Currency.USD)
          b <- Money(5, Currency.USD)
          c <- a + b
        yield c.amount
      result shouldBe Right(BigDecimal(15))
    }

    "fail when adding different currencies" in {
      val result =
        for
          usd <- Money(10, Currency.USD)
          eur <- Money(5, Currency.EUR)
          r   <- usd + eur
        yield r
      result shouldBe Left("Currency mismatch: USD vs EUR")
    }
  }

// Property-based test
class MoneyPropertySpec extends AnyWordSpec with Matchers:
  "Money arithmetic" should {
    "be commutative for addition" in {
      forAll(Gen.posNum[Double], Gen.posNum[Double]): (a, b) =>
        val ma = Money.unsafe(BigDecimal(a), Currency.USD)
        val mb = Money.unsafe(BigDecimal(b), Currency.USD)
        val ab = (ma + mb).map(_.amount)
        val ba = (mb + ma).map(_.amount)
        ab == ba
      .check()
    }
  }
```

### Complete Code Review Template

```scala
// Code Review Checklist - Paste in PR description

/**
 * ## Code Review Checklist
 *
 * ### Correctness
 * - [ ] Business logic ถูกต้องตาม requirements
 * - [ ] Edge cases handled (empty collections, null inputs, overflow)
 * - [ ] No race conditions in concurrent code
 * - [ ] Error cases return appropriate typed errors
 * - [ ] No silent failures (swallowed exceptions)
 *
 * ### Design
 * - [ ] SRP: each class/method has single responsibility
 * - [ ] DRY: no duplicated logic
 * - [ ] Dependencies injected, not created internally
 * - [ ] Immutable data structures used where possible
 * - [ ] Side effects isolated and explicit
 *
 * ### Performance
 * - [ ] No O(n²) algorithms where O(n log n) or O(n) possible
 * - [ ] No memory leaks (unbounded collections, unclosed resources)
 * - [ ] Lazy evaluation used for large/infinite collections
 * - [ ] Database queries have appropriate indexes
 * - [ ] N+1 queries avoided
 *
 * ### Security
 * - [ ] Input validation at boundaries
 * - [ ] No hardcoded secrets/credentials
 * - [ ] SQL injection not possible (parameterized queries)
 * - [ ] Sensitive data not logged
 * - [ ] Authentication/authorization checked
 *
 * ### Testing
 * - [ ] Unit tests for new/changed logic
 * - [ ] Integration tests for DB/HTTP interactions
 * - [ ] Happy path and error path both tested
 * - [ ] Tests don't depend on external state or order
 *
 * ### Documentation
 * - [ ] Public API has ScalaDoc
 * - [ ] Complex algorithms explained in comments
 * - [ ] Breaking changes documented
 */
```

### Common Anti-patterns to Flag

```scala
// ❌ Partial functions without exhaustive matching
def process(opt: Option[Int]): Int =
  opt.get  // throws if None

// ❌ Using null
def maybeUser(): User = null  // use Option[User]

// ❌ Mutable state in shared scope
var sharedCounter = 0  // not thread-safe

// ❌ Thread.sleep in IO code
IO(Thread.sleep(1000))  // blocks thread pool
// ✅ Use IO.sleep(1.second) instead

// ❌ Catching all exceptions silently
try doSomething()
catch case _: Throwable => ()  // swallows errors!

// ❌ Using .toList.head on potentially empty collection
myList.filter(pred).head  // throws if empty
// ✅ Use headOption
myList.filter(pred).headOption

// ❌ String comparison for enums
if status == "active" then ...  // typo-prone
// ✅ Use sealed trait/enum
enum Status { case Active, Inactive, Pending }
if status == Status.Active then ...

// ❌ isInstanceOf/asInstanceOf without good reason
x.asInstanceOf[String]  // use pattern matching
// ✅
x match
  case s: String => s
  case _         => "default"
```

---

## สรุป

Scala Best Practices ครอบคลุมหลายมิติ:

- **Scalafmt**: ใช้ formatter และ config ที่ตกลงกันในทีม ให้ consistent codebase
- **Package Organization**: แยก layers ชัดเจน ตาม domain/application/infrastructure/api
- **Error Handling**: ใช้ typed errors (Either/Option) สำหรับ expected failures, exceptions สำหรับ programming errors
- **Null Safety**: หลีกเลี่ยง null ทุกกรณี ใช้ Option แทน ด้วย opaque types ที่ validate ตั้งแต่ construction
- **Performance**: หลีก O(n²), ใช้ lazy collections, ไม่ box primitives โดยไม่จำเป็น
- **Code Review**: มี checklist ที่ครอบคลุม correctness, design, performance, security, testing

---

*[← ส่วนที่ 93: Scala Interview Preparation](part-93-interview-prep.md)*
