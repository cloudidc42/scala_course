# Part 50: Data Pipeline ด้วย fs2 และ Kafka

## สารบัญ

1. [สถาปัตยกรรม ETL Pipeline](#สถาปัตยกรรม-etl-pipeline)
2. [fs2 Stream Processing](#fs2-stream-processing)
3. [อ่าน CSV และแปลงข้อมูล](#อ่าน-csv-และแปลงข้อมูล)
4. [Windowed Aggregations](#windowed-aggregations)
5. [Kafka Streams Integration](#kafka-streams-integration)
6. [Error Handling และ Dead Letter Queue](#error-handling-และ-dead-letter-queue)
7. [Pipeline Metrics](#pipeline-metrics)
8. [ตัวอย่าง Pipeline สมบูรณ์](#ตัวอย่าง-pipeline-สมบูรณ์)

---

## สถาปัตยกรรม ETL Pipeline

ETL (Extract, Transform, Load) Pipeline เป็นแนวทางหลักในการจัดการข้อมูลในระบบ Data Engineering ใน Scala 3 เราใช้ **fs2** เป็น streaming library หลักร่วมกับ **Kafka** สำหรับ message transport

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌─────────────┐
│   Source    │───▶│  Transform   │───▶│   Enrich     │───▶│    Sink     │
│ (CSV/Kafka) │    │  (Parse/Map) │    │ (Join/Lookup)│    │  (DB/Kafka) │
└─────────────┘    └──────────────┘    └──────────────┘    └─────────────┘
       │                  │                   │                    │
       └──────────────────┴───────────────────┴────────────────────┘
                                    │
                           ┌────────▼────────┐
                           │  Dead Letter Q  │
                           │  (Error Sink)   │
                           └─────────────────┘
```

### Dependencies (build.sbt)

```scala
libraryDependencies ++= Seq(
  "co.fs2"          %% "fs2-core"        % "3.10.0",
  "co.fs2"          %% "fs2-io"          % "3.10.0",
  "com.github.fd4s" %% "fs2-kafka"       % "3.5.1",
  "org.tpolecat"    %% "doobie-core"     % "1.0.0-RC4",
  "org.tpolecat"    %% "doobie-postgres" % "1.0.0-RC4",
  "org.tpolecat"    %% "doobie-hikari"   % "1.0.0-RC4",
  "com.github.fd4s" %% "fs2-kafka"       % "3.5.1",
  "io.circe"        %% "circe-core"      % "0.14.9",
  "io.circe"        %% "circe-generic"   % "0.14.9",
  "io.circe"        %% "circe-parser"    % "0.14.9",
  "org.typelevel"   %% "cats-effect"     % "3.5.4"
)
```

---

## fs2 Stream Processing

### พื้นฐาน fs2 Stream

fs2 Stream เป็น `Stream[F, O]` ที่ประมวลผลแบบ lazy และ composable:

```scala
import cats.effect.*
import fs2.*
import fs2.io.file.*

// Stream พื้นฐาน
val numbers: Stream[Pure, Int] = Stream(1, 2, 3, 4, 5)

// Stream ที่มี effect
val effectfulStream: Stream[IO, String] = Stream
  .eval(IO.println("Starting stream"))
  .flatMap(_ => Stream("hello", "world"))

// Transformation pipeline
val pipeline: Stream[IO, Int] = Stream
  .range(1, 100)
  .filter(_ % 2 == 0)
  .map(_ * 2)
  .take(10)
```

### Stream Combinators ที่สำคัญ

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object StreamCombinators extends IOApp.Simple:

  // merge: รวม 2 streams
  val merged: Stream[IO, Int] =
    Stream(1, 2, 3).merge(Stream(4, 5, 6))

  // zip: จับคู่ elements
  val zipped: Stream[IO, (Int, String)] =
    Stream(1, 2, 3).zip(Stream("a", "b", "c"))

  // flatMap: expand each element
  val expanded: Stream[IO, Int] =
    Stream(1, 2, 3).flatMap(n => Stream.range(0, n))

  // through: pipe transformation
  val doubled: Pipe[IO, Int, Int] = _.map(_ * 2)
  val result = Stream(1, 2, 3).through(doubled)

  // parEvalMap: concurrent processing
  val concurrent: Stream[IO, String] = Stream
    .range(1, 10)
    .parEvalMap(4)(n => IO.sleep(100.millis) >> IO.pure(s"processed: $n"))

  def run: IO[Unit] =
    concurrent
      .evalTap(s => IO.println(s))
      .compile
      .drain
```

---

## อ่าน CSV และแปลงข้อมูล

### Data Models

```scala
import java.time.LocalDate
import io.circe.*
import io.circe.generic.semiauto.*

// Raw CSV row
case class CsvRow(
  id: String,
  userId: String,
  amount: String,
  currency: String,
  date: String,
  category: String
)

// Parsed domain object
case class Transaction(
  id: String,
  userId: String,
  amount: BigDecimal,
  currency: String,
  date: LocalDate,
  category: Category
)

enum Category:
  case Food, Transport, Entertainment, Shopping, Other

object Category:
  def fromString(s: String): Either[String, Category] =
    s.toLowerCase match
      case "food"          => Right(Food)
      case "transport"     => Right(Transport)
      case "entertainment" => Right(Entertainment)
      case "shopping"      => Right(Shopping)
      case "other"         => Right(Other)
      case unknown         => Left(s"Unknown category: $unknown")

// Enriched transaction with derived fields
case class EnrichedTransaction(
  transaction: Transaction,
  isHighValue: Boolean,
  dayOfWeek: String,
  monthYear: String
)
```

### CSV Parser Pipe

```scala
import cats.effect.*
import cats.syntax.all.*
import fs2.*
import fs2.io.file.*
import java.time.LocalDate
import java.time.format.DateTimeFormatter

object CsvPipeline:

  // Parse CSV header and rows
  def parseCsvLine(line: String): Array[String] =
    line.split(",").map(_.trim)

  // Parse raw row to CsvRow
  def toCsvRow(parts: Array[String]): Either[String, CsvRow] =
    if parts.length == 6 then
      Right(CsvRow(
        id       = parts(0),
        userId   = parts(1),
        amount   = parts(2),
        currency = parts(3),
        date     = parts(4),
        category = parts(5)
      ))
    else
      Left(s"Invalid CSV row: expected 6 fields, got ${parts.length}")

  // Parse CsvRow to Transaction
  def parseTransaction(row: CsvRow): Either[String, Transaction] =
    for
      amount   <- row.amount.toBigDecimalOption.toRight(s"Invalid amount: ${row.amount}")
      date     <- Either.catchNonFatal(
                    LocalDate.parse(row.date, DateTimeFormatter.ISO_LOCAL_DATE)
                  ).leftMap(e => s"Invalid date: ${row.date} - ${e.getMessage}")
      category <- Category.fromString(row.category)
    yield Transaction(
      id       = row.id,
      userId   = row.userId,
      amount   = amount,
      currency = row.currency,
      date     = date,
      category = category
    )

  // Enrich transaction
  def enrich(t: Transaction): EnrichedTransaction =
    EnrichedTransaction(
      transaction = t,
      isHighValue = t.amount > 1000,
      dayOfWeek   = t.date.getDayOfWeek.toString,
      monthYear   = f"${t.date.getYear}-${t.date.getMonthValue}%02d"
    )

  // Pipe: String lines -> Transaction (with error tracking)
  def csvToTransaction: Pipe[IO, String, Either[PipelineError, Transaction]] =
    _.drop(1) // skip header
     .filter(_.nonEmpty)
     .map: line =>
       for
         row  <- toCsvRow(parseCsvLine(line))
                   .leftMap(msg => PipelineError.ParseError(line, msg))
         txn  <- parseTransaction(row)
                   .leftMap(msg => PipelineError.ValidationError(row.id, msg))
       yield txn

  // Read CSV file as stream
  def readCsvFile(path: Path): Stream[IO, Either[PipelineError, Transaction]] =
    Files[IO]
      .readAll(path)
      .through(text.utf8.decode)
      .through(text.lines)
      .through(csvToTransaction)
```

### Error Types

```scala
sealed trait PipelineError:
  def message: String

object PipelineError:
  case class ParseError(raw: String, message: String) extends PipelineError
  case class ValidationError(id: String, message: String) extends PipelineError
  case class EnrichmentError(id: String, message: String) extends PipelineError
  case class WriteError(id: String, cause: Throwable) extends PipelineError:
    def message: String = s"Write failed for $id: ${cause.getMessage}"
```

---

## Windowed Aggregations

### GroupWithin - Batch Processing

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

object WindowedAggregations:

  // groupWithin: รวม elements ใน time window หรือ size limit
  def batchTransactions(
    stream: Stream[IO, Transaction],
    maxSize: Int = 100,
    maxWait: FiniteDuration = 5.seconds
  ): Stream[IO, List[Transaction]] =
    stream.groupWithin(maxSize, maxWait).map(_.toList)

  // Aggregate stats per window
  case class WindowStats(
    windowStart: Long,
    count: Int,
    totalAmount: BigDecimal,
    avgAmount: BigDecimal,
    maxAmount: BigDecimal,
    categories: Map[String, Int]
  )

  def computeWindowStats(transactions: List[Transaction]): WindowStats =
    val total = transactions.map(_.amount).sum
    val count = transactions.length
    WindowStats(
      windowStart = System.currentTimeMillis(),
      count       = count,
      totalAmount = total,
      avgAmount   = if count > 0 then total / count else 0,
      maxAmount   = transactions.map(_.amount).maxOption.getOrElse(0),
      categories  = transactions.groupBy(_.category.toString).map((k, v) => k -> v.length)
    )

  // Sliding window aggregation
  def slidingWindowStats(
    stream: Stream[IO, Transaction],
    windowSize: Int = 50,
    slideBy: Int = 10
  ): Stream[IO, WindowStats] =
    stream
      .sliding(windowSize, slideBy)
      .map(chunk => computeWindowStats(chunk.toList))
```

### Tumbling Window

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*
import java.time.Instant

object TumblingWindow:

  case class TimeWindow(
    start: Instant,
    end: Instant,
    transactions: List[Transaction]
  )

  // Group by 1-minute tumbling windows
  def tumblingWindow(
    stream: Stream[IO, Transaction],
    windowDuration: FiniteDuration = 1.minute
  ): Stream[IO, TimeWindow] =

    def windowKey(t: Transaction): Long =
      val millis = t.date.toEpochDay * 86400000L
      millis / windowDuration.toMillis

    stream
      .groupAdjacentBy(windowKey)
      .map: (key, chunk) =>
        val txns  = chunk.toList
        val start = Instant.ofEpochMilli(key * windowDuration.toMillis)
        val end   = start.plusMillis(windowDuration.toMillis)
        TimeWindow(start, end, txns)

  // Per-user aggregation in tumbling window
  case class UserWindowAgg(
    userId: String,
    window: TimeWindow,
    totalSpend: BigDecimal,
    transactionCount: Int
  )

  def aggregatePerUser(windows: Stream[IO, TimeWindow]): Stream[IO, List[UserWindowAgg]] =
    windows.map: window =>
      window.transactions
        .groupBy(_.userId)
        .map: (userId, txns) =>
          UserWindowAgg(
            userId           = userId,
            window           = window,
            totalSpend       = txns.map(_.amount).sum,
            transactionCount = txns.length
          )
        .toList
```

---

## Kafka Streams Integration

### Kafka Consumer

```scala
import cats.effect.*
import fs2.kafka.*
import io.circe.*
import io.circe.parser.*
import io.circe.generic.semiauto.*
import scala.concurrent.duration.*

object KafkaConsumer:

  given Decoder[Transaction] = deriveDecoder

  // Kafka consumer settings
  def consumerSettings(
    bootstrapServers: String,
    groupId: String
  ): ConsumerSettings[IO, String, String] =
    ConsumerSettings[IO, String, String]
      .withBootstrapServers(bootstrapServers)
      .withGroupId(groupId)
      .withAutoOffsetReset(AutoOffsetReset.Earliest)
      .withEnableAutoCommit(false)

  // Consume and parse transactions from Kafka
  def consumeTransactions(
    settings: ConsumerSettings[IO, String, String],
    topic: String
  ): Stream[IO, CommittableConsumerRecord[IO, String, Transaction]] =
    KafkaConsumer
      .stream(settings)
      .subscribeTo(topic)
      .records
      .evalMapFilter: record =>
        parse(record.record.value)
          .flatMap(_.as[Transaction])
          .fold(
            err =>
              IO.println(s"Failed to parse message: $err").as(None),
            txn =>
              IO.pure(Some(record.as(txn)))
          )

  // Process with auto-commit after successful processing
  def processWithCommit[A](
    stream: Stream[IO, CommittableConsumerRecord[IO, String, A]],
    process: A => IO[Unit]
  ): Stream[IO, Unit] =
    stream
      .evalMap: committable =>
        process(committable.record.value)
          .flatMap(_ => committable.offset.commit)
      .handleErrorWith: e =>
        Stream.eval(IO.println(s"Error processing record: ${e.getMessage}"))
```

### Kafka Producer

```scala
import cats.effect.*
import fs2.kafka.*
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*

object KafkaProducer:

  given Encoder[EnrichedTransaction] = deriveEncoder

  def producerSettings(
    bootstrapServers: String
  ): ProducerSettings[IO, String, String] =
    ProducerSettings[IO, String, String]
      .withBootstrapServers(bootstrapServers)
      .withAcks(Acks.All)
      .withRetries(3)

  // Produce enriched transactions to output topic
  def produceTransactions(
    settings: ProducerSettings[IO, String, String],
    inputStream: Stream[IO, EnrichedTransaction],
    outputTopic: String
  ): Stream[IO, ProducerResult[String, String]] =
    inputStream
      .map: enriched =>
        val key   = enriched.transaction.userId
        val value = enriched.asJson.noSpaces
        ProducerRecords.one(ProducerRecord(outputTopic, key, value))
      .through(KafkaProducer.pipe(settings))

  // Produce to dead letter queue
  def produceToDLQ(
    settings: ProducerSettings[IO, String, String],
    errors: Stream[IO, PipelineError],
    dlqTopic: String
  ): Stream[IO, Unit] =
    errors
      .map: error =>
        val value = s"""{"error": "${error.message}", "timestamp": "${System.currentTimeMillis()}"}"""
        ProducerRecords.one(ProducerRecord(dlqTopic, "error", value))
      .through(KafkaProducer.pipe(settings))
      .void
```

---

## Error Handling และ Dead Letter Queue

### Error Channel Pattern

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*

object ErrorHandling:

  // Fan-out: success path และ error path
  def partitionResults[E, A](
    stream: Stream[IO, Either[E, A]]
  ): IO[(Stream[IO, E], Stream[IO, A])] =
    for
      errorQueue   <- Queue.unbounded[IO, Option[E]]
      successQueue <- Queue.unbounded[IO, Option[A]]
    yield
      val feeder: Stream[IO, Unit] = stream.evalMap:
        case Left(err) => errorQueue.offer(Some(err))
        case Right(ok) => successQueue.offer(Some(ok))

      val errors   = Stream.fromQueueNoneTerminated(errorQueue)
      val successes = Stream.fromQueueNoneTerminated(successQueue)
      (errors, successes)

  // Dead Letter Queue handler
  case class DLQRecord(
    originalTopic: String,
    payload: String,
    error: String,
    retryCount: Int,
    timestamp: Long = System.currentTimeMillis()
  )

  def withDeadLetterQueue[A](
    stream: Stream[IO, Either[PipelineError, A]],
    handleSuccess: A => IO[Unit],
    handleDLQ: DLQRecord => IO[Unit],
    topic: String
  ): Stream[IO, Unit] =
    stream.evalMap:
      case Right(a)    => handleSuccess(a)
      case Left(error) =>
        val dlqRecord = DLQRecord(
          originalTopic = topic,
          payload       = error.toString,
          error         = error.message,
          retryCount    = 0
        )
        handleDLQ(dlqRecord)

  // Retry with exponential backoff
  def withRetry[A](
    action: IO[A],
    maxRetries: Int = 3,
    initialDelay: scala.concurrent.duration.FiniteDuration = 100.millis
  ): IO[A] =
    import scala.concurrent.duration.*
    def attempt(retriesLeft: Int, delay: FiniteDuration): IO[A] =
      action.handleErrorWith: e =>
        if retriesLeft <= 0 then IO.raiseError(e)
        else IO.sleep(delay) >> attempt(retriesLeft - 1, delay * 2)
    attempt(maxRetries, initialDelay)
```

### Circuit Breaker Pattern

```scala
import cats.effect.*
import cats.effect.Ref
import scala.concurrent.duration.*

enum CircuitState:
  case Closed        // Normal operation
  case Open(until: Long)  // Failing, reject requests
  case HalfOpen      // Testing if service recovered

class CircuitBreaker(
  state: Ref[IO, CircuitState],
  failureThreshold: Int,
  timeout: FiniteDuration
):
  private val failures = Ref.of[IO, Int](0)

  def protect[A](action: IO[A]): IO[A] =
    state.get.flatMap:
      case CircuitState.Open(until) =>
        val now = System.currentTimeMillis()
        if now > until then
          state.set(CircuitState.HalfOpen) >> tryAction(action)
        else
          IO.raiseError(new RuntimeException("Circuit breaker is OPEN"))

      case CircuitState.HalfOpen =>
        tryAction(action)

      case CircuitState.Closed =>
        tryAction(action)

  private def tryAction[A](action: IO[A]): IO[A] =
    action
      .flatTap(_ => onSuccess)
      .handleErrorWith: e =>
        onFailure >> IO.raiseError(e)

  private def onSuccess: IO[Unit] =
    for
      f <- failures
      _ <- f.set(0)
      _ <- state.set(CircuitState.Closed)
    yield ()

  private def onFailure: IO[Unit] =
    for
      f         <- failures
      failCount <- f.updateAndGet(_ + 1)
      _         <- if failCount >= failureThreshold then
                     state.set(CircuitState.Open(System.currentTimeMillis() + timeout.toMillis))
                   else IO.unit
    yield ()

object CircuitBreaker:
  def apply(
    failureThreshold: Int = 5,
    timeout: FiniteDuration = 30.seconds
  ): IO[CircuitBreaker] =
    for
      state <- Ref.of[IO, CircuitState](CircuitState.Closed)
    yield new CircuitBreaker(state, failureThreshold, timeout)
```

---

## Pipeline Metrics

### Metrics Collection

```scala
import cats.effect.*
import cats.effect.Ref
import java.util.concurrent.atomic.AtomicLong

case class PipelineMetrics(
  processedCount: Long,
  errorCount: Long,
  skippedCount: Long,
  totalDurationMs: Long,
  throughputPerSecond: Double
)

class MetricsCollector(
  processed: Ref[IO, Long],
  errors: Ref[IO, Long],
  skipped: Ref[IO, Long],
  startTime: Long
):
  def recordProcessed(count: Int = 1): IO[Unit] =
    processed.update(_ + count)

  def recordError(count: Int = 1): IO[Unit] =
    errors.update(_ + count)

  def recordSkipped(count: Int = 1): IO[Unit] =
    skipped.update(_ + count)

  def getMetrics: IO[PipelineMetrics] =
    for
      p   <- processed.get
      e   <- errors.get
      s   <- skipped.get
      dur  = System.currentTimeMillis() - startTime
      tps  = if dur > 0 then p.toDouble / (dur.toDouble / 1000) else 0.0
    yield PipelineMetrics(p, e, s, dur, tps)

  def logMetrics: IO[Unit] =
    getMetrics.flatMap: m =>
      IO.println(
        s"""
        |Pipeline Metrics:
        |  Processed:  ${m.processedCount}
        |  Errors:     ${m.errorCount}
        |  Skipped:    ${m.skippedCount}
        |  Duration:   ${m.totalDurationMs}ms
        |  Throughput: ${f"${m.throughputPerSecond}%.2f"} records/sec
        """.stripMargin
      )

object MetricsCollector:
  def create: IO[MetricsCollector] =
    for
      p <- Ref.of[IO, Long](0)
      e <- Ref.of[IO, Long](0)
      s <- Ref.of[IO, Long](0)
    yield new MetricsCollector(p, e, s, System.currentTimeMillis())
```

### Metrics Pipe

```scala
import cats.effect.*
import fs2.*

def withMetrics[A](
  metrics: MetricsCollector
): Pipe[IO, Either[PipelineError, A], Either[PipelineError, A]] =
  _.evalTap:
    case Right(_) => metrics.recordProcessed()
    case Left(_)  => metrics.recordError()

def periodicMetricsLog(
  metrics: MetricsCollector,
  interval: scala.concurrent.duration.FiniteDuration = 10.seconds
): Stream[IO, Unit] =
  Stream
    .awakeEvery[IO](interval)
    .evalMap(_ => metrics.logMetrics)
```

---

## ตัวอย่าง Pipeline สมบูรณ์

### Complete ETL Pipeline

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*
import fs2.io.file.*
import fs2.kafka.*
import doobie.*
import doobie.implicits.*
import scala.concurrent.duration.*

object CompletePipeline extends IOApp:

  // Database operations
  def insertTransactions(xa: Transactor[IO])(
    transactions: List[EnrichedTransaction]
  ): IO[Int] =
    val inserts = transactions.map: et =>
      sql"""
        INSERT INTO transactions (id, user_id, amount, currency, date, category, is_high_value)
        VALUES (
          ${et.transaction.id},
          ${et.transaction.userId},
          ${et.transaction.amount},
          ${et.transaction.currency},
          ${et.transaction.date},
          ${et.transaction.category.toString},
          ${et.isHighValue}
        )
        ON CONFLICT (id) DO NOTHING
      """.update.run
    inserts.traverse(identity).map(_.sum).transact(xa)

  // Main pipeline
  def runPipeline(
    csvPath: Path,
    kafkaSetting: ProducerSettings[IO, String, String],
    xa: Transactor[IO],
    metrics: MetricsCollector
  ): Stream[IO, Unit] =

    val kafkaTopic    = "enriched-transactions"
    val dlqTopic      = "transaction-dlq"

    // 1. Read and parse CSV
    val csvStream: Stream[IO, Either[PipelineError, Transaction]] =
      CsvPipeline.readCsvFile(csvPath)

    // 2. Enrich transactions
    val enriched: Stream[IO, Either[PipelineError, EnrichedTransaction]] =
      csvStream.map(_.map(CsvPipeline.enrich))

    // 3. Apply metrics tracking
    val tracked = enriched.through(withMetrics(metrics))

    // 4. Partition success/error
    val (errors, successes) = tracked.partition(_.isLeft)

    // 5. Batch and write to DB
    val dbWriter: Stream[IO, Unit] =
      successes
        .collect { case Right(et) => et }
        .groupWithin(100, 5.seconds)
        .evalMap: batch =>
          insertTransactions(xa)(batch.toList)
            .flatTap(n => IO.println(s"Wrote $n transactions to DB"))
            .void

    // 6. Produce to Kafka
    val kafkaProducer: Stream[IO, Unit] =
      KafkaProducer.produceTransactions(
        kafkaSetting,
        successes.collect { case Right(et) => et },
        kafkaTopic
      ).void

    // 7. Handle errors -> DLQ
    val dlqWriter: Stream[IO, Unit] =
      errors
        .collect { case Left(e) => e }
        .evalMap: err =>
          IO.println(s"DLQ: ${err.message}")

    // 8. Run all streams concurrently
    Stream(
      dbWriter,
      kafkaProducer,
      dlqWriter,
      periodicMetricsLog(metrics)
    ).parJoinUnbounded

  def run(args: List[String]): IO[ExitCode] =
    val resources = for
      metrics  <- Resource.eval(MetricsCollector.create)
      xa       <- buildTransactor()
      producer <- KafkaProducer.resource(buildProducerSettings())
    yield (metrics, xa, producer)

    resources.use: (metrics, xa, _) =>
      runPipeline(
        csvPath       = Path("/data/transactions.csv"),
        kafkaSetting  = buildProducerSettings(),
        xa            = xa,
        metrics       = metrics
      )
      .compile
      .drain
      .as(ExitCode.Success)

  private def buildTransactor(): Resource[IO, Transactor[IO]] =
    import doobie.hikari.*
    HikariTransactor.newHikariTransactor[IO](
      "org.postgresql.Driver",
      "jdbc:postgresql://localhost:5432/transactions",
      "user",
      "password",
      scala.concurrent.ExecutionContext.global
    )

  private def buildProducerSettings(): ProducerSettings[IO, String, String] =
    ProducerSettings[IO, String, String]
      .withBootstrapServers("localhost:9092")
      .withAcks(Acks.All)
```

### Integration Test

```scala
import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.freespec.AsyncFreeSpec
import fs2.*

class PipelineSpec extends AsyncFreeSpec with AsyncIOSpec with Matchers:

  "CSV Pipeline" - {
    "should parse valid CSV rows" in {
      val csvLines = Stream(
        "id,userId,amount,currency,date,category",
        "txn-1,user-1,100.00,THB,2024-01-15,food",
        "txn-2,user-2,250.50,THB,2024-01-16,shopping"
      )

      csvLines
        .through(CsvPipeline.csvToTransaction)
        .compile
        .toList
        .asserting: results =>
          results should have length 2
          results.forall(_.isRight) shouldBe true
    }

    "should return errors for invalid rows" in {
      val csvLines = Stream(
        "id,userId,amount,currency,date,category",
        "txn-1,user-1,not-a-number,THB,2024-01-15,food"
      )

      csvLines
        .through(CsvPipeline.csvToTransaction)
        .compile
        .toList
        .asserting: results =>
          results should have length 1
          results.head.isLeft shouldBe true
    }
  }

  "Windowed Aggregations" - {
    "should compute window stats correctly" in {
      val transactions = List(
        Transaction("t1", "u1", BigDecimal("100"), "THB",
          java.time.LocalDate.now(), Category.Food),
        Transaction("t2", "u1", BigDecimal("200"), "THB",
          java.time.LocalDate.now(), Category.Shopping),
        Transaction("t3", "u2", BigDecimal("50"), "THB",
          java.time.LocalDate.now(), Category.Transport)
      )

      val stats = WindowedAggregations.computeWindowStats(transactions)

      IO {
        stats.count shouldBe 3
        stats.totalAmount shouldBe BigDecimal("350")
        stats.avgAmount shouldBe BigDecimal("350") / 3
      }
    }
  }
```

---

## สรุป

Data Pipeline ด้วย fs2 และ Kafka มีองค์ประกอบสำคัญ:

| Component | หน้าที่ |
|-----------|---------|
| fs2 Stream | Lazy, composable stream processing |
| CSV Parser | Extract และ parse ข้อมูลจาก files |
| Windowed Agg | Aggregate ข้อมูลตาม time/size windows |
| Kafka Consumer | รับข้อมูลจาก Kafka topics |
| Kafka Producer | ส่งผลลัพธ์ไปยัง Kafka topics |
| Dead Letter Queue | จัดการ errors โดยไม่ทำให้ pipeline หยุด |
| Metrics | ติดตาม throughput และ error rate |
| Circuit Breaker | ป้องกัน cascade failures |

**Best Practices:**
- ใช้ `parEvalMap` สำหรับ concurrent processing
- แยก error path ออกจาก success path อย่างชัดเจน
- Commit Kafka offsets หลัง processing สำเร็จเท่านั้น
- ใช้ batch writes เพื่อลด DB round trips
- Monitor metrics แบบ real-time

---

*[← Part 49: Real-time Chat](part-49-realtime-chat.md) | [Part 51: Saga Pattern →](part-51-saga-pattern.md)*
