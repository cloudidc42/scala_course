# Part 64: Monix

## สารบัญ
1. [Monix คืออะไร](#monix-คืออะไร)
2. [Task vs IO vs Observable](#task-vs-io-vs-observable)
3. [Observable: Hot vs Cold Streams](#observable-hot-vs-cold-streams)
4. [Parallelism กับ Task](#parallelism-กับ-task)
5. [Integration กับ Akka](#integration-กับ-akka)
6. [CancelableFuture](#cancelablefuture)
7. [Monix Reactive: Advanced Operators](#monix-reactive-advanced-operators)
8. [ตัวอย่าง Monix Application ครบถ้วน](#ตัวอย่าง-monix-application-ครบถ้วน)
9. [สรุป](#สรุป)

---

## Monix คืออะไร

Monix เป็น Scala/Scala.js library สำหรับ asynchronous programming และ event-based reactive programming ที่พัฒนาโดย Alexandru Nedelcu โดย Monix มี 3 ส่วนหลัก:

```
Monix Modules:
┌─────────────────────────────────────────────────┐
│                   Monix                         │
│  ┌──────────────┐  ┌────────────────────────┐  │
│  │ monix-eval   │  │   monix-reactive       │  │
│  │              │  │                        │  │
│  │  Task[A]     │  │  Observable[A]         │  │
│  │  Coeval[A]   │  │  Observer[A]           │  │
│  └──────────────┘  └────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │         monix-execution                  │  │
│  │  Scheduler, Cancelable, CancelableFuture │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### ติดตั้ง Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.monix" %% "monix"          % "3.4.1",
  // หรือแยก modules:
  "io.monix" %% "monix-eval"     % "3.4.1",
  "io.monix" %% "monix-reactive" % "3.4.1",
  // สำหรับ Cats integration
  "io.monix" %% "monix-catnap"   % "3.4.1",
)
```

---

## Task vs IO vs Observable

### Task[A] - Lazy Asynchronous Effect

```scala
import monix.eval.Task
import monix.execution.Scheduler.Implicits.global

// Task คือ lazy description ของ computation
// ไม่รันจนกว่าจะ runToFuture หรือ runSyncUnsafe

val task1: Task[Int] = Task.pure(42)
val task2: Task[Int] = Task.eval(println("computing") ; 42)  // lazy
val task3: Task[Int] = Task.raiseError(new RuntimeException("oops"))
val task4: Task[Int] = Task.defer(Task.pure(100))  // extra lazy

// Composition
val program: Task[String] = for
  x <- Task.eval { println("Step 1"); 10 }
  y <- Task.eval { println("Step 2"); 20 }
  _ <- Task.eval { println("Step 3") }
yield s"Result: ${x + y}"

// การรัน
import scala.concurrent.Await
import scala.concurrent.duration.*

// รันแบบ async (แนะนำ)
val future = program.runToFuture
future.foreach(println)
// Step 1
// Step 2
// Step 3
// Result: 30

// รันแบบ blocking (ใช้เฉพาะ main/test)
val result = program.runSyncUnsafe()
println(result)
```

### Coeval[A] - Lazy Synchronous Effect

```scala
import monix.eval.Coeval

// Coeval คือ synchronous version ของ Task
// ไม่ async, ไม่ต้องการ Scheduler

val coeval1: Coeval[Int] = Coeval.pure(42)
val coeval2: Coeval[Int] = Coeval.eval { 1 + 1 }
val coeval3: Coeval[String] = Coeval.raiseError(new RuntimeException("sync error"))

// Composition
val computation: Coeval[Int] = for
  a <- Coeval.eval(10)
  b <- Coeval.eval(20)
yield a + b

// รันทันที (sync)
val value: Int = computation.value()
// หรือ safe:
val result: Either[Throwable, Int] = computation.runTry().toEither

// Convert Task ↔ Coeval
val asTask: Task[Int]   = computation.to[Task]
val asCoeval: Coeval[Int] = Task.eval(42).to[Coeval]  // blocking!
```

### Observable[A] - Reactive Stream

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global

// Observable สำหรับ stream of values
val obs1: Observable[Int] = Observable(1, 2, 3, 4, 5)
val obs2: Observable[Int] = Observable.fromIterable(1 to 100)
val obs3: Observable[Long] = Observable.interval(1.second)  // tick

// Transformation
val processed: Observable[String] =
  Observable.fromIterable(1 to 10)
    .filter(_ % 2 == 0)
    .map(n => s"Even: $n")
    .take(3)

// Run Observable
processed.foreach(println).runToFuture
// Even: 2
// Even: 4
// Even: 6

// Collect to List
val list: Task[List[String]] =
  processed.toListL

list.runToFuture.foreach(println)
// List(Even: 2, Even: 4, Even: 6)
```

### เปรียบเทียบ 3 Types

```
           Task[A]        Coeval[A]     Observable[A]
           --------        ---------     -------------
Values:    1 value         1 value       0..N values
Mode:      Async           Sync          Async/Sync
Lazy:      Yes             Yes           Yes
Error:     Yes             Yes           Yes
Cancel:    Yes             No            Yes
Use when:  Async I/O       Sync compute  Streams/Events
Similar:   cats IO, ZIO   Eval[A]       FS2, Akka Streams
```

---

## Observable: Hot vs Cold Streams

### Cold Observable

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global

// Cold Observable: แต่ละ subscriber ได้รับ sequence ของตัวเอง
// Producer รันใหม่สำหรับทุก subscriber

val coldStream: Observable[Int] = Observable.fromIterable(1 to 5)

// Subscriber 1
coldStream.foreach(n => println(s"Sub1: $n")).runToFuture

// Subscriber 2 ได้รับ elements ใหม่ทั้งหมด (ไม่ share กับ Sub1)
coldStream.foreach(n => println(s"Sub2: $n")).runToFuture

// Output:
// Sub1: 1, Sub1: 2, Sub1: 3, Sub1: 4, Sub1: 5
// Sub2: 1, Sub2: 2, Sub2: 3, Sub2: 4, Sub2: 5
// (แยกกัน ไม่ได้ share)
```

### Hot Observable ด้วย Subject

```scala
import monix.reactive.subjects.*
import monix.reactive.{Observable, Observer}
import monix.execution.Scheduler.Implicits.global

// PublishSubject: Hot Observable ที่ share กับทุก subscriber
// Subscribers รับเฉพาะ events หลังจาก subscribe
val subject = PublishSubject[Int]()

// Subscribe ก่อน
subject.foreach(n => println(s"Sub1: $n")).runToFuture
subject.foreach(n => println(s"Sub2: $n")).runToFuture

// Emit values
subject.onNext(1)
subject.onNext(2)

// Subscribe ที่หลัง - ไม่ได้รับ 1, 2
subject.foreach(n => println(s"Sub3 (late): $n")).runToFuture

subject.onNext(3)
subject.onComplete()

// Output:
// Sub1: 1, Sub2: 1
// Sub1: 2, Sub2: 2
// Sub1: 3, Sub2: 3, Sub3 (late): 3

// BehaviorSubject: ส่ง last value ให้ subscriber ใหม่
val behavior = BehaviorSubject[Int](0)  // default value = 0
behavior.onNext(10)
behavior.onNext(20)

// subscriber ใหม่จะได้รับ 20 ทันที (last value)
behavior.foreach(n => println(s"New sub: $n")).runToFuture
// New sub: 20
behavior.onNext(30)
// New sub: 30

// ReplaySubject: replay ทุก event ให้ subscriber ใหม่
val replay = ReplaySubject[Int]()
replay.onNext(1)
replay.onNext(2)
replay.onNext(3)

// subscriber ใหม่จะได้รับ 1, 2, 3 ทั้งหมด
replay.foreach(n => println(s"Replay sub: $n")).runToFuture
// Replay sub: 1
// Replay sub: 2
// Replay sub: 3
replay.onNext(4)
// Replay sub: 4
```

### Multicast - แปลง Cold เป็น Hot

```scala
import monix.reactive.Observable
import monix.reactive.subjects.PublishSubject
import monix.execution.Scheduler.Implicits.global

// publish: แปลง cold เป็น ConnectableObservable
val cold = Observable.interval(500.millis).take(5)
val hot = cold.publish  // ConnectableObservable

// Subscribe ก่อน connect
hot.foreach(n => println(s"Sub1: $n")).runToFuture
hot.foreach(n => println(s"Sub2: $n")).runToFuture

// Connect เพื่อเริ่ม emit
hot.connect()

// หลัง connect, ทั้ง Sub1 และ Sub2 รับ elements เดียวกัน

// refCount: auto connect เมื่อมี subscriber แรก
// auto disconnect เมื่อไม่มี subscriber
val shared = cold.publish.refCount
// จะเริ่ม emit เมื่อมี subscriber คนแรก
// จะหยุด emit เมื่อ subscriber สุดท้าย unsubscribe
```

---

## Parallelism กับ Task

### Task.parSequence / Task.parZip

```scala
import monix.eval.Task
import monix.execution.Scheduler.Implicits.global

// Sequential (ช้า)
val sequential: Task[List[String]] =
  Task.sequence(List(
    Task.sleep(1.second).as("result-1"),
    Task.sleep(1.second).as("result-2"),
    Task.sleep(1.second).as("result-3"),
  ))
// ใช้เวลา ~3 วินาที

// Parallel (เร็ว)
val parallel: Task[List[String]] =
  Task.parSequence(List(
    Task.sleep(1.second).as("result-1"),
    Task.sleep(1.second).as("result-2"),
    Task.sleep(1.second).as("result-3"),
  ))
// ใช้เวลา ~1 วินาที
```

### Parallel map และ traverse

```scala
import monix.eval.Task
import monix.execution.Scheduler.Implicits.global
import cats.syntax.parallel.*

// parTraverse: map + parallel
val urls = List("http://api1.com", "http://api2.com", "http://api3.com")

val sequential: Task[List[String]] =
  Task.traverse(urls)(fetchUrl)

val parallel: Task[List[String]] =
  Task.parTraverse(urls)(fetchUrl)

// หรือด้วย Cats syntax
val withCats: Task[List[String]] =
  urls.parTraverse(fetchUrl)

def fetchUrl(url: String): Task[String] =
  Task.sleep(500.millis).as(s"Response from $url")

// parZip2, parZip3, ..., parZip6
val results: Task[(String, Int, Boolean)] =
  Task.parZip3(
    Task.eval("hello"),
    Task.eval(42),
    Task.eval(true)
  )
```

### กำหนด Parallelism Level

```scala
import monix.eval.Task
import monix.execution.Scheduler
import monix.execution.schedulers.TestScheduler

// ใช้ custom scheduler สำหรับ limit parallelism
val ioScheduler = Scheduler.io(name = "io-pool")
val cpuScheduler = Scheduler.computation(parallelism = 4)

// executeOn: รัน task บน specific scheduler
val ioTask: Task[String] =
  Task.eval { Thread.currentThread().getName }
    .executeOn(ioScheduler)

// Bounded parallelism ด้วย Observable
import monix.reactive.Observable

val withBoundedParallelism: Task[List[Int]] =
  Observable.fromIterable(1 to 100)
    .mapParallelUnordered(parallelism = 8) { n =>
      Task.sleep(100.millis).as(n * 2)
    }
    .toListL
```

### Gather - รวม Multiple Tasks

```scala
import monix.eval.{Task, TaskLocal}
import monix.execution.Scheduler.Implicits.global

// gather: เหมือน parSequence แต่ return ผลตามลำดับ
val gathered: Task[Vector[Int]] =
  Task.gather(
    Vector(
      Task.eval(1),
      Task.eval(2),
      Task.eval(3),
    )
  )

// gatherUnordered: faster, ผลอาจไม่เรียงลำดับ
val unordered: Task[List[Int]] =
  Task.gatherUnordered(
    List(
      Task.sleep(300.millis).as(1),
      Task.sleep(100.millis).as(2),
      Task.sleep(200.millis).as(3),
    )
  )
// อาจได้ List(2, 3, 1) เพราะ task 2 เสร็จก่อน

// race: เอาผลของ task ที่เสร็จก่อน cancel ที่เหลือ
val fastest: Task[Either[String, Int]] =
  Task.race(
    Task.sleep(200.millis).as("slow"),
    Task.sleep(100.millis).as(42)
  )
// Left("slow") or Right(42) แล้วแต่ว่า task ไหนเร็วกว่า
// จะได้ Right(42) เสมอเพราะ sleep น้อยกว่า
```

---

## Integration กับ Akka

### Monix Task กับ Akka Ask Pattern

```scala
import monix.eval.Task
import monix.execution.Scheduler
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.util.Timeout
import scala.concurrent.duration.*

// แปลง Akka Future เป็น Monix Task
def askActor[A](
  actor: ActorRef[?],
  msg: ActorRef[A] => Any
)(using system: ActorSystem[?], timeout: Timeout): Task[A] =
  val future = akka.actor.typed.scaladsl.AskPattern
    .ask(actor.unsafeUpcast[Any], msg)(timeout, system.scheduler)
  Task.fromFuture(future)

// ตัวอย่างการใช้งาน
object ActorIntegration:
  given timeout: Timeout = 5.seconds

  def program(using system: ActorSystem[?])(using scheduler: Scheduler): Task[Unit] =
    for
      calcActor <- Task.eval {
        system.systemActorOf(Calculator(), "calc")
      }
      result1 <- askActor[Calculator.Result](
        calcActor,
        ref => Calculator.Calculate("add", 3.0, 4.0, ref)
      )
      result2 <- askActor[Calculator.Result](
        calcActor,
        ref => Calculator.Calculate("multiply", 5.0, 6.0, ref)
      )
      _ <- Task.eval {
        println(s"3 + 4 = ${result1.value}")
        println(s"5 * 6 = ${result2.value}")
      }
    yield ()
```

### Akka Streams ↔ Monix Observable

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global
import akka.stream.scaladsl.*
import akka.stream.*
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors

given system: ActorSystem[Nothing] = ActorSystem(Behaviors.empty, "akka-monix")
given materializer: Materializer = Materializer(system)

// Akka Source → Monix Observable
def akkaSourceToObservable[A](source: Source[A, ?]): Observable[A] =
  Observable.fromReactivePublisher(source.runWith(Sink.asPublisher(fanout = false)))

// Monix Observable → Akka Source
def observableToAkkaSource[A](obs: Observable[A]): Source[A, NotUsed] =
  import org.reactivestreams.Publisher
  val publisher: Publisher[A] = obs.toReactivePublisher
  Source.fromPublisher(publisher)

// ตัวอย่าง
val akkaSource = Source(1 to 10)
val monixObs = akkaSourceToObservable(akkaSource)
monixObs.map(_ * 2).foreach(println).runToFuture
// 2, 4, 6, ..., 20

val monixStream = Observable.fromIterable(100 to 110)
val akkaStream = observableToAkkaSource(monixStream)
akkaStream.runForeach(println)
// 100, 101, ..., 110
```

---

## CancelableFuture

### CancelableFuture คืออะไร

```scala
import monix.eval.Task
import monix.execution.{CancelableFuture, Scheduler}
import monix.execution.Scheduler.Implicits.global

// Task.runToFuture ให้ CancelableFuture
val task = Task.sleep(5.seconds).as("done")
val cf: CancelableFuture[String] = task.runToFuture

// Cancel ได้!
Thread.sleep(1000)
cf.cancel()  // หยุด task ก่อน 5 วินาที

// CancelableFuture ทำงานเหมือน Future ทั่วไป
cf.foreach(println)  // จะไม่ print อะไรเพราะถูก cancel
cf.onComplete {
  case scala.util.Success(v)  => println(s"Success: $v")
  case scala.util.Failure(ex) => println(s"Failed: ${ex.getMessage}")
}
```

### การใช้งาน Cancel ใน practice

```scala
import monix.eval.Task
import monix.execution.{CancelableFuture, Cancelable, Scheduler}
import monix.execution.Scheduler.Implicits.global
import scala.concurrent.duration.*

// Resource management ด้วย bracket
val withCleanup: Task[String] =
  Task.bracket(
    acquire = Task.eval {
      println("Acquiring resource")
      "database-connection"
    }
  )(
    use = conn => Task.sleep(3.seconds).as(s"Result from $conn")
  )(
    release = conn => Task.eval(println(s"Releasing $conn"))
  )

val cf = withCleanup.runToFuture
Thread.sleep(1000)
cf.cancel()
// Output:
// Acquiring resource
// Releasing database-connection  ← cleanup รันแม้จะ cancel!

// Guarantee: รัน cleanup เสมอ ไม่ว่าจะ success, fail, หรือ cancel
val withGuarantee: Task[String] =
  Task.sleep(5.seconds).as("result")
    .guaranteeCase {
      case monix.eval.ExitCase.Completed =>
        Task.eval(println("Completed successfully"))
      case monix.eval.ExitCase.Error(ex) =>
        Task.eval(println(s"Failed: $ex"))
      case monix.eval.ExitCase.Canceled =>
        Task.eval(println("Was canceled"))
    }
```

### Cancelable สำหรับ Custom Resources

```scala
import monix.execution.{Cancelable, Scheduler}
import monix.execution.Scheduler.Implicits.global
import monix.reactive.Observable
import scala.concurrent.duration.*

// สร้าง Observable ที่ cancel cleanup ได้
def timerObservable(interval: FiniteDuration): Observable[Long] =
  Observable.create[Long] { subscriber =>
    val timer = new java.util.Timer()
    var count = 0L

    val task = new java.util.TimerTask:
      def run(): Unit =
        subscriber.onNext(count)
        count += 1

    timer.scheduleAtFixedRate(task, 0L, interval.toMillis)

    // Return Cancelable ที่จะ cancel timer
    Cancelable { () =>
      task.cancel()
      timer.cancel()
      println("Timer cancelled!")
    }
  }

// ใช้งาน
val obs = timerObservable(500.millis)
val cf = obs.take(5).foreach(n => println(s"Tick: $n")).runToFuture

Thread.sleep(3000)
cf.cancel()
// Tick: 0
// Tick: 1
// ...
// Timer cancelled!
```

---

## Monix Reactive: Advanced Operators

### Window และ Buffer

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global
import scala.concurrent.duration.*

// buffer by count
val buffered: Observable[Seq[Int]] =
  Observable.fromIterable(1 to 20).bufferTumbling(5)
// List(1,2,3,4,5), List(6,7,8,9,10), ...

// buffer by time
val timedBuffer: Observable[Seq[Long]] =
  Observable.interval(100.millis).bufferTimed(500.millis)
// แต่ละ buffer มี elements ที่ emit ใน 500ms

// sliding window
val sliding: Observable[Seq[Int]] =
  Observable.fromIterable(1 to 10).bufferSliding(count = 3, skip = 1)
// List(1,2,3), List(2,3,4), List(3,4,5), ...

// window (returns Observable of Observables)
val windowed: Observable[Observable[Int]] =
  Observable.fromIterable(1 to 20).window(count = 5)
```

### Operators สำหรับ Rate Control

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global
import scala.concurrent.duration.*

// throttleFirst: เก็บ element แรกใน window
val throttled: Observable[Int] =
  Observable.interval(100.millis)
    .map(_.toInt)
    .throttleFirst(500.millis)

// debounce: รอให้หยุด emit แล้วค่อย emit ค่าสุดท้าย
// (เหมาะกับ text input search)
val debounced: Observable[String] =
  Observable("h", "he", "hel", "hell", "hello")
    .delayElements(50.millis)
    .debounceTime(200.millis)
// จะได้แค่ "hello" (ค่าสุดท้ายหลังหยุด type)

// sample: emit element ล่าสุดในแต่ละ time window
val sampled: Observable[Int] =
  Observable.interval(100.millis)
    .map(_.toInt)
    .sample(300.millis)
// emit ~ทุก 300ms แทนที่จะ 100ms
```

### Combining Observables

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global

// merge: emit เมื่อไหนก็ได้ (unordered)
val merged = Observable.mergeDelayErrors(
  Observable(1, 2, 3),
  Observable(4, 5, 6),
  Observable(7, 8, 9)
)

// concat: emit ตามลำดับ (wait for first to complete)
val concatenated = Observable.concat(
  Observable(1, 2, 3),
  Observable(4, 5, 6)
)
// 1, 2, 3, 4, 5, 6 ตามลำดับ

// zip: จับคู่ element จาก 2 observables
val zipped = Observable.zip2(
  Observable(1, 2, 3),
  Observable("a", "b", "c")
)
// (1,"a"), (2,"b"), (3,"c")

// combineLatest: emit เมื่อไหนก็ได้ที่มีค่าใหม่ + ค่าล่าสุดของอีก observable
val combined = Observable.combineLatest2(
  Observable(1, 2, 3).delayElements(100.millis),
  Observable("x", "y", "z").delayElements(150.millis)
)
// (1,"x"), (2,"x"), (2,"y"), (3,"y"), (3,"z")

// switchMap: cancel previous และ subscribe ใหม่
val switchMapped: Observable[String] =
  Observable.interval(300.millis)
    .switchMap { n =>
      // ทุกครั้งที่มี value ใหม่, cancel previous inner observable
      Observable.interval(100.millis).take(5).map(i => s"$n-$i")
    }
```

### Error Handling ใน Observable

```scala
import monix.reactive.Observable
import monix.execution.Scheduler.Implicits.global

val flakyStream = Observable.fromIterable(1 to 10).map { n =>
  if n == 5 then throw new RuntimeException("Error at 5!")
  else n
}

// onErrorResumeNext: ต่อด้วย Observable อื่นเมื่อ error
val withFallback = flakyStream.onErrorResumeNext { ex =>
  println(s"Caught: ${ex.getMessage}, switching to fallback")
  Observable(-1, -2, -3)
}
// 1, 2, 3, 4, Caught: Error at 5!, -1, -2, -3

// onErrorReturn: ส่งค่า default เมื่อ error
val withDefault = flakyStream.onErrorReturn(ex => -99)
// 1, 2, 3, 4, -99 (และจบ)

// retry: ลอง subscribe ใหม่เมื่อ error
var attempts = 0
val retried = Observable.create[Int] { sub =>
  attempts += 1
  if attempts < 3 then
    sub.onError(new RuntimeException(s"Attempt $attempts failed"))
  else
    sub.onNext(42)
    sub.onComplete()
  monix.execution.Cancelable.empty
}.retry(3)

retried.foreach(println).runToFuture
// (ลองใหม่ 3 ครั้ง แล้วได้ 42)
```

---

## ตัวอย่าง Monix Application ครบถ้วน

### Real-time Data Processing Pipeline

```scala
package com.example.monix

import monix.eval.Task
import monix.reactive.{Observable, Observer}
import monix.reactive.subjects.ConcurrentSubject
import monix.execution.Scheduler
import scala.concurrent.duration.*
import java.time.Instant

// Domain
case class SensorReading(
  deviceId: String,
  sensor: String,
  value: Double,
  timestamp: Instant = Instant.now()
)

case class Alert(
  deviceId: String,
  sensor: String,
  message: String,
  severity: String,
  readings: List[Double]
)

case class AggregatedStats(
  deviceId: String,
  sensor: String,
  min: Double,
  max: Double,
  avg: Double,
  count: Int,
  windowSeconds: Int
)

// Alert Rules
object AlertRules:
  def checkTemperature(readings: Seq[SensorReading]): Option[Alert] =
    val values = readings.map(_.value)
    val avg = if values.isEmpty then 0.0 else values.sum / values.size
    if avg > 80.0 then
      Some(Alert(
        readings.head.deviceId,
        "temperature",
        f"High temperature average: $avg%.1f°C",
        "critical",
        values.toList
      ))
    else if avg > 60.0 then
      Some(Alert(
        readings.head.deviceId,
        "temperature",
        f"Elevated temperature: $avg%.1f°C",
        "warning",
        values.toList
      ))
    else None

  def checkHumidity(readings: Seq[SensorReading]): Option[Alert] =
    val values = readings.map(_.value)
    val avg = if values.isEmpty then 0.0 else values.sum / values.size
    if avg > 90.0 then
      Some(Alert(
        readings.head.deviceId,
        "humidity",
        f"Critically high humidity: $avg%.1f%%",
        "critical",
        values.toList
      ))
    else None

// Data Pipeline
class SensorPipeline(using scheduler: Scheduler):
  // Hot subject รับ readings จากภายนอก
  private val source = ConcurrentSubject.publish[SensorReading]

  // Alert stream
  val alerts: Observable[Alert] =
    source
      .groupBy(r => (r.deviceId, r.sensor))
      .mergeMap { case (grouped, groupedObs) =>
        groupedObs
          .bufferTimed(10.seconds)
          .filter(_.nonEmpty)
          .flatMap { batch =>
            val alerts = batch.head.sensor match
              case "temperature" => AlertRules.checkTemperature(batch).toList
              case "humidity"    => AlertRules.checkHumidity(batch).toList
              case _             => Nil
            Observable.fromIterable(alerts)
          }
      }

  // Stats stream
  val stats: Observable[AggregatedStats] =
    source
      .groupBy(r => (r.deviceId, r.sensor))
      .mergeMap { case (grouped, groupedObs) =>
        groupedObs
          .bufferTimed(30.seconds)
          .filter(_.nonEmpty)
          .map { batch =>
            val values = batch.map(_.value)
            AggregatedStats(
              deviceId = batch.head.deviceId,
              sensor   = batch.head.sensor,
              min      = values.min,
              max      = values.max,
              avg      = values.sum / values.size,
              count    = values.size,
              windowSeconds = 30
            )
          }
      }

  // Inject reading
  def ingest(reading: SensorReading): Task[Unit] =
    Task.fromFuture(source.onNext(reading).map(_ => ()))

  def complete(): Unit = source.onComplete()

// Alert Store
class AlertStore:
  private var alerts = List.empty[Alert]

  def add(alert: Alert): Task[Unit] =
    Task.eval {
      alerts = alert :: alerts
      println(s"[ALERT] ${alert.severity.toUpperCase}: ${alert.message} (device: ${alert.deviceId})")
    }

  def getAll(): Task[List[Alert]] = Task.eval(alerts.reverse)
  def getBySeverity(severity: String): Task[List[Alert]] =
    Task.eval(alerts.filter(_.severity == severity).reverse)

// Stats Store
class StatsStore:
  private var stats = Map.empty[(String, String), AggregatedStats]

  def update(s: AggregatedStats): Task[Unit] =
    Task.eval {
      stats = stats + ((s.deviceId, s.sensor) -> s)
      println(f"[STATS] ${s.deviceId}/${s.sensor}: min=${s.min}%.1f max=${s.max}%.1f avg=${s.avg}%.1f (n=${s.count})")
    }

  def getAll(): Task[Map[(String, String), AggregatedStats]] =
    Task.eval(stats)

// Simulated sensors
object SensorSimulator:
  def generateReadings(deviceId: String): Observable[SensorReading] =
    Observable.interval(200.millis)
      .map { tick =>
        val sensor = if tick % 2 == 0 then "temperature" else "humidity"
        val value = sensor match
          case "temperature" =>
            60.0 + scala.util.Random.nextDouble() * 30  // 60-90°C
          case "humidity" =>
            70.0 + scala.util.Random.nextDouble() * 25  // 70-95%
          case _ => 0.0
        SensorReading(deviceId, sensor, value)
      }
      .take(50)

// Main application
@main def runSensorPipeline(): Unit =
  import monix.execution.Scheduler.Implicits.global

  val pipeline = SensorPipeline()
  val alertStore = AlertStore()
  val statsStore = StatsStore()

  val program: Task[Unit] = for
    // Subscribe to alerts
    alertFiber <- pipeline.alerts
                    .mapEval(alertStore.add)
                    .completedL
                    .start

    // Subscribe to stats
    statsFiber <- pipeline.stats
                    .mapEval(statsStore.update)
                    .completedL
                    .start

    // Simulate 3 devices
    _ <- Task.parSequence(List(
      SensorSimulator.generateReadings("device-001")
        .mapEval(pipeline.ingest)
        .completedL,
      SensorSimulator.generateReadings("device-002")
        .mapEval(pipeline.ingest)
        .completedL,
      SensorSimulator.generateReadings("device-003")
        .mapEval(pipeline.ingest)
        .completedL
    ))

    _ <- Task.eval(pipeline.complete())
    _ <- Task.sleep(2.seconds)  // รอ process ที่เหลือ
    _ <- alertFiber.cancel
    _ <- statsFiber.cancel

    // Report
    allAlerts <- alertStore.getAll()
    critical  <- alertStore.getBySeverity("critical")
    allStats  <- statsStore.getAll()

    _ <- Task.eval {
      println("\n=== Final Report ===")
      println(s"Total alerts: ${allAlerts.size}")
      println(s"Critical alerts: ${critical.size}")
      println(s"\nLatest Stats:")
      allStats.foreach { case ((device, sensor), stat) =>
        println(f"  $device/$sensor: avg=${stat.avg}%.1f")
      }
    }
  yield ()

  program.runSyncUnsafe(60.seconds)
```

---

## สรุป

Monix เป็น library ที่ทรงพลังสำหรับ:

- ✅ **Task**: Async effect ที่ composable, cancelable, และ lazy
- ✅ **Observable**: Reactive stream ที่ handle backpressure และ hot/cold streams
- ✅ **Parallelism**: Task.parSequence, parTraverse สำหรับ parallel execution
- ✅ **Integration**: ทำงานร่วมกับ Akka, Future, Cats Effect ได้ดี
- ✅ **Cancellation**: CancelableFuture และ Cancelable resource management

### เปรียบเทียบกับ alternatives

```
Feature          | Monix Task     | ZIO Task      | Cats IO
-----------------|----------------|---------------|--------
Cancellation     | ✅ First-class  | ✅ Fiber       | ✅ Fiber
Error Typing     | ✗ Throwable   | ✅ Typed       | ✗ Throwable
Environment      | ✗              | ✅ ZLayer      | ✗
Streams          | Observable     | ZStream        | FS2
Akka Integration | ✅ Good         | ✅ Good         | ✅ Good
Scala.js         | ✅ Yes          | ✅ Yes          | ✅ Yes
Learning Curve   | Medium         | Medium         | Higher
```

Monix เหมาะสำหรับ:
- Applications ที่ต้องการ reactive streams (IoT, real-time data)
- Gradual migration จาก Future-based code
- Projects ที่ต้องการ Scala.js compatibility
- Teams ที่คุ้นเคยกับ RxJava/Reactive Extensions

---

*[← Part 63: ZIO](part-63-zio.md) | [Part 65: Advanced Topics →](part-65-advanced.md)*
