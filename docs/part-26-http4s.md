# Part 26: http4s และ Cats Effect

## สารบัญ
1. [Cats Effect IO](#cats-effect-io)
2. [http4s Server](#http4s-server)
3. [HTTP Routes](#http-routes)
4. [Middleware](#middleware)
5. [Client](#client)

---

## Cats Effect IO

### IO Monad พื้นฐาน

```scala
import cats.effect.{IO, IOApp, ExitCode}
import cats.effect.unsafe.implicits.global

// IO[A]: description of an effect that produces A
// ไม่รันจนกว่าจะถูก run explicitly

// Pure values
val pure: IO[Int] = IO.pure(42)
val unit: IO[Unit] = IO.unit

// Side effects
val println: IO[Unit] = IO.println("Hello!")
val readLine: IO[String] = IO.readLine

// Delay: wrap synchronous effects
val getTime: IO[Long] = IO.delay(System.currentTimeMillis())
// Same as: IO(System.currentTimeMillis())

// Error handling
val failed: IO[Int] = IO.raiseError(new RuntimeException("oops"))
val recovered: IO[Int] = failed.handleError(_ => -1)
val handled: IO[Either[Throwable, Int]] = failed.attempt

// Sequencing
val program: IO[String] = for
  _    <- IO.println("Enter name:")
  name <- IO.readLine
  _    <- IO.println(s"Hello, $name!")
yield name

// Running IO (only in main)
val result = pure.unsafeRunSync()
```

### IOApp

```scala
import cats.effect.{IO, IOApp}

object Main extends IOApp.Simple:
  def run: IO[Unit] = for
    _ <- IO.println("Starting application...")
    _ <- IO.println("Processing...")
    _ <- IO.println("Done!")
  yield ()

// Or with ExitCode
object MainFull extends IOApp:
  def run(args: List[String]): IO[ExitCode] = for
    _ <- IO.println(s"Args: $args")
    _ <- program
  yield ExitCode.Success

  private def program: IO[Unit] =
    IO.println("Application running")
```

### Resource Management

```scala
import cats.effect.{IO, Resource}

// Resource: safe acquisition and release
def openFile(path: String): Resource[IO, java.io.BufferedReader] =
  Resource.make(
    IO(new java.io.BufferedReader(new java.io.FileReader(path)))
  )(reader => IO(reader.close()))

// Usage: file is always closed even on error
val lines = openFile("data.txt").use { reader =>
  IO(Iterator.continually(reader.readLine()).takeWhile(_ != null).toList)
}

// Database connection pool
import java.sql.{Connection, DriverManager}

def dbConnection(url: String, user: String, pass: String): Resource[IO, Connection] =
  Resource.make(
    IO(DriverManager.getConnection(url, user, pass))
  )(conn => IO(conn.close()))

// Compose resources
def withConnection[A](f: Connection => IO[A]): IO[A] =
  dbConnection("jdbc:postgresql://localhost/mydb", "user", "pass").use(f)

// Multiple resources
val program = (
  openFile("input.txt"),
  openFile("output.txt")  // simplified - should be BufferedWriter
).tupled.use { case (reader, _) =>
  IO(reader.readLine())
}
```

### Concurrent Programming

```scala
import cats.effect.{IO, Fiber}
import cats.syntax.parallel.*

// Parallel execution
val task1 = IO.sleep(scala.concurrent.duration.1.second) *> IO.pure(1)
val task2 = IO.sleep(scala.concurrent.duration.1.second) *> IO.pure(2)

// parMapN: run in parallel, wait for both
val parallel = (task1, task2).parMapN(_ + _)
// Takes ~1 second, not 2

// parSequence: List[IO[A]] -> IO[List[A]]
val tasks = List(IO.pure(1), IO.pure(2), IO.pure(3))
val all = tasks.parSequence

// Fiber: lightweight thread
val fiber = task1.start.flatMap { fiber =>
  task2.flatMap(r2 => fiber.join.map(r1 => r1 + r2))
}

// Ref: mutable state in IO
import cats.effect.Ref
val counter = for
  ref   <- Ref[IO].of(0)
  _     <- ref.update(_ + 1)
  _     <- ref.update(_ + 1)
  count <- ref.get
yield count

println(counter.unsafeRunSync())  // 2
```

---

## http4s Server

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-ember-server" % "0.23.23",
  "org.http4s" %% "http4s-ember-client" % "0.23.23",
  "org.http4s" %% "http4s-circe"        % "0.23.23",
  "org.http4s" %% "http4s-dsl"          % "0.23.23",
  "io.circe"   %% "circe-generic"       % "0.14.6",
  "io.circe"   %% "circe-parser"        % "0.14.6",
  "ch.qos.logback" % "logback-classic" % "1.4.11"
)
```

### Basic Server

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s.*

object Server extends IOApp.Simple:
  val helloService: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case GET -> Root / "hello" =>
      Ok("Hello, World!")

    case GET -> Root / "hello" / name =>
      Ok(s"Hello, $name!")

    case GET -> Root / "ping" =>
      Ok("pong")
  }

  def run: IO[Unit] =
    EmberServerBuilder
      .default[IO]
      .withHost(ipv4"0.0.0.0")
      .withPort(port"8080")
      .withHttpApp(helloService.orNotFound)
      .build
      .useForever
```

---

## HTTP Routes

### CRUD API

```scala
import cats.effect.*
import cats.syntax.all.*
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import org.http4s.circe.CirceEntityCodec.*
import io.circe.generic.auto.*
import io.circe.syntax.*

// Model
case class User(id: Option[Long], name: String, email: String)
case class CreateUser(name: String, email: String)
case class ErrorResponse(message: String)

// Repository
trait UserRepo[F[_]]:
  def findAll: F[List[User]]
  def findById(id: Long): F[Option[User]]
  def create(user: CreateUser): F[User]
  def update(id: Long, user: CreateUser): F[Option[User]]
  def delete(id: Long): F[Boolean]

// Routes
class UserRoutes[F[_]: Concurrent](repo: UserRepo[F]):
  val routes: HttpRoutes[F] =
    val dsl = new Http4sDsl[F]{}
    import dsl.*

    HttpRoutes.of[F] {
      case GET -> Root / "users" =>
        repo.findAll.flatMap(users => Ok(users.asJson))

      case GET -> Root / "users" / LongVar(id) =>
        repo.findById(id).flatMap {
          case Some(user) => Ok(user.asJson)
          case None       => NotFound(ErrorResponse(s"User $id not found").asJson)
        }

      case req @ POST -> Root / "users" =>
        for
          createReq <- req.as[CreateUser]
          user      <- repo.create(createReq)
          resp      <- Created(user.asJson)
        yield resp

      case req @ PUT -> Root / "users" / LongVar(id) =>
        for
          updateReq <- req.as[CreateUser]
          result    <- repo.update(id, updateReq)
          resp <- result match
            case Some(user) => Ok(user.asJson)
            case None       => NotFound(ErrorResponse(s"User $id not found").asJson)
        yield resp

      case DELETE -> Root / "users" / LongVar(id) =>
        repo.delete(id).flatMap { deleted =>
          if deleted then NoContent()
          else NotFound(ErrorResponse(s"User $id not found").asJson)
        }
    }
```

---

## Middleware

### Logging Middleware

```scala
import org.http4s.server.middleware.*
import scala.concurrent.duration.*

// Request/Response logging
def withLogging(routes: HttpRoutes[IO]): HttpRoutes[IO] =
  Logger.httpRoutes[IO](
    logHeaders = true,
    logBody = false,
    redactHeadersWhen = _ => false
  )(routes)

// CORS
def withCors(routes: HttpRoutes[IO]): HttpRoutes[IO] =
  CORS.policy
    .withAllowOriginAll
    .withAllowMethodsAll
    .withAllowHeadersAll
    .apply(routes)

// Rate limiting
def withRateLimit(routes: HttpRoutes[IO]): IO[HttpRoutes[IO]] =
  Throttle.httpRoutes[IO](
    amount = 100,
    per = 1.minute
  )(routes)

// Authentication middleware
def withAuth(routes: HttpRoutes[IO]): HttpRoutes[IO] =
  routes.compose(authMiddleware)

val authMiddleware: HttpMiddleware[IO] = routes =>
  Kleisli { req =>
    req.headers.get[headers.Authorization] match
      case Some(_) => routes(req)
      case None    =>
        OptionT.some[IO](
          Response[IO](Status.Unauthorized).withEntity("Unauthorized")
        )
  }
```

---

## Client

```scala
import org.http4s.ember.client.EmberClientBuilder
import org.http4s.client.Client

object HttpClientExample extends IOApp.Simple:
  def run: IO[Unit] =
    EmberClientBuilder.default[IO].build.use { client =>
      for
        _ <- fetchJson(client)
        _ <- postData(client)
      yield ()
    }

  def fetchJson(client: Client[IO]): IO[Unit] =
    client.expect[String]("https://api.example.com/data").flatMap { body =>
      IO.println(s"Got: ${body.take(100)}...")
    }

  def postData(client: Client[IO]): IO[Unit] =
    val request = Request[IO](
      method = Method.POST,
      uri = uri"https://api.example.com/users"
    ).withEntity("""{"name": "Alice"}""")

    client.expect[String](request).flatMap { response =>
      IO.println(s"Response: $response")
    }

  // Retry with middleware
  import org.http4s.client.middleware.Retry
  import org.http4s.client.middleware.RetryPolicy

  def withRetry(client: Client[IO]): Client[IO] =
    Retry[IO](RetryPolicy(
      backoff = RetryPolicy.exponentialBackoff(1.second, maxRetry = 3),
      retriable = RetryPolicy.defaultRetriable
    ))(client)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Cats Effect IO: pure effects, resource management
- ✅ IOApp: structuring application
- ✅ Concurrent programming: fibers, Ref, parMapN
- ✅ http4s server setup
- ✅ HTTP Routes: CRUD, path variables, JSON
- ✅ Middleware: logging, CORS, rate limiting, auth
- ✅ HTTP Client

---

*[← Part 25: Play Framework](part-25-play-framework.md) | [Part 27: Doobie Database →](part-27-doobie.md)*
