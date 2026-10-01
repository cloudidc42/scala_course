# Part 42: Domain-Driven Design (DDD)

## สารบัญ
1. [DDD Concepts](#ddd-concepts)
2. [Value Objects](#value-objects)
3. [Entities](#entities)
4. [Aggregates](#aggregates)
5. [Domain Services](#domain-services)
6. [Repositories](#repositories)
7. [Application Services](#application-services)

---

## DDD Concepts

### Ubiquitous Language

```
Domain-Driven Design (DDD):
ออกแบบซอฟต์แวร์โดยเอา domain (problem space) เป็นศูนย์กลาง

Core concepts:
- Ubiquitous Language: ภาษากลางระหว่าง dev และ domain experts
- Bounded Context: boundary ที่ model มีความหมายเฉพาะ
- Value Object: immutable, no identity
- Entity: mutable, has identity
- Aggregate: cluster of objects, consistency boundary
- Domain Service: business logic ที่ไม่อยู่ใน entity/VO
- Repository: abstracts persistence
- Domain Event: something that happened

Layers:
Domain → Application → Infrastructure → Presentation
```

---

## Value Objects

### Immutable, Identity-free Objects

```scala
// Value Object: identity comes from values, not reference
case class Money(amount: BigDecimal, currency: String):
  require(amount >= 0, "Amount must be non-negative")
  require(currency.length == 3, "Currency must be 3 characters")

  def +(other: Money): Money =
    require(currency == other.currency, s"Currency mismatch: $currency vs ${other.currency}")
    Money(amount + other.amount, currency)

  def -(other: Money): Money =
    require(currency == other.currency, s"Currency mismatch")
    require(amount >= other.amount, "Insufficient funds")
    Money(amount - other.amount, currency)

  def *(factor: BigDecimal): Money = Money(amount * factor, currency)

  override def toString: String = s"$currency ${amount.setScale(2)}"

// Usage
val price   = Money(BigDecimal("100.00"), "THB")
val tax     = price * BigDecimal("0.07")
val total   = price + tax
println(total)  // THB 107.00

// Value Object: Email Address
opaque type Email = String

object Email:
  private val pattern = """^[^@]+@[^@]+\.[^@]+$""".r

  def apply(value: String): Either[String, Email] =
    if pattern.matches(value) then Right(value)
    else Left(s"Invalid email: $value")

  extension (e: Email)
    def value: String = e
    def domain: String = e.split("@").last

// Phone number value object
case class PhoneNumber private (countryCode: String, number: String):
  override def toString: String = s"+$countryCode$number"

object PhoneNumber:
  def apply(countryCode: String, number: String): Either[String, PhoneNumber] =
    if countryCode.matches("\\d{1,3}") && number.matches("\\d{6,14}")
    then Right(new PhoneNumber(countryCode, number))
    else Left(s"Invalid phone: +$countryCode$number")
```

---

## Entities

### Objects with Identity

```scala
import java.util.UUID
import java.time.Instant

// Entity: has unique identity (id)
case class CustomerId(value: UUID) extends AnyVal

case class Customer(
  id: CustomerId,
  name: String,
  email: Email,
  phone: Option[PhoneNumber],
  createdAt: Instant,
  updatedAt: Instant
):
  def updateEmail(newEmail: Email): Customer =
    copy(email = newEmail, updatedAt = Instant.now())

  def updatePhone(phone: PhoneNumber): Customer =
    copy(phone = Some(phone), updatedAt = Instant.now())

object Customer:
  def create(name: String, email: Email): Customer =
    Customer(
      id        = CustomerId(UUID.randomUUID()),
      name      = name,
      email     = email,
      phone     = None,
      createdAt = Instant.now(),
      updatedAt = Instant.now()
    )

// Entity with business rules
case class OrderId(value: UUID) extends AnyVal

enum OrderStatus:
  case Draft, Pending, Confirmed, Shipped, Delivered, Cancelled

case class OrderLine(
  productId: UUID,
  quantity: Int,
  unitPrice: Money
):
  def total: Money = unitPrice * BigDecimal(quantity)

class Order private (
  val id: OrderId,
  val customerId: CustomerId,
  private var _status: OrderStatus,
  private var _lines: List[OrderLine],
  val createdAt: Instant
):
  def status: OrderStatus = _status
  def lines: List[OrderLine] = _lines

  def total: Money =
    if _lines.isEmpty then Money(BigDecimal(0), "THB")
    else _lines.map(_.total).reduce(_ + _)

  def addLine(line: OrderLine): Either[String, Unit] =
    if _status != OrderStatus.Draft then
      Left("Can only add lines to draft orders")
    else
      _lines = _lines :+ line
      Right(())

  def confirm(): Either[String, Unit] =
    if _lines.isEmpty then Left("Order has no lines")
    else if _status != OrderStatus.Draft then Left(s"Cannot confirm ${_status} order")
    else
      _status = OrderStatus.Confirmed
      Right(())

  def cancel(reason: String): Either[String, Unit] =
    if _status == OrderStatus.Delivered then Left("Cannot cancel delivered order")
    else
      _status = OrderStatus.Cancelled
      Right(())

object Order:
  def create(customerId: CustomerId): Order =
    new Order(
      id         = OrderId(UUID.randomUUID()),
      customerId = customerId,
      _status    = OrderStatus.Draft,
      _lines     = List.empty,
      createdAt  = Instant.now()
    )
```

---

## Aggregates

### Consistency Boundary

```scala
// Aggregate Root: enforces business rules for the whole aggregate

// Inventory aggregate
case class ProductId(value: UUID) extends AnyVal
case class WarehouseId(value: UUID) extends AnyVal

case class InventoryItem(
  productId: ProductId,
  reserved: Int,
  available: Int
):
  def totalStock: Int = reserved + available

class Inventory private (
  val warehouseId: WarehouseId,
  private var items: Map[ProductId, InventoryItem]
):
  def getItem(productId: ProductId): Option[InventoryItem] =
    items.get(productId)

  def addStock(productId: ProductId, quantity: Int): Either[String, Unit] =
    if quantity <= 0 then Left("Quantity must be positive")
    else
      val current = items.getOrElse(productId, InventoryItem(productId, 0, 0))
      items = items + (productId -> current.copy(available = current.available + quantity))
      Right(())

  def reserve(productId: ProductId, quantity: Int): Either[String, Unit] =
    items.get(productId) match
      case None =>
        Left(s"Product ${productId.value} not found")
      case Some(item) if item.available < quantity =>
        Left(s"Insufficient stock: available=${item.available}, requested=$quantity")
      case Some(item) =>
        items = items + (productId -> item.copy(
          reserved  = item.reserved + quantity,
          available = item.available - quantity
        ))
        Right(())

  def release(productId: ProductId, quantity: Int): Either[String, Unit] =
    items.get(productId) match
      case None => Left(s"Product ${productId.value} not found")
      case Some(item) if item.reserved < quantity =>
        Left(s"Cannot release more than reserved: reserved=${item.reserved}")
      case Some(item) =>
        items = items + (productId -> item.copy(
          reserved  = item.reserved - quantity,
          available = item.available + quantity
        ))
        Right(())

object Inventory:
  def create(warehouseId: WarehouseId): Inventory =
    new Inventory(warehouseId, Map.empty)
```

---

## Domain Services

### Cross-Aggregate Business Logic

```scala
// Domain Service: business logic spanning multiple aggregates or external concerns

trait PricingService:
  def calculatePrice(productId: ProductId, quantity: Int): Either[String, Money]

trait DiscountService:
  def applyDiscounts(order: Order, customer: Customer): Money

// Implementation example
class PricingServiceImpl extends PricingService:
  private val prices = Map(
    // productId -> base price
  )

  def calculatePrice(productId: ProductId, quantity: Int): Either[String, Money] =
    // Complex pricing logic: bulk discounts, seasonal pricing, etc.
    Right(Money(BigDecimal("100.00"), "THB"))

class OrderPricingDomainService(
  pricing: PricingService,
  discount: DiscountService
):
  def calculateOrderTotal(order: Order, customer: Customer): Either[String, Money] =
    for
      lineTotal <- Right(order.total)
      discount   = discount.applyDiscounts(order, customer)
      finalTotal = lineTotal - discount
    yield finalTotal

// Transfer Service: spans two aggregate roots
class TransferService:
  def transfer(
    from: Inventory,
    to: Inventory,
    productId: ProductId,
    quantity: Int
  ): Either[String, Unit] =
    for
      _ <- from.reserve(productId, quantity)
      _ <- to.addStock(productId, quantity)
      _ <- from.release(productId, quantity)  // complete the transfer
    yield ()
```

---

## Repositories

### Persistence Abstraction

```scala
import cats.effect.IO

// Repository interface (in domain layer)
trait CustomerRepository:
  def findById(id: CustomerId): IO[Option[Customer]]
  def findByEmail(email: Email): IO[Option[Customer]]
  def save(customer: Customer): IO[Customer]
  def delete(id: CustomerId): IO[Unit]
  def findAll(page: Int, pageSize: Int): IO[List[Customer]]

trait OrderRepository:
  def findById(id: OrderId): IO[Option[Order]]
  def findByCustomer(customerId: CustomerId): IO[List[Order]]
  def save(order: Order): IO[Order]

// In-memory implementation (for testing)
class InMemoryCustomerRepository extends CustomerRepository:
  private var customers = Map.empty[CustomerId, Customer]

  def findById(id: CustomerId): IO[Option[Customer]] =
    IO.pure(customers.get(id))

  def findByEmail(email: Email): IO[Option[Customer]] =
    IO.pure(customers.values.find(_.email == email))

  def save(customer: Customer): IO[Customer] =
    IO { customers = customers + (customer.id -> customer); customer }

  def delete(id: CustomerId): IO[Unit] =
    IO { customers = customers - id }

  def findAll(page: Int, pageSize: Int): IO[List[Customer]] =
    IO.pure(customers.values.drop(page * pageSize).take(pageSize).toList)
```

---

## Application Services

### Orchestrating Use Cases

```scala
import cats.effect.IO

// Use case request/response
case class RegisterCustomerRequest(name: String, email: String)
case class RegisterCustomerResponse(customerId: String, email: String)

case class PlaceOrderRequest(customerId: String, items: List[OrderItemRequest])
case class OrderItemRequest(productId: String, quantity: Int)
case class PlaceOrderResponse(orderId: String, total: String)

// Application service: orchestrates domain objects
class CustomerApplicationService(
  customerRepo: CustomerRepository
):
  def register(req: RegisterCustomerRequest): IO[Either[String, RegisterCustomerResponse]] =
    for
      email <- IO.fromEither(Email(req.email).left.map(identity))
      existing <- customerRepo.findByEmail(email)
      result <- existing match
        case Some(_) =>
          IO.pure(Left(s"Customer with email ${req.email} already exists"))
        case None =>
          val customer = Customer.create(req.name, email)
          customerRepo.save(customer).map { saved =>
            Right(RegisterCustomerResponse(
              customerId = saved.id.value.toString,
              email      = saved.email.value
            ))
          }
    yield result

class OrderApplicationService(
  orderRepo: OrderRepository,
  customerRepo: CustomerRepository,
  inventory: Inventory,
  pricing: PricingService
):
  def placeOrder(req: PlaceOrderRequest): IO[Either[String, PlaceOrderResponse]] =
    for
      customerId <- IO.pure(CustomerId(java.util.UUID.fromString(req.customerId)))
      customer   <- customerRepo.findById(customerId)
      result     <- customer match
        case None => IO.pure(Left(s"Customer not found: ${req.customerId}"))
        case Some(c) =>
          val order = Order.create(c.id)
          val addLinesResult = req.items.foldLeft[Either[String, Unit]](Right(())) {
            (acc, item) => acc.flatMap { _ =>
              val productId = ProductId(java.util.UUID.fromString(item.productId))
              for
                price <- pricing.calculatePrice(productId, item.quantity)
                line   = OrderLine(productId.value, item.quantity, price)
                _     <- order.addLine(line)
                _     <- inventory.reserve(productId, item.quantity)
              yield ()
            }
          }
          IO.fromEither(addLinesResult.left.map(identity))
            .flatMap { _ =>
              IO.fromEither(order.confirm())
                .flatMap { _ =>
                  orderRepo.save(order).map { saved =>
                    Right(PlaceOrderResponse(
                      orderId = saved.id.value.toString,
                      total   = saved.total.toString
                    ))
                  }
                }
            }
    yield result
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ DDD core concepts: ubiquitous language, bounded context
- ✅ Value Objects: immutable, identity from values
- ✅ Entities: mutable, unique identity
- ✅ Aggregates: consistency boundaries, business rules
- ✅ Domain Services: cross-aggregate logic
- ✅ Repositories: persistence abstraction
- ✅ Application Services: orchestrating use cases

---

*[← Part 41: CQRS/ES](part-41-cqrs-event-sourcing.md) | [Part 43: Performance Optimization →](part-43-performance.md)*
