# ส่วนที่ 100: Course Completion และ Next Steps

## 🎓 ยินดีด้วย! คุณเรียนจบหลักสูตร Scala 3 แล้ว!

---

## สารบัญ

- [1. สิ่งที่คุณได้เรียนรู้: สรุปสมบูรณ์](#1-สิ่งที่คุณได้เรียนรู้-สรุปสมบูรณ์)
- [2. Skills Checklist: จาก Beginner ถึง Professional](#2-skills-checklist-จาก-beginner-ถึง-professional)
- [3. Project Ideas สร้าง Portfolio](#3-project-ideas-สร้าง-portfolio)
- [4. Resources สำหรับการเรียนต่อ](#4-resources-สำหรับการเรียนต่อ)
- [5. Community Involvement](#5-community-involvement)
- [6. Professional Scala Development](#6-professional-scala-development)
- [7. Final Coding Challenge](#7-final-coding-challenge)
- [คำส่งท้าย](#คำส่งท้าย)

---

## 1. สิ่งที่คุณได้เรียนรู้: สรุปสมบูรณ์

### Module 1: พื้นฐาน Scala (Parts 1-20)

```scala
// สิ่งที่คุณรู้แล้ว:
// ✓ Syntax และ semantics ของ Scala 3
// ✓ Immutability และ pure functions
// ✓ Pattern matching แบบ exhaustive
// ✓ Type inference
// ✓ Case classes และ sealed traits

// ตัวอย่างที่คุณทำได้ตอนนี้:
enum Shape:
  case Circle(radius: Double)
  case Rectangle(width: Double, height: Double)
  case Triangle(base: Double, height: Double)

def area(shape: Shape): Double = shape match
  case Shape.Circle(r)      => Math.PI * r * r
  case Shape.Rectangle(w, h) => w * h
  case Shape.Triangle(b, h)  => 0.5 * b * h

def describe(shape: Shape): String = shape match
  case Shape.Circle(r) if r > 10 => "Large circle"
  case Shape.Circle(_)            => "Small circle"
  case Shape.Rectangle(w, h) if w == h => "Square"
  case Shape.Rectangle(_, _)           => "Rectangle"
  case Shape.Triangle(_, _)            => "Triangle"
```

### Module 2: Functional Programming Core (Parts 21-40)

```scala
// ✓ Higher-order functions
// ✓ Option, Either สำหรับ error handling
// ✓ Functor, Monad, Applicative (conceptually)
// ✓ Type classes พื้นฐาน
// ✓ For comprehensions

// ตัวอย่าง:
def parseAge(s: String): Either[String, Int] =
  s.toIntOption.toRight(s"'$s' is not a number")
    .flatMap(n => if n > 0 && n < 150 then Right(n) 
                  else Left(s"Age $n is out of range"))

def parseUser(name: String, age: String): Either[String, (String, Int)] =
  for
    a <- parseAge(age)
    n <- if name.nonEmpty then Right(name)
         else Left("Name cannot be empty")
  yield (n, a)

// Type class
trait Printable[A]:
  def print(a: A): String

given Printable[Int] with
  def print(n: Int): String = s"Int($n)"

given [A: Printable]: Printable[List[A]] with
  def print(list: List[A]): String =
    list.map(summon[Printable[A]].print).mkString("[", ", ", "]")
```

### Module 3: cats-effect และ IO (Parts 41-60)

```scala
// ✓ IO monad: referential transparency
// ✓ Resource management
// ✓ Fibers: structured concurrency
// ✓ Concurrent data structures
// ✓ Error handling ใน effect context

import cats.effect.*
import cats.syntax.all.*

// ตัวอย่าง concurrent program ที่คุณเขียนได้:
def fetchAll(urls: List[String]): IO[List[String]] =
  urls.parTraverse { url =>
    IO.println(s"Fetching $url") >>
    IO.pure(s"Content of $url") // simulate
  }

def withTimeout[A](fa: IO[A], duration: scala.concurrent.duration.FiniteDuration): IO[A] =
  fa.timeout(duration).handleErrorWith {
    case _: java.util.concurrent.TimeoutException =>
      IO.raiseError(new Exception(s"Operation timed out after $duration"))
    case e => IO.raiseError(e)
  }
```

### Module 4: fs2 Streaming (Parts 61-70)

```scala
// ✓ Stream primitives
// ✓ Pipes และ transformation
// ✓ Error handling ใน streams
// ✓ Resource-safe streams
// ✓ Concurrent streams

import fs2.*
import cats.effect.*

// Stream processing pipeline:
def processLogs(path: String): Stream[IO, LogEntry] =
  fs2.io.file.Files[IO]
    .readAll(fs2.io.file.Path(path))
    .through(fs2.text.utf8.decode)
    .through(fs2.text.lines)
    .filter(_.nonEmpty)
    .map(parseLine)
    .filter(_.isRight)
    .map(_.toOption.get)

def parseLine(line: String): Either[String, LogEntry] =
  line.split(" ").toList match
    case level :: msg :: rest =>
      Right(LogEntry(level, (msg :: rest).mkString(" ")))
    case _ =>
      Left(s"Cannot parse: $line")

case class LogEntry(level: String, message: String)
```

### Module 5: HTTP APIs (Parts 71-80)

```scala
// ✓ http4s routes
// ✓ Middleware
// ✓ Authentication
// ✓ Request/Response handling
// ✓ WebSocket

import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import io.circe.generic.auto.*

case class ApiResponse[A](data: A, message: String = "Success")

def routes[F[_]: cats.effect.Sync]: HttpRoutes[F] = HttpRoutes.of[F] {
  case GET -> Root / "health" =>
    Ok(ApiResponse("ok"))
  
  case req @ POST -> Root / "echo" =>
    for
      body <- req.as[String]
      resp <- Ok(ApiResponse(body, "Echoed"))
    yield resp
}
```

### Module 6: Database และ Infrastructure (Parts 81-94)

```scala
// ✓ Doobie: type-safe SQL
// ✓ Redis caching
// ✓ Kafka messaging
// ✓ Docker deployment
// ✓ Testing strategies

import doobie.*
import doobie.implicits.*

// Database operations:
def findUserByEmail(email: String): ConnectionIO[Option[User]] =
  sql"SELECT id, name, email FROM users WHERE email = $email"
    .query[User]
    .option

def createUser(user: User): ConnectionIO[User] =
  sql"""
    INSERT INTO users (id, name, email)
    VALUES (${user.id}, ${user.name}, ${user.email})
    RETURNING id, name, email
  """.query[User].unique
```

### Module 7: Advanced Topics (Parts 95-99)

```scala
// ✓ Advanced type system
// ✓ Concurrent data structures
// ✓ Real-world project architecture
// ✓ Streaming analytics
// ✓ Scala ecosystem

// Type-safe builder pattern:
case class QueryBuilder[F <: Tuple] private(
  table: String,
  filters: List[String],
  columns: F
):
  def select[A](col: String): QueryBuilder[A *: F] =
    QueryBuilder(table, filters, ???)
  
  def where(condition: String): QueryBuilder[F] =
    copy(filters = condition :: filters)
  
  def toSQL: String =
    s"SELECT ... FROM $table WHERE ${filters.mkString(" AND ")}"
```

---

## 2. Skills Checklist: จาก Beginner ถึง Professional

### Level 1: Beginner ✓

```
□ เขียน Scala syntax ได้ถูกต้อง
□ ใช้ immutable values (val) ได้
□ เขียน functions และ methods ได้
□ ใช้ Option และ Either ได้
□ Pattern matching พื้นฐาน
□ Case classes และ sealed traits
□ Collections: List, Map, Set
□ For comprehensions
□ สร้าง sbt project ได้
□ Run tests ได้
```

### Level 2: Intermediate ✓

```
□ Type classes: สร้างและใช้งาน
□ Higher-kinded types: เข้าใจ F[_]
□ Implicit/given: ลำดับ resolution
□ Functor, Applicative, Monad: ใช้งาน
□ IO monad: pure functional effects
□ Resource: safe resource management
□ Fiber: structured concurrency
□ fs2 Streams: basic pipelines
□ http4s: REST API
□ Doobie: basic database ops
□ circe: JSON encode/decode
□ Unit testing ด้วย ScalaTest/MUnit
□ Docker: containerize application
```

### Level 3: Advanced ✓

```
□ Tagless final pattern
□ Free monads (conceptual)
□ cats-effect: Ref, Semaphore, MVar
□ Queue, Topic, SignallingRef
□ fs2: advanced streaming, back-pressure
□ Kafka integration
□ Redis caching strategies
□ Authentication & authorization
□ Middleware patterns
□ Error handling strategies
□ Property-based testing
□ Integration testing
□ Performance optimization basics
□ Deployment: Docker Compose
```

### Level 4: Expert (เป้าหมายต่อไป)

```
□ Type-level programming
□ Match types
□ Macros (Scala 3)
□ Akka/Pekko actors
□ Event sourcing + CQRS
□ Distributed systems patterns
□ Apache Spark
□ Performance profiling
□ JVM internals
□ Open source contribution
□ Architecture design
□ Leading teams
```

---

## 3. Project Ideas สร้าง Portfolio

### Beginner Projects (1-2 สัปดาห์)

```scala
// Project 1: Command-line Todo App
// Skills: IO, State management, File I/O
object TodoApp extends IOApp:
  case class Todo(id: Int, title: String, done: Boolean)
  
  def run(args: List[String]): IO[ExitCode] = ???
  
  // Features:
  // - add/remove/complete todos
  // - save/load จาก file (JSON)
  // - filter by status

// Project 2: Currency Converter
// Skills: HTTP client, JSON, error handling
def convertCurrency(
  amount: Double, 
  from: String, 
  to: String
): IO[Either[String, Double]] = ???
// - Call external API
// - Cache rates
// - Handle errors

// Project 3: Password Manager CLI
// Skills: Encryption, File I/O, User input
def storePassword(service: String, password: String): IO[Unit] = ???
def getPassword(service: String): IO[Option[String]] = ???
```

### Intermediate Projects (2-4 สัปดาห์)

```scala
// Project 4: Blog API
// Skills: http4s, doobie, auth, CRUD
// Endpoints:
// POST /auth/register
// POST /auth/login
// GET  /posts
// POST /posts
// PUT  /posts/:id
// DELETE /posts/:id
// POST /posts/:id/comments
// GET  /posts/:id/comments

// Project 5: Real-time Chat
// Skills: WebSocket, Topic, SignallingRef
// Features:
// - Multiple chat rooms
// - User authentication
// - Message history
// - Online status

// Project 6: File Processor
// Skills: fs2, concurrent processing
def processFiles(dir: String): Stream[IO, ProcessedFile] = ???
// - Watch directory for new files
// - Process with configurable pipeline
// - Output statistics
// - Error recovery
```

### Advanced Projects (1-2 เดือน)

```scala
// Project 7: E-commerce Platform
// Full stack features:
// - Product catalog with search
// - Shopping cart (persistent)
// - Order management
// - Payment simulation
// - Email notifications
// - Admin dashboard
// - Docker deployment

// Project 8: Metrics Aggregation Service
// Similar to Part 98 project:
// - Kafka ingestion
// - Real-time processing
// - TimescaleDB storage
// - Grafana dashboard
// - Alerting

// Project 9: Distributed Task Queue
// Skills: Kafka, Redis, cats-effect
// Features:
// - Task submission API
// - Worker pools
// - Priority queues
// - Dead letter queue
// - Monitoring dashboard
// - Retry with backoff

// Project 10: GraphQL API
// Skills: Caliban (ZIO) or Sangria
// Features:
// - Type-safe schema
// - Subscriptions
// - Dataloader (N+1 prevention)
// - Auth integration
```

### Open Source Contribution Ideas

```
1. Add tests to cats, cats-effect, fs2
2. Improve documentation
3. Fix "good first issue" bugs
4. Write tutorials/blog posts
5. Add Scala 3 support to libraries
6. Create new library filling a gap
7. Improve Metals/IntelliJ plugin
```

---

## 4. Resources สำหรับการเรียนต่อ

### หนังสือที่แนะนำ

```
Foundational:
1. "Programming in Scala 5th Ed" - Odersky et al.
   การอ้างอิงหลักสำหรับ Scala

2. "Functional Programming in Scala" (Red Book)
   - Chiusano & Bjarnason
   - สอน FP จากพื้นฐาน
   - มี exercises ที่ดีมาก

3. "Scala with Cats" (ฟรี)
   - https://www.scalawithcats.com
   - cats library ในเชิงลึก

Advanced:
4. "Category Theory for Programmers" - Bartosz Milewski
   - Theory เบื้องหลัง FP
   - ฟรีออนไลน์

5. "Designing Data-Intensive Applications" - Kleppmann
   - ไม่ใช่ Scala specific แต่ essential
   - Distributed systems

6. "Essential Effects" - Adam Rosien
   - cats-effect เชิงลึก
```

### Online Courses

```
Coursera (ฟรี audit):
- "Functional Programming Principles in Scala" - Odersky
  https://www.coursera.org/learn/progfun1
  
- "Functional Program Design in Scala" - Odersky
  https://www.coursera.org/learn/progfun2

- "Parallel programming" - Prokopec & Odersky
  https://www.coursera.org/learn/parprog1

Rock the JVM (paid, excellent):
- "Scala & Functional Programming Essentials"
- "Advanced Scala and Functional Programming"
- "Akka Essentials"
- "Apache Spark with Scala"
- https://rockthejvm.com

YouTube (ฟรี):
- Rock the JVM YouTube channel
- Scala Days conference talks
- ScalaCon talks
```

### Blogs และ Articles

```
Technical Blogs:
- https://typelevel.org/blog (cats-effect, cats)
- https://zio.dev/blog
- https://rockthejvm.com/articles
- https://blog.rockthejvm.com
- https://scalac.io/blog

Personal Blogs ที่ดี:
- Adam Rosien: https://inner-product.com
- Fabio Labella: articles on cats-effect
- Daniel Spiewak: various FP topics
- John De Goes: ZIO focused

Papers (สำหรับคนชอบ theory):
- "Scala: A Scalable Language" by Odersky (2006)
- "Functional Reactive Programming" 
- "Algebraic Effects for the Rest of Us"
```

### Tools ที่ควรรู้จัก

```
IDE:
- IntelliJ IDEA + Scala plugin (most popular)
- VS Code + Metals (improving)
- Emacs/Vim + Metals

Build Tools:
- sbt (standard)
- Mill (alternative, faster)
- Gradle Scala plugin

Formatters:
- Scalafmt (standard)

Linters:
- Scalafix (refactoring + linting)
- WartRemover (strict FP linting)

Profiling:
- JProfiler
- YourKit
- async-profiler (ฟรี, excellent)

Monitoring:
- Prometheus + Grafana
- Datadog
- New Relic
```

---

## 5. Community Involvement

### เริ่มมีส่วนร่วมกับ Community

```
สัปดาห์แรก:
1. Join Scala Discord: https://discord.gg/scala
2. Join Typelevel Discord (ถ้าใช้ cats)
3. Follow Scala Twitter/X accounts
4. Subscribe r/scala

เดือนแรก:
5. ตอบคำถามใน Scala Discord
6. ตอบคำถามบน Stack Overflow
7. เขียน blog post แรก (แม้แต่สั้นๆ)

หลังจากนั้น:
8. เข้าร่วม Scala meetup ใกล้บ้าน
9. Submit CFP ที่ conference
10. Contribute to open source
```

### การเขียน Blog Post

```scala
// หัวข้อที่ดีสำหรับ blog post แรก:

// 1. "เรียน X ใน Scala ใน 5 นาที"
// - อธิบาย concept ที่คุณเพิ่งเรียน
// - ง่ายๆ สั้นๆ

// 2. "ปัญหาที่ฉันเจอกับ X และวิธีแก้"
// - Document ปัญหาจริงที่คุณเจอ
// - อาจช่วยคนอื่นได้มาก

// 3. "Build X ด้วย Scala ใน Weekend"
// - Tutorial ทำตาม
// - มี GitHub repo ประกอบ

// 4. "เปรียบเทียบ Library X กับ Library Y"
// - Trade-offs ที่คุณค้นพบ
// - Real experience

// Platforms สำหรับ publish:
// - dev.to (ฟรี, friendly)
// - Medium (paid readers program)
// - GitHub Pages (ฟรี)
// - Hashnode (ฟรี)
```

### Mentoring และ Teaching

```
เมื่อคุณ comfortable:
1. ตอบคำถามใน Discord
2. ช่วย review PR ของ beginners
3. สอน FP ให้เพื่อนที่ทำงาน
4. เขียน tutorials สำหรับ beginners
5. Speak ที่ local meetup

"The best way to learn is to teach"
```

---

## 6. Professional Scala Development

### Best Practices ใน Production

```scala
// 1. Error handling strategy
// ใช้ typed errors, ไม่ throw exceptions
sealed trait DomainError extends Throwable
case class NotFound(id: String) extends DomainError
case class ValidationFailed(field: String, msg: String) extends DomainError

// 2. Logging ที่ดี
import org.typelevel.log4cats.*
import org.typelevel.log4cats.slf4j.*

class Service[F[_]: Async: Logger]:
  def process(id: String): F[Result] =
    Logger[F].info(s"Processing $id") >>
    doWork(id)
      .flatTap(r => Logger[F].info(s"Completed $id: $r"))
      .handleErrorWith { e =>
        Logger[F].error(e)(s"Failed to process $id") >>
        Async[F].raiseError(e)
      }

// 3. Configuration ที่ดี
// - ใช้ PureConfig หรือ similar
// - Validate ตั้งแต่ startup
// - Log ค่า config (แต่ mask sensitive values)

// 4. Health checks
case class HealthStatus(
  service: String,
  status: String,  // "UP", "DOWN", "DEGRADED"
  dependencies: Map[String, String]
)

// 5. Metrics
// - ใช้ Prometheus หรือ similar
// - Track: latency, throughput, errors, resource usage
// - Alert บน SLOs (Service Level Objectives)

// 6. Graceful shutdown
// ใช้ cats-effect Resource สำหรับ cleanup
val app: Resource[IO, Unit] = for
  server   <- httpServer
  kafka    <- kafkaProducer
  database <- databasePool
yield ()

// app.useForever จะ cleanup ทุก resource เมื่อ shutdown
```

### Code Review Checklist

```
เมื่อ review Scala code ดูที่:

Correctness:
□ Error handling ครบถ้วน?
□ Edge cases handled?
□ Type safety?
□ Resource leaks?

Functional Style:
□ Mutation ที่ไม่จำเป็น?
□ Exceptions ที่ไม่ควรใช้?
□ Side effects ใน pure code?
□ Pattern matching exhaustive?

Performance:
□ Unnecessary allocations?
□ N+1 queries?
□ Cache ที่เหมาะสม?
□ Blocking operations?

Readability:
□ ชื่อ variable/function ชัดเจน?
□ Complex type ควรมี type alias?
□ Comments ที่จำเป็น?
□ ขนาด function ที่เหมาะสม?

Testing:
□ Unit tests ครอบ happy path?
□ Unit tests ครอบ error cases?
□ Integration tests ที่ critical paths?
□ Property-based tests?
```

### Team Practices

```scala
// Shared utilities ที่ทุกคนควรรู้:

// 1. Retry with exponential backoff
import cats.effect.*
import scala.concurrent.duration.*

def withRetry[F[_]: Temporal, A](
  fa: F[A],
  maxRetries: Int = 3,
  initialDelay: FiniteDuration = 100.milliseconds
): F[A] =
  def loop(remaining: Int, delay: FiniteDuration): F[A] =
    fa.handleErrorWith { e =>
      if remaining <= 0 then Temporal[F].raiseError(e)
      else
        Temporal[F].sleep(delay) >>
        loop(remaining - 1, delay * 2)
    }
  loop(maxRetries, initialDelay)

// 2. Circuit breaker pattern
// ใช้ library: resilience4j หรือ cats-effect semaphore

// 3. Pagination helper
case class Page[A](items: List[A], total: Long, page: Int, pageSize: Int):
  def hasNext: Boolean = (page + 1) * pageSize < total
  def hasPrev: Boolean = page > 0

def paginate[F[_]: Functor, A](
  fetchItems: (Int, Int) => F[List[A]],
  countItems: F[Long],
  page: Int,
  pageSize: Int
): F[Page[A]] =
  (fetchItems(page * pageSize, pageSize), countItems).mapN { (items, total) =>
    Page(items, total, page, pageSize)
  }
```

---

## 7. Final Coding Challenge

ทดสอบทักษะทั้งหมดที่คุณเรียนมาด้วย challenge สุดท้ายนี้!

### Challenge: สร้าง Mini Event Sourcing System

```scala
// ===================================================
// CHALLENGE: Mini Event Sourcing
// ===================================================
// สร้าง bank account system ด้วย event sourcing
// 
// Requirements:
// 1. Account commands: OpenAccount, Deposit, Withdraw, CloseAccount
// 2. Account events:   AccountOpened, Deposited, Withdrawn, AccountClosed
// 3. State: Account (id, balance, status, history)
// 4. Store events ใน memory (Ref)
// 5. Rebuild state จาก events
// 6. HTTP API: POST /accounts, POST /accounts/:id/deposit, 
//              POST /accounts/:id/withdraw, GET /accounts/:id
// 7. Error handling: overdraft, account not found, closed account
// ===================================================

// Starter code - complete the implementation!

import cats.effect.*
import cats.effect.std.*
import cats.syntax.all.*
import java.util.UUID
import java.time.Instant

// Domain
case class AccountId(value: UUID) extends AnyVal

enum AccountStatus:
  case Active, Closed

case class Account(
  id: AccountId,
  owner: String,
  balance: BigDecimal,
  status: AccountStatus,
  events: List[AccountEvent]
)

// Commands
sealed trait AccountCommand
case class OpenAccount(owner: String) extends AccountCommand
case class Deposit(id: AccountId, amount: BigDecimal) extends AccountCommand
case class Withdraw(id: AccountId, amount: BigDecimal) extends AccountCommand
case class CloseAccount(id: AccountId) extends AccountCommand

// Events
sealed trait AccountEvent:
  def timestamp: Instant

case class AccountOpened(
  id: AccountId, owner: String, timestamp: Instant
) extends AccountEvent

case class Deposited(
  id: AccountId, amount: BigDecimal, timestamp: Instant
) extends AccountEvent

case class Withdrawn(
  id: AccountId, amount: BigDecimal, timestamp: Instant
) extends AccountEvent

case class AccountClosed(
  id: AccountId, timestamp: Instant
) extends AccountEvent

// Errors
sealed trait AccountError extends Exception
case class AccountNotFound(id: AccountId) extends AccountError
case class InsufficientFunds(available: BigDecimal, requested: BigDecimal) extends AccountError
case class AccountAlreadyClosed(id: AccountId) extends AccountError
case class InvalidAmount(amount: BigDecimal) extends AccountError

// Event Store
trait EventStore[F[_]]:
  def append(event: AccountEvent): F[Unit]
  def getEvents(id: AccountId): F[List[AccountEvent]]
  def getAllEvents: F[List[AccountEvent]]

class InMemoryEventStore[F[_]: Concurrent](
  store: Ref[F, Map[AccountId, List[AccountEvent]]]
) extends EventStore[F]:
  def append(event: AccountEvent): F[Unit] =
    val id = event match
      case e: AccountOpened  => e.id
      case e: Deposited      => e.id
      case e: Withdrawn      => e.id
      case e: AccountClosed  => e.id
    store.update(m =>
      m.updatedWith(id) {
        case None    => Some(List(event))
        case Some(es) => Some(es :+ event)
      }
    )
  
  def getEvents(id: AccountId): F[List[AccountEvent]] =
    store.get.map(_.getOrElse(id, Nil))
  
  def getAllEvents: F[List[AccountEvent]] =
    store.get.map(_.values.flatten.toList.sortBy(_.timestamp))

object InMemoryEventStore:
  def create[F[_]: Concurrent]: F[InMemoryEventStore[F]] =
    Ref[F].of(Map.empty[AccountId, List[AccountEvent]])
      .map(InMemoryEventStore(_))

// Account Service
class AccountService[F[_]: Sync: Temporal](eventStore: EventStore[F]):
  // Rebuild account state จาก events
  def rebuildAccount(events: List[AccountEvent]): Option[Account] =
    events.foldLeft(Option.empty[Account]) {
      case (None, AccountOpened(id, owner, _)) =>
        Some(Account(id, owner, BigDecimal(0), AccountStatus.Active, List(AccountOpened(id, owner, ???))))
      
      case (Some(acc), Deposited(_, amount, _)) =>
        Some(acc.copy(balance = acc.balance + amount))
      
      case (Some(acc), Withdrawn(_, amount, _)) =>
        Some(acc.copy(balance = acc.balance - amount))
      
      case (Some(acc), AccountClosed(_, _)) =>
        Some(acc.copy(status = AccountStatus.Closed))
      
      case (state, _) => state
    }
  
  def handleCommand(cmd: AccountCommand): F[AccountEvent] = cmd match
    case OpenAccount(owner) =>
      for
        id  <- Sync[F].delay(AccountId(UUID.randomUUID()))
        now <- Temporal[F].realTimeInstant
        event = AccountOpened(id, owner, now)
        _ <- eventStore.append(event)
      yield event
    
    case Deposit(id, amount) =>
      for
        _ <- if amount <= 0 then Sync[F].raiseError(InvalidAmount(amount))
             else Sync[F].unit
        events  <- eventStore.getEvents(id)
        account <- rebuildAccount(events) match
          case None    => Sync[F].raiseError(AccountNotFound(id))
          case Some(a) => Sync[F].pure(a)
        _ <- if account.status == AccountStatus.Closed
             then Sync[F].raiseError(AccountAlreadyClosed(id))
             else Sync[F].unit
        now <- Temporal[F].realTimeInstant
        event = Deposited(id, amount, now)
        _ <- eventStore.append(event)
      yield event
    
    case Withdraw(id, amount) =>
      for
        _ <- if amount <= 0 then Sync[F].raiseError(InvalidAmount(amount))
             else Sync[F].unit
        events  <- eventStore.getEvents(id)
        account <- rebuildAccount(events) match
          case None    => Sync[F].raiseError(AccountNotFound(id))
          case Some(a) => Sync[F].pure(a)
        _ <- if account.status == AccountStatus.Closed
             then Sync[F].raiseError(AccountAlreadyClosed(id))
             else Sync[F].unit
        _ <- if account.balance < amount
             then Sync[F].raiseError(InsufficientFunds(account.balance, amount))
             else Sync[F].unit
        now <- Temporal[F].realTimeInstant
        event = Withdrawn(id, amount, now)
        _ <- eventStore.append(event)
      yield event
    
    case CloseAccount(id) =>
      for
        events  <- eventStore.getEvents(id)
        account <- rebuildAccount(events) match
          case None    => Sync[F].raiseError(AccountNotFound(id))
          case Some(a) => Sync[F].pure(a)
        _ <- if account.status == AccountStatus.Closed
             then Sync[F].raiseError(AccountAlreadyClosed(id))
             else Sync[F].unit
        now <- Temporal[F].realTimeInstant
        event = AccountClosed(id, now)
        _ <- eventStore.append(event)
      yield event
  
  def getAccount(id: AccountId): F[Option[Account]] =
    eventStore.getEvents(id).map(rebuildAccount)

// Test this!
object ChallengeTest extends IOApp.Simple:
  def run: IO[Unit] = for
    store   <- InMemoryEventStore.create[IO]
    service  = AccountService[IO](store)
    
    // Open account
    openEvent <- service.handleCommand(OpenAccount("Alice"))
    accountId  = openEvent.asInstanceOf[AccountOpened].id
    _ <- IO.println(s"Opened account: $accountId")
    
    // Deposit
    _ <- service.handleCommand(Deposit(accountId, BigDecimal(1000)))
    _ <- IO.println("Deposited 1000")
    
    // Withdraw
    _ <- service.handleCommand(Withdraw(accountId, BigDecimal(300)))
    _ <- IO.println("Withdrawn 300")
    
    // Check balance
    account <- service.getAccount(accountId)
    _ <- IO.println(s"Balance: ${account.map(_.balance)}")  // Should be 700
    
    // Try overdraft
    _ <- service.handleCommand(Withdraw(accountId, BigDecimal(10000)))
      .handleError(e => IO.println(s"Expected error: $e"))
    
    // Close account
    _ <- service.handleCommand(CloseAccount(accountId))
    _ <- IO.println("Account closed")
    
    // Try to deposit to closed account
    _ <- service.handleCommand(Deposit(accountId, BigDecimal(100)))
      .handleError(e => IO.println(s"Expected error: $e"))
    
    _ <- IO.println("\nChallenge complete! Event sourcing works!")
  yield ()
```

### Extended Challenge (ถ้าต้องการ)

```
1. เพิ่ม HTTP API ด้วย http4s
2. เพิ่ม event projections (materialized views)
3. เพิ่ม snapshots (optimise event replay)
4. เพิ่ม event publishing ไป Kafka
5. เพิ่ม integration tests
6. Deploy ด้วย Docker

Bonus: 
- Implement CQRS (separate read/write models)
- Add eventual consistency between services
```

---

## คำส่งท้าย

```
╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║   คุณได้ผ่านหลักสูตร Scala 3 ที่ครอบคลุมที่สุดแล้ว!           ║
║                                                               ║
║   จาก "Hello World" ไปถึง Production-Ready Systems           ║
║   จาก simple functions ไปถึง Advanced Type System            ║
║   จาก single thread ไปถึง Concurrent, Streaming Systems      ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### สิ่งที่คุณได้พิสูจน์แล้ว:

- **ความอดทน**: หลักสูตร 100 ส่วนไม่ใช่เรื่องง่าย
- **ความพยายาม**: คุณผ่านทุก concept มาได้
- **ทักษะ**: คุณมีพื้นฐาน FP ที่แข็งแกร่ง
- **ความพร้อม**: คุณพร้อมสำหรับ real-world Scala projects

### ขั้นตอนต่อไปที่สำคัญที่สุด:

```
1. BUILD SOMETHING REAL
   ไม่มีอะไรสอนได้ดีเท่า project จริง
   เลือก project idea จากด้านบนและลงมือทำเลย!

2. CONTRIBUTE TO OPEN SOURCE
   แม้แค่ fix typo ใน docs
   ทำให้คุณเป็นส่วนหนึ่งของ community

3. TEACH OTHERS
   อธิบาย concept ที่คุณเรียนให้คนอื่นฟัง
   คุณจะเข้าใจลึกขึ้นอีก

4. KEEP LEARNING
   Scala และ FP มีอะไรให้เรียนอีกมาก
   แต่ foundation ที่คุณมีตอนนี้แข็งแกร่งมากแล้ว

5. SHARE YOUR JOURNEY
   เขียน blog, tweet, หรือพูดที่ meetup
   ประสบการณ์ของคุณมีค่าสำหรับคนที่กำลังเริ่ม
```

### คำกล่าวสุดท้ายจาก Course

```
Functional Programming ไม่ใช่แค่ style การเขียน code
มันเป็นวิธีคิดเกี่ยวกับปัญหา:

- แยก concerns อย่างชัดเจน (pure vs impure)
- Compose small pieces เป็นระบบใหญ่
- Make illegal states unrepresentable
- ทำให้ effects ชัดเจนใน types

เมื่อคุณคิดแบบ FP แล้ว มันจะช่วยทุกภาษาที่คุณเขียน
ไม่ว่าจะเป็น Python, JavaScript, Java, หรือ Haskell

Scala เป็นภาษาที่ให้คุณ:
- เขียน FP ได้อย่างสวยงาม
- Interop กับ Java ecosystem
- Scale จาก scripts ถึง distributed systems

ยินดีด้วยอีกครั้ง! 🎉
คุณเป็น Scala Developer แล้ว!
```

---

## Summary: 100 Parts Journey

```
Parts 1-10:   Scala fundamentals
Parts 11-20:  Collections and standard library  
Parts 21-30:  Pattern matching and ADTs
Parts 31-40:  Functional programming core
Parts 41-50:  cats-effect basics
Parts 51-60:  Concurrency patterns
Parts 61-70:  fs2 streaming
Parts 71-80:  HTTP APIs
Parts 81-90:  Infrastructure
Parts 91-94:  Real-world patterns
Part 95:      Advanced types
Part 96:      Concurrent data structures
Part 97:      Final project - REST API
Part 98:      Final project - Streaming
Part 99:      Scala ecosystem
Part 100:     Course completion ← คุณอยู่ที่นี่!
```

---

*[← Part 99: Scala Ecosystem](part-99-scala-ecosystem.md)*

---

**หลักสูตร Scala 3 สมบูรณ์แล้ว! ขอบคุณที่เรียนจนจบ!**
