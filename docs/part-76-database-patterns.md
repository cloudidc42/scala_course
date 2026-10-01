# ส่วนที่ 76: Database Patterns ใน Scala

## สารบัญ

1. [Repository Pattern กับ Type Classes](#repository-pattern-กับ-type-classes)
2. [Unit of Work Pattern](#unit-of-work-pattern)
3. [Query Object Pattern](#query-object-pattern)
4. [CQRS กับ Read/Write Models แยกกัน](#cqrs-กับ-readwrite-models-แยกกัน)
5. [Optimistic Locking](#optimistic-locking)
6. [Change Data Capture (CDC)](#change-data-capture-cdc)
7. [Connection Pooling กับ HikariCP](#connection-pooling-กับ-hikaricp)
8. [Complete Database Architecture](#complete-database-architecture)
9. [สรุป](#สรุป)

---

## Repository Pattern กับ Type Classes

Repository Pattern ช่วยแยก Business Logic ออกจาก Data Access Layer

### การกำหนด Type Class

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.tpolecat" %% "doobie-core" % "1.0.0-RC4",
  "org.tpolecat" %% "doobie-hikari" % "1.0.0-RC4",
  "org.tpolecat" %% "doobie-postgres" % "1.0.0-RC4",
  "org.typelevel" %% "cats-effect" % "3.5.2",
  "com.zaxxer" % "HikariCP" % "5.0.1",
  "org.flywaydb" % "flyway-core" % "9.22.3",
  "io.circe" %% "circe-generic" % "0.14.6"
)
```

```scala
// src/main/scala/repository/Repository.scala
package repository

import cats.effect.IO
import java.util.UUID

// Type class สำหรับ Repository operations
trait Repository[F[_], E, ID]:
  def findById(id: ID): F[Option[E]]
  def findAll: F[List[E]]
  def save(entity: E): F[E]
  def update(entity: E): F[Option[E]]
  def delete(id: ID): F[Boolean]
  def exists(id: ID): F[Boolean]

// Type class สำหรับการค้นหา
trait Searchable[F[_], E, Q]:
  def search(query: Q): F[List[E]]
  def count(query: Q): F[Long]

// Type class สำหรับ pagination
trait Pageable[F[_], E, Q]:
  def findPage(query: Q, page: Int, size: Int): F[Page[E]]

case class Page[E](
  content: List[E],
  totalElements: Long,
  totalPages: Int,
  currentPage: Int,
  pageSize: Int
)

// Entity type class
trait Entity[E]:
  extension (e: E)
    def id: Any
    def version: Long
```

### Doobie Repository Implementation

```scala
// src/main/scala/repository/UserRepository.scala
package repository

import cats.effect.IO
import doobie.*
import doobie.implicits.*
import doobie.postgres.implicits.*
import java.util.UUID
import java.time.Instant

// Domain model
case class User(
  id: UUID,
  email: String,
  username: String,
  hashedPassword: String,
  role: UserRole,
  createdAt: Instant,
  updatedAt: Instant,
  version: Long
)

enum UserRole:
  case Admin, User, Moderator

// Query params
case class UserQuery(
  email: Option[String] = None,
  username: Option[String] = None,
  role: Option[UserRole] = None
)

// Doobie Meta instances
given Meta[UUID] = Meta.Advanced.one(
  java.sql.JDBCType.OTHER,
  NonEmptyList.one("uuid"),
  _.getObject(_, classOf[UUID]),
  (ps, n, a) => ps.setObject(n, a),
  (rs, n) => rs.updateObject(n, rs.getObject(n))
)

given Meta[UserRole] = Meta[String].imap(
  str => UserRole.valueOf(str.capitalize)
)(_.toString.toLowerCase)

given Meta[Instant] = Meta.Advanced.one(
  java.sql.JDBCType.TIMESTAMP_WITH_TIMEZONE,
  NonEmptyList.one("timestamptz"),
  (rs, n) => rs.getTimestamp(n).toInstant,
  (ps, n, a) => ps.setTimestamp(n, java.sql.Timestamp.from(a)),
  (rs, n) => rs.updateTimestamp(n, java.sql.Timestamp.from(rs.getTimestamp(n).toInstant))
)

class UserRepository(xa: Transactor[IO])
    extends Repository[IO, User, UUID]
    with Searchable[IO, User, UserQuery]:
  
  def findById(id: UUID): IO[Option[User]] =
    sql"""
      SELECT id, email, username, hashed_password, role, created_at, updated_at, version
      FROM users
      WHERE id = $id AND deleted_at IS NULL
    """.query[User].option.transact(xa)
  
  def findAll: IO[List[User]] =
    sql"""
      SELECT id, email, username, hashed_password, role, created_at, updated_at, version
      FROM users
      WHERE deleted_at IS NULL
      ORDER BY created_at DESC
    """.query[User].to[List].transact(xa)
  
  def save(user: User): IO[User] =
    sql"""
      INSERT INTO users (id, email, username, hashed_password, role, created_at, updated_at, version)
      VALUES (${user.id}, ${user.email}, ${user.username}, ${user.hashedPassword},
              ${user.role}, ${user.createdAt}, ${user.updatedAt}, ${user.version})
      RETURNING *
    """.query[User].unique.transact(xa)
  
  def update(user: User): IO[Option[User]] =
    sql"""
      UPDATE users
      SET email = ${user.email},
          username = ${user.username},
          updated_at = ${Instant.now()},
          version = version + 1
      WHERE id = ${user.id} AND version = ${user.version} AND deleted_at IS NULL
      RETURNING *
    """.query[User].option.transact(xa)
  
  def delete(id: UUID): IO[Boolean] =
    sql"""
      UPDATE users SET deleted_at = NOW() WHERE id = $id AND deleted_at IS NULL
    """.update.run.transact(xa).map(_ > 0)
  
  def exists(id: UUID): IO[Boolean] =
    sql"SELECT EXISTS(SELECT 1 FROM users WHERE id = $id AND deleted_at IS NULL)"
      .query[Boolean].unique.transact(xa)
  
  // Searchable implementation
  def search(query: UserQuery): IO[List[User]] =
    buildSearchQuery(query).to[List].transact(xa)
  
  def count(query: UserQuery): IO[Long] =
    buildCountQuery(query).unique.transact(xa)
  
  private def buildSearchQuery(q: UserQuery): Query0[User] =
    val base = fr"""
      SELECT id, email, username, hashed_password, role, created_at, updated_at, version
      FROM users WHERE deleted_at IS NULL
    """
    
    val emailFilter = q.email.map(e => fr"AND email ILIKE ${"%" + e + "%"}")
    val usernameFilter = q.username.map(u => fr"AND username ILIKE ${"%" + u + "%"}")
    val roleFilter = q.role.map(r => fr"AND role = $r")
    
    val filters = List(emailFilter, usernameFilter, roleFilter).flatten
    val query = filters.foldLeft(base)(_ ++ _) ++ fr"ORDER BY created_at DESC"
    
    query.query[User]
  
  private def buildCountQuery(q: UserQuery): Query0[Long] =
    val base = fr"SELECT COUNT(*) FROM users WHERE deleted_at IS NULL"
    
    val emailFilter = q.email.map(e => fr"AND email ILIKE ${"%" + e + "%"}")
    val usernameFilter = q.username.map(u => fr"AND username ILIKE ${"%" + u + "%"}")
    val roleFilter = q.role.map(r => fr"AND role = $r")
    
    val filters = List(emailFilter, usernameFilter, roleFilter).flatten
    val query = filters.foldLeft(base)(_ ++ _)
    
    query.query[Long]
```

---

## Unit of Work Pattern

Unit of Work ช่วยจัดการ Transaction ของ Database Operations หลายอย่างให้เป็นหน่วยเดียว

```scala
// src/main/scala/repository/UnitOfWork.scala
package repository

import cats.effect.IO
import doobie.*
import doobie.implicits.*

// Unit of Work abstraction
trait UnitOfWork[F[_]]:
  def run[A](work: UoWContext[F] => F[A]): F[A]

case class UoWContext[F[_]](
  users: UserRepository,
  products: ProductRepository,
  orders: OrderRepository
)

// Doobie-based implementation
class DoobieUnitOfWork(xa: Transactor[IO]) extends UnitOfWork[IO]:
  
  private val userRepo = UserRepository(xa)
  private val productRepo = ProductRepository(xa)
  private val orderRepo = OrderRepository(xa)
  
  def run[A](work: UoWContext[IO] => IO[A]): IO[A] =
    val context = UoWContext(userRepo, productRepo, orderRepo)
    work(context)
  
  // Transactional version
  def runTransactionally[A](work: UoWContext[IO] => ConnectionIO[A]): IO[A] =
    val context = UoWContext(userRepo, productRepo, orderRepo)
    // work(context) would use ConnectionIO operations
    // that get transacted together
    ???

// Example usage with Unit of Work
class OrderService(uow: UnitOfWork[IO]):
  
  def createOrder(userId: UUID, items: List[OrderItem]): IO[Order] =
    uow.run { ctx =>
      for
        user     <- ctx.users.findById(userId).flatMap {
                      case Some(u) => IO.pure(u)
                      case None    => IO.raiseError(UserNotFoundException(userId))
                    }
        
        // Check product availability
        products <- items.traverse(item =>
                      ctx.products.findById(item.productId).flatMap {
                        case Some(p) if p.stock >= item.quantity => IO.pure(p)
                        case Some(p) => IO.raiseError(InsufficientStockException(p.id))
                        case None    => IO.raiseError(ProductNotFoundException(item.productId))
                      }
                    )
        
        // Create order
        order    <- ctx.orders.save(Order.create(userId, items))
        
        // Deduct stock
        _        <- products.zip(items).traverse { (product, item) =>
                      ctx.products.update(product.copy(stock = product.stock - item.quantity))
                    }
      yield order
    }
```

---

## Query Object Pattern

Query Object Pattern ช่วยสร้าง Database Queries แบบ type-safe

```scala
// src/main/scala/repository/QueryObject.scala
package repository

import doobie.*
import doobie.implicits.*

// Query specification
sealed trait Specification[E]:
  def toFragment: Fragment

case class AndSpec[E](left: Specification[E], right: Specification[E]) extends Specification[E]:
  def toFragment: Fragment = left.toFragment ++ fr" AND " ++ right.toFragment

case class OrSpec[E](left: Specification[E], right: Specification[E]) extends Specification[E]:
  def toFragment: Fragment = fr"(" ++ left.toFragment ++ fr" OR " ++ right.toFragment ++ fr")"

case class NotSpec[E](spec: Specification[E]) extends Specification[E]:
  def toFragment: Fragment = fr"NOT (" ++ spec.toFragment ++ fr")"

// Product specifications
object ProductSpecs:
  case class ByCategory(categoryId: UUID) extends Specification[Product]:
    def toFragment: Fragment = fr"category_id = $categoryId"
  
  case class PriceRange(min: BigDecimal, max: BigDecimal) extends Specification[Product]:
    def toFragment: Fragment = fr"price >= $min AND price <= $max"
  
  case object InStock extends Specification[Product]:
    def toFragment: Fragment = fr"stock > 0"
  
  case class ByName(name: String) extends Specification[Product]:
    def toFragment: Fragment = fr"name ILIKE ${"%" + name + "%"}"

// DSL for building queries
extension [E](spec: Specification[E])
  def &&(other: Specification[E]): Specification[E] = AndSpec(spec, other)
  def ||(other: Specification[E]): Specification[E] = OrSpec(spec, other)
  def unary_! : Specification[E] = NotSpec(spec)

// Query builder
case class QueryBuilder[E](
  spec: Option[Specification[E]] = None,
  orderBy: Option[String] = None,
  ascending: Boolean = true,
  limit: Option[Int] = None,
  offset: Option[Int] = None
):
  def where(s: Specification[E]): QueryBuilder[E] =
    copy(spec = Some(spec.map(_ && s).getOrElse(s)))
  
  def orderByField(field: String, asc: Boolean = true): QueryBuilder[E] =
    copy(orderBy = Some(field), ascending = asc)
  
  def take(n: Int): QueryBuilder[E] = copy(limit = Some(n))
  def skip(n: Int): QueryBuilder[E] = copy(offset = Some(n))
  
  def toFragment(baseSelect: Fragment): Fragment =
    val whereClause = spec.map(s => fr" WHERE " ++ s.toFragment).getOrElse(fr"")
    val orderClause = orderBy.map { field =>
      val direction = if ascending then fr"ASC" else fr"DESC"
      fr" ORDER BY " ++ Fragment.const(field) ++ fr" " ++ direction
    }.getOrElse(fr"")
    val limitClause = limit.map(n => fr" LIMIT $n").getOrElse(fr"")
    val offsetClause = offset.map(n => fr" OFFSET $n").getOrElse(fr"")
    
    baseSelect ++ whereClause ++ orderClause ++ limitClause ++ offsetClause

// Usage example
class AdvancedProductRepository(xa: Transactor[IO]):
  
  private val baseSelect = fr"""
    SELECT id, name, description, price, category_id, stock, created_at, updated_at
    FROM products WHERE deleted_at IS NULL
  """
  
  def findBy(builder: QueryBuilder[Product]): IO[List[Product]] =
    builder.toFragment(baseSelect).query[Product].to[List].transact(xa)

// Example usage
val expensiveInStockElectronics = QueryBuilder[Product]()
  .where(ProductSpecs.ByCategory(electronicsId))
  .where(ProductSpecs.InStock)
  .where(ProductSpecs.PriceRange(BigDecimal(100), BigDecimal(1000)))
  .orderByField("price", asc = false)
  .take(10)
```

---

## CQRS กับ Read/Write Models แยกกัน

CQRS (Command Query Responsibility Segregation) แยก operations การอ่านและเขียนออกจากกัน

```scala
// src/main/scala/cqrs/WriteModels.scala
package cqrs

import java.util.UUID
import java.time.Instant

// Write side - เน้นความถูกต้องของ domain logic
case class ProductAggregate(
  id: UUID,
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: UUID,
  stock: Int,
  version: Long,
  events: List[DomainEvent] = List.empty
):
  def applyEvent(event: DomainEvent): ProductAggregate =
    event match
      case ProductCreated(id, name, desc, price, catId, stock, _) =>
        this.copy(events = events :+ event)
      case PriceUpdated(_, newPrice, _) =>
        this.copy(price = newPrice, events = events :+ event)
      case StockAdjusted(_, quantity, _) =>
        this.copy(stock = stock + quantity, events = events :+ event)
      case _ => this

object ProductAggregate:
  def create(
    name: String,
    description: String,
    price: BigDecimal,
    categoryId: UUID,
    initialStock: Int
  ): ProductAggregate =
    val id = UUID.randomUUID()
    val now = Instant.now()
    val aggregate = ProductAggregate(
      id = id,
      name = name,
      description = description,
      price = price,
      categoryId = categoryId,
      stock = initialStock,
      version = 0
    )
    aggregate.applyEvent(ProductCreated(id, name, description, price, categoryId, initialStock, now))
  
  def updatePrice(aggregate: ProductAggregate, newPrice: BigDecimal): Either[String, ProductAggregate] =
    if newPrice <= 0 then Left("Price must be positive")
    else Right(aggregate.applyEvent(PriceUpdated(aggregate.id, newPrice, Instant.now())))

// Commands
sealed trait ProductCommand
case class CreateProductCommand(name: String, description: String, price: BigDecimal, categoryId: UUID, stock: Int) extends ProductCommand
case class UpdatePriceCommand(productId: UUID, newPrice: BigDecimal) extends ProductCommand
case class AdjustStockCommand(productId: UUID, quantity: Int) extends ProductCommand

// Domain events
sealed trait DomainEvent:
  def aggregateId: UUID
  def occurredAt: Instant

case class ProductCreated(
  aggregateId: UUID,
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: UUID,
  initialStock: Int,
  occurredAt: Instant
) extends DomainEvent

case class PriceUpdated(
  aggregateId: UUID,
  newPrice: BigDecimal,
  occurredAt: Instant
) extends DomainEvent

case class StockAdjusted(
  aggregateId: UUID,
  quantity: Int,
  occurredAt: Instant
) extends DomainEvent
```

```scala
// src/main/scala/cqrs/ReadModels.scala
package cqrs

// Read side - เน้นความเร็วในการค้นหา
// Denormalized สำหรับ query ที่ซับซ้อน
case class ProductListItem(
  id: UUID,
  name: String,
  price: BigDecimal,
  categoryName: String,  // denormalized
  stock: Int,
  inStock: Boolean       // computed
)

case class ProductDetail(
  id: UUID,
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: UUID,
  categoryName: String,
  stock: Int,
  inStock: Boolean,
  reviewCount: Int,      // aggregated
  averageRating: Double, // aggregated
  relatedProducts: List[ProductSummary]
)

case class ProductSummary(
  id: UUID,
  name: String,
  price: BigDecimal,
  imageUrl: Option[String]
)

// Read repositories - optimized for queries
trait ProductReadRepository[F[_]]:
  def findForList(query: ProductListQuery): F[List[ProductListItem]]
  def findDetail(id: UUID): F[Option[ProductDetail]]
  def findRelated(productId: UUID, limit: Int): F[List[ProductSummary]]

case class ProductListQuery(
  categoryId: Option[UUID] = None,
  searchTerm: Option[String] = None,
  minPrice: Option[BigDecimal] = None,
  maxPrice: Option[BigDecimal] = None,
  inStockOnly: Boolean = false,
  sortBy: String = "createdAt",
  sortAsc: Boolean = false,
  page: Int = 1,
  pageSize: Int = 20
)

// Read model projector - updates read models from events
class ProductReadModelProjector(readRepo: ProductReadRepository[IO]):
  
  def handleEvent(event: DomainEvent): IO[Unit] =
    event match
      case e: ProductCreated =>
        IO.unit // update read model
      case e: PriceUpdated =>
        IO.unit // update price in read model
      case e: StockAdjusted =>
        IO.unit // update stock in read model

// CQRS Command Handler
class ProductCommandHandler(
  writeRepo: ProductWriteRepository[IO],
  eventBus: EventBus[IO]
):
  
  def handle(command: ProductCommand): IO[Either[String, UUID]] =
    command match
      case cmd: CreateProductCommand =>
        val aggregate = ProductAggregate.create(
          cmd.name, cmd.description, cmd.price, cmd.categoryId, cmd.stock
        )
        for
          saved  <- writeRepo.save(aggregate)
          _      <- aggregate.events.traverse(eventBus.publish)
        yield Right(saved.id)
      
      case cmd: UpdatePriceCommand =>
        for
          maybeAggregate <- writeRepo.findById(cmd.productId)
          result <- maybeAggregate match
            case None => IO.pure(Left(s"Product ${cmd.productId} not found"))
            case Some(aggregate) =>
              ProductAggregate.updatePrice(aggregate, cmd.newPrice) match
                case Left(error) => IO.pure(Left(error))
                case Right(updated) =>
                  for
                    _ <- writeRepo.save(updated)
                    _ <- updated.events.traverse(eventBus.publish)
                  yield Right(cmd.productId)
        yield result
```

---

## Optimistic Locking

Optimistic Locking ป้องกัน Concurrent Updates โดยไม่ต้องล็อค database

```scala
// src/main/scala/repository/OptimisticLocking.scala
package repository

import cats.effect.IO
import doobie.*
import doobie.implicits.*

// Version field เป็น core ของ Optimistic Locking
trait Versioned:
  def version: Long

case class VersionConflictException(
  entityType: String,
  id: UUID,
  expectedVersion: Long,
  actualVersion: Long
) extends Exception(
  s"Version conflict for $entityType $id: expected $expectedVersion but found $actualVersion"
)

// Repository with optimistic locking
class OptimisticLockingRepository(xa: Transactor[IO]):
  
  def updateWithVersion[E <: Versioned](
    tableName: String,
    id: UUID,
    entity: E,
    updateFragment: E => Fragment
  ): IO[E] =
    val query =
      updateFragment(entity) ++
      fr"WHERE id = $id AND version = ${entity.version}" ++
      fr"RETURNING *"
    
    query.query[E].option.transact(xa).flatMap {
      case Some(updated) => IO.pure(updated)
      case None =>
        // Check if entity exists
        sql"SELECT version FROM ${Fragment.const(tableName)} WHERE id = $id"
          .query[Long].option.transact(xa).flatMap {
            case Some(currentVersion) =>
              IO.raiseError(VersionConflictException(
                tableName, id, entity.version, currentVersion
              ))
            case None =>
              IO.raiseError(new RuntimeException(s"Entity $id not found in $tableName"))
          }
    }

// Product with optimistic locking
case class ProductWithVersion(
  id: UUID,
  name: String,
  price: BigDecimal,
  stock: Int,
  version: Long
) extends Versioned

class ProductOptimisticRepo(xa: Transactor[IO]):
  private val lockingRepo = OptimisticLockingRepository(xa)
  
  def update(product: ProductWithVersion): IO[ProductWithVersion] =
    sql"""
      UPDATE products
      SET name = ${product.name},
          price = ${product.price},
          stock = ${product.stock},
          version = version + 1,
          updated_at = NOW()
      WHERE id = ${product.id} AND version = ${product.version}
      RETURNING id, name, price, stock, version
    """.query[ProductWithVersion].option.transact(xa).flatMap {
      case Some(updated) => IO.pure(updated)
      case None =>
        sql"SELECT version FROM products WHERE id = ${product.id}"
          .query[Long].option.transact(xa).flatMap {
            case Some(cv) => IO.raiseError(VersionConflictException("products", product.id, product.version, cv))
            case None     => IO.raiseError(new RuntimeException(s"Product ${product.id} not found"))
          }
    }
  
  // Retry on version conflict
  def updateWithRetry(
    productId: UUID,
    updateFn: ProductWithVersion => ProductWithVersion,
    maxRetries: Int = 3
  ): IO[ProductWithVersion] =
    def attempt(retries: Int): IO[ProductWithVersion] =
      findById(productId).flatMap {
        case None => IO.raiseError(new RuntimeException(s"Product $productId not found"))
        case Some(current) =>
          update(updateFn(current)).handleErrorWith {
            case _: VersionConflictException if retries > 0 =>
              attempt(retries - 1)
            case e => IO.raiseError(e)
          }
      }
    
    attempt(maxRetries)
  
  def findById(id: UUID): IO[Option[ProductWithVersion]] =
    sql"SELECT id, name, price, stock, version FROM products WHERE id = $id"
      .query[ProductWithVersion].option.transact(xa)
```

---

## Change Data Capture (CDC)

Change Data Capture ให้เราติดตามการเปลี่ยนแปลงใน Database

```scala
// src/main/scala/cdc/ChangeDataCapture.scala
package cdc

import cats.effect.IO
import cats.effect.std.Queue
import java.time.Instant

// CDC Event types
sealed trait ChangeEvent:
  def table: String
  def operation: ChangeOperation
  def timestamp: Instant

enum ChangeOperation:
  case Insert, Update, Delete

case class RowInserted(
  table: String,
  data: Map[String, Any],
  timestamp: Instant
) extends ChangeEvent:
  val operation = ChangeOperation.Insert

case class RowUpdated(
  table: String,
  before: Map[String, Any],
  after: Map[String, Any],
  timestamp: Instant
) extends ChangeEvent:
  val operation = ChangeOperation.Update

case class RowDeleted(
  table: String,
  before: Map[String, Any],
  timestamp: Instant
) extends ChangeEvent:
  val operation = ChangeOperation.Delete

// CDC Listener
trait CDCListener[F[_]]:
  def listen: fs2.Stream[F, ChangeEvent]
  def stop: F[Unit]

// PostgreSQL WAL-based CDC (using pg_logical_replication_slot)
class PostgreSQLCDCListener(
  connectionString: String,
  slotName: String = "my_slot"
) extends CDCListener[IO]:
  
  def listen: fs2.Stream[IO, ChangeEvent] =
    fs2.Stream.resource(createReplicationConnection)
      .flatMap { conn =>
        fs2.Stream.repeatEval(readNextChange(conn))
          .collect { case Some(event) => event }
      }
  
  def stop: IO[Unit] = IO.unit
  
  private def createReplicationConnection: cats.effect.Resource[IO, Any] =
    cats.effect.Resource.make(
      IO.pure(null) // placeholder
    )(_ => IO.unit)
  
  private def readNextChange(conn: Any): IO[Option[ChangeEvent]] =
    IO.pure(None) // placeholder

// Debezium-style CDC using Kafka
class DebeziumCDCProcessor(
  queue: Queue[IO, ChangeEvent]
):
  
  def processKafkaMessage(
    topic: String,
    payload: io.circe.Json
  ): IO[Unit] =
    for
      event <- parseChangeEvent(topic, payload)
      _     <- queue.offer(event)
    yield ()
  
  private def parseChangeEvent(
    topic: String,
    payload: io.circe.Json
  ): IO[ChangeEvent] =
    // Parse Debezium envelope format
    val operation = payload.hcursor
      .downField("op")
      .as[String]
      .getOrElse("u")
    
    val tableName = topic.split("\\.").lastOption.getOrElse("unknown")
    val now = Instant.now()
    
    IO.pure(operation match
      case "c" =>
        RowInserted(tableName, Map.empty, now)
      case "u" =>
        RowUpdated(tableName, Map.empty, Map.empty, now)
      case "d" =>
        RowDeleted(tableName, Map.empty, now)
      case _ =>
        RowInserted(tableName, Map.empty, now)
    )

// Event processor pipeline
class CDCPipeline(
  listener: CDCListener[IO],
  handlers: Map[String, ChangeEventHandler]
):
  
  def run: IO[Nothing] =
    listener.listen
      .evalMap(processEvent)
      .compile.drain
      .flatMap(_ => IO.never)
  
  private def processEvent(event: ChangeEvent): IO[Unit] =
    handlers.get(event.table) match
      case Some(handler) => handler.handle(event)
      case None          => IO.println(s"No handler for table: ${event.table}")

trait ChangeEventHandler:
  def handle(event: ChangeEvent): IO[Unit]

// Cache invalidation handler
class CacheInvalidationHandler(cache: Cache[IO]) extends ChangeEventHandler:
  def handle(event: ChangeEvent): IO[Unit] =
    event match
      case RowInserted(table, data, _) =>
        cache.invalidateAll(table)
      case RowUpdated(table, _, after, _) =>
        val id = after.get("id").map(_.toString)
        id.fold(IO.unit)(cache.invalidate(table, _))
      case RowDeleted(table, before, _) =>
        val id = before.get("id").map(_.toString)
        id.fold(IO.unit)(cache.invalidate(table, _))
```

---

## Connection Pooling กับ HikariCP

```scala
// src/main/scala/database/ConnectionPool.scala
package database

import cats.effect.*
import com.zaxxer.hikari.{HikariConfig, HikariDataSource}
import doobie.hikari.HikariTransactor
import scala.concurrent.ExecutionContext

case class DatabaseConfig(
  url: String,
  user: String,
  password: String,
  driver: String = "org.postgresql.Driver",
  maxPoolSize: Int = 10,
  minIdle: Int = 2,
  connectionTimeout: Long = 30000,
  idleTimeout: Long = 600000,
  maxLifetime: Long = 1800000,
  schema: Option[String] = None
)

object ConnectionPool:
  
  def make(config: DatabaseConfig): Resource[IO, HikariTransactor[IO]] =
    for
      ec <- ExecutionContexts.fixedThreadPool[IO](config.maxPoolSize)
      xa <- HikariTransactor.newHikariTransactor[IO](
              config.driver,
              config.url,
              config.user,
              config.password,
              ec
            )
      _  <- Resource.eval(configurePool(xa, config))
    yield xa
  
  private def configurePool(
    xa: HikariTransactor[IO],
    config: DatabaseConfig
  ): IO[Unit] =
    xa.configure { ds =>
      IO {
        ds.setMaximumPoolSize(config.maxPoolSize)
        ds.setMinimumIdle(config.minIdle)
        ds.setConnectionTimeout(config.connectionTimeout)
        ds.setIdleTimeout(config.idleTimeout)
        ds.setMaxLifetime(config.maxLifetime)
        ds.setPoolName("scala-app-pool")
        
        config.schema.foreach { schema =>
          ds.setConnectionInitSql(s"SET search_path TO $schema")
        }
        
        // Performance settings
        ds.addDataSourceProperty("cachePrepStmts", "true")
        ds.addDataSourceProperty("prepStmtCacheSize", "250")
        ds.addDataSourceProperty("prepStmtCacheSqlLimit", "2048")
        ds.addDataSourceProperty("useServerPrepStmts", "true")
      }
    }

// Database health check
class DatabaseHealthCheck(xa: HikariTransactor[IO]):
  
  def check: IO[HealthStatus] =
    sql"SELECT 1".query[Int].unique.transact(xa)
      .timeout(scala.concurrent.duration.5.seconds)
      .map(_ => HealthStatus.Healthy)
      .handleError(e => HealthStatus.Unhealthy(e.getMessage))

enum HealthStatus:
  case Healthy
  case Unhealthy(reason: String)

// Flyway migration
object DatabaseMigration:
  
  def run(config: DatabaseConfig): IO[Unit] =
    IO {
      import org.flywaydb.core.Flyway
      
      val flyway = Flyway.configure()
        .dataSource(config.url, config.user, config.password)
        .locations("classpath:db/migration")
        .baselineOnMigrate(true)
        .load()
      
      flyway.migrate()
      ()
    }
```

---

## Complete Database Architecture

### Database Schema

```sql
-- V1__initial_schema.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'user',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    version BIGINT NOT NULL DEFAULT 0
);

CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    category_id UUID NOT NULL REFERENCES categories(id),
    stock INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,
    version BIGINT NOT NULL DEFAULT 0,
    CONSTRAINT positive_price CHECK (price > 0),
    CONSTRAINT non_negative_stock CHECK (stock >= 0)
);

CREATE INDEX idx_products_category ON products(category_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_price ON products(price) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_name_search ON products USING gin(to_tsvector('english', name));

-- Event store table for event sourcing
CREATE TABLE domain_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    metadata JSONB,
    sequence_number BIGINT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(aggregate_id, sequence_number)
);

CREATE INDEX idx_events_aggregate ON domain_events(aggregate_id, sequence_number);
CREATE INDEX idx_events_type ON domain_events(event_type);
```

### Complete Service Setup

```scala
// src/main/scala/AppModule.scala
package app

import cats.effect.*
import database.*
import repository.*
import services.*
import api.*

class AppModule(config: AppConfig):
  
  val resources: Resource[IO, AppDependencies] =
    for
      _  <- Resource.eval(DatabaseMigration.run(config.database))
      xa <- ConnectionPool.make(config.database)
      
      // Repositories
      userRepo    = UserRepository(xa)
      productRepo = ProductRepository(xa)
      orderRepo   = OrderRepository(xa)
      
      // Services
      userService    = UserService(userRepo)
      productService = ProductService(productRepo, orderRepo)
      orderService   = OrderService(orderRepo, productRepo)
      
    yield AppDependencies(
      userService,
      productService,
      orderService
    )

case class AppDependencies(
  userService: UserService,
  productService: ProductService,
  orderService: OrderService
)

case class AppConfig(
  database: DatabaseConfig,
  server: ServerConfig
)

case class ServerConfig(
  host: String,
  port: Int
)
```

---

## สรุป

Database Patterns ที่สำคัญใน Scala:

1. **Repository Pattern**: แยก data access ออกจาก business logic ด้วย type classes
2. **Unit of Work**: จัดการ transactions ที่เกี่ยวข้องหลายอย่างให้เป็นหน่วยเดียว
3. **Query Object**: สร้าง queries แบบ type-safe และ composable
4. **CQRS**: แยก read และ write models เพื่อ optimization
5. **Optimistic Locking**: ป้องกัน concurrent updates โดยไม่ต้องล็อค
6. **CDC**: ติดตามการเปลี่ยนแปลงข้อมูลแบบ real-time
7. **HikariCP**: Connection pooling ที่มีประสิทธิภาพสูง
8. **Flyway**: Database migration management

---

*[← ส่วนที่ 75: API Design Best Practices](part-75-api-design.md) | [ส่วนที่ 77: Service Mesh and Observability →](part-77-service-mesh.md)*
