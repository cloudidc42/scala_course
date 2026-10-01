# ส่วนที่ 73: Reactive Architecture - สถาปัตยกรรมระบบเชิงปฏิกิริยา

## สารบัญ

1. [Reactive Manifesto: Responsive, Resilient, Elastic, Message-Driven](#reactive-manifesto)
2. [CQRS ด้วย Akka Persistence](#cqrs-akka-persistence)
3. [Event-Driven Architecture](#event-driven)
4. [Reactive DDD](#reactive-ddd)
5. [Sharding ด้วย Akka Cluster Sharding](#cluster-sharding)
6. [Distributed Data ด้วย Akka Distributed Data](#distributed-data)
7. [การออกแบบ Reactive System สมบูรณ์](#complete-reactive-system)
8. [สรุป](#summary)

---

## 1. Reactive Manifesto: Responsive, Resilient, Elastic, Message-Driven {#reactive-manifesto}

### หลักการ 4 ข้อของ Reactive Systems

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-actor-typed"           % "2.8.5",
  "com.typesafe.akka" %% "akka-persistence-typed"     % "2.8.5",
  "com.typesafe.akka" %% "akka-cluster-typed"         % "2.8.5",
  "com.typesafe.akka" %% "akka-cluster-sharding-typed" % "2.8.5",
  "com.typesafe.akka" %% "akka-distributed-data"      % "2.8.5",
  "com.typesafe.akka" %% "akka-stream"                % "2.8.5",
  "com.typesafe.akka" %% "akka-http"                  % "10.5.2",
  "org.typelevel"     %% "cats-effect"                % "3.5.4"
)
```

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import scala.concurrent.duration.*

// === 1. RESPONSIVE: ตอบสนองรวดเร็วและสม่ำเสมอ ===
object ResponsiveExample:
  
  sealed trait Command
  case class ProcessRequest(id: String, replyTo: ActorRef[Response]) extends Command
  case class TimeoutCheck(id: String) extends Command
  
  sealed trait Response
  case class Success(id: String, result: String) extends Response
  case class Failure(id: String, reason: String) extends Response
  case class Timeout(id: String) extends Response
  
  def apply(): Behavior[Command] = Behaviors.withTimers { timers =>
    Behaviors.receiveMessage {
      case ProcessRequest(id, replyTo) =>
        // ตั้ง timeout เพื่อ responsive guarantee
        timers.startSingleTimer(
          TimeoutCheck(id),
          TimeoutCheck(id),
          500.millis  // timeout หลัง 500ms
        )
        
        // ประมวลผลแบบ async
        val result = scala.util.Try {
          // simulate work
          Thread.sleep(100)
          s"Processed: $id"
        }
        
        result match
          case scala.util.Success(v) =>
            replyTo ! Success(id, v)
          case scala.util.Failure(e) =>
            replyTo ! Failure(id, e.getMessage)
        
        Behaviors.same
        
      case TimeoutCheck(id) =>
        println(s"Warning: Request $id timed out")
        Behaviors.same
    }
  }

// === 2. RESILIENT: ทนทานต่อความล้มเหลว ===
object ResilientExample:
  
  sealed trait WorkerCommand
  case class DoWork(payload: String, replyTo: ActorRef[WorkResult]) extends WorkerCommand
  
  sealed trait WorkResult
  case class WorkDone(result: String) extends WorkResult
  case class WorkFailed(error: String) extends WorkResult
  
  // Worker ที่อาจล้มเหลว
  def worker(): Behavior[WorkerCommand] = 
    Behaviors.receiveMessage {
      case DoWork(payload, replyTo) =>
        try
          // อาจ throw exception
          if payload.isEmpty then throw new IllegalArgumentException("Empty payload")
          replyTo ! WorkDone(s"Result: $payload")
        catch
          case e: Exception => replyTo ! WorkFailed(e.getMessage)
        Behaviors.same
    }
  
  // Supervisor strategy
  def supervisedWorker(): Behavior[WorkerCommand] =
    Behaviors.supervise(worker())
      .onFailure[Exception](
        SupervisorStrategy.restart
          .withLimit(maxNrOfRetries = 3, withinTimeRange = 1.minute)
      )
  
  // Circuit Breaker pattern
  import akka.pattern.CircuitBreaker
  import scala.concurrent.{Future, ExecutionContext}
  import akka.actor.ActorSystem
  
  class ServiceWithCircuitBreaker(implicit system: ActorSystem):
    import system.dispatcher
    
    val breaker = new CircuitBreaker(
      system.scheduler,
      maxFailures = 5,
      callTimeout = 10.seconds,
      resetTimeout = 1.minute
    )
    .onOpen(println("Circuit Breaker OPENED"))
    .onClose(println("Circuit Breaker CLOSED"))
    .onHalfOpen(println("Circuit Breaker HALF-OPEN"))
    
    def callExternalService(request: String): Future[String] =
      breaker.withCircuitBreaker {
        Future {
          // External service call
          if scala.util.Random.nextDouble() < 0.3 then
            throw new RuntimeException("Service unavailable")
          s"Response for: $request"
        }
      }

// === 3. ELASTIC: ขยายหรือลดขนาดได้ตามต้องการ ===
object ElasticExample:
  
  import akka.actor.typed.scaladsl.Routers
  
  sealed trait Task
  case class ProcessItem(item: Int, replyTo: ActorRef[Int]) extends Task
  
  def worker(): Behavior[Task] = Behaviors.receiveMessage {
    case ProcessItem(item, replyTo) =>
      replyTo ! item * 2
      Behaviors.same
  }
  
  def elasticPool(): Behavior[Task] =
    // Router pool ที่ scale ได้
    val pool = Routers.pool(poolSize = Runtime.getRuntime.availableProcessors())(
      Behaviors.supervise(worker()).onFailure[Exception](SupervisorStrategy.restart)
    )
    pool
  
  // Dynamic scaling
  def dynamicWorkerPool(): Behavior[Nothing] = Behaviors.setup { ctx =>
    var workers = Vector.empty[ActorRef[Task]]
    
    def scale(targetSize: Int): Unit =
      val currentSize = workers.size
      if targetSize > currentSize then
        workers ++= (currentSize until targetSize).map { i =>
          ctx.spawn(worker(), s"worker-$i")
        }
      else if targetSize < currentSize then
        workers.drop(targetSize).foreach(ctx.stop)
        workers = workers.take(targetSize)
    
    // เริ่มต้นด้วย 4 workers
    scale(4)
    println(s"Started with ${workers.size} workers")
    
    Behaviors.empty
  }

// === 4. MESSAGE-DRIVEN: สื่อสารด้วย asynchronous messages ===
object MessageDrivenExample:
  
  sealed trait OrderCommand
  case class PlaceOrder(orderId: String, amount: Double) extends OrderCommand
  case class CancelOrder(orderId: String, reason: String) extends OrderCommand
  
  sealed trait PaymentCommand
  case class ProcessPayment(orderId: String, amount: Double) extends PaymentCommand
  case class RefundPayment(orderId: String) extends PaymentCommand
  
  sealed trait NotificationCommand
  case class SendEmail(to: String, subject: String, body: String) extends NotificationCommand
  case class SendSMS(to: String, message: String) extends NotificationCommand
  
  // Message-passing ระหว่าง actors
  def orderService(
    paymentRef: ActorRef[PaymentCommand],
    notificationRef: ActorRef[NotificationCommand]
  ): Behavior[OrderCommand] =
    Behaviors.receiveMessage {
      case PlaceOrder(orderId, amount) =>
        println(s"Order $orderId placed for $amount")
        paymentRef ! ProcessPayment(orderId, amount)  // async message
        notificationRef ! SendEmail(
          "customer@example.com",
          s"Order $orderId Confirmed",
          s"Your order worth $amount has been placed."
        )
        Behaviors.same
        
      case CancelOrder(orderId, reason) =>
        println(s"Order $orderId cancelled: $reason")
        paymentRef ! RefundPayment(orderId)
        notificationRef ! SendEmail(
          "customer@example.com",
          s"Order $orderId Cancelled",
          s"Your order has been cancelled. Reason: $reason"
        )
        Behaviors.same
    }
```

---

## 2. CQRS ด้วย Akka Persistence {#cqrs-akka-persistence}

### Command Query Responsibility Segregation

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.persistence.typed.scaladsl.*
import akka.persistence.typed.PersistenceId

object CQRSWithAkkaPersistence:
  
  // === Commands ===
  sealed trait BankAccountCommand
  case class CreateAccount(
    accountId: String, 
    owner: String, 
    initialBalance: BigDecimal,
    replyTo: ActorRef[BankAccountReply]
  ) extends BankAccountCommand
  
  case class Deposit(
    amount: BigDecimal, 
    description: String,
    replyTo: ActorRef[BankAccountReply]
  ) extends BankAccountCommand
  
  case class Withdraw(
    amount: BigDecimal, 
    description: String,
    replyTo: ActorRef[BankAccountReply]
  ) extends BankAccountCommand
  
  case class GetBalance(replyTo: ActorRef[BankAccountReply]) extends BankAccountCommand
  case class GetStatement(replyTo: ActorRef[BankAccountReply]) extends BankAccountCommand
  
  // === Replies ===
  sealed trait BankAccountReply
  case class AccountCreated(accountId: String) extends BankAccountReply
  case class TransactionSucceeded(newBalance: BigDecimal) extends BankAccountReply
  case class TransactionFailed(reason: String) extends BankAccountReply
  case class BalanceResult(balance: BigDecimal) extends BankAccountReply
  case class StatementResult(transactions: List[Transaction]) extends BankAccountReply
  
  // === Events ===
  sealed trait BankAccountEvent
  case class AccountOpened(
    accountId: String, 
    owner: String, 
    initialBalance: BigDecimal
  ) extends BankAccountEvent
  
  case class MoneyDeposited(
    amount: BigDecimal, 
    description: String,
    timestamp: Long
  ) extends BankAccountEvent
  
  case class MoneyWithdrawn(
    amount: BigDecimal, 
    description: String,
    timestamp: Long
  ) extends BankAccountEvent
  
  // === State ===
  case class Transaction(
    kind: String,
    amount: BigDecimal,
    description: String,
    timestamp: Long
  )
  
  sealed trait BankAccountState
  case object UninitializedAccount extends BankAccountState
  case class ActiveAccount(
    accountId: String,
    owner: String,
    balance: BigDecimal,
    transactions: List[Transaction]
  ) extends BankAccountState
  
  // === Event Sourced Behavior ===
  def apply(accountId: String): Behavior[BankAccountCommand] =
    EventSourcedBehavior[BankAccountCommand, BankAccountEvent, BankAccountState](
      persistenceId = PersistenceId.ofUniqueId(accountId),
      emptyState = UninitializedAccount,
      commandHandler = commandHandler,
      eventHandler = eventHandler
    ).withRetention(
      RetentionCriteria.snapshotEvery(numberOfEvents = 100, keepNSnapshots = 2)
    )
  
  // Command Handler - บทบาท Write Side
  private val commandHandler: (BankAccountState, BankAccountCommand) => 
    Effect[BankAccountEvent, BankAccountState] = { (state, command) =>
    
    (state, command) match
      case (UninitializedAccount, CreateAccount(id, owner, initial, replyTo)) =>
        if initial < 0 then
          Effect.reply(replyTo)(TransactionFailed("Initial balance cannot be negative"))
        else
          Effect.persist(AccountOpened(id, owner, initial))
            .thenReply(replyTo)(_ => AccountCreated(id))
      
      case (UninitializedAccount, _) =>
        command match
          case cmd: { val replyTo: ActorRef[BankAccountReply] } =>
            Effect.reply(cmd.replyTo)(TransactionFailed("Account not created yet"))
          case _ => Effect.none
      
      case (account: ActiveAccount, Deposit(amount, desc, replyTo)) =>
        if amount <= 0 then
          Effect.reply(replyTo)(TransactionFailed("Deposit amount must be positive"))
        else
          Effect.persist(MoneyDeposited(amount, desc, System.currentTimeMillis()))
            .thenReply(replyTo)(state => state match
              case a: ActiveAccount => TransactionSucceeded(a.balance)
              case _ => TransactionFailed("Unexpected state")
            )
      
      case (account: ActiveAccount, Withdraw(amount, desc, replyTo)) =>
        if amount <= 0 then
          Effect.reply(replyTo)(TransactionFailed("Amount must be positive"))
        else if account.balance < amount then
          Effect.reply(replyTo)(TransactionFailed(
            s"Insufficient funds: balance ${account.balance} < $amount"
          ))
        else
          Effect.persist(MoneyWithdrawn(amount, desc, System.currentTimeMillis()))
            .thenReply(replyTo)(state => state match
              case a: ActiveAccount => TransactionSucceeded(a.balance)
              case _ => TransactionFailed("Unexpected state")
            )
      
      case (account: ActiveAccount, GetBalance(replyTo)) =>
        Effect.reply(replyTo)(BalanceResult(account.balance))
      
      case (account: ActiveAccount, GetStatement(replyTo)) =>
        Effect.reply(replyTo)(StatementResult(account.transactions))
      
      case _ => Effect.unhandled
  }
  
  // Event Handler - อัพเดท State
  private val eventHandler: (BankAccountState, BankAccountEvent) => BankAccountState = {
    case (_, AccountOpened(id, owner, initial)) =>
      ActiveAccount(id, owner, initial, List.empty)
    
    case (account: ActiveAccount, MoneyDeposited(amount, desc, ts)) =>
      account.copy(
        balance = account.balance + amount,
        transactions = account.transactions :+ Transaction("DEPOSIT", amount, desc, ts)
      )
    
    case (account: ActiveAccount, MoneyWithdrawn(amount, desc, ts)) =>
      account.copy(
        balance = account.balance - amount,
        transactions = account.transactions :+ Transaction("WITHDRAWAL", amount, desc, ts)
      )
    
    case (state, _) => state
  }
  
  // === Read Side: Query Model ===
  // ใน production ใช้ Akka Projection เพื่อสร้าง read model
  import akka.persistence.query.*
  
  case class AccountSummary(
    accountId: String,
    owner: String,
    balance: BigDecimal,
    txCount: Int,
    lastActivity: Long
  )
  
  // Read model จะ subscribe event stream และ update read DB
  def buildReadModel(events: List[BankAccountEvent]): Map[String, AccountSummary] =
    events.foldLeft(Map.empty[String, AccountSummary]) { (model, event) =>
      event match
        case AccountOpened(id, owner, initial) =>
          model + (id -> AccountSummary(id, owner, initial, 0, System.currentTimeMillis()))
        
        case MoneyDeposited(amount, _, ts) =>
          model  // ต้องรู้ accountId - ใน real impl จะมา from envelope
        
        case MoneyWithdrawn(amount, _, ts) =>
          model  // เช่นเดียวกัน
    }
```

---

## 3. Event-Driven Architecture {#event-driven}

### Event Bus และ Event Processing

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.actor.typed.eventstream.EventStream

object EventDrivenArchitecture:
  
  // Domain Events
  sealed trait DomainEvent:
    def eventId: String
    def timestamp: Long
    def aggregateId: String
  
  case class UserRegistered(
    eventId: String,
    userId: String,
    email: String,
    name: String,
    timestamp: Long
  ) extends DomainEvent:
    def aggregateId = userId
  
  case class OrderPlaced(
    eventId: String,
    orderId: String,
    userId: String,
    items: List[OrderItem],
    total: BigDecimal,
    timestamp: Long
  ) extends DomainEvent:
    def aggregateId = orderId
  
  case class OrderShipped(
    eventId: String,
    orderId: String,
    trackingNumber: String,
    timestamp: Long
  ) extends DomainEvent:
    def aggregateId = orderId
  
  case class OrderItem(productId: String, quantity: Int, price: BigDecimal)
  
  // Event Bus ด้วย Akka EventStream
  object EventBusDemo:
    
    // Subscriber สำหรับ user events
    def userEventSubscriber(): Behavior[DomainEvent] =
      Behaviors.setup { ctx =>
        ctx.system.eventStream ! EventStream.Subscribe[UserRegistered](ctx.self)
        
        Behaviors.receiveMessage {
          case UserRegistered(id, userId, email, name, ts) =>
            println(s"[UserService] New user registered: $name ($email)")
            // ส่ง welcome email, สร้าง profile, etc.
            Behaviors.same
          case _ => Behaviors.same
        }
      }
    
    // Subscriber สำหรับ order events
    def orderEventSubscriber(): Behavior[DomainEvent] =
      Behaviors.setup { ctx =>
        ctx.system.eventStream ! EventStream.Subscribe[OrderPlaced](ctx.self)
        ctx.system.eventStream ! EventStream.Subscribe[OrderShipped](ctx.self)
        
        Behaviors.receiveMessage {
          case OrderPlaced(_, orderId, userId, items, total, _) =>
            println(s"[InventoryService] Update inventory for order $orderId")
            println(s"[EmailService] Send order confirmation to user $userId")
            Behaviors.same
          
          case OrderShipped(_, orderId, tracking, _) =>
            println(s"[NotificationService] Order $orderId shipped with tracking $tracking")
            Behaviors.same
          
          case _ => Behaviors.same
        }
      }
  
  // Saga Pattern - long-running transactions
  sealed trait SagaState
  case object SagaStarted extends SagaState
  case class OrderCreated(orderId: String) extends SagaState
  case class PaymentProcessed(paymentId: String) extends SagaState
  case class InventoryReserved(reservationId: String) extends SagaState
  case object SagaCompleted extends SagaState
  case class SagaFailed(step: String, reason: String) extends SagaState
  
  sealed trait SagaCommand
  case class StartOrderSaga(orderId: String, userId: String, amount: BigDecimal) 
    extends SagaCommand
  case class PaymentResult(success: Boolean, paymentId: Option[String]) 
    extends SagaCommand
  case class InventoryResult(success: Boolean, reservationId: Option[String]) 
    extends SagaCommand
  
  def orderSaga(
    paymentService: ActorRef[PaymentCommand],
    inventoryService: ActorRef[InventoryCommand]
  ): Behavior[SagaCommand] =
    
    sealed trait PaymentCommand
    case class ChargePayment(orderId: String, amount: BigDecimal, 
                             replyTo: ActorRef[SagaCommand]) extends PaymentCommand
    case class RefundPayment(paymentId: String) extends PaymentCommand
    
    sealed trait InventoryCommand
    case class ReserveInventory(orderId: String, replyTo: ActorRef[SagaCommand]) extends InventoryCommand
    case class ReleaseInventory(reservationId: String) extends InventoryCommand
    
    def waitingForOrder(): Behavior[SagaCommand] = Behaviors.receiveMessage {
      case StartOrderSaga(orderId, userId, amount) =>
        println(s"Saga started for order $orderId")
        // paymentService ! ChargePayment(orderId, amount, ctx.self)
        processingPayment(orderId, amount)
    }
    
    def processingPayment(orderId: String, amount: BigDecimal): Behavior[SagaCommand] =
      Behaviors.receiveMessage {
        case PaymentResult(true, Some(paymentId)) =>
          println(s"Payment $paymentId succeeded")
          // inventoryService ! ReserveInventory(orderId, ctx.self)
          reservingInventory(orderId, paymentId)
        
        case PaymentResult(false, _) =>
          println(s"Payment failed for order $orderId - Saga failed")
          Behaviors.stopped
        
        case _ => Behaviors.same
      }
    
    def reservingInventory(orderId: String, paymentId: String): Behavior[SagaCommand] =
      Behaviors.receiveMessage {
        case InventoryResult(true, Some(reservationId)) =>
          println(s"Inventory reserved: $reservationId - Saga completed!")
          Behaviors.stopped
        
        case InventoryResult(false, _) =>
          println(s"Inventory failed - compensating: refund payment $paymentId")
          // paymentService ! RefundPayment(paymentId)
          Behaviors.stopped
        
        case _ => Behaviors.same
      }
    
    waitingForOrder()
  
  // Outbox Pattern - guaranteeing event delivery
  case class OutboxEvent(
    id: String,
    aggregateType: String,
    aggregateId: String,
    eventType: String,
    payload: String,  // JSON serialized event
    createdAt: Long,
    publishedAt: Option[Long] = None
  )
  
  class OutboxProcessor:
    // ใน production: poll database outbox table
    def processOutbox(events: List[OutboxEvent]): List[OutboxEvent] =
      events.map { event =>
        // Publish ไป message broker
        println(s"Publishing event ${event.id} to ${event.aggregateType}")
        event.copy(publishedAt = Some(System.currentTimeMillis()))
      }
```

---

## 4. Reactive DDD {#reactive-ddd}

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*

object ReactiveDDD:
  
  // Value Objects
  case class Money(amount: BigDecimal, currency: String):
    require(amount >= 0, "Amount must be non-negative")
    
    def +(other: Money): Money =
      require(currency == other.currency, "Currency mismatch")
      copy(amount = amount + other.amount)
    
    def *(factor: BigDecimal): Money = copy(amount = amount * factor)
    
    override def toString: String = f"$amount%.2f $currency"
  
  case class ProductId(value: String) extends AnyVal
  case class CustomerId(value: String) extends AnyVal
  case class OrderId(value: String) extends AnyVal
  
  // Domain Events
  sealed trait OrderDomainEvent:
    def orderId: OrderId
    def occurredAt: Long = System.currentTimeMillis()
  
  case class OrderCreatedEvent(
    orderId: OrderId, 
    customerId: CustomerId,
    createdAt: Long = System.currentTimeMillis()
  ) extends OrderDomainEvent
  
  case class ItemAddedEvent(
    orderId: OrderId,
    productId: ProductId,
    quantity: Int,
    unitPrice: Money
  ) extends OrderDomainEvent
  
  case class OrderConfirmedEvent(
    orderId: OrderId,
    totalAmount: Money
  ) extends OrderDomainEvent
  
  case class OrderCancelledEvent(
    orderId: OrderId,
    reason: String
  ) extends OrderDomainEvent
  
  // Aggregate
  sealed trait OrderStatus
  case object Pending   extends OrderStatus
  case object Confirmed extends OrderStatus
  case object Cancelled extends OrderStatus
  
  case class OrderLine(
    productId: ProductId,
    quantity: Int,
    unitPrice: Money
  ):
    def subtotal: Money = unitPrice * quantity
  
  case class Order private (
    id: OrderId,
    customerId: CustomerId,
    lines: List[OrderLine],
    status: OrderStatus,
    events: List[OrderDomainEvent]
  ):
    def totalAmount: Money =
      if lines.isEmpty then Money(0, "THB")
      else lines.map(_.subtotal).reduce(_ + _)
    
    def addItem(productId: ProductId, qty: Int, price: Money): Either[String, Order] =
      if status != Pending then Left("Cannot add items to non-pending order")
      else
        val event = ItemAddedEvent(id, productId, qty, price)
        val newLine = OrderLine(productId, qty, price)
        Right(copy(
          lines = lines :+ newLine,
          events = events :+ event
        ))
    
    def confirm(): Either[String, Order] =
      if status != Pending then Left(s"Cannot confirm order in $status state")
      else if lines.isEmpty then Left("Cannot confirm empty order")
      else
        val event = OrderConfirmedEvent(id, totalAmount)
        Right(copy(status = Confirmed, events = events :+ event))
    
    def cancel(reason: String): Either[String, Order] =
      if status == Cancelled then Left("Order already cancelled")
      else
        val event = OrderCancelledEvent(id, reason)
        Right(copy(status = Cancelled, events = events :+ event))
    
    def uncommittedEvents: List[OrderDomainEvent] = events
    def clearEvents: Order = copy(events = Nil)
  
  object Order:
    def create(orderId: OrderId, customerId: CustomerId): Order =
      val event = OrderCreatedEvent(orderId, customerId)
      new Order(
        id = orderId,
        customerId = customerId,
        lines = Nil,
        status = Pending,
        events = List(event)
      )
  
  // Repository
  trait OrderRepository:
    def save(order: Order): Either[String, Order]
    def findById(orderId: OrderId): Option[Order]
    def findByCustomer(customerId: CustomerId): List[Order]
  
  class InMemoryOrderRepository extends OrderRepository:
    private var store = Map.empty[OrderId, Order]
    
    def save(order: Order): Either[String, Order] =
      store = store + (order.id -> order.clearEvents)
      Right(order)
    
    def findById(orderId: OrderId): Option[Order] = store.get(orderId)
    
    def findByCustomer(customerId: CustomerId): List[Order] =
      store.values.filter(_.customerId == customerId).toList
  
  // Application Service
  class OrderService(
    repo: OrderRepository,
    eventBus: List[OrderDomainEvent] => Unit  // publish events
  ):
    
    def createOrder(
      customerId: CustomerId
    ): Either[String, Order] =
      val orderId = OrderId(java.util.UUID.randomUUID().toString)
      val order = Order.create(orderId, customerId)
      for
        saved <- repo.save(order)
        _ = eventBus(order.uncommittedEvents)
      yield saved
    
    def addItem(
      orderId: OrderId,
      productId: ProductId,
      quantity: Int,
      price: Money
    ): Either[String, Order] =
      for
        order   <- repo.findById(orderId).toRight(s"Order $orderId not found")
        updated <- order.addItem(productId, quantity, price)
        saved   <- repo.save(updated)
        _ = eventBus(updated.uncommittedEvents)
      yield saved
    
    def confirmOrder(orderId: OrderId): Either[String, Order] =
      for
        order   <- repo.findById(orderId).toRight(s"Order $orderId not found")
        updated <- order.confirm()
        saved   <- repo.save(updated)
        _ = eventBus(updated.uncommittedEvents)
      yield saved
  
  def main(args: Array[String]): Unit =
    val repo = new InMemoryOrderRepository()
    val events = scala.collection.mutable.ListBuffer[OrderDomainEvent]()
    val service = new OrderService(repo, events ++= _)
    
    val customerId = CustomerId("C001")
    
    val result = for
      order    <- service.createOrder(customerId)
      updated1 <- service.addItem(
        order.id, ProductId("P001"), 2, Money(999, "THB")
      )
      updated2 <- service.addItem(
        order.id, ProductId("P002"), 1, Money(299, "THB")
      )
      confirmed <- service.confirmOrder(order.id)
    yield confirmed
    
    result match
      case Right(order) =>
        println(s"Order ${order.id.value} confirmed!")
        println(s"Total: ${order.totalAmount}")
        println(s"Lines: ${order.lines.size}")
      case Left(error) =>
        println(s"Error: $error")
    
    println(s"\nPublished events: ${events.map(_.getClass.getSimpleName).mkString(", ")}")
```

---

## 5. Sharding ด้วย Akka Cluster Sharding {#cluster-sharding}

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.cluster.sharding.typed.scaladsl.*
import akka.persistence.typed.scaladsl.*
import akka.persistence.typed.PersistenceId

object ClusterShardingDemo:
  
  // Entity Commands
  sealed trait ShoppingCartCommand:
    def cartId: String
  
  case class AddItem(
    cartId: String,
    itemId: String,
    quantity: Int,
    replyTo: ActorRef[ShoppingCartReply]
  ) extends ShoppingCartCommand
  
  case class RemoveItem(
    cartId: String,
    itemId: String,
    replyTo: ActorRef[ShoppingCartReply]
  ) extends ShoppingCartCommand
  
  case class Checkout(
    cartId: String,
    replyTo: ActorRef[ShoppingCartReply]
  ) extends ShoppingCartCommand
  
  case class GetCart(
    cartId: String,
    replyTo: ActorRef[ShoppingCartReply]
  ) extends ShoppingCartCommand
  
  // Replies
  sealed trait ShoppingCartReply
  case class CartUpdated(items: Map[String, Int]) extends ShoppingCartReply
  case class CheckoutSucceeded(total: Int) extends ShoppingCartReply
  case class CartContents(items: Map[String, Int]) extends ShoppingCartReply
  case class CartError(reason: String) extends ShoppingCartReply
  
  // Events
  sealed trait ShoppingCartEvent
  case class ItemAdded(itemId: String, quantity: Int) extends ShoppingCartEvent
  case class ItemRemoved(itemId: String) extends ShoppingCartEvent
  case class CartCheckedOut(timestamp: Long) extends ShoppingCartEvent
  
  // State
  sealed trait CartState
  case object EmptyCart extends CartState
  case class ActiveCart(items: Map[String, Int]) extends CartState:
    def addItem(itemId: String, qty: Int): ActiveCart =
      copy(items = items + (itemId -> (items.getOrElse(itemId, 0) + qty)))
    def removeItem(itemId: String): CartState =
      val newItems = items - itemId
      if newItems.isEmpty then EmptyCart else copy(items = newItems)
  case class CheckedOutCart(items: Map[String, Int]) extends CartState
  
  // Entity TypeKey - ใช้สำหรับ routing messages
  val EntityKey: EntityTypeKey[ShoppingCartCommand] =
    EntityTypeKey[ShoppingCartCommand]("ShoppingCart")
  
  // Entity Behavior (Event Sourced)
  def apply(cartId: String): Behavior[ShoppingCartCommand] =
    EventSourcedBehavior[ShoppingCartCommand, ShoppingCartEvent, CartState](
      persistenceId = PersistenceId(EntityKey.name, cartId),
      emptyState = EmptyCart,
      commandHandler = commandHandler,
      eventHandler = eventHandler
    ).withRetention(RetentionCriteria.snapshotEvery(50, 2))
  
  private val commandHandler: (CartState, ShoppingCartCommand) =>
    Effect[ShoppingCartEvent, CartState] = { (state, cmd) =>
    
    (state, cmd) match
      case (EmptyCart | _: ActiveCart, AddItem(_, itemId, qty, replyTo)) =>
        if qty <= 0 then Effect.reply(replyTo)(CartError("Quantity must be positive"))
        else
          Effect.persist(ItemAdded(itemId, qty))
            .thenReply(replyTo) {
              case ActiveCart(items) => CartUpdated(items)
              case _ => CartError("Unexpected state")
            }
      
      case (cart: ActiveCart, RemoveItem(_, itemId, replyTo)) =>
        if !cart.items.contains(itemId) then
          Effect.reply(replyTo)(CartError(s"Item $itemId not in cart"))
        else
          Effect.persist(ItemRemoved(itemId))
            .thenReply(replyTo) {
              case EmptyCart => CartUpdated(Map.empty)
              case ActiveCart(items) => CartUpdated(items)
              case _ => CartError("Unexpected state")
            }
      
      case (cart: ActiveCart, Checkout(_, replyTo)) =>
        Effect.persist(CartCheckedOut(System.currentTimeMillis()))
          .thenReply(replyTo)(_ => CheckoutSucceeded(cart.items.values.sum))
      
      case (state, GetCart(_, replyTo)) =>
        state match
          case EmptyCart          => Effect.reply(replyTo)(CartContents(Map.empty))
          case ActiveCart(items)  => Effect.reply(replyTo)(CartContents(items))
          case CheckedOutCart(items) => Effect.reply(replyTo)(CartContents(items))
      
      case (_, cmd) =>
        cmd match
          case c: { val replyTo: ActorRef[ShoppingCartReply] } =>
            Effect.reply(c.replyTo)(CartError(s"Command not allowed in $state state"))
          case _ => Effect.none
  }
  
  private val eventHandler: (CartState, ShoppingCartEvent) => CartState = {
    case (EmptyCart, ItemAdded(itemId, qty)) =>
      ActiveCart(Map(itemId -> qty))
    
    case (cart: ActiveCart, ItemAdded(itemId, qty)) =>
      cart.addItem(itemId, qty)
    
    case (cart: ActiveCart, ItemRemoved(itemId)) =>
      cart.removeItem(itemId)
    
    case (cart: ActiveCart, CartCheckedOut(_)) =>
      CheckedOutCart(cart.items)
    
    case (state, _) => state
  }
  
  // Setup Cluster Sharding
  def setupSharding(system: ActorSystem[Nothing]): ActorRef[ShardingEnvelope[ShoppingCartCommand]] =
    ClusterSharding(system).init(
      Entity(EntityKey) { entityCtx =>
        ShoppingCartDemo.apply(entityCtx.entityId)
      }
        .withSettings(
          ClusterShardingSettings(system)
            .withPassivateIdleEntityAfter(2.minutes)
        )
        .withMessageExtractor(
          ShardingMessageExtractor.noEnvelope(
            numberOfShards = 100,
            stopMessage = None
          ) { cmd => cmd.cartId }
        )
    )
  
  object ShoppingCartDemo:
    def apply(cartId: String): Behavior[ShoppingCartCommand] =
      ClusterShardingDemo.apply(cartId)
```

---

## 6. Distributed Data ด้วย Akka Distributed Data {#distributed-data}

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.cluster.ddata.typed.scaladsl.*
import akka.cluster.ddata.*

object AkkaDistributedDataDemo:
  
  // CRDTs ที่ Akka Distributed Data รองรับ
  // - Counter (GCounter, PNCounter)
  // - Set (GSet, ORSet)
  // - Map (ORMap, LWWMap)
  // - Register (LWWRegister)
  // - Flag (Flag)
  
  sealed trait ReplicationCommand
  case class IncrementCounter(key: String) extends ReplicationCommand
  case class GetCounter(key: String, replyTo: ActorRef[Long]) extends ReplicationCommand
  case class AddToSet(key: String, value: String) extends ReplicationCommand
  case class GetSet(key: String, replyTo: ActorRef[Set[String]]) extends ReplicationCommand
  case class UpdateMap(key: String, field: String, value: String) extends ReplicationCommand
  case class GetMap(key: String, replyTo: ActorRef[Map[String, String]]) extends ReplicationCommand
  
  def distributedDataActor(): Behavior[ReplicationCommand] =
    Behaviors.setup { ctx =>
      
      // DistributedData extension
      implicit val node: SelfUniqueAddress = 
        DistributedData(ctx.system).selfUniqueAddress
      
      val replicator = DistributedData(ctx.system).replicator
      
      Behaviors.receiveMessage {
        case IncrementCounter(key) =>
          val counterKey = PNCounterKey(key)
          replicator ! Replicator.Update(
            counterKey,
            PNCounter.empty,
            Replicator.WriteMajority(timeout = 3.seconds)
          )(_ :+ 1)
          Behaviors.same
        
        case GetCounter(key, replyTo) =>
          val counterKey = PNCounterKey(key)
          replicator ! Replicator.Get(
            counterKey,
            Replicator.ReadMajority(timeout = 3.seconds)
          )
          // ใน production จะ handle response ด้วย adapter
          Behaviors.same
        
        case AddToSet(key, value) =>
          val setKey = ORSetKey[String](key)
          replicator ! Replicator.Update(
            setKey,
            ORSet.empty[String],
            Replicator.WriteMajority(timeout = 3.seconds)
          )(_ :+ value)
          Behaviors.same
        
        case GetSet(key, replyTo) =>
          val setKey = ORSetKey[String](key)
          replicator ! Replicator.Get(setKey, Replicator.ReadMajority(timeout = 3.seconds))
          Behaviors.same
        
        case UpdateMap(key, field, value) =>
          val mapKey = LWWMapKey[String, String](key)
          replicator ! Replicator.Update(
            mapKey,
            LWWMap.empty[String, String],
            Replicator.WriteMajority(timeout = 3.seconds)
          )(_ :+ (field -> value))
          Behaviors.same
        
        case GetMap(key, replyTo) =>
          val mapKey = LWWMapKey[String, String](key)
          replicator ! Replicator.Get(mapKey, Replicator.ReadMajority(timeout = 3.seconds))
          Behaviors.same
      }
    }
  
  // ใช้ Distributed Data สำหรับ Rate Limiting
  class DistributedRateLimiter(system: ActorSystem[Nothing]):
    
    implicit val node: SelfUniqueAddress = 
      DistributedData(system).selfUniqueAddress
    
    val replicator = DistributedData(system).replicator
    
    // Check rate limit ด้วย PNCounter
    def checkRateLimit(userId: String, maxRequests: Int): Boolean =
      val key = PNCounterKey(s"rate-limit-$userId")
      // ใน real implementation จะเป็น async ด้วย Future หรือ ask pattern
      true  // simplified
    
    // Distributed session store
    def setSession(sessionId: String, userId: String): Unit =
      val key = LWWMapKey[String, String]("sessions")
      replicator ! Replicator.Update(
        key,
        LWWMap.empty[String, String],
        Replicator.WriteAll(timeout = 5.seconds)
      )(_ :+ (sessionId -> userId))
    
    // Feature flags ด้วย ORSet
    def enableFeature(feature: String): Unit =
      val key = ORSetKey[String]("enabled-features")
      replicator ! Replicator.Update(
        key,
        ORSet.empty[String],
        Replicator.WriteMajority(timeout = 3.seconds)
      )(_ :+ feature)
    
    def disableFeature(feature: String): Unit =
      val key = ORSetKey[String]("enabled-features")
      replicator ! Replicator.Update(
        key,
        ORSet.empty[String],
        Replicator.WriteMajority(timeout = 3.seconds)
      )(_.remove(feature))
```

---

## 7. การออกแบบ Reactive System สมบูรณ์ {#complete-reactive-system}

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import scala.concurrent.duration.*

object CompleteReactiveSystem:
  
  // === Domain Model ===
  case class ProductId(value: String) extends AnyVal
  case class UserId(value: String) extends AnyVal
  
  // === API Gateway ===
  sealed trait ApiGatewayCommand
  case class HandleRequest(
    method: String,
    path: String,
    body: String,
    replyTo: ActorRef[ApiResponse]
  ) extends ApiGatewayCommand
  
  case class ApiResponse(status: Int, body: String)
  
  // === Services ===
  sealed trait ProductCommand
  case class GetProduct(id: ProductId, replyTo: ActorRef[Option[Product]]) extends ProductCommand
  case class UpdateStock(id: ProductId, delta: Int) extends ProductCommand
  
  case class Product(id: ProductId, name: String, price: BigDecimal, stock: Int)
  
  sealed trait CartCommand
  case class AddToCart(userId: UserId, productId: ProductId, qty: Int) extends CartCommand
  case class GetCart(userId: UserId, replyTo: ActorRef[Cart]) extends CartCommand
  case class CheckoutCart(userId: UserId, replyTo: ActorRef[CheckoutResult]) extends CartCommand
  
  case class CartItem(productId: ProductId, quantity: Int, price: BigDecimal)
  case class Cart(userId: UserId, items: List[CartItem]):
    def total: BigDecimal = items.map(i => i.price * i.quantity).sum
  
  sealed trait CheckoutResult
  case class CheckoutSuccess(orderId: String, total: BigDecimal) extends CheckoutResult
  case class CheckoutFailure(reason: String) extends CheckoutResult
  
  // === Product Service Actor ===
  def productService(): Behavior[ProductCommand] =
    def running(catalog: Map[ProductId, Product]): Behavior[ProductCommand] =
      Behaviors.receiveMessage {
        case GetProduct(id, replyTo) =>
          replyTo ! catalog.get(id)
          Behaviors.same
        
        case UpdateStock(id, delta) =>
          catalog.get(id) match
            case Some(product) =>
              val newStock = math.max(0, product.stock + delta)
              running(catalog + (id -> product.copy(stock = newStock)))
            case None =>
              Behaviors.same
      }
    
    // Initial catalog
    val initialCatalog = Map(
      ProductId("P001") -> Product(ProductId("P001"), "Laptop", 29999, 50),
      ProductId("P002") -> Product(ProductId("P002"), "Mouse", 599, 200),
      ProductId("P003") -> Product(ProductId("P003"), "Keyboard", 1299, 150)
    )
    running(initialCatalog)
  
  // === Shopping Cart Actor ===
  def shoppingCartService(productRef: ActorRef[ProductCommand]): Behavior[CartCommand] =
    def running(carts: Map[UserId, Cart]): Behavior[CartCommand] =
      Behaviors.receiveMessage {
        case AddToCart(userId, productId, qty) =>
          val cart = carts.getOrElse(userId, Cart(userId, Nil))
          // ใน production จะ query product price ด้วย ask
          val item = CartItem(productId, qty, BigDecimal(999))  // simplified
          val updatedCart = cart.copy(items = cart.items :+ item)
          running(carts + (userId -> updatedCart))
        
        case GetCart(userId, replyTo) =>
          replyTo ! carts.getOrElse(userId, Cart(userId, Nil))
          Behaviors.same
        
        case CheckoutCart(userId, replyTo) =>
          carts.get(userId) match
            case None | Some(Cart(_, Nil)) =>
              replyTo ! CheckoutFailure("Cart is empty")
              Behaviors.same
            case Some(cart) =>
              val orderId = java.util.UUID.randomUUID().toString.take(8).toUpperCase
              // ส่ง events, update inventory, etc.
              cart.items.foreach { item =>
                productRef ! UpdateStock(item.productId, -item.quantity)
              }
              replyTo ! CheckoutSuccess(orderId, cart.total)
              running(carts - userId)  // clear cart after checkout
      }
    running(Map.empty)
  
  // === Metrics Actor ===
  sealed trait MetricsCommand
  case class RecordRequest(path: String, duration: Long) extends MetricsCommand
  case class RecordError(path: String, error: String) extends MetricsCommand
  case class GetMetrics(replyTo: ActorRef[MetricsReport]) extends MetricsCommand
  
  case class PathMetrics(
    count: Int, 
    errors: Int, 
    totalDuration: Long
  ):
    def avgDuration: Double = if count > 0 then totalDuration.toDouble / count else 0.0
  
  case class MetricsReport(paths: Map[String, PathMetrics])
  
  def metricsCollector(): Behavior[MetricsCommand] =
    def running(metrics: Map[String, PathMetrics]): Behavior[MetricsCommand] =
      Behaviors.receiveMessage {
        case RecordRequest(path, duration) =>
          val current = metrics.getOrElse(path, PathMetrics(0, 0, 0))
          running(metrics + (path -> current.copy(
            count = current.count + 1,
            totalDuration = current.totalDuration + duration
          )))
        
        case RecordError(path, _) =>
          val current = metrics.getOrElse(path, PathMetrics(0, 0, 0))
          running(metrics + (path -> current.copy(errors = current.errors + 1)))
        
        case GetMetrics(replyTo) =>
          replyTo ! MetricsReport(metrics)
          Behaviors.same
      }
    running(Map.empty)
  
  // === Main System ===
  def guardianBehavior(): Behavior[Nothing] = Behaviors.setup { ctx =>
    // สร้าง services
    val productRef = ctx.spawn(productService(), "product-service")
    val cartRef = ctx.spawn(shoppingCartService(productRef), "cart-service")
    val metricsRef = ctx.spawn(metricsCollector(), "metrics-service")
    
    // Health check
    ctx.system.scheduler.scheduleAtFixedRate(
      30.seconds, 30.seconds
    )(() => {
      println(s"[HealthCheck] System healthy at ${System.currentTimeMillis()}")
    })(ctx.executionContext)
    
    println("=== Reactive E-commerce System Started ===")
    println(s"Product Service: $productRef")
    println(s"Cart Service: $cartRef")
    println(s"Metrics Service: $metricsRef")
    
    Behaviors.empty
  }
  
  def main(args: Array[String]): Unit =
    val system = ActorSystem(guardianBehavior(), "reactive-ecommerce")
    
    import akka.actor.typed.scaladsl.AskPattern.*
    import scala.concurrent.duration.*
    import scala.concurrent.Await
    
    implicit val timeout: akka.util.Timeout = 3.seconds
    implicit val scheduler: akka.actor.Scheduler = system.scheduler
    
    // Demo: simulate requests
    Thread.sleep(1000)
    println("System is running. Press ENTER to stop.")
    scala.io.StdIn.readLine()
    system.terminate()
```

---

## 8. สรุป {#summary}

### สิ่งที่ได้เรียนรู้

1. **Reactive Manifesto**: เข้าใจหลัก 4 ข้อ - Responsive, Resilient, Elastic, Message-Driven
2. **CQRS + Event Sourcing**: แยก write/read model ด้วย Akka Persistence
3. **Event-Driven**: สื่อสารผ่าน events และ Saga pattern สำหรับ distributed transactions
4. **Reactive DDD**: ออกแบบ domain ที่เข้ากับ reactive architecture
5. **Cluster Sharding**: กระจาย entities ไปยัง cluster nodes
6. **Distributed Data**: ใช้ CRDTs สำหรับ eventual consistency

### เมื่อไหรควรใช้ Reactive Architecture

```scala
// ใช้ Reactive เมื่อ:
// 1. ต้องการ high availability (99.99%+)
// 2. ต้องการ horizontal scaling
// 3. มี complex event flows
// 4. ต้องการ geographic distribution
// 5. มี unpredictable load spikes

// ไม่จำเป็นเมื่อ:
// 1. Simple CRUD applications
// 2. Small team / startup MVP
// 3. Strong consistency required everywhere
// 4. Low traffic applications
```

---

*[← Part 72: Functional Programming Patterns](part-72-fp-patterns.md) | [Part 74: Advanced Concurrency →](part-74-advanced-concurrency.md)*
