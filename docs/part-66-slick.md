# Part 66: Slick - Functional Relational Mapping

## สารบัญ

1. [แนะนำ Slick](#1-แนะนำ-slick)
2. [Setup และการตั้งค่า](#2-setup-และการตั้งค่า)
3. [Table Definitions](#3-table-definitions)
4. [CRUD Operations](#4-crud-operations)
5. [Queries และ Joins](#5-queries-และ-joins)
6. [Compiled Queries](#6-compiled-queries)
7. [Transactions](#7-transactions)
8. [Schema Generation](#8-schema-generation)
9. [Repository Pattern](#9-repository-pattern)
10. [Advanced Features](#10-advanced-features)
11. [สรุป](#11-สรุป)

---

## 1. แนะนำ Slick

Slick (Scala Language Integrated Connection Kit) เป็น Functional Relational Mapping (FRM) library สำหรับ Scala

### Slick vs ORM แบบดั้งเดิม

```
Traditional ORM (Hibernate/JPA):
- Mutable entities
- Implicit lazy loading
- Magic annotations
- N+1 query problems
- Difficult to reason about SQL

Slick (FRM):
- Immutable case classes
- Explicit async queries
- Type-safe SQL
- Composable queries
- Database-agnostic
```

### ข้อดีของ Slick

```scala
// Traditional SQL (String-based, error-prone)
val sql = s"SELECT * FROM users WHERE email = '$email'"  // SQL Injection!

// Slick (Type-safe, compiled)
val query = users.filter(_.email === email)  // Safe, type-checked at compile time
```

### Supported Databases

| Database | Profile |
|----------|---------|
| H2 | `slick.jdbc.H2Profile` |
| MySQL | `slick.jdbc.MySQLProfile` |
| PostgreSQL | `slick.jdbc.PostgresProfile` |
| SQLite | `slick.jdbc.SQLiteProfile` |
| Oracle | `slick.jdbc.OracleProfile` |
| SQL Server | `slick.jdbc.SQLServerProfile` |

---

## 2. Setup และการตั้งค่า

### build.sbt

```scala
// build.sbt
libraryDependencies ++= Seq(
  // Slick core
  "com.typesafe.slick" %% "slick"          % "3.4.1",
  "com.typesafe.slick" %% "slick-hikaricp" % "3.4.1",  // Connection pool
  
  // Database drivers
  "org.postgresql"     %  "postgresql"     % "42.6.0",  // PostgreSQL
  "com.h2database"     %  "h2"             % "2.2.224", // H2 (testing)
  "mysql"              %  "mysql-connector-java" % "8.0.33",  // MySQL
  
  // Logging (สำหรับ SQL logging)
  "ch.qos.logback"     %  "logback-classic" % "1.4.11"
)
```

### application.conf

```hocon
# conf/application.conf

# Default database (PostgreSQL)
slick.dbs.default {
  profile = "slick.jdbc.PostgresProfile$"
  db {
    driver   = "org.postgresql.Driver"
    url      = "jdbc:postgresql://localhost:5432/mydb"
    user     = "myuser"
    password = "mypassword"
    
    # Connection pool settings
    numThreads    = 10
    maxConnections = 10
    minConnections = 5
    connectionTimeout = 30000  # 30 seconds
    
    # HikariCP settings
    hikaricp {
      connectionTestQuery = "SELECT 1"
      maxLifetime         = 600000  # 10 minutes
      idleTimeout         = 60000   # 1 minute
    }
  }
}

# Test database (H2 in-memory)
slick.dbs.test {
  profile = "slick.jdbc.H2Profile$"
  db {
    driver = "org.h2.Driver"
    url    = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;MODE=PostgreSQL"
  }
}
```

### logback.xml สำหรับ SQL Logging

```xml
<!-- conf/logback.xml -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%date{HH:mm:ss.SSS} [%thread] %-5level %logger - %message%n</pattern>
    </encoder>
  </appender>
  
  <!-- แสดง SQL queries ที่ถูก execute -->
  <logger name="slick.jdbc.JdbcBackend.statement" level="DEBUG"/>
  <!-- แสดง bind variables -->
  <logger name="slick.jdbc.JdbcBackend.parameter" level="DEBUG"/>
  <!-- แสดง result sets -->
  <logger name="slick.jdbc.JdbcResultSetAction" level="DEBUG"/>
  
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

### DatabaseConfig Setup

```scala
// app/database/DatabaseConfig.scala
package database

import slick.jdbc.PostgresProfile.api.*
import com.typesafe.config.ConfigFactory

object DatabaseConfig:
  // Standalone (ไม่ใช้ Play)
  val db = Database.forConfig("slick.dbs.default.db")
  
  // ปิด database เมื่อเลิกใช้
  def close(): Unit = db.close()
```

---

## 3. Table Definitions

### Basic Table

```scala
// app/models/Tables.scala
package models

import slick.jdbc.PostgresProfile.api.*
import java.time.{Instant, LocalDate}

// Row case class
case class UserRow(
  id:        Long    = 0L,
  email:     String,
  name:      String,
  role:      String  = "viewer",
  createdAt: Instant = Instant.now(),
  updatedAt: Instant = Instant.now(),
  isActive:  Boolean = true
)

// Table definition
class UsersTable(tag: Tag) extends Table[UserRow](tag, "users"):
  // Primary key (auto increment)
  def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
  
  // Unique constraint
  def email = column[String]("email", O.Unique, O.Length(255))
  
  // Regular columns
  def name = column[String]("name", O.Length(100))
  
  // Column with default value
  def role = column[String]("role", O.Default("viewer"), O.Length(20))
  
  // Timestamp columns - need custom mapping
  implicit val instantMapper = MappedColumnType.base[Instant, Long](
    _.toEpochMilli,
    Instant.ofEpochMilli
  )
  
  def createdAt = column[Instant]("created_at")
  def updatedAt = column[Instant]("updated_at")
  def isActive  = column[Boolean]("is_active", O.Default(true))

  // Default projection (ใช้ <> สำหรับ mapping กับ case class)
  def * = (id, email, name, role, createdAt, updatedAt, isActive)
    .<>(UserRow.apply.tupled, UserRow.unapply)

// TableQuery - entry point สำหรับ queries
object UserTable:
  val query = TableQuery[UsersTable]
```

### Custom Column Mappings

```scala
// Custom types สำหรับ Slick
import slick.jdbc.PostgresProfile.api.*

// Enum mapping
enum UserRole:
  case Admin, Editor, Viewer

given MappedColumnType.base[UserRole, String](
  role => role.toString.toLowerCase,
  str  => UserRole.valueOf(str.capitalize)
)

// Option[Instant] mapping
given MappedColumnType.base[Option[Instant], Long](
  opt => opt.map(_.toEpochMilli).getOrElse(0L),
  ts  => if ts == 0L then None else Some(Instant.ofEpochMilli(ts))
)

// JSON column (PostgreSQL jsonb)
import play.api.libs.json.*
given jsonColumnMapper[T: Format]: MappedColumnType.base[T, String](
  value => Json.stringify(Json.toJson(value)),
  str   => Json.parse(str).as[T]
)
```

### Relationships และ Foreign Keys

```scala
// Post table ที่ reference User
case class PostRow(
  id:        Long    = 0L,
  title:     String,
  content:   String,
  authorId:  Long,
  createdAt: Instant = Instant.now()
)

class PostsTable(tag: Tag) extends Table[PostRow](tag, "posts"):
  def id        = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def title     = column[String]("title", O.Length(200))
  def content   = column[String]("content")
  def authorId  = column[Long]("author_id")
  def createdAt = column[Instant]("created_at")

  // Foreign key constraint
  def authorFk = foreignKey("fk_post_author", authorId, UserTable.query)(
    _.id,
    onUpdate = ForeignKeyAction.Cascade,
    onDelete = ForeignKeyAction.Cascade
  )

  def * = (id, title, content, authorId, createdAt)
    .<>(PostRow.apply.tupled, PostRow.unapply)

object PostTable:
  val query = TableQuery[PostsTable]
```

### Indexes

```scala
class UsersTable(tag: Tag) extends Table[UserRow](tag, "users"):
  // ... columns ...
  
  // Single column index
  def emailIdx    = index("idx_users_email", email, unique = true)
  
  // Composite index
  def roleNameIdx = index("idx_users_role_name", (role, name))
  
  def * = // ...
```

---

## 4. CRUD Operations

### Setup Database Runner

```scala
// app/database/Database.scala
package database

import slick.jdbc.PostgresProfile.api.*
import scala.concurrent.Future

trait DatabaseRunner:
  val db: Database
  
  protected def run[T](action: DBIO[T]): Future[T] = db.run(action)

object DB extends DatabaseRunner:
  val db = Database.forConfig("slick.dbs.default.db")
```

### Create (Insert)

```scala
import slick.jdbc.PostgresProfile.api.*
import models.*

val users = UserTable.query

// Insert single row
def insertUser(user: UserRow): Future[UserRow] =
  val insertAction = (users returning users.map(_.id) into { (row, id) =>
    row.copy(id = id)
  }) += user
  DB.run(insertAction)

// Insert and get auto-generated ID
def insertGetId(user: UserRow): Future[Long] =
  DB.run((users returning users.map(_.id)) += user)

// Insert multiple rows
def insertBatch(userList: Seq[UserRow]): Future[Option[Int]] =
  DB.run(users ++= userList)

// Upsert (insert or update)
def upsertUser(user: UserRow): Future[Int] =
  DB.run(users.insertOrUpdate(user))
```

### Read (Select)

```scala
// Select all
def findAll: Future[Seq[UserRow]] =
  DB.run(users.result)

// Select by ID
def findById(id: Long): Future[Option[UserRow]] =
  DB.run(users.filter(_.id === id).result.headOption)

// Select with condition
def findByEmail(email: String): Future[Option[UserRow]] =
  DB.run(users.filter(_.email === email).result.headOption)

// Select with multiple conditions
def findActive: Future[Seq[UserRow]] =
  DB.run(
    users
      .filter(u => u.isActive === true && u.role === "viewer")
      .sortBy(_.name.asc)
      .result
  )

// Select specific columns
def findEmails: Future[Seq[String]] =
  DB.run(users.map(_.email).result)

// Select with pagination
def findPage(page: Int, size: Int): Future[Seq[UserRow]] =
  val offset = (page - 1) * size
  DB.run(
    users
      .sortBy(_.id.asc)
      .drop(offset)
      .take(size)
      .result
  )

// Count
def countActive: Future[Int] =
  DB.run(users.filter(_.isActive === true).length.result)
```

### Update

```scala
// Update single field
def updateName(id: Long, newName: String): Future[Int] =
  DB.run(
    users
      .filter(_.id === id)
      .map(_.name)
      .update(newName)
  )

// Update multiple fields
def updateNameAndRole(id: Long, name: String, role: String): Future[Int] =
  DB.run(
    users
      .filter(_.id === id)
      .map(u => (u.name, u.role, u.updatedAt))
      .update((name, role, Instant.now()))
  )

// Update with condition
def deactivateByRole(role: String): Future[Int] =
  DB.run(
    users
      .filter(_.role === role)
      .map(_.isActive)
      .update(false)
  )

// Conditional update
def updateIfActive(id: Long, name: String): Future[Int] =
  DB.run(
    users
      .filter(u => u.id === id && u.isActive === true)
      .map(_.name)
      .update(name)
  )
```

### Delete

```scala
// Delete by ID
def deleteById(id: Long): Future[Int] =
  DB.run(users.filter(_.id === id).delete)

// Delete with condition
def deleteInactive: Future[Int] =
  DB.run(users.filter(_.isActive === false).delete)

// Soft delete
def softDelete(id: Long): Future[Int] =
  DB.run(
    users
      .filter(_.id === id)
      .map(u => (u.isActive, u.updatedAt))
      .update((false, Instant.now()))
  )
```

---

## 5. Queries และ Joins

### Query Composition

```scala
// Base query
val activeUsers = users.filter(_.isActive === true)

// Compose queries
val adminUsers = activeUsers.filter(_.role === "admin")
val sortedAdmins = adminUsers.sortBy(_.name.asc)
val pagedAdmins = sortedAdmins.drop(0).take(10)

// Execute
DB.run(pagedAdmins.result)
```

### Inner Join

```scala
val posts = PostTable.query

// Join users and posts
val userPostsQuery = for
  post <- posts
  user <- users if user.id === post.authorId
yield (user.name, post.title, post.createdAt)

DB.run(userPostsQuery.result).map { results =>
  results.map { case (authorName, title, createdAt) =>
    s"$authorName: $title ($createdAt)"
  }
}

// Explicit join syntax
val explicitJoin = posts
  .join(users)
  .on(_.authorId === _.id)
  .map { case (post, user) => (post, user.name) }
```

### Left Join

```scala
// Left join: get all users with their post count
val userWithPosts = users
  .joinLeft(posts)
  .on(_.id === _.authorId)
  .groupBy { case (user, _) => user.id }
  .map { case (userId, group) =>
    (userId, group.length)
  }

// Left join ที่ preserve null (Option)
val usersOptPosts = for
  user     <- users
  postOpt  <- posts.filter(_.authorId === user.id).result.headOption
yield (user, postOpt)
```

### Aggregations

```scala
// Count by role
val countByRole = users
  .groupBy(_.role)
  .map { case (role, group) => (role, group.length) }

DB.run(countByRole.result).map { results =>
  results.foreach { case (role, count) =>
    println(s"$role: $count users")
  }
}

// Sum, avg, min, max
val postStats = posts
  .groupBy(_.authorId)
  .map { case (authorId, group) =>
    (authorId, group.length, group.map(_.title).length)
  }

// Having clause equivalent
val activeAuthors = posts
  .groupBy(_.authorId)
  .map { case (authorId, group) => (authorId, group.length) }
  .filter { case (_, count) => count > 5 }
```

### Subqueries

```scala
// Select users who have posts
val usersWithPosts = users.filter { user =>
  posts.filter(_.authorId === user.id).exists
}

// Select users without any posts
val usersWithoutPosts = users.filterNot { user =>
  posts.filter(_.authorId === user.id).exists
}

// In subquery
val adminIds = users.filter(_.role === "admin").map(_.id)
val adminPosts = posts.filter(_.authorId in adminIds)
```

### Complex Queries

```scala
// Multi-table query
case class UserPostSummary(
  userId: Long,
  userName: String,
  postCount: Int,
  lastPostDate: Option[Instant]
)

val summaryQuery = for
  (userId, (count, lastDate)) <- posts
    .groupBy(_.authorId)
    .map { case (authorId, group) =>
      (authorId, (group.length, group.map(_.createdAt).max))
    }
  user <- users.filter(_.id === userId)
yield (user.id, user.name, count, lastDate)

DB.run(summaryQuery.result).map { rows =>
  rows.map { case (id, name, count, lastDate) =>
    UserPostSummary(id, name, count, lastDate)
  }
}
```

---

## 6. Compiled Queries

Compiled queries ช่วยเพิ่มประสิทธิภาพโดย compile query เพียงครั้งเดียว

### Basic Compiled Query

```scala
import slick.jdbc.PostgresProfile.api.*

// Compile query ครั้งเดียว
val findByIdCompiled = Compiled { (id: Rep[Long]) =>
  users.filter(_.id === id)
}

// ใช้ซ้ำได้หลายครั้งโดยไม่ต้อง compile ใหม่
def findById(id: Long): Future[Option[UserRow]] =
  DB.run(findByIdCompiled(id).result.headOption)

// Compiled query สำหรับ pagination
val findPageCompiled = Compiled { (offset: ConstColumn[Long], limit: ConstColumn[Long]) =>
  users
    .filter(_.isActive === true)
    .sortBy(_.id.asc)
    .drop(offset)
    .take(limit)
}

def findPage(page: Int, size: Int): Future[Seq[UserRow]] =
  val offset = ((page - 1) * size).toLong
  DB.run(findPageCompiled(offset, size.toLong).result)
```

### Compiled Queries ที่ซับซ้อน

```scala
// Compiled update
val updateNameCompiled = Compiled { (id: Rep[Long]) =>
  users.filter(_.id === id).map(u => (u.name, u.updatedAt))
}

def updateName(id: Long, name: String): Future[Int] =
  DB.run(updateNameCompiled(id).update((name, Instant.now())))

// Compiled count
val countByRoleCompiled = Compiled { (role: Rep[String]) =>
  users.filter(_.role === role).length
}

def countByRole(role: String): Future[Int] =
  DB.run(countByRoleCompiled(role).result)
```

### Performance Comparison

```scala
// ไม่ใช้ compiled query - parse ทุกครั้ง
def slowFindById(id: Long): Future[Option[UserRow]] =
  DB.run(users.filter(_.id === id).result.headOption)

// ใช้ compiled query - parse ครั้งเดียว
val fastFindByIdQ = Compiled { (id: Rep[Long]) =>
  users.filter(_.id === id)
}

def fastFindById(id: Long): Future[Option[UserRow]] =
  DB.run(fastFindByIdQ(id).result.headOption)

// สำหรับ high-traffic endpoints ควรใช้ compiled query เสมอ
```

---

## 7. Transactions

### Basic Transaction

```scala
import slick.jdbc.PostgresProfile.api.*

// Transaction: transfer between accounts
def transferFunds(fromId: Long, toId: Long, amount: BigDecimal): Future[Unit] =
  val action = for
    from <- accounts.filter(_.id === fromId).result.headOption
    to   <- accounts.filter(_.id === toId).result.headOption
    _ <- (from, to) match
      case (Some(f), Some(t)) if f.balance >= amount =>
        val debit  = accounts.filter(_.id === fromId)
          .map(_.balance).update(f.balance - amount)
        val credit = accounts.filter(_.id === toId)
          .map(_.balance).update(t.balance + amount)
        debit >> credit
      case (Some(f), _) if f.balance < amount =>
        DBIO.failed(new Exception("Insufficient funds"))
      case _ =>
        DBIO.failed(new Exception("Account not found"))
  yield ()
  
  DB.run(action.transactionally)
```

### Transaction with Rollback

```scala
// Transaction ที่จะ rollback ถ้ามี error
def createUserWithProfile(user: UserRow, profile: ProfileRow): Future[(UserRow, ProfileRow)] =
  val insertUserQ = (users returning users.map(_.id) into { (row, id) =>
    row.copy(id = id)
  }) += user
  
  val action = for
    savedUser    <- insertUserQ
    savedProfile <- (profiles += profile.copy(userId = savedUser.id))
                    .map(_ => profile.copy(userId = savedUser.id))
  yield (savedUser, savedProfile)
  
  // .transactionally ทำให้ทั้งสอง operations อยู่ใน transaction เดียว
  // ถ้า profiles insert ล้มเหลว user ก็จะถูก rollback ด้วย
  DB.run(action.transactionally)

// Transaction isolation level
def criticalUpdate(id: Long): Future[Int] =
  val action = users.filter(_.id === id).map(_.name).update("updated")
  DB.run(
    action.transactionally.withTransactionIsolation(
      TransactionIsolation.Serializable
    )
  )
```

### Batch Operations ใน Transaction

```scala
// Batch insert หลาย records ใน transaction เดียว
def batchInsert(userList: Seq[UserRow]): Future[Seq[UserRow]] =
  val insertActions = userList.map { user =>
    (users returning users.map(_.id) into { (row, id) =>
      row.copy(id = id)
    }) += user
  }
  
  val batchAction = DBIO.sequence(insertActions)
  DB.run(batchAction.transactionally)

// Efficient batch insert (ไม่ต้องการ IDs)
def bulkInsert(userList: Seq[UserRow]): Future[Option[Int]] =
  DB.run((users ++= userList).transactionally)
```

---

## 8. Schema Generation

### DDL Operations

```scala
import slick.jdbc.PostgresProfile.api.*

// Create schema
val schemaAction = (users.schema ++ posts.schema).create

// Create ถ้ายังไม่มี
val safeCreateAction = (users.schema ++ posts.schema).createIfNotExists

// Drop schema
val dropAction = (posts.schema ++ users.schema).drop

// Drop ถ้ามี (reverse order สำหรับ FK)
val safeDropAction = (posts.schema ++ users.schema).dropIfExists

// Truncate
val truncateAction = users.delete

// Execute DDL
DB.run(safeCreateAction)
```

### Schema Migration

```scala
// Simple migration system
object Migrations:
  
  def v1_createUsers: DBIO[Unit] = 
    users.schema.createIfNotExists
  
  def v2_addPosts: DBIO[Unit] = 
    posts.schema.createIfNotExists
  
  def v3_addIndex: DBIO[Unit] = 
    sqlu"CREATE INDEX IF NOT EXISTS idx_posts_author ON posts(author_id)"
  
  val allMigrations: Seq[DBIO[Unit]] = Seq(
    v1_createUsers,
    v2_addPosts,
    v3_addIndex
  )
  
  def runAll(): Future[Unit] =
    DB.run(DBIO.sequence(allMigrations).transactionally.map(_ => ()))

// Print DDL SQL
def printSchema(): Unit =
  val schema = (users.schema ++ posts.schema)
  println("CREATE statements:")
  schema.createStatements.foreach(println)
  println("\nDROP statements:")
  schema.dropStatements.foreach(println)
```

### Flyway Integration

```scala
// build.sbt
libraryDependencies += "org.flywaydb" % "flyway-core" % "9.22.3"

// app/database/FlywayMigration.scala
import org.flywaydb.core.Flyway
import javax.inject.*
import play.api.Configuration

@Singleton
class FlywayMigration @Inject() (config: Configuration):
  
  def migrate(): Int =
    val flyway = Flyway.configure()
      .dataSource(
        config.get[String]("slick.dbs.default.db.url"),
        config.get[String]("slick.dbs.default.db.user"),
        config.get[String]("slick.dbs.default.db.password")
      )
      .locations("classpath:db/migration")
      .load()
    flyway.migrate().migrationsExecuted

// ไฟล์ migration: conf/db/migration/V1__Create_users.sql
// V1__Create_users.sql
```

---

## 9. Repository Pattern

### Generic Repository Interface

```scala
// app/repositories/Repository.scala
package repositories

import scala.concurrent.Future

trait Repository[T, ID]:
  def findById(id: ID): Future[Option[T]]
  def findAll(page: Int, size: Int): Future[Seq[T]]
  def count: Future[Int]
  def insert(entity: T): Future[T]
  def update(id: ID, entity: T): Future[Option[T]]
  def delete(id: ID): Future[Boolean]
```

### UserRepository Implementation

```scala
// app/repositories/UserRepository.scala
package repositories

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.db.slick.{DatabaseConfigProvider, HasDatabaseConfigProvider}
import slick.jdbc.JdbcProfile
import models.*

@Singleton
class UserRepository @Inject() (
  protected val dbConfigProvider: DatabaseConfigProvider
)(using ExecutionContext)
    extends HasDatabaseConfigProvider[JdbcProfile]
    with Repository[UserRow, Long]:

  import profile.api.*

  private val users = UserTable.query

  // Compiled queries สำหรับ performance
  private val findByIdQ = Compiled { (id: Rep[Long]) =>
    users.filter(_.id === id)
  }
  
  private val findByEmailQ = Compiled { (email: Rep[String]) =>
    users.filter(_.email === email)
  }
  
  private val findPageQ = Compiled {
    (offset: ConstColumn[Long], limit: ConstColumn[Long]) =>
      users
        .filter(_.isActive === true)
        .sortBy(_.id.asc)
        .drop(offset)
        .take(limit)
  }

  override def findById(id: Long): Future[Option[UserRow]] =
    db.run(findByIdQ(id).result.headOption)

  override def findAll(page: Int, size: Int): Future[Seq[UserRow]] =
    val offset = ((page - 1) * size).toLong
    db.run(findPageQ(offset, size.toLong).result)

  override def count: Future[Int] =
    db.run(users.filter(_.isActive === true).length.result)

  override def insert(user: UserRow): Future[UserRow] =
    db.run(
      (users returning users.map(_.id) into { (row, id) =>
        row.copy(id = id)
      }) += user
    )

  override def update(id: Long, user: UserRow): Future[Option[UserRow]] =
    db.run(
      users
        .filter(_.id === id)
        .map(u => (u.name, u.role, u.updatedAt))
        .update((user.name, user.role, Instant.now()))
    ).flatMap { count =>
      if count > 0 then findById(id)
      else Future.successful(None)
    }

  override def delete(id: Long): Future[Boolean] =
    db.run(
      users
        .filter(_.id === id)
        .map(u => (u.isActive, u.updatedAt))
        .update((false, Instant.now()))
    ).map(_ > 0)

  // Extra methods specific to User
  def findByEmail(email: String): Future[Option[UserRow]] =
    db.run(findByEmailQ(email).result.headOption)

  def findByRole(role: String, page: Int, size: Int): Future[Seq[UserRow]] =
    val offset = (page - 1) * size
    db.run(
      users
        .filter(u => u.role === role && u.isActive === true)
        .sortBy(_.name.asc)
        .drop(offset)
        .take(size)
        .result
    )

  def existsByEmail(email: String): Future[Boolean] =
    db.run(users.filter(_.email === email).exists.result)

  def hardDelete(id: Long): Future[Boolean] =
    db.run(users.filter(_.id === id).delete).map(_ > 0)
```

### Service Layer

```scala
// app/services/UserService.scala
package services

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import models.*
import repositories.UserRepository

@Singleton
class UserService @Inject() (
  userRepo: UserRepository
)(using ExecutionContext):

  def getUsers(page: Int, size: Int): Future[Page[User]] =
    for
      rows  <- userRepo.findAll(page, size)
      total <- userRepo.count
    yield Page(rows.map(toDomain), page, size, total.toLong)

  def getUserById(id: Long): Future[Option[User]] =
    userRepo.findById(id).map(_.map(toDomain))

  def createUser(email: String, name: String, role: UserRole): Future[User] =
    for
      exists <- userRepo.existsByEmail(email)
      _      <- if exists then Future.failed(DuplicateEmailException(email))
                else Future.unit
      row    = UserRow(email = email, name = name, role = role.toString.toLowerCase)
      saved  <- userRepo.insert(row)
    yield toDomain(saved)

  def updateUser(id: Long, name: Option[String], role: Option[UserRole]): Future[Option[User]] =
    for
      existing <- userRepo.findById(id)
      result   <- existing match
        case None => Future.successful(None)
        case Some(row) =>
          val updated = row.copy(
            name = name.getOrElse(row.name),
            role = role.map(_.toString.toLowerCase).getOrElse(row.role)
          )
          userRepo.update(id, updated)
    yield result.map(toDomain)

  def deleteUser(id: Long): Future[Boolean] =
    userRepo.delete(id)

  private def toDomain(row: UserRow): User =
    User(
      id        = row.id,
      email     = row.email,
      name      = row.name,
      role      = UserRole.valueOf(row.role.capitalize),
      createdAt = row.createdAt,
      isActive  = row.isActive
    )

case class DuplicateEmailException(email: String)
    extends Exception(s"Email $email already exists")
```

---

## 10. Advanced Features

### Raw SQL

```scala
import slick.jdbc.PostgresProfile.api.*

// Plain SQL queries
def searchUsers(query: String): Future[Seq[UserRow]] =
  DB.run(
    sql"""
      SELECT id, email, name, role, created_at, updated_at, is_active
      FROM users
      WHERE name ILIKE ${s"%$query%"}
        OR email ILIKE ${s"%$query%"}
      ORDER BY name ASC
      LIMIT 20
    """.as[UserRow]
  )

// Raw SQL with custom mapper
implicit val userRowGetResult = GetResult[UserRow] { r =>
  UserRow(
    id        = r.<<,
    email     = r.<<,
    name      = r.<<,
    role      = r.<<,
    createdAt = Instant.ofEpochMilli(r.<<[Long]),
    updatedAt = Instant.ofEpochMilli(r.<<[Long]),
    isActive  = r.<<
  )
}

// DML (UPDATE, INSERT, DELETE)
def bulkUpdateRole(oldRole: String, newRole: String): Future[Int] =
  DB.run(sqlu"UPDATE users SET role = $newRole WHERE role = $oldRole")
```

### Streaming

```scala
import org.apache.pekko.stream.scaladsl.Source

// Stream large result sets โดยไม่โหลดทั้งหมดเข้า memory
def streamAllUsers: Source[UserRow, ?] =
  Source.fromPublisher(
    db.stream(users.result)
  )

// ใช้กับ Akka Streams
def exportUsers(sink: Sink[UserRow, Future[Done]]): Future[Done] =
  streamAllUsers.runWith(sink)
```

### Database Health Check

```scala
// app/health/DatabaseHealthCheck.scala
import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.JdbcProfile

@Singleton
class DatabaseHealthCheck @Inject() (
  dbConfigProvider: DatabaseConfigProvider
)(using ExecutionContext):

  private val dbConfig = dbConfigProvider.get[JdbcProfile]
  import dbConfig.*
  import profile.api.*

  def isHealthy: Future[Boolean] =
    db.run(sql"SELECT 1".as[Int])
      .map(_.nonEmpty)
      .recover(_ => false)
```

---

## 11. สรุป

Slick เป็น type-safe, functional database library ที่ทรงพลัง:

### สรุปความสามารถ

| Feature | Description |
|---------|-------------|
| Table Definitions | Type-safe table mappings |
| CRUD | Async, type-safe operations |
| Queries | Composable, chainable queries |
| Joins | Inner, left, right, full outer |
| Transactions | ACID-compliant with rollback |
| Compiled Queries | Pre-compiled for performance |
| Raw SQL | Escape hatch for complex queries |
| Streaming | Memory-efficient large datasets |

### Best Practices

1. **ใช้ Compiled Queries** - สำหรับ queries ที่เรียกบ่อย
2. **Repository Pattern** - แยก data access logic
3. **Transaction Boundaries** - ชัดเจนว่าอะไรอยู่ใน transaction เดียวกัน
4. **Connection Pool** - ตั้งค่า HikariCP อย่างเหมาะสม
5. **Avoid N+1** - ใช้ join แทนการ query หลายครั้ง

### Common Pitfalls

```scala
// ❌ N+1 Problem
val badQuery = for
  user  <- DB.run(users.result)
  posts <- DB.run(posts.filter(_.authorId === user.id).result)  // N queries!
yield (user, posts)

// ✅ Use Join instead
val goodQuery = DB.run(
  users.joinLeft(posts).on(_.id === _.authorId).result
)
```

---

*[← Part 65: Play Framework](part-65-play-framework.md) | [Part 67: Shapeless →](part-67-shapeless.md)*
