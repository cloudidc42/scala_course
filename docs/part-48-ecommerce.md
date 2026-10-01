# Part 48: Real-World Project - E-Commerce Platform

## สารบัญ
1. [Project Architecture](#architecture)
2. [Domain Model](#domain-model)
3. [Product Service](#product-service)
4. [Cart Service](#cart-service)
5. [Order Service](#order-service)
6. [Payment Integration](#payment)

---

## Project Architecture

### System Design

```
E-Commerce Platform Architecture:

┌─────────────┐     ┌─────────────────────────────────────────┐
│   Browser   │────▶│           API Gateway                   │
│  Mobile App │     │  (Authentication, Rate Limiting, Routing)│
└─────────────┘     └──────────┬──────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │Product Service│  │ Cart Service │  │Order Service │
    │  (Catalog)    │  │  (Session)   │  │ (Checkout)   │
    └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
           │                 │                  │
    ┌──────▼───────┐  ┌─────▼────────┐  ┌─────▼────────┐
    │  PostgreSQL  │  │   Redis      │  │  PostgreSQL  │
    └──────────────┘  └──────────────┘  └──────────────┘

Async Communication via Kafka:
- order.created → notification-service
- order.created → inventory-service (reserve stock)
- payment.completed → order-service (update status)
```

### Build Structure

```scala
// build.sbt
lazy val commonSettings = Seq(
  scalaVersion := "3.3.1",
  libraryDependencies ++= Seq(
    "org.typelevel" %% "cats-effect" % "3.5.2",
    "io.circe" %% "circe-generic" % "0.14.6",
    "ch.qos.logback" % "logback-classic" % "1.4.11"
  )
)

lazy val common = (project in file("modules/common")).settings(commonSettings)

lazy val productService = (project in file("modules/product-service"))
  .settings(commonSettings)
  .settings(
    libraryDependencies ++= Seq(
      "org.tpolecat" %% "doobie-core" % "1.0.0-RC4",
      "org.tpolecat" %% "doobie-postgres" % "1.0.0-RC4"
    )
  )
  .dependsOn(common)

lazy val cartService = (project in file("modules/cart-service"))
  .settings(commonSettings)
  .settings(
    libraryDependencies += "dev.profunktor" %% "redis4cats-effects" % "1.6.0"
  )
  .dependsOn(common)

lazy val orderService = (project in file("modules/order-service"))
  .dependsOn(common)
```

---

## Domain Model

### Core Entities

```scala
// common/src/main/scala/domain/

import java.util.UUID
import java.time.Instant

// Product
case class ProductId(value: UUID)    extends AnyVal
case class CategoryId(value: UUID)   extends AnyVal
case class OrderId(value: UUID)      extends AnyVal
case class CustomerId(value: UUID)   extends AnyVal
case class CartId(value: UUID)       extends AnyVal

case class Money(amount: BigDecimal, currency: String = "THB"):
  def +(other: Money): Money = copy(amount = amount + other.amount)
  def *(n: Int): Money = copy(amount = amount * n)
  override def toString: String = f"$currency $amount%.2f"

case class Product(
  id: ProductId,
  name: String,
  description: String,
  price: Money,
  stockQuantity: Int,
  categoryId: CategoryId,
  imageUrl: Option[String],
  isActive: Boolean,
  createdAt: Instant
)

case class CartItem(
  productId: ProductId,
  name: String,
  price: Money,
  quantity: Int
):
  def total: Money = price * quantity

case class Cart(
  id: CartId,
  customerId: CustomerId,
  items: List[CartItem],
  updatedAt: Instant
):
  def total: Money =
    if items.isEmpty then Money(0)
    else items.map(_.total).reduce(_ + _)

  def itemCount: Int = items.map(_.quantity).sum

enum OrderStatus:
  case Pending, Confirmed, Processing, Shipped, Delivered, Cancelled, Refunded

case class OrderItem(
  productId: ProductId,
  name: String,
  price: Money,
  quantity: Int
)

case class Order(
  id: OrderId,
  customerId: CustomerId,
  items: List[OrderItem],
  subtotal: Money,
  tax: Money,
  total: Money,
  status: OrderStatus,
  shippingAddress: Address,
  createdAt: Instant,
  updatedAt: Instant
)

case class Address(
  line1: String,
  line2: Option[String],
  city: String,
  province: String,
  postalCode: String,
  country: String = "Thailand"
)
```

---

## Product Service

### Complete Product Service

```scala
// product-service

import cats.effect.{IO, Ref}
import doobie.*
import doobie.implicits.*
import io.circe.generic.auto.*
import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.server.http4s.Http4sServerInterpreter

// Repository
trait ProductRepository:
  def findById(id: ProductId): IO[Option[Product]]
  def findAll(page: Int, pageSize: Int, category: Option[CategoryId]): IO[List[Product]]
  def search(query: String, page: Int, pageSize: Int): IO[List[Product]]
  def create(product: Product): IO[Product]
  def update(id: ProductId, product: Product): IO[Option[Product]]
  def updateStock(id: ProductId, delta: Int): IO[Either[String, Product]]

class DoobieProductRepository(xa: Transactor[IO]) extends ProductRepository:
  def findById(id: ProductId): IO[Option[Product]] =
    sql"""
      SELECT id, name, description, price, stock_quantity, category_id,
             image_url, is_active, created_at
      FROM products WHERE id = ${id.value} AND is_active = true
    """.query[Product].option.transact(xa)

  def findAll(page: Int, pageSize: Int, category: Option[CategoryId]): IO[List[Product]] =
    val base = fr"SELECT id, name, description, price, stock_quantity, category_id, image_url, is_active, created_at FROM products WHERE is_active = true"
    val catFilter = category.map(c => fr"AND category_id = ${c.value}")
    val pagination = fr"ORDER BY name LIMIT $pageSize OFFSET ${page * pageSize}"
    val query = catFilter.fold(base ++ pagination)(f => base ++ f ++ pagination)
    query.query[Product].to[List].transact(xa)

  def search(q: String, page: Int, pageSize: Int): IO[List[Product]] =
    sql"""
      SELECT id, name, description, price, stock_quantity, category_id,
             image_url, is_active, created_at
      FROM products
      WHERE is_active = true
        AND (name ILIKE ${"%" + q + "%"} OR description ILIKE ${"%" + q + "%"})
      ORDER BY name
      LIMIT $pageSize OFFSET ${page * pageSize}
    """.query[Product].to[List].transact(xa)

  def create(p: Product): IO[Product] =
    sql"""
      INSERT INTO products (id, name, description, price, stock_quantity,
                            category_id, image_url, is_active, created_at)
      VALUES (${p.id.value}, ${p.name}, ${p.description}, ${p.price.amount},
              ${p.stockQuantity}, ${p.categoryId.value}, ${p.imageUrl},
              ${p.isActive}, ${p.createdAt})
    """.update.run.transact(xa).as(p)

  def update(id: ProductId, p: Product): IO[Option[Product]] =
    sql"""
      UPDATE products
      SET name = ${p.name}, description = ${p.description}, price = ${p.price.amount},
          image_url = ${p.imageUrl}, is_active = ${p.isActive}
      WHERE id = ${id.value}
    """.update.run.transact(xa).map(n => if n > 0 then Some(p) else None)

  def updateStock(id: ProductId, delta: Int): IO[Either[String, Product]] =
    val query = sql"""
      UPDATE products SET stock_quantity = stock_quantity + $delta
      WHERE id = ${id.value} AND stock_quantity + $delta >= 0
      RETURNING id, name, description, price, stock_quantity, category_id,
                image_url, is_active, created_at
    """.query[Product].option.transact(xa)

    query.map {
      case Some(p) => Right(p)
      case None    => Left(s"Insufficient stock for product ${id.value}")
    }

// Service layer
class ProductService(repo: ProductRepository):
  case class SearchParams(query: Option[String], category: Option[CategoryId], page: Int = 0, pageSize: Int = 20)
  case class CreateProductRequest(name: String, description: String, price: BigDecimal, stock: Int, categoryId: UUID)

  def search(params: SearchParams): IO[List[Product]] =
    params.query match
      case Some(q) if q.trim.nonEmpty => repo.search(q, params.page, params.pageSize)
      case _                          => repo.findAll(params.page, params.pageSize, params.category)

  def getById(id: ProductId): IO[Either[String, Product]] =
    repo.findById(id).map {
      case Some(p) => Right(p)
      case None    => Left(s"Product not found: ${id.value}")
    }

  def create(req: CreateProductRequest): IO[Either[String, Product]] =
    if req.name.trim.isEmpty then IO.pure(Left("Name cannot be empty"))
    else if req.price <= 0 then IO.pure(Left("Price must be positive"))
    else if req.stock < 0 then IO.pure(Left("Stock cannot be negative"))
    else
      val product = Product(
        id            = ProductId(UUID.randomUUID()),
        name          = req.name.trim,
        description   = req.description,
        price         = Money(req.price),
        stockQuantity = req.stock,
        categoryId    = CategoryId(req.categoryId),
        imageUrl      = None,
        isActive      = true,
        createdAt     = Instant.now()
      )
      repo.create(product).map(Right(_))
```

---

## Cart Service

### Redis-backed Cart

```scala
// cart-service

import dev.profunktor.redis4cats.RedisCommands
import cats.effect.IO
import io.circe.generic.auto.*
import io.circe.syntax.*
import io.circe.parser.*
import scala.concurrent.duration.*

class CartService(redis: RedisCommands[IO, String, String]):
  private def cartKey(customerId: CustomerId) = s"cart:${customerId.value}"
  private val CartTTL = 7.days

  def getCart(customerId: CustomerId): IO[Cart] =
    redis.get(cartKey(customerId)).map {
      case Some(json) =>
        decode[Cart](json).getOrElse(emptyCart(customerId))
      case None =>
        emptyCart(customerId)
    }

  def addItem(customerId: CustomerId, item: CartItem): IO[Cart] =
    for
      cart    <- getCart(customerId)
      updated  = cart.items.indexWhere(_.productId == item.productId) match
        case -1 => cart.copy(items = cart.items :+ item, updatedAt = Instant.now())
        case i  =>
          val existing = cart.items(i)
          val newItem  = existing.copy(quantity = existing.quantity + item.quantity)
          cart.copy(
            items     = cart.items.updated(i, newItem),
            updatedAt = Instant.now()
          )
      _ <- redis.setEx(cartKey(customerId), updated.asJson.noSpaces, CartTTL)
    yield updated

  def removeItem(customerId: CustomerId, productId: ProductId): IO[Cart] =
    for
      cart    <- getCart(customerId)
      updated  = cart.copy(
        items     = cart.items.filterNot(_.productId == productId),
        updatedAt = Instant.now()
      )
      _ <- redis.setEx(cartKey(customerId), updated.asJson.noSpaces, CartTTL)
    yield updated

  def updateQuantity(customerId: CustomerId, productId: ProductId, quantity: Int): IO[Cart] =
    if quantity <= 0 then removeItem(customerId, productId)
    else
      for
        cart    <- getCart(customerId)
        updated  = cart.items.indexWhere(_.productId == productId) match
          case -1 => cart  // item not in cart, no-op
          case i  =>
            cart.copy(
              items     = cart.items.updated(i, cart.items(i).copy(quantity = quantity)),
              updatedAt = Instant.now()
            )
        _ <- redis.setEx(cartKey(customerId), updated.asJson.noSpaces, CartTTL)
      yield updated

  def clearCart(customerId: CustomerId): IO[Unit] =
    redis.del(cartKey(customerId)).void

  private def emptyCart(customerId: CustomerId): Cart =
    Cart(CartId(UUID.randomUUID()), customerId, Nil, Instant.now())
```

---

## Order Service

### Checkout Process

```scala
// order-service

case class CheckoutRequest(
  customerId: CustomerId,
  cartId: CartId,
  shippingAddress: Address,
  paymentMethodId: String
)

class OrderService(
  orderRepo: OrderRepository,
  cartService: CartService,
  productService: ProductService,
  paymentService: PaymentService,
  eventPublisher: EventPublisher
):
  def checkout(req: CheckoutRequest): IO[Either[String, Order]] =
    for
      // 1. Get cart
      cart <- cartService.getCart(req.customerId)
      result <-
        if cart.items.isEmpty then IO.pure(Left("Cart is empty"))
        else processCheckout(req, cart)
    yield result

  private def processCheckout(req: CheckoutRequest, cart: Cart): IO[Either[String, Order]] =
    for
      // 2. Validate and reserve inventory
      reserveResult <- reserveInventory(cart.items)
      result <- reserveResult match
        case Left(err) => IO.pure(Left(err))
        case Right(_)  =>
          for
            // 3. Create order
            order <- createOrder(req, cart)

            // 4. Process payment
            payResult <- paymentService.charge(
              req.paymentMethodId,
              order.total,
              s"Order ${order.id.value}"
            )
            finalResult <- payResult match
              case Left(payErr) =>
                // Rollback inventory reservation
                releaseInventory(cart.items) *>
                IO.pure(Left(s"Payment failed: $payErr"))
              case Right(_) =>
                for
                  confirmed <- orderRepo.updateStatus(order.id, OrderStatus.Confirmed)
                  // 5. Clear cart
                  _ <- cartService.clearCart(req.customerId)
                  // 6. Publish event
                  _ <- eventPublisher.publish("order.created", order.asJson.noSpaces)
                yield Right(confirmed.getOrElse(order))
          yield finalResult
    yield result

  private def reserveInventory(items: List[CartItem]): IO[Either[String, Unit]] =
    items.foldLeft[IO[Either[String, Unit]]](IO.pure(Right(()))) { (acc, item) =>
      acc.flatMap {
        case Left(err) => IO.pure(Left(err))
        case Right(_)  =>
          productService.updateStock(item.productId, -item.quantity)
            .map(_.map(_ => ()))
      }
    }

  private def releaseInventory(items: List[CartItem]): IO[Unit] =
    items.foreach_ { item =>
      productService.updateStock(item.productId, item.quantity).void
    }

  private def createOrder(req: CheckoutRequest, cart: Cart): IO[Order] =
    val items = cart.items.map(ci => OrderItem(ci.productId, ci.name, ci.price, ci.quantity))
    val subtotal = cart.total
    val tax      = Money(subtotal.amount * BigDecimal("0.07"))  // 7% VAT
    val total    = subtotal + tax
    val now      = Instant.now()

    val order = Order(
      id              = OrderId(UUID.randomUUID()),
      customerId      = req.customerId,
      items           = items,
      subtotal        = subtotal,
      tax             = tax,
      total           = total,
      status          = OrderStatus.Pending,
      shippingAddress = req.shippingAddress,
      createdAt       = now,
      updatedAt       = now
    )
    orderRepo.create(order)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ E-Commerce system design: microservices, event-driven
- ✅ Domain model: Products, Cart, Orders
- ✅ Product Service: Doobie + PostgreSQL, full-text search
- ✅ Cart Service: Redis-backed session storage
- ✅ Order Service: checkout flow, inventory reservation, payment

---

*[← Part 47: Testing Strategies](part-47-testing-strategies.md) | [Part 49: Real-Time Chat Application →](part-49-realtime-chat.md)*
