# ส่วนที่ 90: Functional Application Architecture

## สารบัญ

1. [Hexagonal Architecture (Ports and Adapters)](#hexagonal-architecture-ports-and-adapters)
2. [Onion Architecture](#onion-architecture)
3. [Clean Architecture ใน Scala](#clean-architecture-ใน-scala)
4. [การจัดระเบียบ Module](#การจัดระเบียบ-module)
5. [Dependency Management](#dependency-management)
6. [Error Handling ข้ามชั้น Layer](#error-handling-ข้ามชั้น-layer)
7. [Complete Layered Application](#complete-layered-application)
8. [สรุป](#สรุป)

---

## Hexagonal Architecture (Ports and Adapters)

Hexagonal Architecture หรือ Ports and Adapters เป็นแนวคิดที่แยก Business Logic ออกจาก Infrastructure โดยใช้ Ports (Interfaces) เป็นตัวกลาง

### แนวคิดหลัก

- **Domain** - ศูนย์กลางของระบบ ไม่ขึ้นกับ Framework ใดๆ
- **Ports** - Interface ที่ Domain กำหนดขึ้น
- **Adapters** - การ implement Ports สำหรับ Infrastructure จริง

### Port Definitions

```scala
// Domain ports - defined by the domain layer
package domain.ports

import domain.model.*

// Primary ports (driving) - สำหรับ Input จากภายนอก
trait UserService[F[_]]:
  def createUser(cmd: CreateUserCommand): F[Either[DomainError, User]]
  def findUser(id: UserId): F[Option[User]]
  def updateUser(id: UserId, cmd: UpdateUserCommand): F[Either[DomainError, User]]
  def deleteUser(id: UserId): F[Either[DomainError, Unit]]

// Secondary ports (driven) - สำหรับ Output ไปยัง Infrastructure
trait UserRepository[F[_]]:
  def save(user: User): F[User]
  def findById(id: UserId): F[Option[User]]
  def findByEmail(email: Email): F[Option[User]]
  def delete(id: UserId): F[Boolean]
  def findAll(page: Int, size: Int): F[List[User]]

trait EmailNotifier[F[_]]:
  def sendWelcomeEmail(user: User): F[Unit]
  def sendPasswordResetEmail(email: Email, token: ResetToken): F[Unit]

trait EventPublisher[F[_]]:
  def publish[E: EventEncoder](event: E): F[Unit]
```

### Domain Model

```scala
package domain.model

import java.time.Instant
import java.util.UUID

opaque type UserId = UUID
object UserId:
  def apply(value: UUID): UserId = value
  def generate(): UserId = UUID.randomUUID()
  extension (id: UserId) def value: UUID = id

opaque type Email = String
object Email:
  def apply(value: String): Either[String, Email] =
    if value.contains("@") && value.length > 3 then Right(value)
    else Left(s"Invalid email: $value")
  extension (email: Email) def value: String = email

case class User(
  id: UserId,
  name: String,
  email: Email,
  createdAt: Instant,
  updatedAt: Instant
)

case class CreateUserCommand(name: String, email: String)
case class UpdateUserCommand(name: Option[String] = None, email: Option[String] = None)

sealed trait DomainError
case class UserNotFound(id: UserId) extends DomainError
case class EmailAlreadyExists(email: Email) extends DomainError
case class InvalidInput(message: String) extends DomainError
case class DatabaseError(cause: Throwable) extends DomainError
```

### Domain Service Implementation

```scala
package domain.service

import cats.Monad
import cats.syntax.all.*
import domain.model.*
import domain.ports.*

class UserServiceImpl[F[_]: Monad](
  repo: UserRepository[F],
  notifier: EmailNotifier[F],
  publisher: EventPublisher[F]
) extends UserService[F]:

  def createUser(cmd: CreateUserCommand): F[Either[DomainError, User]] =
    Email(cmd.email) match
      case Left(err) =>
        Monad[F].pure(Left(InvalidInput(err)))
      case Right(email) =>
        for
          existing <- repo.findByEmail(email)
          result <- existing match
            case Some(_) =>
              Monad[F].pure(Left(EmailAlreadyExists(email)))
            case None =>
              val user = User(
                id = UserId.generate(),
                name = cmd.name,
                email = email,
                createdAt = java.time.Instant.now(),
                updatedAt = java.time.Instant.now()
              )
              for
                saved    <- repo.save(user)
                _        <- notifier.sendWelcomeEmail(saved)
                _        <- publisher.publish(UserCreatedEvent(saved.id, saved.email))
              yield Right(saved)
        yield result

  def findUser(id: UserId): F[Option[User]] =
    repo.findById(id)

  def updateUser(id: UserId, cmd: UpdateUserCommand): F[Either[DomainError, User]] =
    repo.findById(id).flatMap:
      case None => Monad[F].pure(Left(UserNotFound(id)))
      case Some(user) =>
        val updated = user.copy(
          name = cmd.name.getOrElse(user.name),
          updatedAt = java.time.Instant.now()
        )
        repo.save(updated).map(Right(_))

  def deleteUser(id: UserId): F[Either[DomainError, Unit]] =
    repo.delete(id).map:
      case true  => Right(())
      case false => Left(UserNotFound(id))
```

### Infrastructure Adapters

```scala
package infrastructure.database

import cats.effect.IO
import skunk.*
import skunk.implicits.*
import skunk.codec.all.*
import domain.model.*
import domain.ports.*

class PostgresUserRepository(pool: Resource[IO, Session[IO]]) extends UserRepository[IO]:

  private val userDecoder: Decoder[User] =
    (uuid *: varchar *: varchar *: timestamptz *: timestamptz).map:
      case (id, name, email, createdAt, updatedAt) =>
        User(
          id = UserId(id),
          name = name,
          email = Email(email).getOrElse(throw new RuntimeException("Invalid DB data")),
          createdAt = createdAt.toInstant,
          updatedAt = updatedAt.toInstant
        )

  private val findByIdQuery: Query[UUID, User] =
    sql"""
      SELECT id, name, email, created_at, updated_at
      FROM users
      WHERE id = $uuid
    """.query(userDecoder)

  def findById(id: UserId): IO[Option[User]] =
    pool.use: session =>
      session.option(findByIdQuery)(id.value)

  def save(user: User): IO[User] =
    pool.use: session =>
      val command: Command[User] =
        sql"""
          INSERT INTO users (id, name, email, created_at, updated_at)
          VALUES ($uuid, $varchar, $varchar, $timestamptz, $timestamptz)
          ON CONFLICT (id) DO UPDATE SET
            name = EXCLUDED.name,
            email = EXCLUDED.email,
            updated_at = EXCLUDED.updated_at
        """.command.contramap: u =>
          u.id.value *: u.name *: u.email.value *: u.createdAt *: u.updatedAt *: EmptyTuple
      session.execute(command)(user).map(_ => user)

  def findByEmail(email: Email): IO[Option[User]] =
    pool.use: session =>
      val query: Query[String, User] =
        sql"SELECT id, name, email, created_at, updated_at FROM users WHERE email = $varchar"
          .query(userDecoder)
      session.option(query)(email.value)

  def delete(id: UserId): IO[Boolean] =
    pool.use: session =>
      val command: Command[UUID] =
        sql"DELETE FROM users WHERE id = $uuid".command
      session.execute(command)(id.value).map(_.rowsAffected > 0)

  def findAll(page: Int, size: Int): IO[List[User]] =
    pool.use: session =>
      val query: Query[Int *: Int *: EmptyTuple, User] =
        sql"SELECT id, name, email, created_at, updated_at FROM users LIMIT $int4 OFFSET $int4"
          .query(userDecoder)
      session.execute(query)(size *: (page * size) *: EmptyTuple)
```

---

## Onion Architecture

Onion Architecture จัดชั้นแบบ concentric circles โดย inner layers ไม่รู้จัก outer layers

### Layer Structure

```
┌─────────────────────────────────────┐
│         Infrastructure              │
│   ┌─────────────────────────────┐   │
│   │      Application Layer      │   │
│   │   ┌─────────────────────┐   │   │
│   │   │   Domain Services   │   │   │
│   │   │  ┌───────────────┐  │   │   │
│   │   │  │ Domain Model  │  │   │   │
│   │   │  └───────────────┘  │   │   │
│   │   └─────────────────────┘   │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

```scala
// Domain Model Layer - ไม่มี dependency ภายนอก
package onion.domain.model

case class Order(
  id: OrderId,
  customerId: CustomerId,
  items: List[OrderItem],
  status: OrderStatus,
  total: BigDecimal
)

enum OrderStatus:
  case Pending, Confirmed, Shipped, Delivered, Cancelled

case class OrderItem(productId: ProductId, quantity: Int, price: BigDecimal)

// Domain Services Layer - ขึ้นกับ Domain Model เท่านั้น
package onion.domain.service

trait OrderDomainService:
  def calculateTotal(items: List[OrderItem]): BigDecimal =
    items.map(i => i.price * i.quantity).sum

  def validateOrder(order: Order): Either[String, Order] =
    if order.items.isEmpty then Left("Order must have at least one item")
    else if order.items.exists(_.quantity <= 0) then Left("Quantity must be positive")
    else Right(order)

// Application Layer - orchestrates domain objects
package onion.application

import cats.effect.IO
import onion.domain.model.*
import onion.domain.service.*

trait OrderRepository:
  def save(order: Order): IO[Order]
  def findById(id: OrderId): IO[Option[Order]]

trait PaymentGateway:
  def charge(customerId: CustomerId, amount: BigDecimal): IO[PaymentResult]

class OrderApplicationService(
  orderRepo: OrderRepository,
  paymentGateway: PaymentGateway,
  domainService: OrderDomainService
):
  def placeOrder(customerId: CustomerId, items: List[OrderItem]): IO[Either[String, Order]] =
    val draft = Order(
      id = OrderId.generate(),
      customerId = customerId,
      items = items,
      status = OrderStatus.Pending,
      total = domainService.calculateTotal(items)
    )
    domainService.validateOrder(draft) match
      case Left(err) => IO.pure(Left(err))
      case Right(validOrder) =>
        for
          payResult <- paymentGateway.charge(customerId, validOrder.total)
          result <- payResult match
            case PaymentResult.Success =>
              val confirmed = validOrder.copy(status = OrderStatus.Confirmed)
              orderRepo.save(confirmed).map(Right(_))
            case PaymentResult.Failure(reason) =>
              IO.pure(Left(s"Payment failed: $reason"))
        yield result
```

---

## Clean Architecture ใน Scala

Clean Architecture เน้น Dependency Rule: dependencies ต้องชี้เข้าด้านใน (inward)

```scala
// Use Cases - Application Business Rules
package clean.usecases

import cats.effect.IO

// Use case interface
trait CreateProductUseCase:
  def execute(request: CreateProductRequest): IO[CreateProductResponse]

// Use case data structures (no domain leakage)
case class CreateProductRequest(
  name: String,
  description: String,
  price: Double,
  stock: Int
)

case class CreateProductResponse(
  id: String,
  name: String,
  price: Double,
  createdAt: String
)

// Use case implementation
class CreateProductUseCaseImpl(
  productRepo: ProductRepository,
  eventBus: EventBus
) extends CreateProductUseCase:

  def execute(request: CreateProductRequest): IO[CreateProductResponse] =
    for
      product <- IO:
        Product(
          id = ProductId.generate(),
          name = request.name,
          description = request.description,
          price = Money(request.price),
          stock = Stock(request.stock)
        )
      saved    <- productRepo.save(product)
      _        <- eventBus.emit(ProductCreated(saved.id))
    yield CreateProductResponse(
      id = saved.id.value.toString,
      name = saved.name,
      price = saved.price.amount,
      createdAt = saved.createdAt.toString
    )

// Interface Adapters - Controllers
package clean.adapters.http

import cats.effect.IO
import org.http4s.*
import org.http4s.dsl.io.*
import io.circe.generic.auto.*
import org.http4s.circe.*
import clean.usecases.*

object ProductController:

  case class CreateProductHttpRequest(name: String, description: String, price: Double, stock: Int)

  def routes(createProduct: CreateProductUseCase): HttpRoutes[IO] =
    HttpRoutes.of[IO]:
      case req @ POST -> Root / "products" =>
        for
          body     <- req.as[CreateProductHttpRequest]
          ucReq    = CreateProductRequest(body.name, body.description, body.price, body.stock)
          response <- createProduct.execute(ucReq)
          result   <- Ok(response)
        yield result

// Frameworks & Drivers - Database Adapter
package clean.frameworks.database

import cats.effect.IO
import doobie.*
import doobie.implicits.*
import clean.usecases.ProductRepository

class DoobieProductRepository(xa: Transactor[IO]) extends ProductRepository:

  def save(product: Product): IO[Product] =
    sql"""
      INSERT INTO products (id, name, description, price, stock, created_at)
      VALUES (${product.id.value}, ${product.name}, ${product.description},
              ${product.price.amount}, ${product.stock.value}, ${product.createdAt})
    """.update.run.transact(xa).map(_ => product)

  def findById(id: ProductId): IO[Option[Product]] =
    sql"SELECT id, name, description, price, stock, created_at FROM products WHERE id = ${id.value}"
      .query[ProductRow]
      .option
      .transact(xa)
      .map(_.map(_.toDomain))
```

---

## การจัดระเบียบ Module

### Multi-Module SBT Project

```scala
// build.sbt
lazy val root = project
  .in(file("."))
  .aggregate(domain, application, infrastructure, api)

lazy val domain = project
  .in(file("modules/domain"))
  .settings(
    name := "myapp-domain",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-core" % "2.10.0"
    )
  )

lazy val application = project
  .in(file("modules/application"))
  .dependsOn(domain)
  .settings(
    name := "myapp-application",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-effect" % "3.5.4"
    )
  )

lazy val infrastructure = project
  .in(file("modules/infrastructure"))
  .dependsOn(application)
  .settings(
    name := "myapp-infrastructure",
    libraryDependencies ++= Seq(
      "org.tpolecat" %% "skunk-core"   % "0.6.4",
      "com.github.fd4s" %% "fs2-kafka" % "3.5.1"
    )
  )

lazy val api = project
  .in(file("modules/api"))
  .dependsOn(infrastructure)
  .settings(
    name := "myapp-api",
    libraryDependencies ++= Seq(
      "org.http4s" %% "http4s-ember-server" % "0.23.27"
    )
  )
```

### Package Organization

```scala
// modules/domain/src/main/scala/
// com/example/domain/
//   model/       - Domain entities and value objects
//   service/     - Domain services
//   repository/  - Repository interfaces (ports)
//   event/       - Domain events
//   error/       - Domain-specific errors

package com.example.domain.model

// Value Objects ใช้ opaque types
opaque type ProductId = java.util.UUID
opaque type Money = BigDecimal
opaque type Stock = Int

object ProductId:
  def apply(v: java.util.UUID): ProductId = v
  def generate(): ProductId = java.util.UUID.randomUUID()
  extension (id: ProductId)
    def value: java.util.UUID = id
    def show: String = id.toString

object Money:
  def apply(amount: BigDecimal): Either[String, Money] =
    if amount >= 0 then Right(amount)
    else Left("Money cannot be negative")
  def unsafe(amount: BigDecimal): Money = amount
  extension (m: Money)
    def amount: BigDecimal = m
    def +(other: Money): Money = m + other
    def *(factor: Int): Money = m * factor

object Stock:
  def apply(quantity: Int): Either[String, Stock] =
    if quantity >= 0 then Right(quantity)
    else Left("Stock cannot be negative")
  extension (s: Stock)
    def value: Int = s
    def decrease(qty: Int): Either[String, Stock] =
      if s - qty >= 0 then Right(s - qty)
      else Left("Insufficient stock")
```

---

## Dependency Management

### ZIO-style Dependency Injection

```scala
import zio.*
import zio.config.*

// Service Definitions
trait Database:
  def query[A](sql: String): Task[List[A]]

trait Cache:
  def get(key: String): Task[Option[String]]
  def set(key: String, value: String, ttl: Duration): Task[Unit]

trait Logger:
  def info(msg: String): UIO[Unit]
  def error(msg: String, cause: Throwable): UIO[Unit]

// Live Implementations
object DatabaseLive:
  val layer: ZLayer[DatabaseConfig, Throwable, Database] =
    ZLayer.scoped:
      for
        config <- ZIO.service[DatabaseConfig]
        pool   <- ZIO.acquireRelease(
          ZIO.attempt(createConnectionPool(config))
        )(pool => ZIO.succeed(pool.close()))
      yield new Database:
        def query[A](sql: String): Task[List[A]] =
          ZIO.attempt(pool.execute(sql).as[List[A]])

object CacheLive:
  val layer: ZLayer[RedisConfig, Throwable, Cache] =
    ZLayer.scoped:
      for
        config <- ZIO.service[RedisConfig]
        redis  <- ZIO.acquireRelease(
          ZIO.attempt(RedisClient.connect(config.url))
        )(client => ZIO.succeed(client.close()))
      yield new Cache:
        def get(key: String): Task[Option[String]] =
          ZIO.attempt(Option(redis.get(key)))
        def set(key: String, value: String, ttl: Duration): Task[Unit] =
          ZIO.attempt(redis.setex(key, ttl.getSeconds, value)).unit

// Application with dependency injection
object UserProgram extends ZIOAppDefault:

  val program: ZIO[UserService & Logger, Throwable, Unit] =
    for
      service <- ZIO.service[UserService]
      logger  <- ZIO.service[Logger]
      users   <- service.findAll()
      _       <- logger.info(s"Found ${users.length} users")
    yield ()

  val appLayer: ZLayer[Any, Throwable, UserService & Logger] =
    val config = ZLayer.succeed(AppConfig.default)
    val db     = config >>> DatabaseLive.layer
    val cache  = config >>> CacheLive.layer
    val logger = ZLayer.succeed(ConsoleLogger())
    val repo   = (db ++ cache) >>> UserRepositoryLive.layer
    (repo ++ logger) >>> UserServiceLive.layer ++ logger

  def run: Task[Unit] =
    program.provide(appLayer)
```

### Cats Effect Resource Management

```scala
import cats.effect.*
import cats.effect.std.Console

// Resource Graph สำหรับ Application Dependencies
object AppResources:

  case class Resources(
    database: DatabasePool,
    cache: RedisConnection,
    httpClient: org.http4s.client.Client[IO],
    kafka: KafkaProducer[IO, String, String]
  )

  def make(config: AppConfig): Resource[IO, Resources] =
    for
      db         <- DatabasePool.resource(config.database)
      cache      <- RedisConnection.resource(config.redis)
      httpClient <- org.http4s.ember.client.EmberClientBuilder
                      .default[IO]
                      .build
      kafka      <- KafkaProducer.resource[IO, String, String](config.kafka)
    yield Resources(db, cache, httpClient, kafka)

// Wiring everything together
object Main extends IOApp:

  def run(args: List[String]): IO[ExitCode] =
    AppConfig.load.flatMap: config =>
      AppResources.make(config).use: resources =>
        val userRepo    = PostgresUserRepository(resources.database)
        val userCache   = RedisUserCache(resources.cache)
        val emailClient = SmtpEmailClient(resources.httpClient)
        val eventBus    = KafkaEventBus(resources.kafka)
        val userService = UserServiceImpl(userRepo, userCache, emailClient, eventBus)
        val httpApp     = UserController.routes(userService).orNotFound
        EmberServerBuilder
          .default[IO]
          .withHost(config.server.host)
          .withPort(config.server.port)
          .withHttpApp(httpApp)
          .build
          .use(_ => IO.never)
          .as(ExitCode.Success)
```

---

## Error Handling ข้ามชั้น Layer

### Error Hierarchy

```scala
// Domain errors
sealed trait DomainError:
  def message: String

object DomainError:
  case class NotFound(entityType: String, id: String) extends DomainError:
    def message = s"$entityType with id $id not found"

  case class ValidationFailed(field: String, reason: String) extends DomainError:
    def message = s"Validation failed for $field: $reason"

  case class BusinessRuleViolation(rule: String) extends DomainError:
    def message = s"Business rule violated: $rule"

// Application errors (wraps domain + infra errors)
sealed trait AppError
object AppError:
  case class Domain(error: DomainError) extends AppError
  case class Infrastructure(cause: Throwable) extends AppError
  case class Unauthorized(reason: String) extends AppError
  case class Conflict(message: String) extends AppError

// HTTP error responses
sealed trait HttpError:
  def statusCode: Int
  def message: String

object HttpError:
  case class NotFound(message: String) extends HttpError:
    val statusCode = 404
  case class BadRequest(message: String) extends HttpError:
    val statusCode = 400
  case class InternalServerError(message: String) extends HttpError:
    val statusCode = 500
  case class Unauthorized(message: String) extends HttpError:
    val statusCode = 401

// Error translation between layers
object ErrorTranslator:
  def domainToApp(error: DomainError): AppError =
    error match
      case DomainError.NotFound(_, _)          => AppError.Domain(error)
      case DomainError.ValidationFailed(_, _)  => AppError.Domain(error)
      case DomainError.BusinessRuleViolation(_) => AppError.Domain(error)

  def appToHttp(error: AppError): HttpError =
    error match
      case AppError.Domain(DomainError.NotFound(t, id)) =>
        HttpError.NotFound(s"$t '$id' not found")
      case AppError.Domain(DomainError.ValidationFailed(f, r)) =>
        HttpError.BadRequest(s"Invalid $f: $r")
      case AppError.Domain(DomainError.BusinessRuleViolation(rule)) =>
        HttpError.BadRequest(rule)
      case AppError.Infrastructure(cause) =>
        HttpError.InternalServerError("An internal error occurred")
      case AppError.Unauthorized(reason) =>
        HttpError.Unauthorized(reason)
      case AppError.Conflict(msg) =>
        HttpError.BadRequest(msg)
```

### Error Handling with EitherT

```scala
import cats.data.EitherT
import cats.effect.IO

type AppResult[A] = EitherT[IO, AppError, A]

class OrderService(
  orderRepo: OrderRepository,
  inventoryService: InventoryService,
  paymentService: PaymentService
):
  def placeOrder(request: PlaceOrderRequest): AppResult[Order] =
    for
      // Validate input
      cmd <- EitherT.fromEither[IO]:
        validatePlaceOrderRequest(request).left.map(AppError.Domain.apply)

      // Check inventory
      _ <- EitherT:
        inventoryService.checkAvailability(cmd.items)
          .map(_.left.map(e => AppError.Domain(e)))
          .handleError(t => Left(AppError.Infrastructure(t)))

      // Process payment
      payment <- EitherT:
        paymentService.charge(cmd.customerId, cmd.totalAmount)
          .map(_.left.map(e => AppError.Domain(e)))
          .handleError(t => Left(AppError.Infrastructure(t)))

      // Create order
      order <- EitherT:
        orderRepo.create(cmd, payment.id)
          .map(Right(_))
          .handleError(t => Left(AppError.Infrastructure(t)))
    yield order

  private def validatePlaceOrderRequest(req: PlaceOrderRequest): Either[DomainError, CreateOrderCmd] =
    for
      items <- req.items.traverse: item =>
        if item.quantity > 0 then Right(item)
        else Left(DomainError.ValidationFailed("quantity", "must be positive"))
    yield CreateOrderCmd(req.customerId, items)
```

---

## Complete Layered Application

### Project Structure

```
myapp/
├── build.sbt
├── modules/
│   ├── domain/
│   │   └── src/main/scala/com/example/domain/
│   │       ├── model/
│   │       │   ├── User.scala
│   │       │   └── Product.scala
│   │       ├── repository/
│   │       │   └── UserRepository.scala
│   │       └── service/
│   │           └── UserDomainService.scala
│   ├── application/
│   │   └── src/main/scala/com/example/application/
│   │       ├── usecase/
│   │       │   ├── CreateUserUseCase.scala
│   │       │   └── GetUserUseCase.scala
│   │       └── dto/
│   │           └── UserDto.scala
│   ├── infrastructure/
│   │   └── src/main/scala/com/example/infrastructure/
│   │       ├── database/
│   │       │   └── DoobieUserRepository.scala
│   │       └── email/
│   │           └── SendgridEmailService.scala
│   └── api/
│       └── src/main/scala/com/example/api/
│           ├── routes/
│           │   └── UserRoutes.scala
│           └── Main.scala
```

### Full Application Wiring

```scala
package com.example.api

import cats.effect.*
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s.*
import com.example.infrastructure.database.DatabaseModule
import com.example.infrastructure.email.EmailModule
import com.example.application.usecase.*
import com.example.api.routes.*

object Main extends IOApp:

  def run(args: List[String]): IO[ExitCode] =
    AppConfig.load[IO].flatMap: config =>
      (
        DatabaseModule.transactor(config.database),
        EmailModule.client(config.email)
      ).mapN: (xa, emailClient) =>
        // Wire up infrastructure
        val userRepo   = DoobieUserRepository(xa)
        val emailSvc   = SendgridEmailService(emailClient, config.email)

        // Wire up use cases
        val createUser = CreateUserUseCase(userRepo, emailSvc)
        val getUser    = GetUserUseCase(userRepo)
        val updateUser = UpdateUserUseCase(userRepo)
        val deleteUser = DeleteUserUseCase(userRepo)

        // Wire up HTTP routes
        UserRoutes(createUser, getUser, updateUser, deleteUser).routes
      .use: routes =>
        EmberServerBuilder
          .default[IO]
          .withHost(config.server.host)
          .withPort(config.server.port)
          .withHttpApp(routes.orNotFound)
          .build
          .use(_ => IO.never)
          .as(ExitCode.Success)

// Application Config
case class AppConfig(
  database: DatabaseConfig,
  email: EmailConfig,
  server: ServerConfig
)

object AppConfig:
  import pureconfig.*
  import pureconfig.generic.derivation.default.*

  given ConfigReader[AppConfig] = ConfigReader.derived
  given ConfigReader[DatabaseConfig] = ConfigReader.derived
  given ConfigReader[EmailConfig] = ConfigReader.derived
  given ConfigReader[ServerConfig] = ConfigReader.derived

  def load[F[_]: Sync]: F[AppConfig] =
    Sync[F].delay:
      ConfigSource.default.loadOrThrow[AppConfig]
```

### HTTP Routes with Middleware

```scala
package com.example.api.routes

import cats.effect.IO
import cats.syntax.all.*
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import org.http4s.circe.CirceEntityCodec.*
import io.circe.generic.auto.*
import io.circe.syntax.*
import com.example.application.usecase.*
import com.example.api.middleware.*

object UserRoutes:
  def apply(
    createUser: CreateUserUseCase,
    getUser: GetUserUseCase,
    updateUser: UpdateUserUseCase,
    deleteUser: DeleteUserUseCase
  ): UserRoutes =
    new UserRoutes(createUser, getUser, updateUser, deleteUser)

class UserRoutes(
  createUser: CreateUserUseCase,
  getUser: GetUserUseCase,
  updateUser: UpdateUserUseCase,
  deleteUser: DeleteUserUseCase
):
  case class CreateUserBody(name: String, email: String)
  case class UpdateUserBody(name: Option[String], email: Option[String])

  def routes: HttpRoutes[IO] =
    HttpRoutes.of[IO]:
      case req @ POST -> Root / "users" =>
        req.as[CreateUserBody].flatMap: body =>
          createUser.execute(CreateUserRequest(body.name, body.email)).flatMap:
            case Right(user) => Created(user.asJson)
            case Left(error) => BadRequest(ErrorResponse(error.message).asJson)

      case GET -> Root / "users" / UUIDVar(id) =>
        getUser.execute(id.toString).flatMap:
          case Some(user) => Ok(user.asJson)
          case None       => NotFound()

      case req @ PUT -> Root / "users" / UUIDVar(id) =>
        req.as[UpdateUserBody].flatMap: body =>
          updateUser.execute(id.toString, UpdateUserRequest(body.name, body.email)).flatMap:
            case Right(user) => Ok(user.asJson)
            case Left(error) => BadRequest(ErrorResponse(error.message).asJson)

      case DELETE -> Root / "users" / UUIDVar(id) =>
        deleteUser.execute(id.toString).flatMap:
          case Right(_)    => NoContent()
          case Left(error) => NotFound(ErrorResponse(error.message).asJson)

case class ErrorResponse(message: String)
```

### Integration Test

```scala
package com.example.test

import cats.effect.IO
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.freespec.AsyncFreeSpec
import org.http4s.*
import org.http4s.implicits.*
import org.http4s.client.dsl.io.*
import io.circe.syntax.*

class UserRoutesSpec extends AsyncFreeSpec with AsyncIOSpec with Matchers:

  "UserRoutes" - {
    "POST /users" - {
      "should create a user and return 201" in {
        val fakeRepo    = new InMemoryUserRepository()
        val fakeEmail   = new FakeEmailService()
        val createUser  = CreateUserUseCase(fakeRepo, fakeEmail)
        val getUser     = GetUserUseCase(fakeRepo)
        val routes      = UserRoutes(createUser, getUser, ???, ???).routes.orNotFound
        val body        = """{"name":"Alice","email":"alice@example.com"}"""
        val request     = Request[IO](Method.POST, uri"/users")
                            .withBody(body)
                            .withContentType(MediaType.application.json)
        request.flatMap(routes.run).asserting: response =>
          response.status shouldBe Status.Created
      }
    }
  }

  class InMemoryUserRepository extends UserRepository:
    private var store: Map[String, User] = Map.empty
    def save(user: User): IO[User] = IO { store = store + (user.id -> user); user }
    def findById(id: String): IO[Option[User]] = IO.pure(store.get(id))
    def findByEmail(email: String): IO[Option[User]] =
      IO.pure(store.values.find(_.email == email))
    def delete(id: String): IO[Boolean] =
      IO { val had = store.contains(id); store = store - id; had }
    def findAll(page: Int, size: Int): IO[List[User]] =
      IO.pure(store.values.drop(page * size).take(size).toList)
```

---

## สรุป

Functional Application Architecture ใน Scala ช่วยให้:

- **Hexagonal Architecture** แยก Business Logic ออกจาก Infrastructure ด้วย Ports/Adapters
- **Onion Architecture** จัดชั้น dependencies ให้ชี้เข้าด้านใน
- **Clean Architecture** รักษา Dependency Rule อย่างเข้มงวด
- **Module Organization** แยก concerns ชัดเจนผ่าน multi-module projects
- **Dependency Management** ใช้ ZIO Layers หรือ Cats Effect Resource
- **Error Handling** ส่ง typed errors ข้ามชั้น layer ด้วย EitherT/ZIO

---

*[← ส่วนที่ 89](part-89-advanced-patterns.md) | [ส่วนที่ 91: Blockchain กับ Scala →](part-91-blockchain.md)*
