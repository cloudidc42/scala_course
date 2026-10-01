# ส่วนที่ 81: Functional Design Patterns

## สารบัญ

1. [Tagless Final: Algebra + Interpreter](#tagless-final-algebra--interpreter)
2. [Effect Rotation](#effect-rotation)
3. [Capability-Based Design](#capability-based-design)
4. [MTL vs ZLayer vs Reader](#mtl-vs-zlayer-vs-reader)
5. [Algebraic Design กับ ADTs](#algebraic-design-กับ-adts)
6. [Service Locator vs Dependency Injection](#service-locator-vs-dependency-injection)
7. [Complete Application Design](#complete-application-design)
8. [สรุป](#สรุป)

---

## Tagless Final: Algebra + Interpreter

Tagless Final เป็น pattern ที่แยก "สิ่งที่ต้องการทำ" ออกจาก "วิธีทำ" โดยใช้ higher-kinded types

### Algebra Definition

```scala
// Algebra คือ interface ที่ describe operations
trait UserRepository[F[_]]:
  def findById(id: UserId): F[Option[User]]
  def findAll: F[List[User]]
  def save(user: User): F[User]
  def delete(id: UserId): F[Boolean]

trait EmailService[F[_]]:
  def send(to: Email, subject: String, body: String): F[Unit]
  def sendBatch(emails: List[(Email, String, String)]): F[Int]

trait CacheService[F[_]]:
  def get[A](key: String): F[Option[A]]
  def set[A](key: String, value: A, ttl: Option[Duration]): F[Unit]
  def delete(key: String): F[Boolean]

// Domain types
case class UserId(value: String) extends AnyVal
case class Email(value: String) extends AnyVal
case class User(id: UserId, name: String, email: Email, active: Boolean)
```

### Interpreters

```scala
import cats.effect.IO
import cats.effect.Ref
import cats.Monad
import cats.implicits.*

// In-memory interpreter สำหรับ testing
class InMemoryUserRepo(state: Ref[IO, Map[UserId, User]]) extends UserRepository[IO]:
  def findById(id: UserId): IO[Option[User]] =
    state.get.map(_.get(id))
  
  def findAll: IO[List[User]] =
    state.get.map(_.values.toList)
  
  def save(user: User): IO[User] =
    state.update(_.updated(user.id, user)).as(user)
  
  def delete(id: UserId): IO[Boolean] =
    state.modify { m =>
      if m.contains(id) then (m - id, true)
      else (m, false)
    }

object InMemoryUserRepo:
  def create: IO[InMemoryUserRepo] =
    Ref.of[IO, Map[UserId, User]](Map.empty).map(new InMemoryUserRepo(_))

// Database interpreter
class DatabaseUserRepo(xa: doobie.Transactor[IO]) extends UserRepository[IO]:
  import doobie.*
  import doobie.implicits.*
  
  def findById(id: UserId): IO[Option[User]] =
    sql"SELECT id, name, email, active FROM users WHERE id = ${id.value}"
      .query[User]
      .option
      .transact(xa)
  
  def findAll: IO[List[User]] =
    sql"SELECT id, name, email, active FROM users"
      .query[User]
      .to[List]
      .transact(xa)
  
  def save(user: User): IO[User] =
    sql"""INSERT INTO users (id, name, email, active)
          VALUES (${user.id.value}, ${user.name}, ${user.email.value}, ${user.active})
          ON CONFLICT (id) DO UPDATE
          SET name = EXCLUDED.name, email = EXCLUDED.email, active = EXCLUDED.active"""
      .update.run
      .transact(xa)
      .as(user)
  
  def delete(id: UserId): IO[Boolean] =
    sql"DELETE FROM users WHERE id = ${id.value}"
      .update.run
      .transact(xa)
      .map(_ > 0)

// NoOp interpreter สำหรับ testing ที่ไม่ต้องการ side effects
class NoOpEmailService[F[_]](using F: Monad[F]) extends EmailService[F]:
  def send(to: Email, subject: String, body: String): F[Unit] =
    F.pure(())
  
  def sendBatch(emails: List[(Email, String, String)]): F[Int] =
    F.pure(emails.length)
```

### Business Logic ด้วย Tagless Final

```scala
import cats.Monad
import cats.implicits.*

class UserService[F[_]: Monad](
  userRepo: UserRepository[F],
  emailService: EmailService[F],
  cache: CacheService[F]
):
  def registerUser(name: String, email: String): F[Either[String, User]] =
    for
      // check cache first
      cached <- cache.get[User](s"user:email:$email")
      result <- cached match
        case Some(_) =>
          Monad[F].pure(Left[String, User](s"Email $email already registered"))
        case None =>
          for
            existing <- userRepo.findAll
            emailTaken = existing.exists(_.email.value == email)
            result <-
              if emailTaken then
                Monad[F].pure(Left[String, User](s"Email $email already taken"))
              else
                val user = User(
                  UserId(java.util.UUID.randomUUID().toString),
                  name,
                  Email(email),
                  active = true
                )
                for
                  saved <- userRepo.save(user)
                  _ <- cache.set(s"user:${saved.id.value}", saved, Some(Duration(5, "minutes")))
                  _ <- emailService.send(
                    saved.email,
                    "Welcome!",
                    s"Hello $name, your account has been created."
                  )
                yield Right(saved)
          yield result
    yield result
  
  def deactivateUser(id: UserId): F[Either[String, Unit]] =
    for
      userOpt <- userRepo.findById(id)
      result <- userOpt match
        case None => Monad[F].pure(Left(s"User ${id.value} not found"))
        case Some(user) =>
          for
            _ <- userRepo.save(user.copy(active = false))
            _ <- cache.delete(s"user:${id.value}")
            _ <- emailService.send(
              user.email,
              "Account Deactivated",
              "Your account has been deactivated."
            )
          yield Right(())
    yield result
  
  def getActiveUsers: F[List[User]] =
    userRepo.findAll.map(_.filter(_.active))
```

---

## Effect Rotation

Effect Rotation คือเทคนิคในการเปลี่ยน stacked monad transformers ให้เป็น parallel effects

### ปัญหาของ Monad Transformer Stacks

```scala
import cats.data.*
import cats.effect.IO

// Stack ที่ซับซ้อน: EitherT[WriterT[IO, Log, ?], Error, ?]
type AppEffect[A] = EitherT[WriterT[IO, List[String], *], AppError, A]

// ปัญหา:
// 1. type signatures ซับซ้อน
// 2. performance overhead จาก boxing
// 3. ยาก compose กับ effects อื่น
// 4. error handling ซับซ้อน
```

### Effect Rotation ด้วย IOLocal

```scala
import cats.effect.{IO, IOLocal}

// แทนที่จะ stack effects, ใช้ parallel effects
case class AppEnv(
  correlationId: String,
  userId: Option[UserId],
  debugMode: Boolean
)

// ใช้ IOLocal สำหรับ context propagation
def makeApp: IO[Unit] =
  for
    envRef <- IOLocal(AppEnv("", None, false))
    result <- runWithEnv(envRef)
  yield result

def runWithEnv(env: IOLocal[AppEnv]): IO[Unit] =
  for
    _ <- env.set(AppEnv("req-123", Some(UserId("user-456")), true))
    _ <- processRequest(env)
  yield ()

def processRequest(env: IOLocal[AppEnv]): IO[Unit] =
  for
    appEnv <- env.get
    _ = println(s"Processing request ${appEnv.correlationId}")
    _ <- doWork(env)
  yield ()

def doWork(env: IOLocal[AppEnv]): IO[Unit] =
  for
    appEnv <- env.get
    _ = println(s"User: ${appEnv.userId}")
  yield ()
```

### Effect Rotation ด้วย ZIO

```scala
import zio.*

// ZIO Layer approach - แทน monad transformer stack
case class UserRepoConfig(connectionString: String, poolSize: Int)
case class EmailConfig(smtpHost: String, smtpPort: Int)
case class AppConfig(db: UserRepoConfig, email: EmailConfig)

// Services เป็น independent layers
val userRepoLayer: ZLayer[UserRepoConfig, Throwable, UserRepository] =
  ZLayer.scoped {
    for
      config <- ZIO.service[UserRepoConfig]
      pool <- ZIO.acquireRelease(
        ZIO.attempt(createPool(config))
      )(pool => ZIO.succeed(pool.close()))
    yield DatabaseUserRepo(pool)
  }

val emailLayer: ZLayer[EmailConfig, Throwable, EmailService] =
  ZLayer.fromFunction { config =>
    SmtpEmailService(config.smtpHost, config.smtpPort)
  }

// Compose layers
val appLayer: ZLayer[AppConfig, Throwable, UserRepository & EmailService] =
  ZLayer.makeSome[AppConfig, UserRepository & EmailService](
    ZLayer.fromZIO(ZIO.service[AppConfig].map(_.db)) >>> userRepoLayer,
    ZLayer.fromZIO(ZIO.service[AppConfig].map(_.email)) >>> emailLayer
  )
```

---

## Capability-Based Design

Capability-Based Design ใช้ type system เพื่อ encode ว่า operation ต้องการ capability อะไร

### Capability Types

```scala
// Capabilities เป็น type-level permissions
sealed trait Capability
trait ReadCapability extends Capability
trait WriteCapability extends Capability
trait AdminCapability extends Capability
trait AuditCapability extends Capability

// Context ที่มี capabilities
case class UserContext[+Caps <: Capability](
  userId: UserId,
  capabilities: Set[String]
)

// Type aliases สำหรับ common combinations
type ReadOnlyContext = UserContext[ReadCapability]
type ReadWriteContext = UserContext[ReadCapability & WriteCapability]
type AdminContext = UserContext[ReadCapability & WriteCapability & AdminCapability]

// Operations ที่ require specific capabilities
trait SecureUserRepo:
  // ทุกคนอ่านได้
  def findById(id: UserId)(using ctx: UserContext[ReadCapability]): IO[Option[User]]
  
  // เฉพาะ write capability
  def save(user: User)(using ctx: UserContext[WriteCapability]): IO[User]
  
  // เฉพาะ admin
  def deleteAll()(using ctx: UserContext[AdminCapability]): IO[Int]
  
  // เฉพาะ audit capability
  def getAuditLog()(using ctx: UserContext[AuditCapability]): IO[List[AuditEntry]]

// Implementation
class SecureUserRepoImpl(underlying: UserRepository[IO]) extends SecureUserRepo:
  def findById(id: UserId)(using ctx: UserContext[ReadCapability]): IO[Option[User]] =
    underlying.findById(id)
  
  def save(user: User)(using ctx: UserContext[WriteCapability]): IO[User] =
    underlying.save(user)
  
  def deleteAll()(using ctx: UserContext[AdminCapability]): IO[Int] =
    // only admins can do this
    underlying.findAll.flatMap { users =>
      users.traverse(u => underlying.delete(u.id)).map(_.count(identity))
    }
  
  def getAuditLog()(using ctx: UserContext[AuditCapability]): IO[List[AuditEntry]] =
    // read audit log
    IO.pure(List.empty) // simplified
```

### Capability Elevation

```scala
// Elevate capabilities ด้วย authentication
trait AuthService:
  def authenticate(token: String): IO[Option[UserContext[ReadCapability]]]
  def elevateToAdmin(ctx: UserContext[ReadCapability], adminPassword: String): 
    IO[Option[UserContext[AdminCapability]]]

// ตัวอย่างการใช้
def handleRequest(token: String, isAdmin: Boolean, repo: SecureUserRepo)
    (using authService: AuthService): IO[String] =
  for
    ctxOpt <- authService.authenticate(token)
    result <- ctxOpt match
      case None => IO.pure("Unauthorized")
      case Some(readCtx) =>
        given UserContext[ReadCapability] = readCtx
        for
          users <- repo.findById(UserId("123"))
          _ <- if isAdmin then
            authService.elevateToAdmin(readCtx, "admin-pass").flatMap {
              case None => IO.pure(())
              case Some(adminCtx) =>
                given UserContext[AdminCapability] = adminCtx
                repo.deleteAll().void
            }
          else IO.pure(())
        yield users.map(_.name).getOrElse("Not found")
  yield result
```

---

## MTL vs ZLayer vs Reader

### MTL (Monad Transformer Library)

```scala
import cats.mtl.*
import cats.effect.IO
import cats.data.*

// MTL approach ใช้ typeclasses แทน concrete transformers
def fetchUser[F[_]](id: UserId)(using
  F: cats.Monad[F],
  ask: Ask[F, AppEnv],
  raise: Raise[F, AppError],
  tell: Tell[F, List[LogEntry]]
): F[User] =
  for
    env <- ask.ask
    _ <- tell.tell(List(LogEntry(s"Fetching user $id in env ${env.name}")))
    user <- F.pure(User(id, "Alice", Email("alice@example.com"), true))
    _ <- if !user.active then raise.raise(AppError.UserInactive(id))
         else F.pure(())
  yield user

// เปรียบเทียบ: MTL เหมาะเมื่อ
// - ต้องการ abstract over effect type
// - มี multiple interpreters
// - ใช้ cats ecosystem
```

### ZLayer Approach

```scala
import zio.*

// ZLayer - explicit dependency graph
trait UserRepo:
  def findById(id: UserId): Task[Option[User]]

trait Logger:
  def info(msg: String): UIO[Unit]
  def error(msg: String, cause: Throwable): UIO[Unit]

// Service implementations
case class UserRepoLive(db: DatabasePool) extends UserRepo:
  def findById(id: UserId): Task[Option[User]] =
    ZIO.attempt(db.query(s"SELECT * FROM users WHERE id = '${id.value}'"))
      .map(_.headOption.map(rowToUser))

case class ConsoleLogger() extends Logger:
  def info(msg: String): UIO[Unit] =
    ZIO.succeed(println(s"[INFO] $msg"))
  
  def error(msg: String, cause: Throwable): UIO[Unit] =
    ZIO.succeed(println(s"[ERROR] $msg: ${cause.getMessage}"))

// ZLayer definitions
val userRepoLayer: ZLayer[DatabasePool, Nothing, UserRepo] =
  ZLayer.fromFunction(UserRepoLive.apply)

val loggerLayer: ZLayer[Any, Nothing, Logger] =
  ZLayer.succeed(ConsoleLogger())

// Program ที่ใช้ services
val program: ZIO[UserRepo & Logger, Throwable, Unit] =
  for
    logger <- ZIO.service[Logger]
    repo <- ZIO.service[UserRepo]
    _ <- logger.info("Starting...")
    user <- repo.findById(UserId("123"))
    _ <- logger.info(s"Found: $user")
  yield ()

// Run ด้วย layers
val runnable = program.provide(
  userRepoLayer,
  loggerLayer,
  DatabasePool.layer
)
```

### Reader Monad Approach

```scala
import cats.data.Reader
import cats.effect.IO

// Reader Monad - simple dependency injection
case class AppDependencies(
  userRepo: UserRepository[IO],
  emailService: EmailService[IO],
  config: AppConfig
)

type AppReader[A] = Reader[AppDependencies, IO[A]]

// Functions ที่ return Reader
def findUser(id: UserId): AppReader[Option[User]] =
  Reader(deps => deps.userRepo.findById(id))

def sendWelcome(user: User): AppReader[Unit] =
  Reader(deps => deps.emailService.send(
    user.email,
    "Welcome",
    s"Hello ${user.name}"
  ))

// Compose
def onboard(id: UserId): AppReader[Option[User]] =
  for
    userOpt <- findUser(id)
    _ <- userOpt match
      case None => Reader(_ => IO.pure(()))
      case Some(user) => sendWelcome(user)
  yield userOpt

// Run
def runApp(deps: AppDependencies): IO[Option[User]] =
  onboard(UserId("123")).run(deps)
```

### เปรียบเทียบ Approaches

```
Approach  | Complexity | Performance | Testability | Ecosystem
----------|------------|-------------|-------------|----------
MTL       | Medium     | Good        | Excellent   | cats
ZLayer    | Low-Medium | Excellent   | Excellent   | ZIO
Reader    | Low        | Good        | Good        | Standard
TaglessFinal | Medium  | Good        | Excellent   | Both
```

---

## Algebraic Design กับ ADTs

### Modeling Domain ด้วย ADTs

```scala
// ใช้ sealed traits + case classes เพื่อ model domain
sealed trait OrderStatus
object OrderStatus:
  case object Pending   extends OrderStatus
  case object Confirmed extends OrderStatus
  case object Shipped   extends OrderStatus
  case object Delivered extends OrderStatus
  case object Cancelled extends OrderStatus

sealed trait PaymentMethod
object PaymentMethod:
  case class CreditCard(last4: String, expiry: String) extends PaymentMethod
  case class BankTransfer(accountId: String) extends PaymentMethod
  case object CashOnDelivery extends PaymentMethod

sealed trait OrderError
object OrderError:
  case class InsufficientStock(productId: String, requested: Int, available: Int) extends OrderError
  case class PaymentFailed(reason: String) extends OrderError
  case class AddressInvalid(address: String) extends OrderError
  case class OrderAlreadyProcessed(orderId: String) extends OrderError

// Smart constructors ป้องกัน invalid states
case class OrderQuantity private (value: Int)
object OrderQuantity:
  def of(n: Int): Either[String, OrderQuantity] =
    if n <= 0 then Left(s"Quantity must be positive, got $n")
    else if n > 1000 then Left(s"Quantity cannot exceed 1000, got $n")
    else Right(OrderQuantity(n))

case class Price private (cents: Long)
object Price:
  def of(cents: Long): Either[String, Price] =
    if cents < 0 then Left("Price cannot be negative")
    else Right(Price(cents))
  
  def fromDollars(dollars: Double): Either[String, Price] =
    of(Math.round(dollars * 100))
  
  extension (p: Price)
    def toDisplayString: String = f"$$${p.cents / 100.0}%.2f"
    def +(other: Price): Price = Price(p.cents + other.cents)
    def *(qty: OrderQuantity): Price = Price(p.cents * qty.value)
```

### Algebra สำหรับ Order Processing

```scala
// Order algebra - pure ADT-based
sealed trait OrderCommand
object OrderCommand:
  case class PlaceOrder(
    customerId: CustomerId,
    items: List[OrderItem],
    payment: PaymentMethod,
    shippingAddress: Address
  ) extends OrderCommand
  
  case class CancelOrder(orderId: OrderId, reason: String) extends OrderCommand
  case class ShipOrder(orderId: OrderId, trackingNumber: String) extends OrderCommand
  case class ConfirmDelivery(orderId: OrderId) extends OrderCommand

sealed trait OrderEvent
object OrderEvent:
  case class OrderPlaced(
    orderId: OrderId,
    customerId: CustomerId,
    items: List[OrderItem],
    total: Price,
    timestamp: Instant
  ) extends OrderEvent
  
  case class OrderCancelled(orderId: OrderId, reason: String, timestamp: Instant) extends OrderEvent
  case class OrderShipped(orderId: OrderId, trackingNumber: String, timestamp: Instant) extends OrderEvent
  case class OrderDelivered(orderId: OrderId, timestamp: Instant) extends OrderEvent

// State machine ด้วย ADT
case class Order(
  id: OrderId,
  customerId: CustomerId,
  items: List[OrderItem],
  status: OrderStatus,
  total: Price,
  events: List[OrderEvent]
)

// Pure function - ไม่มี side effects
def applyCommand(order: Order, cmd: OrderCommand): Either[OrderError, (Order, OrderEvent)] =
  cmd match
    case OrderCommand.CancelOrder(id, reason) =>
      order.status match
        case OrderStatus.Delivered | OrderStatus.Cancelled =>
          Left(OrderError.OrderAlreadyProcessed(id.value))
        case _ =>
          val event = OrderEvent.OrderCancelled(id, reason, Instant.now())
          Right((order.copy(status = OrderStatus.Cancelled, events = event :: order.events), event))
    
    case OrderCommand.ShipOrder(id, tracking) =>
      order.status match
        case OrderStatus.Confirmed =>
          val event = OrderEvent.OrderShipped(id, tracking, Instant.now())
          Right((order.copy(status = OrderStatus.Shipped, events = event :: order.events), event))
        case other =>
          Left(OrderError.OrderAlreadyProcessed(s"Cannot ship order in status $other"))
    
    case _ => Left(OrderError.OrderAlreadyProcessed("Command not applicable"))
```

---

## Service Locator vs Dependency Injection

### Service Locator Pattern (Anti-pattern)

```scala
// Service Locator - ปัญหา: hidden dependencies
object ServiceLocator:
  private var services: Map[String, Any] = Map.empty
  
  def register[T](name: String, service: T): Unit =
    services = services.updated(name, service)
  
  def get[T](name: String): T =
    services(name).asInstanceOf[T]

// ปัญหา:
class UserService2:
  // hidden dependency - ไม่รู้ว่า depends on อะไรจาก signature
  def findUser(id: String): Option[User] =
    val repo = ServiceLocator.get[UserRepository[IO]]("userRepo")
    // impossible to test without global state
    None
```

### Constructor Injection (Better)

```scala
// Constructor Injection - explicit dependencies
class UserService3(
  private val userRepo: UserRepository[IO],
  private val emailService: EmailService[IO],
  private val cache: CacheService[IO]
):
  def findUser(id: UserId): IO[Option[User]] =
    for
      cached <- cache.get[User](s"user:${id.value}")
      result <- cached match
        case Some(u) => IO.pure(Some(u))
        case None => userRepo.findById(id).flatTap {
          case Some(u) => cache.set(s"user:${id.value}", u, None)
          case None => IO.pure(())
        }
    yield result
  
  // ง่ายต่อการ test ด้วย mock/stub
```

### ZIO Service Pattern

```scala
import zio.*

// ZIO Service Pattern - best of both worlds
trait NotificationService:
  def notify(userId: UserId, message: String): Task[Unit]

object NotificationService:
  // accessor ที่ใช้ง่าย
  def notify(userId: UserId, message: String): ZIO[NotificationService, Throwable, Unit] =
    ZIO.serviceWithZIO(_.notify(userId, message))
  
  // Live implementation
  case class Live(emailService: EmailService[Task], pushService: PushService) 
    extends NotificationService:
    def notify(userId: UserId, message: String): Task[Unit] =
      for
        _ <- emailService.send(Email(s"${userId.value}@example.com"), "Notification", message)
        _ <- pushService.send(userId, message)
      yield ()
  
  // Layer
  val live: ZLayer[EmailService[Task] & PushService, Nothing, NotificationService] =
    ZLayer.fromFunction(Live.apply)
  
  // Test double
  val test: ZLayer[Any, Nothing, NotificationService] =
    ZLayer.succeed(new NotificationService:
      val sent = scala.collection.mutable.ArrayBuffer[(UserId, String)]()
      def notify(userId: UserId, message: String): Task[Unit] =
        ZIO.succeed(sent += ((userId, message)))
    )
```

---

## Complete Application Design

### E-commerce Application Architecture

```scala
// Domain layer
package domain

// Value objects
case class ProductId(value: String) extends AnyVal
case class CustomerId(value: String) extends AnyVal
case class OrderId(value: String) extends AnyVal

// Entities
case class Product(
  id: ProductId,
  name: String,
  price: Price,
  stock: Int,
  category: String
)

case class Cart(
  customerId: CustomerId,
  items: Map[ProductId, Int],
  couponCode: Option[String]
)

// Repository algebras
trait ProductRepository[F[_]]:
  def findById(id: ProductId): F[Option[Product]]
  def findByCategory(category: String): F[List[Product]]
  def updateStock(id: ProductId, delta: Int): F[Either[String, Product]]
  def search(query: String): F[List[Product]]

trait CartRepository[F[_]]:
  def findByCustomer(customerId: CustomerId): F[Option[Cart]]
  def save(cart: Cart): F[Cart]
  def delete(customerId: CustomerId): F[Unit]

// Application services
package application

import cats.effect.IO
import cats.implicits.*

class CartService(
  cartRepo: CartRepository[IO],
  productRepo: ProductRepository[IO]
):
  def addToCart(customerId: CustomerId, productId: ProductId, qty: Int): 
      IO[Either[String, Cart]] =
    for
      cartOpt <- cartRepo.findByCustomer(customerId)
      productOpt <- productRepo.findById(productId)
      result <- (cartOpt, productOpt) match
        case (_, None) => IO.pure(Left(s"Product ${productId.value} not found"))
        case (_, Some(product)) if product.stock < qty =>
          IO.pure(Left(s"Insufficient stock. Available: ${product.stock}"))
        case (cartOpt, Some(_)) =>
          val cart = cartOpt.getOrElse(Cart(customerId, Map.empty, None))
          val currentQty = cart.items.getOrElse(productId, 0)
          val updatedCart = cart.copy(
            items = cart.items.updated(productId, currentQty + qty)
          )
          cartRepo.save(updatedCart).map(Right(_))
    yield result
  
  def checkout(customerId: CustomerId, paymentMethod: PaymentMethod): 
      IO[Either[OrderError, Order]] =
    for
      cartOpt <- cartRepo.findByCustomer(customerId)
      result <- cartOpt match
        case None => IO.pure(Left(OrderError.OrderAlreadyProcessed("Empty cart")))
        case Some(cart) =>
          processCheckout(cart, paymentMethod)
    yield result
  
  private def processCheckout(cart: Cart, payment: PaymentMethod):
      IO[Either[OrderError, Order]] =
    for
      products <- cart.items.keys.toList.traverse(productRepo.findById)
      allFound = products.forall(_.isDefined)
      result <- if !allFound then
        IO.pure(Left(OrderError.InsufficientStock("unknown", 0, 0)))
      else
        val items = cart.items.toList.zip(products.flatten).map {
          case ((productId, qty), product) =>
            OrderItem(productId, qty, product.price)
        }
        val total = items.foldLeft(Price(0)) { (acc, item) =>
          acc + item.price * OrderQuantity.of(item.quantity).getOrElse(OrderQuantity(1))
        }
        val order = Order(
          OrderId(java.util.UUID.randomUUID().toString),
          cart.customerId,
          items,
          OrderStatus.Pending,
          total,
          List.empty
        )
        IO.pure(Right(order))
    yield result

// Infrastructure layer
package infrastructure

import doobie.*
import doobie.implicits.*
import cats.effect.IO

class PostgresProductRepo(xa: Transactor[IO]) extends ProductRepository[IO]:
  def findById(id: ProductId): IO[Option[Product]] =
    sql"""
      SELECT id, name, price_cents, stock, category 
      FROM products WHERE id = ${id.value}
    """.query[(String, String, Long, Int, String)]
      .option
      .transact(xa)
      .map(_.map { (id, name, priceCents, stock, cat) =>
        Product(ProductId(id), name, Price(priceCents), stock, cat)
      })
  
  def findByCategory(category: String): IO[List[Product]] =
    sql"""
      SELECT id, name, price_cents, stock, category 
      FROM products WHERE category = $category AND stock > 0
      ORDER BY name
    """.query[(String, String, Long, Int, String)]
      .to[List]
      .transact(xa)
      .map(_.map { (id, name, priceCents, stock, cat) =>
        Product(ProductId(id), name, Price(priceCents), stock, cat)
      })
  
  def updateStock(id: ProductId, delta: Int): IO[Either[String, Product]] =
    (for
      result <- sql"""
        UPDATE products 
        SET stock = stock + $delta 
        WHERE id = ${id.value} AND stock + $delta >= 0
        RETURNING id, name, price_cents, stock, category
      """.query[(String, String, Long, Int, String)].option
    yield result).transact(xa).map {
      case None => Left(s"Insufficient stock for product ${id.value}")
      case Some((id, name, price, stock, cat)) =>
        Right(Product(ProductId(id), name, Price(price), stock, cat))
    }
  
  def search(query: String): IO[List[Product]] =
    val searchPattern = s"%$query%"
    sql"""
      SELECT id, name, price_cents, stock, category 
      FROM products 
      WHERE name ILIKE $searchPattern OR category ILIKE $searchPattern
      ORDER BY name LIMIT 20
    """.query[(String, String, Long, Int, String)]
      .to[List]
      .transact(xa)
      .map(_.map { (id, name, priceCents, stock, cat) =>
        Product(ProductId(id), name, Price(priceCents), stock, cat)
      })

// HTTP layer ด้วย http4s
package http

import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import io.circe.*
import io.circe.generic.auto.*
import cats.effect.IO

class ProductRoutes(productRepo: ProductRepository[IO]):
  val routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case GET -> Root / "products" / productId =>
      productRepo.findById(ProductId(productId)).flatMap {
        case None => NotFound()
        case Some(product) => Ok(product.asJson)
      }
    
    case GET -> Root / "products" :? CategoryParam(category) =>
      productRepo.findByCategory(category).flatMap(products =>
        Ok(products.asJson)
      )
    
    case GET -> Root / "products" / "search" :? QueryParam(q) =>
      productRepo.search(q).flatMap(results =>
        Ok(results.asJson)
      )
  }
  
  object CategoryParam extends QueryParamDecoderMatcher[String]("category")
  object QueryParam extends QueryParamDecoderMatcher[String]("q")

// Wiring ทุกอย่างเข้าด้วยกัน
package main

import cats.effect.*
import doobie.*
import org.http4s.ember.server.*

object Application extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    for
      // Create database connection
      xa <- IO.pure(Transactor.fromDriverManager[IO](
        "org.postgresql.Driver",
        "jdbc:postgresql://localhost:5432/shop",
        "user", "password"
      ))
      
      // Create repositories
      productRepo = PostgresProductRepo(xa)
      cartRepo = InMemoryCartRepo()  // or PostgresCartRepo
      
      // Create services
      cartService = CartService(cartRepo, productRepo)
      
      // Create HTTP routes
      productRoutes = ProductRoutes(productRepo)
      
      // Start server
      _ <- EmberServerBuilder
        .default[IO]
        .withHost(host"0.0.0.0")
        .withPort(port"8080")
        .withHttpApp(productRoutes.routes.orNotFound)
        .build
        .useForever
    yield ExitCode.Success
```

---

## สรุป

Functional Design Patterns ช่วยสร้าง software ที่:

| Pattern | ประโยชน์ | เมื่อไหร่ใช้ |
|---------|---------|------------|
| Tagless Final | Testable, swappable implementations | Complex business logic |
| Effect Rotation | Simple effect composition | Multi-effect programs |
| Capability-Based | Type-safe permissions | Security-sensitive code |
| ZLayer | Explicit dependency graph | ZIO applications |
| ADT Design | Correct-by-construction | Domain modeling |

### หลักการสำคัญ

```
1. Make illegal states unrepresentable ด้วย ADTs
2. Separate algebra (what) จาก interpreter (how)
3. ใช้ type system เพื่อ enforce invariants
4. Prefer composition over inheritance
5. Pure functions + managed effects
```

---

*[← ส่วนที่ 80: Scala Native](part-80-scala-native.md) | [ส่วนที่ 82: Distributed Streaming Systems →](part-82-streaming-systems.md)*
