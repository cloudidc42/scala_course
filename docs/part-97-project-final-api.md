# ส่วนที่ 97: Final Project - Complete REST API

## สารบัญ

- [1. Architecture Overview](#1-architecture-overview)
- [2. Project Setup](#2-project-setup)
- [3. Domain Models](#3-domain-models)
- [4. Database Layer ด้วย Doobie](#4-database-layer-ด้วย-doobie)
- [5. Redis Caching Layer](#5-redis-caching-layer)
- [6. Authentication และ Authorization](#6-authentication-และ-authorization)
- [7. HTTP API ด้วย http4s](#7-http-api-ด้วย-http4s)
- [8. Kafka Event Publishing](#8-kafka-event-publishing)
- [9. Docker Deployment](#9-docker-deployment)
- [10. Integration Tests](#10-integration-tests)
- [สรุป](#สรุป)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  HTTP Client                     │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│              http4s HTTP Server                  │
│  ┌─────────────────────────────────────────────┐ │
│  │  Auth Middleware → Rate Limiting             │ │
│  └─────────────────────────────────────────────┘ │
│  ┌─────────────┐ ┌────────────┐ ┌─────────────┐ │
│  │ Auth Routes │ │  Products  │ │   Orders    │ │
│  └──────┬──────┘ └─────┬──────┘ └──────┬──────┘ │
└─────────┼──────────────┼───────────────┼─────────┘
          │              │               │
┌─────────▼──────────────▼───────────────▼─────────┐
│                  Service Layer                    │
│  AuthService  │  ProductService  │  OrderService  │
└─────────┬──────────────┬───────────────┬──────────┘
          │         ┌────┤               │
┌─────────▼────┐ ┌──▼───▼──┐ ┌──────────▼──────────┐
│  PostgreSQL  │ │  Redis  │ │    Kafka Producer    │
│  (Doobie)   │ │  Cache  │ │  (Event Publishing)  │
└─────────────┘ └─────────┘ └─────────────────────┘
```

---

## 2. Project Setup

### build.sbt

```scala
// build.sbt
val ScalaVersion  = "3.3.1"
val Http4sVersion = "0.23.23"
val DoobieVersion = "1.0.0-RC4"
val CirceVersion  = "0.14.6"
val CatsEffect    = "3.5.2"
val Fs2Version    = "3.9.3"

lazy val root = project
  .in(file("."))
  .settings(
    name         := "scala-rest-api",
    version      := "0.1.0",
    scalaVersion := ScalaVersion,
    
    libraryDependencies ++= Seq(
      // HTTP
      "org.http4s" %% "http4s-ember-server" % Http4sVersion,
      "org.http4s" %% "http4s-ember-client" % Http4sVersion,
      "org.http4s" %% "http4s-circe"        % Http4sVersion,
      "org.http4s" %% "http4s-dsl"          % Http4sVersion,
      
      // JSON
      "io.circe" %% "circe-core"    % CirceVersion,
      "io.circe" %% "circe-generic" % CirceVersion,
      "io.circe" %% "circe-parser"  % CirceVersion,
      
      // Database
      "org.tpolecat" %% "doobie-core"     % DoobieVersion,
      "org.tpolecat" %% "doobie-hikari"   % DoobieVersion,
      "org.tpolecat" %% "doobie-postgres" % DoobieVersion,
      
      // Redis
      "dev.profunktor" %% "redis4cats-effects" % "1.5.2",
      "dev.profunktor" %% "redis4cats-log4cats" % "1.5.2",
      
      // Kafka
      "com.github.fd4s" %% "fs2-kafka" % "3.1.0",
      
      // Auth
      "com.github.jwt-scala" %% "jwt-circe" % "10.0.1",
      "org.mindrot" % "jbcrypt" % "0.4",
      
      // Config
      "com.github.pureconfig" %% "pureconfig-core"       % "0.17.4",
      "com.github.pureconfig" %% "pureconfig-cats-effect" % "0.17.4",
      
      // Logging
      "org.typelevel" %% "log4cats-core"  % "2.6.0",
      "org.typelevel" %% "log4cats-slf4j" % "2.6.0",
      "ch.qos.logback" % "logback-classic" % "1.4.11",
      
      // Testing
      "org.typelevel" %% "cats-effect-testing-scalatest" % "1.5.0" % Test,
      "org.scalatest" %% "scalatest"                     % "3.2.17" % Test,
      "org.http4s"    %% "http4s-client"                 % Http4sVersion % Test,
    )
  )
```

### Application Configuration

```scala
// src/main/scala/config/AppConfig.scala
import pureconfig.*
import pureconfig.generic.derivation.default.*

case class ServerConfig(host: String, port: Int) derives ConfigReader
case class DatabaseConfig(
  url: String,
  user: String,
  password: String,
  maxConnections: Int
) derives ConfigReader
case class RedisConfig(uri: String) derives ConfigReader
case class KafkaConfig(bootstrapServers: String, topic: String) derives ConfigReader
case class JwtConfig(secret: String, expirationHours: Int) derives ConfigReader

case class AppConfig(
  server: ServerConfig,
  database: DatabaseConfig,
  redis: RedisConfig,
  kafka: KafkaConfig,
  jwt: JwtConfig
) derives ConfigReader

object AppConfig:
  def load: IO[AppConfig] =
    IO(ConfigSource.default.loadOrThrow[AppConfig])
```

### application.conf

```hocon
# src/main/resources/application.conf
server {
  host = "0.0.0.0"
  host = ${?SERVER_HOST}
  port = 8080
  port = ${?SERVER_PORT}
}

database {
  url = "jdbc:postgresql://localhost:5432/shopdb"
  url = ${?DATABASE_URL}
  user = "postgres"
  user = ${?DATABASE_USER}
  password = "password"
  password = ${?DATABASE_PASSWORD}
  max-connections = 10
}

redis {
  uri = "redis://localhost:6379"
  uri = ${?REDIS_URI}
}

kafka {
  bootstrap-servers = "localhost:9092"
  bootstrap-servers = ${?KAFKA_BOOTSTRAP_SERVERS}
  topic = "shop-events"
}

jwt {
  secret = "super-secret-key-change-in-production"
  secret = ${?JWT_SECRET}
  expiration-hours = 24
}
```

---

## 3. Domain Models

```scala
// src/main/scala/domain/Models.scala
import java.time.Instant
import java.util.UUID

// User domain
case class UserId(value: UUID) extends AnyVal
case class User(
  id: UserId,
  username: String,
  email: String,
  passwordHash: String,
  role: UserRole,
  createdAt: Instant
)

enum UserRole:
  case Admin, Customer

// Product domain
case class ProductId(value: UUID) extends AnyVal
case class CategoryId(value: UUID) extends AnyVal

case class Category(
  id: CategoryId,
  name: String,
  description: Option[String]
)

case class Product(
  id: ProductId,
  name: String,
  description: Option[String],
  price: BigDecimal,
  stock: Int,
  categoryId: CategoryId,
  createdAt: Instant,
  updatedAt: Instant
)

// Order domain
case class OrderId(value: UUID) extends AnyVal
case class OrderItem(productId: ProductId, quantity: Int, price: BigDecimal)

enum OrderStatus:
  case Pending, Confirmed, Shipped, Delivered, Cancelled

case class Order(
  id: OrderId,
  userId: UserId,
  items: List[OrderItem],
  totalAmount: BigDecimal,
  status: OrderStatus,
  createdAt: Instant,
  updatedAt: Instant
)

// Request/Response DTOs
case class CreateProductRequest(
  name: String,
  description: Option[String],
  price: BigDecimal,
  stock: Int,
  categoryId: String
)

case class UpdateProductRequest(
  name: Option[String],
  description: Option[Option[String]],
  price: Option[BigDecimal],
  stock: Option[Int]
)

case class RegisterRequest(username: String, email: String, password: String)
case class LoginRequest(email: String, password: String)
case class AuthResponse(token: String, userId: String, role: String)

// Error types
sealed trait AppError extends Exception
case class NotFoundError(message: String) extends AppError
case class ValidationError(message: String) extends AppError
case class AuthError(message: String) extends AppError
case class DatabaseError(message: String, cause: Throwable) extends AppError
case class ConflictError(message: String) extends AppError
```

---

## 4. Database Layer ด้วย Doobie

### Database Setup

```scala
// src/main/scala/db/Database.scala
import cats.effect.*
import doobie.*
import doobie.hikari.*
import org.flywaydb.core.Flyway

object Database:
  def transactor(config: DatabaseConfig): Resource[IO, HikariTransactor[IO]] =
    HikariTransactor.newHikariTransactor[IO](
      driverClassName = "org.postgresql.Driver",
      url             = config.url,
      user            = config.user,
      pass            = config.password,
      connectEC       = scala.concurrent.ExecutionContext.global
    ).evalTap { xa =>
      xa.configure(_.setMaximumPoolSize(config.maxConnections))
    }
  
  def migrate(config: DatabaseConfig): IO[Unit] = IO {
    Flyway.configure()
      .dataSource(config.url, config.user, config.password)
      .locations("classpath:db/migration")
      .load()
      .migrate()
  }.void
```

### SQL Migrations

```sql
-- src/main/resources/db/migration/V1__create_tables.sql
CREATE TABLE IF NOT EXISTS users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username     VARCHAR(50) UNIQUE NOT NULL,
    email        VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role         VARCHAR(20) NOT NULL DEFAULT 'Customer',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS categories (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) UNIQUE NOT NULL,
    description TEXT
);

CREATE TABLE IF NOT EXISTS products (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    description TEXT,
    price       DECIMAL(10, 2) NOT NULL,
    stock       INTEGER NOT NULL DEFAULT 0,
    category_id UUID NOT NULL REFERENCES categories(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS orders (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL REFERENCES users(id),
    total_amount DECIMAL(10, 2) NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'Pending',
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS order_items (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id   UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL REFERENCES products(id),
    quantity   INTEGER NOT NULL,
    price      DECIMAL(10, 2) NOT NULL
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```

### Repository Layer

```scala
// src/main/scala/db/ProductRepository.scala
import cats.effect.*
import doobie.*
import doobie.implicits.*
import doobie.postgres.implicits.*
import java.util.UUID

class ProductRepository(xa: Transactor[IO]):
  import ProductRepository.*
  
  def findById(id: ProductId): IO[Option[Product]] =
    sql"""
      SELECT id, name, description, price, stock, category_id, created_at, updated_at
      FROM products
      WHERE id = ${id.value}
    """.query[ProductRow].option
      .map(_.map(toProduct))
      .transact(xa)
  
  def findAll(
    categoryId: Option[CategoryId] = None,
    minPrice: Option[BigDecimal] = None,
    maxPrice: Option[BigDecimal] = None,
    offset: Int = 0,
    limit: Int = 20
  ): IO[List[Product]] =
    val baseQuery = fr"""
      SELECT id, name, description, price, stock, category_id, created_at, updated_at
      FROM products
      WHERE 1=1
    """
    val catFilter   = categoryId.map(c => fr"AND category_id = ${c.value}").getOrElse(Fragment.empty)
    val minFilter   = minPrice.map(p => fr"AND price >= $p").getOrElse(Fragment.empty)
    val maxFilter   = maxPrice.map(p => fr"AND price <= $p").getOrElse(Fragment.empty)
    val pagination  = fr"ORDER BY created_at DESC OFFSET $offset LIMIT $limit"
    
    (baseQuery ++ catFilter ++ minFilter ++ maxFilter ++ pagination)
      .query[ProductRow]
      .to[List]
      .map(_.map(toProduct))
      .transact(xa)
  
  def create(product: Product): IO[Product] =
    sql"""
      INSERT INTO products (id, name, description, price, stock, category_id)
      VALUES (
        ${product.id.value},
        ${product.name},
        ${product.description},
        ${product.price},
        ${product.stock},
        ${product.categoryId.value}
      )
      RETURNING id, name, description, price, stock, category_id, created_at, updated_at
    """.query[ProductRow]
      .unique
      .map(toProduct)
      .transact(xa)
  
  def update(id: ProductId, req: UpdateProductRequest): IO[Option[Product]] =
    val updates = List(
      req.name.map(n => fr"name = $n"),
      req.description.flatten.map(d => fr"description = $d"),
      req.price.map(p => fr"price = $p"),
      req.stock.map(s => fr"stock = $s")
    ).flatten
    
    if updates.isEmpty then findById(id)
    else
      val setClause = updates.reduce(_ ++ fr"," ++ _)
      val query = fr"UPDATE products SET" ++ setClause ++
                  fr", updated_at = NOW() WHERE id = ${id.value}" ++
                  fr"RETURNING id, name, description, price, stock, category_id, created_at, updated_at"
      query.query[ProductRow].option.map(_.map(toProduct)).transact(xa)
  
  def delete(id: ProductId): IO[Boolean] =
    sql"DELETE FROM products WHERE id = ${id.value}"
      .update.run
      .map(_ > 0)
      .transact(xa)
  
  def updateStock(id: ProductId, delta: Int): IO[Boolean] =
    sql"""
      UPDATE products
      SET stock = stock + $delta, updated_at = NOW()
      WHERE id = ${id.value} AND stock + $delta >= 0
    """.update.run
      .map(_ > 0)
      .transact(xa)

object ProductRepository:
  private case class ProductRow(
    id: UUID, name: String, description: Option[String],
    price: BigDecimal, stock: Int, categoryId: UUID,
    createdAt: java.time.Instant, updatedAt: java.time.Instant
  )
  
  private def toProduct(row: ProductRow): Product =
    Product(
      id = ProductId(row.id),
      name = row.name,
      description = row.description,
      price = row.price,
      stock = row.stock,
      categoryId = CategoryId(row.categoryId),
      createdAt = row.createdAt,
      updatedAt = row.updatedAt
    )
```

---

## 5. Redis Caching Layer

```scala
// src/main/scala/cache/RedisCache.scala
import cats.effect.*
import dev.profunktor.redis4cats.*
import dev.profunktor.redis4cats.effect.Log.Stdout.*
import io.circe.*
import io.circe.syntax.*
import io.circe.parser.*
import scala.concurrent.duration.*

class RedisCache[F[_]: Concurrent](redis: RedisCommands[F, String, String]):
  def get[A: Decoder](key: String): F[Option[A]] =
    redis.get(key).map {
      _.flatMap(s => parse(s).flatMap(_.as[A]).toOption)
    }
  
  def set[A: Encoder](key: String, value: A, ttl: Option[FiniteDuration] = None): F[Unit] =
    val json = value.asJson.noSpaces
    ttl match
      case None    => redis.set(key, json)
      case Some(d) => redis.setEx(key, json, d)
  
  def del(key: String): F[Unit] =
    redis.del(key).void
  
  def invalidatePattern(pattern: String): F[Unit] =
    redis.keys(pattern).flatMap { keys =>
      if keys.isEmpty then Concurrent[F].unit
      else redis.del(keys.head, keys.tail*).void
    }

object RedisCache:
  def create[F[_]: Async](config: RedisConfig): Resource[F, RedisCache[F]] =
    Redis[F].utf8(config.uri)
      .map(cmd => RedisCache(cmd))
```

### Cached Service Layer

```scala
// src/main/scala/service/CachedProductService.scala
import cats.effect.*
import cats.syntax.all.*
import org.typelevel.log4cats.Logger
import scala.concurrent.duration.*

class CachedProductService[F[_]: Concurrent: Logger](
  repo: ProductRepository,
  cache: RedisCache[F]
):
  private val ProductTTL = 5.minutes
  
  def findById(id: ProductId): F[Option[Product]] =
    val cacheKey = s"product:${id.value}"
    
    cache.get[Product](cacheKey).flatMap {
      case Some(product) =>
        Logger[F].debug(s"Cache hit for product ${id.value}") >>
        Concurrent[F].pure(Some(product))
      
      case None =>
        for
          _       <- Logger[F].debug(s"Cache miss for product ${id.value}")
          product <- repo.findById(id)
          _       <- product.traverse_(p => cache.set(cacheKey, p, Some(ProductTTL)))
        yield product
    }
  
  def create(product: Product): F[Product] =
    for
      created <- repo.create(product)
      _       <- cache.set(s"product:${created.id.value}", created, Some(ProductTTL))
      _       <- cache.invalidatePattern("products:list:*")
    yield created
  
  def update(id: ProductId, req: UpdateProductRequest): F[Option[Product]] =
    for
      updated <- repo.update(id, req)
      _ <- updated match
        case None => Concurrent[F].unit
        case Some(p) =>
          cache.set(s"product:${id.value}", p, Some(ProductTTL)) >>
          cache.invalidatePattern("products:list:*")
    yield updated
  
  def delete(id: ProductId): F[Boolean] =
    for
      deleted <- repo.delete(id)
      _ <- if deleted then
        cache.del(s"product:${id.value}") >>
        cache.invalidatePattern("products:list:*")
      else Concurrent[F].unit
    yield deleted
  
  def findAll(
    categoryId: Option[CategoryId] = None,
    page: Int = 0,
    pageSize: Int = 20
  ): F[List[Product]] =
    val cacheKey = s"products:list:cat=${categoryId.map(_.value)}:page=$page:size=$pageSize"
    
    cache.get[List[Product]](cacheKey).flatMap {
      case Some(products) => Concurrent[F].pure(products)
      case None =>
        repo.findAll(categoryId, offset = page * pageSize, limit = pageSize)
          .flatTap(ps => cache.set(cacheKey, ps, Some(1.minute)))
    }
```

---

## 6. Authentication และ Authorization

```scala
// src/main/scala/auth/AuthService.scala
import cats.effect.*
import cats.syntax.all.*
import org.mindrot.jbcrypt.BCrypt
import pdi.jwt.*
import io.circe.*
import io.circe.syntax.*
import java.time.Instant

class AuthService[F[_]: Sync](
  userRepo: UserRepository[F],
  jwtConfig: JwtConfig
):
  def register(req: RegisterRequest): F[User] =
    for
      _ <- validateEmail(req.email)
      _ <- validatePassword(req.password)
      existing <- userRepo.findByEmail(req.email)
      _ <- existing match
        case Some(_) => Sync[F].raiseError(ConflictError(s"Email ${req.email} already exists"))
        case None    => Sync[F].unit
      
      hashedPassword = BCrypt.hashpw(req.password, BCrypt.gensalt(12))
      user = User(
        id = UserId(java.util.UUID.randomUUID()),
        username = req.username,
        email = req.email,
        passwordHash = hashedPassword,
        role = UserRole.Customer,
        createdAt = Instant.now()
      )
      created <- userRepo.create(user)
    yield created
  
  def login(req: LoginRequest): F[AuthResponse] =
    for
      user <- userRepo.findByEmail(req.email)
        .flatMap {
          case None    => Sync[F].raiseError(AuthError("Invalid credentials"))
          case Some(u) => Sync[F].pure(u)
        }
      
      _ <- if BCrypt.checkpw(req.password, user.passwordHash) then Sync[F].unit
           else Sync[F].raiseError(AuthError("Invalid credentials"))
      
      token = generateToken(user)
    yield AuthResponse(token, user.id.value.toString, user.role.toString)
  
  def verifyToken(token: String): F[AuthClaims] =
    Sync[F].fromTry {
      JwtCirce.decode(token, jwtConfig.secret, Seq(JwtAlgorithm.HS256))
        .map(_.content)
        .map(parse(_).flatMap(_.as[AuthClaims]).toTry.get)
        .flatten
    }.adaptError(e => AuthError(s"Invalid token: ${e.getMessage}"))
  
  private def generateToken(user: User): String =
    val claims = AuthClaims(
      userId = user.id.value.toString,
      role   = user.role.toString,
      exp    = Instant.now().plusSeconds(jwtConfig.expirationHours * 3600L).getEpochSecond
    )
    JwtCirce.encode(claims.asJson.noSpaces, jwtConfig.secret, JwtAlgorithm.HS256)
  
  private def validateEmail(email: String): F[Unit] =
    if email.contains("@") then Sync[F].unit
    else Sync[F].raiseError(ValidationError(s"Invalid email: $email"))
  
  private def validatePassword(password: String): F[Unit] =
    if password.length >= 8 then Sync[F].unit
    else Sync[F].raiseError(ValidationError("Password must be at least 8 characters"))

case class AuthClaims(userId: String, role: String, exp: Long)
  derives io.circe.Codec.AsObject

// Auth Middleware
import org.http4s.*
import org.http4s.server.AuthMiddleware

object AuthMiddleware:
  def create[F[_]: Sync](authService: AuthService[F]): AuthMiddleware[F, AuthClaims] =
    val authUser: Kleisli[F, Request[F], Either[String, AuthClaims]] = Kleisli { req =>
      req.headers.get[headers.Authorization] match
        case None => Sync[F].pure(Left("Missing Authorization header"))
        case Some(authHeader) =>
          authHeader.credentials match
            case Credentials.Token(AuthScheme.Bearer, token) =>
              authService.verifyToken(token).map(Right(_)).handleError(e => Left(e.getMessage))
            case _ =>
              Sync[F].pure(Left("Invalid Authorization scheme"))
    }
    
    val onFailure: AuthedRoutes[String, F] = Kleisli { cx =>
      OptionT.pure(Response[F](Status.Unauthorized)
        .withEntity(s"""{"error": "${cx.context}"}"""))
    }
    
    org.http4s.server.AuthMiddleware(authUser, onFailure)
```

---

## 7. HTTP API ด้วย http4s

```scala
// src/main/scala/http/ProductRoutes.scala
import cats.effect.*
import cats.syntax.all.*
import org.http4s.*
import org.http4s.circe.*
import org.http4s.circe.CirceEntityCodec.*
import org.http4s.dsl.Http4sDsl
import io.circe.generic.auto.*
import io.circe.syntax.*

class ProductRoutes[F[_]: Sync](
  productService: CachedProductService[F],
  eventPublisher: KafkaEventPublisher[F]
) extends Http4sDsl[F]:
  
  // Public routes (no auth)
  val publicRoutes: HttpRoutes[F] = HttpRoutes.of[F] {
    case GET -> Root / "products" :? CategoryIdParam(catId) +& PageParam(page) +& PageSizeParam(size) =>
      val categoryId = catId.flatMap(s => scala.util.Try(CategoryId(java.util.UUID.fromString(s))).toOption)
      productService.findAll(categoryId, page.getOrElse(0), size.getOrElse(20))
        .flatMap(ps => Ok(ps.asJson))
        .handleErrorWith(handleError)
    
    case GET -> Root / "products" / UUIDVar(id) =>
      productService.findById(ProductId(id))
        .flatMap {
          case None    => NotFound(s"""{"error": "Product not found"}""")
          case Some(p) => Ok(p.asJson)
        }
        .handleErrorWith(handleError)
  }
  
  // Admin routes (require auth + admin role)
  def adminRoutes(claims: AuthClaims): HttpRoutes[F] =
    if claims.role != "Admin" then
      HttpRoutes.of { _ => Forbidden("""{"error": "Admin access required"}""") }
    else HttpRoutes.of[F] {
      case req @ POST -> Root / "products" =>
        for
          body    <- req.as[CreateProductRequest]
          _       <- validateCreateRequest(body)
          product  = buildProduct(body)
          created <- productService.create(product)
          _       <- eventPublisher.publish(ProductCreated(created.id.value.toString, created.name))
          resp    <- Created(created.asJson)
        yield resp
      
      case req @ PUT -> Root / "products" / UUIDVar(id) =>
        for
          body    <- req.as[UpdateProductRequest]
          updated <- productService.update(ProductId(id), body)
          resp <- updated match
            case None    => NotFound(s"""{"error": "Product not found"}""")
            case Some(p) => Ok(p.asJson)
        yield resp
      
      case DELETE -> Root / "products" / UUIDVar(id) =>
        productService.delete(ProductId(id)).flatMap {
          case true  => NoContent()
          case false => NotFound(s"""{"error": "Product not found"}""")
        }
    }
  
  private def handleError(error: Throwable): F[Response[F]] = error match
    case NotFoundError(msg)   => NotFound(s"""{"error": "$msg"}""")
    case ValidationError(msg) => BadRequest(s"""{"error": "$msg"}""")
    case AuthError(msg)       => Unauthorized(s"""{"error": "$msg"}""")
    case ConflictError(msg)   => Conflict(s"""{"error": "$msg"}""")
    case e                    => InternalServerError(s"""{"error": "Internal error: ${e.getMessage}"}""")
  
  private def validateCreateRequest(req: CreateProductRequest): F[Unit] =
    val errors = List(
      Option.when(req.name.isBlank)("Name cannot be blank"),
      Option.when(req.price <= 0)("Price must be positive"),
      Option.when(req.stock < 0)("Stock cannot be negative")
    ).flatten
    
    if errors.isEmpty then Sync[F].unit
    else Sync[F].raiseError(ValidationError(errors.mkString(", ")))
  
  private def buildProduct(req: CreateProductRequest): Product =
    Product(
      id = ProductId(java.util.UUID.randomUUID()),
      name = req.name,
      description = req.description,
      price = req.price,
      stock = req.stock,
      categoryId = CategoryId(java.util.UUID.fromString(req.categoryId)),
      createdAt = java.time.Instant.now(),
      updatedAt = java.time.Instant.now()
    )

// Query parameters
object CategoryIdParam extends OptionalQueryParamDecoderMatcher[String]("categoryId")
object PageParam extends OptionalQueryParamDecoderMatcher[Int]("page")
object PageSizeParam extends OptionalQueryParamDecoderMatcher[Int]("pageSize")
```

### Main Application Server

```scala
// src/main/scala/Main.scala
import cats.effect.*
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s.*

object Main extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    AppConfig.load.flatMap { config =>
      val resources = for
        xa      <- Database.transactor(config.database)
        redis   <- RedisCache.create[IO](config.redis)
        kafka   <- KafkaEventPublisher.create[IO](config.kafka)
      yield (xa, redis, kafka)
      
      resources.use { case (xa, redis, kafka) =>
        for
          _ <- Database.migrate(config.database)
          
          // Repositories
          userRepo    = UserRepository(xa)
          productRepo = ProductRepository(xa)
          
          // Services
          authService    = AuthService(userRepo, config.jwt)
          productService = CachedProductService(productRepo, redis)
          
          // Auth middleware
          authMiddleware = AuthMiddleware.create(authService)
          
          // Routes
          authRoutes    = AuthRoutes(authService)
          productRoutes = ProductRoutes(productService, kafka)
          
          // Combine routes
          allRoutes = authRoutes.routes <+>
                      authMiddleware(AuthedRoutes.of {
                        case req @ _ => productRoutes.adminRoutes(req.context).orNotFound(req.req)
                      }) <+>
                      productRoutes.publicRoutes
          
          // Start server
          _ <- EmberServerBuilder.default[IO]
            .withHost(Host.fromString(config.server.host).get)
            .withPort(Port.fromInt(config.server.port).get)
            .withHttpApp(allRoutes.orNotFound)
            .build
            .useForever
        yield ExitCode.Success
      }
    }
```

---

## 8. Kafka Event Publishing

```scala
// src/main/scala/kafka/KafkaEventPublisher.scala
import cats.effect.*
import fs2.kafka.*
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.auto.*

// Events
sealed trait ShopEvent derives Encoder.AsObject
case class ProductCreated(id: String, name: String) extends ShopEvent
case class ProductUpdated(id: String) extends ShopEvent
case class ProductDeleted(id: String) extends ShopEvent
case class OrderPlaced(orderId: String, userId: String, total: Double) extends ShopEvent
case class OrderStatusChanged(orderId: String, status: String) extends ShopEvent

class KafkaEventPublisher[F[_]: Concurrent](
  producer: KafkaProducer[F, String, String],
  topic: String
):
  def publish(event: ShopEvent): F[Unit] =
    val key   = event.getClass.getSimpleName
    val value = event.asJson.noSpaces
    val record = ProducerRecord(topic, key, value)
    producer.produce(ProducerRecords.one(record)).flatten.void
  
  def publishBatch(events: List[ShopEvent]): F[Unit] =
    val records = events.map { event =>
      ProducerRecord(topic, event.getClass.getSimpleName, event.asJson.noSpaces)
    }
    producer.produce(ProducerRecords(records*)).flatten.void

object KafkaEventPublisher:
  def create[F[_]: Async](config: KafkaConfig): Resource[F, KafkaEventPublisher[F]] =
    val settings = ProducerSettings[F, String, String]
      .withBootstrapServers(config.bootstrapServers)
      .withAcks(Acks.One)
      .withRetries(3)
    
    KafkaProducer.resource(settings)
      .map(producer => KafkaEventPublisher(producer, config.topic))
```

---

## 9. Docker Deployment

### Dockerfile

```dockerfile
# Dockerfile
FROM sbtscala/scala-sbt:eclipse-temurin-17.0.5_8_1.9.7_3.3.1 AS builder
WORKDIR /build
COPY build.sbt .
COPY project project
COPY src src
RUN sbt assembly

FROM eclipse-temurin:17-jre-alpine
RUN addgroup -S app && adduser -S app -G app
USER app
WORKDIR /app
COPY --from=builder /build/target/scala-3.3.1/scala-rest-api-assembly-0.1.0.jar app.jar
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75"
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]
```

### docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: jdbc:postgresql://postgres:5432/shopdb
      DATABASE_USER: postgres
      DATABASE_PASSWORD: password
      REDIS_URI: redis://redis:6379
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      JWT_SECRET: ${JWT_SECRET:-change-me-in-production}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 3

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    healthcheck:
      test: ["CMD-SHELL", "kafka-topics --bootstrap-server localhost:9092 --list"]
      interval: 10s
      timeout: 10s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

---

## 10. Integration Tests

```scala
// src/test/scala/integration/ProductApiSpec.scala
import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import org.http4s.*
import org.http4s.client.*
import org.http4s.circe.*
import org.http4s.circe.CirceEntityCodec.*
import io.circe.generic.auto.*
import org.scalatest.freespec.AsyncFreeSpec
import org.scalatest.matchers.should.Matchers

class ProductApiSpec extends AsyncFreeSpec with AsyncIOSpec with Matchers:
  // สมมติว่า server กำลังรันบน localhost:8080
  val baseUrl = "http://localhost:8080"
  
  "Product API" - {
    "GET /products should return list" in {
      EmberClientBuilder.default[IO].build.use { client =>
        client.expect[List[Product]](s"$baseUrl/products")
          .map(products => products.isInstanceOf[List[?]] shouldBe true)
      }
    }
    
    "POST /products should require admin auth" in {
      EmberClientBuilder.default[IO].build.use { client =>
        val req = Request[IO](method = Method.POST, uri = Uri.unsafeFromString(s"$baseUrl/products"))
          .withEntity(CreateProductRequest("Test", None, 10.0, 5, "category-id"))
        
        client.status(req).map(_ shouldBe Status.Unauthorized)
      }
    }
    
    "Full CRUD flow" in {
      EmberClientBuilder.default[IO].build.use { client =>
        for
          // Login as admin
          loginResp <- client.expect[AuthResponse](
            Request[IO](method = Method.POST, uri = Uri.unsafeFromString(s"$baseUrl/auth/login"))
              .withEntity(LoginRequest("admin@test.com", "password123"))
          )
          token = loginResp.token
          
          // Create product
          created <- client.expect[Product](
            Request[IO](method = Method.POST, uri = Uri.unsafeFromString(s"$baseUrl/products"))
              .withHeaders(Header.Raw("Authorization".ci, s"Bearer $token"))
              .withEntity(CreateProductRequest("Test Product", Some("Desc"), 99.99, 10, "cat-id"))
          )
          _ = created.name shouldBe "Test Product"
          
          // Get product
          fetched <- client.expect[Product](s"$baseUrl/products/${created.id.value}")
          _ = fetched.id shouldBe created.id
          
          // Update product
          updated <- client.expect[Product](
            Request[IO](method = Method.PUT, uri = Uri.unsafeFromString(s"$baseUrl/products/${created.id.value}"))
              .withHeaders(Header.Raw("Authorization".ci, s"Bearer $token"))
              .withEntity(UpdateProductRequest(Some("Updated Name"), None, None, None))
          )
          _ = updated.name shouldBe "Updated Name"
          
          // Delete product
          deleteStatus <- client.status(
            Request[IO](method = Method.DELETE, uri = Uri.unsafeFromString(s"$baseUrl/products/${created.id.value}"))
              .withHeaders(Header.Raw("Authorization".ci, s"Bearer $token"))
          )
          _ = deleteStatus shouldBe Status.NoContent
        yield succeed
      }
    }
  }
```

---

## สรุป

Project นี้แสดงให้เห็น full-stack REST API ด้วย Scala 3:

| Component | Technology |
|-----------|-----------|
| HTTP Server | http4s + Ember |
| Database | PostgreSQL + Doobie |
| Caching | Redis + redis4cats |
| Events | Kafka + fs2-kafka |
| Authentication | JWT + BCrypt |
| Configuration | PureConfig |
| Testing | cats-effect-testing |
| Deployment | Docker + Compose |

### สิ่งที่ควรปรับปรุงสำหรับ Production

1. เพิ่ม rate limiting
2. เพิ่ม circuit breaker สำหรับ external services
3. เพิ่ม distributed tracing (OpenTelemetry)
4. เพิ่ม metrics (Prometheus)
5. เพิ่ม comprehensive logging
6. เพิ่ม input sanitization
7. ใช้ connection pooling ที่ tune ค่าแล้ว

---

*[← Part 96: Concurrent Data Structures](part-96-concurrent-data.md) | [Part 98: Final Project Streaming →](part-98-project-final-streaming.md)*
