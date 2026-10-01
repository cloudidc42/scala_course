# ส่วนที่ 55: Reactive Streams และ Back-pressure

## สารบัญ

1. [Reactive Streams Specification](#reactive-streams-specification)
2. [Back-pressure คืออะไร](#back-pressure-คืออะไร)
3. [fs2 และ Back-pressure Mechanism](#fs2-และ-back-pressure-mechanism)
4. [Publisher/Subscriber Pattern](#publishersubscriber-pattern)
5. [Stream Throttling และ Batching](#stream-throttling-และ-batching)
6. [Rate-limited Stream Processing](#rate-limited-stream-processing)
7. [การจัดการ Errors ใน Reactive Streams](#การจัดการ-errors-ใน-reactive-streams)
8. [การรวม Multiple Streams](#การรวม-multiple-streams)
9. [Complete Reactive Pipeline](#complete-reactive-pipeline)
10. [Performance Tuning](#performance-tuning)
11. [สรุป](#สรุป)

---

## Reactive Streams Specification

Reactive Streams เป็น specification ที่กำหนดมาตรฐานสำหรับ asynchronous stream processing พร้อม non-blocking back-pressure ใน JVM ecosystem

### หลักการพื้นฐาน 4 ประการ

```
Publisher  ──produces──>  Subscriber
    ^                         |
    |      <──demand───────   |
    |      (back-pressure)    |
    └─────────────────────────┘
```

### ส่วนประกอบหลักของ Reactive Streams Spec

```scala
// reactive-streams interfaces (simplified)
trait Publisher[T]:
  def subscribe(subscriber: Subscriber[T]): Unit

trait Subscriber[T]:
  def onSubscribe(subscription: Subscription): Unit
  def onNext(item: T): Unit
  def onError(throwable: Throwable): Unit
  def onComplete(): Unit

trait Subscription:
  def request(n: Long): Unit  // demand signal
  def cancel(): Unit

trait Processor[T, R] extends Subscriber[T] with Publisher[R]
```

### ทำไมต้องมี Back-pressure?

```
ปัญหาที่เกิดขึ้นโดยไม่มี back-pressure:

Producer (1,000,000 items/sec)  ──────>  Consumer (100 items/sec)
                                              |
                                         Memory overflow!
                                         OutOfMemoryError!

ด้วย back-pressure:

Producer  <── request(100) ──  Consumer
Producer  ──── 100 items ──>  Consumer
Producer  <── request(100) ──  Consumer
(ควบคุมอัตราการผลิตตาม capacity ของ consumer)
```

### Dependencies สำหรับ Project

```scala
// build.sbt
libraryDependencies ++= Seq(
  "co.fs2"         %% "fs2-core"           % "3.9.4",
  "co.fs2"         %% "fs2-io"             % "3.9.4",
  "co.fs2"         %% "fs2-reactive-streams" % "3.9.4",
  "org.typelevel"  %% "cats-effect"        % "3.5.4",
  "org.typelevel"  %% "cats-effect-kernel" % "3.5.4",
  "org.typelevel"  %% "cats-effect-std"    % "3.5.4",
  "io.monix"       %% "monix-reactive"     % "3.4.1",
  "com.typesafe.akka" %% "akka-stream"     % "2.8.5"
)
```

---

## Back-pressure คืออะไร

Back-pressure คือกลไกที่ให้ consumer บอก producer ว่าสามารถรับข้อมูลได้เท่าไร ป้องกันไม่ให้ producer ส่งข้อมูลเร็วเกินกว่า consumer จะรับได้

### ตัวอย่างปัญหาที่เกิดขึ้นโดยไม่มี back-pressure

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object BackpressureProblem extends IOApp.Simple:

  // Producer ที่เร็วมาก
  val fastProducer: Stream[IO, Int] =
    Stream.iterate(0)(_ + 1)

  // Consumer ที่ช้า - จำลองการประมวลผลที่ใช้เวลา
  def slowConsumer(item: Int): IO[Unit] =
    IO.sleep(10.millis) >> IO.println(s"Processed: $item")

  // ปัญหา: ถ้าไม่มี back-pressure จะ buffer ข้อมูลไว้ในหน่วยความจำ
  // และอาจเกิด OOM ได้
  def run: IO[Unit] =
    fastProducer
      .take(100)
      .evalMap(slowConsumer)  // fs2 จัดการ back-pressure ให้อัตโนมัติ
      .compile
      .drain
```

### Back-pressure ทำงานอย่างไรใน fs2

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*
import scala.concurrent.duration.*

object BackpressureDemo extends IOApp.Simple:

  // fs2 ใช้ pull-based model
  // Consumer "ดึง" ข้อมูลจาก Producer
  // Producer ผลิตเฉพาะเมื่อถูกร้องขอ

  def demonstratePullModel: IO[Unit] =
    val stream = Stream
      .iterate(1)(_ + 1)
      .evalMap { n =>
        IO.println(s"Producing: $n") >> IO.pure(n)
      }
      .take(5)
      .evalMap { n =>
        IO.sleep(50.millis) >> IO.println(s"Consuming: $n") >> IO.pure(n)
      }

    stream.compile.drain

  // Bounded Queue เป็นอีกวิธีหนึ่งของ back-pressure
  def demonstrateBoundedQueue: IO[Unit] =
    for
      queue <- Queue.bounded[IO, Int](10)  // buffer size = 10
      
      producer = Stream
        .iterate(1)(_ + 1)
        .take(50)
        .evalMap { n =>
          IO.println(s"Enqueuing: $n") >> queue.offer(n)
        }
      
      consumer = Stream
        .fromQueueUnterminated(queue)
        .evalMap { n =>
          IO.sleep(20.millis) >> IO.println(s"Dequeuing: $n")
        }
        .take(50)
      
      _ <- producer.concurrently(consumer).compile.drain
    yield ()

  def run: IO[Unit] =
    IO.println("=== Pull Model Demo ===") >>
    demonstratePullModel >>
    IO.println("\n=== Bounded Queue Demo ===") >>
    demonstrateBoundedQueue
```

---

## fs2 และ Back-pressure Mechanism

fs2 (Functional Streams for Scala) ใช้ pull-based model ซึ่งมี back-pressure ในตัว

### Stream Fundamentals

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object Fs2Basics extends IOApp.Simple:

  // 1. Pure streams - ไม่มี side effects
  val numbers: Stream[Pure, Int] = 
    Stream(1, 2, 3, 4, 5)
  
  val range: Stream[Pure, Int] = 
    Stream.range(1, 100)
  
  val infinite: Stream[Pure, Int] = 
    Stream.iterate(0)(_ + 1)

  // 2. Effectful streams
  val effectful: Stream[IO, Int] = 
    Stream.eval(IO.pure(42))

  val periodic: Stream[IO, Unit] = 
    Stream.fixedDelay[IO](1.second)

  // 3. Resource-safe streams
  def resourceStream: Stream[IO, String] =
    Stream.bracket(
      IO.println("Opening resource") >> IO.pure("resource")
    )(
      r => IO.println(s"Closing: $r")
    ).flatMap { r =>
      Stream.emit(s"Using $r")
    }

  // 4. Stream operations
  def streamOps: IO[Unit] =
    Stream
      .range(1, 20)
      // Transformation
      .filter(_ % 2 == 0)
      .map(_ * 3)
      // Chunking สำหรับ performance
      .chunkN(4)
      .flatMap(chunk => Stream.chunk(chunk))
      // Side effects
      .evalTap(n => IO.println(s"Item: $n"))
      // Aggregation
      .fold(0)(_ + _)
      .compile
      .lastOrError
      .flatMap(sum => IO.println(s"Sum: $sum"))

  def run: IO[Unit] = streamOps
```

### Chunk-based Processing

```scala
import cats.effect.*
import fs2.*
import fs2.Chunk

object ChunkProcessing extends IOApp.Simple:

  // Chunks คือ การรวมกลุ่มข้อมูลเพื่อประสิทธิภาพ
  def chunkDemo: IO[Unit] =
    Stream
      .range(1, 101)
      // จัดกลุ่มเป็น chunks ขนาด 10
      .chunkN(10, allowFewer = true)
      .evalMap { chunk =>
        IO.println(s"Processing chunk of ${chunk.size}: ${chunk.toList.mkString(", ")}")
      }
      .compile
      .drain

  // Unchunked vs Chunked processing
  def comparePerformance: IO[Unit] =
    val data = Stream.range(1, 10001)
    
    // แบบ unchunked - process ทีละ item
    val unchunked = data
      .evalMap(n => IO.pure(n * 2))
      .compile
      .toList
    
    // แบบ chunked - process เป็น batch
    val chunked = data
      .chunkN(100)
      .evalMap { chunk =>
        IO.pure(chunk.map(_ * 2))
      }
      .flatMap(Stream.chunk)
      .compile
      .toList
    
    for
      _ <- IO.println("Processing 10000 items...")
      start1 <- IO.monotonic
      _ <- unchunked
      end1 <- IO.monotonic
      _ <- IO.println(s"Unchunked: ${(end1 - start1).toMillis}ms")
      
      start2 <- IO.monotonic
      _ <- chunked
      end2 <- IO.monotonic
      _ <- IO.println(s"Chunked: ${(end2 - start2).toMillis}ms")
    yield ()

  def run: IO[Unit] = 
    IO.println("=== Chunk Demo ===") >>
    chunkDemo >>
    IO.println("\n=== Performance Comparison ===") >>
    comparePerformance
```

### Concurrent Streams

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*
import scala.concurrent.duration.*

object ConcurrentStreams extends IOApp.Simple:

  // mergeN - รวมหลาย streams พร้อมกัน
  def mergeStreams: IO[Unit] =
    val stream1 = Stream
      .iterate(1)(_ + 1)
      .take(5)
      .evalMap(n => IO.sleep(100.millis) >> IO.pure(s"S1: $n"))
    
    val stream2 = Stream
      .iterate(10)(_ + 10)
      .take(5)
      .evalMap(n => IO.sleep(150.millis) >> IO.pure(s"S2: $n"))
    
    val stream3 = Stream
      .iterate(100)(_ + 100)
      .take(5)
      .evalMap(n => IO.sleep(200.millis) >> IO.pure(s"S3: $n"))
    
    stream1
      .merge(stream2)
      .merge(stream3)
      .evalMap(s => IO.println(s))
      .compile
      .drain

  // parEvalMap - parallel evaluation
  def parallelProcessing: IO[Unit] =
    Stream
      .range(1, 11)
      .parEvalMap(maxConcurrent = 3) { n =>
        IO.sleep((100 * n % 5 + 50).millis) >>
        IO.println(s"Completed item $n") >>
        IO.pure(n * n)
      }
      .compile
      .toList
      .flatMap(results => IO.println(s"Results: $results"))

  def run: IO[Unit] =
    IO.println("=== Merge Streams ===") >>
    mergeStreams >>
    IO.println("\n=== Parallel Processing ===") >>
    parallelProcessing
```

---

## Publisher/Subscriber Pattern

### การใช้ Topic สำหรับ Publish/Subscribe

```scala
import cats.effect.*
import cats.effect.std.{Queue, Topic}
import fs2.*
import scala.concurrent.duration.*

object PubSubPattern extends IOApp.Simple:

  // Topic ใน fs2 - broadcast ข้อมูลไปยัง subscribers หลายคน
  def topicDemo: IO[Unit] =
    for
      topic <- Topic[IO, String]
      
      // Publisher
      publisher = Stream
        .iterate(1)(_ + 1)
        .take(10)
        .evalMap { n =>
          IO.sleep(100.millis) >>
          IO.println(s"Publishing: message-$n") >>
          topic.publish1(s"message-$n")
        }
      
      // Subscriber 1 - รับทุก message
      subscriber1 = topic
        .subscribe(maxQueued = 10)
        .evalMap(msg => IO.println(s"Sub1 received: $msg"))
        .take(10)
      
      // Subscriber 2 - filter เฉพาะ message ที่ต้องการ
      subscriber2 = topic
        .subscribe(maxQueued = 10)
        .filter(msg => msg.contains("5") || msg.contains("10"))
        .evalMap(msg => IO.println(s"Sub2 (filtered) received: $msg"))
        .take(2)
      
      _ <- publisher
        .concurrently(subscriber1)
        .concurrently(subscriber2)
        .compile
        .drain
    yield ()

  // Event Bus Pattern
  sealed trait Event
  case class UserCreated(id: Int, name: String) extends Event
  case class OrderPlaced(userId: Int, amount: Double) extends Event
  case class PaymentProcessed(orderId: Int) extends Event

  def eventBusDemo: IO[Unit] =
    for
      eventBus <- Topic[IO, Event]
      
      // Event producers
      userEvents = Stream.emits(List(
        UserCreated(1, "Alice"),
        UserCreated(2, "Bob"),
        UserCreated(3, "Charlie")
      )).evalMap { event =>
        IO.sleep(50.millis) >> eventBus.publish1(event)
      }
      
      orderEvents = Stream.emits(List(
        OrderPlaced(1, 150.0),
        OrderPlaced(2, 75.5),
        PaymentProcessed(1)
      )).evalMap { event =>
        IO.sleep(75.millis) >> eventBus.publish1(event)
      }
      
      // Specialized subscribers
      userHandler = eventBus
        .subscribe(maxQueued = 100)
        .collect { case e: UserCreated => e }
        .evalMap(e => IO.println(s"[UserHandler] New user: ${e.name} (ID: ${e.id})"))
        .take(3)
      
      orderHandler = eventBus
        .subscribe(maxQueued = 100)
        .collect { case e: OrderPlaced => e }
        .evalMap(e => IO.println(s"[OrderHandler] Order for user ${e.userId}: $${e.amount}"))
        .take(2)
      
      auditHandler = eventBus
        .subscribe(maxQueued = 100)
        .evalMap(e => IO.println(s"[Audit] Event: $e"))
        .take(6)
      
      _ <- userEvents
        .merge(orderEvents)
        .concurrently(userHandler)
        .concurrently(orderHandler)
        .concurrently(auditHandler)
        .compile
        .drain
    yield ()

  def run: IO[Unit] =
    IO.println("=== Topic/PubSub Demo ===") >>
    topicDemo >>
    IO.println("\n=== Event Bus Demo ===") >>
    eventBusDemo
```

### Signal และ State Management

```scala
import cats.effect.*
import cats.effect.std.Semaphore
import fs2.*
import fs2.concurrent.{SignallingRef, Topic}
import scala.concurrent.duration.*

object SignalDemo extends IOApp.Simple:

  // SignallingRef - reactive state ที่ streams สามารถ observe ได้
  def signalDemo: IO[Unit] =
    for
      counter <- SignallingRef[IO, Int](0)
      
      // Stream ที่ observe การเปลี่ยนแปลงของ counter
      observer = counter.discrete
        .evalMap(n => IO.println(s"Counter changed to: $n"))
        .take(6)
      
      // Modifier ที่เปลี่ยนค่า counter
      modifier = Stream
        .iterate(1)(_ + 1)
        .take(5)
        .evalMap { n =>
          IO.sleep(200.millis) >>
          counter.update(_ + n) >>
          IO.println(s"Incremented by $n")
        }
      
      _ <- modifier.concurrently(observer).compile.drain
    yield ()

  // Interrupt signal
  def interruptDemo: IO[Unit] =
    for
      shouldStop <- SignallingRef[IO, Boolean](false)
      
      // Stream ที่หยุดเมื่อ signal เป็น true
      infiniteStream = Stream
        .iterate(1)(_ + 1)
        .covary[IO]
        .evalMap { n =>
          IO.sleep(100.millis) >> IO.println(s"Working: $n") >> IO.pure(n)
        }
        .interruptWhen(shouldStop)
      
      // หยุด stream หลังจาก 500ms
      stopper = IO.sleep(500.millis) >>
        IO.println("Sending stop signal!") >>
        shouldStop.set(true)
      
      _ <- infiniteStream.compile.drain.both(stopper).void
    yield ()

  def run: IO[Unit] =
    IO.println("=== Signal Demo ===") >>
    signalDemo >>
    IO.println("\n=== Interrupt Demo ===") >>
    interruptDemo
```

---

## Stream Throttling และ Batching

### Throttling Techniques

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object ThrottlingDemo extends IOApp.Simple:

  // 1. metered - ส่งข้อมูลในอัตราที่กำหนด
  def meteredDemo: IO[Unit] =
    IO.println("Metered stream (1 item/second):") >>
    Stream
      .iterate(1)(_ + 1)
      .take(5)
      .metered(1.second)  // 1 item per second
      .evalMap(n => IO.println(s"  Item $n at ${System.currentTimeMillis()}ms"))
      .compile
      .drain

  // 2. throttle - จำกัดอัตราการประมวลผล
  def throttleDemo: IO[Unit] =
    IO.println("\nThrottled stream (3 items/second):") >>
    Stream
      .iterate(1)(_ + 1)
      .take(9)
      .covary[IO]
      .throttle(3, 1.second)  // max 3 items per second
      .evalMap(n => IO.println(s"  Throttled item $n"))
      .compile
      .drain

  // 3. debounce - รอให้ไม่มีข้อมูลใหม่ก่อนส่ง (useful สำหรับ search input)
  def debounceDemo: IO[Unit] =
    IO.println("\nDebounced stream (wait 300ms of silence):") >>
    Stream
      .emits(List(
        (0.millis, "h"),
        (50.millis, "he"),
        (100.millis, "hel"),
        (150.millis, "hell"),
        (200.millis, "hello"),
        (700.millis, "w"),   // หยุดนาน แล้วเริ่มใหม่
        (750.millis, "wo"),
        (800.millis, "wor"),
        (850.millis, "worl"),
        (900.millis, "world")
      ))
      .covary[IO]
      .evalMap { (delay, text) =>
        IO.sleep(delay) >> IO.pure(text)
      }
      .debounce(300.millis)  // ส่งเฉพาะหลังจากไม่มีข้อมูลใหม่ 300ms
      .evalMap(text => IO.println(s"  Search for: '$text'"))
      .compile
      .drain

  def run: IO[Unit] =
    meteredDemo >> throttleDemo >> debounceDemo
```

### Batching Strategies

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object BatchingDemo extends IOApp.Simple:

  // 1. groupN - จัดกลุ่มตามจำนวน
  def groupByCountDemo: IO[Unit] =
    IO.println("Group by count (batches of 5):") >>
    Stream
      .range(1, 26)
      .covary[IO]
      .groupN(5)
      .evalMap { batch =>
        IO.println(s"  Batch: ${batch.toList}")
      }
      .compile
      .drain

  // 2. groupWithin - จัดกลุ่มตามเวลาหรือจำนวน (whichever comes first)
  def groupWithinDemo: IO[Unit] =
    IO.println("\nGroup within time window (max 3 items or 200ms):") >>
    Stream
      .iterate(1)(_ + 1)
      .take(15)
      .covary[IO]
      .evalMap { n =>
        // simulate varying arrival times
        IO.sleep((n * 30 % 100).millis) >> IO.pure(n)
      }
      .groupWithin(3, 200.millis)  // max 3 items or 200ms timeout
      .evalMap { chunk =>
        IO.println(s"  Window batch: ${chunk.toList}")
      }
      .compile
      .drain

  // 3. Custom sliding window
  def slidingWindowDemo: IO[Unit] =
    IO.println("\nSliding window (size=3, step=1):") >>
    Stream
      .range(1, 11)
      .covary[IO]
      .sliding(3)  // sliding window of size 3
      .evalMap { window =>
        val avg = window.toList.sum.toDouble / window.size
        IO.println(s"  Window ${window.toList}: avg=${f"$avg%.1f"}")
      }
      .compile
      .drain

  // 4. Batch processing with flushOnEmpty
  def dynamicBatchDemo: IO[Unit] =
    IO.println("\nDynamic batching (flush when buffer has items):") >>
    Stream
      .emits(1 to 20)
      .covary[IO]
      .chunkMin(5, allowFewer = true)  // min 5 items, but flush remainder
      .evalMap { chunk =>
        IO.println(s"  Processing batch of ${chunk.size}: ${chunk.toList.mkString(", ")}")
      }
      .compile
      .drain

  def run: IO[Unit] =
    groupByCountDemo >> groupWithinDemo >> slidingWindowDemo >> dynamicBatchDemo
```

---

## Rate-limited Stream Processing

### Token Bucket Algorithm

```scala
import cats.effect.*
import cats.effect.std.Semaphore
import fs2.*
import scala.concurrent.duration.*

// Token Bucket สำหรับ rate limiting
class TokenBucket[F[_]: Temporal](
  tokens: Ref[F, Double],
  capacity: Double,
  refillRate: Double  // tokens per second
):
  def acquire(n: Double = 1.0): F[Unit] =
    def loop: F[Unit] =
      tokens.modify { current =>
        if current >= n then
          (current - n, true)
        else
          (current, false)
      }.flatMap { canProceed =>
        if canProceed then Temporal[F].unit
        else
          val waitTime = ((n - ???) / refillRate * 1000).toLong.millis
          Temporal[F].sleep(50.millis) >> loop
      }
    loop

  private def refill: F[Unit] =
    tokens.update(current => math.min(capacity, current + refillRate / 10))

object TokenBucket:
  def make[F[_]: Temporal](
    capacity: Int, 
    ratePerSecond: Int
  ): Resource[F, TokenBucket[F]] =
    Resource.eval(
      Ref.of[F, Double](capacity.toDouble)
    ).map(tokens => new TokenBucket[F](tokens, capacity, ratePerSecond))
```

### Rate Limiting ด้วย fs2

```scala
import cats.effect.*
import cats.effect.std.Semaphore
import fs2.*
import scala.concurrent.duration.*

object RateLimitedProcessing extends IOApp.Simple:

  // วิธีที่ 1: metered - ควบคุม throughput
  def rateLimitedWithMetered: IO[Unit] =
    IO.println("Rate limited: 5 items/second") >>
    Stream
      .iterate(1)(_ + 1)
      .take(15)
      .covary[IO]
      .metered(200.millis)  // 5 items per second (200ms interval)
      .evalMap { n =>
        IO.realTime.flatMap { t =>
          IO.println(s"  Processing $n at ${t.toSeconds}s")
        }
      }
      .compile
      .drain

  // วิธีที่ 2: Semaphore-based rate limiter
  def rateLimitedWithSemaphore: IO[Unit] =
    for
      sem <- Semaphore[IO](3)  // max 3 concurrent operations
      
      _ <- IO.println("\nMax 3 concurrent operations:")
      
      _ <- Stream
        .range(1, 13)
        .covary[IO]
        .parEvalMap(10) { n =>
          sem.permit.use { _ =>
            IO.println(s"  Start processing $n") >>
            IO.sleep((200 + n * 30 % 300).millis) >>
            IO.println(s"  Done processing $n") >>
            IO.pure(n * n)
          }
        }
        .compile
        .drain
    yield ()

  // วิธีที่ 3: Time-windowed rate limiting
  case class RateLimiter(
    windowSize: FiniteDuration,
    maxRequests: Int
  )

  def withRateLimit[A](
    limiter: RateLimiter,
    stream: Stream[IO, A]
  ): Stream[IO, A] =
    stream
      .chunks
      .flatMap { chunk =>
        Stream.chunk(chunk)
          .zipWithScan(0) { (count, _) => count + 1 }
          .evalMap { (item, count) =>
            if count % limiter.maxRequests == 0 then
              IO.sleep(limiter.windowSize) >> IO.pure(item)
            else
              IO.pure(item)
          }
      }

  def customRateLimitDemo: IO[Unit] =
    IO.println("\nCustom rate limit: 5 per 1 second:") >>
    withRateLimit(
      RateLimiter(1.second, 5),
      Stream.range(1, 21).covary[IO]
    )
    .evalMap { n =>
      IO.realTime.flatMap { t =>
        IO.println(s"  Item $n at ${t.toMillis}ms")
      }
    }
    .compile
    .drain

  def run: IO[Unit] =
    rateLimitedWithMetered >>
    rateLimitedWithSemaphore >>
    customRateLimitDemo
```

### API Rate Limiter สำหรับ External Services

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*
import scala.concurrent.duration.*

// จำลอง API ที่มี rate limit
object ApiRateLimiterExample extends IOApp.Simple:

  // จำลอง API call
  def callExternalApi(request: String): IO[String] =
    IO.sleep(10.millis) >> IO.pure(s"Response for: $request")

  // Rate-limited API client
  class RateLimitedApiClient(
    requestsPerSecond: Int,
    semaphore: Semaphore[IO]
  ):
    private val interval = (1000.0 / requestsPerSecond).millis

    def call(request: String): IO[String] =
      semaphore.permit.use { _ =>
        callExternalApi(request)
      }

  object RateLimitedApiClient:
    def make(requestsPerSecond: Int): Resource[IO, RateLimitedApiClient] =
      Resource.eval(Semaphore[IO](requestsPerSecond))
        .map(sem => new RateLimitedApiClient(requestsPerSecond, sem))

  def run: IO[Unit] =
    RateLimitedApiClient.make(10).use { client =>
      val requests = Stream
        .range(1, 51)
        .map(n => s"request-$n")
        .covary[IO]

      requests
        .parEvalMap(20)(client.call)  // 20 concurrent, but limited to 10/sec
        .evalMap(response => IO.println(s"Got: $response"))
        .compile
        .drain
    }
```

---

## การจัดการ Errors ใน Reactive Streams

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object ErrorHandlingStreams extends IOApp.Simple:

  // 1. handleErrorWith - recover จาก error
  def basicErrorHandling: IO[Unit] =
    IO.println("Basic error handling:") >>
    Stream
      .range(1, 11)
      .covary[IO]
      .evalMap { n =>
        if n == 5 then IO.raiseError(new RuntimeException(s"Error at $n"))
        else IO.pure(n)
      }
      .handleErrorWith { error =>
        Stream.eval(IO.println(s"  Caught error: ${error.getMessage}")) >>
        Stream.empty
      }
      .evalMap(n => IO.println(s"  Success: $n"))
      .compile
      .drain

  // 2. attempt - แปลง error เป็น Either
  def attemptDemo: IO[Unit] =
    IO.println("\nAttempt (convert errors to Either):") >>
    Stream
      .range(1, 8)
      .covary[IO]
      .evalMap { n =>
        if n % 3 == 0 then IO.raiseError(new RuntimeException(s"Failed at $n"))
        else IO.pure(n)
      }
      .attempt  // Stream[IO, Either[Throwable, Int]]
      .evalMap {
        case Right(n)  => IO.println(s"  Success: $n")
        case Left(err) => IO.println(s"  Error: ${err.getMessage}")
      }
      .compile
      .drain

  // 3. Retry logic
  def withRetry(
    maxAttempts: Int,
    delay: FiniteDuration
  )(action: IO[String]): IO[String] =
    def loop(attempt: Int): IO[String] =
      action.handleErrorWith { err =>
        if attempt < maxAttempts then
          IO.println(s"  Attempt $attempt failed: ${err.getMessage}. Retrying...") >>
          IO.sleep(delay) >>
          loop(attempt + 1)
        else
          IO.raiseError(err)
      }
    loop(1)

  def retryDemo: IO[Unit] =
    IO.println("\nRetry demo (max 3 attempts):") >>
    var attemptCount = 0
    val flakyOperation = IO {
      attemptCount += 1
      if attemptCount < 3 then throw new RuntimeException(s"Transient error $attemptCount")
      else s"Success on attempt $attemptCount"
    }

    withRetry(3, 100.millis)(flakyOperation)
      .flatMap(result => IO.println(s"  Final result: $result"))
      .handleErrorWith(err => IO.println(s"  All attempts failed: ${err.getMessage}"))

  // 4. Circuit Breaker pattern
  sealed trait CircuitState
  case object Closed extends CircuitState
  case object Open extends CircuitState
  case object HalfOpen extends CircuitState

  def circuitBreakerDemo: IO[Unit] =
    for
      state <- Ref.of[IO, CircuitState](Closed)
      failures <- Ref.of[IO, Int](0)
      
      _ <- IO.println("\nCircuit Breaker demo:")
      
      callWithCircuitBreaker = (n: Int) =>
        state.get.flatMap {
          case Open =>
            IO.println(s"  Circuit OPEN, rejecting request $n")
          case _ =>
            val operation =
              if n <= 3 then IO.raiseError(new RuntimeException(s"Service down at $n"))
              else IO.println(s"  Request $n succeeded")
            
            operation.handleErrorWith { err =>
              failures.updateAndGet(_ + 1).flatMap { count =>
                if count >= 3 then
                  state.set(Open) >>
                  IO.println("  Circuit OPENED due to repeated failures!")
                else
                  IO.println(s"  Failure $count: ${err.getMessage}")
              }
            }
        }
      
      _ <- Stream
        .range(1, 9)
        .covary[IO]
        .evalMap(callWithCircuitBreaker)
        .compile
        .drain
    yield ()

  def run: IO[Unit] =
    basicErrorHandling >>
    attemptDemo >>
    retryDemo >>
    circuitBreakerDemo
```

---

## การรวม Multiple Streams

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object StreamCombination extends IOApp.Simple:

  // 1. zip - รวม 2 streams ทีละคู่
  def zipDemo: IO[Unit] =
    IO.println("Zip streams:") >>
    (
      Stream.range(1, 6).covary[IO],
      Stream("a", "b", "c", "d", "e").covary[IO]
    ).tupled
      .evalMap { (n, s) => IO.println(s"  ($n, $s)") }
      .compile
      .drain

  // 2. zipAll - zip โดยมีค่า default
  def zipAllDemo: IO[Unit] =
    IO.println("\nZipAll (different lengths):") >>
    Stream
      .range(1, 4)
      .covary[IO]
      .zipAll(Stream.range(10, 16).covary[IO])(0, 0)
      .evalMap { (a, b) => IO.println(s"  ($a, $b)") }
      .compile
      .drain

  // 3. either - รวม 2 streams โดยรู้ว่ามาจาก stream ไหน
  def eitherDemo: IO[Unit] =
    IO.println("\nEither (tag by source):") >>
    Stream
      .range(1, 4)
      .covary[IO]
      .evalMap(n => IO.sleep(100.millis) >> IO.pure(n))
      .either(
        Stream.range(10, 14).covary[IO]
          .evalMap(n => IO.sleep(80.millis) >> IO.pure(n))
      )
      .evalMap {
        case Left(n)  => IO.println(s"  From stream1: $n")
        case Right(n) => IO.println(s"  From stream2: $n")
      }
      .compile
      .drain

  // 4. flatMap - แปลง item เป็น stream
  def flatMapDemo: IO[Unit] =
    IO.println("\nFlatMap (expand items to streams):") >>
    Stream
      .range(1, 4)
      .covary[IO]
      .flatMap { n =>
        Stream
          .range(1, n + 1)
          .covary[IO]
          .map(i => s"$n.$i")
      }
      .evalMap(s => IO.println(s"  $s"))
      .compile
      .drain

  // 5. concatMap - sequential flatMap
  def concatMapDemo: IO[Unit] =
    IO.println("\nConcatMap (sequential sub-streams):") >>
    Stream
      .range(1, 4)
      .covary[IO]
      .flatMap { n =>
        Stream
          .range(1, 4)
          .covary[IO]
          .evalMap { i =>
            IO.sleep(50.millis) >> IO.pure(s"$n*$i=${n*i}")
          }
      }
      .evalMap(s => IO.println(s"  $s"))
      .compile
      .drain

  def run: IO[Unit] =
    zipDemo >> zipAllDemo >> eitherDemo >> flatMapDemo >> concatMapDemo
```

---

## Complete Reactive Pipeline

### Real-World Example: Log Processing Pipeline

```scala
import cats.effect.*
import cats.effect.std.{Queue, Topic}
import fs2.*
import fs2.io.file.{Files, Path}
import scala.concurrent.duration.*

// Domain Models
case class RawLogLine(timestamp: Long, content: String)
case class ParsedLog(
  timestamp: Long,
  level: String,
  service: String,
  message: String
)
case class LogAlert(
  severity: String,
  service: String,
  message: String,
  count: Int
)
case class Metrics(
  errorCount: Int,
  warnCount: Int,
  infoCount: Int,
  servicesAffected: Set[String]
)

object CompleteReactivePipeline extends IOApp.Simple:

  // Stage 1: Ingest - รับข้อมูล logs
  def ingestLogs: Stream[IO, RawLogLine] =
    // จำลองการรับ log จาก multiple sources
    val source1 = Stream
      .iterate(System.currentTimeMillis())(_ + 100)
      .take(20)
      .map(ts => RawLogLine(ts, s"INFO service-a Request processed successfully"))

    val source2 = Stream
      .iterate(System.currentTimeMillis())(_ + 150)
      .take(15)
      .map(ts => RawLogLine(ts, s"ERROR service-b Connection timeout"))

    val source3 = Stream
      .iterate(System.currentTimeMillis())(_ + 200)
      .take(10)
      .map(ts => RawLogLine(ts, s"WARN service-c High memory usage detected"))

    source1.covary[IO]
      .merge(source2.covary[IO])
      .merge(source3.covary[IO])
      .evalTap(_ => IO.sleep(10.millis))  // จำลอง network delay

  // Stage 2: Parse - แปลง raw logs เป็น structured data
  def parseLogs(raw: Stream[IO, RawLogLine]): Stream[IO, ParsedLog] =
    raw.evalMap { line =>
      IO {
        val parts = line.content.split(" ", 3)
        ParsedLog(
          timestamp = line.timestamp,
          level = parts(0),
          service = parts(1),
          message = parts(2)
        )
      }.handleErrorWith(_ =>
        IO.pure(ParsedLog(line.timestamp, "UNKNOWN", "unknown", line.content))
      )
    }

  // Stage 3: Enrich - เพิ่มข้อมูลเพิ่มเติม
  def enrichLogs(parsed: Stream[IO, ParsedLog]): Stream[IO, ParsedLog] =
    parsed.evalMap { log =>
      // จำลองการ lookup service metadata
      IO.pure(log.copy(
        service = log.service + "-v1.0"
      ))
    }

  // Stage 4: Filter และ Route
  def routeLogs(
    enriched: Stream[IO, ParsedLog],
    errorTopic: Topic[IO, ParsedLog],
    warnTopic: Topic[IO, ParsedLog]
  ): Stream[IO, ParsedLog] =
    enriched.evalTap { log =>
      log.level match
        case "ERROR" => errorTopic.publish1(log).void
        case "WARN"  => warnTopic.publish1(log).void
        case _       => IO.unit
    }

  // Stage 5: Aggregate metrics
  def aggregateMetrics(logs: Stream[IO, ParsedLog]): Stream[IO, Metrics] =
    logs
      .groupWithin(100, 1.second)  // batch ทุก 100 items หรือ 1 second
      .map { chunk =>
        val logList = chunk.toList
        Metrics(
          errorCount = logList.count(_.level == "ERROR"),
          warnCount = logList.count(_.level == "WARN"),
          infoCount = logList.count(_.level == "INFO"),
          servicesAffected = logList.map(_.service).toSet
        )
      }

  // Stage 6: Alert generation
  def generateAlerts(
    errorLogs: Stream[IO, ParsedLog]
  ): Stream[IO, LogAlert] =
    errorLogs
      .groupWithin(50, 30.seconds)
      .map { chunk =>
        val errors = chunk.toList
        val serviceGroups = errors.groupBy(_.service)
        serviceGroups.map { (service, logs) =>
          LogAlert(
            severity = if logs.size > 10 then "CRITICAL" else "HIGH",
            service = service,
            message = logs.head.message,
            count = logs.size
          )
        }
      }
      .flatMap(alerts => Stream.emits(alerts.toList))

  // Main pipeline
  def run: IO[Unit] =
    for
      errorTopic <- Topic[IO, ParsedLog]
      warnTopic  <- Topic[IO, ParsedLog]

      // Setup log pipeline
      logStream = ingestLogs
        .through(s => parseLogs(s))
        .through(s => enrichLogs(s))
        .through(s => routeLogs(s, errorTopic, warnTopic))

      // Error monitoring subscriber
      errorMonitor = errorTopic
        .subscribe(100)
        .evalMap { log =>
          IO.println(s"[ERROR MONITOR] ${log.service}: ${log.message}")
        }
        .take(15)

      // Metrics aggregator
      metricsStream = aggregateMetrics(logStream)
        .evalMap { metrics =>
          IO.println(s"""
            |[METRICS] Window summary:
            |  Errors: ${metrics.errorCount}
            |  Warnings: ${metrics.warnCount}
            |  Info: ${metrics.infoCount}
            |  Services: ${metrics.servicesAffected.mkString(", ")}
            |""".stripMargin)
        }
        .take(5)

      // Alert stream from errors
      alertStream = errorTopic
        .subscribe(100)
        .through(generateAlerts)
        .evalMap { alert =>
          IO.println(s"[ALERT] ${alert.severity} - ${alert.service}: ${alert.message} (${alert.count} occurrences)")
        }
        .take(3)

      _ <- IO.println("=== Starting Reactive Log Processing Pipeline ===\n")
      
      _ <- errorMonitor
        .concurrently(metricsStream)
        .concurrently(alertStream)
        .compile
        .drain
      
      _ <- IO.println("\n=== Pipeline completed ===")
    yield ()
```

---

## Performance Tuning

### การปรับแต่งประสิทธิภาพ

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object PerformanceTuning extends IOApp.Simple:

  // 1. ปรับ chunk size
  def optimizeChunkSize: IO[Unit] =
    val data = Stream.range(1, 100001).covary[IO]
    
    def processWithChunkSize(size: Int): IO[Long] =
      for
        start <- IO.monotonic
        _ <- data
          .chunkN(size)
          .evalMap { chunk =>
            IO.pure(chunk.map(_ * 2))
          }
          .flatMap(Stream.chunk)
          .compile
          .drain
        end <- IO.monotonic
      yield (end - start).toMillis
    
    for
      t1 <- processWithChunkSize(1)
      _ <- IO.println(s"Chunk size 1: ${t1}ms")
      t2 <- processWithChunkSize(100)
      _ <- IO.println(s"Chunk size 100: ${t2}ms")
      t3 <- processWithChunkSize(1000)
      _ <- IO.println(s"Chunk size 1000: ${t3}ms")
      t4 <- processWithChunkSize(10000)
      _ <- IO.println(s"Chunk size 10000: ${t4}ms")
    yield ()

  // 2. ปรับ parallelism
  def optimizeParallelism: IO[Unit] =
    val data = Stream.range(1, 101).covary[IO]
    
    def processWithParallelism(n: Int): IO[Long] =
      for
        start <- IO.monotonic
        _ <- data
          .parEvalMap(n) { item =>
            IO.sleep(10.millis) >> IO.pure(item * 2)
          }
          .compile
          .drain
        end <- IO.monotonic
      yield (end - start).toMillis
    
    for
      t1 <- processWithParallelism(1)
      _ <- IO.println(s"\nParallelism 1: ${t1}ms")
      t2 <- processWithParallelism(5)
      _ <- IO.println(s"Parallelism 5: ${t2}ms")
      t3 <- processWithParallelism(10)
      _ <- IO.println(s"Parallelism 10: ${t3}ms")
      t4 <- processWithParallelism(20)
      _ <- IO.println(s"Parallelism 20: ${t4}ms")
    yield ()

  // 3. Buffer tuning
  def bufferTuning: IO[Unit] =
    IO.println("\nBuffer tuning:") >>
    Stream
      .range(1, 1001)
      .covary[IO]
      .buffer(100)  // pre-fetch up to 100 items
      .evalMap { n =>
        // simulate I/O bound work
        IO.sleep(1.millis) >> IO.pure(n)
      }
      .compile
      .drain
      .flatMap(_ => IO.println("  Buffered processing complete"))

  def run: IO[Unit] =
    IO.println("=== Performance Tuning ===") >>
    optimizeChunkSize >>
    optimizeParallelism >>
    bufferTuning
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Reactive Streams และ Back-pressure:

### สิ่งที่ได้เรียนรู้

| แนวคิด | การนำไปใช้ |
|--------|-----------|
| Reactive Streams Spec | Publisher/Subscriber/Subscription interfaces |
| Back-pressure | fs2 pull-based model, bounded queues |
| fs2 Streams | Pure, effectful, resource-safe streams |
| Topic/Signal | Broadcast patterns, reactive state |
| Throttling | metered, throttle, debounce |
| Batching | groupN, groupWithin, sliding |
| Rate Limiting | Semaphore, token bucket, metered |
| Error Handling | handleErrorWith, attempt, retry, circuit breaker |
| Stream Combination | zip, merge, either, flatMap |
| Performance | chunk size, parallelism, buffering |

### Best Practices

1. **ใช้ bounded structures** - Queue, Topic ที่มี bound เพื่อป้องกัน OOM
2. **เลือก chunk size ที่เหมาะสม** - ใหญ่เกินไปก็ไม่ดี เล็กเกินไปก็ช้า
3. **handle errors ในทุก stage** - ใช้ attempt หรือ handleErrorWith
4. **ใช้ Resource สำหรับ cleanup** - ป้องกัน resource leak
5. **monitor back-pressure** - ตรวจสอบว่า consumer ตาม producer ทัน

### เมื่อไหรควรใช้ Reactive Streams

- เมื่อมีข้อมูลต่อเนื่อง (continuous data flow)
- เมื่อ producer เร็วกว่า consumer
- เมื่อต้องการ composition ของ async operations
- เมื่อต้องการ resource safety (cleanup on error/cancellation)

---

*[← Part 54: Stream Processing Fundamentals](part-54-stream-processing.md) | [Part 56: Typeclass Derivation →](part-56-typeclass-derivation.md)*
