# Part 62: Akka Streams

## สารบัญ
1. [Akka Streams คืออะไร](#akka-streams-คืออะไร)
2. [Source, Flow, Sink](#source-flow-sink)
3. [Backpressure Mechanics](#backpressure-mechanics)
4. [Graph DSL สำหรับ Complex Topologies](#graph-dsl-สำหรับ-complex-topologies)
5. [Integration กับ Kafka](#integration-กับ-kafka)
6. [Error Handling: Restart และ Supervision](#error-handling-restart-และ-supervision)
7. [ตัวอย่าง Streaming Pipeline ครบถ้วน](#ตัวอย่าง-streaming-pipeline-ครบถ้วน)
8. [สรุป](#สรุป)

---

## Akka Streams คืออะไร

Akka Streams เป็น implementation ของ Reactive Streams specification ที่สร้างบน Akka Actor System โดยมีเป้าหมายหลักคือ:

- **Backpressure**: ควบคุม flow rate เพื่อป้องกัน fast producer ท่วม slow consumer
- **Composability**: ประกอบ pipeline จาก components ที่ reuse ได้
- **Type Safety**: ทุกขั้นตอนมี type ที่ชัดเจน
- **Resource Management**: จัดการ resources อัตโนมัติ

### Reactive Streams Protocol

```
Reactive Streams: async message passing with backpressure

Publisher              Subscriber
    │                      │
    │◄─── request(n) ──────│  ← subscriber บอกว่ารับได้อีก n elements
    │                      │
    │──── onNext(x1) ──────►│
    │──── onNext(x2) ──────►│
    │──── onNext(x3) ──────►│  ← publish ได้ไม่เกิน n elements
    │                      │
    │◄─── request(m) ──────│  ← ขอเพิ่ม
    │                      │
    │──── onComplete() ─────►│  หรือ
    │──── onError(ex) ──────►│
```

### ติดตั้ง Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-stream"        % "2.9.3",
  "com.typesafe.akka" %% "akka-stream-testkit" % "2.9.3" % Test,
  // สำหรับ Kafka integration
  "com.typesafe.akka" %% "akka-stream-kafka"   % "6.0.0",
)
```

---

## Source, Flow, Sink

### Building Blocks

```
Stream Pipeline:
┌────────────┐    ┌──────────────┐    ┌────────────┐
│   Source   │───►│     Flow     │───►│    Sink    │
│ (producer) │    │(transformer) │    │ (consumer) │
└────────────┘    └──────────────┘    └────────────┘
Source[Out, Mat]  Flow[In, Out, Mat]  Sink[In, Mat]
```

### Source - แหล่งข้อมูล

```scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.stream.scaladsl.*
import akka.stream.*
import scala.concurrent.{Future, ExecutionContext}

given system: ActorSystem[Nothing] = ActorSystem(Behaviors.empty, "streams")
given ec: ExecutionContext = system.executionContext

// Source จากค่าตรงๆ
val s1: Source[Int, NotUsed] = Source(1 to 10)
val s2: Source[String, NotUsed] = Source(List("a", "b", "c"))

// Source จาก Future
val s3: Source[String, NotUsed] = Source.future(Future.successful("hello"))

// Source จาก Iterator
val s4: Source[Int, NotUsed] = Source.fromIterator(() => Iterator.continually(scala.util.Random.nextInt(100)))

// Source tick (เหมือน interval)
import scala.concurrent.duration.*
val s5: Source[Int, akka.actor.Cancellable] =
  Source.tick(0.seconds, 1.second, 1)
    .scan(0)(_ + _)  // running total

// Source จากไฟล์
val s6: Source[akka.util.ByteString, Future[IOResult]] =
  FileIO.fromPath(java.nio.file.Paths.get("data.txt"))
```

### Flow - การแปลงข้อมูล

```scala
// Flow พื้นฐาน
val double: Flow[Int, Int, NotUsed] =
  Flow[Int].map(_ * 2)

val filterEven: Flow[Int, Int, NotUsed] =
  Flow[Int].filter(_ % 2 == 0)

val toString: Flow[Int, String, NotUsed] =
  Flow[Int].map(n => s"Number: $n")

// Combine flows
val transform: Flow[Int, String, NotUsed] =
  double.via(filterEven).via(toString)

// Async flow
val asyncTransform: Flow[Int, Int, NotUsed] =
  Flow[Int].mapAsync(parallelism = 4) { n =>
    Future {
      Thread.sleep(100)
      n * 3
    }
  }

// Sliding window
val windows: Flow[Int, Seq[Int], NotUsed] =
  Flow[Int].sliding(n = 3, step = 1)

// Grouped
val grouped: Flow[Int, Seq[Int], NotUsed] =
  Flow[Int].grouped(5)

// Throttle
val throttled: Flow[Int, Int, NotUsed] =
  Flow[Int].throttle(
    elements = 10,
    per = 1.second,
    maximumBurst = 5,
    mode = ThrottleMode.Shaping
  )
```

### Sink - ปลายทาง

```scala
// Sink.foreach
val printSink: Sink[String, Future[Done]] =
  Sink.foreach(println)

// Sink.seq - เก็บทั้งหมดใน List
val collectSink: Sink[Int, Future[Seq[Int]]] =
  Sink.seq[Int]

// Sink.fold - aggregate
val sumSink: Sink[Int, Future[Int]] =
  Sink.fold(0)(_ + _)

// Sink.ignore - ทิ้งทั้งหมด
val ignoreSink: Sink[Any, Future[Done]] =
  Sink.ignore

// Sink ไปไฟล์
val fileSink: Sink[akka.util.ByteString, Future[IOResult]] =
  FileIO.toPath(java.nio.file.Paths.get("output.txt"))
```

### ประกอบ Pipeline

```scala
// via = เชื่อม Source กับ Flow
// to  = เชื่อม Flow/Source กับ Sink
// run = รัน pipeline

val result: Future[Int] =
  Source(1 to 100)
    .via(Flow[Int].filter(_ % 3 == 0))
    .via(Flow[Int].map(_ * 2))
    .toMat(Sink.fold(0)(_ + _))(Keep.right)
    .run()

result.foreach(sum => println(s"Sum of doubled multiples of 3: $sum"))
// 3,6,9,...,99 → doubled → summed

// Shorthand methods
val result2: Future[Int] =
  Source(1 to 100)
    .filter(_ % 3 == 0)
    .map(_ * 2)
    .runFold(0)(_ + _)

// runWith - เลือก materialized value ของ Sink
val result3: Future[Seq[Int]] =
  Source(1 to 10)
    .filter(_ % 2 == 0)
    .runWith(Sink.seq)
```

### Materialized Values

```scala
// Materialized value คือค่าที่ stream คืนให้เมื่อ materialize
// Source[Out, Mat] - Mat คือ materialized value type

val (cancellable, done): (akka.actor.Cancellable, Future[Done]) =
  Source.tick(0.seconds, 100.millis, "tick")
    .take(5)
    .toMat(Sink.foreach(println))(Keep.both)
    .run()

// Cancel stream หลัง 300ms
Thread.sleep(300)
cancellable.cancel()
done.foreach(_ => println("Stream completed"))
```

---

## Backpressure Mechanics

### ปัญหาที่ Backpressure แก้

```scala
// Fast producer, slow consumer
// ถ้าไม่มี backpressure: OutOfMemoryError!

val fastProducer: Source[Int, NotUsed] =
  Source.fromIterator(() => Iterator.from(0))  // ผลิตเร็วมาก

val slowConsumer: Sink[Int, Future[Done]] =
  Sink.foreach { n =>
    Thread.sleep(100)  // ช้ามาก
    println(s"Processed: $n")
  }

// Akka Streams จัดการ backpressure อัตโนมัติ
// Consumer จะ request เมื่อพร้อม, Producer จะหยุดรอ
fastProducer
  .take(5)
  .runWith(slowConsumer)
```

### Buffer Strategies

```scala
import akka.stream.OverflowStrategy

// Buffer.dropHead - ทิ้ง element เก่าสุด
val dropHead: Flow[Int, Int, NotUsed] =
  Flow[Int].buffer(100, OverflowStrategy.dropHead)

// Buffer.dropTail - ทิ้ง element ใหม่สุด
val dropTail: Flow[Int, Int, NotUsed] =
  Flow[Int].buffer(100, OverflowStrategy.dropTail)

// Buffer.dropNew - ทิ้ง element ใหม่เมื่อ buffer เต็ม
val dropNew: Flow[Int, Int, NotUsed] =
  Flow[Int].buffer(100, OverflowStrategy.dropNew)

// Buffer.backpressure - default: block producer
val backpressure: Flow[Int, Int, NotUsed] =
  Flow[Int].buffer(100, OverflowStrategy.backpressure)

// Buffer.fail - throw exception เมื่อ buffer เต็ม
val fail: Flow[Int, Int, NotUsed] =
  Flow[Int].buffer(100, OverflowStrategy.fail)
```

### Conflate - ลด rate โดย merge elements

```scala
// Conflate จะ aggregate elements ที่ยังไม่ได้ consume
val conflate: Flow[Int, Int, NotUsed] =
  Flow[Int].conflate(_ + _)  // sum elements ที่ backup

// ตัวอย่าง: sensor ส่งค่าเร็ว แต่เราสนใจแค่ average
val sensorFlow: Flow[Double, Double, NotUsed] =
  Flow[Double]
    .conflateWithSeed(seed = value => (value, 1)) { case ((sum, count), value) =>
      (sum + value, count + 1)
    }
    .map { case (sum, count) => sum / count }
```

### Expand - เพิ่ม rate สำหรับ slow producer

```scala
// Expand เมื่อ consumer เร็วกว่า producer
// ส่งค่าเดิมซ้ำๆ จนกว่า producer จะผลิตใหม่
val expand: Flow[Int, Int, NotUsed] =
  Flow[Int].expand(Iterator.continually(_))
```

---

## Graph DSL สำหรับ Complex Topologies

### Fan-out: Broadcast

```scala
import akka.stream.scaladsl.GraphDSL
import akka.stream.{ClosedShape, FanOutShape2}

// Broadcast: ส่ง element เดียวกันไปหลาย outputs
val broadcastGraph = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
  import GraphDSL.Implicits.*

  val source = Source(1 to 10)
  val broadcast = builder.add(Broadcast[Int](2))  // 2 outputs
  val sink1 = Sink.foreach[Int](n => println(s"Sink1: $n"))
  val sink2 = Sink.foreach[Int](n => println(s"Sink2: ${n * 10}"))

  source ~> broadcast.in
  broadcast.out(0) ~> sink1
  broadcast.out(1) ~> sink2

  ClosedShape
})

broadcastGraph.run()
```

### Fan-in: Merge

```scala
// Merge: รวมหลาย sources เป็น stream เดียว
val mergeGraph = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
  import GraphDSL.Implicits.*

  val source1 = Source(List("a", "b", "c"))
  val source2 = Source(List("x", "y", "z"))
  val merge = builder.add(Merge[String](2))  // 2 inputs
  val sink = Sink.foreach[String](println)

  source1 ~> merge.in(0)
  source2 ~> merge.in(1)
  merge.out ~> sink

  ClosedShape
})
```

### Zip and ZipWith

```scala
// Zip: จับคู่ elements จาก 2 streams
val zipGraph = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
  import GraphDSL.Implicits.*

  val s1 = Source(1 to 5)
  val s2 = Source(List("a", "b", "c", "d", "e"))
  val zip = builder.add(Zip[Int, String]())
  val sink = Sink.foreach[(Int, String)] { case (n, s) =>
    println(s"$n -> $s")
  }

  s1 ~> zip.in0
  s2 ~> zip.in1
  zip.out ~> sink

  ClosedShape
})
```

### Balance - Round-robin Fan-out

```scala
// Balance: กระจาย load แบบ round-robin
val balanceGraph = RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
  import GraphDSL.Implicits.*

  val source = Source(1 to 100)
  val balance = builder.add(Balance[Int](3))  // 3 workers

  def worker(id: Int): Sink[Int, Future[Done]] =
    Sink.foreach[Int] { n =>
      Thread.sleep(scala.util.Random.nextInt(100))
      println(s"Worker $id processed: $n")
    }

  source ~> balance.in
  balance.out(0) ~> worker(1)
  balance.out(1) ~> worker(2)
  balance.out(2) ~> worker(3)

  ClosedShape
})
```

### Complex Graph: Partial Graphs

```scala
// สร้าง reusable graph component
val processingStage: Flow[String, String, NotUsed] =
  Flow.fromGraph(GraphDSL.create() { implicit builder =>
    import GraphDSL.Implicits.*

    val broadcast = builder.add(Broadcast[String](2))
    val merge = builder.add(Merge[String](2))

    val uppercase = Flow[String].map(_.toUpperCase)
    val exclaim   = Flow[String].map(s => s"$s!")

    // Split, process, merge
    broadcast.out(0) ~> uppercase ~> merge.in(0)
    broadcast.out(1) ~> exclaim   ~> merge.in(1)

    FlowShape(broadcast.in, merge.out)
  })

// ใช้ component ที่สร้าง
Source(List("hello", "world"))
  .via(processingStage)
  .runForeach(println)
// HELLO
// hello!
// WORLD
// world!
```

### Unzip Pattern

```scala
// ประมวลผล 2 types แยกกันแล้วรวม
case class Event(id: Int, eventType: String, data: String)

val processEvents: RunnableGraph[NotUsed] =
  RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
    import GraphDSL.Implicits.*

    val source = Source(List(
      Event(1, "click", "button-1"),
      Event(2, "view", "page-home"),
      Event(3, "click", "link-about"),
      Event(4, "purchase", "item-42")
    ))

    val partition = builder.add(Partition[Event](2, e =>
      if e.eventType == "click" then 0 else 1
    ))

    val clickSink = Sink.foreach[Event](e => println(s"Click: ${e.data}"))
    val otherSink = Sink.foreach[Event](e => println(s"Other(${e.eventType}): ${e.data}"))

    source ~> partition.in
    partition.out(0) ~> clickSink
    partition.out(1) ~> otherSink

    ClosedShape
  })
```

---

## Integration กับ Kafka

### Alpakka Kafka (akka-stream-kafka)

```scala
// build.sbt
// "com.typesafe.akka" %% "akka-stream-kafka" % "6.0.0"

import akka.kafka.*
import akka.kafka.scaladsl.*
import org.apache.kafka.clients.consumer.ConsumerConfig
import org.apache.kafka.clients.producer.ProducerRecord
import org.apache.kafka.common.serialization.*
```

### Kafka Consumer Source

```scala
val consumerSettings: ConsumerSettings[String, String] =
  ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
    .withBootstrapServers("localhost:9092")
    .withGroupId("my-group")
    .withProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")

// Plain Consumer - manual offset control
val plainConsumer: Source[ConsumerRecord[String, String], Consumer.Control] =
  Consumer.plainSource(
    consumerSettings,
    Subscriptions.topics("my-topic")
  )

// Committable Consumer - เพื่อ at-least-once processing
val committableConsumer = Consumer.committableSource(
  consumerSettings,
  Subscriptions.topics("events-topic")
)

// ประมวลผลและ commit
committableConsumer
  .mapAsync(10) { msg =>
    // ประมวลผล
    val result = processEvent(msg.record.value())
    // commit offset
    result.flatMap(_ => msg.committableOffset.commitScaladsl())
  }
  .runWith(Sink.ignore)

def processEvent(value: String): Future[Unit] =
  Future { println(s"Processing: $value") }
```

### Kafka Producer Sink

```scala
val producerSettings: ProducerSettings[String, String] =
  ProducerSettings(system, new StringSerializer, new StringSerializer)
    .withBootstrapServers("localhost:9092")

// Producer Sink
val kafkaProducer: Sink[ProducerRecord[String, String], Future[Done]] =
  Producer.plainSink(producerSettings)

// ส่ง messages
Source(1 to 100)
  .map { n =>
    new ProducerRecord[String, String](
      "output-topic",
      s"key-$n",
      s"""{"id": $n, "value": "${n * 2}"}"""
    )
  }
  .runWith(kafkaProducer)
```

### Consumer-Transform-Producer Pipeline

```scala
def kafkaPipeline(): Future[Done] =
  Consumer.committableSource(consumerSettings, Subscriptions.topics("input"))
    .mapAsync(4) { msg =>
      Future {
        // Transform: parse JSON, process, re-serialize
        val input = msg.record.value()
        val output = s"""{"processed": $input, "ts": ${System.currentTimeMillis()}}"""
        (output, msg.committableOffset)
      }
    }
    .map { case (value, offset) =>
      ProducerMessage.single(
        new ProducerRecord[String, String]("output", value),
        passThrough = offset
      )
    }
    .via(Producer.flexiFlow(producerSettings))
    .mapAsync(4)(_.passThrough.commitScaladsl())
    .runWith(Sink.ignore)
```

### Kafka กับ Backpressure

```scala
// Control consumer rate ด้วย throttle
Consumer.plainSource(consumerSettings, Subscriptions.topics("fast-topic"))
  .throttle(100, 1.second)  // ไม่เกิน 100 messages/second
  .mapAsync(8)(msg => Future { processSlowly(msg.value()) })
  .runWith(Sink.ignore)

def processSlowly(msg: String): String = {
  Thread.sleep(50)
  msg.toUpperCase
}
```

---

## Error Handling: Restart และ Supervision

### Stream Supervision

```scala
import akka.stream.ActorAttributes
import akka.stream.Supervision

// Supervision strategy สำหรับ stream
val decider: Supervision.Decider = {
  case _: NumberFormatException =>
    println("Skip bad number")
    Supervision.Resume  // ข้าม element นี้, ไม่หยุด stream

  case _: IllegalArgumentException =>
    println("Restarting after illegal arg")
    Supervision.Restart  // restart stream (state reset)

  case ex =>
    println(s"Fatal: ${ex.getMessage}")
    Supervision.Stop     // หยุด stream
}

val result: Future[Seq[Int]] =
  Source(List("1", "2", "oops", "4", "5"))
    .map(_.toInt)  // จะ throw NumberFormatException สำหรับ "oops"
    .withAttributes(ActorAttributes.supervisionStrategy(decider))
    .runWith(Sink.seq)

result.foreach(nums => println(s"Got: $nums"))
// Skip bad number
// Got: Vector(1, 2, 4, 5)
```

### RestartSource

```scala
import akka.stream.RestartSettings
import akka.stream.scaladsl.RestartSource

val restartSettings = RestartSettings(
  minBackoff   = 100.millis,
  maxBackoff   = 10.seconds,
  randomFactor = 0.2
).withMaxRestarts(10, 1.minute)

// Source ที่ restart อัตโนมัติเมื่อล้มเหลว
val resilientSource: Source[String, NotUsed] =
  RestartSource.withBackoff(restartSettings) { () =>
    createKafkaSource()  // อาจ fail เพราะ Kafka unavailable
  }

def createKafkaSource(): Source[String, Consumer.Control] =
  Consumer.plainSource(consumerSettings, Subscriptions.topics("topic"))
    .map(_.value())
```

### RestartFlow

```scala
import akka.stream.scaladsl.RestartFlow

// HTTP call ที่ retry เมื่อ fail
val resilientHttpFlow: Flow[String, String, NotUsed] =
  RestartFlow.withBackoff(restartSettings) { () =>
    Flow[String].mapAsync(4) { url =>
      callExternalApi(url)
    }
  }

def callExternalApi(url: String): Future[String] =
  Future {
    if scala.util.Random.nextBoolean() then s"Response from $url"
    else throw new RuntimeException("API unavailable")
  }
```

### RetryFlow

```scala
import akka.stream.scaladsl.RetryFlow

// Retry individual elements (ไม่ใช่ทั้ง stream)
case class Request(id: Int, url: String)
case class Response(id: Int, body: String)

val retryFlow: Flow[Request, Response, NotUsed] =
  RetryFlow.withBackoff(
    minBackoff = 100.millis,
    maxBackoff = 5.seconds,
    randomFactor = 0.1,
    maxRetries = 3,
    flow = Flow[Request].mapAsync(4) { req =>
      processRequest(req)
        .map(Right(_))
        .recover { case ex => Left((req, ex)) }
    }
  ) {
    case (req, Left((failedReq, ex))) =>
      println(s"Retrying ${req.id}: ${ex.getMessage}")
      Some(failedReq)
    case _ => None
  }

def processRequest(req: Request): Future[Response] =
  Future {
    if scala.util.Random.nextInt(3) == 0 then
      throw new RuntimeException("Transient error")
    Response(req.id, s"OK from ${req.url}")
  }
```

### Timeout Handling

```scala
import scala.concurrent.duration.*

val withTimeout: Flow[String, String, NotUsed] =
  Flow[String]
    .mapAsync(4) { url =>
      callExternalApi(url)
        .recover { case ex =>
          s"Error: ${ex.getMessage}"
        }
    }
    .completionTimeout(30.seconds)  // timeout ทั้ง stream

// Timeout ต่อ element
val elementTimeout: Flow[String, String, NotUsed] =
  Flow[String].mapAsync(4) { url =>
    import akka.pattern.after
    val callFuture = callExternalApi(url)
    val timeoutFuture = after(5.seconds)(
      Future.failed(new RuntimeException(s"Timeout for $url"))
    )
    Future.firstCompletedOf(List(callFuture, timeoutFuture))
  }
```

---

## ตัวอย่าง Streaming Pipeline ครบถ้วน

### Log Processing Pipeline

```scala
package com.example.logprocessing

import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.stream.scaladsl.*
import akka.stream.*
import akka.NotUsed
import akka.util.ByteString
import java.nio.file.Paths
import scala.concurrent.{ExecutionContext, Future}
import java.time.Instant

// Domain
case class LogLine(
  timestamp: Instant,
  level: LogLevel,
  service: String,
  message: String,
  metadata: Map[String, String] = Map.empty
)

enum LogLevel:
  case DEBUG, INFO, WARN, ERROR

case class Alert(
  severity: String,
  service: String,
  count: Int,
  timeWindow: String,
  samples: List[String]
)

case class Metric(
  name: String,
  value: Double,
  labels: Map[String, String]
)

// Parser
object LogParser:
  // Format: 2024-01-15T10:30:00Z [ERROR] service-name: message
  private val pattern = """(\S+) \[(\w+)\] (\S+): (.+)""".r

  def parse(line: String): Option[LogLine] =
    line match
      case pattern(ts, level, service, msg) =>
        for
          t <- scala.util.Try(Instant.parse(ts)).toOption
          l <- scala.util.Try(LogLevel.valueOf(level)).toOption
        yield LogLine(t, l, service, msg)
      case _ => None

// Alert rules
object AlertRules:
  def isAlert(log: LogLine): Boolean =
    log.level == LogLevel.ERROR || log.level == LogLevel.WARN

  def severity(log: LogLine): String =
    log.level match
      case LogLevel.ERROR => "critical"
      case LogLevel.WARN  => "warning"
      case _              => "info"

// Metrics extractor
object MetricsExtractor:
  def extract(log: LogLine): List[Metric] =
    val baseLabels = Map("service" -> log.service, "level" -> log.level.toString)

    val levelMetric = Metric(
      "log_lines_total",
      1.0,
      baseLabels
    )

    // Extract duration if message contains "duration=XXXms"
    val durationMetric = """\bduration=(\d+)ms""".r
      .findFirstMatchIn(log.message)
      .map(m => Metric(
        "operation_duration_ms",
        m.group(1).toDouble,
        baseLabels + ("operation" -> "unknown")
      ))

    levelMetric :: durationMetric.toList

// Main Pipeline
object LogProcessingPipeline:
  def create(
    logDir: String,
    alertSink: Sink[Alert, Future[Done]],
    metricsSink: Sink[Metric, Future[Done]]
  )(using system: ActorSystem[Nothing]): RunnableGraph[NotUsed] =

    given ec: ExecutionContext = system.executionContext

    RunnableGraph.fromGraph(GraphDSL.create() { implicit builder =>
      import GraphDSL.Implicits.*

      // Source: read log file(s)
      val logSource: Source[String, Future[IOResult]] =
        FileIO.fromPath(Paths.get(logDir, "app.log"))
          .via(Framing.delimiter(ByteString("\n"), 4096, allowTruncation = true))
          .map(_.utf8String)

      // Parse log lines
      val parsed: Source[LogLine, Future[IOResult]] =
        logSource
          .map(LogParser.parse)
          .collect { case Some(log) => log }

      // Broadcast: ส่งไปทั้ง alerting และ metrics
      val broadcast = builder.add(Broadcast[LogLine](2))

      // Alert branch
      val alertFlow: Flow[LogLine, Alert, NotUsed] =
        Flow[LogLine]
          .filter(AlertRules.isAlert)
          .groupedWithin(100, 10.seconds)  // batch by time/count
          .map { logs =>
            val byService = logs.groupBy(_.service)
            byService.map { case (service, serviceLogs) =>
              Alert(
                severity = serviceLogs.map(AlertRules.severity).maxBy {
                  case "critical" => 2
                  case "warning"  => 1
                  case _          => 0
                },
                service = service,
                count = serviceLogs.size,
                timeWindow = "10s",
                samples = serviceLogs.take(3).map(_.message)
              )
            }.toList
          }
          .mapConcat(identity)  // flatten

      // Metrics branch
      val metricsFlow: Flow[LogLine, Metric, NotUsed] =
        Flow[LogLine]
          .mapConcat(MetricsExtractor.extract)

      // Wire everything
      parsed ~> broadcast.in
      broadcast.out(0) ~> alertFlow  ~> alertSink
      broadcast.out(1) ~> metricsFlow ~> metricsSink

      ClosedShape
    })

// Alert handler
object AlertHandler:
  def sink(): Sink[Alert, Future[Done]] =
    Sink.foreach { alert =>
      println(f"""
        |ALERT [${alert.severity.toUpperCase}]
        |  Service: ${alert.service}
        |  Count: ${alert.count} in ${alert.timeWindow}
        |  Samples:
        |${alert.samples.map(s => s"    - $s").mkString("\n")}
        |""".stripMargin)
    }

// Metrics handler (e.g., push to Prometheus)
object MetricsHandler:
  private var metrics = Map.empty[String, Double]

  def sink(): Sink[Metric, Future[Done]] =
    Sink.foreach { m =>
      val key = s"${m.name}{${m.labels.map(kv => s"${kv._1}=${kv._2}").mkString(",")}}"
      metrics = metrics.updated(key, metrics.getOrElse(key, 0.0) + m.value)
    }

  def getAll(): Map[String, Double] = metrics

// สร้าง sample log file และรัน
@main def runLogPipeline(): Unit =
  given system: ActorSystem[Nothing] =
    ActorSystem(Behaviors.empty, "log-pipeline")
  given ec: ExecutionContext = system.executionContext

  // สร้าง sample logs
  val sampleLogs = """2024-01-15T10:30:00Z [INFO] api-service: Request received id=123
2024-01-15T10:30:01Z [INFO] api-service: Processing duration=45ms
2024-01-15T10:30:02Z [ERROR] api-service: Database connection failed
2024-01-15T10:30:03Z [WARN] cache-service: Cache miss rate=85%
2024-01-15T10:30:04Z [ERROR] auth-service: Invalid token attempt
2024-01-15T10:30:05Z [INFO] api-service: Request completed duration=120ms
2024-01-15T10:30:06Z [ERROR] api-service: Timeout waiting for downstream
2024-01-15T10:30:07Z [DEBUG] cache-service: Cache cleanup started"""

  val logFile = java.nio.file.Files.createTempFile("test-logs", ".log")
  java.nio.file.Files.write(logFile, sampleLogs.getBytes)
  logFile.toFile.deleteOnExit()

  val pipeline = LogProcessingPipeline.create(
    logFile.getParent.toString,
    AlertHandler.sink(),
    MetricsHandler.sink()
  )

  // Override default log path
  val done = FileIO.fromPath(logFile)
    .via(Framing.delimiter(ByteString("\n"), 4096, allowTruncation = true))
    .map(_.utf8String)
    .map(LogParser.parse)
    .collect { case Some(log) => log }
    .alsoTo(
      Flow[LogLine]
        .filter(AlertRules.isAlert)
        .groupedWithin(10, 5.seconds)
        .map { logs =>
          Alert("critical", logs.headOption.map(_.service).getOrElse("unknown"),
                logs.size, "5s", logs.take(3).map(_.message))
        }
        .to(AlertHandler.sink())
    )
    .mapConcat(MetricsExtractor.extract)
    .runWith(MetricsHandler.sink())

  done.onComplete { _ =>
    println("\n=== Metrics Summary ===")
    MetricsHandler.getAll().foreach { case (k, v) =>
      println(f"$k = $v%.0f")
    }
    system.terminate()
  }
```

---

## สรุป

Akka Streams ให้เครื่องมือที่ครบถ้วนสำหรับการสร้าง streaming pipelines ที่:

- ✅ **Backpressure**: ป้องกัน fast producer ท่วม slow consumer อัตโนมัติ
- ✅ **Composable**: ประกอบ Source, Flow, Sink ได้อย่างยืดหยุ่น
- ✅ **Type Safe**: compile-time type checking ทุกขั้นตอน
- ✅ **Resilient**: restart, retry, supervision strategies ในตัว
- ✅ **Scalable**: ทำงานได้ทั้ง single machine และ distributed

Key concepts:
- **Source**: แหล่งข้อมูล (0 inputs, 1 output)
- **Flow**: การแปลงข้อมูล (1 input, 1 output)
- **Sink**: ปลายทาง (1 input, 0 outputs)
- **Graph DSL**: สำหรับ fan-out, fan-in, complex topologies
- **Materialized values**: ค่าที่ stream คืนเมื่อรัน

---

*[← Part 61: Akka Actors](part-61-akka-actors.md) | [Part 63: ZIO →](part-63-zio.md)*
