# ส่วนที่ 78: Event-Driven Architecture

## สารบัญ

1. [Domain Events vs Integration Events](#domain-events-vs-integration-events)
2. [Event Bus: In-Process vs Distributed](#event-bus-in-process-vs-distributed)
3. [Event Sourcing และ Projections](#event-sourcing-และ-projections)
4. [Eventual Consistency](#eventual-consistency)
5. [Two-Phase Commit vs Saga Pattern](#two-phase-commit-vs-saga-pattern)
6. [Event Schema Design](#event-schema-design)
7. [Complete Event-Driven System](#complete-event-driven-system)
8. [สรุป](#สรุป)

---

## Domain Events vs Integration Events

### ความแตกต่างระหว่าง Domain Events และ Integration Events

```scala
// src/main/scala/events/EventTypes.scala
package events

import java.util.UUID
import java.time.Instant

/**
 * Domain Events:
 * - เกิดขึ้นภายใน Bounded Context เดียว
 * - ประมวลผลแบบ synchronous หรือ async ภายใน aggregate
 * - รายละเอียดมาก (domain-specific)
 * - ใช้ภาษา domain (ubiquitous language)
 *
 * Integration Events:
 * - ส่งระหว่าง Bounded Contexts หรือ services
 * - ประมวลผลแบบ async ผ่าน message broker
 * - รายละเอียดน้อยกว่า (เฉพาะข้อมูลที่จำเป็น)
 * - ต้องมี backward compatibility
 */

// Base event traits
sealed trait Event:
  def eventId: UUID
  def occurredAt: Instant
  def correlationId: Option[UUID]
  def causationId: Option[UUID]

// Domain Events - รายละเอียดสูง, เฉพาะ bounded context
sealed trait DomainEvent extends Event

case class OrderPlaced(
  eventId: UUID = UUID.randomUUID(),
  orderId: UUID,
  customerId: UUID,
  items: List[OrderItemEvent],
  totalAmount: BigDecimal,
  shippingAddress: AddressEvent,
  paymentMethod: PaymentMethodEvent,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends DomainEvent

case class OrderShipped(
  eventId: UUID = UUID.randomUUID(),
  orderId: UUID,
  trackingNumber: String,
  carrier: String,
  estimatedDelivery: Instant,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends DomainEvent

case class PaymentProcessed(
  eventId: UUID = UUID.randomUUID(),
  orderId: UUID,
  paymentId: UUID,
  amount: BigDecimal,
  currency: String,
  status: PaymentStatus,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends DomainEvent

// Integration Events - น้อยกว่า, ใช้ระหว่าง services
sealed trait IntegrationEvent extends Event:
  def version: Int  // สำคัญมากสำหรับ backward compatibility

case class OrderConfirmedIntegration(
  eventId: UUID = UUID.randomUUID(),
  orderId: String,         // ใช้ String แทน UUID สำหรับ interoperability
  customerId: String,
  totalAmount: BigDecimal,
  itemCount: Int,          // summary แทนรายละเอียด
  version: Int = 1,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends IntegrationEvent

case class InventoryReservedIntegration(
  eventId: UUID = UUID.randomUUID(),
  orderId: String,
  items: List[ReservedItemSummary],
  version: Int = 1,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends IntegrationEvent

// Supporting types
case class OrderItemEvent(productId: UUID, quantity: Int, price: BigDecimal)
case class AddressEvent(street: String, city: String, country: String, postalCode: String)
case class PaymentMethodEvent(type_: String, lastFour: Option[String])
case class ReservedItemSummary(productId: String, quantity: Int)

enum PaymentStatus:
  case Pending, Completed, Failed, Refunded
```

---

## Event Bus: In-Process vs Distributed

### In-Process Event Bus

```scala
// src/main/scala/events/InProcessEventBus.scala
package events

import cats.effect.*
import cats.effect.std.{Queue, PubSub}
import cats.syntax.all.*
import fs2.Stream

// In-process event bus สำหรับ domain events
trait EventBus[F[_]]:
  def publish[E <: Event](event: E): F[Unit]
  def subscribe[E <: Event](handler: EventHandler[E]): F[Unit]

trait EventHandler[E <: Event]:
  def handle(event: E): IO[Unit]

// Simple in-process implementation
class InMemoryEventBus extends EventBus[IO]:
  
  private val handlers = scala.collection.mutable.Map[Class[?], List[EventHandler[?]]]()
  
  def publish[E <: Event](event: E): IO[Unit] =
    IO.defer {
      val eventClass = event.getClass
      val matchingHandlers = handlers
        .filter((cls, _) => cls.isAssignableFrom(eventClass))
        .values.flatten.toList
      
      matchingHandlers.traverse_ { handler =>
        handler.asInstanceOf[EventHandler[E]].handle(event)
          .handleErrorWith { e =>
            IO.println(s"Error handling event ${event.eventId}: ${e.getMessage}")
          }
      }
    }
  
  def subscribe[E <: Event](handler: EventHandler[E])(using ct: reflect.ClassTag[E]): IO[Unit] =
    IO.delay {
      val cls = ct.runtimeClass
      val existing = handlers.getOrElse(cls, List.empty)
      handlers.update(cls, existing :+ handler)
    }

// Async event bus with retry
class AsyncEventBus(
  queue: Queue[IO, (Event, Int)],  // event + retry count
  maxRetries: Int = 3
) extends EventBus[IO]:
  
  private val handlers = scala.collection.mutable.Map[String, List[EventHandler[?]]]()
  
  def publish[E <: Event](event: E): IO[Unit] =
    queue.offer((event, 0))
  
  def subscribe[E <: Event](handler: EventHandler[E]): IO[Unit] =
    IO.delay {
      val key = handler.getClass.getName
      handlers.update(key, handlers.getOrElse(key, List.empty) :+ handler)
    }
  
  def processEvents: IO[Nothing] =
    Stream.fromQueueUnterminated(queue)
      .evalMap { (event, retryCount) =>
        processEvent(event).handleErrorWith { e =>
          if retryCount < maxRetries then
            IO.sleep(scala.concurrent.duration.FiniteDuration(
              (math.pow(2, retryCount) * 1000).toLong,
              java.util.concurrent.TimeUnit.MILLISECONDS
            )) >> queue.offer((event, retryCount + 1))
          else
            IO.println(s"Failed to process event ${event.eventId} after $maxRetries retries: ${e.getMessage}")
        }
      }
      .compile.drain.flatMap(_ => IO.never)
  
  private def processEvent(event: Event): IO[Unit] =
    handlers.values.flatten.toList
      .traverse_ { handler =>
        handler.asInstanceOf[EventHandler[Event]].handle(event)
      }

object AsyncEventBus:
  def make(maxRetries: Int = 3): IO[AsyncEventBus] =
    Queue.bounded[IO, (Event, Int)](1000).map(q => AsyncEventBus(q, maxRetries))
```

### Distributed Event Bus กับ Kafka

```scala
// src/main/scala/events/KafkaEventBus.scala
package events

import cats.effect.*
import cats.syntax.all.*
import fs2.kafka.*
import io.circe.generic.auto.*
import io.circe.syntax.*
import io.circe.parser.*

case class KafkaConfig(
  bootstrapServers: String,
  groupId: String,
  clientId: String
)

class KafkaEventPublisher(config: KafkaConfig):
  
  private val producerSettings = ProducerSettings[IO, String, String]
    .withBootstrapServers(config.bootstrapServers)
    .withClientId(config.clientId)
    .withAcks(Acks.All)
    .withRetries(3)
    .withProperty("enable.idempotence", "true")
    .withProperty("compression.type", "lz4")
  
  def publish[E <: IntegrationEvent](
    event: E,
    topic: String
  )(using io.circe.Encoder[E]): IO[Unit] =
    KafkaProducer.resource(producerSettings).use { producer =>
      val record = ProducerRecord(
        topic,
        event.eventId.toString,  // key = event ID for ordering
        event.asJson.noSpaces
      )
      producer.produce(ProducerRecords.one(record)).flatten.void
    }

class KafkaEventConsumer[E <: IntegrationEvent](
  config: KafkaConfig,
  topic: String,
  handler: E => IO[Unit]
)(using decoder: io.circe.Decoder[E]):
  
  private val consumerSettings = ConsumerSettings[IO, String, String]
    .withBootstrapServers(config.bootstrapServers)
    .withGroupId(config.groupId)
    .withAutoOffsetReset(AutoOffsetReset.Earliest)
    .withEnableAutoCommit(false)
    .withProperty("max.poll.records", "100")
  
  def start: IO[Nothing] =
    KafkaConsumer.stream(consumerSettings)
      .subscribeTo(topic)
      .records
      .mapAsync(maxConcurrent = 10) { committable =>
        val payload = committable.record.value
        
        decode[E](payload) match
          case Right(event) =>
            handler(event)
              .as(committable.offset)
              .handleErrorWith { e =>
                IO.println(s"Failed to handle event: ${e.getMessage}")
                  .as(committable.offset)
              }
          case Left(error) =>
            IO.println(s"Failed to decode event: $error")
              .as(committable.offset)
      }
      .through(commitBatchWithin(100, scala.concurrent.duration.5.seconds))
      .compile.drain.flatMap(_ => IO.never)
```

---

## Event Sourcing และ Projections

### Event Store

```scala
// src/main/scala/eventsourcing/EventStore.scala
package eventsourcing

import cats.effect.IO
import doobie.*
import doobie.implicits.*
import io.circe.Json
import io.circe.generic.auto.*
import io.circe.syntax.*
import java.util.UUID
import java.time.Instant

// Event store record
case class EventRecord(
  id: UUID,
  aggregateType: String,
  aggregateId: UUID,
  eventType: String,
  payload: Json,
  metadata: Json,
  sequenceNumber: Long,
  occurredAt: Instant
)

// Event store interface
trait EventStore[F[_]]:
  def append(
    aggregateId: UUID,
    aggregateType: String,
    expectedVersion: Long,
    events: List[DomainEvent]
  ): F[Unit]
  
  def load(aggregateId: UUID): F[List[EventRecord]]
  def loadFromVersion(aggregateId: UUID, fromVersion: Long): F[List[EventRecord]]

// Doobie implementation
class PostgreSQLEventStore(xa: Transactor[IO]) extends EventStore[IO]:
  
  def append(
    aggregateId: UUID,
    aggregateType: String,
    expectedVersion: Long,
    events: List[DomainEvent]
  ): IO[Unit] =
    val insertions = events.zipWithIndex.map { (event, idx) =>
      val seqNum = expectedVersion + idx + 1
      sql"""
        INSERT INTO domain_events
          (id, aggregate_type, aggregate_id, event_type, payload, metadata, sequence_number, occurred_at)
        VALUES
          (${UUID.randomUUID()}, $aggregateType, $aggregateId,
           ${event.getClass.getSimpleName}, ${serializeEvent(event)},
           ${Json.obj("correlationId" -> event.correlationId.asJson)},
           $seqNum, ${event.occurredAt})
        ON CONFLICT (aggregate_id, sequence_number) DO NOTHING
      """.update.run
    }
    
    // Optimistic concurrency check
    val versionCheck = sql"""
      SELECT MAX(sequence_number) FROM domain_events WHERE aggregate_id = $aggregateId
    """.query[Option[Long]].unique
    
    (for
      currentVersion <- versionCheck
      _ <- currentVersion match
            case Some(v) if v != expectedVersion =>
              FC.raiseError(new RuntimeException(
                s"Version conflict: expected $expectedVersion but got $v"
              ))
            case _ =>
              insertions.traverse_(_.transact(xa).to[ConnectionIO])
    yield ()).transact(xa)
  
  def load(aggregateId: UUID): IO[List[EventRecord]] =
    sql"""
      SELECT id, aggregate_type, aggregate_id, event_type, payload, metadata, sequence_number, occurred_at
      FROM domain_events
      WHERE aggregate_id = $aggregateId
      ORDER BY sequence_number ASC
    """.query[EventRecord].to[List].transact(xa)
  
  def loadFromVersion(aggregateId: UUID, fromVersion: Long): IO[List[EventRecord]] =
    sql"""
      SELECT id, aggregate_type, aggregate_id, event_type, payload, metadata, sequence_number, occurred_at
      FROM domain_events
      WHERE aggregate_id = $aggregateId AND sequence_number >= $fromVersion
      ORDER BY sequence_number ASC
    """.query[EventRecord].to[List].transact(xa)
  
  private def serializeEvent(event: DomainEvent): Json =
    event.asJson  // using circe auto derivation

// Aggregate reconstruction from events
trait EventSourcedAggregate[E]:
  def applyEvent(event: DomainEvent): E
  def version: Long

class AggregateRepository[A <: EventSourcedAggregate[A]](
  eventStore: EventStore[IO],
  aggregateType: String,
  empty: A
):
  def load(id: UUID): IO[Option[A]] =
    eventStore.load(id).map { records =>
      if records.isEmpty then None
      else Some(records.foldLeft(empty) { (agg, record) =>
        val event = deserializeEvent(record)
        agg.applyEvent(event)
      })
    }
  
  def save(id: UUID, aggregate: A, newEvents: List[DomainEvent]): IO[Unit] =
    eventStore.append(id, aggregateType, aggregate.version, newEvents)
  
  private def deserializeEvent(record: EventRecord): DomainEvent =
    // deserialize based on event_type
    record.eventType match
      case "OrderPlaced" => record.payload.as[OrderPlaced].getOrElse(???)
      case "OrderShipped" => record.payload.as[OrderShipped].getOrElse(???)
      case _ => throw new RuntimeException(s"Unknown event type: ${record.eventType}")
```

### Projections

```scala
// src/main/scala/eventsourcing/Projections.scala
package eventsourcing

import cats.effect.IO
import doobie.*
import doobie.implicits.*

// Projection - สร้าง read model จาก events
trait Projection:
  def name: String
  def handle(event: EventRecord): IO[Unit]
  def getLastProcessedPosition: IO[Long]
  def updateLastProcessedPosition(position: Long): IO[Unit]

// Order summary projection
case class OrderSummary(
  orderId: UUID,
  customerId: UUID,
  status: String,
  totalAmount: BigDecimal,
  itemCount: Int,
  createdAt: Instant,
  updatedAt: Instant
)

class OrderSummaryProjection(xa: Transactor[IO]) extends Projection:
  val name = "order_summary"
  
  def handle(event: EventRecord): IO[Unit] =
    event.eventType match
      case "OrderPlaced" =>
        val order = event.payload.as[OrderPlaced].getOrElse(
          throw new RuntimeException("Invalid OrderPlaced event")
        )
        sql"""
          INSERT INTO order_summaries
            (order_id, customer_id, status, total_amount, item_count, created_at, updated_at)
          VALUES
            (${order.orderId}, ${order.customerId}, 'PLACED',
             ${order.totalAmount}, ${order.items.length}, ${order.occurredAt}, ${order.occurredAt})
          ON CONFLICT (order_id) DO NOTHING
        """.update.run.transact(xa).void
      
      case "OrderShipped" =>
        val shipped = event.payload.as[OrderShipped].getOrElse(
          throw new RuntimeException("Invalid OrderShipped event")
        )
        sql"""
          UPDATE order_summaries
          SET status = 'SHIPPED', updated_at = ${shipped.occurredAt}
          WHERE order_id = ${shipped.orderId}
        """.update.run.transact(xa).void
      
      case _ => IO.unit
  
  def getLastProcessedPosition: IO[Long] =
    sql"SELECT last_position FROM projection_checkpoints WHERE name = $name"
      .query[Long].option.transact(xa).map(_.getOrElse(0L))
  
  def updateLastProcessedPosition(position: Long): IO[Unit] =
    sql"""
      INSERT INTO projection_checkpoints (name, last_position, updated_at)
      VALUES ($name, $position, NOW())
      ON CONFLICT (name) DO UPDATE SET last_position = $position, updated_at = NOW()
    """.update.run.transact(xa).void

// Projection runner
class ProjectionRunner(
  projections: List[Projection],
  eventStore: EventStore[IO]
):
  def run: IO[Nothing] =
    projections.traverse_(runProjection).flatMap(_ => IO.never)
  
  private def runProjection(projection: Projection): IO[Unit] =
    for
      lastPosition <- projection.getLastProcessedPosition
      events       <- loadEventsSince(lastPosition)
      _            <- events.traverse_ { event =>
                        projection.handle(event) >>
                        projection.updateLastProcessedPosition(event.sequenceNumber)
                      }
      _            <- IO.sleep(scala.concurrent.duration.1.second)
      _            <- runProjection(projection)  // loop
    yield ()
  
  private def loadEventsSince(position: Long): IO[List[EventRecord]] =
    IO.pure(List.empty)  // placeholder
```

---

## Eventual Consistency

```scala
// src/main/scala/consistency/EventualConsistency.scala
package consistency

import cats.effect.IO
import cats.effect.Ref
import scala.concurrent.duration.*

/**
 * Eventual Consistency - ระบบจะสอดคล้องกันในที่สุด
 *
 * Read-your-writes: ผู้ใช้เห็นการเปลี่ยนแปลงของตัวเองทันที
 * Monotonic reads: ข้อมูลจะไม่ย้อนกลับไปเวอร์ชันเก่า
 * Monotonic writes: การเขียนจะเกิดขึ้นตามลำดับ
 */

// Version vector สำหรับ tracking consistency
case class VersionVector(versions: Map[String, Long]):
  def increment(nodeId: String): VersionVector =
    copy(versions = versions.updated(nodeId, versions.getOrElse(nodeId, 0L) + 1))
  
  def merge(other: VersionVector): VersionVector =
    val merged = versions.keySet ++ other.versions.keySet
    val newVersions = merged.map { nodeId =>
      nodeId -> math.max(
        versions.getOrElse(nodeId, 0L),
        other.versions.getOrElse(nodeId, 0L)
      )
    }.toMap
    VersionVector(newVersions)
  
  def happensBefore(other: VersionVector): Boolean =
    versions.forall { (nodeId, v) =>
      other.versions.getOrElse(nodeId, 0L) >= v
    } && versions != other.versions

object VersionVector:
  val empty = VersionVector(Map.empty)

// Read-your-writes consistency token
case class ConsistencyToken(
  userId: UUID,
  vectorClock: VersionVector,
  issuedAt: Instant
)

// Session consistency tracker
class SessionConsistencyTracker:
  private val tokens = Ref.unsafe[IO, Map[UUID, ConsistencyToken]](Map.empty)
  
  def issueToken(userId: UUID, vectorClock: VersionVector): IO[ConsistencyToken] =
    val token = ConsistencyToken(userId, vectorClock, Instant.now())
    tokens.update(_ + (userId -> token)).as(token)
  
  def waitForConsistency(
    userId: UUID,
    readReplica: ReadReplica
  ): IO[Unit] =
    for
      maybeToken <- tokens.get.map(_.get(userId))
      _          <- maybeToken.fold(IO.unit) { token =>
                      readReplica.waitUntilCaughtUp(token.vectorClock)
                    }
    yield ()

trait ReadReplica:
  def waitUntilCaughtUp(vectorClock: VersionVector): IO[Unit]
  def getCurrentVersion: IO[VersionVector]

// Compensation logic สำหรับ eventual consistency
trait Compensatable[F[_]]:
  def compensate: F[Unit]

case class CompensationAction[F[_]](
  action: () => F[Unit],
  description: String
)

class CompensationLog[F[_]](actions: List[CompensationAction[F]]):
  def rollback: F[Unit] =
    actions.reverse.foldLeft(IO.unit) { (acc, action) =>
      acc.flatMap(_ => action.action().handleErrorWith { e =>
        IO.println(s"Compensation failed for ${action.description}: ${e.getMessage}")
      }).asInstanceOf[F[Unit]]
    }.asInstanceOf[F[Unit]]
```

---

## Two-Phase Commit vs Saga Pattern

### Saga Pattern - Choreography

```scala
// src/main/scala/saga/SagaPattern.scala
package saga

import cats.effect.IO
import events.*
import java.util.UUID

/**
 * Saga Pattern สำหรับจัดการ distributed transactions
 *
 * Choreography: แต่ละ service รับฟัง events และตัดสินใจเอง
 * Orchestration: Saga orchestrator บอกแต่ละ service ว่าทำอะไร
 */

// Choreography-based Saga
// Order Saga: Order -> Payment -> Inventory -> Shipping
object OrderCreationSaga:
  
  // Step 1: Order Service เผยแพร่ event
  def placeOrder(
    customerId: UUID,
    items: List[OrderItemEvent],
    eventBus: EventBus[IO]
  ): IO[UUID] =
    val orderId = UUID.randomUUID()
    val event = OrderPlaced(
      orderId = orderId,
      customerId = customerId,
      items = items,
      totalAmount = items.map(i => i.price * i.quantity).sum,
      shippingAddress = AddressEvent("123 Main St", "Bangkok", "Thailand", "10100"),
      paymentMethod = PaymentMethodEvent("CREDIT_CARD", Some("4242"))
    )
    eventBus.publish(event).as(orderId)
  
  // Step 2: Payment Service รับฟัง OrderPlaced
  class PaymentEventHandler(
    paymentService: PaymentService,
    eventBus: EventBus[IO]
  ) extends EventHandler[OrderPlaced]:
    def handle(event: OrderPlaced): IO[Unit] =
      paymentService.processPayment(
        event.orderId,
        event.totalAmount,
        event.paymentMethod
      ).flatMap {
        case Right(paymentId) =>
          eventBus.publish(PaymentProcessed(
            orderId = event.orderId,
            paymentId = paymentId,
            amount = event.totalAmount,
            currency = "THB",
            status = PaymentStatus.Completed
          ))
        case Left(error) =>
          // Compensate: Cancel the order
          eventBus.publish(OrderCancelled(
            orderId = event.orderId,
            reason = s"Payment failed: $error"
          ))
      }
  
  // Step 3: Inventory Service รับฟัง PaymentProcessed
  class InventoryEventHandler(
    inventoryService: InventoryService,
    eventBus: EventBus[IO]
  ) extends EventHandler[PaymentProcessed]:
    def handle(event: PaymentProcessed): IO[Unit] =
      // Reserve inventory
      inventoryService.reserve(event.orderId).flatMap {
        case Right(_) =>
          eventBus.publish(InventoryReservedIntegration(
            orderId = event.orderId.toString,
            items = List.empty  // simplified
          ))
        case Left(error) =>
          // Compensate: Refund payment
          eventBus.publish(PaymentRefundRequested(
            orderId = event.orderId,
            reason = s"Inventory not available: $error"
          ))
      }

// Orchestration-based Saga
sealed trait SagaState
case object SagaStarted extends SagaState
case object PaymentPending extends SagaState
case object PaymentCompleted extends SagaState
case object InventoryPending extends SagaState
case object InventoryReserved extends SagaState
case object ShippingPending extends SagaState
case class SagaCompleted(orderId: UUID) extends SagaState
case class SagaFailed(reason: String) extends SagaState

case class OrderSagaData(
  orderId: UUID,
  customerId: UUID,
  items: List[OrderItemEvent],
  totalAmount: BigDecimal,
  state: SagaState,
  compensationLog: List[String] = List.empty
)

class OrderSagaOrchestrator(
  paymentService: PaymentService,
  inventoryService: InventoryService,
  shippingService: ShippingService,
  eventBus: EventBus[IO]
):
  
  def execute(orderId: UUID, data: OrderSagaData): IO[Either[String, UUID]] =
    for
      paymentResult    <- executePaymentStep(data)
      inventoryResult  <- paymentResult match
                            case Right(_) => executeInventoryStep(data)
                            case Left(e)  => IO.pure(Left(e))
      shippingResult   <- inventoryResult match
                            case Right(_) => executeShippingStep(data)
                            case Left(e)  =>
                              compensatePayment(data) >> IO.pure(Left(e))
      finalResult      <- shippingResult match
                            case Right(_) => IO.pure(Right(orderId))
                            case Left(e)  =>
                              compensateInventory(data) >>
                              compensatePayment(data) >>
                              IO.pure(Left(e))
    yield finalResult
  
  private def executePaymentStep(data: OrderSagaData): IO[Either[String, UUID]] =
    paymentService.processPayment(data.orderId, data.totalAmount, PaymentMethodEvent("CREDIT_CARD", None))
  
  private def executeInventoryStep(data: OrderSagaData): IO[Either[String, Unit]] =
    inventoryService.reserve(data.orderId)
  
  private def executeShippingStep(data: OrderSagaData): IO[Either[String, String]] =
    shippingService.createShipment(data.orderId)
  
  private def compensatePayment(data: OrderSagaData): IO[Unit] =
    paymentService.refund(data.orderId).void
  
  private def compensateInventory(data: OrderSagaData): IO[Unit] =
    inventoryService.release(data.orderId).void

// Service interfaces
trait PaymentService:
  def processPayment(
    orderId: UUID,
    amount: BigDecimal,
    method: PaymentMethodEvent
  ): IO[Either[String, UUID]]
  def refund(orderId: UUID): IO[Either[String, Unit]]

trait InventoryService:
  def reserve(orderId: UUID): IO[Either[String, Unit]]
  def release(orderId: UUID): IO[Either[String, Unit]]

trait ShippingService:
  def createShipment(orderId: UUID): IO[Either[String, String]]

// Additional domain events for saga
case class OrderCancelled(
  eventId: UUID = UUID.randomUUID(),
  orderId: UUID,
  reason: String,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends DomainEvent

case class PaymentRefundRequested(
  eventId: UUID = UUID.randomUUID(),
  orderId: UUID,
  reason: String,
  occurredAt: Instant = Instant.now(),
  correlationId: Option[UUID] = None,
  causationId: Option[UUID] = None
) extends DomainEvent
```

---

## Event Schema Design

### Schema Evolution

```scala
// src/main/scala/schema/EventSchema.scala
package schema

import io.circe.*
import io.circe.generic.auto.*
import io.circe.syntax.*

/**
 * หลักการออกแบบ Event Schema:
 * 1. เพิ่ม field ได้ (backward compatible)
 * 2. ลบ field ต้องระวัง (breaking change)
 * 3. เปลี่ยน type ของ field ไม่ได้
 * 4. ใช้ Optional fields สำหรับ new fields
 */

// V1 - เวอร์ชันแรก
case class OrderPlacedV1(
  orderId: String,
  customerId: String,
  totalAmount: Double,
  timestamp: Long  // epoch millis
)

// V2 - เพิ่ม fields ใหม่ (backward compatible)
case class OrderPlacedV2(
  orderId: String,
  customerId: String,
  totalAmount: Double,
  currency: String = "THB",  // new field with default
  timestamp: Long,
  metadata: Option[Map[String, String]] = None  // optional new field
)

// V3 - เปลี่ยน type (ต้องมี migration)
case class OrderPlacedV3(
  orderId: String,
  customerId: String,
  totalAmount: BigDecimal,  // เปลี่ยนจาก Double เป็น BigDecimal
  currency: String = "THB",
  occurredAt: String,  // เปลี่ยนจาก Long เป็น ISO 8601 string
  metadata: Option[Map[String, String]] = None
)

// Schema version handling
object SchemaVersionHandler:
  
  def decode(json: Json, version: Int): Either[String, OrderPlacedV3] =
    version match
      case 1 =>
        json.as[OrderPlacedV1].map { v1 =>
          OrderPlacedV3(
            orderId = v1.orderId,
            customerId = v1.customerId,
            totalAmount = BigDecimal(v1.totalAmount),
            occurredAt = java.time.Instant.ofEpochMilli(v1.timestamp).toString
          )
        }.left.map(_.message)
      case 2 =>
        json.as[OrderPlacedV2].map { v2 =>
          OrderPlacedV3(
            orderId = v2.orderId,
            customerId = v2.customerId,
            totalAmount = BigDecimal(v2.totalAmount),
            currency = v2.currency,
            occurredAt = java.time.Instant.ofEpochMilli(v2.timestamp).toString,
            metadata = v2.metadata
          )
        }.left.map(_.message)
      case 3 =>
        json.as[OrderPlacedV3].left.map(_.message)
      case v =>
        Left(s"Unknown schema version: $v")

// CloudEvents spec compliance
case class CloudEvent[A](
  specversion: String = "1.0",
  id: String,
  source: String,
  `type`: String,
  datacontenttype: String = "application/json",
  dataschema: Option[String] = None,
  subject: Option[String] = None,
  time: String,
  data: A
)

object CloudEventEnvelope:
  def wrap[A: Encoder](
    event: A,
    source: String,
    eventType: String,
    subject: Option[String] = None
  )(using io.circe.Encoder[A]): CloudEvent[A] =
    CloudEvent(
      id = java.util.UUID.randomUUID().toString,
      source = source,
      `type` = s"io.example.$eventType",
      subject = subject,
      time = java.time.Instant.now().toString,
      data = event
    )
```

---

## Complete Event-Driven System

### System Overview

```scala
// src/main/scala/system/EventDrivenSystem.scala
package system

import cats.effect.*
import cats.syntax.all.*

class EventDrivenSystem(
  eventBus: EventBus[IO],
  eventStore: EventStore[IO],
  kafkaPublisher: KafkaEventPublisher,
  projections: List[Projection]
):
  
  // Outbox pattern - ป้องกัน dual-write problem
  def publishWithOutbox[E <: DomainEvent](
    event: E,
    outboxRepo: OutboxRepository[IO]
  )(using io.circe.Encoder[E]): IO[Unit] =
    outboxRepo.save(OutboxEvent(
      id = java.util.UUID.randomUUID(),
      aggregateType = event.getClass.getSimpleName,
      eventType = event.getClass.getSimpleName,
      payload = event.asJson.noSpaces,
      createdAt = java.time.Instant.now()
    ))
  
  // Outbox processor
  def processOutbox(outboxRepo: OutboxRepository[IO]): IO[Nothing] =
    fs2.Stream.repeatEval(
      for
        pending <- outboxRepo.findPending(limit = 100)
        _ <- pending.traverse_ { outboxEvent =>
          publishToKafka(outboxEvent) >>
          outboxRepo.markAsProcessed(outboxEvent.id)
        }
        _ <- IO.sleep(scala.concurrent.duration.1.second)
      yield ()
    ).compile.drain.flatMap(_ => IO.never)
  
  private def publishToKafka(event: OutboxEvent): IO[Unit] =
    IO.unit  // placeholder

// Outbox pattern
case class OutboxEvent(
  id: java.util.UUID,
  aggregateType: String,
  eventType: String,
  payload: String,
  createdAt: java.time.Instant,
  processedAt: Option[java.time.Instant] = None
)

trait OutboxRepository[F[_]]:
  def save(event: OutboxEvent): F[Unit]
  def findPending(limit: Int): F[List[OutboxEvent]]
  def markAsProcessed(id: java.util.UUID): F[Unit]

// Dead letter queue
case class DeadLetterEvent(
  originalEvent: String,
  error: String,
  retryCount: Int,
  firstFailedAt: java.time.Instant,
  lastFailedAt: java.time.Instant
)

class DeadLetterQueue(xa: doobie.Transactor[IO]):
  
  def save(event: DeadLetterEvent): IO[Unit] =
    import doobie.implicits.*
    sql"""
      INSERT INTO dead_letter_queue
        (original_event, error, retry_count, first_failed_at, last_failed_at)
      VALUES
        (${event.originalEvent}, ${event.error}, ${event.retryCount},
         ${event.firstFailedAt}, ${event.lastFailedAt})
    """.update.run.transact(xa).void
  
  def reprocess(id: java.util.UUID, eventBus: EventBus[IO]): IO[Unit] =
    IO.unit // placeholder

// Application startup
object Application extends IOApp:
  
  def run(args: List[String]): IO[ExitCode] =
    resources.use { deps =>
      val system = EventDrivenSystem(
        deps.eventBus,
        deps.eventStore,
        deps.kafkaPublisher,
        deps.projections
      )
      
      // Run everything in parallel
      (
        system.processOutbox(deps.outboxRepo),
        deps.projectionRunner.run,
        deps.kafkaConsumer.start
      ).parTupled.void.as(ExitCode.Success)
    }
  
  private def resources: Resource[IO, AppDependencies] =
    for
      xa           <- database.ConnectionPool.make(loadDatabaseConfig())
      eventStore   <- Resource.pure(PostgreSQLEventStore(xa))
      eventBus     <- Resource.eval(AsyncEventBus.make())
      _            <- Resource.eval(IO.println("Event-driven system started"))
    yield AppDependencies(xa, eventStore, eventBus)

case class AppDependencies(
  xa: doobie.Transactor[IO],
  eventStore: EventStore[IO],
  eventBus: EventBus[IO]
)

def loadDatabaseConfig(): database.DatabaseConfig =
  database.DatabaseConfig(
    url = sys.env.getOrElse("DATABASE_URL", "jdbc:postgresql://localhost:5432/ecommerce"),
    user = sys.env.getOrElse("DATABASE_USER", "postgres"),
    password = sys.env.getOrElse("DATABASE_PASSWORD", "postgres")
  )
```

---

## สรุป

Event-Driven Architecture ใน Scala ประกอบด้วย:

1. **Domain Events vs Integration Events**: แยกประเภท events ตาม context และ audience
2. **In-Process Event Bus**: สำหรับ domain events ภายใน bounded context
3. **Distributed Event Bus (Kafka)**: สำหรับ integration events ระหว่าง services
4. **Event Sourcing**: เก็บ state เป็น sequence ของ events
5. **Projections**: สร้าง read models จาก events
6. **Eventual Consistency**: ยอมรับว่าข้อมูลจะสอดคล้องกันในที่สุด
7. **Saga Pattern**: จัดการ distributed transactions ด้วย choreography หรือ orchestration
8. **Outbox Pattern**: ป้องกัน dual-write problem
9. **Dead Letter Queue**: จัดการ events ที่ประมวลผลไม่สำเร็จ
10. **Schema Evolution**: ออกแบบ event schema ที่ evolve ได้อย่างปลอดภัย

---

*[← ส่วนที่ 77: Service Mesh and Observability](part-77-service-mesh.md) | [ส่วนที่ 79: Scala.js →](part-79-scala-js.md)*
