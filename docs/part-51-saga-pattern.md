# Part 51: Saga Pattern สำหรับ Distributed Transactions

## สารบัญ

1. [ปัญหาของ Distributed Transactions](#ปัญหาของ-distributed-transactions)
2. [Choreography vs Orchestration](#choreography-vs-orchestration)
3. [Compensating Transactions](#compensating-transactions)
4. [Saga State Machine](#saga-state-machine)
5. [Retry และ Timeout Handling](#retry-และ-timeout-handling)
6. [ตัวอย่าง Order Saga สมบูรณ์](#ตัวอย่าง-order-saga-สมบูรณ์)
7. [Testing Sagas](#testing-sagas)

---

## ปัญหาของ Distributed Transactions

ใน microservices เราไม่สามารถใช้ ACID transactions ข้าม services ได้ เนื่องจากแต่ละ service มี database ของตัวเอง Saga Pattern แก้ปัญหานี้ด้วยการแบ่ง transaction ใหญ่เป็น local transactions ย่อยๆ

```
แบบเดิม (ไม่ได้ผลใน Microservices):
BEGIN TRANSACTION
  UPDATE orders SET status = 'confirmed' WHERE id = ?
  UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?
  INSERT INTO payments (order_id, amount) VALUES (?, ?)
COMMIT  -- ❌ ไม่สามารถ commit ข้าม services ได้!

แบบ Saga:
1. Order Service: createOrder()              → ถ้าสำเร็จ → 2
2. Inventory Service: reserveItems()         → ถ้าสำเร็จ → 3
3. Payment Service: processPayment()         → ถ้าสำเร็จ → Done!
   ถ้าล้มเหลวที่ 3: 
   ← Inventory Service: cancelReservation() (compensate 2)
   ← Order Service: cancelOrder()            (compensate 1)
```

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.typelevel"  %% "cats-effect"      % "3.5.4",
  "co.fs2"         %% "fs2-core"         % "3.10.0",
  "com.github.fd4s" %% "fs2-kafka"       % "3.5.1",
  "org.tpolecat"   %% "doobie-core"      % "1.0.0-RC4",
  "org.tpolecat"   %% "doobie-postgres"  % "1.0.0-RC4",
  "io.circe"       %% "circe-core"       % "0.14.9",
  "io.circe"       %% "circe-generic"    % "0.14.9"
)
```

---

## Choreography vs Orchestration

### Choreography Saga

Services สื่อสารกันผ่าน events โดยตรง ไม่มี central coordinator:

```
OrderService ──publishes──▶ OrderCreated
                                │
                    InventoryService listens
                                │
                    InventoryService ──publishes──▶ ItemsReserved
                                                          │
                                              PaymentService listens
                                                          │
                                              PaymentService ──publishes──▶ PaymentProcessed
                                                                                   │
                                                               OrderService updates to CONFIRMED
```

```scala
import cats.effect.*
import fs2.kafka.*
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*

// Domain Events
sealed trait SagaEvent:
  def orderId: String
  def timestamp: Long

case class OrderCreated(
  orderId: String,
  customerId: String,
  items: List[OrderItem],
  totalAmount: BigDecimal,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

case class ItemsReserved(
  orderId: String,
  reservationId: String,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

case class ItemsReservationFailed(
  orderId: String,
  reason: String,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

case class PaymentProcessed(
  orderId: String,
  paymentId: String,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

case class PaymentFailed(
  orderId: String,
  reason: String,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

// Compensation Events
case class OrderCancelled(
  orderId: String,
  reason: String,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

case class ReservationCancelled(
  orderId: String,
  reservationId: String,
  timestamp: Long = System.currentTimeMillis()
) extends SagaEvent

case class OrderItem(productId: String, quantity: Int, price: BigDecimal)
```

### Choreography Inventory Service

```scala
import cats.effect.*
import fs2.*
import fs2.kafka.*

class ChoreographyInventoryService(
  producer: KafkaProducer.Fiber[IO, String, String],
  reservationRepo: ReservationRepository
):

  // Listen for OrderCreated, attempt reservation
  def handleOrderCreated(event: OrderCreated): IO[Unit] =
    reservationRepo
      .reserve(event.orderId, event.items)
      .flatMap:
        case Right(reservationId) =>
          publishEvent(ItemsReserved(event.orderId, reservationId))
        case Left(reason) =>
          publishEvent(ItemsReservationFailed(event.orderId, reason))

  // Listen for PaymentFailed, cancel reservation
  def handlePaymentFailed(event: PaymentFailed): IO[Unit] =
    reservationRepo
      .findByOrderId(event.orderId)
      .flatMap:
        case Some(reservation) =>
          reservationRepo.cancel(reservation.id) >>
          publishEvent(ReservationCancelled(event.orderId, reservation.id))
        case None =>
          IO.println(s"No reservation found for order ${event.orderId}")

  private def publishEvent(event: SagaEvent): IO[Unit] =
    val topic = event match
      case _: ItemsReserved          => "inventory.items-reserved"
      case _: ItemsReservationFailed => "inventory.reservation-failed"
      case _: ReservationCancelled   => "inventory.reservation-cancelled"
      case _                         => "inventory.unknown"

    val record = ProducerRecord(topic, event.orderId, event.asJson.noSpaces)
    producer.produce(ProducerRecords.one(record)).flatten.void
```

### Orchestration Saga

Central orchestrator ควบคุม flow ทั้งหมด:

```scala
import cats.effect.*
import cats.effect.Ref

// Saga states
enum OrderSagaState:
  case Pending
  case OrderCreated(orderId: String)
  case ItemsReserved(orderId: String, reservationId: String)
  case PaymentProcessed(orderId: String, reservationId: String, paymentId: String)
  case Completed(orderId: String)
  case CompensatingReservation(orderId: String, reservationId: String)
  case CompensatingOrder(orderId: String)
  case Failed(reason: String)

// Saga commands
enum SagaCommand:
  case CreateOrder(customerId: String, items: List[OrderItem])
  case ReserveItems(orderId: String, items: List[OrderItem])
  case ProcessPayment(orderId: String, amount: BigDecimal)
  case CancelReservation(orderId: String, reservationId: String)
  case CancelOrder(orderId: String)
```

---

## Compensating Transactions

### Compensation Pattern

```scala
import cats.effect.*
import cats.syntax.all.*

// Each step has a forward action and a compensating action
case class SagaStep[A](
  name: String,
  execute: IO[A],
  compensate: A => IO[Unit]
)

// Saga result after executing all steps
case class SagaResult[A](value: A, compensations: List[IO[Unit]])

// Execute saga with automatic compensation on failure
def executeSaga[A](steps: List[SagaStep[?]]): IO[Unit] =

  def go(
    remaining: List[SagaStep[?]],
    compensations: List[IO[Unit]]
  ): IO[Unit] =
    remaining match
      case Nil => IO.unit // All steps succeeded
      case step :: rest =>
        step.execute
          .flatMap: result =>
            val compensation = step.compensate.asInstanceOf[Any => IO[Unit]](result)
            go(rest, compensation :: compensations)
          .handleErrorWith: e =>
            // Execute compensations in reverse order
            IO.println(s"Step '${step.name}' failed: ${e.getMessage}") >>
            compensations.traverse_(identity).flatMap: _ =>
              IO.raiseError(new RuntimeException(
                s"Saga failed at step '${step.name}': ${e.getMessage}",
                e
              ))

  go(steps, Nil)

// Type-safe saga builder
class SagaBuilder[Input, Output]:

  def pure[A](value: A): Saga[A] = Saga.Pure(value)

  def step[A](
    name: String,
    action: IO[A],
    compensate: A => IO[Unit]
  ): Saga[A] = Saga.Step(name, action, compensate)

sealed trait Saga[+A]:
  def flatMap[B](f: A => Saga[B]): Saga[B] = Saga.FlatMap(this, f)
  def map[B](f: A => B): Saga[B] = flatMap(a => Saga.Pure(f(a)))

object Saga:
  case class Pure[A](value: A) extends Saga[A]
  case class Step[A](name: String, action: IO[A], compensate: A => IO[Unit]) extends Saga[A]
  case class FlatMap[A, B](saga: Saga[A], f: A => Saga[B]) extends Saga[B]

  // Interpret saga to IO with compensation
  def run[A](saga: Saga[A]): IO[A] =
    runWith(saga, Nil).map(_._1)

  private def runWith[A](
    saga: Saga[A],
    compensations: List[IO[Unit]]
  ): IO[(A, List[IO[Unit]])] =
    saga match
      case Pure(value) =>
        IO.pure((value, compensations))

      case Step(name, action, compensate) =>
        action
          .flatMap: a =>
            IO.pure((a, compensate(a) :: compensations))
          .handleErrorWith: e =>
            compensations.traverse_(identity) >>
            IO.raiseError(new RuntimeException(s"Saga step '$name' failed", e))

      case FlatMap(inner, f) =>
        runWith(inner, compensations).flatMap: (a, comps) =>
          runWith(f(a), comps)
```

---

## Saga State Machine

### State Machine Implementation

```scala
import cats.effect.*
import cats.effect.Ref

class OrderSagaMachine(
  orderService: OrderService,
  inventoryService: InventoryService,
  paymentService: PaymentService,
  sagaRepo: SagaRepository
):

  def startOrderSaga(
    sagaId: String,
    customerId: String,
    items: List[OrderItem]
  ): IO[SagaResult] =

    val saga = for
      // Step 1: Create Order
      orderId <- Saga.step(
        name       = "create-order",
        action     = orderService.create(customerId, items),
        compensate = orderId => orderService.cancel(orderId, "Saga compensation")
      )

      // Step 2: Reserve Inventory
      reservationId <- Saga.step(
        name       = "reserve-inventory",
        action     = inventoryService.reserve(orderId, items),
        compensate = resId => inventoryService.cancel(resId)
      )

      // Step 3: Process Payment
      paymentId <- Saga.step(
        name       = "process-payment",
        action     = paymentService.charge(orderId, calculateTotal(items)),
        compensate = payId => paymentService.refund(payId)
      )

      // Step 4: Confirm Order (no compensation needed - final step)
      _ <- Saga.step(
        name       = "confirm-order",
        action     = orderService.confirm(orderId),
        compensate = _ => IO.unit
      )

    yield OrderConfirmed(orderId, reservationId, paymentId)

    sagaRepo.save(sagaId, OrderSagaState.Pending) >>
    Saga
      .run(saga)
      .flatTap(result =>
        sagaRepo.update(sagaId, OrderSagaState.Completed(result.orderId))
      )
      .handleErrorWith: e =>
        sagaRepo.update(sagaId, OrderSagaState.Failed(e.getMessage)) >>
        IO.raiseError(e)

  private def calculateTotal(items: List[OrderItem]): BigDecimal =
    items.map(i => i.price * i.quantity).sum

// Saga result types
case class OrderConfirmed(
  orderId: String,
  reservationId: String,
  paymentId: String
)

sealed trait SagaResult
case class SagaSuccess(orderId: String) extends SagaResult
case class SagaFailure(reason: String, compensated: Boolean) extends SagaResult
```

### Persistent Saga State

```scala
import doobie.*
import doobie.implicits.*
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*

case class SagaRecord(
  sagaId: String,
  sagaType: String,
  state: String,
  payload: String,
  createdAt: java.time.Instant,
  updatedAt: java.time.Instant,
  version: Int
)

class SagaRepository(xa: Transactor[IO]):

  def save(sagaId: String, state: OrderSagaState): IO[Unit] =
    sql"""
      INSERT INTO sagas (saga_id, saga_type, state, payload, created_at, updated_at, version)
      VALUES (
        $sagaId,
        'order-saga',
        ${state.toString},
        ${state.asJson.noSpaces},
        NOW(),
        NOW(),
        1
      )
    """.update.run.transact(xa).void

  def update(sagaId: String, newState: OrderSagaState): IO[Unit] =
    sql"""
      UPDATE sagas
      SET state = ${newState.toString},
          payload = ${newState.asJson.noSpaces},
          updated_at = NOW(),
          version = version + 1
      WHERE saga_id = $sagaId
    """.update.run.transact(xa).void

  def findById(sagaId: String): IO[Option[SagaRecord]] =
    sql"""
      SELECT saga_id, saga_type, state, payload, created_at, updated_at, version
      FROM sagas
      WHERE saga_id = $sagaId
    """.query[SagaRecord].option.transact(xa)

  def findStuckSagas(olderThanMinutes: Int = 30): IO[List[SagaRecord]] =
    sql"""
      SELECT saga_id, saga_type, state, payload, created_at, updated_at, version
      FROM sagas
      WHERE state NOT IN ('Completed', 'Failed')
        AND updated_at < NOW() - INTERVAL '${olderThanMinutes} minutes'
      ORDER BY created_at ASC
      LIMIT 100
    """.query[SagaRecord].to[List].transact(xa)

// Database schema
val createSagaTable: Fragment = fr"""
  CREATE TABLE IF NOT EXISTS sagas (
    saga_id    VARCHAR(255) PRIMARY KEY,
    saga_type  VARCHAR(100) NOT NULL,
    state      VARCHAR(100) NOT NULL,
    payload    JSONB        NOT NULL,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    version    INT          NOT NULL DEFAULT 1
  );

  CREATE INDEX IF NOT EXISTS idx_sagas_state ON sagas(state);
  CREATE INDEX IF NOT EXISTS idx_sagas_updated_at ON sagas(updated_at);
"""
```

---

## Retry และ Timeout Handling

### Retry Policy

```scala
import cats.effect.*
import scala.concurrent.duration.*

sealed trait RetryPolicy

object RetryPolicy:
  case class FixedDelay(delay: FiniteDuration, maxRetries: Int) extends RetryPolicy
  case class ExponentialBackoff(
    initialDelay: FiniteDuration,
    maxDelay: FiniteDuration,
    maxRetries: Int,
    multiplier: Double = 2.0
  ) extends RetryPolicy
  case class NoRetry() extends RetryPolicy

def retry[A](
  action: IO[A],
  policy: RetryPolicy,
  isRetryable: Throwable => Boolean = _ => true
): IO[A] =

  def attempt(retriesLeft: Int, delay: FiniteDuration): IO[A] =
    action.handleErrorWith: e =>
      if !isRetryable(e) then
        IO.raiseError(e)
      else if retriesLeft <= 0 then
        IO.raiseError(new RuntimeException(s"Max retries exhausted: ${e.getMessage}", e))
      else
        IO.println(s"Retrying after ${delay}ms, $retriesLeft retries left") >>
        IO.sleep(delay) >>
        attempt(retriesLeft - 1, nextDelay(delay, policy))

  policy match
    case RetryPolicy.NoRetry() => action
    case RetryPolicy.FixedDelay(delay, maxRetries) =>
      attempt(maxRetries, delay)
    case RetryPolicy.ExponentialBackoff(initial, maxDelay, maxRetries, multiplier) =>
      attempt(maxRetries, initial)

def nextDelay(current: FiniteDuration, policy: RetryPolicy): FiniteDuration =
  policy match
    case RetryPolicy.ExponentialBackoff(_, maxDelay, _, multiplier) =>
      val next = FiniteDuration((current.toMillis * multiplier).toLong, java.util.concurrent.TimeUnit.MILLISECONDS)
      if next > maxDelay then maxDelay else next
    case _ => current

// Timeout wrapper
def withTimeout[A](
  action: IO[A],
  timeout: FiniteDuration
): IO[A] =
  action.timeout(timeout).handleErrorWith:
    case e: java.util.concurrent.TimeoutException =>
      IO.raiseError(new RuntimeException(s"Action timed out after $timeout", e))
    case e => IO.raiseError(e)
```

### Saga with Retry

```scala
import cats.effect.*
import scala.concurrent.duration.*

class ResilientSagaStep[A](
  name: String,
  action: IO[A],
  compensate: A => IO[Unit],
  retryPolicy: RetryPolicy = RetryPolicy.ExponentialBackoff(100.millis, 30.seconds, 3),
  timeout: FiniteDuration = 30.seconds
):
  def executeWithRetry: IO[A] =
    withTimeout(
      retry(action, retryPolicy, isTransientError),
      timeout
    )

  private def isTransientError(e: Throwable): Boolean =
    e match
      case _: java.net.ConnectException     => true
      case _: java.io.IOException           => true
      case _: java.sql.SQLTransientException => true
      case e if e.getMessage.contains("timeout") => true
      case _ => false

// Idempotency key support
class IdempotentSagaStep[A](
  name: String,
  idempotencyKey: String,
  action: IO[A],
  compensate: A => IO[Unit],
  idempotencyStore: IdempotencyStore
):
  def executeIdempotent: IO[A] =
    idempotencyStore.get[A](idempotencyKey).flatMap:
      case Some(cached) =>
        IO.println(s"Using cached result for $idempotencyKey") >>
        IO.pure(cached)
      case None =>
        action.flatTap: result =>
          idempotencyStore.set(idempotencyKey, result, ttl = 24.hours)

// Idempotency store (Redis-backed)
trait IdempotencyStore:
  def get[A: io.circe.Decoder](key: String): IO[Option[A]]
  def set[A: io.circe.Encoder](key: String, value: A, ttl: FiniteDuration): IO[Unit]
```

---

## ตัวอย่าง Order Saga สมบูรณ์

### Domain Models

```scala
import java.util.UUID
import java.time.Instant

// Order domain
case class OrderId(value: String) extends AnyVal
case class CustomerId(value: String) extends AnyVal
case class ProductId(value: String) extends AnyVal

case class OrderItem(
  productId: ProductId,
  productName: String,
  quantity: Int,
  unitPrice: BigDecimal
):
  def total: BigDecimal = unitPrice * quantity

case class ShippingAddress(
  street: String,
  city: String,
  postalCode: String,
  country: String
)

case class CreateOrderRequest(
  customerId: CustomerId,
  items: List[OrderItem],
  shippingAddress: ShippingAddress,
  idempotencyKey: String
)

// Service interfaces
trait OrderService:
  def create(req: CreateOrderRequest): IO[OrderId]
  def confirm(orderId: OrderId): IO[Unit]
  def cancel(orderId: OrderId, reason: String): IO[Unit]
  def getStatus(orderId: OrderId): IO[OrderStatus]

trait InventoryService:
  def reserve(orderId: OrderId, items: List[OrderItem]): IO[ReservationId]
  def confirm(reservationId: ReservationId): IO[Unit]
  def cancel(reservationId: ReservationId): IO[Unit]

trait PaymentService:
  def charge(orderId: OrderId, amount: BigDecimal, customerId: CustomerId): IO[PaymentId]
  def refund(paymentId: PaymentId): IO[Unit]

trait NotificationService:
  def notifyOrderConfirmed(orderId: OrderId, customerId: CustomerId): IO[Unit]
  def notifyOrderFailed(orderId: OrderId, customerId: CustomerId, reason: String): IO[Unit]

case class ReservationId(value: String) extends AnyVal
case class PaymentId(value: String) extends AnyVal

enum OrderStatus:
  case Pending, Confirmed, Cancelled, Failed
```

### Complete Order Saga Orchestrator

```scala
import cats.effect.*
import cats.effect.std.Queue
import scala.concurrent.duration.*
import io.circe.generic.semiauto.*

class OrderSagaOrchestrator(
  orderService: OrderService,
  inventoryService: InventoryService,
  paymentService: PaymentService,
  notificationService: NotificationService,
  sagaRepo: SagaRepository
):

  def execute(req: CreateOrderRequest): IO[SagaResult] =
    val sagaId = UUID.randomUUID().toString

    val saga: Saga[OrderConfirmed] = for

      // Step 1: Create pending order
      orderId <- Saga.step(
        name = "create-order",
        action = orderService
          .create(req)
          .flatTap(id => IO.println(s"[$sagaId] Order created: $id")),
        compensate = (orderId: OrderId) =>
          orderService
            .cancel(orderId, s"Saga $sagaId compensation")
            .flatTap(_ => IO.println(s"[$sagaId] Order cancelled: $orderId"))
      )

      // Step 2: Reserve inventory
      reservationId <- Saga.step(
        name = "reserve-inventory",
        action = retry(
          inventoryService.reserve(orderId, req.items),
          RetryPolicy.ExponentialBackoff(100.millis, 5.seconds, 3)
        ).flatTap(r => IO.println(s"[$sagaId] Inventory reserved: $r")),
        compensate = (reservationId: ReservationId) =>
          inventoryService
            .cancel(reservationId)
            .flatTap(_ => IO.println(s"[$sagaId] Reservation cancelled: $reservationId"))
      )

      // Step 3: Process payment
      paymentId <- Saga.step(
        name = "process-payment",
        action = withTimeout(
          retry(
            paymentService.charge(orderId, calculateTotal(req.items), req.customerId),
            RetryPolicy.FixedDelay(500.millis, 2)
          ),
          timeout = 30.seconds
        ).flatTap(p => IO.println(s"[$sagaId] Payment processed: $p")),
        compensate = (paymentId: PaymentId) =>
          paymentService
            .refund(paymentId)
            .flatTap(_ => IO.println(s"[$sagaId] Payment refunded: $paymentId"))
      )

      // Step 4: Confirm order (idempotent final step)
      _ <- Saga.step(
        name = "confirm-order",
        action = orderService.confirm(orderId) >>
                 inventoryService.confirm(reservationId) >>
                 IO.println(s"[$sagaId] Order confirmed"),
        compensate = _ => IO.unit // Final step: no compensation
      )

      // Step 5: Send notification (best-effort, no compensation)
      _ <- Saga.step(
        name = "send-notification",
        action = notificationService
          .notifyOrderConfirmed(orderId, req.customerId)
          .handleErrorWith(e =>
            IO.println(s"[$sagaId] Notification failed (non-critical): $e")
          ),
        compensate = _ => IO.unit
      )

    yield OrderConfirmed(orderId, reservationId, paymentId)

    // Execute with state persistence
    for
      _      <- sagaRepo.save(sagaId, OrderSagaState.Pending)
      result <- Saga.run(saga)
                  .flatTap: confirmed =>
                    sagaRepo.update(sagaId, OrderSagaState.Completed(confirmed.orderId.value))
                  .handleErrorWith: e =>
                    sagaRepo.update(sagaId, OrderSagaState.Failed(e.getMessage)) >>
                    notificationService
                      .notifyOrderFailed(
                        OrderId("unknown"),
                        req.customerId,
                        e.getMessage
                      )
                      .attempt
                      .void >>
                    IO.raiseError(e)
    yield SagaSuccess(result.orderId.value)

  private def calculateTotal(items: List[OrderItem]): BigDecimal =
    items.map(_.total).sum
```

### Saga Recovery (Resume Stuck Sagas)

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

class SagaRecoveryWorker(
  sagaRepo: SagaRepository,
  orchestrator: OrderSagaOrchestrator
):

  // Periodically find and recover stuck sagas
  def run: Stream[IO, Unit] =
    Stream
      .awakeEvery[IO](1.minute)
      .evalMap(_ => recoverStuckSagas)

  private def recoverStuckSagas: IO[Unit] =
    for
      stuck <- sagaRepo.findStuckSagas(olderThanMinutes = 10)
      _     <- IO.println(s"Found ${stuck.length} stuck sagas")
      _     <- stuck.traverse_(recoverSaga)
    yield ()

  private def recoverSaga(record: SagaRecord): IO[Unit] =
    record.state match
      case "OrderCreated" =>
        // Order was created but payment not processed
        // Can resume from inventory reservation step
        IO.println(s"Recovering saga ${record.sagaId} from OrderCreated state")
        // In real implementation, parse state and resume

      case "ItemsReserved" =>
        // Items reserved but payment failed
        IO.println(s"Recovering saga ${record.sagaId} from ItemsReserved state")

      case _ =>
        IO.println(s"Cannot recover saga ${record.sagaId} in state ${record.state}")
```

---

## Testing Sagas

### Unit Tests

```scala
import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.freespec.AsyncFreeSpec
import org.mockito.MockitoSugar
import org.mockito.ArgumentMatchers.*

class OrderSagaSpec extends AsyncFreeSpec
    with AsyncIOSpec
    with Matchers
    with MockitoSugar:

  val mockOrderSvc    = mock[OrderService]
  val mockInventory   = mock[InventoryService]
  val mockPayment     = mock[PaymentService]
  val mockNotification = mock[NotificationService]
  val mockSagaRepo    = mock[SagaRepository]

  val orchestrator = new OrderSagaOrchestrator(
    mockOrderSvc,
    mockInventory,
    mockPayment,
    mockNotification,
    mockSagaRepo
  )

  val testRequest = CreateOrderRequest(
    customerId      = CustomerId("customer-1"),
    items           = List(OrderItem(ProductId("prod-1"), "Widget", 2, BigDecimal("50.00"))),
    shippingAddress = ShippingAddress("123 Main St", "Bangkok", "10100", "TH"),
    idempotencyKey  = "idem-key-1"
  )

  "OrderSagaOrchestrator" - {

    "should complete successfully when all steps succeed" in {
      val orderId       = OrderId("order-123")
      val reservationId = ReservationId("res-456")
      val paymentId     = PaymentId("pay-789")

      when(mockSagaRepo.save(any, any)).thenReturn(IO.unit)
      when(mockSagaRepo.update(any, any)).thenReturn(IO.unit)
      when(mockOrderSvc.create(any)).thenReturn(IO.pure(orderId))
      when(mockOrderSvc.confirm(any)).thenReturn(IO.unit)
      when(mockInventory.reserve(any, any)).thenReturn(IO.pure(reservationId))
      when(mockInventory.confirm(any)).thenReturn(IO.unit)
      when(mockPayment.charge(any, any, any)).thenReturn(IO.pure(paymentId))
      when(mockNotification.notifyOrderConfirmed(any, any)).thenReturn(IO.unit)

      orchestrator
        .execute(testRequest)
        .asserting:
          case SagaSuccess(id) => id shouldBe "order-123"
          case SagaFailure(r, _) => fail(s"Expected success but got failure: $r")
    }

    "should compensate when payment fails" in {
      val orderId       = OrderId("order-123")
      val reservationId = ReservationId("res-456")

      when(mockSagaRepo.save(any, any)).thenReturn(IO.unit)
      when(mockSagaRepo.update(any, any)).thenReturn(IO.unit)
      when(mockOrderSvc.create(any)).thenReturn(IO.pure(orderId))
      when(mockOrderSvc.cancel(any, any)).thenReturn(IO.unit)
      when(mockInventory.reserve(any, any)).thenReturn(IO.pure(reservationId))
      when(mockInventory.cancel(any)).thenReturn(IO.unit)
      when(mockPayment.charge(any, any, any)).thenReturn(
        IO.raiseError(new RuntimeException("Payment gateway error"))
      )
      when(mockNotification.notifyOrderFailed(any, any, any)).thenReturn(IO.unit)

      orchestrator
        .execute(testRequest)
        .attempt
        .asserting: result =>
          result.isLeft shouldBe true
          // Verify compensations were called
          verify(mockInventory).cancel(reservationId)
          verify(mockOrderSvc).cancel(orderId, s"Saga compensation")
    }
  }
```

---

## สรุป

Saga Pattern เป็นวิธีหลักในการจัดการ distributed transactions ใน microservices:

| แบบ | ข้อดี | ข้อเสีย |
|-----|-------|---------|
| **Choreography** | ไม่มี single point of failure, loosely coupled | ยากต่อการ debug, flow กระจาย |
| **Orchestration** | เห็น flow ชัดเจน, ง่ายต่อ monitoring | Central coordinator อาจเป็น bottleneck |

**หลักการสำคัญ:**

1. **Idempotency** - ทุก step ต้องสามารถ retry ได้โดยผลลัพธ์เหมือนเดิม
2. **Compensating Transactions** - ต้องมี rollback สำหรับทุก step
3. **State Persistence** - บันทึก saga state ไว้เพื่อ recovery
4. **Timeout Handling** - ทุก step ต้องมี timeout
5. **Monitoring** - Track saga state เพื่อ detect stuck sagas

**เมื่อใช้ Saga:**
- ต้องการ multi-service transactions
- Services มี database แยกกัน
- Eventual consistency เป็นที่ยอมรับได้

---

*[← Part 50: Data Pipeline](part-50-data-pipeline.md) | [Part 52: Outbox Pattern →](part-52-outbox-pattern.md)*
