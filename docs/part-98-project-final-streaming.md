# ส่วนที่ 98: Final Project - Streaming Analytics Platform

## สารบัญ

- [1. Architecture Overview](#1-architecture-overview)
- [2. Project Setup](#2-project-setup)
- [3. Kafka Event Ingestion](#3-kafka-event-ingestion)
- [4. fs2 Stream Processing](#4-fs2-stream-processing)
- [5. Aggregations และ Metrics](#5-aggregations-และ-metrics)
- [6. Dashboard API ด้วย WebSocket](#6-dashboard-api-ด้วย-websocket)
- [7. Alerting System](#7-alerting-system)
- [8. Complete Project ด้วย Docker Compose](#8-complete-project-ด้วย-docker-compose)
- [สรุป](#สรุป)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                     Event Sources                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │  App 1  │  │  App 2  │  │  App 3  │  │  App N  │        │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘        │
└───────┼────────────┼────────────┼─────────────┼──────────────┘
        │            │            │             │
        └────────────┴────────────┴─────────────┘
                           │
                    ┌──────▼──────┐
                    │    Kafka    │
                    │  (events)   │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐
│  fs2 Stream   │  │  fs2 Stream   │  │  fs2 Stream   │
│  Processor 1  │  │  Processor 2  │  │  Processor N  │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                   ┌───────▼───────┐
                   │  Aggregation  │
                   │   Engine      │
                   └───────┬───────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐
│   TimescaleDB │  │  Dashboard    │  │   Alerting    │
│   (storage)   │  │   WebSocket   │  │   Service     │
└───────────────┘  └───────────────┘  └───────────────┘
```

---

## 2. Project Setup

### build.sbt

```scala
// build.sbt
val ScalaVersion  = "3.3.1"
val CatsEffect    = "3.5.2"
val Fs2Version    = "3.9.3"
val Http4sVersion = "0.23.23"
val Fs2KafkaVersion = "3.1.0"
val DoobieVersion = "1.0.0-RC4"
val CirceVersion  = "0.14.6"

lazy val root = project
  .in(file("."))
  .settings(
    name         := "streaming-analytics",
    version      := "0.1.0",
    scalaVersion := ScalaVersion,
    
    libraryDependencies ++= Seq(
      // Streaming
      "co.fs2"     %% "fs2-core"  % Fs2Version,
      "co.fs2"     %% "fs2-io"    % Fs2Version,
      
      // Kafka
      "com.github.fd4s" %% "fs2-kafka" % Fs2KafkaVersion,
      
      // HTTP + WebSocket
      "org.http4s" %% "http4s-ember-server" % Http4sVersion,
      "org.http4s" %% "http4s-circe"        % Http4sVersion,
      "org.http4s" %% "http4s-dsl"          % Http4sVersion,
      
      // Database (TimescaleDB = PostgreSQL extension)
      "org.tpolecat" %% "doobie-core"     % DoobieVersion,
      "org.tpolecat" %% "doobie-hikari"   % DoobieVersion,
      "org.tpolecat" %% "doobie-postgres" % DoobieVersion,
      
      // JSON
      "io.circe" %% "circe-core"    % CirceVersion,
      "io.circe" %% "circe-generic" % CirceVersion,
      "io.circe" %% "circe-parser"  % CirceVersion,
      
      // Metrics
      "io.prometheus" % "simpleclient" % "0.16.0",
      "io.prometheus" % "simpleclient_hotspot" % "0.16.0",
      
      // Notifications
      "com.softwaremill.sttp.client4" %% "core"        % "4.0.0",
      "com.softwaremill.sttp.client4" %% "circe"       % "4.0.0",
      
      // Testing
      "org.typelevel" %% "cats-effect-testing-scalatest" % "1.5.0" % Test,
    )
  )
```

---

## 3. Kafka Event Ingestion

### Event Models

```scala
// src/main/scala/domain/Events.scala
import java.time.Instant
import io.circe.*
import io.circe.generic.auto.*

// Raw events จาก Kafka
sealed trait RawEvent:
  def eventId: String
  def timestamp: Instant
  def source: String

case class PageViewEvent(
  eventId: String,
  timestamp: Instant,
  source: String,
  userId: Option[String],
  sessionId: String,
  url: String,
  referrer: Option[String],
  userAgent: String,
  country: Option[String]
) extends RawEvent

case class ClickEvent(
  eventId: String,
  timestamp: Instant,
  source: String,
  userId: Option[String],
  sessionId: String,
  elementId: String,
  elementType: String,
  url: String
) extends RawEvent

case class PurchaseEvent(
  eventId: String,
  timestamp: Instant,
  source: String,
  userId: String,
  orderId: String,
  items: List[PurchaseItem],
  total: Double,
  currency: String
) extends RawEvent

case class ErrorEvent(
  eventId: String,
  timestamp: Instant,
  source: String,
  errorCode: String,
  message: String,
  stackTrace: Option[String],
  userId: Option[String],
  url: Option[String]
) extends RawEvent

case class PurchaseItem(productId: String, quantity: Int, price: Double)
```

### Kafka Consumer

```scala
// src/main/scala/ingestion/KafkaIngestion.scala
import cats.effect.*
import fs2.*
import fs2.kafka.*
import io.circe.parser.*
import io.circe.generic.auto.*
import org.typelevel.log4cats.Logger

class KafkaIngestion[F[_]: Async: Logger](
  settings: ConsumerSettings[F, String, String],
  topics: List[String]
):
  def eventStream: Stream[F, CommittableConsumerRecord[F, String, RawEvent]] =
    KafkaConsumer.stream(settings)
      .subscribeTo(topics.head, topics.tail*)
      .records
      .evalMapFilter { record =>
        parseEvent(record.record.key(), record.record.value())
          .map(event => Some(record.as(event)))
          .handleErrorWith { e =>
            Logger[F].warn(s"Failed to parse event: ${e.getMessage}") >>
            Async[F].pure(None)
          }
      }
  
  private def parseEvent(key: String, value: String): F[RawEvent] =
    Async[F].fromEither {
      parse(value).flatMap { json =>
        key match
          case "PageView" => json.as[PageViewEvent].widen
          case "Click"    => json.as[ClickEvent].widen
          case "Purchase" => json.as[PurchaseEvent].widen
          case "Error"    => json.as[ErrorEvent].widen
          case other      => Left(io.circe.DecodingFailure(s"Unknown event type: $other", Nil))
      }
    }

object KafkaIngestion:
  def create[F[_]: Async: Logger](config: KafkaConfig): KafkaIngestion[F] =
    val settings = ConsumerSettings[F, String, String]
      .withBootstrapServers(config.bootstrapServers)
      .withGroupId(config.groupId)
      .withAutoOffsetReset(AutoOffsetReset.Earliest)
      .withEnableAutoCommit(false)
    
    KafkaIngestion(settings, config.topics)
```

---

## 4. fs2 Stream Processing

### Event Processor

```scala
// src/main/scala/processing/EventProcessor.scala
import cats.effect.*
import cats.effect.std.*
import fs2.*
import fs2.kafka.*
import scala.concurrent.duration.*

class EventProcessor[F[_]: Temporal: Concurrent](
  ingestion: KafkaIngestion[F],
  metricsStore: MetricsStore[F],
  alertEngine: AlertEngine[F]
):
  // Main processing pipeline
  def pipeline: Stream[F, Unit] =
    ingestion.eventStream
      .through(parseAndValidate)
      .through(enrichEvents)
      .through(fanout)
  
  // Validate และ filter bad events
  private def parseAndValidate: Pipe[F, CommittableConsumerRecord[F, String, RawEvent], RawEvent] =
    _.evalMapFilter { record =>
      val event = record.record.value
      if isValidEvent(event) then
        record.offset.commit >> Concurrent[F].pure(Some(event))
      else
        record.offset.commit >> Concurrent[F].pure(None)
    }
  
  // เพิ่ม derived fields
  private def enrichEvents: Pipe[F, RawEvent, EnrichedEvent] =
    _.map { event =>
      EnrichedEvent(
        original = event,
        hour     = event.timestamp.atZone(java.time.ZoneOffset.UTC).getHour,
        dayOfWeek = event.timestamp.atZone(java.time.ZoneOffset.UTC).getDayOfWeek.getValue,
        isWeekend = event.timestamp.atZone(java.time.ZoneOffset.UTC).getDayOfWeek.getValue >= 6
      )
    }
  
  // แยก stream ไปหลาย processor
  private def fanout: Pipe[F, EnrichedEvent, Unit] =
    events =>
      val broadcastPipe = events.broadcastThrough(
        storeMetrics,
        checkAlerts,
        publishToWebSocket
      )
      broadcastPipe
  
  private def storeMetrics: Pipe[F, EnrichedEvent, Unit] =
    _.groupWithin(1000, 10.seconds)
      .evalMap(chunk => metricsStore.storeBatch(chunk.toList))
  
  private def checkAlerts: Pipe[F, EnrichedEvent, Unit] =
    _.through(alertEngine.checkStream)
  
  private def publishToWebSocket: Pipe[F, EnrichedEvent, Unit] =
    _.through(WebSocketBroadcaster.pipe)
  
  private def isValidEvent(event: RawEvent): Boolean =
    event.eventId.nonEmpty && !event.timestamp.isAfter(java.time.Instant.now().plusSeconds(60))

case class EnrichedEvent(
  original: RawEvent,
  hour: Int,
  dayOfWeek: Int,
  isWeekend: Boolean
)
```

### Windowed Aggregations

```scala
// src/main/scala/processing/WindowedAggregator.scala
import cats.effect.*
import cats.effect.std.*
import fs2.*
import scala.concurrent.duration.*
import java.time.Instant

class WindowedAggregator[F[_]: Temporal: Concurrent]:
  // Tumbling window: ไม่ overlap
  def tumblingWindow[A, B](
    duration: FiniteDuration,
    aggregate: Chunk[A] => B
  ): Pipe[F, A, B] =
    _.groupWithin(Int.MaxValue, duration)
      .map(chunk => aggregate(chunk))
  
  // Sliding window: overlap
  def slidingWindow[A, B](
    windowSize: FiniteDuration,
    slideInterval: FiniteDuration,
    aggregate: List[A] => B
  ): Pipe[F, A, B] =
    stream =>
      Stream.eval(Queue.unbounded[F, (Instant, A)]).flatMap { queue =>
        val enqueue = stream
          .evalMap(a => queue.offer((Instant.now(), a)))
        
        val process = Stream.awakeEvery[F](slideInterval)
          .evalMap { _ =>
            val cutoff = Instant.now().minusMillis(windowSize.toMillis)
            // ในทางปฏิบัติต้องใช้ structure ที่ดีกว่านี้
            queue.tryTake.map(_ => aggregate(List.empty))
          }
        
        enqueue.drain.merge(process)
      }
  
  // Session window: group by inactivity gap
  def sessionWindow[A, K](
    keyFn: A => K,
    inactivityTimeout: FiniteDuration
  ): Pipe[F, A, (K, List[A])] =
    stream =>
      Stream.eval(Ref[F].of(Map.empty[K, (Instant, List[A])])).flatMap { sessions =>
        val process = stream
          .evalMapFilter { item =>
            val key = keyFn(item)
            sessions.modify { sess =>
              val now = Instant.now()
              val current = sess.get(key)
              
              // Check for expired sessions
              val expired = sess.filter { case (_, (lastSeen, _)) =>
                now.minusMillis(inactivityTimeout.toMillis).isAfter(lastSeen)
              }.toList
              
              val newSess = sess
                .removed(key)
                .filterNot { case (k, _) => expired.exists(_._1 == k) }
                .updated(key, (now, current.map(_._2 :+ item).getOrElse(List(item))))
              
              val results = expired.map { case (k, (_, items)) => (k, items) }
              (newSess, results)
            }.flatMap {
              case Nil     => Concurrent[F].pure(None)
              case results => Concurrent[F].pure(Some(results))
            }
          }
          .flatMap(Stream.emits)
        
        process
      }
```

---

## 5. Aggregations และ Metrics

### Metrics Models

```scala
// src/main/scala/metrics/Models.scala
import java.time.Instant

// Time series metrics
case class TimeSeriesPoint(
  timestamp: Instant,
  metricName: String,
  tags: Map[String, String],
  value: Double
)

// Aggregated metrics
case class PageViewMetrics(
  windowStart: Instant,
  windowEnd: Instant,
  totalViews: Long,
  uniqueUsers: Long,
  uniqueSessions: Long,
  topPages: List[(String, Long)],
  bounceRate: Double,
  avgSessionDuration: Double
)

case class RevenueMetrics(
  windowStart: Instant,
  windowEnd: Instant,
  totalRevenue: Double,
  orderCount: Long,
  avgOrderValue: Double,
  topProducts: List[(String, Double)],
  revenueByCountry: Map[String, Double]
)

case class ErrorMetrics(
  windowStart: Instant,
  windowEnd: Instant,
  errorCount: Long,
  errorRate: Double,
  topErrors: List[(String, Long)],
  affectedUsers: Long
)

case class RealTimeStats(
  timestamp: Instant,
  activeUsers: Long,
  requestsPerSecond: Double,
  errorRate: Double,
  p99Latency: Double,
  revenueLastHour: Double
)
```

### Metrics Computation

```scala
// src/main/scala/metrics/MetricsEngine.scala
import cats.effect.*
import cats.effect.std.*
import fs2.*
import fs2.Stream
import scala.concurrent.duration.*
import java.time.Instant

class MetricsEngine[F[_]: Temporal: Concurrent](
  metricsStore: MetricsStore[F]
):
  // คำนวณ page view metrics ทุก 1 นาที
  def pageViewMetricsPipeline(events: Stream[F, EnrichedEvent]): Stream[F, PageViewMetrics] =
    events
      .collect { case e if e.original.isInstanceOf[PageViewEvent] =>
        e.original.asInstanceOf[PageViewEvent]
      }
      .groupWithin(Int.MaxValue, 1.minute)
      .evalMap(chunk => computePageViewMetrics(chunk.toList))
  
  private def computePageViewMetrics(events: List[PageViewEvent]): F[PageViewMetrics] =
    Temporal[F].realTimeInstant.map { now =>
      val windowStart = now.minusSeconds(60)
      val totalViews  = events.size.toLong
      val uniqueUsers = events.flatMap(_.userId).toSet.size.toLong
      val uniqueSessions = events.map(_.sessionId).toSet.size.toLong
      
      val pageCounts = events.groupBy(_.url)
        .view.mapValues(_.size.toLong)
        .toList.sortBy(-_._2).take(10)
      
      // Simple bounce rate: sessions with only 1 page view
      val sessionPageCounts = events.groupBy(_.sessionId).view.mapValues(_.size)
      val bounceCount = sessionPageCounts.count(_._2 == 1).toLong
      val bounceRate = if uniqueSessions > 0 then bounceCount.toDouble / uniqueSessions else 0.0
      
      PageViewMetrics(
        windowStart     = windowStart,
        windowEnd       = now,
        totalViews      = totalViews,
        uniqueUsers     = uniqueUsers,
        uniqueSessions  = uniqueSessions,
        topPages        = pageCounts,
        bounceRate      = bounceRate,
        avgSessionDuration = 0.0 // simplified
      )
    }
  
  // Revenue analytics
  def revenueMetricsPipeline(events: Stream[F, EnrichedEvent]): Stream[F, RevenueMetrics] =
    events
      .collect { case e if e.original.isInstanceOf[PurchaseEvent] =>
        e.original.asInstanceOf[PurchaseEvent]
      }
      .groupWithin(Int.MaxValue, 5.minutes)
      .evalMap(chunk => computeRevenueMetrics(chunk.toList))
  
  private def computeRevenueMetrics(events: List[PurchaseEvent]): F[RevenueMetrics] =
    Temporal[F].realTimeInstant.map { now =>
      val totalRevenue = events.map(_.total).sum
      val orderCount   = events.size.toLong
      val avgOrderValue = if orderCount > 0 then totalRevenue / orderCount else 0.0
      
      val productRevenue = events
        .flatMap(_.items.map(item => item.productId -> item.price * item.quantity))
        .groupBy(_._1)
        .view.mapValues(_.map(_._2).sum)
        .toList.sortBy(-_._2).take(10)
      
      RevenueMetrics(
        windowStart    = now.minusSeconds(300),
        windowEnd      = now,
        totalRevenue   = totalRevenue,
        orderCount     = orderCount,
        avgOrderValue  = avgOrderValue,
        topProducts    = productRevenue,
        revenueByCountry = Map.empty // simplified
      )
    }
  
  // Real-time stats: update ทุก 5 วินาที
  def realTimeStatsPipeline(
    events: Stream[F, EnrichedEvent]
  ): Stream[F, RealTimeStats] =
    Stream.awakeEvery[F](5.seconds)
      .evalMap { _ =>
        for
          activeUsers <- metricsStore.getActiveUsersCount
          rps         <- metricsStore.getRequestsPerSecond
          errorRate   <- metricsStore.getErrorRate(5.minutes)
          revenue     <- metricsStore.getRevenueSince(java.time.Instant.now().minusSeconds(3600))
          now         <- Temporal[F].realTimeInstant
        yield RealTimeStats(
          timestamp         = now,
          activeUsers       = activeUsers,
          requestsPerSecond = rps,
          errorRate         = errorRate,
          p99Latency        = 0.0, // simplified
          revenueLastHour   = revenue
        )
      }
```

### Metrics Storage ด้วย TimescaleDB

```scala
// src/main/scala/storage/MetricsStore.scala
import cats.effect.*
import doobie.*
import doobie.implicits.*
import doobie.postgres.implicits.*
import scala.concurrent.duration.*
import java.time.Instant

class MetricsStore[F[_]: Async](xa: Transactor[F]):
  def storeBatch(events: List[EnrichedEvent]): F[Unit] =
    val points = events.map { event =>
      event.original match
        case pv: PageViewEvent =>
          TimeSeriesPoint(pv.timestamp, "page_view", Map("url" -> pv.url), 1.0)
        case c: ClickEvent =>
          TimeSeriesPoint(c.timestamp, "click", Map("element" -> c.elementId), 1.0)
        case p: PurchaseEvent =>
          TimeSeriesPoint(p.timestamp, "revenue", Map("currency" -> p.currency), p.total)
        case e: ErrorEvent =>
          TimeSeriesPoint(e.timestamp, "error", Map("code" -> e.errorCode), 1.0)
    }
    
    storePoints(points)
  
  def storePoints(points: List[TimeSeriesPoint]): F[Unit] =
    val sql = Update[(Instant, String, Map[String, String], Double)](
      """INSERT INTO metrics (timestamp, metric_name, tags, value)
         VALUES (?, ?, ?, ?)
         ON CONFLICT DO NOTHING"""
    )
    sql.updateMany(points.map(p => (p.timestamp, p.metricName, p.tags, p.value)))
       .transact(xa).void
  
  def getActiveUsersCount: F[Long] =
    sql"""
      SELECT COUNT(DISTINCT tags->>'user_id')
      FROM metrics
      WHERE metric_name = 'page_view'
        AND timestamp > NOW() - INTERVAL '5 minutes'
        AND tags->>'user_id' IS NOT NULL
    """.query[Long].unique.transact(xa)
  
  def getRequestsPerSecond: F[Double] =
    sql"""
      SELECT COUNT(*) / 60.0
      FROM metrics
      WHERE metric_name = 'page_view'
        AND timestamp > NOW() - INTERVAL '1 minute'
    """.query[Double].unique.transact(xa)
  
  def getErrorRate(window: FiniteDuration): F[Double] =
    val windowSeconds = window.toSeconds
    sql"""
      WITH totals AS (
        SELECT
          COUNT(*) FILTER (WHERE metric_name = 'page_view') as total,
          COUNT(*) FILTER (WHERE metric_name = 'error') as errors
        FROM metrics
        WHERE timestamp > NOW() - make_interval(secs => $windowSeconds)
      )
      SELECT CASE WHEN total > 0 THEN errors::double precision / total ELSE 0 END
      FROM totals
    """.query[Double].unique.transact(xa)
  
  def getRevenueSince(since: Instant): F[Double] =
    sql"""
      SELECT COALESCE(SUM(value), 0)
      FROM metrics
      WHERE metric_name = 'revenue'
        AND timestamp > $since
    """.query[Double].unique.transact(xa)
  
  def getMetricTimeSeries(
    metricName: String,
    bucketInterval: String,
    from: Instant,
    to: Instant
  ): F[List[(Instant, Double)]] =
    // TimescaleDB time_bucket function
    sql"""
      SELECT
        time_bucket($bucketInterval::interval, timestamp) as bucket,
        SUM(value)
      FROM metrics
      WHERE metric_name = $metricName
        AND timestamp BETWEEN $from AND $to
      GROUP BY bucket
      ORDER BY bucket
    """.query[(Instant, Double)].to[List].transact(xa)
```

---

## 6. Dashboard API ด้วย WebSocket

### WebSocket Server

```scala
// src/main/scala/api/DashboardApi.scala
import cats.effect.*
import cats.effect.std.*
import org.http4s.*
import org.http4s.dsl.Http4sDsl
import org.http4s.server.websocket.WebSocketBuilder2
import org.http4s.websocket.WebSocketFrame
import fs2.*
import fs2.concurrent.*
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.auto.*
import scala.concurrent.duration.*

class DashboardApi[F[_]: Concurrent: Temporal](
  metricsEngine: MetricsEngine[F],
  metricsStore: MetricsStore[F]
) extends Http4sDsl[F]:
  
  // Broadcast channel สำหรับ real-time stats
  private val broadcaster: F[Topic[F, DashboardUpdate]] = Topic[F, DashboardUpdate]
  
  def routes(wsBuilder: WebSocketBuilder2[F]): HttpRoutes[F] = HttpRoutes.of[F] {
    // REST endpoints
    case GET -> Root / "api" / "metrics" / "realtime" =>
      metricsStore.getActiveUsersCount.flatMap { users =>
        Ok(Map("activeUsers" -> users).asJson)
      }
    
    case GET -> Root / "api" / "metrics" / "timeseries" :? MetricParam(metric) +& FromParam(from) +& ToParam(to) =>
      metricsStore.getMetricTimeSeries(metric, "1 minute", from, to)
        .flatMap(ts => Ok(ts.map { case (t, v) => Map("timestamp" -> t.toString, "value" -> v) }.asJson))
    
    // WebSocket endpoint สำหรับ real-time dashboard
    case GET -> Root / "ws" / "dashboard" =>
      for
        topic <- broadcaster
        
        // Output: stream stats to client
        send = topic.subscribe(100)
          .map(update => WebSocketFrame.Text(update.asJson.noSpaces))
        
        // Input: handle client messages
        receive: Pipe[F, WebSocketFrame, Unit] = _.evalMap {
          case WebSocketFrame.Text(text, _) =>
            Concurrent[F].unit // handle client commands if needed
          case WebSocketFrame.Close(_) =>
            Concurrent[F].unit
          case _ =>
            Concurrent[F].unit
        }
        
        response <- wsBuilder.build(send, receive)
      yield response
    
    // Server-Sent Events (alternative to WebSocket)
    case GET -> Root / "api" / "events" / "stream" =>
      val eventStream = Stream.awakeEvery[F](1.second)
        .evalMap(_ => buildDashboardUpdate)
        .map { update =>
          val data = update.asJson.noSpaces
          s"data: $data\n\n"
        }
      
      Ok(eventStream)
        .map(_.withHeaders(
          Header.Raw("Content-Type".ci, "text/event-stream"),
          Header.Raw("Cache-Control".ci, "no-cache"),
          Header.Raw("Connection".ci, "keep-alive")
        ))
  }
  
  // Start broadcasting updates
  def startBroadcasting: Stream[F, Unit] =
    Stream.eval(broadcaster).flatMap { topic =>
      Stream.awakeEvery[F](2.seconds)
        .evalMap(_ => buildDashboardUpdate)
        .through(topic.publish)
    }
  
  private def buildDashboardUpdate: F[DashboardUpdate] =
    for
      activeUsers <- metricsStore.getActiveUsersCount
      rps         <- metricsStore.getRequestsPerSecond
      errorRate   <- metricsStore.getErrorRate(5.minutes)
      revenue     <- metricsStore.getRevenueSince(
                       java.time.Instant.now().minusSeconds(3600)
                     )
      now         <- Temporal[F].realTimeInstant
    yield DashboardUpdate(
      timestamp   = now.toString,
      activeUsers = activeUsers,
      rps         = rps,
      errorRate   = errorRate,
      revenueHour = revenue
    )

case class DashboardUpdate(
  timestamp: String,
  activeUsers: Long,
  rps: Double,
  errorRate: Double,
  revenueHour: Double
)

// Query params
object MetricParam extends QueryParamDecoderMatcher[String]("metric")
object FromParam extends QueryParamDecoderMatcher[java.time.Instant]("from")
object ToParam extends QueryParamDecoderMatcher[java.time.Instant]("to")
```

### Simple Dashboard HTML

```scala
// src/main/scala/api/StaticRoutes.scala
import org.http4s.*
import org.http4s.dsl.Http4sDsl

class StaticRoutes[F[_]: cats.effect.Sync] extends Http4sDsl[F]:
  val routes: HttpRoutes[F] = HttpRoutes.of[F] {
    case GET -> Root =>
      Ok(dashboardHtml, Header.Raw("Content-Type".ci, "text/html"))
  }
  
  private val dashboardHtml = """
<!DOCTYPE html>
<html>
<head>
  <title>Analytics Dashboard</title>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; background: #1a1a2e; color: #eee; }
    .metric { background: #16213e; padding: 20px; margin: 10px; border-radius: 8px; display: inline-block; min-width: 200px; }
    .metric .value { font-size: 2em; color: #0f3460; color: #e94560; }
    .metric .label { font-size: 0.9em; color: #aaa; }
    #chart { width: 100%; height: 300px; background: #16213e; margin-top: 20px; border-radius: 8px; }
    .status { color: #2ecc71; }
    .status.disconnected { color: #e74c3c; }
  </style>
</head>
<body>
  <h1>Real-Time Analytics</h1>
  <div id="status" class="status">⚡ Connected</div>
  
  <div id="metrics">
    <div class="metric">
      <div class="value" id="activeUsers">-</div>
      <div class="label">Active Users</div>
    </div>
    <div class="metric">
      <div class="value" id="rps">-</div>
      <div class="label">Requests/sec</div>
    </div>
    <div class="metric">
      <div class="value" id="errorRate">-</div>
      <div class="label">Error Rate</div>
    </div>
    <div class="metric">
      <div class="value" id="revenue">-</div>
      <div class="label">Revenue (1h)</div>
    </div>
  </div>
  
  <canvas id="chart"></canvas>
  
  <script>
    const ws = new WebSocket('ws://' + window.location.host + '/ws/dashboard');
    const history = [];
    
    ws.onopen = () => {
      document.getElementById('status').textContent = '⚡ Connected';
    };
    
    ws.onclose = () => {
      document.getElementById('status').textContent = '⚠️ Disconnected';
      document.getElementById('status').className = 'status disconnected';
    };
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      
      document.getElementById('activeUsers').textContent = data.activeUsers;
      document.getElementById('rps').textContent = data.rps.toFixed(1);
      document.getElementById('errorRate').textContent = (data.errorRate * 100).toFixed(2) + '%';
      document.getElementById('revenue').textContent = '$' + data.revenueHour.toFixed(2);
      
      history.push(data);
      if (history.length > 60) history.shift();
      drawChart();
    };
    
    function drawChart() {
      const canvas = document.getElementById('chart');
      const ctx = canvas.getContext('2d');
      canvas.width = canvas.offsetWidth;
      canvas.height = 300;
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      
      if (history.length < 2) return;
      
      const maxUsers = Math.max(...history.map(d => d.activeUsers));
      
      ctx.strokeStyle = '#e94560';
      ctx.lineWidth = 2;
      ctx.beginPath();
      
      history.forEach((d, i) => {
        const x = (i / (history.length - 1)) * canvas.width;
        const y = canvas.height - (d.activeUsers / maxUsers) * (canvas.height - 20);
        i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
      });
      
      ctx.stroke();
    }
  </script>
</body>
</html>
"""
```

---

## 7. Alerting System

### Alert Models

```scala
// src/main/scala/alerting/Models.scala
import java.time.Instant

sealed trait AlertCondition
case class ThresholdAlert(
  metric: String,
  threshold: Double,
  comparison: Comparison,
  windowMinutes: Int
) extends AlertCondition

case class AnomalyAlert(
  metric: String,
  deviationFactor: Double,  // alert if value deviates by this factor from moving avg
  windowMinutes: Int
) extends AlertCondition

enum Comparison:
  case GreaterThan, LessThan, GreaterThanOrEqual, LessThanOrEqual

case class AlertRule(
  id: String,
  name: String,
  condition: AlertCondition,
  severity: AlertSeverity,
  channels: List[NotificationChannel]
)

enum AlertSeverity:
  case Info, Warning, Critical

sealed trait NotificationChannel
case class SlackChannel(webhookUrl: String) extends NotificationChannel
case class EmailChannel(addresses: List[String]) extends NotificationChannel
case class PagerDutyChannel(serviceKey: String) extends NotificationChannel

case class Alert(
  ruleId: String,
  ruleName: String,
  message: String,
  severity: AlertSeverity,
  metricValue: Double,
  threshold: Double,
  triggeredAt: Instant
)
```

### Alert Engine

```scala
// src/main/scala/alerting/AlertEngine.scala
import cats.effect.*
import cats.effect.std.*
import cats.syntax.all.*
import fs2.*
import scala.concurrent.duration.*
import java.time.Instant

class AlertEngine[F[_]: Temporal: Concurrent](
  rules: List[AlertRule],
  metricsStore: MetricsStore[F],
  notifier: AlertNotifier[F]
):
  // Pipe ที่ตรวจสอบ alerts ตาม stream
  def checkStream: Pipe[F, EnrichedEvent, Unit] =
    events =>
      // Check alerts ทุก 30 วินาที
      Stream.awakeEvery[F](30.seconds)
        .evalMap(_ => checkAllRules)
        .flatMap(Stream.emits)
        .evalMap(notifier.send)
  
  def checkAllRules: F[List[Alert]] =
    rules.traverseFilter(checkRule)
  
  private def checkRule(rule: AlertRule): F[Option[Alert]] =
    rule.condition match
      case ThresholdAlert(metric, threshold, comparison, windowMins) =>
        checkThreshold(rule, metric, threshold, comparison, windowMins)
      case AnomalyAlert(metric, deviationFactor, windowMins) =>
        checkAnomaly(rule, metric, deviationFactor, windowMins)
  
  private def checkThreshold(
    rule: AlertRule,
    metric: String,
    threshold: Double,
    comparison: Comparison,
    windowMins: Int
  ): F[Option[Alert]] =
    val since = Instant.now().minusSeconds(windowMins * 60L)
    metricsStore.getRevenueSince(since).map { currentValue =>
      val triggered = comparison match
        case Comparison.GreaterThan        => currentValue > threshold
        case Comparison.LessThan           => currentValue < threshold
        case Comparison.GreaterThanOrEqual => currentValue >= threshold
        case Comparison.LessThanOrEqual    => currentValue <= threshold
      
      if triggered then
        Some(Alert(
          ruleId       = rule.id,
          ruleName     = rule.name,
          message      = s"${rule.name}: $metric = $currentValue ${comparison} $threshold",
          severity     = rule.severity,
          metricValue  = currentValue,
          threshold    = threshold,
          triggeredAt  = Instant.now()
        ))
      else None
    }
  
  private def checkAnomaly(
    rule: AlertRule,
    metric: String,
    deviationFactor: Double,
    windowMins: Int
  ): F[Option[Alert]] =
    // คำนวณ moving average และ current value
    val since = Instant.now().minusSeconds(windowMins * 60L)
    metricsStore.getMetricTimeSeries(metric, "1 minute", since, Instant.now())
      .map { timeSeries =>
        if timeSeries.size < 10 then None
        else
          val values = timeSeries.map(_._2)
          val avg = values.sum / values.size
          val current = values.lastOption.getOrElse(0.0)
          
          val deviation = if avg != 0 then Math.abs(current - avg) / avg else 0.0
          
          if deviation > deviationFactor then
            Some(Alert(
              ruleId      = rule.id,
              ruleName    = rule.name,
              message     = s"Anomaly detected in $metric: current=$current, avg=$avg, deviation=${(deviation * 100).toInt}%",
              severity    = rule.severity,
              metricValue = current,
              threshold   = avg * deviationFactor,
              triggeredAt = Instant.now()
            ))
          else None
      }

// Alert Notifier
class AlertNotifier[F[_]: Concurrent](
  slackClient: Option[SlackClient[F]],
  emailClient: Option[EmailClient[F]]
):
  def send(alert: Alert, channels: List[NotificationChannel] = Nil): F[Unit] =
    val message = formatAlert(alert)
    channels.traverse_ {
      case SlackChannel(url) =>
        slackClient.traverse_(_.send(url, message))
      case EmailChannel(addresses) =>
        emailClient.traverse_(_.send(addresses, s"Alert: ${alert.ruleName}", message))
      case PagerDutyChannel(key) =>
        Concurrent[F].unit // implement if needed
    }
  
  private def formatAlert(alert: Alert): String =
    s"""🚨 ${alert.severity} Alert: ${alert.ruleName}
       |Message: ${alert.message}
       |Triggered at: ${alert.triggeredAt}
       |Value: ${alert.metricValue}
       |""".stripMargin
```

---

## 8. Complete Project ด้วย Docker Compose

### docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  analytics:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: jdbc:postgresql://timescaledb:5432/analytics
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      KAFKA_TOPICS: events,purchases,errors
    depends_on:
      timescaledb:
        condition: service_healthy
      kafka:
        condition: service_healthy
    deploy:
      replicas: 2  # Scale horizontally
      resources:
        limits:
          cpus: '1.0'
          memory: 512M

  timescaledb:
    image: timescale/timescaledb:latest-pg15
    environment:
      POSTGRES_DB: analytics
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - timescale_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_HEAP_OPTS: "-Xmx256m"

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_HEAP_OPTS: "-Xmx512m"
    healthcheck:
      test: ["CMD-SHELL", "kafka-topics --bootstrap-server localhost:9092 --list"]
      interval: 10s
      retries: 5

  # Load generator สำหรับทดสอบ
  load-generator:
    image: alpine
    command: >
      sh -c "
        apk add --no-cache curl &&
        while true; do
          curl -s -X POST http://analytics:8080/api/events -H 'Content-Type: application/json'
            -d '{\"type\": \"PageView\", \"url\": \"/home\"}' > /dev/null;
          sleep 0.1;
        done
      "
    depends_on: [analytics]

  # Prometheus สำหรับ metrics
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  # Grafana สำหรับ visualization
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on: [prometheus, timescaledb]

volumes:
  timescale_data:
  grafana_data:
```

### SQL Init Script

```sql
-- init.sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE IF NOT EXISTS metrics (
    timestamp   TIMESTAMPTZ NOT NULL,
    metric_name VARCHAR(100) NOT NULL,
    tags        JSONB NOT NULL DEFAULT '{}',
    value       DOUBLE PRECISION NOT NULL
);

-- Convert to hypertable (TimescaleDB feature)
SELECT create_hypertable('metrics', 'timestamp', if_not_exists => TRUE);

-- Create indexes for common queries
CREATE INDEX IF NOT EXISTS idx_metrics_name_time ON metrics(metric_name, timestamp DESC);
CREATE INDEX IF NOT EXISTS idx_metrics_tags ON metrics USING GIN(tags);

-- Retention policy: keep 30 days of data
SELECT add_retention_policy('metrics', INTERVAL '30 days', if_not_exists => TRUE);

-- Continuous aggregates: pre-compute hourly stats
CREATE MATERIALIZED VIEW IF NOT EXISTS metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', timestamp) as bucket,
    metric_name,
    COUNT(*) as count,
    SUM(value) as total,
    AVG(value) as average,
    MAX(value) as maximum,
    MIN(value) as minimum
FROM metrics
GROUP BY bucket, metric_name;

SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute',
    if_not_exists => TRUE);
```

### Main Application

```scala
// src/main/scala/Main.scala
import cats.effect.*
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s.*
import fs2.Stream

object Main extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    val config = AppConfig.load
    
    val app = for
      xa      <- Database.transactor(config.database)
      _       <- Resource.eval(Database.migrate(config.database))
      
      metricsStore    = MetricsStore(xa)
      alertRules      = loadAlertRules(config)
      alertNotifier   = AlertNotifier(None, None) // configure in production
      alertEngine     = AlertEngine(alertRules, metricsStore, alertNotifier)
      metricsEngine   = MetricsEngine(metricsStore)
      dashboardApi    = DashboardApi(metricsEngine, metricsStore)
      staticRoutes    = StaticRoutes()
      
      ingestion = KafkaIngestion.create(config.kafka)
      processor = EventProcessor(ingestion, metricsStore, alertEngine)
      
      // HTTP Server
      server <- EmberServerBuilder.default[IO]
        .withHost(Host.fromString("0.0.0.0").get)
        .withPort(Port.fromInt(8080).get)
        .withHttpWebSocketApp(ws =>
          (dashboardApi.routes(ws) <+> staticRoutes.routes).orNotFound
        )
        .build
    yield (server, processor)
    
    app.use { case (_, processor) =>
      // รัน processing pipeline ใน background
      processor.pipeline.compile.drain
        .as(ExitCode.Success)
    }
  
  def loadAlertRules(config: AppConfig): List[AlertRule] =
    List(
      AlertRule(
        id = "high-error-rate",
        name = "High Error Rate",
        condition = ThresholdAlert("error", 0.05, Comparison.GreaterThan, 5),
        severity = AlertSeverity.Critical,
        channels = List()
      ),
      AlertRule(
        id = "low-revenue",
        name = "Low Revenue Alert",
        condition = ThresholdAlert("revenue", 100.0, Comparison.LessThan, 60),
        severity = AlertSeverity.Warning,
        channels = List()
      )
    )
```

---

## สรุป

Streaming Analytics Platform นี้ประกอบด้วย:

| Component | Technology | หน้าที่ |
|-----------|-----------|---------|
| Event Ingestion | fs2-kafka | consume events จาก Kafka |
| Stream Processing | fs2 | transform, filter, aggregate |
| Time Series DB | TimescaleDB | เก็บ metrics efficiently |
| Real-time API | http4s + WebSocket | dashboard updates |
| Alerting | Custom engine | แจ้งเตือนเมื่อ threshold ถึง |
| Visualization | Grafana | dashboard graphs |
| Monitoring | Prometheus | system metrics |

### Key Concepts ที่ได้เรียน

1. **Back-pressure**: Bounded queues ป้องกัน out-of-memory
2. **Windowed aggregations**: Tumbling vs sliding vs session windows  
3. **Fan-out**: หนึ่ง stream ไปหลาย consumers
4. **Real-time vs batch**: Trade-offs ของแต่ละ approach
5. **Fault tolerance**: Handle failures gracefully ด้วย retry logic

---

*[← Part 97: Final Project REST API](part-97-project-final-api.md) | [Part 99: Scala Ecosystem →](part-99-scala-ecosystem.md)*
