# Part 45: Logging และ Monitoring

## สารบัญ
1. [Logging Setup](#logging-setup)
2. [Structured Logging](#structured-logging)
3. [Cats Effect Logging](#cats-effect-logging)
4. [Metrics with Prometheus](#metrics)
5. [Health Checks](#health-checks)
6. [Distributed Tracing](#distributed-tracing)

---

## Logging Setup

### Dependencies

```scala
libraryDependencies ++= Seq(
  // Logging facade
  "org.typelevel" %% "log4cats-core"  % "2.6.0",
  "org.typelevel" %% "log4cats-slf4j" % "2.6.0",
  // SLF4J backend
  "ch.qos.logback" % "logback-classic" % "1.4.11",
  // Structured JSON logging
  "net.logstash.logback" % "logstash-logback-encoder" % "7.4"
)
```

### Logback Configuration

```xml
<!-- src/main/resources/logback.xml -->
<configuration>
  <!-- Console appender (development) -->
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeContext>true</includeContext>
      <includeCallerData>false</includeCallerData>
    </encoder>
  </appender>

  <!-- Rolling file appender (production) -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>logs/app.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
      <maxHistory>30</maxHistory>
      <totalSizeCap>5GB</totalSizeCap>
      <timeBasedFileNamingAndTriggeringPolicy
          class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
        <maxFileSize>100MB</maxFileSize>
      </timeBasedFileNamingAndTriggeringPolicy>
    </rollingPolicy>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>

  <root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="FILE"/>
  </root>

  <!-- Reduce noise from libraries -->
  <logger name="org.http4s" level="WARN"/>
  <logger name="io.grpc" level="WARN"/>
</configuration>
```

---

## Structured Logging

### Log with Context

```scala
import org.typelevel.log4cats.{Logger, SelfAwareStructuredLogger}
import org.typelevel.log4cats.slf4j.Slf4jLogger
import cats.effect.IO

// Basic logging
object LoggingExample:
  given logger: SelfAwareStructuredLogger[IO] = Slf4jLogger.getLogger[IO]

  def processRequest(userId: String, action: String): IO[Unit] =
    import logger.*
    for
      _ <- info(s"Processing request: userId=$userId, action=$action")
      _ <- debug(s"Starting processing")
      _ <- warn(s"Deprecated endpoint called")
      _ <- error(s"Something went wrong")
    yield ()

// Structured logging with context
def withContext(ctx: Map[String, String])(f: SelfAwareStructuredLogger[IO] => IO[Unit]): IO[Unit] =
  val ctxLogger = Slf4jLogger.getLoggerFromName[IO]("app").addContext(ctx)
  f(ctxLogger)

// Request-scoped logging
def handleRequest(requestId: String, userId: String)(
  handler: Logger[IO] => IO[String]
): IO[String] =
  val ctx = Map(
    "requestId" -> requestId,
    "userId"    -> userId,
    "service"   -> "user-service"
  )
  withContext(ctx) { log =>
    for
      _      <- log.info("Request started")
      result <- handler(log)
      _      <- log.info("Request completed")
    yield result
  }

// Custom log entries with MDC (Mapped Diagnostic Context)
import org.slf4j.MDC

def withMDC[A](fields: Map[String, String])(action: IO[A]): IO[A] =
  IO.delay(fields.foreach { case (k, v) => MDC.put(k, v) })
    .bracket(_ => action)(_ => IO.delay(fields.keys.foreach(MDC.remove)))
```

---

## Cats Effect Logging

### Log4Cats with IO

```scala
import org.typelevel.log4cats.*
import org.typelevel.log4cats.slf4j.Slf4jLogger
import cats.effect.{IO, Resource}

// Logger as dependency
trait AppLogger:
  def info(msg: String): IO[Unit]
  def warn(msg: String): IO[Unit]
  def error(msg: String, cause: Option[Throwable] = None): IO[Unit]
  def debug(msg: String): IO[Unit]

class AppLoggerImpl(logger: SelfAwareStructuredLogger[IO]) extends AppLogger:
  def info(msg: String)  = logger.info(msg)
  def warn(msg: String)  = logger.warn(msg)
  def debug(msg: String) = logger.debug(msg)
  def error(msg: String, cause: Option[Throwable] = None): IO[Unit] =
    cause.fold(logger.error(msg))(e => logger.error(e)(msg))

object AppLoggerImpl:
  def make: IO[AppLogger] =
    Slf4jLogger.create[IO].map(AppLoggerImpl(_))

// Structured request logging
case class RequestLog(
  method: String,
  path: String,
  statusCode: Int,
  durationMs: Long,
  userId: Option[String]
)

def logRequest(log: AppLogger, req: RequestLog): IO[Unit] =
  if req.statusCode >= 500 then
    log.error(s"[${req.method}] ${req.path} -> ${req.statusCode} (${req.durationMs}ms)")
  else if req.statusCode >= 400 then
    log.warn(s"[${req.method}] ${req.path} -> ${req.statusCode} (${req.durationMs}ms)")
  else
    log.info(s"[${req.method}] ${req.path} -> ${req.statusCode} (${req.durationMs}ms)")
```

---

## Metrics with Prometheus

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.http4s"    %% "http4s-prometheus-metrics" % "0.23.24",
  "io.prometheus"  % "simpleclient"               % "0.16.0",
  "io.prometheus"  % "simpleclient_hotspot"        % "0.16.0"
)
```

### Custom Metrics

```scala
import io.prometheus.client.*
import cats.effect.IO

object Metrics:
  // Counter: values only go up
  val requestCount = Counter.build()
    .name("http_requests_total")
    .help("Total HTTP requests")
    .labelNames("method", "path", "status")
    .register()

  // Gauge: can go up or down
  val activeConnections = Gauge.build()
    .name("active_connections")
    .help("Current active connections")
    .register()

  // Histogram: distribution of values
  val requestDuration = Histogram.build()
    .name("http_request_duration_seconds")
    .help("Request duration in seconds")
    .labelNames("method", "path")
    .buckets(0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0)
    .register()

  // Summary: similar to histogram, with quantiles
  val dbQueryTime = Summary.build()
    .name("db_query_duration_seconds")
    .help("Database query duration")
    .quantile(0.5, 0.05)   // 50th percentile
    .quantile(0.95, 0.01)  // 95th percentile
    .quantile(0.99, 0.001) // 99th percentile
    .register()

// Record metrics
def recordRequest(method: String, path: String, status: Int, duration: Double): IO[Unit] =
  IO {
    requestCount.labels(method, path, status.toString).inc()
    requestDuration.labels(method, path).observe(duration)
  }

def trackConnections[A](action: IO[A]): IO[A] =
  IO(activeConnections.inc())
    .bracket(_ => action)(_ => IO(activeConnections.dec()))

// Prometheus HTTP endpoint
import org.http4s.{HttpRoutes, Response, Status}
import io.prometheus.client.exporter.common.TextFormat
import java.io.StringWriter

val metricsRoute: HttpRoutes[IO] = HttpRoutes.of[IO] {
  case req if req.method.name == "GET" && req.uri.path.renderString == "/metrics" =>
    IO {
      val writer = new StringWriter()
      TextFormat.write004(writer, CollectorRegistry.defaultRegistry.metricFamilySamples())
      Response[IO](Status.Ok).withEntity(writer.toString)
    }
}
```

---

## Health Checks

### Liveness and Readiness

```scala
import org.http4s.{HttpRoutes, Response, Status}
import cats.effect.IO
import io.circe.generic.auto.*
import io.circe.syntax.*

// Health check types
case class HealthStatus(status: String, checks: Map[String, String])

// Health check for database
def dbHealthCheck(xa: doobie.Transactor[IO]): IO[Boolean] =
  import doobie.implicits.*
  sql"SELECT 1".query[Int].unique.transact(xa)
    .map(_ == 1)
    .handleError(_ => false)

// Health check for Redis
def redisHealthCheck(redis: dev.profunktor.redis4cats.RedisCommands[IO, String, String]): IO[Boolean] =
  redis.ping.map(_.toLowerCase == "pong").handleError(_ => false)

// Aggregate health checks
def healthRoutes(
  dbCheck: IO[Boolean],
  redisCheck: IO[Boolean]
): HttpRoutes[IO] =
  HttpRoutes.of[IO] {
    // Liveness: is the app alive?
    case req if req.uri.path.renderString == "/health/live" =>
      IO.pure(Response[IO](Status.Ok).withEntity("""{"status":"ok"}"""))

    // Readiness: is the app ready to serve traffic?
    case req if req.uri.path.renderString == "/health/ready" =>
      for
        dbOk    <- dbCheck
        redisOk <- redisCheck
        checks = Map(
          "database" -> (if dbOk then "healthy" else "unhealthy"),
          "redis"    -> (if redisOk then "healthy" else "unhealthy")
        )
        allOk = dbOk && redisOk
        status = HealthStatus(if allOk then "healthy" else "degraded", checks)
        resp = Response[IO](if allOk then Status.Ok else Status.ServiceUnavailable)
          .withEntity(status.asJson.noSpaces)
      yield resp
  }
```

---

## Distributed Tracing

### OpenTelemetry Integration

```scala
import io.opentelemetry.api.GlobalOpenTelemetry
import io.opentelemetry.api.trace.*
import io.opentelemetry.context.Context
import cats.effect.IO

object Tracing:
  private val tracer = GlobalOpenTelemetry.getTracer("scala-app", "1.0.0")

  // Trace a single operation
  def trace[A](spanName: String, attrs: Map[String, String] = Map.empty)(
    operation: IO[A]
  ): IO[A] =
    val builder = tracer.spanBuilder(spanName)
    attrs.foreach { case (k, v) => builder.setAttribute(k, v) }
    val span = builder.startSpan()
    operation
      .tapError { e =>
        IO {
          span.recordException(e)
          span.setStatus(StatusCode.ERROR, e.getMessage)
        }
      }
      .guarantee(IO(span.end()))

  // Propagate trace context across service calls
  def getTraceContext: IO[Map[String, String]] =
    IO {
      val ctx = Context.current()
      val span = Span.fromContext(ctx)
      val spanCtx = span.getSpanContext
      Map(
        "trace-id"  -> spanCtx.getTraceId,
        "span-id"   -> spanCtx.getSpanId
      )
    }

// Usage in service
class OrderService:
  def processOrder(orderId: String): IO[String] =
    Tracing.trace("process-order", Map("order.id" -> orderId)) {
      for
        _      <- Tracing.trace("validate-order")(validateOrder(orderId))
        _      <- Tracing.trace("reserve-inventory")(reserveInventory(orderId))
        result <- Tracing.trace("charge-payment")(chargePayment(orderId))
      yield result
    }

  private def validateOrder(id: String): IO[Unit] = IO.unit
  private def reserveInventory(id: String): IO[Unit] = IO.unit
  private def chargePayment(id: String): IO[String] = IO.pure("success")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Logback configuration กับ JSON structured logging
- ✅ Log4Cats: pure functional logging
- ✅ Structured request/response logging
- ✅ Prometheus metrics: counters, gauges, histograms
- ✅ Health checks: liveness, readiness endpoints
- ✅ Distributed tracing กับ OpenTelemetry

---

*[← Part 44: Security](part-44-security.md) | [Part 46: Docker and Deployment →](part-46-docker.md)*
