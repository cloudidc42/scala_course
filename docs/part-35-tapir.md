# Part 35: Tapir - Type-Safe API

## สารบัญ
1. [Tapir Overview](#tapir-overview)
2. [Endpoint Definition](#endpoint-definition)
3. [Server Implementation](#server-implementation)
4. [Client Generation](#client-generation)
5. [Documentation](#documentation)
6. [Complete Example](#complete-example)

---

## Tapir Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "com.softwaremill.sttp.tapir" %% "tapir-core"             % "1.9.3",
  "com.softwaremill.sttp.tapir" %% "tapir-json-circe"       % "1.9.3",
  "com.softwaremill.sttp.tapir" %% "tapir-http4s-server"    % "1.9.3",
  "com.softwaremill.sttp.tapir" %% "tapir-swagger-ui-bundle" % "1.9.3",
  "com.softwaremill.sttp.tapir" %% "tapir-sttp-client"      % "1.9.3",
  "org.http4s"                  %% "http4s-ember-server"    % "0.23.24"
)
```

### Core Concept

```
Tapir: Typed API descRiptions

Endpoint[SECURITY_INPUT, INPUT, ERROR_OUTPUT, OUTPUT, REQUIREMENTS]

- Describe endpoints as data
- Generate servers, clients, and docs from the same description
- Type-safe input/output

Benefits:
- Compile-time safety
- Auto-generated OpenAPI/Swagger docs
- Auto-generated clients
- No runtime surprises
```

---

## Endpoint Definition

### Basic Endpoints

```scala
import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.generic.auto.*
import io.circe.generic.auto.*

// Data models
case class User(id: Long, name: String, email: String)
case class CreateUser(name: String, email: String)
case class ErrorInfo(code: String, message: String)

// GET /users
val listUsers: Endpoint[Unit, Unit, ErrorInfo, List[User], Any] =
  endpoint
    .get
    .in("users")
    .out(jsonBody[List[User]])
    .errorOut(jsonBody[ErrorInfo])

// GET /users/{id}
val getUser: Endpoint[Unit, Long, ErrorInfo, User, Any] =
  endpoint
    .get
    .in("users" / path[Long]("id"))
    .out(jsonBody[User])
    .errorOut(jsonBody[ErrorInfo])

// POST /users
val createUser: Endpoint[Unit, CreateUser, ErrorInfo, User, Any] =
  endpoint
    .post
    .in("users")
    .in(jsonBody[CreateUser])
    .out(jsonBody[User].and(statusCode(sttp.model.StatusCode.Created)))
    .errorOut(jsonBody[ErrorInfo])

// PUT /users/{id}
val updateUser: Endpoint[Unit, (Long, CreateUser), ErrorInfo, User, Any] =
  endpoint
    .put
    .in("users" / path[Long]("id"))
    .in(jsonBody[CreateUser])
    .out(jsonBody[User])
    .errorOut(jsonBody[ErrorInfo])

// DELETE /users/{id}
val deleteUser: Endpoint[Unit, Long, ErrorInfo, Unit, Any] =
  endpoint
    .delete
    .in("users" / path[Long]("id"))
    .out(emptyOutput)
    .errorOut(jsonBody[ErrorInfo])
```

### Advanced Endpoint Types

```scala
import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.model.{StatusCode, Header}
import sttp.tapir.model.UsernamePassword

// Query parameters
val searchUsers: Endpoint[Unit, (Option[String], Option[Int], Int), ErrorInfo, List[User], Any] =
  endpoint
    .get
    .in("users" / "search")
    .in(query[Option[String]]("name"))
    .in(query[Option[Int]]("minAge"))
    .in(query[Int]("limit").default(20))
    .out(jsonBody[List[User]])
    .errorOut(jsonBody[ErrorInfo])

// Headers
val authenticatedEndpoint: Endpoint[String, Unit, ErrorInfo, Unit, Any] =
  endpoint
    .securityIn(auth.bearer[String]())
    .errorOut(jsonBody[ErrorInfo])

// File upload
val uploadFile: Endpoint[Unit, (String, Array[Byte]), ErrorInfo, String, Any] =
  endpoint
    .post
    .in("upload")
    .in(multipartBody[
      (String, Array[Byte])
    ])
    .out(stringBody)
    .errorOut(jsonBody[ErrorInfo])

// Webhook description
case class WebhookPayload(event: String, data: String)

val webhookEndpoint: Endpoint[Unit, WebhookPayload, ErrorInfo, Unit, Any] =
  endpoint
    .post
    .in("webhook")
    .in(jsonBody[WebhookPayload])
    .out(emptyOutput)
    .errorOut(jsonBody[ErrorInfo])
```

---

## Server Implementation

### http4s Server

```scala
import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.server.http4s.Http4sServerInterpreter
import cats.effect.{IO, IOApp, Resource}
import org.http4s.HttpRoutes
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s.*
import io.circe.generic.auto.*
import sttp.tapir.generic.auto.*

// In-memory store
object UserStore:
  private var users = Map(
    1L -> User(1L, "Alice", "alice@example.com"),
    2L -> User(2L, "Bob", "bob@example.com")
  )
  private var nextId = 3L

  def getAll: IO[List[User]] = IO.pure(users.values.toList)
  def getById(id: Long): IO[Option[User]] = IO.pure(users.get(id))
  def create(req: CreateUser): IO[User] = IO {
    val user = User(nextId, req.name, req.email)
    users = users + (nextId -> user)
    nextId += 1
    user
  }
  def delete(id: Long): IO[Boolean] = IO {
    val existed = users.contains(id)
    users = users - id
    existed
  }

// Server implementations
val listUsersImpl = Http4sServerInterpreter[IO]().toRoutes(
  listUsers.serverLogic { _ =>
    UserStore.getAll.map(Right(_))
  }
)

val getUserImpl = Http4sServerInterpreter[IO]().toRoutes(
  getUser.serverLogic { id =>
    UserStore.getById(id).map {
      case Some(user) => Right(user)
      case None       => Left(ErrorInfo("not_found", s"User $id not found"))
    }
  }
)

val createUserImpl = Http4sServerInterpreter[IO]().toRoutes(
  createUser.serverLogic { req =>
    UserStore.create(req).map(user => Right((user, ())))
  }
)

val deleteUserImpl = Http4sServerInterpreter[IO]().toRoutes(
  deleteUser.serverLogic { id =>
    UserStore.delete(id).map {
      case true  => Right(())
      case false => Left(ErrorInfo("not_found", s"User $id not found"))
    }
  }
)

// Combine all routes
val allRoutes = listUsersImpl <+> getUserImpl <+> createUserImpl <+> deleteUserImpl
```

---

## Documentation

### Swagger UI

```scala
import sttp.tapir.swagger.bundle.SwaggerInterpreter
import sttp.tapir.server.http4s.Http4sServerInterpreter

// Collect all endpoints
val allEndpoints = List(
  listUsers, getUser, createUser, updateUser, deleteUser
)

// Generate Swagger docs routes
val swaggerRoutes = Http4sServerInterpreter[IO]().toRoutes(
  SwaggerInterpreter().fromEndpoints[IO](
    allEndpoints,
    "User API",
    "1.0.0"
  )
)

// Add descriptions to endpoints
val listUsersWithDocs =
  endpoint
    .get
    .in("users")
    .name("List Users")
    .description("Returns all users in the system")
    .tag("Users")
    .out(jsonBody[List[User]].description("List of users"))
    .errorOut(jsonBody[ErrorInfo].description("Error details"))
```

---

## Complete Example

### Full REST API with Tapir

```scala
import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.generic.auto.*
import sttp.tapir.server.http4s.Http4sServerInterpreter
import sttp.tapir.swagger.bundle.SwaggerInterpreter
import cats.effect.{IO, IOApp, Ref}
import org.http4s.ember.server.EmberServerBuilder
import org.http4s.server.Router
import com.comcast.ip4s.*
import io.circe.generic.auto.*
import cats.syntax.semigroupk.*

// Models
case class Product(id: Long, name: String, price: Double, stock: Int)
case class CreateProduct(name: String, price: Double, stock: Int)
case class ApiError(message: String)

// Endpoints definition
object ProductEndpoints:
  val base = endpoint.in("api" / "products").errorOut(jsonBody[ApiError])

  val list   = base.get.out(jsonBody[List[Product]])
  val getById = base.get.in(path[Long]("id")).out(jsonBody[Product])
  val create = base.post.in(jsonBody[CreateProduct]).out(jsonBody[Product])
  val update = base.put
    .in(path[Long]("id"))
    .in(jsonBody[CreateProduct])
    .out(jsonBody[Product])
  val delete = base.delete.in(path[Long]("id"))

  val all = List(list, getById, create, update, delete)

// Repository
class ProductRepo(ref: Ref[IO, Map[Long, Product]], counter: Ref[IO, Long]):
  def getAll: IO[List[Product]] = ref.get.map(_.values.toList)

  def getById(id: Long): IO[Either[ApiError, Product]] =
    ref.get.map(_.get(id).toRight(ApiError(s"Product $id not found")))

  def create(req: CreateProduct): IO[Product] =
    for
      id <- counter.getAndUpdate(_ + 1)
      p   = Product(id, req.name, req.price, req.stock)
      _  <- ref.update(_ + (id -> p))
    yield p

  def update(id: Long, req: CreateProduct): IO[Either[ApiError, Product]] =
    for
      exists <- ref.get.map(_.contains(id))
      result <- if exists then
        val updated = Product(id, req.name, req.price, req.stock)
        ref.update(_ + (id -> updated)).as(Right(updated))
      else IO.pure(Left(ApiError(s"Product $id not found")))
    yield result

  def delete(id: Long): IO[Either[ApiError, Unit]] =
    for
      exists <- ref.get.map(_.contains(id))
      result <- if exists then
        ref.update(_ - id).as(Right(()))
      else IO.pure(Left(ApiError(s"Product $id not found")))
    yield result

object ProductRoutes:
  def make(repo: ProductRepo): IO[org.http4s.HttpRoutes[IO]] = IO {
    val interpreter = Http4sServerInterpreter[IO]()
    import ProductEndpoints.*

    val routes = List(
      interpreter.toRoutes(list.serverLogic(_ => repo.getAll.map(Right(_)))),
      interpreter.toRoutes(getById.serverLogic(repo.getById)),
      interpreter.toRoutes(create.serverLogic(req => repo.create(req).map(Right(_)))),
      interpreter.toRoutes(update.serverLogic((id, req) => repo.update(id, req))),
      interpreter.toRoutes(delete.serverLogic(repo.delete))
    )

    val docs = interpreter.toRoutes(
      SwaggerInterpreter().fromEndpoints[IO](all, "Product API", "1.0")
    )

    routes.foldLeft(docs)(_ <+> _)
  }

object App extends IOApp.Simple:
  def run: IO[Unit] =
    for
      ref   <- Ref.of[IO, Map[Long, Product]](Map.empty)
      cnt   <- Ref.of[IO, Long](1L)
      repo   = ProductRepo(ref, cnt)
      routes <- ProductRoutes.make(repo)
      _ <- EmberServerBuilder.default[IO]
        .withHost(ipv4"0.0.0.0")
        .withPort(port"8080")
        .withHttpApp(Router("/" -> routes).orNotFound)
        .build
        .useForever
    yield ()
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Endpoint definition: type-safe API descriptions
- ✅ Path params, query params, headers, body
- ✅ Server implementation with http4s
- ✅ Auto-generated Swagger/OpenAPI docs
- ✅ Complete REST API with Tapir

---

*[← Part 34: fs2 Streams](part-34-fs2.md) | [Part 36: Redis →](part-36-redis.md)*
