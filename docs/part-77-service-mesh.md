# ส่วนที่ 77: Service Mesh และ Observability

## สารบัญ

1. [Service Mesh Concepts](#service-mesh-concepts)
2. [mTLS ระหว่าง Services](#mtls-ระหว่าง-services)
3. [Traffic Management](#traffic-management)
4. [Circuit Breaking ที่ Mesh Level](#circuit-breaking-ที่-mesh-level)
5. [Distributed Tracing: Jaeger และ Zipkin](#distributed-tracing-jaeger-และ-zipkin)
6. [Metrics Collection: Prometheus และ Grafana](#metrics-collection-prometheus-และ-grafana)
7. [Log Aggregation: ELK Stack](#log-aggregation-elk-stack)
8. [Complete Observability Setup](#complete-observability-setup)
9. [สรุป](#สรุป)

---

## Service Mesh Concepts

Service Mesh คือ infrastructure layer สำหรับจัดการ service-to-service communication

### ทำไมต้องใช้ Service Mesh?

```
ปัญหาใน Microservices:
- การสื่อสารระหว่าง services ซับซ้อน
- การรักษาความปลอดภัยระหว่าง services
- Visibility ของ traffic ต่ำ
- Load balancing และ routing ยาก

Service Mesh แก้ปัญหาเหล่านี้ด้วย:
- Sidecar proxy (Envoy)
- Control plane (Istiod/Linkerd)
- ตรวจสอบ traffic ทั้งหมดผ่าน proxy

Istio Architecture:
┌─────────────────────────────────────┐
│           Control Plane              │
│  ┌─────────┐  ┌─────────────────┐   │
│  │  Pilot  │  │    Citadel      │   │
│  │(routing)│  │(certificate mgr)│   │
│  └─────────┘  └─────────────────┘   │
└─────────────────────────────────────┘
           ↕ xDS API
┌──────────────────────────────────────┐
│              Data Plane               │
│  ┌────────────────────────────────┐  │
│  │   Service A Pod                │  │
│  │  ┌──────────┐ ┌────────────┐  │  │
│  │  │  App     │ │  Envoy     │  │  │
│  │  │Container │ │  Sidecar   │  │  │
│  │  └──────────┘ └────────────┘  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Kubernetes Deployment กับ Istio

```yaml
# k8s/product-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    app: product-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  template:
    metadata:
      labels:
        app: product-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: product-service
        image: myregistry/product-service:1.0.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: ecommerce
spec:
  selector:
    app: product-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: metrics
    port: 9090
    targetPort: 9090
```

### Health Check Endpoints ใน Scala

```scala
// src/main/scala/health/HealthCheck.scala
package health

import cats.effect.IO
import cats.syntax.all.*
import org.http4s.*
import org.http4s.dsl.io.*
import io.circe.generic.auto.*
import org.http4s.circe.*

case class HealthStatus(
  status: String,
  checks: Map[String, CheckResult]
)

case class CheckResult(
  status: String,
  message: Option[String] = None,
  latencyMs: Option[Long] = None
)

trait HealthChecker:
  def name: String
  def check: IO[CheckResult]

class DatabaseHealthChecker(xa: doobie.Transactor[IO]) extends HealthChecker:
  val name = "database"
  
  def check: IO[CheckResult] =
    val start = System.currentTimeMillis()
    import doobie.implicits.*
    doobie.Query0[Int]("SELECT 1").option.transact(xa)
      .map { _ =>
        CheckResult(
          status = "UP",
          latencyMs = Some(System.currentTimeMillis() - start)
        )
      }
      .handleError { e =>
        CheckResult(status = "DOWN", message = Some(e.getMessage))
      }

class RedisHealthChecker(redisClient: RedisClient[IO]) extends HealthChecker:
  val name = "redis"
  
  def check: IO[CheckResult] =
    redisClient.ping
      .map(_ => CheckResult(status = "UP"))
      .handleError(e => CheckResult(status = "DOWN", message = Some(e.getMessage)))

class HealthRoutes(checkers: List[HealthChecker]):
  
  val routes = HttpRoutes.of[IO] {
    case GET -> Root / "health" / "live" =>
      Ok("""{"status":"UP"}""")
    
    case GET -> Root / "health" / "ready" =>
      for
        results  <- checkers.traverse(c => c.check.map(c.name -> _))
        allUp     = results.forall(_._2.status == "UP")
        status    = HealthStatus(
                      if allUp then "UP" else "DOWN",
                      results.toMap
                    )
        response <- if allUp then Ok(status) else ServiceUnavailable(status)
      yield response
    
    case GET -> Root / "health" / "startup" =>
      Ok("""{"status":"STARTED"}""")
  }
```

---

## mTLS ระหว่าง Services

mTLS (Mutual TLS) ทำให้ทั้ง client และ server ต้องแสดงตัวตน

```yaml
# istio/peer-authentication.yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT  # บังคับใช้ mTLS ทุก connection
---
# เฉพาะบาง service ที่ต้องการ mode แตกต่าง
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: legacy-permissive
  namespace: ecommerce
spec:
  selector:
    matchLabels:
      app: legacy-service
  mtls:
    mode: PERMISSIVE  # รองรับทั้ง mTLS และ plain text
```

```yaml
# istio/authorization-policy.yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-policy
  namespace: ecommerce
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - "cluster.local/ns/ecommerce/sa/order-service"
        - "cluster.local/ns/ecommerce/sa/api-gateway"
    to:
    - operation:
        methods: ["GET", "POST", "PUT", "DELETE"]
        paths: ["/api/v1/products/*"]
  - from:
    - source:
        principals:
        - "cluster.local/ns/ecommerce/sa/inventory-service"
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/v1/products/*/stock"]
```

### Service Account Configuration

```yaml
# k8s/service-accounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: product-service
  namespace: ecommerce
  annotations:
    # สำหรับ AWS EKS
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/product-service-role
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: ecommerce
```

---

## Traffic Management

### Virtual Service และ Destination Rule

```yaml
# istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service
  namespace: ecommerce
spec:
  hosts:
  - product-service
  http:
  # Canary deployment: 10% ไป v2, 90% ไป v1
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: product-service
        subset: v2
  - route:
    - destination:
        host: product-service
        subset: v1
      weight: 90
    - destination:
        host: product-service
        subset: v2
      weight: 10
    # Fault injection สำหรับ testing
    fault:
      delay:
        percentage:
          value: 5
        fixedDelay: 100ms
      abort:
        percentage:
          value: 1
        httpStatus: 503
    # Timeout
    timeout: 5s
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service
  namespace: ecommerce
spec:
  host: product-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: LEAST_CONN
```

### Circuit Breaking ใน Scala Code

```scala
// src/main/scala/resilience/CircuitBreaker.scala
package resilience

import cats.effect.*
import cats.effect.std.AtomicCell
import scala.concurrent.duration.*
import java.time.Instant

enum CircuitState:
  case Closed    // ปกติ
  case Open      // ปิดชั่วคราว (ไม่ให้ request ผ่าน)
  case HalfOpen  // ทดสอบว่า service กลับมาแล้วหรือยัง

case class CircuitBreakerConfig(
  failureThreshold: Int = 5,
  successThreshold: Int = 2,
  timeout: FiniteDuration = 30.seconds,
  halfOpenMaxCalls: Int = 3
)

case class CircuitBreakerState(
  state: CircuitState,
  failureCount: Int,
  successCount: Int,
  lastFailureTime: Option[Instant],
  halfOpenCalls: Int
)

class CircuitBreaker[F[_]](
  name: String,
  config: CircuitBreakerConfig,
  stateRef: AtomicCell[F, CircuitBreakerState]
)(using F: Temporal[F]):
  
  def run[A](action: F[A]): F[A] =
    for
      state   <- stateRef.get
      result  <- state.state match
                   case CircuitState.Open =>
                     checkTimeout(state).flatMap {
                       case true  => transitionToHalfOpen >> runHalfOpen(action)
                       case false => F.raiseError(CircuitOpenException(name))
                     }
                   case CircuitState.HalfOpen =>
                     runHalfOpen(action)
                   case CircuitState.Closed =>
                     runClosed(action)
    yield result
  
  private def runClosed[A](action: F[A]): F[A] =
    action.attempt.flatMap {
      case Right(result) =>
        stateRef.update(s => s.copy(failureCount = 0)) >> F.pure(result)
      case Left(error) =>
        stateRef.update { s =>
          val newCount = s.failureCount + 1
          if newCount >= config.failureThreshold then
            s.copy(
              state = CircuitState.Open,
              failureCount = newCount,
              lastFailureTime = Some(Instant.now())
            )
          else
            s.copy(failureCount = newCount)
        } >> F.raiseError(error)
    }
  
  private def runHalfOpen[A](action: F[A]): F[A] =
    action.attempt.flatMap {
      case Right(result) =>
        stateRef.update { s =>
          val newSuccess = s.successCount + 1
          if newSuccess >= config.successThreshold then
            s.copy(state = CircuitState.Closed, successCount = 0, failureCount = 0)
          else
            s.copy(successCount = newSuccess)
        } >> F.pure(result)
      case Left(error) =>
        stateRef.update(s =>
          s.copy(state = CircuitState.Open, lastFailureTime = Some(Instant.now()))
        ) >> F.raiseError(error)
    }
  
  private def checkTimeout(state: CircuitBreakerState): F[Boolean] =
    state.lastFailureTime.fold(F.pure(false)) { lastFailure =>
      F.realTimeInstant.map { now =>
        now.toEpochMilli - lastFailure.toEpochMilli >= config.timeout.toMillis
      }
    }
  
  private def transitionToHalfOpen: F[Unit] =
    stateRef.update(s => s.copy(state = CircuitState.HalfOpen, successCount = 0))

case class CircuitOpenException(circuitName: String)
    extends Exception(s"Circuit breaker '$circuitName' is open")

object CircuitBreaker:
  def make[F[_]: Temporal](
    name: String,
    config: CircuitBreakerConfig = CircuitBreakerConfig()
  ): F[CircuitBreaker[F]] =
    AtomicCell[F].of(CircuitBreakerState(
      state = CircuitState.Closed,
      failureCount = 0,
      successCount = 0,
      lastFailureTime = None,
      halfOpenCalls = 0
    )).map(ref => CircuitBreaker(name, config, ref))
```

---

## Distributed Tracing: Jaeger และ Zipkin

### OpenTelemetry Setup

```scala
// build.sbt - เพิ่ม dependencies
libraryDependencies ++= Seq(
  "io.opentelemetry" % "opentelemetry-api" % "1.31.0",
  "io.opentelemetry" % "opentelemetry-sdk" % "1.31.0",
  "io.opentelemetry" % "opentelemetry-exporter-jaeger" % "1.31.0",
  "io.opentelemetry" % "opentelemetry-exporter-zipkin" % "1.31.0",
  "io.opentelemetry.instrumentation" % "opentelemetry-http4s-server-0.23" % "1.31.0-alpha"
)
```

```scala
// src/main/scala/tracing/Tracing.scala
package tracing

import cats.effect.*
import io.opentelemetry.api.OpenTelemetry
import io.opentelemetry.api.trace.{Tracer, SpanKind, StatusCode}
import io.opentelemetry.context.Context

// Tracing abstraction
trait Trace[F[_]]:
  def span[A](name: String)(fa: F[A]): F[A]
  def spanWithAttributes[A](
    name: String,
    attributes: Map[String, String]
  )(fa: F[A]): F[A]
  def addEvent(name: String, attributes: Map[String, String] = Map.empty): F[Unit]
  def recordException(e: Throwable): F[Unit]
  def setAttribute(key: String, value: String): F[Unit]

class OpenTelemetryTrace[F[_]](
  tracer: Tracer
)(using F: Sync[F]) extends Trace[F]:
  
  def span[A](name: String)(fa: F[A]): F[A] =
    F.bracket(
      F.delay(tracer.spanBuilder(name).startSpan())
    )(span =>
      F.delay(span.makeCurrent()).bracket(_ => fa)(scope =>
        F.delay(scope.close())
      )
    )(span =>
      F.delay(span.end())
    )
  
  def spanWithAttributes[A](name: String, attrs: Map[String, String])(fa: F[A]): F[A] =
    F.bracket(
      F.delay {
        val builder = tracer.spanBuilder(name)
        attrs.foreach { (k, v) =>
          builder.setAttribute(k, v)
        }
        builder.startSpan()
      }
    )(span =>
      F.delay(span.makeCurrent()).bracket(_ => fa.handleErrorWith { e =>
        F.delay(span.recordException(e)) >> F.delay(span.setStatus(StatusCode.ERROR)) >> F.raiseError(e)
      })(scope => F.delay(scope.close()))
    )(span => F.delay(span.end()))
  
  def addEvent(name: String, attributes: Map[String, String]): F[Unit] =
    F.delay {
      val span = io.opentelemetry.api.trace.Span.current()
      span.addEvent(name)
    }
  
  def recordException(e: Throwable): F[Unit] =
    F.delay {
      io.opentelemetry.api.trace.Span.current().recordException(e)
    }
  
  def setAttribute(key: String, value: String): F[Unit] =
    F.delay {
      io.opentelemetry.api.trace.Span.current().setAttribute(key, value)
    }

// Tracing middleware for HTTP4s
class TracingMiddleware(trace: Trace[IO]):
  
  def apply(routes: org.http4s.HttpRoutes[IO]): org.http4s.HttpRoutes[IO] =
    org.http4s.HttpRoutes { req =>
      val spanName = s"${req.method.name} ${req.pathInfo}"
      val attributes = Map(
        "http.method" -> req.method.name,
        "http.url" -> req.uri.renderString,
        "http.user_agent" -> req.headers.get[org.http4s.headers.`User-Agent`].map(_.toString).getOrElse("unknown")
      )
      
      cats.data.OptionT(
        trace.spanWithAttributes(spanName, attributes) {
          routes(req).value.flatMap {
            case Some(response) =>
              trace.setAttribute("http.status_code", response.status.code.toString)
                .as(Some(response))
            case None => IO.pure(None)
          }
        }
      )
    }

// Jaeger exporter setup
object JaegerSetup:
  def configure(
    serviceName: String,
    jaegerEndpoint: String = "http://jaeger:14268/api/traces"
  ): IO[Tracer] =
    IO {
      val exporter = io.opentelemetry.exporter.jaeger.JaegerGrpcSpanExporter.builder()
        .setEndpoint(jaegerEndpoint)
        .build()
      
      val provider = io.opentelemetry.sdk.trace.SdkTracerProvider.builder()
        .addSpanProcessor(
          io.opentelemetry.sdk.trace.export.BatchSpanProcessor.builder(exporter).build()
        )
        .build()
      
      io.opentelemetry.api.OpenTelemetry.noop()
        .getTracer(serviceName)
    }
```

---

## Metrics Collection: Prometheus และ Grafana

### Prometheus Metrics

```scala
// src/main/scala/metrics/PrometheusMetrics.scala
package metrics

import cats.effect.*
import io.prometheus.client.*
import io.prometheus.client.exporter.HTTPServer
import io.prometheus.client.hotspot.DefaultExports

// Metrics definitions
object ServiceMetrics:
  
  // HTTP metrics
  val httpRequestsTotal: Counter = Counter.build()
    .name("http_requests_total")
    .help("Total number of HTTP requests")
    .labelNames("method", "path", "status")
    .register()
  
  val httpRequestDuration: Histogram = Histogram.build()
    .name("http_request_duration_seconds")
    .help("HTTP request duration in seconds")
    .labelNames("method", "path")
    .buckets(0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0)
    .register()
  
  val httpRequestsInFlight: Gauge = Gauge.build()
    .name("http_requests_in_flight")
    .help("Number of HTTP requests currently being processed")
    .register()
  
  // Business metrics
  val ordersCreatedTotal: Counter = Counter.build()
    .name("orders_created_total")
    .help("Total number of orders created")
    .labelNames("status")
    .register()
  
  val orderValue: Histogram = Histogram.build()
    .name("order_value_baht")
    .help("Order value distribution in Baht")
    .buckets(100, 500, 1000, 5000, 10000, 50000)
    .register()
  
  val productStockLevel: Gauge = Gauge.build()
    .name("product_stock_level")
    .help("Current stock level per product")
    .labelNames("product_id", "product_name")
    .register()
  
  // Database metrics
  val dbQueryDuration: Histogram = Histogram.build()
    .name("db_query_duration_seconds")
    .help("Database query duration")
    .labelNames("operation", "table")
    .register()
  
  val dbConnectionPoolSize: Gauge = Gauge.build()
    .name("db_connection_pool_size")
    .help("Database connection pool size")
    .labelNames("state")  // active, idle, waiting
    .register()
  
  // Initialize JVM metrics
  def initializeJvmMetrics(): Unit =
    DefaultExports.initialize()

// Prometheus metrics middleware
class MetricsMiddleware:
  
  def apply(routes: org.http4s.HttpRoutes[IO]): org.http4s.HttpRoutes[IO] =
    org.http4s.HttpRoutes { req =>
      val path = req.pathInfo.renderString
      val method = req.method.name
      val timer = ServiceMetrics.httpRequestDuration.labels(method, path).startTimer()
      
      ServiceMetrics.httpRequestsInFlight.inc()
      
      cats.data.OptionT(
        routes(req).value.flatMap { maybeResponse =>
          val status = maybeResponse.map(_.status.code.toString).getOrElse("404")
          IO.delay {
            ServiceMetrics.httpRequestsTotal.labels(method, path, status).inc()
            ServiceMetrics.httpRequestsInFlight.dec()
            timer.observeDuration()
          } >> IO.pure(maybeResponse)
        }.handleErrorWith { e =>
          IO.delay {
            ServiceMetrics.httpRequestsTotal.labels(method, path, "500").inc()
            ServiceMetrics.httpRequestsInFlight.dec()
            timer.observeDuration()
          } >> IO.raiseError(e)
        }
      )
    }

// Metrics endpoint for Prometheus scraping
class MetricsRoutes:
  import org.http4s.*
  import org.http4s.dsl.io.*
  
  val routes = org.http4s.HttpRoutes.of[IO] {
    case GET -> Root / "metrics" =>
      IO.delay {
        val writer = new java.io.StringWriter()
        io.prometheus.client.exporter.common.TextFormat.write004(
          writer,
          CollectorRegistry.defaultRegistry.metricFamilySamples()
        )
        writer.toString
      }.flatMap(metrics =>
        Ok(metrics, org.http4s.Header.Raw(
          org.typelevel.ci.CIString("Content-Type"),
          io.prometheus.client.exporter.common.TextFormat.CONTENT_TYPE_004
        ))
      )
  }
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Scala Microservice Dashboard",
    "panels": [
      {
        "title": "HTTP Requests Per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{path}} {{status}}"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "P99 {{method}} {{path}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~'5..'}[5m]) / rate(http_requests_total[5m]) * 100",
            "legendFormat": "Error Rate %"
          }
        ]
      },
      {
        "title": "JVM Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "jvm_memory_bytes_used / jvm_memory_bytes_max * 100",
            "legendFormat": "{{area}} Memory %"
          }
        ]
      }
    ]
  }
}
```

---

## Log Aggregation: ELK Stack

### Structured Logging

```scala
// src/main/scala/logging/StructuredLogging.scala
package logging

import cats.effect.IO
import org.typelevel.log4cats.Logger
import org.typelevel.log4cats.slf4j.Slf4jLogger
import io.circe.Json
import io.circe.syntax.*
import java.time.Instant

// Structured log context
case class LogContext(
  requestId: String,
  userId: Option[String] = None,
  service: String = "product-service",
  version: String = "1.0.0",
  environment: String = "production",
  extra: Map[String, Json] = Map.empty
)

object LogContext:
  given Conversion[LogContext, Map[String, String]] = ctx =>
    Map(
      "requestId" -> ctx.requestId,
      "service" -> ctx.service,
      "version" -> ctx.version,
      "environment" -> ctx.environment
    ) ++ ctx.userId.map("userId" -> _).toMap

// Structured logger
class StructuredLogger(underlying: Logger[IO], ctx: LogContext):
  
  private def withCtx(message: String): String =
    val contextJson = Map(
      "message" -> message.asJson,
      "requestId" -> ctx.requestId.asJson,
      "service" -> ctx.service.asJson,
      "timestamp" -> Instant.now().toString.asJson
    ) ++ ctx.userId.map("userId" -> _.asJson)
    Json.obj(contextJson.toSeq*).noSpaces
  
  def info(message: String): IO[Unit] =
    underlying.info(withCtx(message))
  
  def warn(message: String): IO[Unit] =
    underlying.warn(withCtx(message))
  
  def error(message: String, cause: Option[Throwable] = None): IO[Unit] =
    cause match
      case Some(t) => underlying.error(t)(withCtx(message))
      case None    => underlying.error(withCtx(message))
  
  def debug(message: String): IO[Unit] =
    underlying.debug(withCtx(message))

// Logback configuration (logback.xml)
// ใช้ logstash-logback-encoder สำหรับ JSON output
```

```xml
<!-- src/main/resources/logback.xml -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeContext>false</includeContext>
      <timeZone>UTC</timeZone>
      <customFields>{"service":"product-service","env":"production"}</customFields>
    </encoder>
  </appender>
  
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/app/service.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>/var/log/app/service.%d{yyyy-MM-dd}.log</fileNamePattern>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>
  
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
    <appender-ref ref="FILE"/>
  </root>
</configuration>
```

### ELK Stack Configuration

```yaml
# docker-compose.elk.yml
version: '3.8'
services:
  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: elastic/filebeat:8.11.0
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/log/app:/var/log/app:ro
    depends_on:
      - logstash

volumes:
  elasticsearch-data:
```

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/app/*.log
  json.keys_under_root: true
  json.add_error_key: true

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
  - add_cloud_metadata: ~
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      matchers:
      - logs_path:
          logs_path: "/var/log/containers/"

output.logstash:
  hosts: ["logstash:5044"]
```

---

## Complete Observability Setup

### Alerting Rules

```yaml
# prometheus/alert-rules.yml
groups:
- name: service-alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"

  - alert: HighLatency
    expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High P99 latency"
      description: "P99 latency is {{ $value }}s for {{ $labels.service }}"

  - alert: ServiceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Service is down"
      description: "{{ $labels.job }} is down"

  - alert: DatabaseConnectionPoolExhausted
    expr: db_connection_pool_size{state="waiting"} > 10
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Database connection pool nearly exhausted"
```

### Complete Observability Stack

```scala
// src/main/scala/observability/ObservabilitySetup.scala
package observability

import cats.effect.*

case class ObservabilityConfig(
  serviceName: String,
  jaegerEndpoint: String,
  prometheusPort: Int,
  logLevel: String
)

object ObservabilitySetup:
  
  def initialize(config: ObservabilityConfig): Resource[IO, Unit] =
    Resource.eval {
      for
        _ <- IO.delay(ServiceMetrics.initializeJvmMetrics())
        _ <- IO.println(s"Observability initialized for ${config.serviceName}")
      yield ()
    }
  
  def makeTracing(config: ObservabilityConfig): IO[Trace[IO]] =
    JaegerSetup.configure(config.serviceName, config.jaegerEndpoint)
      .map(tracer => OpenTelemetryTrace[IO](tracer))
  
  def makeMetrics: MetricsMiddleware = MetricsMiddleware()
  
  def makeLogging(requestId: String): StructuredLogger =
    val underlying = org.typelevel.log4cats.slf4j.Slf4jLogger.getLogger[IO]
    StructuredLogger(
      underlying,
      LogContext(requestId = requestId)
    )
```

---

## สรุป

Service Mesh และ Observability เป็นส่วนสำคัญของระบบ Microservices:

1. **Service Mesh (Istio/Linkerd)**: จัดการ network concerns ผ่าน sidecar proxy
2. **mTLS**: ความปลอดภัยระหว่าง services โดยไม่ต้องแก้ code
3. **Traffic Management**: Canary deployments, A/B testing, fault injection
4. **Circuit Breaking**: ป้องกัน cascade failures ทั้งที่ mesh level และ code level
5. **Distributed Tracing**: Jaeger/Zipkin ติดตาม request ข้าม services
6. **Prometheus + Grafana**: Metrics collection และ visualization
7. **ELK Stack**: Centralized log aggregation และ analysis
8. **Alerting**: แจ้งเตือนเมื่อระบบมีปัญหา

---

*[← ส่วนที่ 76: Database Patterns](part-76-database-patterns.md) | [ส่วนที่ 78: Event-Driven Architecture →](part-78-event-driven.md)*
