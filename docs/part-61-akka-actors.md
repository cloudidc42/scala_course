# Part 61: Akka Actors

## สารบัญ
1. [Actor Model คืออะไร](#actor-model-คืออะไร)
2. [ActorSystem และ ActorRef](#actorsystem-และ-actorref)
3. [Typed Actors: Behaviors](#typed-actors-behaviors)
4. [Actor Lifecycle และ Supervision](#actor-lifecycle-และ-supervision)
5. [Ask Pattern และ Future](#ask-pattern-และ-future)
6. [Actor Communication Patterns](#actor-communication-patterns)
7. [Persistence กับ Event Sourcing](#persistence-กับ-event-sourcing)
8. [ตัวอย่าง Actor System ครบถ้วน](#ตัวอย่าง-actor-system-ครบถ้วน)
9. [สรุป](#สรุป)

---

## Actor Model คืออะไร

Actor Model เป็น concurrency model ที่ Carl Hewitt คิดค้นขึ้นในปี 1973 ซึ่ง Akka นำมาใช้เป็น foundation ของ framework

### หลักการสำคัญ

```
Actor Model Principles:
┌─────────────────────────────────────────────────────┐
│                    Actor System                     │
│  ┌──────────┐    message    ┌──────────┐           │
│  │  Actor A │ ────────────► │  Actor B │           │
│  │          │               │          │           │
│  │ private  │               │ private  │           │
│  │  state   │               │  state   │           │
│  └──────────┘               └──────────┘           │
│       │                          │                 │
│       │ spawn                    │ spawn           │
│       ▼                          ▼                 │
│  ┌──────────┐               ┌──────────┐           │
│  │  Child A │               │  Child B │           │
│  └──────────┘               └──────────┘           │
└─────────────────────────────────────────────────────┘

กฎ 3 ข้อ:
1. แต่ละ Actor มี state เป็นของตัวเอง ไม่ share กัน
2. Actors สื่อสารผ่าน message เท่านั้น
3. Messages ถูก process ทีละ message (sequential)
```

### ทำไมต้องใช้ Actors?

```scala
// ❌ Shared mutable state - prone to race conditions
var counter = 0
// Thread 1:
counter += 1  // Read: 0, Write: 1
// Thread 2 (concurrent):
counter += 1  // Read: 0, Write: 1  ← LOST UPDATE!
// Result: 1 แทนที่จะเป็น 2!

// ✅ Actor - ไม่มี race condition
// Counter actor รับ messages ทีละอัน
// state เปลี่ยนแปลงอย่าง sequential และ safe
```

---

## ActorSystem และ ActorRef

### ตั้งค่า Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-actor-typed"         % "2.9.3",
  "com.typesafe.akka" %% "akka-actor-testkit-typed" % "2.9.3" % Test,
  "ch.qos.logback"     % "logback-classic"           % "1.5.6",
)
```

### ActorSystem คืออะไร

```scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors

// ActorSystem คือ "โลก" ที่ actors ทำงานอยู่
// มักจะมีแค่ 1 ActorSystem ต่อ application
val system: ActorSystem[Nothing] =
  ActorSystem(Behaviors.empty, "my-system")

// ActorRef คือ "ที่อยู่" ของ actor
// เราส่ง message ผ่าน ActorRef เสมอ
// ไม่เคย access actor โดยตรง
```

### Hello World Actor

```scala
import akka.actor.typed.{ActorRef, ActorSystem, Behavior}
import akka.actor.typed.scaladsl.Behaviors

// 1. กำหนด Protocol (messages ที่ actor รับได้)
object Greeter:
  // sealed trait สำหรับ type safety
  sealed trait Command
  case class Greet(whom: String, replyTo: ActorRef[Greeted]) extends Command

  case class Greeted(whom: String, from: ActorRef[Command])

  // 2. กำหนด Behavior
  def apply(): Behavior[Command] =
    Behaviors.receive { (context, message) =>
      message match
        case Greet(whom, replyTo) =>
          context.log.info(s"Hello, $whom!")
          replyTo ! Greeted(whom, context.self)
          Behaviors.same  // ยังคง behavior เดิม
    }

// 3. ใช้งาน
object GreeterMain:
  sealed trait Command
  case class SayHello(name: String) extends Command
  case class GreetingDone(whom: String) extends Command

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      val greeter = context.spawn(Greeter(), "greeter")

      Behaviors.receiveMessage {
        case SayHello(name) =>
          val replyAdapter = context.messageAdapter[Greeter.Greeted] { g =>
            GreetingDone(g.whom)
          }
          greeter ! Greeter.Greet(name, replyAdapter)
          Behaviors.same

        case GreetingDone(whom) =>
          context.log.info(s"Greeting of $whom done!")
          Behaviors.same
      }
    }

@main def runGreeter(): Unit =
  val system = ActorSystem(GreeterMain(), "hello-akka")
  system ! GreeterMain.SayHello("World")
  system ! GreeterMain.SayHello("Akka")

  Thread.sleep(1000)
  system.terminate()
```

---

## Typed Actors: Behaviors

### Behaviors API

```scala
import akka.actor.typed.scaladsl.Behaviors
import akka.actor.typed.Behavior

object BehaviorExamples:
  sealed trait Msg
  case class Add(n: Int) extends Msg
  case object Reset extends Msg
  case class Get(replyTo: ActorRef[Int]) extends Msg
  case object Stop extends Msg

  // Behaviors.receive - รับทั้ง context และ message
  def counterBehavior(count: Int = 0): Behavior[Msg] =
    Behaviors.receive { (context, msg) =>
      msg match
        case Add(n) =>
          context.log.debug(s"Adding $n to $count")
          counterBehavior(count + n)  // recursive: return new behavior

        case Reset =>
          context.log.info("Counter reset")
          counterBehavior(0)

        case Get(replyTo) =>
          replyTo ! count
          Behaviors.same  // ไม่เปลี่ยน behavior

        case Stop =>
          Behaviors.stopped  // หยุด actor
    }

  // Behaviors.receiveMessage - รับแค่ message (ไม่มี context)
  def simpleCounter(count: Int = 0): Behavior[Msg] =
    Behaviors.receiveMessage {
      case Add(n)   => simpleCounter(count + n)
      case Reset    => simpleCounter(0)
      case Get(to)  => to ! count; Behaviors.same
      case Stop     => Behaviors.stopped
    }
```

### Stateful Behavior ด้วย Behaviors.setup

```scala
object StatefulActor:
  sealed trait Command
  case class Store(key: String, value: String) extends Command
  case class Retrieve(key: String, replyTo: ActorRef[Option[String]]) extends Command
  case object Clear extends Command

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      context.log.info("KeyValue store initialized")

      // state เก็บในตัวแปร mutable ภายใน closure
      // ปลอดภัยเพราะ actor process message ทีละอัน
      var store = Map.empty[String, String]

      Behaviors.receiveMessage {
        case Store(key, value) =>
          store = store + (key -> value)
          context.log.debug(s"Stored: $key = $value")
          Behaviors.same

        case Retrieve(key, replyTo) =>
          replyTo ! store.get(key)
          Behaviors.same

        case Clear =>
          store = Map.empty
          context.log.info("Store cleared")
          Behaviors.same
      }
    }
```

### Behavior Transitions (State Machine)

```scala
object TrafficLight:
  sealed trait Command
  case object Tick extends Command
  case class Query(replyTo: ActorRef[String]) extends Command

  sealed trait State
  case object Red extends State
  case object Yellow extends State
  case object Green extends State

  def red(): Behavior[Command] =
    Behaviors.receiveMessage {
      case Tick =>
        println("Red -> Green")
        green()  // เปลี่ยนไป green behavior
      case Query(to) =>
        to ! "RED"
        Behaviors.same
    }

  def green(): Behavior[Command] =
    Behaviors.receiveMessage {
      case Tick =>
        println("Green -> Yellow")
        yellow()
      case Query(to) =>
        to ! "GREEN"
        Behaviors.same
    }

  def yellow(): Behavior[Command] =
    Behaviors.receiveMessage {
      case Tick =>
        println("Yellow -> Red")
        red()
      case Query(to) =>
        to ! "YELLOW"
        Behaviors.same
    }

  def apply(): Behavior[Command] = red()

@main def runTrafficLight(): Unit =
  val system = ActorSystem(TrafficLight(), "traffic")
  // Tick ทุก 1 วินาที
  (1 to 6).foreach { _ =>
    Thread.sleep(1000)
    system ! TrafficLight.Tick
  }
  Thread.sleep(500)
  system.terminate()
```

### Behaviors.withTimers

```scala
import akka.actor.typed.scaladsl.TimerScheduler
import scala.concurrent.duration.*

object HeartbeatActor:
  sealed trait Command
  private case object Tick extends Command
  case class Subscribe(id: String, replyTo: ActorRef[String]) extends Command
  case class Unsubscribe(id: String) extends Command

  def apply(): Behavior[Command] =
    Behaviors.withTimers { timers =>
      timers.startTimerWithFixedDelay(Tick, 1.second)
      running(timers, Map.empty)
    }

  def running(
    timers: TimerScheduler[Command],
    subscribers: Map[String, ActorRef[String]]
  ): Behavior[Command] =
    Behaviors.receiveMessage {
      case Tick =>
        val time = java.time.Instant.now().toString
        subscribers.values.foreach(_ ! s"heartbeat: $time")
        Behaviors.same

      case Subscribe(id, ref) =>
        running(timers, subscribers + (id -> ref))

      case Unsubscribe(id) =>
        running(timers, subscribers - id)
    }
```

---

## Actor Lifecycle และ Supervision

### Lifecycle Hooks

```scala
object LifecycleActor:
  sealed trait Command
  case class Process(data: String) extends Command
  case object Fail extends Command

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      context.log.info(s"Actor starting: ${context.self.path}")

      // PostStop hook
      Behaviors.receiveMessage[Command] {
        case Process(data) =>
          context.log.info(s"Processing: $data")
          Behaviors.same

        case Fail =>
          throw new RuntimeException("Intentional failure!")
      }
      .receiveSignal {
        case (context, akka.actor.typed.PreRestart) =>
          context.log.warn("Actor restarting...")
          Behaviors.same

        case (context, akka.actor.typed.PostStop) =>
          context.log.info("Actor stopped, cleaning up")
          Behaviors.same
      }
    }
```

### Supervision Strategies

```scala
import akka.actor.typed.SupervisorStrategy
import scala.concurrent.duration.*

object SupervisedActor:
  sealed trait Command
  case class DoWork(n: Int) extends Command

  def apply(): Behavior[Command] =
    Behaviors.supervise {
      Behaviors.setup[Command] { context =>
        Behaviors.receiveMessage {
          case DoWork(n) if n < 0 =>
            throw new IllegalArgumentException(s"Negative: $n")
          case DoWork(n) =>
            context.log.info(s"Working on $n")
            Behaviors.same
        }
      }
    }.onFailure[IllegalArgumentException](
      SupervisorStrategy.restart
        .withLimit(maxNrOfRetries = 3, withinTimeRange = 1.minute)
    )

// Parent managing children
object ParentActor:
  sealed trait Command
  case class CreateWorker(id: String) extends Command
  case class DelegateWork(workerId: String, n: Int) extends Command
  case class WorkerFailed(id: String, cause: Throwable) extends Command

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      var workers = Map.empty[String, ActorRef[SupervisedActor.Command]]

      // Watch children for failure
      context.watch(context.self)

      Behaviors.receiveMessage {
        case CreateWorker(id) =>
          val worker = context.spawn(
            Behaviors.supervise(SupervisedActor())
              .onFailure(SupervisorStrategy.restart),
            s"worker-$id"
          )
          context.watchWith(worker, WorkerFailed(id, new Exception("died")))
          workers = workers + (id -> worker)
          Behaviors.same

        case DelegateWork(workerId, n) =>
          workers.get(workerId).foreach(_ ! SupervisedActor.DoWork(n))
          Behaviors.same

        case WorkerFailed(id, cause) =>
          context.log.error(s"Worker $id failed: ${cause.getMessage}")
          workers = workers - id
          Behaviors.same
      }
    }
```

### Death Watch

```scala
import akka.actor.typed.{Terminated, ChildFailed}

object MonitorActor:
  sealed trait Command
  case class WatchActor(ref: ActorRef[?]) extends Command

  def apply(): Behavior[Command] =
    Behaviors.receive { (context, msg) =>
      msg match
        case WatchActor(ref) =>
          context.watch(ref)
          Behaviors.same
    }
    .receiveSignal {
      case (context, Terminated(ref)) =>
        context.log.info(s"Actor terminated: ${ref.path}")
        Behaviors.same

      case (context, ChildFailed(ref, cause)) =>
        context.log.error(s"Child failed: ${ref.path}, cause: $cause")
        Behaviors.same
    }
```

---

## Ask Pattern และ Future

### การใช้ Ask Pattern

```scala
import akka.actor.typed.scaladsl.AskPattern.*
import akka.util.Timeout
import scala.concurrent.{ExecutionContext, Future}
import scala.concurrent.duration.*

object Calculator:
  sealed trait Command
  case class Calculate(
    op: String,
    a: Double,
    b: Double,
    replyTo: ActorRef[Result]
  ) extends Command

  case class Result(value: Either[String, Double])

  def apply(): Behavior[Command] =
    Behaviors.receiveMessage {
      case Calculate("add", a, b, replyTo) =>
        replyTo ! Result(Right(a + b))
        Behaviors.same
      case Calculate("divide", a, b, replyTo) =>
        if b == 0 then replyTo ! Result(Left("Division by zero"))
        else replyTo ! Result(Right(a / b))
        Behaviors.same
      case Calculate(op, _, _, replyTo) =>
        replyTo ! Result(Left(s"Unknown operation: $op"))
        Behaviors.same
    }

// การใช้งาน ask
object CalculatorClient:
  def main(args: Array[String]): Unit =
    given system: ActorSystem[Nothing] = ActorSystem(Behaviors.empty, "calc")
    given ec: ExecutionContext = system.executionContext
    given timeout: Timeout = 3.seconds

    val calc = system.systemActorOf(Calculator(), "calculator")

    // Ask returns Future
    val result: Future[Calculator.Result] =
      calc.ask(Calculator.Calculate("add", 3.0, 4.0, _))

    result.foreach { r =>
      println(s"Result: ${r.value}")  // Right(7.0)
    }

    // Chain multiple asks
    val combined: Future[Double] =
      for
        r1 <- calc.ask(Calculator.Calculate("add", 10.0, 5.0, _))
        r2 <- calc.ask(Calculator.Calculate("divide", r1.value.getOrElse(0), 3.0, _))
      yield r2.value.getOrElse(0.0)

    combined.foreach { v =>
      println(s"Combined result: $v")  // 5.0
    }

    Thread.sleep(1000)
    system.terminate()
```

### Pipe Pattern

```scala
import akka.actor.typed.scaladsl.AskPattern.*
import scala.concurrent.Future

object PipeExample:
  sealed trait Command
  case class FetchData(url: String) extends Command
  private case class DataFetched(data: String) extends Command
  private case class FetchFailed(cause: Throwable) extends Command

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      given ec: scala.concurrent.ExecutionContext =
        context.executionContext

      Behaviors.receiveMessage {
        case FetchData(url) =>
          // simulate async fetch
          val future: Future[String] =
            Future { s"Data from $url" }  // ใน production: real HTTP call

          // pipe Future result back to self
          context.pipeToSelf(future) {
            case scala.util.Success(data) => DataFetched(data)
            case scala.util.Failure(ex)   => FetchFailed(ex)
          }
          Behaviors.same

        case DataFetched(data) =>
          context.log.info(s"Got data: $data")
          Behaviors.same

        case FetchFailed(ex) =>
          context.log.error(s"Fetch failed: ${ex.getMessage}")
          Behaviors.same
      }
    }
```

---

## Actor Communication Patterns

### Request-Response

```scala
object RequestResponse:
  // Service actor
  object TemperatureService:
    sealed trait Command
    case class GetTemp(city: String, replyTo: ActorRef[Response]) extends Command
    sealed trait Response
    case class TempResult(city: String, celsius: Double) extends Response
    case class CityNotFound(city: String) extends Response

    private val temps = Map(
      "Bangkok" -> 35.0,
      "London" -> 15.0,
      "Tokyo" -> 22.0
    )

    def apply(): Behavior[Command] =
      Behaviors.receiveMessage {
        case GetTemp(city, replyTo) =>
          temps.get(city) match
            case Some(t) => replyTo ! TempResult(city, t)
            case None    => replyTo ! CityNotFound(city)
          Behaviors.same
      }

  // Client actor
  object WeatherClient:
    sealed trait Command
    case class CheckWeather(city: String) extends Command
    private case class GotTemp(result: TemperatureService.Response) extends Command

    def apply(service: ActorRef[TemperatureService.Command]): Behavior[Command] =
      Behaviors.receive { (context, msg) =>
        msg match
          case CheckWeather(city) =>
            val adapter = context.messageAdapter[TemperatureService.Response](GotTemp.apply)
            service ! TemperatureService.GetTemp(city, adapter)
            Behaviors.same

          case GotTemp(TemperatureService.TempResult(city, temp)) =>
            context.log.info(f"$city: ${temp}%.1f°C")
            Behaviors.same

          case GotTemp(TemperatureService.CityNotFound(city)) =>
            context.log.warn(s"City not found: $city")
            Behaviors.same
      }
```

### Publish-Subscribe

```scala
import akka.actor.typed.pubsub.{PubSub, Topic}

object PubSubExample:
  case class Message(content: String, sender: String)

  sealed trait ChatCommand
  case class Publish(msg: Message) extends ChatCommand
  case class Subscribe(subscriber: ActorRef[Message]) extends ChatCommand

  def apply(): Behavior[ChatCommand] =
    Behaviors.setup { context =>
      val topic = context.spawn(Topic[Message]("chat-messages"), "chat-topic")

      Behaviors.receiveMessage {
        case Publish(msg) =>
          topic ! Topic.Publish(msg)
          Behaviors.same

        case Subscribe(subscriber) =>
          topic ! Topic.Subscribe(subscriber)
          Behaviors.same
      }
    }
```

### Router Pattern

```scala
import akka.actor.typed.receptionist.{Receptionist, ServiceKey}

object RouterExample:
  val WorkerKey: ServiceKey[Worker.Command] =
    ServiceKey[Worker.Command]("workers")

  object Worker:
    sealed trait Command
    case class DoJob(id: Int, payload: String) extends Command

    def apply(workerId: String): Behavior[Command] =
      Behaviors.setup { context =>
        // ลงทะเบียนกับ receptionist
        context.system.receptionist ! Receptionist.Register(WorkerKey, context.self)

        Behaviors.receiveMessage {
          case DoJob(id, payload) =>
            context.log.info(s"Worker $workerId processing job $id: $payload")
            Thread.sleep(100)  // simulate work
            Behaviors.same
        }
      }

  object Dispatcher:
    sealed trait Command
    case class Dispatch(jobId: Int, payload: String) extends Command
    private case class WorkersUpdated(workers: Set[ActorRef[Worker.Command]]) extends Command

    def apply(): Behavior[Command] =
      Behaviors.setup { context =>
        val subscribeAdapter = context.messageAdapter[Receptionist.Listing] {
          case WorkerKey.Listing(workers) => WorkersUpdated(workers)
        }
        context.system.receptionist ! Receptionist.Subscribe(WorkerKey, subscribeAdapter)

        dispatching(Vector.empty, 0)
      }

    def dispatching(
      workers: Vector[ActorRef[Worker.Command]],
      roundRobin: Int
    ): Behavior[Command] =
      Behaviors.receiveMessage {
        case WorkersUpdated(newWorkers) =>
          dispatching(newWorkers.toVector, roundRobin)

        case Dispatch(jobId, payload) if workers.nonEmpty =>
          val idx = roundRobin % workers.size
          workers(idx) ! Worker.DoJob(jobId, payload)
          dispatching(workers, roundRobin + 1)

        case Dispatch(jobId, _) =>
          println(s"No workers available for job $jobId")
          Behaviors.same
      }
```

---

## Persistence กับ Event Sourcing

### Akka Persistence Typed

```scala
// build.sbt: "com.typesafe.akka" %% "akka-persistence-typed" % "2.9.3"
import akka.persistence.typed.scaladsl.{Effect, EventSourcedBehavior}
import akka.persistence.typed.PersistenceId

object BankAccount:
  // Commands
  sealed trait Command
  case class Deposit(amount: BigDecimal, replyTo: ActorRef[Response]) extends Command
  case class Withdraw(amount: BigDecimal, replyTo: ActorRef[Response]) extends Command
  case class GetBalance(replyTo: ActorRef[BigDecimal]) extends Command

  // Events (persisted to journal)
  sealed trait Event
  case class Deposited(amount: BigDecimal) extends Event
  case class Withdrawn(amount: BigDecimal) extends Event

  // Responses
  sealed trait Response
  case object Success extends Response
  case class Failure(reason: String) extends Response

  // State
  case class State(balance: BigDecimal = 0)

  def apply(accountId: String): Behavior[Command] =
    EventSourcedBehavior[Command, Event, State](
      persistenceId = PersistenceId.ofUniqueId(accountId),
      emptyState    = State(),
      commandHandler = handleCommand,
      eventHandler   = handleEvent
    )

  private def handleCommand(state: State, cmd: Command): Effect[Event, State] =
    cmd match
      case Deposit(amount, replyTo) =>
        Effect
          .persist(Deposited(amount))
          .thenReply(replyTo)(_ => Success)

      case Withdraw(amount, replyTo) =>
        if state.balance >= amount then
          Effect
            .persist(Withdrawn(amount))
            .thenReply(replyTo)(_ => Success)
        else
          Effect.reply(replyTo)(Failure(s"Insufficient funds: ${state.balance} < $amount"))

      case GetBalance(replyTo) =>
        Effect.reply(replyTo)(state.balance)

  private def handleEvent(state: State, event: Event): State =
    event match
      case Deposited(amount) => state.copy(balance = state.balance + amount)
      case Withdrawn(amount) => state.copy(balance = state.balance - amount)
```

---

## ตัวอย่าง Actor System ครบถ้วน

### Order Processing System

```scala
package com.example.orders

import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.util.Timeout
import scala.concurrent.duration.*
import java.util.UUID

// Domain
case class Order(
  id: String,
  customerId: String,
  items: List[OrderItem],
  status: OrderStatus
)
case class OrderItem(productId: String, qty: Int, price: BigDecimal)

enum OrderStatus:
  case Pending, PaymentPending, Confirmed, Shipped, Delivered, Cancelled

// Inventory Actor
object InventoryActor:
  sealed trait Command
  case class CheckAvailability(
    productId: String,
    qty: Int,
    replyTo: ActorRef[AvailabilityResult]
  ) extends Command
  case class Reserve(productId: String, qty: Int) extends Command
  case class Release(productId: String, qty: Int) extends Command

  sealed trait AvailabilityResult
  case object Available extends AvailabilityResult
  case class Unavailable(productId: String, available: Int) extends AvailabilityResult

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      var stock = Map(
        "PROD-001" -> 100,
        "PROD-002" -> 50,
        "PROD-003" -> 200
      )

      Behaviors.receiveMessage {
        case CheckAvailability(productId, qty, replyTo) =>
          val available = stock.getOrElse(productId, 0)
          if available >= qty then replyTo ! Available
          else replyTo ! Unavailable(productId, available)
          Behaviors.same

        case Reserve(productId, qty) =>
          val current = stock.getOrElse(productId, 0)
          stock = stock.updated(productId, math.max(0, current - qty))
          context.log.info(s"Reserved $qty of $productId, remaining: ${stock(productId)}")
          Behaviors.same

        case Release(productId, qty) =>
          val current = stock.getOrElse(productId, 0)
          stock = stock.updated(productId, current + qty)
          Behaviors.same
      }
    }

// Payment Actor
object PaymentActor:
  sealed trait Command
  case class ProcessPayment(
    orderId: String,
    amount: BigDecimal,
    replyTo: ActorRef[PaymentResult]
  ) extends Command

  sealed trait PaymentResult
  case class PaymentSuccess(transactionId: String) extends PaymentResult
  case class PaymentFailed(reason: String) extends PaymentResult

  def apply(): Behavior[Command] =
    Behaviors.receiveMessage {
      case ProcessPayment(orderId, amount, replyTo) =>
        // Simulate payment processing
        if amount > 0 then
          val txId = UUID.randomUUID().toString.take(8).toUpperCase
          replyTo ! PaymentSuccess(txId)
        else
          replyTo ! PaymentFailed("Invalid amount")
        Behaviors.same
    }

// Order Actor (Per-order state machine)
object OrderActor:
  sealed trait Command
  case class StartProcessing(
    inventory: ActorRef[InventoryActor.Command],
    payment: ActorRef[PaymentActor.Command]
  ) extends Command
  case class GetStatus(replyTo: ActorRef[Order]) extends Command
  private case class InventoryChecked(result: InventoryActor.AvailabilityResult) extends Command
  private case class PaymentProcessed(result: PaymentActor.PaymentResult) extends Command

  def apply(order: Order): Behavior[Command] =
    processing(order)

  def processing(order: Order): Behavior[Command] =
    Behaviors.receive { (context, msg) =>
      msg match
        case StartProcessing(inventory, payment) =>
          // Check all items
          val adapter = context.messageAdapter[InventoryActor.AvailabilityResult](InventoryChecked.apply)
          order.items.foreach { item =>
            inventory ! InventoryActor.CheckAvailability(item.productId, item.qty, adapter)
          }
          checking(order, inventory, payment, pendingChecks = order.items.size)

        case GetStatus(replyTo) =>
          replyTo ! order
          Behaviors.same

        case _ =>
          Behaviors.same
    }

  def checking(
    order: Order,
    inventory: ActorRef[InventoryActor.Command],
    payment: ActorRef[PaymentActor.Command],
    pendingChecks: Int
  ): Behavior[Command] =
    Behaviors.receive { (context, msg) =>
      msg match
        case InventoryChecked(InventoryActor.Available) =>
          if pendingChecks == 1 then
            // All items available, process payment
            val adapter = context.messageAdapter[PaymentActor.PaymentResult](PaymentProcessed.apply)
            val total = order.items.map(i => i.price * i.qty).sum
            payment ! PaymentActor.ProcessPayment(order.id, total, adapter)
            awaitingPayment(order.copy(status = OrderStatus.PaymentPending), inventory)
          else
            checking(order, inventory, payment, pendingChecks - 1)

        case InventoryChecked(InventoryActor.Unavailable(productId, available)) =>
          context.log.warn(s"Item $productId unavailable (have $available)")
          processing(order.copy(status = OrderStatus.Cancelled))

        case GetStatus(replyTo) =>
          replyTo ! order
          Behaviors.same

        case _ =>
          Behaviors.same
    }

  def awaitingPayment(
    order: Order,
    inventory: ActorRef[InventoryActor.Command]
  ): Behavior[Command] =
    Behaviors.receive { (context, msg) =>
      msg match
        case PaymentProcessed(PaymentActor.PaymentSuccess(txId)) =>
          context.log.info(s"Payment $txId successful for order ${order.id}")
          // Reserve inventory
          order.items.foreach { item =>
            inventory ! InventoryActor.Reserve(item.productId, item.qty)
          }
          val confirmed = order.copy(status = OrderStatus.Confirmed)
          processing(confirmed)

        case PaymentProcessed(PaymentActor.PaymentFailed(reason)) =>
          context.log.error(s"Payment failed: $reason")
          processing(order.copy(status = OrderStatus.Cancelled))

        case GetStatus(replyTo) =>
          replyTo ! order
          Behaviors.same

        case _ =>
          Behaviors.same
    }

// Order Manager
object OrderManager:
  sealed trait Command
  case class CreateOrder(
    customerId: String,
    items: List[OrderItem],
    replyTo: ActorRef[String]
  ) extends Command
  case class QueryOrder(orderId: String, replyTo: ActorRef[Option[Order]]) extends Command

  def apply(): Behavior[Command] =
    Behaviors.setup { context =>
      val inventory = context.spawn(InventoryActor(), "inventory")
      val payment   = context.spawn(PaymentActor(), "payment")
      var orders    = Map.empty[String, ActorRef[OrderActor.Command]]

      Behaviors.receiveMessage {
        case CreateOrder(customerId, items, replyTo) =>
          val orderId = UUID.randomUUID().toString.take(8)
          val order = Order(orderId, customerId, items, OrderStatus.Pending)
          val orderActor = context.spawn(OrderActor(order), s"order-$orderId")
          orders = orders + (orderId -> orderActor)
          orderActor ! OrderActor.StartProcessing(inventory, payment)
          replyTo ! orderId
          Behaviors.same

        case QueryOrder(orderId, replyTo) =>
          orders.get(orderId) match
            case Some(actor) =>
              given timeout: Timeout = 3.seconds
              given ec: scala.concurrent.ExecutionContext =
                context.executionContext
              context.pipeToSelf(
                actor.ask[Order](OrderActor.GetStatus.apply)
              ) {
                case scala.util.Success(order) =>
                  QueryOrder(orderId, replyTo)
                case _ =>
                  QueryOrder(orderId, replyTo)
              }
              // Simplified: just reply None for now
              replyTo ! None
              Behaviors.same
            case None =>
              replyTo ! None
              Behaviors.same
      }
    }

// Main
@main def runOrderSystem(): Unit =
  val system = ActorSystem(OrderManager(), "order-system")
  given timeout: Timeout = 5.seconds
  given ec: scala.concurrent.ExecutionContext = system.executionContext

  val items = List(
    OrderItem("PROD-001", 2, BigDecimal("29.99")),
    OrderItem("PROD-002", 1, BigDecimal("49.99"))
  )

  val orderId = akka.actor.typed.scaladsl.AskPattern
    .ask(system, (replyTo: ActorRef[String]) =>
      OrderManager.CreateOrder("CUST-123", items, replyTo)
    )(timeout, system.scheduler)

  orderId.foreach { id =>
    println(s"Created order: $id")
  }

  Thread.sleep(2000)
  system.terminate()
```

---

## สรุป

Akka Typed Actors ให้ความสามารถที่ทรงพลังสำหรับ concurrent และ distributed systems:

- ✅ **Type Safety**: Typed Actors บังคับ message protocol ตั้งแต่ compile time
- ✅ **Immutability**: State เปลี่ยนแปลงผ่าน message เท่านั้น ไม่มี shared mutable state
- ✅ **Fault Tolerance**: Supervision hierarchy จัดการ failure อัตโนมัติ
- ✅ **Location Transparency**: Actors ทำงานได้ทั้ง local และ remote ด้วย code เดียวกัน
- ✅ **Scalability**: Actor system scale ได้ทั้ง vertical และ horizontal

Key patterns:
- **Behavior transitions** สำหรับ state machines
- **Ask pattern** สำหรับ request-response
- **Pipe to self** สำหรับ async operations
- **Router** สำหรับ load balancing
- **Event sourcing** สำหรับ persistence

---

*[← Part 60: GraalVM Native Image](part-60-graalvm.md) | [Part 62: Akka Streams →](part-62-akka-streams.md)*
