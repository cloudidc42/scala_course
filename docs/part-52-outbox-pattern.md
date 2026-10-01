# Part 52: Transactional Outbox Pattern

## สารบัญ

1. [ปัญหา Dual-Write Consistency](#ปัญหา-dual-write-consistency)
2. [Outbox Pattern คืออะไร](#outbox-pattern-คืออะไร)
3. [Outbox Table ใน PostgreSQL](#outbox-table-ใน-postgresql)
4. [Polling Publisher](#polling-publisher)
5. [At-Least-Once Delivery](#at-least-once-delivery)
6. [Idempotency Keys](#idempotency-keys)
7. [การ Implementation สมบูรณ์](#การ-implementation-สมบูรณ์)
8. [Change Data Capture (CDC) Alternative](#change-data-capture-cdc-alternative)

---

## ปัญหา Dual-Write Consistency

ปัญหาคลาสสิกที่เกิดขึ้นเมื่อต้องเขียนข้อมูลไปสองที่พร้อมกัน:

```
❌ Naive Approach - อันตราย!

BEGIN TRANSACTION
  INSERT INTO orders (id, status) VALUES ('ord-1', 'confirmed')
COMMIT

-- ถ้า crash ระหว่างนี้ หรือ Kafka timeout?
KAFKA.produce("order-confirmed", orderId = "ord-1")  -- ❌ อาจ fail!

ผลลัพธ์: Order ถูก commit ใน DB แต่ event ไม่ถูกส่ง
         หรือ Event ถูกส่งแต่ DB rollback → inconsistent!
```

```
✅ Outbox Pattern - ปลอดภัย!

BEGIN TRANSACTION
  INSERT INTO orders (id, status) VALUES ('ord-1', 'confirmed')
  INSERT INTO outbox (event_type, payload) VALUES ('ORDER_CONFIRMED', '{"orderId": "ord-1"}')
COMMIT  ← ทั้งสองต้องสำเร็จพร้อมกัน!

-- Worker แยกต่างหากอ่าน outbox และ publish ไปยัง Kafka
-- ถ้า publish ล้มเหลว → retry จาก outbox (ไม่ตกหาย!)
```

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.typelevel"  %% "cats-effect"      % "3.5.4",
  "co.fs2"         %% "fs2-core"         % "3.10.0",
  "org.tpolecat"   %% "doobie-core"      % "1.0.0-RC4",
  "org.tpolecat"   %% "doobie-postgres"  % "1.0.0-RC4",
  "org.tpolecat"   %% "doobie-hikari"    % "1.0.0-RC4",
  "com.github.fd4s" %% "fs2-kafka"       % "3.5.1",
  "io.circe"       %% "circe-core"       % "0.14.9",
  "io.circe"       %% "circe-generic"    % "0.14.9",
  "io.circe"       %% "circe-parser"     % "0.14.9"
)
```

---

## Outbox Pattern คืออะไร

Outbox Pattern ทำงานโดยการเขียน events ลง outbox table ใน DB เดียวกับ business data ภายใน transaction เดียวกัน จากนั้น background process จะอ่าน outbox table และ publish events ไปยัง message broker

```
┌──────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                     │
│                                                           │
│  ┌────────────┐  ┌─────────────────────────────────────┐ │
│  │  orders    │  │           outbox                    │ │
│  │────────────│  │─────────────────────────────────────│ │
│  │ id         │  │ id          (UUID)                  │ │
│  │ status     │  │ event_type  (VARCHAR)                │ │
│  │ ...        │  │ aggregate_id (VARCHAR)               │ │
│  └────────────┘  │ payload     (JSONB)                  │ │
│        │         │ status      (pending/sent/failed)    │ │
│        │ SAME    │ created_at  (TIMESTAMPTZ)            │ │
│        └─── TX ──│ sent_at     (TIMESTAMPTZ NULL)       │ │
│                  │ retry_count (INT)                    │ │
│                  └─────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
                                │
                     ┌──────────▼──────────┐
                     │   Polling Publisher  │
                     │ (reads + publishes) │
                     └──────────┬──────────┘
                                │
                     ┌──────────▼──────────┐
                     │    Apache Kafka     │
                     └─────────────────────┘
```

---

## Outbox Table ใน PostgreSQL

### Database Schema

```sql
-- Outbox table
CREATE TABLE outbox (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(100) NOT NULL,
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    VARCHAR(255) NOT NULL,
    payload         JSONB        NOT NULL,
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    sent_at         TIMESTAMPTZ,
    retry_count     INT          NOT NULL DEFAULT 0,
    last_error      TEXT,
    idempotency_key VARCHAR(255) UNIQUE,
    
    CONSTRAINT status_check CHECK (status IN ('PENDING', 'PROCESSING', 'SENT', 'FAILED'))
);

-- Indexes for efficient polling
CREATE INDEX idx_outbox_status_created ON outbox(status, created_at)
    WHERE status IN ('PENDING', 'FAILED');

CREATE INDEX idx_outbox_aggregate ON outbox(aggregate_type, aggregate_id);

-- Partition by status for better performance (optional)
-- For high-volume systems, consider time-based partitioning
```

### Scala Models

```scala
import java.util.UUID
import java.time.Instant
import io.circe.*
import io.circe.generic.semiauto.*

case class OutboxEvent(
  id: UUID,
  eventType: String,
  aggregateType: String,
  aggregateId: String,
  payload: io.circe.Json,
  status: OutboxStatus,
  createdAt: Instant,
  sentAt: Option[Instant],
  retryCount: Int,
  lastError: Option[String],
  idempotencyKey: Option[String]
)

enum OutboxStatus:
  case Pending, Processing, Sent, Failed

// For creating new outbox events
case class NewOutboxEvent(
  eventType: String,
  aggregateType: String,
  aggregateId: String,
  payload: io.circe.Json,
  idempotencyKey: Option[String] = None
)

object OutboxEvent:
  given Encoder[OutboxEvent] = deriveEncoder
  given Decoder[OutboxEvent] = deriveDecoder
```

---

## Polling Publisher

### Outbox Repository

```scala
import cats.effect.*
import doobie.*
import doobie.implicits.*
import doobie.postgres.implicits.*
import doobie.postgres.circe.jsonb.implicits.*
import java.util.UUID
import java.time.Instant

class OutboxRepository(xa: Transactor[IO]):

  // เขียน outbox event (ใช้ภายใน transaction)
  def insert(event: NewOutboxEvent): ConnectionIO[UUID] =
    sql"""
      INSERT INTO outbox (event_type, aggregate_type, aggregate_id, payload, idempotency_key)
      VALUES (
        ${event.eventType},
        ${event.aggregateType},
        ${event.aggregateId},
        ${event.payload},
        ${event.idempotencyKey}
      )
      RETURNING id
    """.query[UUID].unique

  // Fetch pending events with pessimistic locking (SKIP LOCKED = ไม่รอ lock)
  def fetchPending(batchSize: Int = 100): IO[List[OutboxEvent]] =
    sql"""
      SELECT id, event_type, aggregate_type, aggregate_id, payload,
             status, created_at, sent_at, retry_count, last_error, idempotency_key
      FROM outbox
      WHERE status IN ('PENDING', 'FAILED')
        AND retry_count < 5
      ORDER BY created_at ASC
      LIMIT $batchSize
      FOR UPDATE SKIP LOCKED
    """.query[OutboxEvent].to[List].transact(xa)

  // Mark as processing (prevents other instances from picking up same events)
  def markProcessing(ids: List[UUID]): IO[Unit] =
    NonEmptyList.fromList(ids).fold(IO.unit): nel =>
      val inClause = nel.map(_ => "?").toList.mkString(",")
      Update[UUID](
        s"UPDATE outbox SET status = 'PROCESSING' WHERE id IN (${nel.map(_ => "?").toList.mkString(",")})"
      ).updateMany(nel).transact(xa).void

  // Mark as sent
  def markSent(id: UUID): IO[Unit] =
    sql"""
      UPDATE outbox
      SET status = 'SENT',
          sent_at = NOW()
      WHERE id = $id
    """.update.run.transact(xa).void

  // Mark as failed (with retry count increment)
  def markFailed(id: UUID, error: String): IO[Unit] =
    sql"""
      UPDATE outbox
      SET status = CASE
            WHEN retry_count + 1 >= 5 THEN 'FAILED'
            ELSE 'PENDING'
          END,
          retry_count = retry_count + 1,
          last_error = $error
      WHERE id = $id
    """.update.run.transact(xa).void

  // Clean up old sent events
  def deleteOldSent(olderThanDays: Int = 30): IO[Int] =
    sql"""
      DELETE FROM outbox
      WHERE status = 'SENT'
        AND sent_at < NOW() - INTERVAL '${olderThanDays} days'
    """.update.run.transact(xa)
```

### Business Logic with Outbox

```scala
import cats.effect.*
import doobie.*
import doobie.implicits.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*

// Order domain
case class Order(id: String, customerId: String, status: String, amount: BigDecimal)
case class OrderConfirmedEvent(orderId: String, customerId: String, amount: BigDecimal)

given Encoder[OrderConfirmedEvent] = deriveEncoder

class OrderService(
  xa: Transactor[IO],
  outboxRepo: OutboxRepository
):

  // ✅ Atomic: save order + outbox event in same transaction
  def confirmOrder(orderId: String): IO[Order] =

    val transaction: ConnectionIO[Order] = for
      // 1. Update order status
      order <- sql"""
        UPDATE orders
        SET status = 'CONFIRMED', confirmed_at = NOW()
        WHERE id = $orderId AND status = 'PENDING'
        RETURNING id, customer_id, status, amount
      """.query[Order].unique

      // 2. Insert outbox event (SAME transaction!)
      event = NewOutboxEvent(
        eventType     = "ORDER_CONFIRMED",
        aggregateType = "Order",
        aggregateId   = orderId,
        payload       = OrderConfirmedEvent(
          orderId    = order.id,
          customerId = order.customerId,
          amount     = order.amount
        ).asJson,
        idempotencyKey = Some(s"order-confirmed-$orderId")
      )
      _ <- outboxRepo.insert(event)  // Runs in same ConnectionIO

    yield order

    // Run both operations in a single DB transaction
    transaction.transact(xa)

  // Multiple events in one transaction
  def cancelOrder(orderId: String, reason: String): IO[Order] =

    val transaction: ConnectionIO[Order] = for
      order <- sql"""
        UPDATE orders SET status = 'CANCELLED' WHERE id = $orderId
        RETURNING id, customer_id, status, amount
      """.query[Order].unique

      // Insert multiple events in the same transaction
      _ <- outboxRepo.insert(NewOutboxEvent(
        eventType     = "ORDER_CANCELLED",
        aggregateType = "Order",
        aggregateId   = orderId,
        payload       = io.circe.Json.obj(
          "orderId" -> orderId.asJson,
          "reason"  -> reason.asJson
        )
      ))

      _ <- outboxRepo.insert(NewOutboxEvent(
        eventType     = "INVENTORY_RELEASE_REQUESTED",
        aggregateType = "Order",
        aggregateId   = orderId,
        payload       = io.circe.Json.obj("orderId" -> orderId.asJson)
      ))

    yield order

    transaction.transact(xa)
```

---

## At-Least-Once Delivery

### Polling Publisher Implementation

```scala
import cats.effect.*
import cats.syntax.all.*
import fs2.*
import fs2.kafka.*
import scala.concurrent.duration.*
import java.util.UUID

class OutboxPollingPublisher(
  outboxRepo: OutboxRepository,
  producer: KafkaProducer.Fiber[IO, String, String],
  config: PublisherConfig
):

  case class PublisherConfig(
    pollInterval: FiniteDuration = 100.millis,
    batchSize: Int               = 100,
    maxConcurrency: Int          = 4
  )

  // Main polling loop
  def run: Stream[IO, Unit] =
    Stream
      .awakeEvery[IO](config.pollInterval)
      .evalMap(_ => publishBatch)
      .handleErrorWith: e =>
        Stream.eval(IO.println(s"Publisher error: $e")) >>
        Stream.sleep[IO](5.seconds) >>
        run  // Restart on error

  private def publishBatch: IO[Unit] =
    for
      events <- outboxRepo.fetchPending(config.batchSize)
      _      <- if events.isEmpty then IO.unit
                else
                  IO.println(s"Publishing ${events.length} events") >>
                  events
                    .parTraverseN(config.maxConcurrency)(publishEvent)
                    .void
    yield ()

  private def publishEvent(event: OutboxEvent): IO[Unit] =
    val topic  = topicForEventType(event.eventType)
    val key    = event.aggregateId
    val value  = event.payload.noSpaces

    val record = ProducerRecord(topic, key, value)
      .withHeader("event-id", event.id.toString)
      .withHeader("event-type", event.eventType)
      .withHeader("created-at", event.createdAt.toString)

    producer
      .produce(ProducerRecords.one(record))
      .flatten
      .flatMap(_ => outboxRepo.markSent(event.id))
      .handleErrorWith: e =>
        outboxRepo.markFailed(event.id, e.getMessage) >>
        IO.println(s"Failed to publish event ${event.id}: ${e.getMessage}")

  private def topicForEventType(eventType: String): String =
    eventType match
      case "ORDER_CONFIRMED"              => "orders.confirmed"
      case "ORDER_CANCELLED"              => "orders.cancelled"
      case "INVENTORY_RELEASE_REQUESTED"  => "inventory.release-requested"
      case "PAYMENT_PROCESSED"            => "payments.processed"
      case unknown                        => s"events.${unknown.toLowerCase.replace("_", "-")}"

object OutboxPollingPublisher:
  def resource(
    outboxRepo: OutboxRepository,
    kafkaConfig: ProducerSettings[IO, String, String],
    config: PublisherConfig = PublisherConfig()
  ): Resource[IO, OutboxPollingPublisher] =
    KafkaProducer
      .resource(kafkaConfig)
      .map(producer => new OutboxPollingPublisher(outboxRepo, producer, config))

  // Start publisher in background
  def start(publisher: OutboxPollingPublisher): Resource[IO, Unit] =
    publisher.run.compile.drain.background.void
```

### Multi-Instance Publisher (Distributed)

```scala
import cats.effect.*
import cats.effect.Ref
import java.net.InetAddress
import scala.concurrent.duration.*

// Distributed lock using PostgreSQL advisory locks
class DistributedLock(xa: Transactor[IO]):

  // Try to acquire advisory lock (non-blocking)
  def tryAcquire(lockId: Long): IO[Boolean] =
    import doobie.*
    import doobie.implicits.*
    sql"SELECT pg_try_advisory_lock($lockId)"
      .query[Boolean]
      .unique
      .transact(xa)

  def release(lockId: Long): IO[Unit] =
    import doobie.*
    import doobie.implicits.*
    sql"SELECT pg_advisory_unlock($lockId)"
      .query[Boolean]
      .unique
      .transact(xa)
      .void

// Only one instance runs the publisher at a time
class SingleLeaderPublisher(
  publisher: OutboxPollingPublisher,
  lock: DistributedLock,
  lockId: Long = 12345L  // Unique ID for this lock
):

  def run: Stream[IO, Unit] =
    Stream.eval(lock.tryAcquire(lockId)).flatMap: acquired =>
      if acquired then
        IO.println("Acquired leader lock, starting publisher").toStream ++
        publisher.run
          .onFinalize(lock.release(lockId))
      else
        IO.println("Another instance is leader, waiting...").toStream ++
        Stream.sleep[IO](10.seconds) >>
        run  // Try again

  extension [F[_], A](io: IO[A])
    def toStream: Stream[IO, A] = Stream.eval(io)
```

---

## Idempotency Keys

### Consumer-Side Idempotency

```scala
import cats.effect.*
import doobie.*
import doobie.implicits.*
import java.util.UUID

// Track processed events to prevent duplicate processing
class IdempotencyStore(xa: Transactor[IO]):

  // Check if event was already processed
  def isProcessed(eventId: String): IO[Boolean] =
    sql"""
      SELECT EXISTS(
        SELECT 1 FROM processed_events WHERE event_id = $eventId
      )
    """.query[Boolean].unique.transact(xa)

  // Mark event as processed
  def markProcessed(eventId: String, consumerId: String): IO[Unit] =
    sql"""
      INSERT INTO processed_events (event_id, consumer_id, processed_at)
      VALUES ($eventId, $consumerId, NOW())
      ON CONFLICT (event_id, consumer_id) DO NOTHING
    """.update.run.transact(xa).void

  // Clean up old records
  def cleanup(olderThanDays: Int = 7): IO[Int] =
    sql"""
      DELETE FROM processed_events
      WHERE processed_at < NOW() - INTERVAL '${olderThanDays} days'
    """.update.run.transact(xa)

// Schema
val createProcessedEventsTable: String = """
  CREATE TABLE IF NOT EXISTS processed_events (
    event_id     VARCHAR(255) NOT NULL,
    consumer_id  VARCHAR(100) NOT NULL,
    processed_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    
    PRIMARY KEY (event_id, consumer_id)
  );
  
  CREATE INDEX IF NOT EXISTS idx_processed_events_cleanup
    ON processed_events(processed_at);
"""

// Idempotent event handler
class IdempotentEventHandler[E](
  handler: E => IO[Unit],
  idempotencyStore: IdempotencyStore,
  consumerId: String
):

  def handle(eventId: String, event: E): IO[Unit] =
    idempotencyStore.isProcessed(eventId).flatMap:
      case true =>
        IO.println(s"Event $eventId already processed, skipping")
      case false =>
        handler(event).flatMap: _ =>
          idempotencyStore.markProcessed(eventId, consumerId)
```

### Producer-Side Idempotency

```scala
import cats.effect.*
import doobie.*
import doobie.implicits.*

// Prevent duplicate outbox inserts
class IdempotentOutboxWriter(
  outboxRepo: OutboxRepository
):

  // Write event only if idempotency key not seen before
  def writeIfNew(event: NewOutboxEvent): ConnectionIO[Option[UUID]] =
    event.idempotencyKey match
      case None =>
        outboxRepo.insert(event).map(Some.apply)

      case Some(key) =>
        // Check if already exists
        sql"""
          SELECT id FROM outbox WHERE idempotency_key = $key
        """.query[UUID].option.flatMap:
          case Some(existingId) =>
            // Already written, return existing ID
            doobie.free.connection.pure(Some(existingId))
          case None =>
            outboxRepo.insert(event).map(Some.apply)
```

---

## การ Implementation สมบูรณ์

### Complete Module

```scala
import cats.effect.*
import cats.syntax.all.*
import fs2.*
import fs2.kafka.*
import doobie.*
import doobie.implicits.*
import doobie.postgres.implicits.*
import scala.concurrent.duration.*
import java.util.UUID
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*

// ==================== Models ====================

case class OutboxMessage(
  id: UUID,
  eventType: String,
  aggregateType: String,
  aggregateId: String,
  payload: Json,
  status: String,
  createdAt: java.time.Instant,
  retryCount: Int
)

// ==================== Repository ====================

class OutboxDao(xa: Transactor[IO]):

  def insertTx(
    eventType: String,
    aggregateType: String,
    aggregateId: String,
    payload: Json,
    idempotencyKey: Option[String] = None
  ): ConnectionIO[UUID] =
    sql"""
      INSERT INTO outbox (event_type, aggregate_type, aggregate_id, payload, idempotency_key)
      VALUES ($eventType, $aggregateType, $aggregateId, $payload::jsonb, $idempotencyKey)
      RETURNING id
    """.query[UUID].unique

  def claimBatch(batchSize: Int): IO[List[OutboxMessage]] =
    sql"""
      WITH claimed AS (
        SELECT id FROM outbox
        WHERE status = 'PENDING' AND retry_count < 5
        ORDER BY created_at
        LIMIT $batchSize
        FOR UPDATE SKIP LOCKED
      )
      UPDATE outbox SET status = 'PROCESSING'
      WHERE id IN (SELECT id FROM claimed)
      RETURNING id, event_type, aggregate_type, aggregate_id,
                payload, status, created_at, retry_count
    """.query[OutboxMessage].to[List].transact(xa)

  def ack(id: UUID): IO[Unit] =
    sql"UPDATE outbox SET status = 'SENT', sent_at = NOW() WHERE id = $id"
      .update.run.transact(xa).void

  def nack(id: UUID, error: String): IO[Unit] =
    sql"""
      UPDATE outbox
      SET status = CASE WHEN retry_count + 1 >= 5 THEN 'DEAD' ELSE 'PENDING' END,
          retry_count = retry_count + 1,
          last_error = $error
      WHERE id = $id
    """.update.run.transact(xa).void

// ==================== Publisher ====================

class OutboxPublisher(
  dao: OutboxDao,
  producer: KafkaProducer.Fiber[IO, String, String],
  batchSize: Int = 50,
  pollInterval: FiniteDuration = 200.millis
):

  def stream: Stream[IO, Unit] =
    Stream
      .awakeEvery[IO](pollInterval)
      .evalMap(_ => processBatch)
      .void

  private def processBatch: IO[Int] =
    for
      messages <- dao.claimBatch(batchSize)
      _        <- messages.parTraverseN(8)(sendMessage)
    yield messages.length

  private def sendMessage(msg: OutboxMessage): IO[Unit] =
    val record = ProducerRecord(
      topic = s"domain.${msg.aggregateType.toLowerCase}.${msg.eventType.toLowerCase}",
      key   = msg.aggregateId,
      value = msg.payload.noSpaces
    )

    producer
      .produce(ProducerRecords.one(record))
      .flatten
      .flatMap(_ => dao.ack(msg.id))
      .handleErrorWith: e =>
        dao.nack(msg.id, e.getMessage)

// ==================== Application Wiring ====================

object OutboxApp extends IOApp:

  def buildTransactor(config: DbConfig): Resource[IO, Transactor[IO]] =
    import doobie.hikari.*
    HikariTransactor.newHikariTransactor[IO](
      config.driver,
      config.url,
      config.user,
      config.password,
      scala.concurrent.ExecutionContext.global
    )

  def buildKafkaProducer(
    bootstrapServers: String
  ): Resource[IO, KafkaProducer.Fiber[IO, String, String]] =
    val settings = ProducerSettings[IO, String, String]
      .withBootstrapServers(bootstrapServers)
      .withAcks(Acks.All)
      .withEnableIdempotence(true)  // Kafka-level idempotence
      .withMaxInFlightRequestsPerConnection(1)
    KafkaProducer.resource(settings)

  def run(args: List[String]): IO[ExitCode] =
    val program = for
      xa       <- buildTransactor(DbConfig.fromEnv)
      producer <- buildKafkaProducer("localhost:9092")
      dao       = new OutboxDao(xa)
      publisher = new OutboxPublisher(dao, producer)
    yield publisher

    program.use: publisher =>
      publisher.stream.compile.drain.as(ExitCode.Success)

case class DbConfig(driver: String, url: String, user: String, password: String)
object DbConfig:
  def fromEnv: DbConfig = DbConfig(
    driver   = "org.postgresql.Driver",
    url      = sys.env.getOrElse("DB_URL", "jdbc:postgresql://localhost:5432/mydb"),
    user     = sys.env.getOrElse("DB_USER", "postgres"),
    password = sys.env.getOrElse("DB_PASS", "password")
  )
```

### Usage Example

```scala
import cats.effect.*
import doobie.*
import doobie.implicits.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*

// Business service that uses outbox
class PaymentService(xa: Transactor[IO], outboxDao: OutboxDao):

  given Encoder[PaymentProcessed] = deriveEncoder

  def processPayment(
    orderId: String,
    customerId: String,
    amount: BigDecimal
  ): IO[PaymentResult] =

    val transaction: ConnectionIO[PaymentResult] = for

      // 1. Insert payment record
      paymentId <- sql"""
        INSERT INTO payments (order_id, customer_id, amount, status)
        VALUES ($orderId, $customerId, $amount, 'PROCESSED')
        RETURNING id
      """.query[String].unique

      // 2. Insert outbox event (SAME transaction - guaranteed consistency!)
      _ <- outboxDao.insertTx(
        eventType     = "PAYMENT_PROCESSED",
        aggregateType = "Payment",
        aggregateId   = paymentId,
        payload       = PaymentProcessed(
          paymentId  = paymentId,
          orderId    = orderId,
          customerId = customerId,
          amount     = amount
        ).asJson,
        idempotencyKey = Some(s"payment-$orderId")
      )

    yield PaymentResult(paymentId, "PROCESSED")

    transaction.transact(xa)

case class PaymentProcessed(
  paymentId: String,
  orderId: String,
  customerId: String,
  amount: BigDecimal
)

case class PaymentResult(paymentId: String, status: String)
```

---

## Change Data Capture (CDC) Alternative

นอกจาก Polling Approach แล้ว เราสามารถใช้ Debezium เพื่อ capture การเปลี่ยนแปลงจาก PostgreSQL WAL โดยตรง:

```scala
// CDC-based approach using Debezium
// ไม่ต้องมี polling - CDC push changes ไปยัง Kafka โดยตรง

// Debezium configuration (ใน Kafka Connect):
val debeziumConfig = Map(
  "name"                          -> "outbox-connector",
  "connector.class"               -> "io.debezium.connector.postgresql.PostgresConnector",
  "database.hostname"             -> "localhost",
  "database.port"                 -> "5432",
  "database.user"                 -> "postgres",
  "database.password"             -> "secret",
  "database.dbname"               -> "mydb",
  "table.include.list"            -> "public.outbox",
  "transforms"                    -> "outbox",
  "transforms.outbox.type"        -> "io.debezium.transforms.outbox.EventRouter",
  "transforms.outbox.table.field.event.type"    -> "event_type",
  "transforms.outbox.table.field.event.id"      -> "id",
  "transforms.outbox.route.by.field"            -> "aggregate_type"
)

// Scala consumer for CDC events
class CdcOutboxConsumer(
  kafkaSettings: ConsumerSettings[IO, String, String]
):

  def consumeOrderEvents: Stream[IO, OrderEvent] =
    KafkaConsumer
      .stream(kafkaSettings)
      .subscribeTo("outbox.public.Order")  // Debezium creates topic per aggregate_type
      .records
      .evalMapFilter: record =>
        parseOutboxRecord(record.record.value)
          .traverse(event =>
            record.offset.commit.as(event)
          )

  private def parseOutboxRecord(json: String): IO[Option[OrderEvent]] =
    import io.circe.parser.*
    parse(json).flatMap(_.as[DebeziumOutboxEvent]) match
      case Right(evt) if evt.op == "c" =>  // Only INSERT operations
        parseOrderEvent(evt.after).map(Some.apply)
      case Right(_) => IO.pure(None)
      case Left(e)  =>
        IO.println(s"Failed to parse CDC event: $e").as(None)

case class DebeziumOutboxEvent(
  op: String,  // c = create, u = update, d = delete
  before: Option[io.circe.Json],
  after: Option[io.circe.Json]
)
```

---

## Monitoring และ Alerting

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

class OutboxMonitor(dao: OutboxDao, alertService: AlertService):

  // Monitor for stuck events
  def runMonitoring: Stream[IO, Unit] =
    Stream
      .awakeEvery[IO](1.minute)
      .evalMap(_ => checkHealth)

  private def checkHealth: IO[Unit] =
    for
      pendingCount    <- dao.countByStatus("PENDING")
      processingCount <- dao.countByStatus("PROCESSING")
      deadCount       <- dao.countByStatus("DEAD")
      _               <- IO.println(
                           s"Outbox health: pending=$pendingCount processing=$processingCount dead=$deadCount"
                         )
      _               <- if pendingCount > 1000 then
                           alertService.alert(s"High pending outbox count: $pendingCount")
                         else IO.unit
      _               <- if deadCount > 0 then
                           alertService.alert(s"Dead outbox events: $deadCount")
                         else IO.unit
    yield ()

trait AlertService:
  def alert(message: String): IO[Unit]
```

---

## Testing

```scala
import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.freespec.AsyncFreeSpec
import org.scalatest.matchers.should.Matchers
import doobie.implicits.*

class OutboxIntegrationSpec extends AsyncFreeSpec
    with AsyncIOSpec
    with Matchers:

  // ใช้ testcontainers หรือ embedded postgres สำหรับ testing
  "OutboxDao" - {

    "should insert and claim events atomically" in {
      withTestDb: xa =>
        val dao = new OutboxDao(xa)

        val insertAndClaim = for
          // Insert event in transaction
          id <- dao.insertTx(
            "ORDER_CONFIRMED", "Order", "ord-1",
            io.circe.Json.obj("orderId" -> io.circe.Json.fromString("ord-1"))
          ).transact(xa)

          // Claim it
          claimed <- dao.claimBatch(10)

        yield (id, claimed)

        insertAndClaim.asserting: (id, claimed) =>
          claimed should have length 1
          claimed.head.id shouldBe id
          claimed.head.status shouldBe "PROCESSING"
    }

    "should handle idempotent inserts" in {
      withTestDb: xa =>
        val dao = new OutboxDao(xa)

        val idempKey = "unique-key-1"

        for
          id1 <- dao.insertTx(
            "EVENT", "Agg", "agg-1",
            io.circe.Json.Null,
            Some(idempKey)
          ).transact(xa)
          // Second insert with same key
          id2 <- dao.insertTx(
            "EVENT", "Agg", "agg-1",
            io.circe.Json.Null,
            Some(idempKey)
          ).transact(xa).attempt
        yield id2.isLeft shouldBe true  // Should fail on duplicate key
    }
  }

  def withTestDb[A](test: Transactor[IO] => IO[A]): IO[A] = ???
```

---

## สรุป

Transactional Outbox Pattern แก้ปัญหา dual-write ได้อย่างสมบูรณ์:

| ปัญหา | วิธีแก้ใน Outbox Pattern |
|-------|------------------------|
| Dual-write inconsistency | เขียน DB + outbox ใน transaction เดียวกัน |
| Lost events | At-least-once delivery + retry |
| Duplicate processing | Idempotency keys ทั้งฝั่ง producer และ consumer |
| Performance | SKIP LOCKED + batch processing |
| High availability | Advisory locks สำหรับ leader election |
| Monitoring | Count events ตาม status + alerting |

**Best Practices:**
1. ใช้ `FOR UPDATE SKIP LOCKED` เพื่อ distributed processing
2. Set `retry_count` limit เพื่อป้องกัน infinite retry
3. เก็บ `idempotency_key` เพื่อป้องกัน duplicate events
4. Clean up old `SENT` records เป็นประจำ
5. Monitor `DEAD` events และ alert ทีม

---

*[← Part 51: Saga Pattern](part-51-saga-pattern.md) | [Part 53: Rate Limiting →](part-53-rate-limiting.md)*
