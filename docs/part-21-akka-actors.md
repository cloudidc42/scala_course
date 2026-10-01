# Part 21: Akka Actors

## สารบัญ
1. [Actor Model](#actor-model)
2. [Classic Actors](#classic-actors)
3. [Typed Actors (Akka Typed)](#typed-actors-akka-typed)
4. [Actor Supervision](#actor-supervision)
5. [Actor Communication Patterns](#actor-communication-patterns)

---

## Actor Model

### แนวคิด Actor Model

```
Actor Model ใช้แนวคิด:
- Actors เป็น units of computation
- Communicate ด้วย messages (immutable)
- ไม่ share state กัน
- เป็น concurrent โดยธรรมชาติ

ประโยชน์:
- No shared mutable state → ไม่มี race conditions
- Location transparency (local หรือ remote เหมือนกัน)
- Fault tolerance ด้วย supervision
- Scalability: millions of lightweight actors
```

### SBT Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.typesafe.akka" %% "akka-actor-typed"   % "2.8.0",
  "com.typesafe.akka" %% "akka-stream"         % "2.8.0",
  "com.typesafe.akka" %% "akka-actor-testkit-typed" % "2.8.0" % Test,
  "ch.qos.logback"    %  "logback-classic"     % "1.4.7"
)
```

---

## Classic Actors

### Basic Actor

```scala
import akka.actor.{Actor, ActorRef, ActorSystem, Props}

// Messages (immutable!)
case class Greet(name: String)
case class Greeted(name: String, from: ActorRef)

// Actor
class GreeterActor extends Actor:
  def receive: Receive =
    case Greet(name) =>
      println(s"Hello, $name!")
      sender() ! Greeted(name, self)

class PrinterActor extends Actor:
  def receive: Receive =
    case Greeted(name, from) =>
      println(s"$name has been greeted by $from")

// System
val system = ActorSystem("hello-world")
val greeter = system.actorOf(Props[GreeterActor](), "greeter")
val printer = system.actorOf(Props[PrinterActor](), "printer")

// กำหนด receiver เป็น printer
greeter.tell(Greet("Alice"), printer)
greeter.tell(Greet("Bob"), printer)

Thread.sleep(100)
system.terminate()
```

### Stateful Actor

```scala
import akka.actor.{Actor, ActorSystem, Props}

sealed trait BankMessage
case class Deposit(amount: Double) extends BankMessage
case class Withdraw(amount: Double) extends BankMessage
case object GetBalance extends BankMessage
case class Balance(amount: Double)

class BankAccountActor(initialBalance: Double) extends Actor:
  private var balance = initialBalance

  def receive: Receive =
    case Deposit(amount) =>
      balance += amount
      println(s"Deposited $amount. Balance: $balance")

    case Withdraw(amount) =>
      if amount > balance
      then println(s"Insufficient funds. Balance: $balance")
      else
        balance -= amount
        println(s"Withdrew $amount. Balance: $balance")

    case GetBalance =>
      sender() ! Balance(balance)

val system = ActorSystem("bank")
val account = system.actorOf(
  Props(classOf[BankAccountActor], 1000.0),
  "account"
)

account ! Deposit(500.0)
account ! Withdraw(200.0)
account ! Withdraw(2000.0)  // Insufficient funds

Thread.sleep(100)
system.terminate()
```

---

## Typed Actors (Akka Typed)

### Basic Typed Actor

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*

// Protocol: sealed hierarchy สำหรับ type safety
sealed trait GreeterCommand
final case class Greet(name: String, replyTo: ActorRef[Greeted]) extends GreeterCommand
final case class Greeted(name: String, from: ActorRef[GreeterCommand])

// Behavior definition
object Greeter:
  def apply(): Behavior[GreeterCommand] =
    Behaviors.receiveMessage {
      case Greet(name, replyTo) =>
        println(s"Hello, $name!")
        replyTo ! Greeted(name, ???)  // need self reference
        Behaviors.same
    }

  // Better: using receive with context
  def behavior: Behavior[GreeterCommand] =
    Behaviors.receive { (context, message) =>
      message match
        case Greet(name, replyTo) =>
          context.log.info(s"Hello, $name!")
          replyTo ! Greeted(name, context.self)
          Behaviors.same
    }

// Printer actor
object GreetPrinter:
  def apply(): Behavior[Greeted] =
    Behaviors.receiveMessage { greeted =>
      println(s"${greeted.name} was greeted by ${greeted.from}")
      Behaviors.same
    }

// Main guardian
object Main:
  def apply(): Behavior[NotUsed] =
    Behaviors.setup { context =>
      val greeter = context.spawn(Greeter.behavior, "greeter")
      val printer = context.spawn(GreetPrinter(), "printer")

      greeter ! Greet("Alice", printer)
      greeter ! Greet("Bob", printer)

      Behaviors.same
    }

// Start system
val system = ActorSystem(Main(), "hello-world")
Thread.sleep(100)
system.terminate()
```

### Typed Actor กับ State

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*

sealed trait CounterCommand
case object Increment extends CounterCommand
case object Decrement extends CounterCommand
case class GetCount(replyTo: ActorRef[Int]) extends CounterCommand
case class Reset(replyTo: ActorRef[Unit]) extends CounterCommand

object Counter:
  def apply(initial: Int = 0): Behavior[CounterCommand] =
    behavior(initial)

  private def behavior(count: Int): Behavior[CounterCommand] =
    Behaviors.receiveMessage {
      case Increment =>
        behavior(count + 1)  // return new behavior with new state!

      case Decrement =>
        behavior(count - 1)

      case GetCount(replyTo) =>
        replyTo ! count
        Behaviors.same

      case Reset(replyTo) =>
        replyTo ! ()
        behavior(0)
    }

// ใช้งาน
import akka.actor.typed.scaladsl.AskPattern.*
import scala.concurrent.duration.*

given ActorSystem[?] = ActorSystem(Counter(0), "counter-system")
given Timeout = Timeout(3.seconds)

given(system)
val counter = system

// Note: ในที่นี้ system IS the counter actor
// แต่ใน real app จะ spawn actors แยก
```

---

## Actor Supervision

### Supervision Strategies

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*

// Child actor ที่อาจ fail
object Worker:
  sealed trait Command
  case class Process(data: String) extends Command

  def apply(): Behavior[Command] =
    Behaviors.receive { (ctx, msg) =>
      msg match
        case Process(data) if data == "bad" =>
          throw new RuntimeException("Bad data!")
        case Process(data) =>
          ctx.log.info(s"Processing: $data")
          Behaviors.same
    }

// Supervisor actor
object Supervisor:
  def apply(): Behavior[String] =
    Behaviors.setup { context =>
      // Supervision strategy
      val worker = context.spawn(
        Behaviors.supervise(Worker())
          .onFailure[RuntimeException](SupervisorStrategy.restart),
        "worker"
      )

      Behaviors.receiveMessage { msg =>
        worker ! Worker.Process(msg)
        Behaviors.same
      }
    }

// Supervision strategies:
// - SupervisorStrategy.restart: restart actor ใหม่
// - SupervisorStrategy.stop: stop actor
// - SupervisorStrategy.resume: continue (ignore error)
// - SupervisorStrategy.restart.withLimit(maxNrOfRetries, withinTimeRange)
```

---

## Actor Communication Patterns

### Ask Pattern

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.actor.typed.scaladsl.AskPattern.*
import scala.concurrent.*
import scala.concurrent.duration.*

object Calculator:
  sealed trait Command
  case class Add(a: Int, b: Int, replyTo: ActorRef[Int]) extends Command
  case class Multiply(a: Int, b: Int, replyTo: ActorRef[Int]) extends Command

  def apply(): Behavior[Command] =
    Behaviors.receiveMessage {
      case Add(a, b, replyTo) =>
        replyTo ! (a + b)
        Behaviors.same
      case Multiply(a, b, replyTo) =>
        replyTo ! (a * b)
        Behaviors.same
    }

val system = ActorSystem(Calculator(), "calc")
given ActorSystem[?] = system
given Timeout = Timeout(3.seconds)

val sumFuture: Future[Int] = system.ask(replyTo => Calculator.Add(3, 4, replyTo))
println(Await.result(sumFuture, 5.seconds))  // 7

system.terminate()
```

### Router Pattern

```scala
import akka.actor.typed.*
import akka.actor.typed.scaladsl.*
import akka.actor.typed.scaladsl.Routers

object Worker2:
  def apply(): Behavior[String] =
    Behaviors.receive { (ctx, msg) =>
      ctx.log.info(s"Worker ${ctx.self.path.name} processing: $msg")
      Behaviors.same
    }

object RouterExample:
  def apply(): Behavior[String] =
    Behaviors.setup { ctx =>
      // Pool of 5 workers (round-robin)
      val pool = Routers.pool(5)(Behaviors.supervise(Worker2())
        .onFailure(SupervisorStrategy.restart))
      val router = ctx.spawn(pool, "worker-pool")

      Behaviors.receiveMessage { msg =>
        router ! msg
        Behaviors.same
      }
    }
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Actor Model concepts
- ✅ Classic Actors (Akka Classic)
- ✅ Typed Actors (Akka Typed) - แนะนำสำหรับ modern Scala
- ✅ Supervision strategies
- ✅ Communication patterns: tell, ask, router

---

*[← Part 20: Futures](part-20-futures-concurrency.md) | [Part 22: Akka Streams →](part-22-akka-streams.md)*
