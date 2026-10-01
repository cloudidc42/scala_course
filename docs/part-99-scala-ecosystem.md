# ส่วนที่ 99: Scala Ecosystem Overview

## สารบัญ

- [1. Scala Version History และ Roadmap](#1-scala-version-history-และ-roadmap)
- [2. Key Libraries Ecosystem Map](#2-key-libraries-ecosystem-map)
- [3. Typelevel Stack vs ZIO Stack](#3-typelevel-stack-vs-zio-stack)
- [4. Community และ Resources](#4-community-และ-resources)
- [5. Contributing to Open Source](#5-contributing-to-open-source)
- [6. Career Paths กับ Scala](#6-career-paths-กับ-scala)
- [7. Learning Roadmap](#7-learning-roadmap)
- [สรุป](#สรุป)

---

## 1. Scala Version History และ Roadmap

### ประวัติของ Scala

```
2003 - Scala 1.0: Martin Odersky เริ่มที่ EPFL
       ผสม OOP + FP บน JVM เป็นครั้งแรก

2006 - Scala 2.0: Rewrite สมบูรณ์
       เพิ่ม case classes, pattern matching ที่ดีขึ้น

2011 - Scala 2.9: Parallel collections
       Futures และ Promises

2012 - Scala 2.10: Futures และ Promises เสถียร
       Macros (experimental), String interpolation, 
       Value classes, Implicit classes

2013 - Scala 2.11: ปรับปรุง performance
       Modular standard library

2016 - Scala 2.12: Java 8 baseline
       Better lambda support, trait encoding ใหม่

2019 - Scala 2.13: Collections rewrite
       Literal types, โปรแกรมหน้าผาก performance

2021 - Scala 3.0: "Dotty" - การเปลี่ยนแปลงครั้งใหญ่
       New type system, given/using, opaque types,
       enums, union/intersection types, macros ใหม่

2022 - Scala 3.2: Performance improvements
       Better error messages

2023 - Scala 3.3 LTS: Long-Term Support release
       Capture checking (experimental),
       Better type inference

2024+ - Scala 3.4+: Capture checking stable,
        Better tooling, Scala Native improvements
```

### Scala 3 vs Scala 2: ความแตกต่างสำคัญ

```scala
// === Implicits ===
// Scala 2
implicit val ordering: Ordering[Int] = Ordering.Int
implicit def convert(s: String): Int = s.length
implicitly[Ordering[Int]]

// Scala 3
given ordering: Ordering[Int] = Ordering.Int
given Conversion[String, Int] = _.length
summon[Ordering[Int]]

// === Type classes ===
// Scala 2
trait Show[A] {
  def show(a: A): String
}
implicit val intShow: Show[Int] = n => n.toString

// Scala 3
trait Show[A]:
  def show(a: A): String
given Show[Int] with
  def show(a: Int): String = a.toString

// === Enums ===
// Scala 2 (sealed traits + case objects)
sealed trait Color
object Color {
  case object Red extends Color
  case object Green extends Color
  case object Blue extends Color
}

// Scala 3 (enums)
enum Color:
  case Red, Green, Blue

// Parameterized enum
enum Shape:
  case Circle(radius: Double)
  case Rectangle(w: Double, h: Double)
  case Triangle(base: Double, height: Double)

// === Union Types ===
// Scala 2: ไม่มี native union types (ใช้ Either หรือ sealed trait)
type StringOrInt = Either[String, Int]

// Scala 3: native union types
type StringOrInt = String | Int
val value: String | Int = 42
val str: String | Int = "hello"

// === Intersection Types ===
// Scala 3
trait Serializable:
  def serialize: String

trait Loggable:
  def log: Unit

type SerializableAndLoggable = Serializable & Loggable

// === Extension Methods ===
// Scala 2 (implicit class)
implicit class IntOps(val n: Int) extends AnyVal {
  def doubled: Int = n * 2
}

// Scala 3
extension (n: Int)
  def doubled: Int = n * 2
  def tripled: Int = n * 3

// === Opaque Types ===
// Scala 3 (ไม่มีใน Scala 2)
opaque type Meters = Double
object Meters:
  def apply(d: Double): Meters = d
  extension (m: Meters)
    def value: Double = m
    def +(other: Meters): Meters = m + other

// === Match Types ===
// Scala 3 (ไม่มีใน Scala 2)
type ElementType[X] = X match
  case String     => Char
  case List[t]    => t
  case Array[t]   => t
```

### Scala Roadmap (Future)

```
Scala 3.x (ปัจจุบันและอนาคต):
- Capture Checking: ป้องกัน resource leaks ที่ compile time
- Better Staging (staging macros)
- Improved performance
- Better IDE support (Metals, IntelliJ)
- Scala Native improvements
- Scala.js improvements

Scala 4 (ยังไม่แน่ชัด):
- อาจมี breaking changes เพิ่มเติม
- Better interop กับ Java
- More advanced type features
```

---

## 2. Key Libraries Ecosystem Map

### HTTP

```scala
// === http4s: purely functional HTTP ===
// - Tagless final design
// - cats-effect integration
// - Composable middleware
"org.http4s" %% "http4s-ember-server" % "0.23.x"

// === Play Framework: batteries included ===
// - เหมาะกับ traditional web apps
// - Akka/Pekko based
"com.typesafe.play" %% "play" % "2.9.x"

// === Akka HTTP / Pekko HTTP ===
// - High performance
// - Actor-based
"org.apache.pekko" %% "pekko-http" % "1.0.x"

// === ZIO HTTP ===
// - ZIO native
// - High performance
"dev.zio" %% "zio-http" % "3.x"

// === Tapir: type-safe API definitions ===
// - Generate docs, server, client จาก definition เดียว
"com.softwaremill.sttp.tapir" %% "tapir-core" % "1.x"
```

### Database

```scala
// === Doobie: pure functional JDBC ===
// - Cats-effect integration
// - Type-safe SQL
"org.tpolecat" %% "doobie-core" % "1.0.x"

// === Quill: compile-time SQL generation ===
// - Macro-based
// - Type-safe queries
"io.getquill" %% "quill-jdbc" % "4.x"

// === Slick: functional relational mapping ===
// - DSL สำหรับ SQL
// - Async support
"com.typesafe.slick" %% "slick" % "3.x"

// === Skunk: pure Scala PostgreSQL client ===
// - Protocol-level PostgreSQL
// - Prepared statements
// - Session-level type checking
"org.tpolecat" %% "skunk-core" % "0.6.x"

// === MongoDB Scala Driver ===
"org.mongodb.scala" %% "mongo-scala-driver" % "4.x"
```

### JSON

```scala
// === Circe: type-safe JSON ===
// - Derive codecs อัตโนมัติ
// - Cursor-based traversal
"io.circe" %% "circe-core"    % "0.14.x"
"io.circe" %% "circe-generic" % "0.14.x"

// === Play JSON ===
// - ส่วนหนึ่งของ Play framework
// - Macro-based derivation
"com.typesafe.play" %% "play-json" % "2.10.x"

// === uPickle ===
// - Simple, fast
// - Less FP-idiomatic
"com.lihaoyi" %% "upickle" % "3.x"

// === spray-json ===
// - Simple สำหรับ simple cases
"io.spray" %% "spray-json" % "1.3.x"
```

### Streaming

```scala
// === fs2: functional streams ===
// - Composable, resource-safe
// - cats-effect native
"co.fs2" %% "fs2-core" % "3.x"

// === Akka Streams / Pekko Streams ===
// - Back-pressure built-in
// - Mature ecosystem
"org.apache.pekko" %% "pekko-stream" % "1.x"

// === ZIO Streams ===
// - ZIO native
"dev.zio" %% "zio-streams" % "2.x"

// === Kafka integrations ===
"com.github.fd4s" %% "fs2-kafka"  % "3.x"  // fs2
"io.confluent"     % "kafka-streams-scala_2.13" % "7.x"
```

### Testing

```scala
// === ScalaTest: most popular ===
"org.scalatest" %% "scalatest" % "3.2.x" % Test

// === MUnit: lightweight ===
"org.scalameta" %% "munit"             % "0.7.x" % Test
"org.typelevel" %% "munit-cats-effect" % "1.x"   % Test

// === Weaver: cats-effect focused ===
"com.disneystreaming" %% "weaver-cats" % "0.8.x" % Test

// === ScalaCheck: property-based testing ===
"org.scalacheck" %% "scalacheck" % "1.17.x" % Test

// === Hedgehog: better property testing ===
"qa.hedgehog" %% "hedgehog-core" % "0.10.x" % Test
```

---

## 3. Typelevel Stack vs ZIO Stack

### Typelevel Stack

```scala
// Core: cats-effect เป็น foundation
import cats.effect.*
import cats.syntax.all.*

// Stack:
// cats-core: type classes (Functor, Monad, etc.)
// cats-effect: IO, Resource, Fiber
// fs2: streaming
// http4s: HTTP
// doobie: database
// circe: JSON
// redis4cats: Redis
// fs2-kafka: Kafka

// ตัวอย่าง Typelevel style
def program: IO[Unit] =
  for
    config <- AppConfig.load
    _ <- Database.migrate(config.database)
    result <- HttpServer.run(config)
  yield result
```

### ZIO Stack

```scala
// Core: ZIO เป็น foundation
import zio.*
import zio.http.*

// Stack:
// zio: core effect system
// zio-streams: streaming
// zio-http: HTTP
// zio-jdbc: database
// zio-json: JSON
// zio-kafka: Kafka
// zio-redis: Redis

// ตัวอย่าง ZIO style
def program: ZIO[AppConfig & Database, Throwable, Unit] =
  for
    config <- ZIO.service[AppConfig]
    _      <- Database.migrate(config.database)
    result <- HttpServer.run(config)
  yield result
```

### เปรียบเทียบ Typelevel vs ZIO

```
┌──────────────────┬──────────────────────┬──────────────────────┐
│ ลักษณะ           │ Typelevel            │ ZIO                  │
├──────────────────┼──────────────────────┼──────────────────────┤
│ Type signature   │ F[_] (tagless final) │ ZIO[R, E, A]         │
│ Error handling   │ MonadError           │ Built-in E type      │
│ Dependency inj.  │ Implicit parameters  │ ZEnvironment (R)     │
│ Learning curve   │ สูงกว่า              │ อาจง่ายกว่า            │
│ Type complexity  │ สูง                  │ ปานกลาง              │
│ Documentation    │ กระจัดกระจาย          │ รวมศูนย์             │
│ Community        │ กว้าง                │ กำลังเติบโต           │
│ Performance      │ ดี                   │ ดีมาก                │
│ Testing          │ cats-effect-testing  │ zio-test (excellent) │
│ Streaming        │ fs2 (excellent)      │ ZIO Streams          │
│ Enterprise use   │ ใช้กันมาก             │ กำลังเพิ่มขึ้น        │
└──────────────────┴──────────────────────┴──────────────────────┘
```

### Tagless Final (Typelevel approach)

```scala
// Tagless final: abstract เหนือ effect type
trait UserRepository[F[_]]:
  def findById(id: UserId): F[Option[User]]
  def create(user: User): F[User]

// Implementation สำหรับ IO
class PostgresUserRepo(xa: Transactor[IO]) extends UserRepository[IO]:
  def findById(id: UserId): IO[Option[User]] = ???
  def create(user: User): IO[User] = ???

// Implementation สำหรับ test
class InMemoryUserRepo[F[_]: Ref.Make](ref: Ref[F, Map[UserId, User]])
    extends UserRepository[F]:
  def findById(id: UserId): F[Option[User]] = ref.get.map(_.get(id))
  def create(user: User): F[User] = ref.update(_.updated(user.id, user)).as(user)

// Service ที่ generic
class UserService[F[_]: Monad](repo: UserRepository[F]):
  def getOrCreate(id: UserId, default: User): F[User] =
    repo.findById(id).flatMap {
      case Some(user) => user.pure[F]
      case None       => repo.create(default)
    }
```

### ZIO Layer approach

```scala
// ZIO: dependency injection ผ่าน ZLayer
import zio.*

trait UserRepository:
  def findById(id: UserId): Task[Option[User]]
  def create(user: User): Task[User]

case class PostgresUserRepo(ds: javax.sql.DataSource) extends UserRepository:
  def findById(id: UserId): Task[Option[User]] = ZIO.attempt(???)
  def create(user: User): Task[User] = ZIO.attempt(???)

object PostgresUserRepo:
  val layer: ZLayer[javax.sql.DataSource, Nothing, UserRepository] =
    ZLayer.fromFunction(PostgresUserRepo(_))

class UserService(repo: UserRepository):
  def getOrCreate(id: UserId, default: User): Task[User] =
    repo.findById(id).flatMap {
      case Some(user) => ZIO.succeed(user)
      case None       => repo.create(default)
    }

object UserService:
  val layer: ZLayer[UserRepository, Nothing, UserService] =
    ZLayer.fromFunction(UserService(_))

// Program ที่ใช้ ZLayer
val program: ZIO[UserService, Throwable, Unit] =
  ZIO.serviceWithZIO[UserService](_.getOrCreate(???, ???)).unit

// Run ด้วย layers
def run = program.provide(
  UserService.layer,
  PostgresUserRepo.layer,
  ZLayer.fromZIO(ZIO.attempt(getDataSource()))
)
```

### เมื่อไรควรเลือกอะไร

```
เลือก Typelevel stack เมื่อ:
- ทีมมีประสบการณ์ FP แล้ว
- ต้องการ maximum flexibility
- ใช้ existing libraries ใน typelevel ecosystem
- ต้องการ streaming ที่ทรงพลัง (fs2)

เลือก ZIO stack เมื่อ:
- ทีมกำลังเรียน FP
- ต้องการ error channel ที่ชัดเจน
- ต้องการ dependency injection ที่ง่าย
- ต้องการ testing ที่ดี (zio-test)
- ต้องการ documentation ที่ดีกว่า
```

---

## 4. Community และ Resources

### Official Resources

```
Scala Official:
- https://scala-lang.org (official site)
- https://docs.scala-lang.org (documentation)
- https://scastie.scala-lang.org (online playground)
- https://scaladex.scala-lang.org (library search)

Scala Center (non-profit):
- https://scala.epfl.ch
- MOOCs บน Coursera (ฟรี)
- Open source contributions
```

### Learning Resources

```
Books:
- "Programming in Scala" by Odersky, Spoon, Venners
  (อ่านเพิ่ม: https://www.artima.com/pins1ed/)
  
- "Functional Programming in Scala" (Red Book)
  by Chiusano & Bjarnason
  
- "Scala with Cats" (ฟรีออนไลน์)
  https://www.scalawithcats.com
  
- "Essential Effects" by Adam Rosien
  (cats-effect เชิงลึก)

Online Courses:
- Coursera: "Functional Programming Principles in Scala" by Odersky
- Rock the JVM: https://rockthejvm.com (ดีมาก!)
- Scala Exercises: https://www.scala-exercises.org

YouTube:
- Rock the JVM (Daniel Ciocîrlan)
- Scala Days talks
- ScalaCon talks
```

### Community Channels

```
Forums:
- https://users.scala-lang.org (official Scala Users forum)
- https://contributors.scala-lang.org (contributors)
- Stack Overflow: tag "scala"

Discord/Slack:
- Scala Discord: https://discord.gg/scala
- Typelevel Discord: https://discord.gg/typelevel
- ZIO Discord
- Functional Programming Slack

Reddit:
- r/scala
- r/functionalprogramming

Twitter/X: ติดตาม
- @scala_lang
- @odersky (Martin Odersky)
- @djspiewak (Daniel Spiewak, cats-effect)
- @timperrett (ZIO contributor)
```

### Conferences

```
Scala Days:
- ปีละ 2 ครั้ง (EU + US)
- talks จาก core contributors
- https://scaladays.org

ScalaCon:
- Online conference
- https://scalacon.org

Functional Scala:
- London
- https://www.functionalscala.com

LambdaConf:
- FP conference (not Scala-specific)
- talks ที่ excellent
```

---

## 5. Contributing to Open Source

### เริ่มต้น Contributing

```bash
# 1. เลือก project ที่สนใจ
# ดู: https://scalacenter.github.io/scala-developer-survey/

# 2. ดู issues ที่เหมาะสมสำหรับ beginners
# มองหา labels: "good first issue", "help wanted", "beginner friendly"

# Popular projects:
# - scala/scala (compiler)
# - typelevel/cats
# - typelevel/cats-effect
# - zio/zio
# - http4s/http4s
# - tpolecat/doobie
# - circe/circe
# - fs2/fs2

# 3. Fork และ clone
git clone https://github.com/your-username/cats.git
cd cats
git remote add upstream https://github.com/typelevel/cats.git

# 4. สร้าง branch
git checkout -b fix/issue-123-description

# 5. ทำ changes และ test
sbt test

# 6. Push และ create PR
git push origin fix/issue-123-description
```

### Typical Open Source Workflow

```scala
// ตัวอย่าง: contribute ไปยัง cats (simplified)

// 1. อ่าน CONTRIBUTING.md ก่อนเสมอ

// 2. เพิ่ม tests ก่อน (TDD approach)
// ใน project: cats-tests/src/test/scala/...
class MySpec extends CatsSuite:
  test("my new feature should work"):
    // ...

// 3. Implement feature
// ใน project: core/src/main/scala/cats/...

// 4. เพิ่ม docs ถ้า needed
// ใน project: docs/src/main/mdoc/...

// 5. เพิ่ม entry ใน CHANGELOG หรือ release notes

// 6. Run full test suite
// sbt "+test"  // ทดสอบกับทุก Scala version

// 7. Format code
// sbt scalafmtAll

// 8. สร้าง meaningful commit message
// "fix(MapK): handle None case in traverseK
//  
//  Fixes #1234"
```

### Documentation Contributions

```scala
// สำคัญมาก! Docs contributions ยินดีรับ

// 1. mdoc: Scala documentation tool
// - Type-checks code ใน markdown
// - Generate docs site

// ตัวอย่าง mdoc format:
/*
```scala mdoc
import cats.syntax.all.*

val x: Option[Int] = Some(1)
val y: Option[Int] = Some(2)

(x, y).mapN(_ + _) // mdoc will show output
```
*/

// 2. Scaladoc: inline documentation
/**
 * Applies a binary function to two effects, combining their results.
 *
 * @example
 * ```scala
 * import cats.effect.*
 * 
 * val r = (IO.pure(1), IO.pure(2)).mapN(_ + _)
 * // r: IO[Int] = IO(3)
 * ```
 */
def mapN[A, B, C](fa: F[A], fb: F[B])(f: (A, B) => C): F[C] = ???
```

---

## 6. Career Paths กับ Scala

### ประเภทงานที่ใช้ Scala

```
1. Backend Engineer (FP Focus)
   - ทำ REST APIs, microservices
   - ใช้ http4s, akka-http, play
   - Salary: สูงกว่า average Java engineer

2. Data Engineer / Platform Engineer
   - Apache Spark (Scala is native language)
   - Kafka, Flink
   - ETL pipelines
   - Big data processing
   - Salary: สูงมาก

3. Distributed Systems Engineer
   - Akka/Pekko actors
   - ระบบ fault-tolerant
   - Event sourcing, CQRS
   - Salary: สูงมาก

4. Compiler / Language Engineer
   - Work on Scala compiler
   - Language tooling
   - Meta-programming
   - Rare, highly specialized

5. FinTech / Trading Systems
   - Low-latency trading
   - Risk calculations
   - Many financial companies ใช้ Scala
```

### บริษัทที่ใช้ Scala

```
Large companies:
- Twitter (now X) - สร้าง Finagle, Finatra
- LinkedIn - Kafka เขียนด้วย Scala/Java
- Netflix - Spark for analytics
- Airbnb - Data pipelines
- Stripe - Payment processing
- Goldman Sachs - Trading systems
- Morgan Stanley - Financial systems
- Walmart - E-commerce platform

Smaller/startup:
- Databricks - Spark company
- Confluent - Kafka company
- Wix - Website builder
- SoundCloud - Music platform
- Guardian - News website
- ING Bank - Dutch bank
- Zalando - Fashion e-commerce
```

### Skills ที่ต้องการ

```
Essential:
✓ Scala (obviously)
✓ JVM knowledge (GC, memory, threads)
✓ Functional programming concepts
✓ cats-effect หรือ ZIO
✓ SQL + database design

Important:
✓ Kafka / message queues
✓ Docker / Kubernetes
✓ REST API design
✓ Apache Spark (สำหรับ data engineering)
✓ Git workflow

Nice to have:
✓ Type-level programming
✓ Akka/Pekko
✓ Distributed systems concepts (CAP, CRDT)
✓ Performance tuning
✓ Open source contributions
```

### Tips สำหรับหางาน Scala

```
1. Build visible portfolio
   - GitHub projects ด้วย Scala
   - Contribute to open source
   - Technical blog posts
   - สร้าง projects จาก courses นี้!

2. Networking
   - เข้าร่วม Scala meetups
   - Contribute online (Discourse, Discord)
   - LinkedIn ใส่ Scala projects

3. Interview prep
   - FP concepts: Functor, Monad, Applicative
   - cats-effect: IO, Fiber, Resource
   - Concurrency: Ref, MVar, Queue
   - Type system: HKT, type classes
   - System design กับ FP

4. Salary negotiation
   - Scala engineers หายาก
   - อย่า settle for average!
   - Research salaries: levels.fyi, glassdoor
```

---

## 7. Learning Roadmap

### Beginner → Junior (3-6 เดือน)

```
เดือน 1-2: Scala Basics
□ Syntax, variables, expressions
□ Functions, methods
□ Collections: List, Map, Set
□ Pattern matching
□ Case classes, sealed traits
□ Option, Either (basics)
□ Basic OOP: classes, traits, objects
□ sbt basics

เดือน 3-4: FP Fundamentals
□ Pure functions, immutability
□ Higher-order functions
□ map, flatMap, filter
□ For comprehensions
□ Option, Either (advanced)
□ Type classes (basics)
□ Implicit parameters (Scala 3: given/using)

เดือน 5-6: Real World
□ cats-effect: IO, Resource
□ http4s: basic HTTP
□ doobie: basic SQL
□ JSON ด้วย circe
□ Testing ด้วย ScalaTest
□ Docker basics
□ Build one project!
```

### Junior → Mid-Level (6-12 เดือน)

```
เดือน 7-9: Concurrency
□ cats-effect: Fiber, Ref, Semaphore
□ fs2: Streams, Pipes
□ Queue, Topic, SignallingRef
□ Error handling strategies
□ Resource management

เดือน 10-12: Advanced Topics
□ Type classes in depth
□ Tagless final pattern
□ Kafka integration
□ Redis caching
□ Advanced error handling
□ Performance basics
□ Build microservice project!
```

### Mid-Level → Senior (1-2 ปี)

```
ปีที่ 1-2:
□ Type system mastery (HKT, dependent types)
□ Akka/Pekko actors (if needed)
□ Distributed systems patterns
□ Event sourcing, CQRS
□ Architecture patterns
□ Performance optimization
□ Apache Spark (if data engineering)
□ Leading technical discussions
□ Contribute to open source
□ Mentor others
□ Build production systems
```

### Specialized Paths

```
FP Specialist:
□ Category theory basics
□ Free monads
□ Recursion schemes
□ Optics (Monocle)
□ Type-level programming

Data Engineer:
□ Apache Spark mastery
□ Kafka Streams
□ Flink
□ Data warehouse concepts
□ SQL optimization

Distributed Systems:
□ Akka Cluster
□ Event sourcing
□ CRDT
□ Consensus algorithms
□ Performance engineering
```

---

## สรุป

Scala Ecosystem มีทุกสิ่งที่จำเป็น:

| หมวด | ตัวเลือกหลัก |
|------|-------------|
| HTTP | http4s, Play, ZIO HTTP |
| Database | Doobie, Skunk, Quill |
| Streaming | fs2, Akka Streams, ZIO Streams |
| JSON | Circe, Play JSON |
| Testing | ScalaTest, MUnit, ZIO Test |
| Effects | cats-effect, ZIO |
| Messaging | fs2-kafka, Alpakka Kafka |

### Key Takeaways

1. Scala 3 เป็น major improvement - เรียน Scala 3 ดีกว่า Scala 2
2. ทั้ง Typelevel และ ZIO เป็น production-ready
3. Community เล็กแต่ high quality
4. Scala ใช้กันมากใน FinTech, Data Engineering, Backend
5. Investment ในการเรียน FP คุ้มค่าระยะยาว

---

*[← Part 98: Final Project Streaming](part-98-project-final-streaming.md) | [Part 100: Course Completion →](part-100-course-completion.md)*
