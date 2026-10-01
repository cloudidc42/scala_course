# Part 40: Microservices กับ Scala

## สารบัญ
1. [Microservices Architecture](#microservices-architecture)
2. [Service Communication](#service-communication)
3. [Service Discovery](#service-discovery)
4. [Resilience Patterns](#resilience-patterns)
5. [Distributed Tracing](#distributed-tracing)
6. [Complete Example](#complete-example)

---

## Microservices Architecture

### แนวคิด

```
Microservices:
- Small, independent services
- Each owns its own data
- Communicate via network
- Deploy independently

Trade-offs vs Monolith:
Pros: Scale independently, tech diversity, resilience
Cons: Network latency, distributed transactions, complexity

Key patterns:
- Service per bounded context (DDD)
- API Gateway
- Event-driven communication
- CQRS + Event Sourcing
- Saga pattern for distributed transactions
- Circuit Breaker for resilience
```

### Project Structure

```
scala-microservices/
├── build.sbt
├── project/
│   └── plugins.sbt
├── common/
│   └── src/main/scala/common/  # shared models, protocols
├── user-service/
│   └── src/main/scala/user/
│       ├── domain/
│       ├── application/
│       ├── infrastructure/
│       └── Main.scala
├── order-service/
│   └── src/main/scala/order/
├── notification-service/
│   └── src/main/scala/notification/
└── api-gateway/
    └── src/main/scala/gateway/
```

```scala
// build.sbt for multi-module
lazy val commonSettings = Seq(
  scalaVersion := "3.3.1",
  organization := "com.example"
)

lazy val common = (project in file("common"))
  .settings(commonSettings)

lazy val userService = (project in file("user-service"))
  .settings(commonSettings)
  .dependsOn(common)

lazy val orderService = (project in file("order-service"))
  .settings(commonSettings)
  .dependsOn(common)

lazy val apiGateway = (project in file("api-gateway"))
  .settings(commonSettings)
  .dependsOn(common)

lazy val root = (project in file("."))
  .aggregate(common, userService, orderService, apiGateway)
```

---

## Service Communication

### HTTP Client with Retry

```scala
import cats.effect.IO
import org.http4s.client.Client
import org.http4s.client.ember.EmberClientBuilder
import org.http4s.{Uri, Request, Method}
import cats.effect.Resource
import scala.concurrent.duration.*
import io.circe.*
import io.circe.generic.auto.*

// Typed HTTP client wrapper
class HttpServiceClient(client: Client[IO], baseUrl: String):

  def get[A: Decoder](path: String): IO[A] =
    val uri = Uri.unsafeFromString(s"$baseUrl$path")
    client.expect[A](uri)(using org.http4s.circe.jsonOf[IO, A])

  def post[In: Encoder, Out: Decoder](path: String, body: In): IO[Out] =
    val uri = Uri.unsafeFromString(s"$baseUrl$path")
    val req = Request[IO](Method.POST, uri)
      .withEntity(body.asJson)(using org.http4s.circe.jsonEncoderOf)
    client.expect[Out](req)(using org.http4s.circe.jsonOf[IO, Out])

// Service clients
case class UserDto(id: Long, name: String, email: String)
case class OrderDto(id: Long, userId: Long, items: List[String], total: Double)

class UserServiceClient(client: HttpServiceClient):
  def getUser(id: Long): IO[Option[UserDto]] =
    client.get[Option[UserDto]](s"/users/$id")
      .handleErrorWith(_ => IO.pure(None))

class OrderServiceClient(client: HttpServiceClient):
  def getUserOrders(userId: Long): IO[List[OrderDto]] =
    client.get[List[OrderDto]](s"/orders?userId=$userId")
      .handleErrorWith(_ => IO.pure(Nil))
```

---

## Resilience Patterns

### Circuit Breaker

```scala
import cats.effect.{IO, Ref}
import scala.concurrent.duration.*

enum CircuitState:
  case Closed, Open, HalfOpen

case class CircuitBreakerConfig(
  maxFailures:   Int,
  resetTimeout:  FiniteDuration,
  halfOpenCalls: Int = 1
)

class CircuitBreaker(
  config: CircuitBreakerConfig,
  state:  Ref[IO, CircuitState],
  fails:  Ref[IO, Int],
  openAt: Ref[IO, Option[Long]]
):
  def protect[A](action: IO[A]): IO[A] =
    state.get.flatMap {
      case CircuitState.Open =>
        openAt.get.flatMap { ts =>
          val elapsed = System.currentTimeMillis() - ts.getOrElse(0L)
          if elapsed > config.resetTimeout.toMillis then
            state.set(CircuitState.HalfOpen) *> attempt(action)
          else
            IO.raiseError(new RuntimeException("Circuit is OPEN"))
        }
      case CircuitState.Closed | CircuitState.HalfOpen =>
        attempt(action)
    }

  private def attempt[A](action: IO[A]): IO[A] =
    action.attempt.flatMap {
      case Right(result) =>
        state.set(CircuitState.Closed) *>
        fails.set(0) *>
        IO.pure(result)
      case Left(err) =>
        fails.updateAndGet(_ + 1).flatMap { failures =>
          if failures >= config.maxFailures then
            state.set(CircuitState.Open) *>
            openAt.set(Some(System.currentTimeMillis())) *>
            IO.raiseError(err)
          else
            IO.raiseError(err)
        }
    }

object CircuitBreaker:
  def make(config: CircuitBreakerConfig): IO[CircuitBreaker] =
    for
      state  <- Ref.of[IO, CircuitState](CircuitState.Closed)
      fails  <- Ref.of[IO, Int](0)
      openAt <- Ref.of[IO, Option[Long]](None)
    yield CircuitBreaker(config, state, fails, openAt)
```

### Retry with Backoff

```scala
import cats.effect.IO
import scala.concurrent.duration.*

def retryWithBackoff[A](
  action:     IO[A],
  maxRetries: Int,
  baseDelay:  FiniteDuration = 100.millis,
  maxDelay:   FiniteDuration = 30.seconds
): IO[A] =
  def attempt(retriesLeft: Int, delay: FiniteDuration): IO[A] =
    action.handleErrorWith { err =>
      if retriesLeft <= 0 then IO.raiseError(err)
      else
        IO.sleep(delay) *>
        attempt(
          retriesLeft - 1,
          (delay * 2).min(maxDelay)  // exponential backoff
        )
    }

  attempt(maxRetries, baseDelay)

// With jitter to avoid thundering herd
import scala.util.Random

def retryWithJitter[A](action: IO[A], maxRetries: Int): IO[A] =
  def attempt(n: Int): IO[A] =
    action.handleErrorWith { err =>
      if n <= 0 then IO.raiseError(err)
      else
        val base  = math.pow(2, maxRetries - n).toInt * 100
        val jitter = Random.nextInt(base / 2)
        IO.sleep((base + jitter).millis) *> attempt(n - 1)
    }
  attempt(maxRetries)
```

---

## Distributed Tracing

### OpenTelemetry

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.opentelemetry" % "opentelemetry-api"  % "1.31.0",
  "io.opentelemetry" % "opentelemetry-sdk"  % "1.31.0",
  "io.opentelemetry" % "opentelemetry-exporter-otlp" % "1.31.0"
)

import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.trace.*
import cats.effect.IO

// Tracing helper
object Tracing:
  private val tracer = GlobalOpenTelemetry.getTracer("scala-service")

  def span[A](name: String)(action: Span => IO[A]): IO[A] =
    IO.defer {
      val span = tracer.spanBuilder(name).startSpan()
      action(span)
        .guarantee(IO(span.end()))
        .handleErrorWith { e =>
          IO(span.recordException(e)) *>
          IO(span.setStatus(StatusCode.ERROR)) *>
          IO.raiseError(e)
        }
    }

  def withAttribute(span: Span, key: String, value: String): IO[Unit] =
    IO(span.setAttribute(key, value))

// Usage
def handleRequest(userId: Long): IO[String] =
  Tracing.span("handle-request") { span =>
    for
      _ <- Tracing.withAttribute(span, "user.id", userId.toString)
      user <- Tracing.span("fetch-user")(_ => fetchUser(userId))
      orders <- Tracing.span("fetch-orders")(_ => fetchOrders(userId))
      _ <- Tracing.withAttribute(span, "orders.count", orders.size.toString)
    yield s"User ${user} has ${orders.size} orders"
  }

def fetchUser(id: Long): IO[String] = IO.pure(s"User-$id")
def fetchOrders(userId: Long): IO[List[String]] = IO.pure(List("order1", "order2"))
```

---

## Complete Example

### Order Management Microservice

```scala
import cats.effect.{IO, IOApp, Resource, Ref}
import org.http4s.ember.server.EmberServerBuilder
import org.http4s.server.Router
import com.comcast.ip4s.*
import io.circe.generic.auto.*
import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.server.http4s.Http4sServerInterpreter

// Domain
case class Order(
  id: String,
  userId: Long,
  items: List[OrderItem],
  status: String,
  total: Double
)
case class OrderItem(productId: Long, quantity: Int, price: Double)
case class CreateOrderRequest(userId: Long, items: List[OrderItem])
case class ApiError(code: String, message: String)

// Repository
class OrderRepository(store: Ref[IO, Map[String, Order]]):
  def findAll: IO[List[Order]] = store.get.map(_.values.toList)

  def findById(id: String): IO[Option[Order]] = store.get.map(_.get(id))

  def findByUser(userId: Long): IO[List[Order]] =
    store.get.map(_.values.filter(_.userId == userId).toList)

  def create(req: CreateOrderRequest): IO[Order] =
    val id    = java.util.UUID.randomUUID().toString
    val total = req.items.map(i => i.quantity * i.price).sum
    val order = Order(id, req.userId, req.items, "pending", total)
    store.update(_ + (id -> order)).as(order)

  def updateStatus(id: String, status: String): IO[Option[Order]] =
    store.modify { m =>
      m.get(id) match
        case Some(order) =>
          val updated = order.copy(status = status)
          (m + (id -> updated), Some(updated))
        case None => (m, None)
    }

// Service (business logic)
class OrderService(repo: OrderRepository, userClient: UserServiceClient):
  def createOrder(req: CreateOrderRequest): IO[Either[ApiError, Order]] =
    for
      // Validate user exists
      userExists <- userClient.getUser(req.userId).map(_.isDefined)
      result <- if !userExists then
        IO.pure(Left(ApiError("user_not_found", s"User ${req.userId} not found")))
      else if req.items.isEmpty then
        IO.pure(Left(ApiError("empty_order", "Order must have at least one item")))
      else
        repo.create(req).map(Right(_))
    yield result

  def getOrder(id: String): IO[Either[ApiError, Order]] =
    repo.findById(id).map {
      case Some(o) => Right(o)
      case None    => Left(ApiError("not_found", s"Order $id not found"))
    }

  def getUserOrders(userId: Long): IO[List[Order]] =
    repo.findByUser(userId)

  def cancelOrder(id: String): IO[Either[ApiError, Order]] =
    repo.findById(id).flatMap {
      case None => IO.pure(Left(ApiError("not_found", s"Order $id not found")))
      case Some(o) if o.status == "completed" =>
        IO.pure(Left(ApiError("cannot_cancel", "Completed orders cannot be cancelled")))
      case _ =>
        repo.updateStatus(id, "cancelled").map {
          case Some(o) => Right(o)
          case None    => Left(ApiError("not_found", s"Order $id not found"))
        }
    }

// HTTP Routes using Tapir
object OrderRoutes:
  val base = endpoint.in("api" / "orders").errorOut(jsonBody[ApiError])

  val list        = base.get.out(jsonBody[List[Order]])
  val getById     = base.get.in(path[String]("id")).out(jsonBody[Order])
  val create      = base.post.in(jsonBody[CreateOrderRequest]).out(jsonBody[Order])
  val getUserOrds = base.get.in("user" / path[Long]("userId")).out(jsonBody[List[Order]])
  val cancel      = base.delete.in(path[String]("id")).out(jsonBody[Order])

  def routes(svc: OrderService) =
    val interp = Http4sServerInterpreter[IO]()
    List(
      interp.toRoutes(list.serverLogic(_ => svc.getUserOrders(0L).map(Right(_)))),
      interp.toRoutes(getById.serverLogic(svc.getOrder)),
      interp.toRoutes(create.serverLogic(svc.createOrder)),
      interp.toRoutes(getUserOrds.serverLogic(id => svc.getUserOrders(id).map(Right(_)))),
      interp.toRoutes(cancel.serverLogic(svc.cancelOrder))
    ).foldLeft(org.http4s.HttpRoutes.empty[IO])(_ <+> _)

// Dummy user client
class UserServiceClient(client: org.http4s.client.Client[IO]):
  case class Usr(id: Long, name: String)
  def getUser(id: Long): IO[Option[Usr]] = IO.pure(Some(Usr(id, "User")))

// Main app
object OrderServiceApp extends IOApp.Simple:
  def run: IO[Unit] =
    for
      store  <- Ref.of[IO, Map[String, Order]](Map.empty)
      repo    = OrderRepository(store)
      _ <- EmberClientBuilder.default[IO].build.use { client =>
        val userClient = UserServiceClient(client)
        val svc        = OrderService(repo, userClient)
        val routes     = OrderRoutes.routes(svc)
        EmberServerBuilder.default[IO]
          .withHost(ipv4"0.0.0.0")
          .withPort(port"8081")
          .withHttpApp(Router("/" -> routes).orNotFound)
          .build
          .useForever
      }
    yield ()
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Microservices architecture และ multi-module SBT
- ✅ HTTP service client สำหรับ inter-service communication
- ✅ Circuit Breaker pattern
- ✅ Retry with exponential backoff + jitter
- ✅ Distributed tracing กับ OpenTelemetry
- ✅ Complete Order Service microservice example

---

*[← Part 39: gRPC](part-39-grpc.md) | [Part 41: CQRS and Event Sourcing →](part-41-cqrs-event-sourcing.md)*
