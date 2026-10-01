# ส่วนที่ 82: Distributed Streaming Systems

## สารบัญ

1. [Stream Processing Models](#stream-processing-models)
2. [Lambda vs Kappa Architecture](#lambda-vs-kappa-architecture)
3. [Apache Flink Concepts](#apache-flink-concepts)
4. [Exactly-Once Processing](#exactly-once-processing)
5. [State Management ใน Streaming](#state-management-ใน-streaming)
6. [Kafka Streams กับ Scala](#kafka-streams-กับ-scala)
7. [Complete Stream Processing Design](#complete-stream-processing-design)
8. [สรุป](#สรุป)

---

## Stream Processing Models

Stream processing คือการประมวลผลข้อมูลแบบ real-time ที่ไหลเข้ามาอย่างต่อเนื่อง

### Batch vs Stream Processing

```scala
// Batch Processing - ประมวลผลข้อมูลเป็น chunks
def batchProcess(data: Seq[Event]): Seq[Result] =
  data
    .groupBy(_.hour)
    .map { (hour, events) =>
      Result(hour, events.map(_.value).sum)
    }
    .toSeq

// Stream Processing - ประมวลผล event by event
// ใช้ fs2 สำหรับ stream processing ใน Scala
import fs2.Stream
import cats.effect.IO

def streamProcess(events: Stream[IO, Event]): Stream[IO, Result] =
  events
    .groupAdjacentBy(_.hour)
    .evalMap { case (hour, chunk) =>
      IO.pure(Result(hour, chunk.toList.map(_.value).sum))
    }
```

### Event Time vs Processing Time

```scala
import java.time.Instant
import java.time.Duration

// Event time: เวลาที่ event เกิดจริง
// Processing time: เวลาที่ event ถูก process

case class Event(
  id: String,
  value: Double,
  eventTime: Instant,    // เวลาที่เกิดจริง
  processingTime: Instant = Instant.now() // เวลาที่ถูก process
)

// Late data: event ที่มาช้ากว่ากำหนด
def isLate(event: Event, watermark: Instant): Boolean =
  event.eventTime.isBefore(watermark)

// Watermark: เส้นแบ่ง "เร็วพอ" vs "ช้าเกินไป"
class WatermarkTracker(maxLateness: Duration):
  private var currentWatermark: Instant = Instant.EPOCH
  
  def update(eventTime: Instant): Unit =
    val newWatermark = eventTime.minus(maxLateness)
    if newWatermark.isAfter(currentWatermark) then
      currentWatermark = newWatermark
  
  def watermark: Instant = currentWatermark
  
  def isLate(event: Event): Boolean =
    event.eventTime.isBefore(currentWatermark)
```

### Window Types

```scala
// Tumbling Window - windows ที่ไม่ overlap
def tumblingWindow(duration: Duration): Window = Window.Tumbling(duration)

// Sliding Window - windows ที่ overlap กัน
def slidingWindow(duration: Duration, slide: Duration): Window = 
  Window.Sliding(duration, slide)

// Session Window - windows ที่ขึ้นอยู่กับ activity gap
def sessionWindow(gap: Duration): Window = Window.Session(gap)

sealed trait Window:
  def assignWindows(eventTime: Instant): List[WindowKey]

case class WindowKey(start: Instant, end: Instant):
  def contains(time: Instant): Boolean =
    !time.isBefore(start) && time.isBefore(end)

object Window:
  case class Tumbling(duration: Duration) extends Window:
    def assignWindows(eventTime: Instant): List[WindowKey] =
      val startMs = (eventTime.toEpochMilli / duration.toMillis) * duration.toMillis
      val start = Instant.ofEpochMilli(startMs)
      List(WindowKey(start, start.plus(duration)))
  
  case class Sliding(duration: Duration, slide: Duration) extends Window:
    def assignWindows(eventTime: Instant): List[WindowKey] =
      val numWindows = (duration.toMillis / slide.toMillis).toInt
      (0 until numWindows).toList.map { i =>
        val windowStart = eventTime.toEpochMilli - (eventTime.toEpochMilli % slide.toMillis) - i * slide.toMillis
        val start = Instant.ofEpochMilli(windowStart)
        WindowKey(start, start.plus(duration))
      }.filter(_.contains(eventTime))
  
  case class Session(gap: Duration) extends Window:
    // Session windows ต้องการ stateful processing
    def assignWindows(eventTime: Instant): List[WindowKey] =
      List(WindowKey(eventTime, eventTime.plus(gap))) // simplified
```

### Stream Aggregations

```scala
import fs2.Stream
import cats.effect.IO
import scala.concurrent.duration.*

// Count events per window
def countPerWindow(
  events: Stream[IO, Event],
  windowDuration: FiniteDuration
): Stream[IO, (Long, Long)] =  // (windowStart, count)
  events
    .groupWithin(Int.MaxValue, windowDuration)
    .map { chunk =>
      val windowStart = chunk.headOption.map(_.eventTime.toEpochMilli).getOrElse(0L)
      (windowStart, chunk.size.toLong)
    }

// Sum values per window
def sumPerWindow(
  events: Stream[IO, Event],
  windowDuration: FiniteDuration
): Stream[IO, (Long, Double)] =
  events
    .groupWithin(Int.MaxValue, windowDuration)
    .map { chunk =>
      val windowStart = chunk.headOption.map(_.eventTime.toEpochMilli).getOrElse(0L)
      val sum = chunk.toList.map(_.value).sum
      (windowStart, sum)
    }
```

---

## Lambda vs Kappa Architecture

### Lambda Architecture

```
          ┌─────────────────────────────────────────┐
          │           Data Sources                   │
          └───────────────┬─────────────────────────┘
                          │
                          ▼
          ┌───────────────────────────────────────┐
          │              Message Queue              │
          │            (Kafka/Kinesis)              │
          └──────────┬─────────────────────────────┘
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
    ┌──────────┐          ┌──────────┐
    │  Batch   │          │  Speed   │
    │  Layer   │          │  Layer   │
    │ (Spark)  │          │ (Flink)  │
    └────┬─────┘          └────┬─────┘
         │                     │
         ▼                     ▼
    ┌──────────┐          ┌──────────┐
    │  Batch   │          │  Real-   │
    │  Views   │          │  time    │
    │ (HDFS)   │          │  Views   │
    └────┬─────┘          └────┬─────┘
         │                     │
         └──────────┬──────────┘
                    ▼
             ┌──────────┐
             │  Serving │
             │  Layer   │
             └──────────┘
```

```scala
// Lambda Architecture ใน Scala
// Batch Layer - Apache Spark
object BatchLayer:
  def recompute(spark: SparkSession, dataPath: String): Unit =
    import spark.implicits.*
    
    val events = spark.read.parquet(dataPath)
      .as[Event]
    
    // recompute batch views
    val hourlyAggregations = events
      .groupBy(
        window($"eventTime", "1 hour"),
        $"userId"
      )
      .agg(
        sum("value").as("totalValue"),
        count("*").as("eventCount"),
        avg("value").as("avgValue")
      )
    
    hourlyAggregations
      .write
      .mode("overwrite")
      .parquet(s"$dataPath/batch-views/hourly")

// Speed Layer - Kafka Streams / Flink
// ครอบคลุมข้อมูลล่าสุดจนกว่า batch จะ recompute

// Serving Layer - merge batch + speed views
case class HourlyStats(
  hour: Long,
  userId: String,
  totalValue: Double,
  eventCount: Long,
  isFromBatch: Boolean
)

def getStats(userId: String, hour: Long): IO[HourlyStats] =
  for
    batchStats <- getBatchStats(userId, hour)
    realtimeStats <- getRealtimeStats(userId, hour)
  yield
    // เลือก batch ถ้ามี, ไม่งั้นใช้ realtime
    batchStats.orElse(realtimeStats)
      .getOrElse(HourlyStats(hour, userId, 0, 0, false))
```

### Kappa Architecture (Simpler)

```scala
// Kappa Architecture - ใช้เฉพาะ stream processing
// ประมวลผลทุกอย่างด้วย streaming, reprocess เมื่อต้องการ

// การ reprocess ใน Kappa Architecture:
// 1. เก็บ raw events ใน Kafka ด้วย long retention
// 2. เมื่อต้องการ recompute: อ่าน from beginning
// 3. เปรียบเทียบกับ Lambda: ไม่ต้องมี batch layer แยก

object KappaArchitecture:
  // Stream processor ที่ handle ทั้ง realtime และ historical
  def createProcessor(bootstrapServers: String, fromBeginning: Boolean = false):
      fs2.kafka.KafkaConsumer[IO, String, Event] =
    import fs2.kafka.*
    
    val offsetReset = if fromBeginning then "earliest" else "latest"
    
    val consumerSettings = ConsumerSettings[IO, String, Event]
      .withBootstrapServers(bootstrapServers)
      .withGroupId("kappa-processor")
      .withAutoOffsetReset(
        if fromBeginning then AutoOffsetReset.Earliest 
        else AutoOffsetReset.Latest
      )
    
    KafkaConsumer.make(consumerSettings)
      .allocated
      .map(_._1)
      .unsafeRunSync()
```

---

## Apache Flink Concepts

Apache Flink เป็น stream processing framework ที่ทรงพลัง

### Flink Job Structure (Scala API)

```scala
import org.apache.flink.streaming.api.scala.*
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.windowing.windows.TimeWindow
import org.apache.flink.util.Collector
import org.apache.flink.streaming.api.functions.windowing.WindowFunction

// Setup Flink execution environment
val env = StreamExecutionEnvironment.getExecutionEnvironment
env.setParallelism(4)
env.enableCheckpointing(60000) // checkpoint every 60 seconds

// Define event types
case class ClickEvent(userId: String, url: String, timestamp: Long, value: Double)
case class PageStats(url: String, windowEnd: Long, uniqueUsers: Long, totalClicks: Long)

// Create stream from Kafka
import org.apache.flink.connector.kafka.source.KafkaSource
import org.apache.flink.api.common.serialization.SimpleStringSchema

val kafkaSource = KafkaSource.builder[String]()
  .setBootstrapServers("kafka:9092")
  .setTopics("click-events")
  .setGroupId("flink-processor")
  .setStartingOffsets(
    org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer.latest()
  )
  .setValueOnlyDeserializer(new SimpleStringSchema())
  .build()

val clickStream: DataStream[ClickEvent] = env
  .fromSource(kafkaSource, 
    org.apache.flink.api.common.eventtime.WatermarkStrategy.noWatermarks(),
    "Kafka Source")
  .map(json => parseClickEvent(json)) // parse JSON to case class

// Process stream
val pageStatsStream: DataStream[PageStats] = clickStream
  .assignTimestampsAndWatermarks(
    org.apache.flink.api.common.eventtime.WatermarkStrategy
      .forBoundedOutOfOrderness[ClickEvent](java.time.Duration.ofSeconds(5))
      .withTimestampAssigner { (event, _) => event.timestamp }
  )
  .keyBy(_.url)
  .window(Time.minutes(1) |> TumblingEventTimeWindows.of(_))
  .apply(new WindowFunction[ClickEvent, PageStats, String, TimeWindow] {
    override def apply(
      key: String,
      window: TimeWindow,
      input: Iterable[ClickEvent],
      out: Collector[PageStats]
    ): Unit =
      val events = input.toList
      val uniqueUsers = events.map(_.userId).distinct.size.toLong
      val totalClicks = events.size.toLong
      out.collect(PageStats(key, window.getEnd, uniqueUsers, totalClicks))
  })

// Sink to Kafka
pageStatsStream
  .map(stats => upickle.default.write(stats))
  .sinkTo(createKafkaSink("page-stats"))

env.execute("Click Events Processing")
```

### Flink State Management

```scala
import org.apache.flink.api.common.state.*
import org.apache.flink.streaming.api.functions.KeyedProcessFunction
import org.apache.flink.util.Collector

// Stateful processing ด้วย Flink
class UserSessionProcessor 
    extends KeyedProcessFunction[String, ClickEvent, SessionSummary]:
  
  // State
  @transient lazy val sessionStart: ValueState[Long] = 
    getRuntimeContext.getState(
      new ValueStateDescriptor[Long]("session-start", classOf[Long])
    )
  
  @transient lazy val clickCount: ValueState[Int] =
    getRuntimeContext.getState(
      new ValueStateDescriptor[Int]("click-count", classOf[Int])
    )
  
  @transient lazy val urlsVisited: ListState[String] =
    getRuntimeContext.getListState(
      new ListStateDescriptor[String]("urls", classOf[String])
    )
  
  private val SESSION_TIMEOUT = 30 * 60 * 1000L // 30 minutes
  
  override def processElement(
    event: ClickEvent,
    ctx: KeyedProcessFunction[String, ClickEvent, SessionSummary]#Context,
    out: Collector[SessionSummary]
  ): Unit =
    val currentTime = event.timestamp
    
    // Initialize session if needed
    if sessionStart.value() == 0 then
      sessionStart.update(currentTime)
    
    // Update state
    val count = Option(clickCount.value()).getOrElse(0)
    clickCount.update(count + 1)
    urlsVisited.add(event.url)
    
    // Register timer for session timeout
    ctx.timerService().registerEventTimeTimer(currentTime + SESSION_TIMEOUT)
  
  override def onTimer(
    timestamp: Long,
    ctx: KeyedProcessFunction[String, ClickEvent, SessionSummary]#OnTimerContext,
    out: Collector[SessionSummary]
  ): Unit =
    import scala.jdk.CollectionConverters.*
    
    // Session expired - emit summary
    val summary = SessionSummary(
      userId = ctx.getCurrentKey,
      sessionStart = sessionStart.value(),
      sessionEnd = timestamp,
      clickCount = Option(clickCount.value()).getOrElse(0),
      uniqueUrls = urlsVisited.get().asScala.toSet.size
    )
    out.collect(summary)
    
    // Clear state
    sessionStart.clear()
    clickCount.clear()
    urlsVisited.clear()

case class SessionSummary(
  userId: String,
  sessionStart: Long,
  sessionEnd: Long,
  clickCount: Int,
  uniqueUrls: Int
)
```

---

## Exactly-Once Processing

### ปัญหาของ At-Least-Once Processing

```scala
// At-least-once: อาจ process event ซ้ำ
// ปัญหา: ถ้า failure เกิดหลัง process แต่ก่อน commit offset
// event จะถูก process ซ้ำเมื่อ restart

// ตัวอย่าง: bank transfer ที่ process ซ้ำ = disaster!

// Exactly-once ต้องการ:
// 1. Idempotent producers (Kafka)
// 2. Transactional processing
// 3. Atomic commit (offset + state)
```

### Kafka Transactions

```scala
import org.apache.kafka.clients.producer.*
import org.apache.kafka.clients.consumer.*

// Transactional producer
val producerProps = new java.util.Properties()
producerProps.put("bootstrap.servers", "kafka:9092")
producerProps.put("transactional.id", "my-processor-1") // unique per instance
producerProps.put("enable.idempotence", "true")

val producer = new KafkaProducer[String, String](producerProps)
producer.initTransactions()

def processExactlyOnce(
  consumer: KafkaConsumer[String, String],
  inputTopic: String,
  outputTopic: String
): IO[Unit] =
  IO {
    val records = consumer.poll(java.time.Duration.ofMillis(100))
    if !records.isEmpty then
      producer.beginTransaction()
      try
        import scala.jdk.CollectionConverters.*
        
        // process records
        val results = records.asScala.map { record =>
          val processed = processRecord(record.value())
          new ProducerRecord[String, String](outputTopic, record.key(), processed)
        }
        
        // produce output
        results.foreach(producer.send)
        
        // commit offsets atomically with producer transaction
        val offsets = records.asScala.groupBy(_.topic()).map { (topic, recs) =>
          val partitionOffsets = recs.groupBy(_.partition()).map { (partition, partRecs) =>
            new org.apache.kafka.common.TopicPartition(topic, partition) ->
            new OffsetAndMetadata(partRecs.map(_.offset()).max + 1)
          }
          partitionOffsets
        }.flatten.toMap
        
        producer.sendOffsetsToTransaction(
          offsets.asJava,
          consumer.groupMetadata()
        )
        
        producer.commitTransaction()
      catch
        case e: Exception =>
          producer.abortTransaction()
          throw e
  }
```

### Flink Exactly-Once ด้วย Checkpointing

```scala
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment
import org.apache.flink.runtime.state.filesystem.FsStateBackend

val env = StreamExecutionEnvironment.getExecutionEnvironment

// Enable checkpointing
env.enableCheckpointing(30000) // checkpoint every 30 seconds

// Checkpoint configuration
val checkpointConfig = env.getCheckpointConfig
checkpointConfig.setCheckpointingMode(
  org.apache.flink.streaming.api.CheckpointingMode.EXACTLY_ONCE
)
checkpointConfig.setMinPauseBetweenCheckpoints(5000)
checkpointConfig.setCheckpointTimeout(60000)
checkpointConfig.setMaxConcurrentCheckpoints(1)
checkpointConfig.enableExternalizedCheckpoints(
  org.apache.flink.streaming.api.environment.CheckpointConfig
    .ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION
)

// State backend (S3 หรือ HDFS สำหรับ production)
env.setStateBackend(new FsStateBackend("s3://my-bucket/flink-checkpoints"))
```

---

## State Management ใน Streaming

### fs2 Stateful Streams

```scala
import fs2.Stream
import cats.effect.{IO, Ref}
import scala.concurrent.duration.*

// Stateful stream processing ด้วย fs2
case class StreamState(
  windowStart: Long,
  events: List[Event],
  totalValue: Double
)

def processWithState(
  events: Stream[IO, Event],
  windowDuration: FiniteDuration
): Stream[IO, WindowResult] =
  // ใช้ scan สำหรับ stateful transformation
  events
    .scan(StreamState(0L, List.empty, 0.0)) { (state, event) =>
      val eventMs = event.eventTime.toEpochMilli
      val windowMs = windowDuration.toMillis
      
      // Check ว่า event อยู่ใน window เดิมหรือไม่
      if state.windowStart == 0L || eventMs < state.windowStart + windowMs then
        // Add to current window
        val start = if state.windowStart == 0L then
          (eventMs / windowMs) * windowMs
        else state.windowStart
        
        state.copy(
          windowStart = start,
          events = event :: state.events,
          totalValue = state.totalValue + event.value
        )
      else
        // Start new window
        val newStart = (eventMs / windowMs) * windowMs
        StreamState(newStart, List(event), event.value)
    }
    .zipWithNext
    .collect {
      case (state, Some(next)) if next.windowStart != state.windowStart =>
        WindowResult(state.windowStart, state.events.size, state.totalValue)
      case (state, None) if state.events.nonEmpty =>
        WindowResult(state.windowStart, state.events.size, state.totalValue)
    }

case class WindowResult(windowStart: Long, count: Int, total: Double)
```

### Redis-backed State

```scala
import dev.profunktor.redis4cats.*
import dev.profunktor.redis4cats.effects.*
import cats.effect.IO

// Distributed state ด้วย Redis
class RedisStateStore(redis: RedisCommands[IO, String, String]):
  def getWindowState(key: String): IO[Option[WindowState]] =
    for
      countStr <- redis.get(s"window:$key:count")
      totalStr <- redis.get(s"window:$key:total")
      startStr <- redis.get(s"window:$key:start")
    yield
      for
        count <- countStr.flatMap(_.toIntOption)
        total <- totalStr.flatMap(_.toDoubleOption)
        start <- startStr.flatMap(_.toLongOption)
      yield WindowState(start, count, total)
  
  def updateWindowState(key: String, event: Event): IO[WindowState] =
    for
      currentState <- getWindowState(key)
      newState = currentState match
        case None =>
          WindowState(event.eventTime.toEpochMilli, 1, event.value)
        case Some(state) =>
          state.copy(count = state.count + 1, total = state.total + event.value)
      
      _ <- redis.set(s"window:$key:count", newState.count.toString)
      _ <- redis.set(s"window:$key:total", newState.total.toString)
      _ <- redis.set(s"window:$key:start", newState.start.toString)
      _ <- redis.expire(s"window:$key:count", 3600) // expire after 1 hour
    yield newState

case class WindowState(start: Long, count: Int, total: Double)
```

---

## Kafka Streams กับ Scala

### Kafka Streams Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.kafka" % "kafka-streams" % "3.6.0",
  "org.apache.kafka" %% "kafka-streams-scala" % "3.6.0"
)
```

```scala
import org.apache.kafka.streams.scala.*
import org.apache.kafka.streams.scala.kstream.*
import org.apache.kafka.streams.scala.ImplicitConversions.*
import org.apache.kafka.streams.scala.serialization.Serdes.*
import org.apache.kafka.streams.{KafkaStreams, StreamsConfig, Topology}
import java.util.Properties

// Kafka Streams DSL ใน Scala
object ClickStreamProcessor:
  def buildTopology(): Topology =
    val builder = new StreamsBuilder()
    
    // Input stream
    val clickStream: KStream[String, String] = builder.stream[String, String]("clicks")
    
    // Parse JSON
    val parsedClicks: KStream[String, ClickEvent] = clickStream
      .filter { (_, value) => value != null }
      .mapValues { json =>
        try Some(upickle.default.read[ClickEvent](json))
        catch case _ => None
      }
      .filter { (_, opt) => opt.isDefined }
      .mapValues(_.get)
    
    // Repartition by URL
    val byUrl: KStream[String, ClickEvent] = parsedClicks
      .selectKey { (_, event) => event.url }
    
    // Count clicks per URL in 1-minute windows
    val urlCounts = byUrl
      .groupByKey
      .windowedBy(TimeWindows.ofSizeWithNoGrace(java.time.Duration.ofMinutes(1)))
      .count()
    
    // Output to results topic
    urlCounts.toStream
      .map { (windowedKey, count) =>
        val url = windowedKey.key()
        val windowEnd = windowedKey.window().end()
        val value = upickle.default.write(UrlClickCount(url, windowEnd, count))
        (url, value)
      }
      .to("url-click-counts")
    
    // Alert on high-traffic URLs
    val highTrafficUrls = urlCounts.toStream
      .filter { (_, count) => count > 1000 }
      .map { (windowedKey, count) =>
        val alert = Alert(windowedKey.key(), count, "High traffic!")
        (windowedKey.key(), upickle.default.write(alert))
      }
    
    highTrafficUrls.to("traffic-alerts")
    
    builder.build()
  
  def start(bootstrapServers: String): KafkaStreams =
    val props = new Properties()
    props.put(StreamsConfig.APPLICATION_ID_CONFIG, "click-processor")
    props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers)
    props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String.getClass)
    props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String.getClass)
    props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2)
    
    val topology = buildTopology()
    val streams = new KafkaStreams(topology, props)
    
    streams.start()
    
    // Graceful shutdown
    Runtime.getRuntime.addShutdownHook(new Thread(() => streams.close()))
    
    streams

case class UrlClickCount(url: String, windowEnd: Long, count: Long)
case class Alert(url: String, count: Long, message: String)
```

---

## Complete Stream Processing Design

### Real-time Analytics Platform

```scala
// Complete design: Real-time E-commerce Analytics

// 1. Data Models
case class OrderEvent(
  orderId: String,
  userId: String,
  productId: String,
  quantity: Int,
  price: Double,
  category: String,
  timestamp: Long,
  region: String
)

case class RevenueMetric(
  windowStart: Long,
  windowEnd: Long,
  region: String,
  category: String,
  totalRevenue: Double,
  orderCount: Long,
  avgOrderValue: Double
)

case class UserActivity(
  userId: String,
  orderCount: Int,
  totalSpend: Double,
  lastOrderTime: Long,
  preferredCategory: String
)

// 2. Stream Processors

// Revenue aggregation processor
class RevenueProcessor(kafka: KafkaConfig):
  import org.apache.kafka.streams.scala.*
  import org.apache.kafka.streams.scala.kstream.*
  
  def build(): Topology =
    val builder = new StreamsBuilder()
    
    val orders: KStream[String, OrderEvent] = builder
      .stream[String, String]("orders")
      .mapValues(parseOrderEvent)
      .filter((_, opt) => opt.isDefined)
      .mapValues(_.get)
    
    // Revenue by region and category
    orders
      .groupBy { (_, order) => s"${order.region}:${order.category}" }
      .windowedBy(TimeWindows.ofSizeWithNoGrace(java.time.Duration.ofMinutes(5)))
      .aggregate(
        () => RevenueAggregate(0L, 0.0),
        (_, order, agg) => RevenueAggregate(
          agg.count + 1,
          agg.total + order.price * order.quantity
        )
      )
      .toStream
      .mapValues { (windowedKey, agg) =>
        val parts = windowedKey.key().split(":")
        RevenueMetric(
          windowStart = windowedKey.window().start(),
          windowEnd = windowedKey.window().end(),
          region = parts(0),
          category = parts(1),
          totalRevenue = agg.total,
          orderCount = agg.count,
          avgOrderValue = if agg.count > 0 then agg.total / agg.count else 0
        )
      }
      .to("revenue-metrics")
    
    // Anomaly detection: unusually large orders
    orders
      .filter { (_, order) => order.price * order.quantity > 10000 }
      .mapValues { order =>
        s"Large order detected: ${order.orderId}, amount: ${order.price * order.quantity}"
      }
      .to("anomalies")
    
    builder.build()

case class RevenueAggregate(count: Long, total: Double)

// 3. Materialized Views ด้วย Interactive Queries
class RevenueQueryService(streams: KafkaStreams):
  def getRevenueByRegion(region: String): Map[String, Double] =
    val store = streams.store(
      StoreQueryParameters.fromNameAndType(
        "revenue-store",
        QueryableStoreTypes.keyValueStore[String, RevenueAggregate]()
      )
    )
    
    import scala.jdk.CollectionConverters.*
    store.all().asScala
      .filter { kv => kv.key.startsWith(s"$region:") }
      .map { kv =>
        val category = kv.key.split(":")(1)
        category -> kv.value.total
      }
      .toMap

// 4. Real-time Dashboard ด้วย HTTP/SSE

import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.server.middleware.CORS
import fs2.Stream
import cats.effect.{IO, Ref}
import scala.concurrent.duration.*

class DashboardService(
  metricsStore: Ref[IO, Map[String, RevenueMetric]]
):
  // Server-Sent Events endpoint
  val routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case GET -> Root / "metrics" / "stream" =>
      val sseStream: Stream[IO, ServerSentEvent] = 
        Stream
          .fixedRate[IO](1.second)
          .evalMap { _ => metricsStore.get }
          .map { metrics =>
            ServerSentEvent(
              data = Some(metrics.values.toList.asJson.noSpaces),
              eventType = Some("metrics-update")
            )
          }
      
      Ok(sseStream)
    
    case GET -> Root / "metrics" / "latest" =>
      metricsStore.get.flatMap { metrics =>
        Ok(metrics.values.toList.asJson)
      }
    
    case GET -> Root / "metrics" / "region" / region =>
      metricsStore.get.flatMap { metrics =>
        val regionMetrics = metrics.values
          .filter(_.region == region)
          .toList
        Ok(regionMetrics.asJson)
      }
  }

// 5. Deployment Configuration

object DeploymentConfig:
  // Kubernetes deployment สำหรับ stream processor
  val k8sDeployment = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stream-processor
spec:
  replicas: 3
  selector:
    matchLabels:
      app: stream-processor
  template:
    metadata:
      labels:
        app: stream-processor
    spec:
      containers:
      - name: processor
        image: stream-processor:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          valueFrom:
            secretKeyRef:
              name: kafka-config
              key: bootstrap-servers
        - name: JAVA_OPTS
          value: "-Xmx1g -XX:+UseG1GC"
"""

// 6. Monitoring

class StreamingMetrics:
  import io.prometheus.client.*
  
  val processedEvents = Counter.build()
    .name("stream_processed_events_total")
    .help("Total number of processed events")
    .labelNames("topic", "status")
    .register()
  
  val processingLatency = Histogram.build()
    .name("stream_processing_latency_seconds")
    .help("Event processing latency in seconds")
    .labelNames("topic")
    .buckets(0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0)
    .register()
  
  val windowLag = Gauge.build()
    .name("stream_window_lag_seconds")
    .help("How far behind the processing window is")
    .register()
  
  def recordEvent(topic: String, success: Boolean, latencyMs: Double): Unit =
    val status = if success then "success" else "failure"
    processedEvents.labels(topic, status).inc()
    processingLatency.labels(topic).observe(latencyMs / 1000.0)
```

---

## สรุป

Distributed Streaming Systems ที่ดีต้องประกอบด้วย:

| Component | ทางเลือก | Use Case |
|-----------|---------|---------|
| Message Queue | Kafka, Kinesis, Pulsar | Event transport |
| Stream Processor | Flink, Kafka Streams, Spark | Processing logic |
| State Store | RocksDB, Redis, Cassandra | Stateful operations |
| Serving Layer | Redis, Elasticsearch | Query results |
| Monitoring | Prometheus, Grafana | Observability |

### Key Principles

```
1. Exactly-once ต้องการ: idempotent producers + transactions
2. Watermarks ช่วย handle late data
3. Checkpointing ช่วย fault tolerance
4. Back-pressure ป้องกัน consumer overload
5. Schema evolution ต้องคิดล่วงหน้า
```

---

*[← ส่วนที่ 81: Functional Design Patterns](part-81-fp-design.md) | [ส่วนที่ 83: Machine Learning with Scala →](part-83-ml-scala.md)*
