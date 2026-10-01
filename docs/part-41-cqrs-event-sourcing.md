# Part 41: CQRS และ Event Sourcing

## สารบัญ
1. [CQRS Overview](#cqrs-overview)
2. [Event Sourcing Basics](#event-sourcing-basics)
3. [Aggregate Pattern](#aggregate-pattern)
4. [Event Store](#event-store)
5. [Read Model (Projections)](#read-model)
6. [Complete Example](#complete-example)

---

## CQRS Overview

### Command Query Responsibility Segregation

```
CQRS: แยก write model (Commands) ออกจาก read model (Queries)

Write Side:
- Commands: ความตั้งใจที่จะเปลี่ยนแปลงสถานะ
- Aggregates: business objects ที่ enforce invariants
- Events: สิ่งที่เกิดขึ้นแล้ว (fact)

Read Side:
- Queries: อ่านข้อมูลสำหรับ display
- Projections: สร้าง read models จาก events
- Denormalized views สำหรับ performance

Benefits:
- Scale read/write independently
- Simpler queries (denormalized)
- Full audit trail
- Time-travel: rebuild state at any point in time
```

---

## Event Sourcing Basics

### Events เป็น Source of Truth

```scala
// Events: อธิบายสิ่งที่เกิดขึ้น (ไม่ใช่คำสั่ง)
sealed trait BankEvent
object BankEvent:
  case class AccountOpened(
    accountId: String,
    customerId: String,
    initialBalance: BigDecimal,
    openedAt: java.time.Instant
  ) extends BankEvent

  case class MoneyDeposited(
    accountId: String,
    amount: BigDecimal,
    description: String,
    depositedAt: java.time.Instant
  ) extends BankEvent

  case class MoneyWithdrawn(
    accountId: String,
    amount: BigDecimal,
    description: String,
    withdrawnAt: java.time.Instant
  ) extends BankEvent

  case class AccountClosed(
    accountId: String,
    reason: String,
    closedAt: java.time.Instant
  ) extends BankEvent

// State: built by folding over events
case class BankAccountState(
  accountId: String,
  customerId: String,
  balance: BigDecimal,
  isOpen: Boolean,
  transactions: List[String]
)

object BankAccountState:
  def empty: BankAccountState =
    BankAccountState("", "", BigDecimal(0), isOpen = false, Nil)

  def apply(events: List[BankEvent]): BankAccountState =
    events.foldLeft(empty)(applyEvent)

  def applyEvent(state: BankAccountState, event: BankEvent): BankAccountState =
    event match
      case BankEvent.AccountOpened(id, customerId, initial, _) =>
        state.copy(
          accountId  = id,
          customerId = customerId,
          balance    = initial,
          isOpen     = true
        )
      case BankEvent.MoneyDeposited(_, amount, desc, _) =>
        state.copy(
          balance      = state.balance + amount,
          transactions = transactions :+ s"+$amount: $desc"
        )
      case BankEvent.MoneyWithdrawn(_, amount, desc, _) =>
        state.copy(
          balance      = state.balance - amount,
          transactions = state.transactions :+ s"-$amount: $desc"
        )
      case BankEvent.AccountClosed(_, reason, _) =>
        state.copy(isOpen = false)
```

---

## Aggregate Pattern

### Domain Aggregate with Commands

```scala
import java.time.Instant

// Commands: requests to change state
sealed trait BankCommand
object BankCommand:
  case class OpenAccount(customerId: String, initialBalance: BigDecimal) extends BankCommand
  case class Deposit(amount: BigDecimal, description: String)             extends BankCommand
  case class Withdraw(amount: BigDecimal, description: String)            extends BankCommand
  case class CloseAccount(reason: String)                                 extends BankCommand

// Domain errors
sealed trait BankError
object BankError:
  case object AccountNotOpen     extends BankError
  case object InsufficientFunds  extends BankError
  case object AccountAlreadyOpen extends BankError
  case class InvalidAmount(msg: String) extends BankError

// Aggregate: handles commands, produces events
object BankAccountAggregate:
  type Result = Either[BankError, List[BankEvent]]

  def handle(state: BankAccountState, cmd: BankCommand): Result =
    cmd match
      case BankCommand.OpenAccount(customerId, initial) =>
        if state.isOpen then Left(BankError.AccountAlreadyOpen)
        else if initial < 0 then Left(BankError.InvalidAmount("Initial balance cannot be negative"))
        else Right(List(BankEvent.AccountOpened(
          accountId     = java.util.UUID.randomUUID().toString,
          customerId    = customerId,
          initialBalance = initial,
          openedAt      = Instant.now()
        )))

      case BankCommand.Deposit(amount, desc) =>
        if !state.isOpen then Left(BankError.AccountNotOpen)
        else if amount <= 0 then Left(BankError.InvalidAmount("Deposit amount must be positive"))
        else Right(List(BankEvent.MoneyDeposited(
          accountId   = state.accountId,
          amount      = amount,
          description = desc,
          depositedAt = Instant.now()
        )))

      case BankCommand.Withdraw(amount, desc) =>
        if !state.isOpen then Left(BankError.AccountNotOpen)
        else if amount <= 0 then Left(BankError.InvalidAmount("Withdraw amount must be positive"))
        else if state.balance < amount then Left(BankError.InsufficientFunds)
        else Right(List(BankEvent.MoneyWithdrawn(
          accountId   = state.accountId,
          amount      = amount,
          description = desc,
          withdrawnAt = Instant.now()
        )))

      case BankCommand.CloseAccount(reason) =>
        if !state.isOpen then Left(BankError.AccountNotOpen)
        else Right(List(BankEvent.AccountClosed(
          accountId = state.accountId,
          reason    = reason,
          closedAt  = Instant.now()
        )))
```

---

## Event Store

### Storing and Loading Events

```scala
import cats.effect.{IO, Ref}

// Event store interface
trait EventStore[E]:
  def append(streamId: String, events: List[E], expectedVersion: Long): IO[Unit]
  def load(streamId: String): IO[List[E]]
  def loadFrom(streamId: String, fromVersion: Long): IO[List[E]]

// In-memory implementation
class InMemoryEventStore[E] private (
  store: Ref[IO, Map[String, List[(Long, E)]]]
) extends EventStore[E]:

  def append(streamId: String, events: List[E], expectedVersion: Long): IO[Unit] =
    store.modify { m =>
      val existing = m.getOrElse(streamId, List.empty)
      val currentVersion = existing.length.toLong

      if currentVersion != expectedVersion then
        throw new RuntimeException(
          s"Optimistic concurrency violation: expected $expectedVersion, got $currentVersion"
        )

      val newEvents = events.zipWithIndex.map { case (e, i) =>
        (currentVersion + i + 1, e)
      }
      (m + (streamId -> (existing ++ newEvents)), ())
    }

  def load(streamId: String): IO[List[E]] =
    store.get.map(_.getOrElse(streamId, List.empty).map(_._2))

  def loadFrom(streamId: String, fromVersion: Long): IO[List[E]] =
    store.get.map(
      _.getOrElse(streamId, List.empty)
        .filter(_._1 > fromVersion)
        .map(_._2)
    )

object InMemoryEventStore:
  def make[E]: IO[InMemoryEventStore[E]] =
    Ref.of[IO, Map[String, List[(Long, E)]]](Map.empty)
      .map(InMemoryEventStore(_))
```

---

## Read Model

### Projections for Queries

```scala
import cats.effect.{IO, Ref}

// Read model: optimized for queries
case class AccountSummary(
  accountId: String,
  customerId: String,
  balance: BigDecimal,
  transactionCount: Int,
  isOpen: Boolean
)

case class TransactionHistory(
  accountId: String,
  entries: List[TransactionEntry]
)

case class TransactionEntry(
  eventType: String,
  amount: BigDecimal,
  description: String,
  timestamp: java.time.Instant
)

// Projection: builds read model from events
class AccountProjection(
  summaries: Ref[IO, Map[String, AccountSummary]],
  histories: Ref[IO, Map[String, List[TransactionEntry]]]
):
  // Process event stream
  def project(event: BankEvent): IO[Unit] =
    event match
      case BankEvent.AccountOpened(id, custId, initial, ts) =>
        summaries.update(_ + (id -> AccountSummary(id, custId, initial, 0, isOpen = true)))

      case BankEvent.MoneyDeposited(id, amount, desc, ts) =>
        val entry = TransactionEntry("deposit", amount, desc, ts)
        for
          _ <- summaries.update(m => m.get(id).fold(m) { s =>
            m + (id -> s.copy(balance = s.balance + amount, transactionCount = s.transactionCount + 1))
          })
          _ <- histories.update(m =>
            m + (id -> (m.getOrElse(id, Nil) :+ entry))
          )
        yield ()

      case BankEvent.MoneyWithdrawn(id, amount, desc, ts) =>
        val entry = TransactionEntry("withdrawal", -amount, desc, ts)
        for
          _ <- summaries.update(m => m.get(id).fold(m) { s =>
            m + (id -> s.copy(balance = s.balance - amount, transactionCount = s.transactionCount + 1))
          })
          _ <- histories.update(m =>
            m + (id -> (m.getOrElse(id, Nil) :+ entry))
          )
        yield ()

      case BankEvent.AccountClosed(id, _, _) =>
        summaries.update(m => m.get(id).fold(m) { s =>
          m + (id -> s.copy(isOpen = false))
        })

  // Query methods
  def getAccount(id: String): IO[Option[AccountSummary]] =
    summaries.get.map(_.get(id))

  def getHistory(id: String): IO[List[TransactionEntry]] =
    histories.get.map(_.getOrElse(id, Nil))

  def getAllAccounts: IO[List[AccountSummary]] =
    summaries.get.map(_.values.toList)
```

---

## Complete Example

### Bank Account System

```scala
import cats.effect.{IO, Ref}

// Command handler: ties everything together
class BankAccountCommandHandler(
  eventStore: EventStore[BankEvent],
  projection: AccountProjection
):
  def handle(accountId: String, cmd: BankCommand): IO[Either[BankError, Unit]] =
    for
      events     <- eventStore.load(accountId)
      state       = BankAccountState(events)
      version     = events.length.toLong
      result     <- IO.pure(BankAccountAggregate.handle(state, cmd))
      finalResult <- result match
        case Left(err) => IO.pure(Left(err))
        case Right(newEvents) =>
          eventStore.append(accountId, newEvents, version)
            .flatMap(_ => IO.foreach(newEvents)(projection.project))
            .as(Right(()))
    yield finalResult

// Demo program
object BankDemo extends cats.effect.IOApp.Simple:
  def run: IO[Unit] =
    for
      store      <- InMemoryEventStore.make[BankEvent]
      summaries  <- Ref.of[IO, Map[String, AccountSummary]](Map.empty)
      histories  <- Ref.of[IO, Map[String, List[TransactionEntry]]](Map.empty)
      projection  = AccountProjection(summaries, histories)
      handler     = BankAccountCommandHandler(store, projection)

      accountId  = "ACC-001"

      // Open account
      r1         <- handler.handle(accountId, BankCommand.OpenAccount("CUST-001", BigDecimal("1000")))
      _ <- IO.println(s"Open: $r1")

      // Deposit
      r2 <- handler.handle(accountId, BankCommand.Deposit(BigDecimal("500"), "Salary"))
      _ <- IO.println(s"Deposit: $r2")

      // Withdraw
      r3 <- handler.handle(accountId, BankCommand.Withdraw(BigDecimal("200"), "Rent"))
      _ <- IO.println(s"Withdraw: $r3")

      // Try to overdraw
      r4 <- handler.handle(accountId, BankCommand.Withdraw(BigDecimal("5000"), "Big purchase"))
      _ <- IO.println(s"Overdraw attempt: $r4")

      // Query
      account <- projection.getAccount(accountId)
      _ <- IO.println(s"Account: $account")

      history <- projection.getHistory(accountId)
      _ <- IO.println("Transaction history:")
      _ <- IO.foreach(history)(e => IO.println(s"  ${e.eventType}: ${e.amount} - ${e.description}"))
    yield ()
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ CQRS: แยก command/query models
- ✅ Event Sourcing: events เป็น source of truth
- ✅ Aggregate pattern: handle commands, enforce invariants
- ✅ Optimistic concurrency กับ event version
- ✅ Read model/projections: denormalized views
- ✅ Complete bank account example

---

*[← Part 40: Microservices](part-40-microservices.md) | [Part 42: Domain-Driven Design →](part-42-ddd.md)*
