# ส่วนที่ 96: Concurrent Data Structures

## สารบัญ

- [1. cats-effect Ref Internals](#1-cats-effect-ref-internals)
- [2. MVar: Mutable Cell with Synchronization](#2-mvar-mutable-cell-with-synchronization)
- [3. Queue: Bounded และ Unbounded](#3-queue-bounded-และ-unbounded)
- [4. Topic: Publish-Subscribe](#4-topic-publish-subscribe)
- [5. SignallingRef for Reactive State](#5-signallingref-for-reactive-state)
- [6. Complete Reactive State Management System](#6-complete-reactive-state-management-system)
- [สรุป](#สรุป)

---

## 1. cats-effect Ref Internals

`Ref` ใน cats-effect คือ mutable reference ที่ thread-safe สำหรับ concurrent programming

### พื้นฐาน Ref

```scala
import cats.effect.*
import cats.effect.syntax.all.*
import cats.syntax.all.*

// สร้าง Ref
def basicRef: IO[Unit] = for
  ref <- Ref[IO].of(0)         // สร้าง Ref ที่มีค่าเริ่มต้น 0
  _   <- ref.update(_ + 1)      // update value
  _   <- ref.update(_ + 1)
  v   <- ref.get                // อ่านค่า
  _   <- IO.println(s"Value: $v") // Value: 2
yield ()
```

### Ref Operations อย่างละเอียด

```scala
import cats.effect.*

def refOperations: IO[Unit] = for
  ref <- Ref[IO].of(10)
  
  // get: อ่านค่าปัจจุบัน
  v1 <- ref.get
  _ <- IO.println(s"Initial: $v1") // 10
  
  // set: กำหนดค่าใหม่ (ไม่สนใจค่าเดิม)
  _ <- ref.set(20)
  v2 <- ref.get
  _ <- IO.println(s"After set: $v2") // 20
  
  // update: แก้ไขค่าด้วย function
  _ <- ref.update(n => n * 2)
  v3 <- ref.get
  _ <- IO.println(s"After update: $v3") // 40
  
  // updateAndGet: แก้ไขและ return ค่าใหม่
  v4 <- ref.updateAndGet(n => n + 5)
  _ <- IO.println(s"After updateAndGet: $v4") // 45
  
  // getAndUpdate: return ค่าเดิม แล้วแก้ไข
  v5 <- ref.getAndUpdate(n => n - 5)
  _ <- IO.println(s"getAndUpdate returned: $v5") // 45
  v6 <- ref.get
  _ <- IO.println(s"Current: $v6") // 40
  
  // modify: แก้ไขและ return ค่าที่คำนวณได้
  result <- ref.modify(n => (n * 2, s"doubled to ${n * 2}"))
  _ <- IO.println(s"modify result: $result") // "doubled to 80"
  
  // modifyState: ใช้ State monad
  // compareAndSet: ใช้ CAS (Compare-And-Set)
  success <- ref.access.flatMap { case (current, setter) =>
    setter(current + 1)
  }
  _ <- IO.println(s"CAS success: $success")
yield ()
```

### Ref กับ Concurrent Updates

```scala
import cats.effect.*
import cats.syntax.parallel.*

// ทดสอบ concurrent safety
def concurrentCounter: IO[Int] =
  for
    counter <- Ref[IO].of(0)
    
    // รัน 1000 fibers พร้อมกัน แต่ละอันเพิ่ม 1
    _ <- (1 to 1000).toList.parTraverse { _ =>
      counter.update(_ + 1)
    }
    
    result <- counter.get
  yield result

// result จะเป็น 1000 เสมอ (thread-safe)

// ตัวอย่างที่ซับซ้อน: accumulate results
def collectResults: IO[List[Int]] =
  for
    results <- Ref[IO].of(List.empty[Int])
    _ <- (1 to 10).toList.parTraverse { i =>
      results.update(i :: _)
    }
    final_ <- results.get
  yield final_.sorted
```

### Ref internals: ใช้ AtomicReference

```scala
// ภายในใน cats-effect, Ref ใช้ java.util.concurrent.atomic.AtomicReference
// ทำให้ operations เป็น lock-free และ wait-free

// Low-level implementation concept:
import java.util.concurrent.atomic.AtomicReference

class SimpleRef[A](private val ar: AtomicReference[A]):
  def get: A = ar.get()
  
  def set(newValue: A): Unit = ar.set(newValue)
  
  def update(f: A => A): Unit =
    var done = false
    while !done do
      val current = ar.get()
      val newValue = f(current)
      done = ar.compareAndSet(current, newValue)
  
  def modify[B](f: A => (A, B)): B =
    var result: B = null.asInstanceOf[B]
    var done = false
    while !done do
      val current = ar.get()
      val (newValue, b) = f(current)
      if ar.compareAndSet(current, newValue) then
        result = b
        done = true
    result

// cats-effect ทำ wrap ด้วย IO เพื่อ referential transparency
```

---

## 2. MVar: Mutable Cell with Synchronization

`MVar` คือ mutable cell ที่ synchronize ระหว่าง producer และ consumer

### พื้นฐาน MVar

```scala
import cats.effect.*
import cats.effect.std.MVar

// MVar มีสองสถานะ: empty หรือ full
// - put: ใส่ค่า (block ถ้า full)
// - take: ดึงค่าออก (block ถ้า empty)
// - read: อ่านค่า (block ถ้า empty, ไม่เอาออก)

def basicMVar: IO[Unit] = for
  mvar <- MVar[IO].empty[Int]
  
  // take จะ block จน put
  fiber <- mvar.take.start
  
  // ใส่ค่าหลังจาก delay
  _ <- IO.sleep(100.milliseconds) >> mvar.put(42)
  
  // fiber จะ unblock และรับค่า
  result <- fiber.joinWithNever
  _ <- IO.println(s"Got: $result") // Got: 42
yield ()
```

### MVar เป็น Mutex (Mutual Exclusion)

```scala
import cats.effect.*
import cats.effect.std.MVar

// ใช้ MVar เป็น semaphore/mutex
class Mutex[F[_]: Concurrent]:
  private val token: MVar[F, Unit] = 
    MVar.in[F, F, Unit](Concurrent[F]).flatMap(_.put(())).unsafeRunSync()
  
  def withLock[A](fa: F[A]): F[A] =
    for
      _ <- token.take  // acquire lock
      result <- fa.guarantee(token.put(())) // release on finish
    yield result

// สร้าง mutex สำหรับ shared resource
def criticalSection: IO[Unit] = for
  mutex <- MVar[IO].of(())  // initialized = unlocked
  
  // simulate multiple threads accessing shared resource
  shared <- Ref[IO].of(0)
  
  _ <- (1 to 5).toList.parTraverse { i =>
    for
      _ <- mutex.take    // acquire
      current <- shared.get
      _ <- IO.sleep(10.milliseconds) // simulate work
      _ <- shared.set(current + i)
      _ <- mutex.put(()) // release
    yield ()
  }
  
  result <- shared.get
  _ <- IO.println(s"Result: $result") // 15 (1+2+3+4+5)
yield ()
```

### MVar สำหรับ Producer-Consumer

```scala
import cats.effect.*
import cats.effect.std.MVar
import cats.syntax.all.*

def producerConsumer: IO[Unit] = for
  channel <- MVar[IO].empty[Int]
  
  // Producer
  producer = (1 to 5).toList.traverse_ { i =>
    IO.println(s"Producing $i") >>
    channel.put(i) >>
    IO.sleep(50.milliseconds)
  }
  
  // Consumer
  consumer = (1 to 5).toList.traverse { _ =>
    channel.take.flatTap(v => IO.println(s"Consumed $v"))
  }
  
  // รัน concurrent
  (_, results) <- (producer, consumer).parTupled
  _ <- IO.println(s"All consumed: $results")
yield ()
```

---

## 3. Queue: Bounded และ Unbounded

`Queue` ใน cats-effect เหมาะสำหรับ producer-consumer patterns

### Unbounded Queue

```scala
import cats.effect.*
import cats.effect.std.Queue

def unboundedQueue: IO[Unit] = for
  q <- Queue.unbounded[IO, Int]
  
  // Offer (non-blocking สำหรับ unbounded)
  _ <- q.offer(1)
  _ <- q.offer(2)
  _ <- q.offer(3)
  
  // Take (block ถ้า empty)
  v1 <- q.take
  v2 <- q.take
  v3 <- q.take
  
  _ <- IO.println(s"$v1, $v2, $v3") // 1, 2, 3 (FIFO)
  
  // tryTake: non-blocking
  _ <- q.offer(4)
  v4 <- q.tryTake
  v5 <- q.tryTake // empty
  _ <- IO.println(s"tryTake: $v4, $v5") // Some(4), None
yield ()
```

### Bounded Queue (Back-pressure)

```scala
import cats.effect.*
import cats.effect.std.Queue
import cats.syntax.all.*

def boundedQueue: IO[Unit] = for
  // Queue ขนาด 3
  q <- Queue.bounded[IO, Int](3)
  
  // Offer จะ block เมื่อ full
  _ <- q.offer(1) // OK
  _ <- q.offer(2) // OK
  _ <- q.offer(3) // OK
  
  // offer ที่ 4 จะ block จน consumer เอาออก
  offerFiber <- q.offer(4).start
  
  // Consumer เอาออก 1 item
  v <- q.take
  _ <- IO.println(s"Took: $v") // 1
  
  // offer ที่ 4 สำเร็จได้แล้ว
  _ <- offerFiber.join
  
  // tryOffer: non-blocking version
  success1 <- q.tryOffer(5)
  success2 <- q.tryOffer(6) // full (มี 2, 3, 4, 5 อยู่)
  // wait, after take 1, and 4 was added, we have 2, 3, 4 in queue
  _ <- IO.println(s"tryOffer: $success1") // true
yield ()
```

### Circular Buffer Queue

```scala
import cats.effect.*
import cats.effect.std.Queue

// Dropping Queue: ทิ้ง item เก่าเมื่อ full
def droppingQueue: IO[Unit] = for
  q <- Queue.dropping[IO, Int](3)
  
  _ <- q.offer(1)
  _ <- q.offer(2)
  _ <- q.offer(3)
  
  // เมื่อ full, item ใหม่จะถูก drop (offer ไม่ block แต่ return false)
  success <- q.tryOffer(4)
  _ <- IO.println(s"Dropped: ${!success}")
  
  items <- q.take *> q.take *> q.take
  _ <- IO.println("Queue had 1, 2, 3")
yield ()

// Circularly-dropping Queue: ทิ้ง item เก่าแทน
def circularQueue: IO[Unit] = for
  q <- Queue.circularBuffer[IO, Int](3)
  
  _ <- q.offer(1)
  _ <- q.offer(2)
  _ <- q.offer(3)
  // เมื่อ full, item เก่าสุดถูกทิ้ง
  _ <- q.offer(4) // 1 ถูกทิ้ง
  _ <- q.offer(5) // 2 ถูกทิ้ง
  
  v1 <- q.take
  v2 <- q.take
  v3 <- q.take
  _ <- IO.println(s"$v1, $v2, $v3") // 3, 4, 5
yield ()
```

### Queue กับ fs2 Streams

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.Stream

def queueWithStreams: IO[Unit] = for
  q <- Queue.bounded[IO, Option[Int]](100)
  
  // Stream producer
  producer = Stream.range(1, 11)
    .evalMap(i => q.offer(Some(i)))
    .append(Stream.eval(q.offer(None))) // sentinel value
  
  // Stream consumer
  consumer = Stream.fromQueueNoneTerminated(q)
    .evalMap(i => IO.println(s"Processing: $i"))
  
  // Run both
  _ <- producer.compile.drain.start
  _ <- consumer.compile.drain
yield ()
```

---

## 4. Topic: Publish-Subscribe

`Topic` คือ concurrent pub-sub mechanism ที่ subscribers ทุกคนได้รับ messages ทุกอัน

### พื้นฐาน Topic

```scala
import cats.effect.*
import cats.effect.std.Topic
import fs2.Stream

def basicTopic: IO[Unit] = for
  topic <- Topic[IO, String]
  
  // Subscriber 1
  sub1 = topic.subscribe(10).take(3).compile.toList
  
  // Subscriber 2  
  sub2 = topic.subscribe(10).take(3).compile.toList
  
  // Start subscribers
  f1 <- sub1.start
  f2 <- sub2.start
  
  // Publish messages
  _ <- IO.sleep(10.milliseconds)
  _ <- topic.publish1("Hello")
  _ <- topic.publish1("World")
  _ <- topic.publish1("!")
  
  // Get results
  r1 <- f1.joinWithNever
  r2 <- f2.joinWithNever
  
  _ <- IO.println(s"Sub1: $r1") // List(Hello, World, !)
  _ <- IO.println(s"Sub2: $r2") // List(Hello, World, !)
yield ()
```

### Topic กับ Multiple Subscribers

```scala
import cats.effect.*
import cats.effect.std.Topic
import cats.syntax.all.*
import fs2.Stream

def multiSubscriberTopic: IO[Unit] = for
  topic <- Topic[IO, Int]
  
  // สร้าง 5 subscribers พร้อมกัน
  subscriberFibers <- (1 to 5).toList.traverse { id =>
    topic.subscribe(100)
      .take(3)
      .evalMap(msg => IO.println(s"Subscriber $id got: $msg"))
      .compile.drain
      .start
  }
  
  // รอ subscribers พร้อม
  _ <- IO.sleep(50.milliseconds)
  
  // Publish
  _ <- Stream.range(1, 4)
    .covary[IO]
    .through(topic.publish)
    .compile.drain
  
  // รอทุก subscriber เสร็จ
  _ <- subscriberFibers.traverse_(_.join)
yield ()
```

### Topic: Event Bus Pattern

```scala
import cats.effect.*
import cats.effect.std.Topic
import fs2.Stream

// Events
sealed trait AppEvent
case class UserLoggedIn(userId: String, timestamp: Long) extends AppEvent
case class OrderPlaced(orderId: String, amount: Double) extends AppEvent
case class PaymentProcessed(orderId: String, success: Boolean) extends AppEvent

class EventBus[F[_]: Concurrent]:
  private val topic: F[Topic[F, AppEvent]] = Topic[F, AppEvent]
  
  def publish(event: AppEvent): F[Unit] =
    topic.flatMap(_.publish1(event))
  
  def subscribe: Stream[F, AppEvent] =
    Stream.eval(topic).flatMap(_.subscribe(1000))
  
  def subscribeFiltered[E <: AppEvent](pf: PartialFunction[AppEvent, E]): Stream[F, E] =
    subscribe.collect(pf)

// Service ที่ใช้ EventBus
def orderService(bus: EventBus[IO]): IO[Unit] =
  bus.subscribeFiltered { case e: OrderPlaced => e }
    .evalMap { order =>
      IO.println(s"Processing order: ${order.orderId}")
    }
    .compile.drain

def notificationService(bus: EventBus[IO]): IO[Unit] =
  bus.subscribe
    .evalMap {
      case UserLoggedIn(id, ts) =>
        IO.println(s"User $id logged in at $ts")
      case PaymentProcessed(id, true) =>
        IO.println(s"Payment for order $id succeeded - sending confirmation")
      case PaymentProcessed(id, false) =>
        IO.println(s"Payment for order $id failed - sending alert")
      case _ => IO.unit
    }
    .compile.drain
```

---

## 5. SignallingRef for Reactive State

`SignallingRef` รวม `Ref` กับ `Signal` ทำให้ components สามารถ react ต่อการเปลี่ยนแปลง state

### พื้นฐาน SignallingRef

```scala
import cats.effect.*
import cats.effect.std.SignallingRef
import fs2.Stream

def basicSignallingRef: IO[Unit] = for
  // สร้าง SignallingRef ที่มีค่าเริ่มต้น
  signal <- SignallingRef[IO, Int](0)
  
  // Subscribe ต่อ changes (เป็น Stream)
  subscriber = signal.discrete  // เฉพาะเมื่อค่าเปลี่ยน
  // หรือ signal.continuous     // ส่งค่าปัจจุบันเสมอ
  
  // Start subscriber
  fiber <- subscriber.take(3)
    .evalMap(v => IO.println(s"Signal changed to: $v"))
    .compile.drain
    .start
  
  // Update signal
  _ <- IO.sleep(10.milliseconds)
  _ <- signal.set(1)
  _ <- IO.sleep(10.milliseconds)
  _ <- signal.set(2)
  _ <- IO.sleep(10.milliseconds)
  _ <- signal.set(3)
  
  _ <- fiber.join
yield ()
```

### SignallingRef: Interruptible Processes

```scala
import cats.effect.*
import cats.effect.std.SignallingRef
import fs2.Stream
import scala.concurrent.duration.*

def interruptibleProcess: IO[Unit] = for
  // Signal ที่ใช้ control process termination
  stopSignal <- SignallingRef[IO, Boolean](false)
  
  // Background process ที่ทำงานจน signal เป็น true
  process = Stream.repeatEval(IO.println("Working...") >> IO.sleep(200.milliseconds))
    .interruptWhen(stopSignal.discrete)
    .compile.drain
  
  // Start process
  fiber <- process.start
  
  // ทำงาน 1 วินาที แล้วหยุด
  _ <- IO.sleep(1.second)
  _ <- stopSignal.set(true) // ส่ง stop signal
  
  _ <- fiber.join
  _ <- IO.println("Process stopped!")
yield ()
```

### State Machine ด้วย SignallingRef

```scala
import cats.effect.*
import cats.effect.std.SignallingRef
import fs2.Stream

// Application state
sealed trait AppState
case object Starting extends AppState
case object Running extends AppState
case class Paused(reason: String) extends AppState
case object Stopping extends AppState
case class Failed(error: String) extends AppState

class StateMachine[F[_]: Concurrent: Temporal]:
  def create(initial: AppState): F[StateMachineInstance[F]] =
    SignallingRef[F, AppState](initial).map(StateMachineInstance(_))

class StateMachineInstance[F[_]: Concurrent](private val ref: SignallingRef[F, AppState]):
  def currentState: F[AppState] = ref.get
  
  def stateStream: fs2.Stream[F, AppState] = ref.discrete
  
  def transition(to: AppState): F[Boolean] =
    ref.modify { current =>
      (current, to) match
        case (Starting, Running)             => (Running, true)
        case (Running, Paused(reason))       => (Paused(reason), true)
        case (Paused(_), Running)            => (Running, true)
        case (Running, Stopping)             => (Stopping, true)
        case (_, Failed(err))                => (Failed(err), true)
        case (current, _)                    => (current, false) // invalid transition
    }
  
  def waitForState(target: AppState): F[Unit] =
    ref.discrete
      .find(_ == target)
      .compile.drain

// ใช้งาน
def runStateMachine: IO[Unit] = for
  machine <- StateMachineInstance[IO](???)
  
  // Monitor state changes
  monitor = machine.stateStream
    .evalMap(s => IO.println(s"State: $s"))
    .compile.drain
  
  _ <- monitor.start
  
  _ <- machine.transition(Running)
  _ <- IO.sleep(100.milliseconds)
  _ <- machine.transition(Paused("maintenance"))
  _ <- IO.sleep(100.milliseconds)
  _ <- machine.transition(Running)
  _ <- IO.sleep(100.milliseconds)
  _ <- machine.transition(Stopping)
yield ()
```

---

## 6. Complete Reactive State Management System

ตัวอย่างสมบูรณ์: สร้างระบบ reactive state management

### Data Models

```scala
import cats.effect.*
import cats.effect.std.*
import fs2.Stream
import cats.syntax.all.*
import scala.concurrent.duration.*

// Domain models
case class Product(id: String, name: String, price: Double, stock: Int)
case class CartItem(productId: String, quantity: Int)
case class Cart(items: Map[String, CartItem], total: Double)

sealed trait CartEvent
case class ItemAdded(productId: String, quantity: Int) extends CartEvent
case class ItemRemoved(productId: String) extends CartEvent
case class ItemQuantityChanged(productId: String, quantity: Int) extends CartEvent
case object CartCleared extends CartEvent
```

### Store Implementation

```scala
class CartStore[F[_]: Concurrent](
  private val state: SignallingRef[F, Cart],
  private val events: Topic[F, CartEvent],
  private val products: Ref[F, Map[String, Product]]
):
  // Query
  def currentCart: F[Cart] = state.get
  def cartStream: Stream[F, Cart] = state.discrete
  def eventStream: Stream[F, CartEvent] = events.subscribe(1000)
  
  // Commands
  def addItem(productId: String, qty: Int): F[Either[String, Unit]] =
    products.get.flatMap { prods =>
      prods.get(productId) match
        case None =>
          Concurrent[F].pure(Left(s"Product $productId not found"))
        case Some(product) if product.stock < qty =>
          Concurrent[F].pure(Left(s"Insufficient stock"))
        case Some(product) =>
          val updateCart = state.update { cart =>
            val newItems = cart.items.updatedWith(productId) {
              case None       => Some(CartItem(productId, qty))
              case Some(item) => Some(item.copy(quantity = item.quantity + qty))
            }
            val newTotal = calcTotal(newItems, prods)
            Cart(newItems, newTotal)
          }
          val publishEvent = events.publish1(ItemAdded(productId, qty))
          (updateCart >> publishEvent).as(Right(()))
    }
  
  def removeItem(productId: String): F[Unit] =
    products.get.flatMap { prods =>
      val updateCart = state.update { cart =>
        val newItems = cart.items.removed(productId)
        Cart(newItems, calcTotal(newItems, prods))
      }
      val publishEvent = events.publish1(ItemRemoved(productId))
      updateCart >> publishEvent
    }
  
  def clearCart: F[Unit] =
    state.set(Cart(Map.empty, 0.0)) >>
    events.publish1(CartCleared)
  
  private def calcTotal(items: Map[String, CartItem], prods: Map[String, Product]): Double =
    items.values.foldLeft(0.0) { (acc, item) =>
      prods.get(item.productId)
        .map(p => acc + p.price * item.quantity)
        .getOrElse(acc)
    }

object CartStore:
  def create[F[_]: Concurrent](
    initialProducts: List[Product]
  ): F[CartStore[F]] =
    for
      state    <- SignallingRef[F, Cart](Cart(Map.empty, 0.0))
      events   <- Topic[F, CartEvent]
      products <- Ref[F].of(initialProducts.map(p => p.id -> p).toMap)
    yield CartStore(state, events, products)
```

### Analytics ด้วย Event Streams

```scala
class CartAnalytics[F[_]: Concurrent: Temporal](store: CartStore[F]):
  // Track adds per minute
  def addRatePerMinute: Stream[F, Int] =
    store.eventStream
      .collect { case _: ItemAdded => 1 }
      .groupWithin(Int.MaxValue, 1.minute)
      .map(_.size)
  
  // Track cart value over time  
  def cartValueHistory: Stream[F, (Long, Double)] =
    store.cartStream
      .map(cart => (System.currentTimeMillis(), cart.total))
  
  // Alert เมื่อ cart value สูงกว่า threshold
  def highValueAlert(threshold: Double): Stream[F, String] =
    store.cartStream
      .filter(_.total > threshold)
      .map(cart => s"High value cart! Total: ${cart.total}")
      .changes // emit เฉพาะเมื่อ message เปลี่ยน
  
  // Statistics
  def stats: Stream[F, Map[String, Any]] =
    store.cartStream.map { cart =>
      Map(
        "itemCount" -> cart.items.size,
        "total"     -> cart.total,
        "avgPrice"  -> (if cart.items.isEmpty then 0.0 
                        else cart.total / cart.items.size)
      )
    }
```

### Inventory Synchronization

```scala
class InventorySync[F[_]: Concurrent: Temporal](
  store: CartStore[F],
  inventoryRef: Ref[F, Map[String, Int]]
):
  // Sync inventory เมื่อ items เพิ่มใน cart
  def startSync: F[Unit] =
    store.eventStream
      .collect { 
        case ItemAdded(productId, qty) => (productId, -qty)  // reduce stock
        case ItemRemoved(productId)    => (productId, 0)     // restore needs special handling
      }
      .evalMap { case (productId, delta) =>
        if delta < 0 then
          inventoryRef.update { inv =>
            inv.updatedWith(productId)(_.map(_ + delta))
          }
        else
          Concurrent[F].unit
      }
      .compile.drain
  
  // Periodic inventory check
  def periodicCheck(interval: FiniteDuration): Stream[F, List[String]] =
    Stream.awakeEvery[F](interval)
      .evalMap { _ =>
        inventoryRef.get.map { inv =>
          inv.filter(_._2 < 5).keys.toList // low stock items
        }
      }
      .filter(_.nonEmpty)
```

### Application Runner

```scala
object ShoppingApp extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    val initialProducts = List(
      Product("p1", "Laptop", 999.99, 10),
      Product("p2", "Mouse", 29.99, 50),
      Product("p3", "Keyboard", 79.99, 30)
    )
    
    CartStore.create[IO](initialProducts).flatMap { store =>
      val analytics = CartAnalytics[IO](store)
      
      // Start analytics in background
      val monitorCart = store.cartStream
        .evalMap(cart => IO.println(s"Cart updated: ${cart.items.size} items, total: $${cart.total}"))
        .compile.drain
      
      val highValueAlerts = analytics.highValueAlert(500.0)
        .evalMap(alert => IO.println(s"ALERT: $alert"))
        .compile.drain
      
      val program = for
        _ <- IO.println("Starting shopping system...")
        
        // Start background processes
        _ <- monitorCart.start
        _ <- highValueAlerts.start
        
        // Simulate shopping
        _ <- store.addItem("p1", 1)  // Add laptop
        _ <- IO.sleep(100.milliseconds)
        _ <- store.addItem("p2", 2)  // Add 2 mice
        _ <- IO.sleep(100.milliseconds)
        _ <- store.addItem("p3", 1)  // Add keyboard
        _ <- IO.sleep(100.milliseconds)
        _ <- store.removeItem("p2") // Remove mice
        _ <- IO.sleep(100.milliseconds)
        
        cart <- store.currentCart
        _ <- IO.println(s"Final cart: $cart")
      yield ()
      
      program.as(ExitCode.Success)
    }
```

### Testing Concurrent Code

```scala
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.matchers.should.Matchers
import org.scalatest.freespec.AsyncFreeSpec

class CartStoreSpec extends AsyncFreeSpec with AsyncIOSpec with Matchers:
  val testProducts = List(
    Product("p1", "Widget", 10.0, 100),
    Product("p2", "Gadget", 20.0, 50)
  )
  
  "CartStore" - {
    "should add items correctly" in {
      CartStore.create[IO](testProducts).flatMap { store =>
        for
          result <- store.addItem("p1", 2)
          cart   <- store.currentCart
        yield {
          result shouldBe Right(())
          cart.items("p1").quantity shouldBe 2
          cart.total shouldBe 20.0
        }
      }
    }
    
    "should reject items with insufficient stock" in {
      CartStore.create[IO](testProducts).flatMap { store =>
        store.addItem("p2", 100).map { result =>
          result shouldBe Left("Insufficient stock")
        }
      }
    }
    
    "should handle concurrent adds safely" in {
      CartStore.create[IO](testProducts).flatMap { store =>
        // รัน 10 concurrent adds
        (1 to 10).toList.parTraverse { _ =>
          store.addItem("p1", 1)
        }.flatMap { _ =>
          store.currentCart.map { cart =>
            cart.items.get("p1").map(_.quantity) shouldBe Some(10)
          }
        }
      }
    }
    
    "should emit events on changes" in {
      CartStore.create[IO](testProducts).flatMap { store =>
        for
          events <- Ref[IO].of(List.empty[CartEvent])
          
          // Start collecting events
          _ <- store.eventStream
            .take(2)
            .evalMap(e => events.update(e :: _))
            .compile.drain
            .start
          
          _ <- IO.sleep(50.milliseconds)
          _ <- store.addItem("p1", 1)
          _ <- store.addItem("p2", 1)
          _ <- IO.sleep(100.milliseconds)
          
          collected <- events.get
        yield collected.length shouldBe 2
      }
    }
  }
```

---

## สรุป

| Structure | Use Case | Semantics |
|-----------|----------|-----------|
| `Ref[F, A]` | Thread-safe mutable state | Lock-free, CAS-based |
| `MVar[F, A]` | Synchronization point | Blocking put/take |
| `Queue[F, A]` | Producer-consumer | FIFO, optional bounded |
| `Topic[F, A]` | Pub-sub fan-out | All subscribers get all messages |
| `SignallingRef[F, A]` | Reactive state | Discrete change notifications |

### เลือกใช้อย่างไร

- **Ref**: เหมาะสำหรับ state ที่หลาย fiber ต้องการ update พร้อมกัน
- **MVar**: เหมาะสำหรับ handoff ระหว่าง producer และ consumer
- **Queue (Bounded)**: เหมาะสำหรับ back-pressure และ rate limiting
- **Queue (Unbounded)**: เหมาะสำหรับ buffering เมื่อรู้ว่า producer ไม่เร็วเกินไป
- **Topic**: เหมาะสำหรับ event broadcasting
- **SignallingRef**: เหมาะสำหรับ reactive state ที่ต้องการ observe changes

---

*[← Part 95: Advanced Types](part-95-advanced-types.md) | [Part 97: Final Project REST API →](part-97-project-final-api.md)*
