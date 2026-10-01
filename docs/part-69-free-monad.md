# Part 69: Free Monad - DSL และ Interpreters

## สารบัญ

1. [แนะนำ Free Monad](#1-แนะนำ-free-monad)
2. [Free Monad แนวคิด](#2-free-monad-แนวคิด)
3. [DSL ด้วย Free](#3-dsl-ด้วย-free)
4. [Interpreters: Pure, IO, Test](#4-interpreters-pure-io-test)
5. [Free Monad vs Tagless Final](#5-free-monad-vs-tagless-final)
6. [Coproduct ของ Algebras](#6-coproduct-ของ-algebras)
7. [Complete CRUD DSL](#7-complete-crud-dsl)
8. [Advanced Patterns](#8-advanced-patterns)
9. [สรุป](#9-สรุป)

---

## 1. แนะนำ Free Monad

Free Monad เป็น technique สำหรับสร้าง DSL (Domain Specific Language) ที่:

- **แยก description จาก execution** - โปรแกรมเป็นแค่ data structure
- **เปลี่ยน interpreter ได้** - ใช้ test interpreter แทน production ได้ง่าย
- **Composable** - รวม DSLs หลายตัวเข้าด้วยกัน

### ทำไมต้องใช้ Free Monad?

```
Traditional code:
  Program = Instructions that execute immediately
  Hard to test, hard to compose

Free Monad:
  Program = Data structure describing what to do
  Interpreter = The thing that actually executes
  
Benefits:
  1. Pure programs (no side effects in program itself)
  2. Multiple interpreters (production, test, debug)
  3. Program analysis (optimization, logging, etc.)
```

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.typelevel" %% "cats-core"   % "2.10.0",
  "org.typelevel" %% "cats-free"   % "2.10.0",
  "org.typelevel" %% "cats-effect" % "3.5.1"
)
```

---

## 2. Free Monad แนวคิด

### Free Monad คืออะไร?

```scala
// Free Monad คือ structure ที่ทำให้ functor กลายเป็น monad
// Free[F, A] ที่ F เป็น Functor

sealed trait Free[F[_], A]
case class Pure[F[_], A](a: A)                       extends Free[F, A]
case class Suspend[F[_], A](fa: F[Free[F, A]])       extends Free[F, A]

// การ flatMap สร้าง tree ของ operations
// Interpreter traverse tree นี้และ execute แต่ละ node
```

### Free[F, A] ทำงานอย่างไร

```
Program represented as a tree:

FlatMap(
  Suspend(GetUser(1)),       <- node 1: get user
  user => FlatMap(
    Suspend(GetPosts(user)),  <- node 2: get posts
    posts => Pure(
      (user, posts)           <- leaf: final result
    )
  )
)

Interpreter walks this tree:
1. See GetUser(1) -> execute -> get User(1, "Alice", ...)
2. See GetPosts(alice) -> execute -> get List(Post(...), ...)
3. See Pure((alice, posts)) -> return (alice, posts)
```

---

## 3. DSL ด้วย Free

### Step 1: Define Algebra (ADT)

```scala
// algebra.scala - อธิบาย operations ของ DSL
package free.example

// User operations
sealed trait UserOp[A]

case class GetUser(id: Long)                                    extends UserOp[Option[User]]
case class GetAllUsers()                                        extends UserOp[List[User]]
case class CreateUser(name: String, email: String)              extends UserOp[User]
case class UpdateUser(id: Long, name: String, email: String)    extends UserOp[Option[User]]
case class DeleteUser(id: Long)                                 extends UserOp[Boolean]
case class FindUserByEmail(email: String)                       extends UserOp[Option[User]]
```

### Step 2: Lift into Free

```scala
import cats.free.Free
import cats.free.Free.liftF

// Type alias
type UserProgram[A] = Free[UserOp, A]

// Smart constructors (lifts operations into Free context)
def getUser(id: Long): UserProgram[Option[User]] =
  liftF[UserOp, Option[User]](GetUser(id))

def getAllUsers(): UserProgram[List[User]] =
  liftF[UserOp, List[User]](GetAllUsers())

def createUser(name: String, email: String): UserProgram[User] =
  liftF[UserOp, User](CreateUser(name, email))

def updateUser(id: Long, name: String, email: String): UserProgram[Option[User]] =
  liftF[UserOp, Option[User]](UpdateUser(id, name, email))

def deleteUser(id: Long): UserProgram[Boolean] =
  liftF[UserOp, Boolean](DeleteUser(id))

def findUserByEmail(email: String): UserProgram[Option[User]] =
  liftF[UserOp, Option[User]](FindUserByEmail(email))
```

### Step 3: Write Programs

```scala
import cats.implicits.*

// Programs เป็นแค่ data - ไม่ execute จริง
val createAndFetch: UserProgram[(User, Option[User])] = for
  newUser  <- createUser("Alice", "alice@example.com")
  fetched  <- getUser(newUser.id)
yield (newUser, fetched)

val findOrCreate: UserProgram[User] = for
  existing <- findUserByEmail("bob@example.com")
  user     <- existing match
    case Some(u) => Free.pure[UserOp, User](u)
    case None    => createUser("Bob", "bob@example.com")
yield user

val batchOperation: UserProgram[List[User]] = for
  users <- getAllUsers()
  _     <- users.filter(_.name.isEmpty).traverse { u =>
    deleteUser(u.id)
  }
  remaining <- getAllUsers()
yield remaining
```

---

## 4. Interpreters: Pure, IO, Test

### Interpreter คืออะไร?

```scala
import cats.~>

// Natural transformation F ~> G
// แปลงทุก F[A] เป็น G[A]
// ใช้สำหรับ interpret Free[F, A] -> G[A]
```

### Pure Interpreter (Identity)

```scala
import cats.Id

// Pure interpreter ที่ทำงานกับ in-memory state
object PureInterpreter extends (UserOp ~> Id):

  // In-memory database
  private var db = Map[Long, User](
    1L -> User(1L, "Alice", "alice@example.com"),
    2L -> User(2L, "Bob", "bob@example.com")
  )
  private var nextId = 3L

  def apply[A](op: UserOp[A]): Id[A] = op match
    case GetUser(id) =>
      db.get(id)

    case GetAllUsers() =>
      db.values.toList.sortBy(_.id)

    case CreateUser(name, email) =>
      val user = User(nextId, name, email)
      db = db + (nextId -> user)
      nextId += 1
      user

    case UpdateUser(id, name, email) =>
      db.get(id).map { _ =>
        val updated = User(id, name, email)
        db = db + (id -> updated)
        updated
      }

    case DeleteUser(id) =>
      if db.contains(id) then
        db = db - id
        true
      else false

    case FindUserByEmail(email) =>
      db.values.find(_.email == email)
```

### IO Interpreter (Production)

```scala
import cats.effect.IO
import scala.concurrent.Future

// Real database interpreter
class IOInterpreter(db: Database) extends (UserOp ~> IO):

  def apply[A](op: UserOp[A]): IO[A] = op match
    case GetUser(id) =>
      db.findById(id)

    case GetAllUsers() =>
      db.findAll()

    case CreateUser(name, email) =>
      for
        existing <- db.findByEmail(email)
        _        <- if existing.isDefined then
                      IO.raiseError(new Exception(s"Email $email already exists"))
                    else IO.unit
        user     <- db.insert(User(0L, name, email))
      yield user

    case UpdateUser(id, name, email) =>
      db.update(id, name, email)

    case DeleteUser(id) =>
      db.delete(id)

    case FindUserByEmail(email) =>
      db.findByEmail(email)
```

### Test Interpreter

```scala
import cats.data.State

// State สำหรับ test (เก็บ call history)
case class TestState(
  users:   Map[Long, User],
  nextId:  Long,
  history: List[String]  // เก็บ operations ที่ถูกเรียก
)

object TestState:
  val empty = TestState(Map.empty, 1L, List.empty)

// Test interpreter ที่บันทึก operations
type TestProgram[A] = State[TestState, A]

object TestInterpreter extends (UserOp ~> TestProgram):

  def apply[A](op: UserOp[A]): TestProgram[A] = op match
    case GetUser(id) =>
      State { s =>
        val result = s.users.get(id)
        (s.copy(history = s.history :+ s"GetUser($id)"), result)
      }

    case GetAllUsers() =>
      State { s =>
        (s.copy(history = s.history :+ "GetAllUsers()"),
         s.users.values.toList.sortBy(_.id))
      }

    case CreateUser(name, email) =>
      State { s =>
        val user       = User(s.nextId, name, email)
        val newState   = s.copy(
          users   = s.users + (s.nextId -> user),
          nextId  = s.nextId + 1,
          history = s.history :+ s"CreateUser($name, $email)"
        )
        (newState, user)
      }

    case UpdateUser(id, name, email) =>
      State { s =>
        val result = s.users.get(id).map(_ => User(id, name, email))
        val newUsers = result.fold(s.users)(u => s.users + (id -> u))
        (s.copy(users = newUsers, history = s.history :+ s"UpdateUser($id)"), result)
      }

    case DeleteUser(id) =>
      State { s =>
        val exists = s.users.contains(id)
        (s.copy(
          users   = s.users - id,
          history = s.history :+ s"DeleteUser($id)"
        ), exists)
      }

    case FindUserByEmail(email) =>
      State { s =>
        (s.copy(history = s.history :+ s"FindByEmail($email)"),
         s.users.values.find(_.email == email))
      }
```

### Running Programs with Interpreters

```scala
// รัน program ด้วย Pure interpreter
val purResult: Id[(User, Option[User])] =
  createAndFetch.foldMap(PureInterpreter)

// รัน program ด้วย IO interpreter
val ioResult: IO[(User, Option[User])] =
  createAndFetch.foldMap(new IOInterpreter(database))

// รัน program ด้วย Test interpreter
val (finalState, testResult) =
  createAndFetch.foldMap(TestInterpreter).run(TestState.empty).value

println(s"History: ${finalState.history}")
// History: List("CreateUser(Alice, alice@example.com)", "GetUser(1)")
```

---

## 5. Free Monad vs Tagless Final

### Tagless Final

```scala
// Tagless Final: algebra เป็น type class
trait UserAlgebra[F[_]]:
  def getUser(id: Long): F[Option[User]]
  def createUser(name: String, email: String): F[User]
  def deleteUser(id: Long): F[Boolean]

// Implementation
class UserAlgebraIO extends UserAlgebra[IO]:
  def getUser(id: Long): IO[Option[User]] = ???
  def createUser(name: String, email: String): IO[User] = ???
  def deleteUser(id: Long): IO[Boolean] = ???

// Program
def program[F[_]: Monad](using algebra: UserAlgebra[F]): F[(User, Boolean)] = for
  user    <- algebra.createUser("Alice", "alice@example.com")
  deleted <- algebra.deleteUser(999L)
yield (user, deleted)
```

### เปรียบเทียบ

```
Free Monad:
✅ Program เป็น data (can inspect, serialize)
✅ ง่ายต่อการ switch interpreters
✅ Program analysis (logging, optimization)
❌ Performance overhead (tree traversal)
❌ Harder to compose algebras (need Coproduct)
❌ Verbose boilerplate

Tagless Final:
✅ Better performance (no tree overhead)
✅ Simpler to write
✅ Easy to extend
✅ Type system catches errors better
❌ Program ไม่ใช่ data (ไม่สามารถ inspect)
❌ ยากต่อ program analysis
```

### เมื่อไหร่ใช้อะไร?

```
ใช้ Free Monad เมื่อ:
- ต้องการ inspect หรือ optimize programs
- ต้องการ serialize programs
- ต้องการ multiple very different interpreters
- สร้าง scripting engine หรือ query planner

ใช้ Tagless Final เมื่อ:
- ต้องการ performance ดี
- ต้องการ simple codebase
- Effects เป็นหลัก
- ทีมคุ้นเคยกับ type classes
```

---

## 6. Coproduct ของ Algebras

### ปัญหา: รวม DSLs หลายตัว

```scala
// ต้องการรวม UserOp และ EmailOp ใน program เดียวกัน
sealed trait EmailOp[A]
case class SendEmail(to: String, subject: String, body: String) extends EmailOp[Unit]
case class GetInbox(userId: Long)                                extends EmailOp[List[Email]]
```

### Cats Inject สำหรับ Coproduct

```scala
import cats.free.Free
import cats.free.Free.liftF
import cats.InjectK

// Combined algebra เป็น Coproduct
type AppOp[A] = cats.data.EitherK[UserOp, EmailOp, A]
type AppProgram[A] = Free[AppOp, A]

// Smart constructors ที่ใช้ InjectK
class UserOps[F[_]](using inject: InjectK[UserOp, F]):
  def getUser(id: Long): Free[F, Option[User]] =
    Free.liftInject[F](GetUser(id))
  
  def createUser(name: String, email: String): Free[F, User] =
    Free.liftInject[F](CreateUser(name, email))

class EmailOps[F[_]](using inject: InjectK[EmailOp, F]):
  def sendEmail(to: String, subject: String, body: String): Free[F, Unit] =
    Free.liftInject[F](SendEmail(to, subject, body))
  
  def getInbox(userId: Long): Free[F, List[Email]] =
    Free.liftInject[F](GetInbox(userId))

// Program ที่ใช้ทั้งสอง DSLs
def registerAndNotify(
  name: String, 
  emailAddr: String
)(using
  users:  UserOps[AppOp],
  emails: EmailOps[AppOp]
): AppProgram[(User, Unit)] = for
  user <- users.createUser(name, emailAddr)
  _    <- emails.sendEmail(
    emailAddr,
    "Welcome!",
    s"Hello ${user.name}, welcome to our service!"
  )
yield (user, ())
```

### Combined Interpreter

```scala
import cats.~>

// Interpreter สำหรับ UserOp
val userInterp: UserOp ~> IO = new (UserOp ~> IO):
  def apply[A](op: UserOp[A]): IO[A] = op match
    case GetUser(id)            => database.findUser(id)
    case CreateUser(name, email) => database.insertUser(name, email)
    // ...

// Interpreter สำหรับ EmailOp
val emailInterp: EmailOp ~> IO = new (EmailOp ~> IO):
  def apply[A](op: EmailOp[A]): IO[A] = op match
    case SendEmail(to, subject, body) => emailService.send(to, subject, body)
    case GetInbox(userId)             => emailService.getInbox(userId)

// Combined interpreter
import cats.data.EitherK

val combinedInterp: AppOp ~> IO =
  userInterp or emailInterp
  // or เป็น method บน ~> ที่รวม two interpreters

// รัน combined program
val result: IO[(User, Unit)] =
  registerAndNotify("Alice", "alice@example.com").foldMap(combinedInterp)
```

---

## 7. Complete CRUD DSL

### Full CRUD DSL Implementation

```scala
// ============================================
// FULL CRUD DSL EXAMPLE
// ============================================
package free.crud

import cats.free.Free
import cats.free.Free.liftF
import cats.{~>, Id}
import cats.data.State
import cats.implicits.*

// ==================
// Domain Models
// ==================
case class User(
  id:        Long,
  name:      String,
  email:     String,
  createdAt: Long = System.currentTimeMillis()
)

case class Post(
  id:       Long,
  userId:   Long,
  title:    String,
  content:  String,
  tags:     List[String] = Nil
)

// ==================
// Algebra (ADT)
// ==================
sealed trait CrudOp[A]

// User operations
case class InsertUser(name: String, email: String)           extends CrudOp[User]
case class SelectUser(id: Long)                              extends CrudOp[Option[User]]
case class SelectAllUsers(limit: Int, offset: Int)           extends CrudOp[List[User]]
case class SelectUserByEmail(email: String)                  extends CrudOp[Option[User]]
case class UpdateUserName(id: Long, newName: String)         extends CrudOp[Option[User]]
case class DeleteUser(id: Long)                              extends CrudOp[Boolean]

// Post operations
case class InsertPost(userId: Long, title: String, content: String, tags: List[String] = Nil)
    extends CrudOp[Post]
case class SelectPost(id: Long)                              extends CrudOp[Option[Post]]
case class SelectPostsByUser(userId: Long)                   extends CrudOp[List[Post]]
case class UpdatePost(id: Long, title: String, content: String) extends CrudOp[Option[Post]]
case class DeletePost(id: Long)                              extends CrudOp[Boolean]
case class SearchPosts(keyword: String)                      extends CrudOp[List[Post]]

// ==================
// Smart Constructors
// ==================
type CrudProgram[A] = Free[CrudOp, A]

// User DSL
def insertUser(name: String, email: String): CrudProgram[User] =
  liftF(InsertUser(name, email))

def selectUser(id: Long): CrudProgram[Option[User]] =
  liftF(SelectUser(id))

def selectAllUsers(limit: Int = 20, offset: Int = 0): CrudProgram[List[User]] =
  liftF(SelectAllUsers(limit, offset))

def selectUserByEmail(email: String): CrudProgram[Option[User]] =
  liftF(SelectUserByEmail(email))

def updateUserName(id: Long, newName: String): CrudProgram[Option[User]] =
  liftF(UpdateUserName(id, newName))

def deleteUser(id: Long): CrudProgram[Boolean] =
  liftF(DeleteUser(id))

// Post DSL
def insertPost(userId: Long, title: String, content: String, tags: List[String] = Nil): CrudProgram[Post] =
  liftF(InsertPost(userId, title, content, tags))

def selectPost(id: Long): CrudProgram[Option[Post]] =
  liftF(SelectPost(id))

def selectPostsByUser(userId: Long): CrudProgram[List[Post]] =
  liftF(SelectPostsByUser(userId))

def updatePost(id: Long, title: String, content: String): CrudProgram[Option[Post]] =
  liftF(UpdatePost(id, title, content))

def deletePost(id: Long): CrudProgram[Boolean] =
  liftF(DeletePost(id))

def searchPosts(keyword: String): CrudProgram[List[Post]] =
  liftF(SearchPosts(keyword))

// ==================
// Programs
// ==================

// Program: สร้าง user พร้อม post
def createUserWithPost(
  name:    String,
  email:   String,
  title:   String,
  content: String
): CrudProgram[(User, Post)] = for
  user <- insertUser(name, email)
  post <- insertPost(user.id, title, content)
yield (user, post)

// Program: ย้าย posts จาก user หนึ่งไปอีก user
def transferPosts(fromId: Long, toId: Long): CrudProgram[Int] = for
  fromOpt <- selectUser(fromId)
  toOpt   <- selectUser(toId)
  result  <- (fromOpt, toOpt) match
    case (Some(_), Some(_)) =>
      for
        posts <- selectPostsByUser(fromId)
        _     <- posts.traverse { post =>
          insertPost(toId, post.title, post.content, post.tags)
        }
        _ <- posts.traverse(p => deletePost(p.id))
      yield posts.length
    case _ =>
      Free.pure[CrudOp, Int](0)
yield result

// Program: cleanup inactive users
def cleanupUsers(activeIds: Set[Long]): CrudProgram[List[Long]] = for
  users   <- selectAllUsers(100, 0)
  deleted <- users
    .filterNot(u => activeIds.contains(u.id))
    .traverse { user =>
      for
        posts <- selectPostsByUser(user.id)
        _     <- posts.traverse(p => deletePost(p.id))
        ok    <- deleteUser(user.id)
      yield if ok then Some(user.id) else None
    }
yield deleted.flatten

// ==================
// In-Memory Interpreter
// ==================
case class DbState(
  users:   Map[Long, User] = Map.empty,
  posts:   Map[Long, Post] = Map.empty,
  nextId:  Long            = 1L
)

type DbProgram[A] = State[DbState, A]

object InMemoryInterpreter extends (CrudOp ~> DbProgram):

  def apply[A](op: CrudOp[A]): DbProgram[A] = op match

    case InsertUser(name, email) =>
      State { s =>
        val user = User(s.nextId, name, email)
        (s.copy(users = s.users + (s.nextId -> user), nextId = s.nextId + 1), user)
      }

    case SelectUser(id) =>
      State.inspect(_.users.get(id))

    case SelectAllUsers(limit, offset) =>
      State.inspect(_.users.values.toList.sortBy(_.id).drop(offset).take(limit))

    case SelectUserByEmail(email) =>
      State.inspect(_.users.values.find(_.email == email))

    case UpdateUserName(id, newName) =>
      State { s =>
        s.users.get(id) match
          case None => (s, None)
          case Some(u) =>
            val updated = u.copy(name = newName)
            (s.copy(users = s.users + (id -> updated)), Some(updated))
      }

    case DeleteUser(id) =>
      State { s =>
        if s.users.contains(id)
        then (s.copy(users = s.users - id), true)
        else (s, false)
      }

    case InsertPost(userId, title, content, tags) =>
      State { s =>
        val post = Post(s.nextId, userId, title, content, tags)
        (s.copy(posts = s.posts + (s.nextId -> post), nextId = s.nextId + 1), post)
      }

    case SelectPost(id) =>
      State.inspect(_.posts.get(id))

    case SelectPostsByUser(userId) =>
      State.inspect(_.posts.values.filter(_.userId == userId).toList.sortBy(_.id))

    case UpdatePost(id, title, content) =>
      State { s =>
        s.posts.get(id) match
          case None => (s, None)
          case Some(p) =>
            val updated = p.copy(title = title, content = content)
            (s.copy(posts = s.posts + (id -> updated)), Some(updated))
      }

    case DeletePost(id) =>
      State { s =>
        if s.posts.contains(id)
        then (s.copy(posts = s.posts - id), true)
        else (s, false)
      }

    case SearchPosts(keyword) =>
      State.inspect { s =>
        s.posts.values.filter { post =>
          post.title.toLowerCase.contains(keyword.toLowerCase) ||
          post.content.toLowerCase.contains(keyword.toLowerCase) ||
          post.tags.exists(_.toLowerCase.contains(keyword.toLowerCase))
        }.toList.sortBy(_.id)
      }
```

### Running the Complete Example

```scala
// ==================
// Running Programs
// ==================
object CrudExample:
  def main(args: Array[String]): Unit =
    
    // Program 1: Create user with post
    val createProgram = createUserWithPost(
      "Alice", "alice@example.com",
      "My First Post", "Hello, World!"
    )
    
    val (state1, (user1, post1)) = createProgram
      .foldMap(InMemoryInterpreter)
      .run(DbState())
      .value
    
    println(s"Created: $user1")
    println(s"Post: $post1")
    println(s"DB state: ${state1.users.size} users, ${state1.posts.size} posts")
    
    // Program 2: Transfer posts
    val fullProgram = for
      alice     <- insertUser("Alice", "alice@example.com")
      bob       <- insertUser("Bob", "bob@example.com")
      _         <- insertPost(alice.id, "Alice Post 1", "Content 1")
      _         <- insertPost(alice.id, "Alice Post 2", "Content 2")
      count     <- transferPosts(alice.id, bob.id)
      alicePosts <- selectPostsByUser(alice.id)
      bobPosts  <- selectPostsByUser(bob.id)
    yield (alice, bob, count, alicePosts, bobPosts)
    
    val (state2, (alice, bob, count, alicePosts, bobPosts)) = fullProgram
      .foldMap(InMemoryInterpreter)
      .run(DbState())
      .value
    
    println(s"\nTransferred $count posts from ${alice.name} to ${bob.name}")
    println(s"Alice's posts: ${alicePosts.length}")
    println(s"Bob's posts: ${bobPosts.length}")
```

---

## 8. Advanced Patterns

### Program Optimization

```scala
// Optimizer interpreter ที่ batch similar operations
class OptimizingInterpreter[G[_]: Monad](
  underlying: CrudOp ~> G,
  batch: List[SelectUser] => G[Map[Long, Option[User]]]
) extends (CrudOp ~> G):
  
  def apply[A](op: CrudOp[A]): G[A] = op match
    // สำหรับ select operations สามารถ batch ได้
    case SelectUser(id) =>
      underlying(op)  // simplified - real impl would batch
    case other =>
      underlying(other)
```

### Logging Interpreter

```scala
import cats.effect.IO

// Wrap any interpreter กับ logging
def withLogging[F[_]](interpreter: CrudOp ~> IO): CrudOp ~> IO =
  new (CrudOp ~> IO):
    def apply[A](op: CrudOp[A]): IO[A] =
      for
        _      <- IO.println(s"[DB] Executing: ${op.getClass.getSimpleName}")
        start  =  System.currentTimeMillis()
        result <- interpreter(op)
        elapsed = System.currentTimeMillis() - start
        _      <- IO.println(s"[DB] Completed in ${elapsed}ms")
      yield result
```

### Retry Interpreter

```scala
import cats.effect.IO
import scala.concurrent.duration.*

// Interpreter ที่ retry ถ้าเกิด error
def withRetry(
  interpreter: CrudOp ~> IO,
  maxRetries: Int = 3
): CrudOp ~> IO =
  new (CrudOp ~> IO):
    def apply[A](op: CrudOp[A]): IO[A] =
      def attempt(remaining: Int): IO[A] =
        interpreter(op).handleErrorWith { error =>
          if remaining > 0 then
            IO.sleep(100.millis) *> attempt(remaining - 1)
          else
            IO.raiseError(error)
        }
      attempt(maxRetries)
```

### Caching Interpreter

```scala
import cats.effect.{IO, Ref}

// Interpreter ที่ cache read results
def withCache(
  interpreter: CrudOp ~> IO,
  userCache: Ref[IO, Map[Long, User]]
): CrudOp ~> IO =
  new (CrudOp ~> IO):
    def apply[A](op: CrudOp[A]): IO[A] = op match
      case SelectUser(id) =>
        for
          cache  <- userCache.get
          result <- cache.get(id) match
            case Some(user) =>
              IO.println(s"Cache hit for user $id") *> IO.pure(Some(user))
            case None =>
              interpreter(SelectUser(id)).flatMap {
                case Some(user) =>
                  userCache.update(_ + (id -> user)).as(Some(user))
                case None =>
                  IO.pure(None)
              }
        yield result.asInstanceOf[A]
      case other =>
        interpreter(other)
```

---

## 9. สรุป

### Free Monad Pattern สรุป

```
1. Define Algebra (ADT)        <- What can be done
2. Create Smart Constructors   <- How to build programs  
3. Write Programs              <- What to do (no execution)
4. Create Interpreters         <- How to execute
5. foldMap(interpreter)        <- Actually run
```

### Interpreters ที่เรียนรู้

| Interpreter | ใช้สำหรับ | ข้อดี |
|-------------|-----------|-------|
| Pure (Id) | Testing, prototyping | Simple, no side effects |
| IO | Production | Full async, error handling |
| State | Testing with state | Verifiable, predictable |
| Logging | Debugging | Add logging transparently |
| Caching | Performance | Transparent memoization |
| Retry | Resilience | Automatic retries |

### Free Monad Architecture

```
┌──────────────────────────────────────────────┐
│              Business Logic Layer             │
│    (Programs: pure data, no side effects)    │
├──────────────────────────────────────────────┤
│           Algebra (ADT operations)           │
├──────────────────────────────────────────────┤
│              Interpreter Layer               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ IO Interp│  │Test Interp│  │Log Interp│  │
│  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────┘
```

### ขั้นตอนต่อไป

1. ศึกษา [Cats Effect](https://typelevel.org/cats-effect/) สำหรับ production-grade effects
2. ดู [ZIO](https://zio.dev/) เป็น alternative ที่ทรงพลังกว่า
3. ทำความเข้าใจ tradeoffs ระหว่าง Free Monad และ Tagless Final

### สรุปความแตกต่าง

```scala
// Free Monad: โปรแกรมเป็น data
val program: Free[UserOp, User] = for
  user <- createUser("Alice", "alice@example.com")
  ...
yield user

// สามารถ inspect, serialize, optimize ได้
// แล้ว execute ด้วย interpreter ที่ต้องการ
program.foldMap(testInterpreter)
program.foldMap(ioInterpreter)

// Tagless Final: โปรแกรมรัน immediately ใน F[_]
def program[F[_]: Monad: UserAlgebra]: F[User] = for
  user <- UserAlgebra[F].createUser("Alice", "alice@example.com")
  ...
yield user

// รัน immediatelly ใน concrete F
program[IO]     // production
program[TestIO] // testing
```

---

*[← Part 68: Cats MTL](part-68-cats-mtl.md) | [Part 70: Advanced Patterns →](part-70-advanced-patterns.md)*
