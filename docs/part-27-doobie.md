# Part 27: Doobie - Database Access

## สารบัญ
1. [Doobie Overview](#doobie-overview)
2. [Basic Queries](#basic-queries)
3. [Type-Safe Queries](#type-safe-queries)
4. [Transactions](#transactions)
5. [Connection Pooling](#connection-pooling)
6. [Schema Evolution](#schema-evolution)

---

## Doobie Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.tpolecat" %% "doobie-core"      % "1.0.0-RC4",
  "org.tpolecat" %% "doobie-postgres"  % "1.0.0-RC4",
  "org.tpolecat" %% "doobie-hikari"    % "1.0.0-RC4",
  "org.tpolecat" %% "doobie-scalatest" % "1.0.0-RC4" % Test,
  "com.h2database" % "h2"             % "2.2.224"
)
```

### Transactor

```scala
import doobie.*
import doobie.implicits.*
import cats.effect.{IO, Resource}

// Transactor: connection + transaction management
val xa = Transactor.fromDriverManager[IO](
  driver = "org.postgresql.Driver",
  url    = "jdbc:postgresql://localhost:5432/mydb",
  user   = "postgres",
  password = "password"
)

// H2 for testing
val testXa = Transactor.fromDriverManager[IO](
  driver   = "org.h2.Driver",
  url      = "jdbc:h2:mem:test;DB_CLOSE_DELAY=-1",
  user     = "sa",
  password = ""
)
```

---

## Basic Queries

### SELECT Queries

```scala
import doobie.*
import doobie.implicits.*

// Simple query
val query1: Query0[String] = sql"SELECT name FROM users".query[String]

// Run the query
val names: IO[List[String]] = query1.to[List].transact(xa)

// Query with parameters
def findByEmail(email: String): Query0[User] =
  sql"SELECT id, name, email, age FROM users WHERE email = $email".query[User]

// Case class mapping (automatic via Generic derivation)
case class User(id: Long, name: String, email: String, age: Int)

val allUsers: IO[List[User]] =
  sql"SELECT id, name, email, age FROM users ORDER BY name"
    .query[User]
    .to[List]
    .transact(xa)

// With options
val userById: IO[Option[User]] =
  sql"SELECT id, name, email, age FROM users WHERE id = 1"
    .query[User]
    .option
    .transact(xa)

// Stream large result
import fs2.Stream
val userStream: Stream[IO, User] =
  sql"SELECT id, name, email, age FROM users"
    .query[User]
    .stream
    .transact(xa)

// Process stream
userStream
  .filter(_.age > 18)
  .evalMap(user => IO.println(s"Adult: ${user.name}"))
  .compile
  .drain
  .unsafeRunSync()
```

### INSERT, UPDATE, DELETE

```scala
import doobie.*
import doobie.implicits.*

// INSERT
def createUser(name: String, email: String, age: Int): Update0 =
  sql"""
    INSERT INTO users (name, email, age)
    VALUES ($name, $email, $age)
  """.update

// INSERT returning generated key
def createUserReturningId(name: String, email: String, age: Int): ConnectionIO[Long] =
  sql"""
    INSERT INTO users (name, email, age)
    VALUES ($name, $email, $age)
  """.update.withUniqueGeneratedKeys[Long]("id")

// UPDATE
def updateUserEmail(id: Long, newEmail: String): Update0 =
  sql"UPDATE users SET email = $newEmail WHERE id = $id".update

// DELETE
def deleteUser(id: Long): Update0 =
  sql"DELETE FROM users WHERE id = $id".update

// Run
val createResult: IO[Int] =
  createUser("Alice", "alice@example.com", 30)
    .run
    .transact(xa)

val newId: IO[Long] =
  createUserReturningId("Bob", "bob@example.com", 25).transact(xa)
```

---

## Type-Safe Queries

### Meta and Put Instances

```scala
// Doobie ต้องการ Meta instance สำหรับ custom types

import doobie.*
import doobie.implicits.*

// Enum mapping
sealed trait UserRole
object UserRole:
  case object Admin   extends UserRole
  case object Regular extends UserRole
  case object Guest   extends UserRole

  given Meta[UserRole] = Meta[String].timap(
    {
      case "admin"   => Admin
      case "regular" => Regular
      case "guest"   => Guest
      case s         => throw new IllegalArgumentException(s"Unknown role: $s")
    }
  )(
    {
      case Admin   => "admin"
      case Regular => "regular"
      case Guest   => "guest"
    }
  )

case class User(id: Long, name: String, email: String, role: UserRole)

// Opaque type mapping
opaque type UserId = Long
object UserId:
  def apply(l: Long): UserId = l
  given Meta[UserId] = Meta[Long].timap(UserId(_))(identity)

// UUID mapping
import java.util.UUID
// PostgreSQL UUID
given Meta[UUID] = Meta[String].timap(UUID.fromString)(_.toString)

// JSON column
import io.circe.*
import io.circe.parser.*

def jsonMeta[A: Encoder: Decoder]: Meta[A] =
  Meta[String].timap { s =>
    decode[A](s).getOrElse(throw new RuntimeException(s"Invalid JSON: $s"))
  }(_.asJson.noSpaces)
```

### Fragment Composition

```scala
import doobie.*
import doobie.implicits.*
import doobie.util.fragment.Fragment

// Fragments: composable SQL pieces
case class UserFilter(
  name: Option[String] = None,
  minAge: Option[Int] = None,
  maxAge: Option[Int] = None,
  role: Option[String] = None
)

def buildQuery(filter: UserFilter): Query0[User] =
  val base = fr"SELECT id, name, email, age FROM users WHERE 1=1"

  val nameFilter  = filter.name.map(n => fr"AND name ILIKE ${"%" + n + "%"}")
  val minAgeFilter = filter.minAge.map(a => fr"AND age >= $a")
  val maxAgeFilter = filter.maxAge.map(a => fr"AND age <= $a")
  val roleFilter  = filter.role.map(r => fr"AND role = $r")

  val conditions = List(nameFilter, minAgeFilter, maxAgeFilter, roleFilter)
    .flatten
    .foldLeft(base)(_ ++ _)

  (conditions ++ fr"ORDER BY name").query[User]

// Dynamic IN clause
def findByIds(ids: List[Long]): Query0[User] =
  val inClause = ids match
    case Nil => fr"1=0"  // return nothing
    case _   => fr"id IN (" ++ ids.map(id => fr"$id").intercalate(fr",") ++ fr")"

  (fr"SELECT id, name, email, age FROM users WHERE" ++ inClause).query[User]
```

---

## Transactions

### Transaction Management

```scala
import doobie.*
import doobie.implicits.*
import cats.syntax.all.*

// ConnectionIO: operations in a single connection
def transferMoney(fromId: Long, toId: Long, amount: BigDecimal): ConnectionIO[Unit] =
  for
    fromBalance <- sql"SELECT balance FROM accounts WHERE id = $fromId FOR UPDATE"
                    .query[BigDecimal].unique
    _ <- if fromBalance < amount
         then doobie.free.connection.raiseError(new RuntimeException("Insufficient funds"))
         else sql"UPDATE accounts SET balance = balance - $amount WHERE id = $fromId".update.run
    _ <- sql"UPDATE accounts SET balance = balance + $amount WHERE id = $toId".update.run
  yield ()

// Run in transaction (all or nothing)
val transfer: IO[Unit] =
  transferMoney(1L, 2L, BigDecimal("100.00")).transact(xa)

// Savepoints
def transactionWithSavepoint: ConnectionIO[Unit] =
  for
    sp <- HC.setSavepoint("my_savepoint")
    _  <- sql"INSERT INTO log (msg) VALUES ('test')".update.run
    _  <- HC.rollback(sp)  // rollback to savepoint
    _  <- sql"INSERT INTO log (msg) VALUES ('after rollback')".update.run
  yield ()
```

---

## Connection Pooling

### HikariCP

```scala
import doobie.hikari.HikariTransactor
import cats.effect.{IO, Resource}
import scala.concurrent.ExecutionContext

def mkTransactor(
  url:      String,
  user:     String,
  password: String
): Resource[IO, HikariTransactor[IO]] =
  HikariTransactor.newHikariTransactor[IO](
    driverClassName = "org.postgresql.Driver",
    url             = url,
    user            = user,
    pass            = password,
    connectEC       = scala.concurrent.ExecutionContext.global
  )

// With configuration
def mkTransactorWithConfig: Resource[IO, HikariTransactor[IO]] =
  HikariTransactor.newHikariTransactor[IO](
    driverClassName = "org.postgresql.Driver",
    url             = "jdbc:postgresql://localhost:5432/mydb",
    user            = "postgres",
    pass            = "password",
    connectEC       = scala.concurrent.ExecutionContext.global
  ).evalTap { xa =>
    xa.configure { ds =>
      IO {
        ds.setMaximumPoolSize(20)
        ds.setMinimumIdle(5)
        ds.setConnectionTimeout(30000)
        ds.setIdleTimeout(600000)
        ds.setMaxLifetime(1800000)
      }
    }
  }

object App extends IOApp.Simple:
  def run: IO[Unit] =
    mkTransactorWithConfig.use { xa =>
      // Use xa here
      sql"SELECT count(*) FROM users".query[Long].unique.transact(xa).flatMap { n =>
        IO.println(s"Users: $n")
      }
    }
```

---

## Schema Evolution

### Flyway Integration

```scala
// build.sbt
libraryDependencies += "org.flywaydb" % "flyway-core" % "9.22.3"

// src/main/resources/db/migration/V1__create_users.sql
/*
CREATE TABLE users (
  id         BIGSERIAL PRIMARY KEY,
  name       VARCHAR(255) NOT NULL,
  email      VARCHAR(255) UNIQUE NOT NULL,
  age        INT,
  role       VARCHAR(50) DEFAULT 'regular',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
*/

// src/main/resources/db/migration/V2__add_profile.sql
/*
CREATE TABLE user_profiles (
  user_id    BIGINT PRIMARY KEY REFERENCES users(id),
  bio        TEXT,
  avatar_url VARCHAR(500),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
*/

// Running Flyway
import org.flywaydb.core.Flyway

def runMigrations(jdbcUrl: String, user: String, pass: String): IO[Unit] = IO {
  val flyway = Flyway.configure()
    .dataSource(jdbcUrl, user, pass)
    .locations("classpath:db/migration")
    .baselineOnMigrate(true)
    .load()
  flyway.migrate()
  println("Database migrations complete")
}
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Doobie setup กับ Transactor
- ✅ Basic queries: SELECT, INSERT, UPDATE, DELETE
- ✅ Type-safe mapping: Meta instances สำหรับ custom types
- ✅ Fragment composition สำหรับ dynamic queries
- ✅ Transactions: ConnectionIO
- ✅ Connection pooling ด้วย HikariCP
- ✅ Schema evolution ด้วย Flyway

---

*[← Part 26: http4s](part-26-http4s.md) | [Part 28: Circe JSON →](part-28-circe-json.md)*
