# Part 68: Cats MTL และ Monad Transformers

## สารบัญ

1. [แนะนำ Monad Transformers](#1-แนะนำ-monad-transformers)
2. [Transformer Stack พื้นฐาน](#2-transformer-stack-พื้นฐาน)
3. [OptionT](#3-optiont)
4. [EitherT](#4-eithert)
5. [WriterT](#5-writert)
6. [StateT](#6-statet)
7. [Cats MTL Type Classes](#7-cats-mtl-type-classes)
8. [MTL vs Tagless Final](#8-mtl-vs-tagless-final)
9. [Application สมบูรณ์ด้วย MTL](#9-application-สมบูรณ์ด้วย-mtl)
10. [สรุป](#10-สรุป)

---

## 1. แนะนำ Monad Transformers

Monad Transformers แก้ปัญหา "monad stacking" - การรวม effects หลายอย่างเข้าด้วยกัน

### ปัญหาที่ Monad Transformers แก้

```scala
// ปัญหา: ต้องการ Effect หลายอย่างพร้อมกัน
// - Async (Future/IO)
// - Optional value (Option)
// - Error handling (Either)
// - State
// - Logging

// ❌ ไม่มี Transformer: nested types ที่จัดการยาก
def findUser(id: Long): Future[Option[User]]
def findPosts(userId: Long): Future[Option[List[Post]]]

val result: Future[Option[(User, List[Post])]] = for
  userOpt  <- findUser(userId)
  postsOpt <- userOpt.fold(Future.successful(None: Option[List[Post]]))(u =>
    findPosts(u.id)
  )
yield for
  user  <- userOpt
  posts <- postsOpt
yield (user, posts)
// ซับซ้อน อ่านยาก

// ✅ ด้วย Monad Transformer: compose ง่ายขึ้น
import cats.data.OptionT
import cats.implicits.*

def findUser(id: Long): OptionT[Future, User] = ???
def findPosts(userId: Long): OptionT[Future, List[Post]] = ???

val result: OptionT[Future, (User, List[Post])] = for
  user  <- findUser(userId)
  posts <- findPosts(user.id)
yield (user, posts)
// สะอาด อ่านง่ายกว่ามาก
```

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.typelevel" %% "cats-core"   % "2.10.0",
  "org.typelevel" %% "cats-effect" % "3.5.1",
  "org.typelevel" %% "cats-mtl"    % "1.3.0"
)
```

---

## 2. Transformer Stack พื้นฐาน

### ทำความเข้าใจ Transformer Stack

```
Monad Transformer Stack:

EitherT[OptionT[IO, *], Error, *]
│
├── IO: ชั้น outer สุด - async effect
│   ├── OptionT: middle layer - optional values
│   │   ├── EitherT: inner layer - error handling
│   │   └── value: A
```

### ตัวอย่าง Stack

```scala
import cats.*
import cats.data.*
import cats.effect.*
import cats.implicits.*

type AppError = String
type App[A] = EitherT[IO, AppError, A]

// ใช้งาน
def validateAge(age: Int): App[Int] =
  if age >= 0 && age <= 150
  then EitherT.rightT[IO, AppError](age)
  else EitherT.leftT[IO, Int](s"Invalid age: $age")

def fetchUser(id: Long): App[User] =
  EitherT(IO {
    if id > 0 then Right(User(id, "Alice", "alice@example.com"))
    else Left(s"Invalid user id: $id")
  })

val program: App[(User, Int)] = for
  user <- fetchUser(1L)
  age  <- validateAge(25)
yield (user, age)

val result: IO[Either[AppError, (User, Int)]] = program.value
```

---

## 3. OptionT

`OptionT[F, A]` = `F[Option[A]]` แต่ compose ได้

### Basic OptionT

```scala
import cats.data.OptionT
import cats.effect.IO
import cats.implicits.*

// สร้าง OptionT
val some: OptionT[IO, Int]  = OptionT.some[IO](42)
val none: OptionT[IO, Int]  = OptionT.none[IO, Int]
val fromOpt: OptionT[IO, Int] = OptionT.fromOption[IO](Some(10))
val liftIO: OptionT[IO, Int]  = OptionT.liftF(IO.pure(5))

// From IO[Option[A]]
val fromIO: OptionT[IO, String] = OptionT(IO.pure(Some("hello")))
```

### OptionT Composition

```scala
import cats.data.OptionT
import cats.effect.IO

case class User(id: Long, name: String, managerId: Option[Long])
case class Department(id: Long, name: String, managerId: Long)

// Simulate database
val users = Map(
  1L -> User(1L, "Alice", Some(2L)),
  2L -> User(2L, "Bob", None)
)
val departments = Map(
  1L -> Department(1L, "Engineering", 1L)
)

def findUser(id: Long): OptionT[IO, User] =
  OptionT(IO.pure(users.get(id)))

def findManager(user: User): OptionT[IO, User] =
  user.managerId.fold(OptionT.none[IO, User])(findUser)

def findDepartment(userId: Long): OptionT[IO, Department] =
  OptionT(IO.pure(departments.values.find(_.managerId == userId)))

// Compose ด้วย for-comprehension
def getManagerDepartment(userId: Long): OptionT[IO, (User, Department)] =
  for
    user       <- findUser(userId)
    manager    <- findManager(user)
    department <- findDepartment(manager.id)
  yield (manager, department)

// Run
val result = getManagerDepartment(1L).value
// IO(Some((User(2, "Bob", None), Department(1, "Engineering", 1))))
```

### OptionT Utilities

```scala
import cats.data.OptionT
import cats.implicits.*

// getOrElse
val value = OptionT.some[IO](42)
val withDefault: IO[Int] = value.getOrElse(0)

// getOrElseF - ถ้า None ให้รัน effect
val withDefaultF: IO[Int] = value.getOrElseF(IO.pure(99))

// orElse - ถ้า None ลอง OptionT อื่น
val fallback: OptionT[IO, Int] = OptionT.none[IO, Int].orElse(OptionT.some[IO](10))

// filter
val filtered: OptionT[IO, Int] = OptionT.some[IO](42).filter(_ > 10)
// Some(42) ถ้า > 10, None ถ้าไม่ใช่

// map
val mapped: OptionT[IO, String] = OptionT.some[IO](42).map(_.toString)

// semiflatMap
val semiMapped: OptionT[IO, Int] = OptionT.some[IO](42).semiflatMap { n =>
  IO.pure(n * 2)
}

// foldF
val folded: IO[String] = OptionT.some[IO](42).foldF(
  ifNone  = IO.pure("was None"),
  ifSome  = n => IO.pure(s"was $n")
)
```

---

## 4. EitherT

`EitherT[F, E, A]` = `F[Either[E, A]]`

### Basic EitherT

```scala
import cats.data.EitherT
import cats.effect.IO

sealed trait AppError
case class NotFound(id: Long)      extends AppError
case class ValidationError(msg: String) extends AppError
case class DatabaseError(msg: String)   extends AppError

type Result[A] = EitherT[IO, AppError, A]

// สร้าง EitherT
def success[A](a: A): Result[A]  = EitherT.rightT[IO, AppError](a)
def failure[A](e: AppError): Result[A] = EitherT.leftT[IO, A](e)
def liftIO[A](io: IO[A]): Result[A] = EitherT.liftF[IO, AppError, A](io)
def fromEither[A](e: Either[AppError, A]): Result[A] = EitherT.fromEither[IO](e)
```

### EitherT Composition

```scala
import cats.data.EitherT
import cats.effect.IO
import cats.implicits.*

def validateEmail(email: String): Result[String] =
  if email.contains("@")
  then success(email)
  else failure(ValidationError(s"Invalid email: $email"))

def validateAge(age: Int): Result[Int] =
  if age >= 18
  then success(age)
  else failure(ValidationError(s"Must be 18+, got $age"))

def saveUser(email: String, age: Int): Result[User] =
  EitherT(IO {
    Right(User(1L, "New User", email))
  })

// Compose multiple validations
def registerUser(email: String, age: Int): Result[User] = for
  validEmail <- validateEmail(email)
  validAge   <- validateAge(age)
  user       <- saveUser(validEmail, validAge)
yield user

// Run
val success = registerUser("alice@example.com", 25).value
// IO(Right(User(...)))

val fail = registerUser("invalid-email", 15).value
// IO(Left(ValidationError("Invalid email: invalid-email")))
```

### Error Recovery

```scala
// recover - แก้ error แต่ต้องให้ค่า default
val result = failure[Int](ValidationError("oops"))
  .recover {
    case ValidationError(_) => 0
  }

// recoverWith - แก้ error โดยใช้ EitherT อีกตัว
val result2 = failure[User](NotFound(1L))
  .recoverWith {
    case NotFound(id) => success(User(id, "Default", "default@example.com"))
  }

// leftMap - transform error type
val mapped = failure[Int](ValidationError("bad"))
  .leftMap(e => s"Error: ${e.toString}")

// bimap - transform both sides
val transformed = failure[Int](ValidationError("bad"))
  .bimap(
    error => s"Error: $error",
    value => value * 2
  )

// handleErrorWith
val handled = failure[Int](DatabaseError("connection failed"))
  .handleErrorWith {
    case DatabaseError(_) => liftIO(IO.pure(99))
    case other            => failure(other)
  }
```

---

## 5. WriterT

`WriterT[F, W, A]` = `F[(W, A)]` - เก็บ log ควบคู่กับค่า

### Basic WriterT

```scala
import cats.data.WriterT
import cats.effect.IO
import cats.implicits.*

type Logged[A] = WriterT[IO, List[String], A]

// สร้าง WriterT
def logged[A](a: A, log: String): Logged[A] =
  WriterT.put[IO, List[String], A](a)(List(log))

def pureLog[A](a: A): Logged[A] =
  WriterT.value[IO, List[String], A](a)

def tell(msg: String): Logged[Unit] =
  WriterT.tell[IO, List[String]](List(msg))
```

### WriterT Composition

```scala
def processOrder(orderId: Long): Logged[ProcessResult] = for
  _      <- tell(s"Processing order $orderId")
  order  <- logged(Order(orderId, "PENDING"), s"Fetched order $orderId")
  _      <- tell("Validating order")
  valid  <- logged(true, "Order validation passed")
  _      <- tell("Calculating total")
  total  <- logged(BigDecimal(99.99), s"Total: 99.99")
  _      <- tell(s"Order $orderId processed successfully")
yield ProcessResult(orderId, total)

// Run
val (logs, result) = processOrder(42L).run.unsafeRunSync()
logs.foreach(println)
// Processing order 42
// Fetched order 42
// Order validation passed
// Total: 99.99
// Order 42 processed successfully
```

### WriterT Utilities

```scala
// Listen - รับทั้ง value และ log
val (log, value) = logged(42, "answer").listen.run.unsafeRunSync()

// Pass - modify log
val modified = logged(42, "original").pass.map { case (w, a) =>
  (w.map(_.toUpperCase), a)
}

// mapWritten - transform log
val upperLog = logged(42, "hello").mapWritten(_.map(_.toUpperCase))
```

---

## 6. StateT

`StateT[F, S, A]` = `S => F[(S, A)]` - computation ที่มี state

### Basic StateT

```scala
import cats.data.StateT
import cats.effect.IO
import cats.implicits.*

// State type
case class Counter(count: Int, total: Long)

type StateIO[A] = StateT[IO, Counter, A]

// Get state
def getState: StateIO[Counter] = StateT.get[IO, Counter]

// Modify state
def increment: StateIO[Unit] =
  StateT.modify[IO, Counter](s => s.copy(count = s.count + 1))

def addToTotal(n: Long): StateIO[Unit] =
  StateT.modify[IO, Counter](s => s.copy(total = s.total + n))

// Pure value
def pure[A](a: A): StateIO[A] = StateT.pure[IO, Counter, A](a)
```

### StateT Composition

```scala
def processItem(item: Item): StateIO[String] = for
  _     <- increment
  _     <- addToTotal(item.price.toLong)
  state <- getState
  _     <- StateT.liftF(IO.println(s"Processed item ${item.id}"))
yield s"Item ${item.id} processed. Total items: ${state.count}"

// Run with initial state
val items = List(Item(1, "A", 10.0), Item(2, "B", 20.0), Item(3, "C", 30.0))
val initialState = Counter(0, 0L)

val program: StateIO[List[String]] =
  items.traverse(processItem)

val (finalState, results) = program.run(initialState).unsafeRunSync()
println(s"Final: ${finalState.count} items, total: ${finalState.total}")
// Final: 3 items, total: 60
```

### StateT Operations

```scala
// inspect - get value from state
def getCount: StateIO[Int] = StateT.inspect[IO, Counter, Int](_.count)

// set - replace state entirely
def resetCounter: StateIO[Unit] = StateT.set[IO, Counter](Counter(0, 0L))

// Combining state operations
val program = for
  _     <- increment
  count <- getCount
  _     <- if count > 10 then resetCounter else StateT.pure[IO, Counter, Unit](())
yield count
```

---

## 7. Cats MTL Type Classes

Cats MTL ให้ type class abstraction สำหรับ monad transformer capabilities

### Setup

```scala
import cats.mtl.*

// Core MTL type classes:
// - Ask[F, E]    - read environment (Reader)
// - Raise[F, E]  - raise errors (EitherT)
// - Handle[F, E] - handle errors
// - Tell[F, L]   - emit log (Writer)
// - Listen[F, L] - read written log
// - Stateful[F, S] - mutable state (State)
// - Local[F, E]  - locally modify environment
```

### Ask (Reader)

```scala
import cats.mtl.Ask
import cats.effect.IO

case class AppConfig(
  dbUrl: String,
  maxConnections: Int,
  debug: Boolean
)

// ฟังก์ชันที่ต้องการ config โดยไม่ระบุ F[_] เฉพาะเจาะจง
def getDbUrl[F[_]: Ask[*[_], AppConfig]]: F[String] =
  Ask[F, AppConfig].asks(_.dbUrl)

def isDebug[F[_]: Ask[*[_], AppConfig]]: F[Boolean] =
  Ask[F, AppConfig].asks(_.debug)

def runQuery[F[_]: Monad: Ask[*[_], AppConfig]](query: String): F[String] = for
  url   <- getDbUrl[F]
  debug <- isDebug[F]
  _     <- if debug then Applicative[F].unit else Applicative[F].unit
yield s"Executed '$query' on $url"
```

### Raise และ Handle

```scala
import cats.mtl.{Raise, Handle}
import cats.implicits.*

sealed trait AppError
case class NotFound(resource: String) extends AppError
case class Unauthorized(msg: String)  extends AppError
case class ServerError(msg: String)   extends AppError

// Raise errors
def findUser[F[_]: Monad: Raise[*[_], AppError]](id: Long): F[User] =
  if id > 0 then Monad[F].pure(User(id, "Alice", "alice@example.com"))
  else Raise[F, AppError].raise(NotFound(s"User $id"))

def checkAuth[F[_]: Monad: Raise[*[_], AppError]](token: String): F[Unit] =
  if token == "valid-token" then Monad[F].unit
  else Raise[F, AppError].raise(Unauthorized("Invalid token"))

// Handle errors
def withFallback[F[_]: Monad: Handle[*[_], AppError]](userId: Long): F[User] =
  Handle[F, AppError].handleWith(findUser[F](userId)) {
    case NotFound(_) =>
      Monad[F].pure(User(0L, "Guest", "guest@example.com"))
    case e =>
      Handle[F, AppError].raise(e)
  }
```

### Tell และ Listen

```scala
import cats.mtl.{Tell, Listen}

// Tell - เขียน log
def logEvent[F[_]: Tell[*[_], List[String]]](event: String): F[Unit] =
  Tell[F, List[String]].tell(List(event))

def processWithLogging[F[_]: Monad: Tell[*[_], List[String]]](
  data: List[Int]
): F[Int] = for
  _      <- logEvent[F]("Starting processing")
  result <- data.traverse { n =>
    logEvent[F](s"Processing $n").as(n * 2)
  }.map(_.sum)
  _      <- logEvent[F](s"Finished with result: $result")
yield result

// Listen - อ่าน log ที่เขียน
def listenToLogs[F[_]: Listen[*[_], List[String]]: Monad](
  program: F[Int]
): F[(Int, List[String])] =
  Listen[F, List[String]].listen(program)
```

### Stateful

```scala
import cats.mtl.Stateful

case class AppState(
  requestCount: Int,
  errorCount: Int,
  lastError: Option[String]
)

def trackRequest[F[_]: Stateful[*[_], AppState]: Monad]: F[Unit] =
  Stateful[F, AppState].modify(s => s.copy(requestCount = s.requestCount + 1))

def trackError[F[_]: Stateful[*[_], AppState]: Monad](error: String): F[Unit] =
  Stateful[F, AppState].modify(s => s.copy(
    errorCount = s.errorCount + 1,
    lastError = Some(error)
  ))

def getStats[F[_]: Stateful[*[_], AppState]]: F[AppState] =
  Stateful[F, AppState].get
```

### Local (Reader variant)

```scala
import cats.mtl.Local

// Local allows locally modifying the environment
def withDebug[F[_]: Local[*[_], AppConfig]: Monad](program: F[String]): F[String] =
  Local[F, AppConfig].local(program)(config => config.copy(debug = true))

// Useful for tracing, correlation IDs, etc.
def withCorrelationId[F[_]: Local[*[_], RequestContext]: Monad](
  correlationId: String
)(program: F[String]): F[String] =
  Local[F, RequestContext].local(program)(ctx =>
    ctx.copy(correlationId = Some(correlationId))
  )
```

---

## 8. MTL vs Tagless Final

### Tagless Final Approach

```scala
// Tagless Final: explicit type class constraints
trait UserRepo[F[_]]:
  def findById(id: Long): F[Option[User]]
  def save(user: User): F[User]

trait EmailService[F[_]]:
  def send(to: String, subject: String, body: String): F[Unit]

class UserService[F[_]: Monad](
  userRepo: UserRepo[F],
  emailService: EmailService[F]
):
  def register(email: String): F[User] = for
    user  <- userRepo.save(User(0, "New", email))
    _     <- emailService.send(email, "Welcome!", "Welcome to our service")
  yield user
```

### MTL Approach

```scala
import cats.mtl.*

// MTL: capabilities เป็น type class constraints
def register[F[_]: Monad](
  email: String
)(using
  raise:     Raise[F, AppError],
  userState: Stateful[F, UserState],
  logger:    Tell[F, List[String]]
): F[User] = for
  _    <- Tell[F, List[String]].tell(List(s"Registering $email"))
  _    <- if email.isEmpty then Raise[F, AppError].raise(ValidationError("Email required"))
          else Monad[F].unit
  user =  User(0, "New", email)
  _    <- Stateful[F, UserState].modify(s => s.copy(userCount = s.userCount + 1))
yield user
```

### เปรียบเทียบ

```
Tagless Final:
+ Explicit interfaces (UserRepo, EmailService)
+ Easy to mock/test
+ Type-safe
- More boilerplate

MTL:
+ Composable effects via type classes
+ Less boilerplate for simple effects (state, error, logging)
+ Uniform interface
- More complex for beginners
- Implicit resolution can be tricky
```

---

## 9. Application สมบูรณ์ด้วย MTL

### Domain Models

```scala
// domain/models.scala
package domain

case class User(
  id:    Long,
  email: String,
  name:  String,
  role:  String
)

case class Product(
  id:    Long,
  name:  String,
  price: BigDecimal,
  stock: Int
)

case class Order(
  id:        Long,
  userId:    Long,
  productId: Long,
  quantity:  Int,
  total:     BigDecimal
)

sealed trait DomainError
case class NotFound(resource: String, id: Long) extends DomainError
case class InsufficientStock(productId: Long, requested: Int, available: Int) extends DomainError
case class InvalidInput(field: String, message: String) extends DomainError
case class DatabaseError(message: String) extends DomainError
```

### Application State

```scala
// app/state.scala
case class AppState(
  orderCount:    Int,
  totalRevenue:  BigDecimal,
  errorCount:    Int,
  processingIds: Set[Long]
)

object AppState:
  val empty = AppState(0, BigDecimal.ZERO, 0, Set.empty)
```

### Service Layer

```scala
// app/services.scala
import cats.*
import cats.mtl.*
import cats.implicits.*

// Type aliases
type Log = List[String]

// Order service ที่ใช้ MTL
class OrderService[F[_]: Monad](
  userRepo:    UserRepo[F],
  productRepo: ProductRepo[F]
)(using
  raise:   Raise[F, DomainError],
  state:   Stateful[F, AppState],
  logger:  Tell[F, Log]
):

  def placeOrder(userId: Long, productId: Long, quantity: Int): F[Order] = for
    // Log
    _ <- Tell[F, Log].tell(List(s"[ORDER] User $userId ordering $quantity of product $productId"))
    
    // Validate
    _ <- if quantity <= 0 then
           Raise[F, DomainError].raise(InvalidInput("quantity", "Must be > 0"))
         else Monad[F].unit
    
    // Find user
    user <- userRepo.findById(userId)
              .flatMap {
                case Some(u) => Monad[F].pure(u)
                case None    => Raise[F, DomainError].raise(NotFound("User", userId))
              }
    
    // Find product
    product <- productRepo.findById(productId)
                 .flatMap {
                   case Some(p) => Monad[F].pure(p)
                   case None    => Raise[F, DomainError].raise(NotFound("Product", productId))
                 }
    
    // Check stock
    _ <- if product.stock < quantity then
           Raise[F, DomainError].raise(
             InsufficientStock(productId, quantity, product.stock)
           )
         else Monad[F].unit
    
    // Calculate total
    total = product.price * quantity
    
    // Create order
    order = Order(0L, userId, productId, quantity, total)
    
    // Update state
    _ <- Stateful[F, AppState].modify(s => s.copy(
           orderCount   = s.orderCount + 1,
           totalRevenue = s.totalRevenue + total
         ))
    
    // Log success
    _ <- Tell[F, Log].tell(List(s"[ORDER] Success: order created, total: $total"))
    
  yield order

  def cancelOrder(orderId: Long): F[Unit] = for
    _ <- Tell[F, Log].tell(List(s"[ORDER] Cancelling order $orderId"))
    _ <- Stateful[F, AppState].modify(s => s.copy(
           processingIds = s.processingIds - orderId
         ))
  yield ()
```

### Concrete Implementation

```scala
import cats.data.*
import cats.effect.IO

// Concrete effect type
type AppEffect[A] = EitherT[
  WriterT[StateT[IO, AppState, *], Log, *],
  DomainError,
  A
]

// ขั้นตอนการ implement ซับซ้อน - ในทางปฏิบัติใช้ IOLocal หรือ simpler stacks

// Simpler version: ใช้ IO + Ref + error handling
import cats.effect.Ref

class OrderServiceImpl(
  userRepo:    UserRepoImpl,
  productRepo: ProductRepoImpl,
  stateRef:    Ref[IO, AppState],
  logRef:      Ref[IO, List[String]]
):

  def placeOrder(userId: Long, productId: Long, quantity: Int): IO[Either[DomainError, Order]] =
    val program = for
      _ <- logRef.update(_ :+ s"Processing order for user $userId")
      
      userOpt <- userRepo.findById(userId)
      user    <- userOpt match
        case Some(u) => IO.pure(u)
        case None    => IO.raiseError(new Exception(s"User $userId not found"))
      
      productOpt <- productRepo.findById(productId)
      product    <- productOpt match
        case Some(p) => IO.pure(p)
        case None    => IO.raiseError(new Exception(s"Product $productId not found"))
      
      _ <- if product.stock < quantity
           then IO.raiseError(new Exception("Insufficient stock"))
           else IO.unit
      
      total  = product.price * quantity
      order  = Order(0L, userId, productId, quantity, total)
      
      _ <- stateRef.update(s => s.copy(
             orderCount   = s.orderCount + 1,
             totalRevenue = s.totalRevenue + total
           ))
      
      _ <- logRef.update(_ :+ s"Order created successfully, total: $total")
      
    yield order
    
    program.attempt.map(_.left.map(e => DatabaseError(e.getMessage)))
```

### Running the Application

```scala
import cats.effect.*

object MainApp extends IOApp.Simple:

  def run: IO[Unit] = for
    stateRef <- Ref.of[IO, AppState](AppState.empty)
    logRef   <- Ref.of[IO, List[String]](List.empty)
    
    userRepo    = new UserRepoImpl()
    productRepo = new ProductRepoImpl()
    service     = new OrderServiceImpl(userRepo, productRepo, stateRef, logRef)
    
    // Place some orders
    result1 <- service.placeOrder(1L, 1L, 2)
    result2 <- service.placeOrder(1L, 2L, 1)
    result3 <- service.placeOrder(99L, 1L, 1)  // Should fail - user not found
    
    // Print results
    _ <- IO.println(s"Order 1: $result1")
    _ <- IO.println(s"Order 2: $result2")
    _ <- IO.println(s"Order 3: $result3")
    
    // Print final state
    finalState <- stateRef.get
    _ <- IO.println(s"Final state: $finalState")
    
    // Print logs
    logs <- logRef.get
    _ <- logs.traverse(log => IO.println(s"LOG: $log"))
    
  yield ()
```

---

## 10. สรุป

### Monad Transformers ที่เรียนรู้

| Transformer | Type | ใช้สำหรับ |
|-------------|------|-----------|
| OptionT | F[Option[A]] | Optional values ใน context |
| EitherT | F[Either[E, A]] | Error handling |
| WriterT | F[(W, A)] | Accumulating logs/output |
| StateT | S => F[(S, A)] | Stateful computation |

### Cats MTL Type Classes

| Type Class | ใช้สำหรับ |
|------------|-----------|
| Ask[F, E] | Read environment |
| Local[F, E] | Locally modify environment |
| Raise[F, E] | Raise errors |
| Handle[F, E] | Handle errors |
| Tell[F, L] | Emit log/output |
| Listen[F, L] | Read emitted output |
| Stateful[F, S] | Mutable state |

### Best Practices

```scala
// 1. ระวัง stack ordering - outer transformer มี performance overhead
// ✅ IO (async) เป็น base, EitherT อยู่ข้างนอก
type App[A] = EitherT[IO, Error, A]

// 2. ใช้ IORef แทน StateT ถ้าทำได้ - performance ดีกว่า
// ❌ StateT สำหรับ concurrent state
type Stateful[A] = StateT[IO, State, A]  // ไม่ thread-safe
// ✅ Ref สำหรับ concurrent state
val stateRef: Ref[IO, State]  // thread-safe

// 3. MTL เหมาะสำหรับ dependency injection via type classes
def myService[F[_]: Monad: Raise[*[_], Error]: Tell[*[_], Log]](
  ...
): F[Result] = ???
```

### ข้อสรุปสำคัญ

1. **Monad Transformers** ช่วย compose effects
2. **Cats MTL** ทำให้โค้ด abstract และ testable ขึ้น
3. **ความซับซ้อน** เพิ่มขึ้นตาม stack depth
4. **Cats Effect IO** มี built-in tools หลายอย่าง (IORef, IOLocal) ที่ดีกว่า transformers

---

*[← Part 67: Shapeless](part-67-shapeless.md) | [Part 69: Free Monad →](part-69-free-monad.md)*
