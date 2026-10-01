# ส่วนที่ 75: การออกแบบ API ที่ดี (API Design Best Practices)

## สารบัญ

1. [หลักการออกแบบ REST API](#หลักการออกแบบ-rest-api)
2. [การตั้งชื่อ Resource](#การตั้งชื่อ-resource)
3. [HTTP Verbs และ Status Codes](#http-verbs-และ-status-codes)
4. [กลยุทธ์การแบ่งหน้า (Pagination)](#กลยุทธ์การแบ่งหน้า-pagination)
5. [การกรอง เรียงลำดับ และค้นหา](#การกรอง-เรียงลำดับ-และค้นหา)
6. [HATEOAS](#hateoas)
7. [การกำหนดเวอร์ชัน API](#การกำหนดเวอร์ชัน-api)
8. [OpenAPI/Swagger กับ Tapir](#openapiswagger-กับ-tapir)
9. [ตัวอย่าง API Design แบบสมบูรณ์](#ตัวอย่าง-api-design-แบบสมบูรณ์)
10. [สรุป](#สรุป)

---

## หลักการออกแบบ REST API

REST (Representational State Transfer) เป็นสถาปัตยกรรมที่ใช้กันอย่างแพร่หลายสำหรับการสร้าง Web API หลักการสำคัญของ REST ได้แก่:

- **Stateless**: แต่ละ request ต้องมีข้อมูลครบถ้วนในตัวเอง
- **Client-Server**: แยก UI ออกจาก Data Storage
- **Cacheable**: Response ต้องบอกว่าสามารถ Cache ได้หรือไม่
- **Uniform Interface**: Interface ที่สม่ำเสมอ
- **Layered System**: รองรับ Proxy, Gateway, Load Balancer
- **Code on Demand (optional)**: Server สามารถส่ง executable code ให้ Client

### โครงสร้าง Project

```scala
// build.sbt
lazy val root = project
  .in(file("."))
  .settings(
    name := "scala-rest-api",
    scalaVersion := "3.3.1",
    libraryDependencies ++= Seq(
      "com.softwaremill.sttp.tapir" %% "tapir-core" % "1.9.0",
      "com.softwaremill.sttp.tapir" %% "tapir-http4s-server" % "1.9.0",
      "com.softwaremill.sttp.tapir" %% "tapir-swagger-ui-bundle" % "1.9.0",
      "com.softwaremill.sttp.tapir" %% "tapir-json-circe" % "1.9.0",
      "org.http4s" %% "http4s-ember-server" % "0.23.24",
      "io.circe" %% "circe-generic" % "0.14.6",
      "io.circe" %% "circe-parser" % "0.14.6",
      "org.typelevel" %% "cats-effect" % "3.5.2",
      "ch.qos.logback" % "logback-classic" % "1.4.11"
    )
  )
```

### Domain Models

```scala
// src/main/scala/domain/models.scala
package domain

import java.time.Instant
import java.util.UUID

// Product domain
case class ProductId(value: UUID) extends AnyVal
case class CategoryId(value: UUID) extends AnyVal
case class UserId(value: UUID) extends AnyVal

case class Product(
  id: ProductId,
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: CategoryId,
  stock: Int,
  createdAt: Instant,
  updatedAt: Instant
)

case class Category(
  id: CategoryId,
  name: String,
  description: String
)

// Request/Response DTOs
case class CreateProductRequest(
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: UUID,
  stock: Int
)

case class UpdateProductRequest(
  name: Option[String],
  description: Option[String],
  price: Option[BigDecimal],
  stock: Option[Int]
)

case class ProductResponse(
  id: UUID,
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: UUID,
  stock: Int,
  createdAt: String,
  updatedAt: String
)

// Error types
sealed trait ApiError
case class NotFoundError(resource: String, id: String) extends ApiError
case class ValidationError(field: String, message: String) extends ApiError
case class ConflictError(message: String) extends ApiError
case class InternalError(message: String) extends ApiError

case class ErrorResponse(
  error: String,
  message: String,
  details: Option[List[String]] = None
)
```

---

## การตั้งชื่อ Resource

การตั้งชื่อ Resource ที่ดีเป็นพื้นฐานของ REST API ที่ใช้งานง่าย

### หลักการตั้งชื่อ

```
# ✅ ถูกต้อง - ใช้ noun พหูพจน์
GET /api/v1/products
GET /api/v1/products/{id}
GET /api/v1/products/{id}/reviews
POST /api/v1/products
PUT /api/v1/products/{id}
PATCH /api/v1/products/{id}
DELETE /api/v1/products/{id}

# ✅ ถูกต้อง - Nested resources
GET /api/v1/categories/{categoryId}/products
GET /api/v1/orders/{orderId}/items/{itemId}

# ❌ ผิด - ใช้ verb
GET /api/v1/getProducts
POST /api/v1/createProduct
DELETE /api/v1/deleteProduct/{id}

# ❌ ผิด - ใช้ singular
GET /api/v1/product
POST /api/v1/product

# ✅ ถูกต้อง - การใช้ kebab-case สำหรับ multi-word
GET /api/v1/product-categories
GET /api/v1/user-profiles/{id}/shipping-addresses
```

### Endpoint Definitions

```scala
// src/main/scala/api/endpoints.scala
package api

import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.generic.auto.*
import io.circe.generic.auto.*
import domain.*
import java.util.UUID

object Endpoints:
  
  // Base endpoint with common headers
  private val baseEndpoint = endpoint
    .in("api" / "v1")
    .errorOut(jsonBody[ErrorResponse])
  
  // Product endpoints
  val listProducts = baseEndpoint.get
    .in("products")
    .in(query[Option[Int]]("page"))
    .in(query[Option[Int]]("pageSize"))
    .in(query[Option[String]]("category"))
    .in(query[Option[String]]("sort"))
    .in(query[Option[String]]("order"))
    .in(query[Option[String]]("search"))
    .out(jsonBody[PaginatedResponse[ProductResponse]])
    .description("รายการสินค้าทั้งหมด พร้อม pagination, filtering และ sorting")
  
  val getProduct = baseEndpoint.get
    .in("products" / path[UUID]("id"))
    .out(jsonBody[ProductResponse])
    .description("ดึงข้อมูลสินค้าตาม ID")
  
  val createProduct = baseEndpoint.post
    .in("products")
    .in(jsonBody[CreateProductRequest])
    .out(statusCode(sttp.model.StatusCode.Created))
    .out(jsonBody[ProductResponse])
    .description("สร้างสินค้าใหม่")
  
  val updateProduct = baseEndpoint.put
    .in("products" / path[UUID]("id"))
    .in(jsonBody[CreateProductRequest])
    .out(jsonBody[ProductResponse])
    .description("อัพเดทสินค้า (แทนที่ทั้งหมด)")
  
  val patchProduct = baseEndpoint.patch
    .in("products" / path[UUID]("id"))
    .in(jsonBody[UpdateProductRequest])
    .out(jsonBody[ProductResponse])
    .description("อัพเดทสินค้าบางส่วน")
  
  val deleteProduct = baseEndpoint.delete
    .in("products" / path[UUID]("id"))
    .out(statusCode(sttp.model.StatusCode.NoContent))
    .description("ลบสินค้า")
  
  // Category endpoints
  val getCategoryProducts = baseEndpoint.get
    .in("categories" / path[UUID]("categoryId") / "products")
    .in(query[Option[Int]]("page"))
    .in(query[Option[Int]]("pageSize"))
    .out(jsonBody[PaginatedResponse[ProductResponse]])
    .description("สินค้าในหมวดหมู่")
```

---

## HTTP Verbs และ Status Codes

### HTTP Methods

```scala
// src/main/scala/api/httpMethods.scala
package api

/**
 * HTTP Methods และการใช้งานที่ถูกต้อง
 *
 * GET    - ดึงข้อมูล (Idempotent, Safe)
 * POST   - สร้างข้อมูลใหม่ (ไม่ Idempotent)
 * PUT    - แทนที่ข้อมูลทั้งหมด (Idempotent)
 * PATCH  - อัพเดทบางส่วน (ไม่ Idempotent)
 * DELETE - ลบข้อมูล (Idempotent)
 * HEAD   - เหมือน GET แต่ไม่ส่ง body
 * OPTIONS - ดู methods ที่รองรับ (CORS preflight)
 */

// Status Codes
enum HttpStatus(val code: Int, val message: String):
  // 2xx Success
  case OK extends HttpStatus(200, "OK")
  case Created extends HttpStatus(201, "Created")
  case Accepted extends HttpStatus(202, "Accepted")
  case NoContent extends HttpStatus(204, "No Content")
  
  // 3xx Redirection
  case MovedPermanently extends HttpStatus(301, "Moved Permanently")
  case NotModified extends HttpStatus(304, "Not Modified")
  
  // 4xx Client Errors
  case BadRequest extends HttpStatus(400, "Bad Request")
  case Unauthorized extends HttpStatus(401, "Unauthorized")
  case Forbidden extends HttpStatus(403, "Forbidden")
  case NotFound extends HttpStatus(404, "Not Found")
  case MethodNotAllowed extends HttpStatus(405, "Method Not Allowed")
  case Conflict extends HttpStatus(409, "Conflict")
  case UnprocessableEntity extends HttpStatus(422, "Unprocessable Entity")
  case TooManyRequests extends HttpStatus(429, "Too Many Requests")
  
  // 5xx Server Errors
  case InternalServerError extends HttpStatus(500, "Internal Server Error")
  case BadGateway extends HttpStatus(502, "Bad Gateway")
  case ServiceUnavailable extends HttpStatus(503, "Service Unavailable")
```

### Error Handling

```scala
// src/main/scala/api/errorHandling.scala
package api

import cats.effect.IO
import org.typelevel.log4cats.Logger
import domain.*

class ErrorHandler(using logger: Logger[IO]):
  
  def handleApiError(error: ApiError): (Int, ErrorResponse) =
    error match
      case NotFoundError(resource, id) =>
        (404, ErrorResponse(
          error = "NOT_FOUND",
          message = s"$resource with id '$id' not found"
        ))
      
      case ValidationError(field, message) =>
        (422, ErrorResponse(
          error = "VALIDATION_ERROR",
          message = "Validation failed",
          details = Some(List(s"$field: $message"))
        ))
      
      case ConflictError(message) =>
        (409, ErrorResponse(
          error = "CONFLICT",
          message = message
        ))
      
      case InternalError(message) =>
        logger.error(s"Internal error: $message")
        (500, ErrorResponse(
          error = "INTERNAL_SERVER_ERROR",
          message = "An internal error occurred"
        ))

// Request validation
case class ValidationResult[A](value: A, errors: List[String]):
  def isValid: Boolean = errors.isEmpty

object Validator:
  def validateCreateProduct(req: CreateProductRequest): ValidationResult[CreateProductRequest] =
    val errors = List.newBuilder[String]
    
    if req.name.trim.isEmpty then
      errors += "name: must not be empty"
    
    if req.name.length > 100 then
      errors += "name: must be 100 characters or less"
    
    if req.price <= 0 then
      errors += "price: must be greater than 0"
    
    if req.stock < 0 then
      errors += "stock: must be 0 or greater"
    
    ValidationResult(req, errors.result())
```

---

## กลยุทธ์การแบ่งหน้า (Pagination)

### Offset-based Pagination

```scala
// src/main/scala/api/pagination.scala
package api

// Offset-based pagination
case class OffsetPaginationParams(
  page: Int = 1,
  pageSize: Int = 20
):
  def offset: Int = (page - 1) * pageSize
  def limit: Int = pageSize
  
  require(page >= 1, "page must be >= 1")
  require(pageSize >= 1 && pageSize <= 100, "pageSize must be between 1 and 100")

case class PaginatedResponse[A](
  data: List[A],
  pagination: PaginationMeta
)

case class PaginationMeta(
  page: Int,
  pageSize: Int,
  totalItems: Long,
  totalPages: Int,
  hasNextPage: Boolean,
  hasPreviousPage: Boolean
)

object PaginationMeta:
  def fromOffsetParams(params: OffsetPaginationParams, totalItems: Long): PaginationMeta =
    val totalPages = Math.ceil(totalItems.toDouble / params.pageSize).toInt
    PaginationMeta(
      page = params.page,
      pageSize = params.pageSize,
      totalItems = totalItems,
      totalPages = totalPages,
      hasNextPage = params.page < totalPages,
      hasPreviousPage = params.page > 1
    )
```

### Cursor-based Pagination

```scala
// Cursor-based pagination (ดีกว่าสำหรับข้อมูลที่เปลี่ยนบ่อย)
import java.util.Base64
import java.time.Instant

case class Cursor(value: String) extends AnyVal:
  def decode: Option[CursorData] =
    try
      val decoded = Base64.getDecoder.decode(value)
      val str = new String(decoded)
      // parse "id:timestamp" format
      str.split(":").toList match
        case id :: ts :: Nil =>
          Some(CursorData(id, Instant.ofEpochMilli(ts.toLong)))
        case _ => None
    catch
      case _: Exception => None

case class CursorData(id: String, timestamp: Instant)

object Cursor:
  def encode(id: String, timestamp: Instant): Cursor =
    val str = s"$id:${timestamp.toEpochMilli}"
    Cursor(Base64.getEncoder.encodeToString(str.getBytes))

case class CursorPaginationParams(
  cursor: Option[Cursor] = None,
  limit: Int = 20,
  direction: CursorDirection = CursorDirection.Forward
)

enum CursorDirection:
  case Forward, Backward

case class CursorPaginatedResponse[A](
  data: List[A],
  nextCursor: Option[String],
  previousCursor: Option[String],
  hasMore: Boolean
)

// Repository implementation
trait ProductRepository[F[_]]:
  def findWithCursor(
    params: CursorPaginationParams,
    filter: ProductFilter
  ): F[CursorPaginatedResponse[Product]]
  
  def findWithOffset(
    params: OffsetPaginationParams,
    filter: ProductFilter
  ): F[PaginatedResponse[Product]]

case class ProductFilter(
  categoryId: Option[CategoryId] = None,
  minPrice: Option[BigDecimal] = None,
  maxPrice: Option[BigDecimal] = None,
  inStock: Option[Boolean] = None,
  search: Option[String] = None
)
```

---

## การกรอง เรียงลำดับ และค้นหา

### Query Parameters Design

```scala
// src/main/scala/api/queryParams.scala
package api

import sttp.tapir.*

// Filtering parameters
case class ProductFilterParams(
  category: Option[String] = None,
  minPrice: Option[BigDecimal] = None,
  maxPrice: Option[BigDecimal] = None,
  inStock: Option[Boolean] = None,
  search: Option[String] = None,
  tags: Option[String] = None  // comma-separated
):
  def parseTags: List[String] =
    tags.map(_.split(",").toList.map(_.trim)).getOrElse(List.empty)

// Sorting parameters
case class SortParams(
  sortBy: Option[String] = None,    // field name
  sortOrder: Option[String] = None  // asc or desc
):
  def toSortSpec: Option[SortSpec] =
    sortBy.map { field =>
      SortSpec(
        field = field,
        order = sortOrder.map(_.toLowerCase) match
          case Some("desc") => SortOrder.Desc
          case _ => SortOrder.Asc
      )
    }

case class SortSpec(field: String, order: SortOrder)

enum SortOrder:
  case Asc, Desc

// Allowed sort fields (whitelist to prevent SQL injection)
object AllowedSortFields:
  val products = Set("name", "price", "createdAt", "updatedAt", "stock")
  
  def validate(field: String, allowed: Set[String]): Either[String, String] =
    if allowed.contains(field) then Right(field)
    else Left(s"Sort field '$field' is not allowed. Allowed fields: ${allowed.mkString(", ")}")

// Query builder
class ProductQueryBuilder:
  
  def buildQuery(
    filter: ProductFilterParams,
    sort: SortParams,
    pagination: OffsetPaginationParams
  ): ProductQuery =
    val sortSpec = sort.toSortSpec.flatMap { spec =>
      AllowedSortFields.validate(spec.field, AllowedSortFields.products)
        .toOption
        .map(validField => spec.copy(field = validField))
    }
    
    ProductQuery(
      categoryId = filter.category,
      minPrice = filter.minPrice,
      maxPrice = filter.maxPrice,
      inStock = filter.inStock,
      searchTerm = filter.search,
      tags = filter.parseTags,
      sortBy = sortSpec.map(_.field),
      sortOrder = sortSpec.map(_.order),
      offset = pagination.offset,
      limit = pagination.limit
    )

case class ProductQuery(
  categoryId: Option[String],
  minPrice: Option[BigDecimal],
  maxPrice: Option[BigDecimal],
  inStock: Option[Boolean],
  searchTerm: Option[String],
  tags: List[String],
  sortBy: Option[String],
  sortOrder: Option[SortOrder],
  offset: Int,
  limit: Int
)
```

### Full-text Search

```scala
// src/main/scala/api/search.scala
package api

import cats.effect.IO

// Search implementation
case class SearchRequest(
  query: String,
  filters: Map[String, String] = Map.empty,
  highlight: Boolean = false
)

case class SearchResult[A](
  item: A,
  score: Double,
  highlights: Map[String, List[String]] = Map.empty
)

trait SearchService[F[_]]:
  def search(request: SearchRequest): F[List[SearchResult[ProductResponse]]]

// Simple in-memory search implementation
class InMemorySearchService(products: List[Product]) extends SearchService[IO]:
  
  def search(request: SearchRequest): IO[List[SearchResult[ProductResponse]]] =
    IO.pure {
      val query = request.query.toLowerCase
      products
        .filter { p =>
          p.name.toLowerCase.contains(query) ||
          p.description.toLowerCase.contains(query)
        }
        .map { p =>
          val score = calculateScore(p, query)
          SearchResult(
            item = p.toResponse,
            score = score,
            highlights = if request.highlight then
              Map(
                "name" -> List(highlightText(p.name, query)),
                "description" -> List(highlightText(p.description, query))
              )
            else Map.empty
          )
        }
        .sortBy(-_.score)
    }
  
  private def calculateScore(product: Product, query: String): Double =
    val nameMatch = if product.name.toLowerCase.contains(query) then 2.0 else 0.0
    val descMatch = if product.description.toLowerCase.contains(query) then 1.0 else 0.0
    nameMatch + descMatch
  
  private def highlightText(text: String, query: String): String =
    text.replaceAll(
      s"(?i)($query)",
      "<em>$1</em>"
    )
  
  extension (p: Product)
    def toResponse: ProductResponse = ProductResponse(
      id = p.id.value,
      name = p.name,
      description = p.description,
      price = p.price,
      categoryId = p.categoryId.value,
      stock = p.stock,
      createdAt = p.createdAt.toString,
      updatedAt = p.updatedAt.toString
    )
```

---

## HATEOAS

HATEOAS (Hypermedia as the Engine of Application State) ช่วยให้ Client รู้ว่าทำอะไรได้บ้างจาก Response

```scala
// src/main/scala/api/hateoas.scala
package api

import java.util.UUID

// Link definition
case class Link(
  href: String,
  rel: String,
  method: String = "GET",
  title: Option[String] = None
)

// HAL-style response
case class HalResource[A](
  data: A,
  links: Map[String, Link],
  embedded: Map[String, List[Any]] = Map.empty
)

// HATEOAS response for Product
case class ProductHateoasResponse(
  id: UUID,
  name: String,
  description: String,
  price: BigDecimal,
  categoryId: UUID,
  stock: Int,
  createdAt: String,
  updatedAt: String,
  links: Map[String, Link]
)

object HateoasBuilder:
  val baseUrl = "http://api.example.com/api/v1"
  
  def productLinks(productId: UUID, categoryId: UUID): Map[String, Link] =
    Map(
      "self" -> Link(
        href = s"$baseUrl/products/$productId",
        rel = "self",
        title = Some("Product details")
      ),
      "update" -> Link(
        href = s"$baseUrl/products/$productId",
        rel = "update",
        method = "PUT",
        title = Some("Update product")
      ),
      "patch" -> Link(
        href = s"$baseUrl/products/$productId",
        rel = "patch",
        method = "PATCH",
        title = Some("Partial update")
      ),
      "delete" -> Link(
        href = s"$baseUrl/products/$productId",
        rel = "delete",
        method = "DELETE",
        title = Some("Delete product")
      ),
      "category" -> Link(
        href = s"$baseUrl/categories/$categoryId",
        rel = "category",
        title = Some("Product category")
      ),
      "reviews" -> Link(
        href = s"$baseUrl/products/$productId/reviews",
        rel = "reviews",
        title = Some("Product reviews")
      )
    )
  
  def collectionLinks(
    resource: String,
    page: Int,
    pageSize: Int,
    totalPages: Int,
    queryParams: Map[String, String] = Map.empty
  ): Map[String, Link] =
    val baseParams = queryParams ++ Map("pageSize" -> pageSize.toString)
    val selfParams = baseParams + ("page" -> page.toString)
    
    val links = Map.newBuilder[String, Link]
    
    links += "self" -> Link(
      href = buildUrl(resource, selfParams),
      rel = "self"
    )
    
    links += "first" -> Link(
      href = buildUrl(resource, baseParams + ("page" -> "1")),
      rel = "first"
    )
    
    links += "last" -> Link(
      href = buildUrl(resource, baseParams + ("page" -> totalPages.toString)),
      rel = "last"
    )
    
    if page > 1 then
      links += "previous" -> Link(
        href = buildUrl(resource, baseParams + ("page" -> (page - 1).toString)),
        rel = "previous"
      )
    
    if page < totalPages then
      links += "next" -> Link(
        href = buildUrl(resource, baseParams + ("page" -> (page + 1).toString)),
        rel = "next"
      )
    
    links.result()
  
  private def buildUrl(resource: String, params: Map[String, String]): String =
    val queryString = params.map((k, v) => s"$k=$v").mkString("&")
    s"$baseUrl/$resource?$queryString"
```

---

## การกำหนดเวอร์ชัน API

### กลยุทธ์การกำหนดเวอร์ชัน

```scala
// src/main/scala/api/versioning.scala
package api

/**
 * กลยุทธ์การกำหนดเวอร์ชัน API:
 *
 * 1. URL Path Versioning: /api/v1/products, /api/v2/products
 *    - ข้อดี: ชัดเจน, ง่ายต่อการทดสอบ
 *    - ข้อเสีย: URL ยาวขึ้น, ไม่ตรงตาม REST หลักการ
 *
 * 2. Query Parameter: /api/products?version=1
 *    - ข้อดี: ง่าย
 *    - ข้อเสีย: ไม่ค่อย semantic
 *
 * 3. Header Versioning: Accept: application/vnd.api+json;version=1
 *    - ข้อดี: Clean URL, ตรงตาม HTTP spec
 *    - ข้อเสีย: ยากต่อการทดสอบใน browser
 *
 * 4. Media Type Versioning: Accept: application/vnd.myapi.v1+json
 *    - ข้อดี: ถูกต้องที่สุดตาม REST
 *    - ข้อเสีย: ซับซ้อน
 */

// Version handling with type-safe approach
sealed trait ApiVersion:
  def major: Int
  def minor: Int
  def path: String = s"v$major"

case object V1 extends ApiVersion:
  val major = 1
  val minor = 0

case object V2 extends ApiVersion:
  val major = 2
  val minor = 0

// Deprecated endpoint marker
case class DeprecatedEndpoint(
  deprecatedIn: ApiVersion,
  removedIn: ApiVersion,
  alternative: String
)

// V1 Product response (legacy)
case class ProductResponseV1(
  id: String,
  name: String,
  price: Double,     // V1 uses Double
  stock: Int
)

// V2 Product response (current)
case class ProductResponseV2(
  id: String,
  name: String,
  description: String,
  price: BigDecimal,  // V2 uses BigDecimal
  categoryId: String,
  stock: Int,
  metadata: Map[String, String],
  createdAt: String,
  updatedAt: String
)

// Migration adapter
object ProductMigration:
  def v1ToV2(v1: ProductResponseV1): ProductResponseV2 =
    ProductResponseV2(
      id = v1.id,
      name = v1.name,
      description = "",
      price = BigDecimal(v1.price),
      categoryId = "",
      stock = v1.stock,
      metadata = Map.empty,
      createdAt = "",
      updatedAt = ""
    )
  
  def toV1(product: Product): ProductResponseV1 =
    ProductResponseV1(
      id = product.id.value.toString,
      name = product.name,
      price = product.price.toDouble,
      stock = product.stock
    )
```

---

## OpenAPI/Swagger กับ Tapir

### Tapir Endpoint Definitions

```scala
// src/main/scala/api/tapirEndpoints.scala
package api

import sttp.tapir.*
import sttp.tapir.json.circe.*
import sttp.tapir.generic.auto.*
import sttp.tapir.swagger.bundle.SwaggerInterpreter
import io.circe.generic.auto.*
import cats.effect.IO
import org.http4s.HttpRoutes
import sttp.tapir.server.http4s.Http4sServerInterpreter

// Complete Tapir endpoint setup
object TapirApi:
  
  // Authentication
  val bearerAuth = auth.bearer[String]()
  
  // Common error output
  val commonErrors = oneOf[ErrorResponse](
    oneOfVariant(statusCode(sttp.model.StatusCode.BadRequest).and(jsonBody[ErrorResponse])),
    oneOfVariant(statusCode(sttp.model.StatusCode.Unauthorized).and(jsonBody[ErrorResponse])),
    oneOfVariant(statusCode(sttp.model.StatusCode.NotFound).and(jsonBody[ErrorResponse])),
    oneOfVariant(statusCode(sttp.model.StatusCode.InternalServerError).and(jsonBody[ErrorResponse]))
  )
  
  // Product endpoints with full documentation
  val listProductsEndpoint: Endpoint[
    String,
    (Option[Int], Option[Int], Option[String], Option[String]),
    ErrorResponse,
    PaginatedResponse[ProductResponseV2],
    Any
  ] = endpoint
    .name("List Products")
    .description("ดึงรายการสินค้าทั้งหมด พร้อมการกรองและแบ่งหน้า")
    .tag("Products")
    .securityIn(bearerAuth)
    .get
    .in("api" / "v2" / "products")
    .in(
      query[Option[Int]]("page")
        .description("หน้าที่ต้องการ (เริ่มต้น: 1)")
        .example(Some(1))
    )
    .in(
      query[Option[Int]]("pageSize")
        .description("จำนวนรายการต่อหน้า (สูงสุด: 100, ค่าเริ่มต้น: 20)")
        .example(Some(20))
    )
    .in(
      query[Option[String]]("category")
        .description("กรองตามหมวดหมู่ (UUID)")
    )
    .in(
      query[Option[String]]("search")
        .description("ค้นหาจากชื่อหรือคำอธิบาย")
    )
    .out(jsonBody[PaginatedResponse[ProductResponseV2]])
    .errorOut(commonErrors)
  
  val getProductEndpoint = endpoint
    .name("Get Product")
    .description("ดึงข้อมูลสินค้าตาม ID")
    .tag("Products")
    .securityIn(bearerAuth)
    .get
    .in("api" / "v2" / "products" / path[java.util.UUID]("id").description("Product UUID"))
    .out(jsonBody[ProductResponseV2])
    .errorOut(commonErrors)
  
  val createProductEndpoint = endpoint
    .name("Create Product")
    .description("สร้างสินค้าใหม่")
    .tag("Products")
    .securityIn(bearerAuth)
    .post
    .in("api" / "v2" / "products")
    .in(
      jsonBody[CreateProductRequest]
        .description("ข้อมูลสินค้าที่ต้องการสร้าง")
        .example(CreateProductRequest(
          name = "สินค้าตัวอย่าง",
          description = "คำอธิบายสินค้า",
          price = BigDecimal("99.99"),
          categoryId = java.util.UUID.randomUUID(),
          stock = 100
        ))
    )
    .out(statusCode(sttp.model.StatusCode.Created))
    .out(jsonBody[ProductResponseV2])
    .errorOut(commonErrors)

// Swagger UI setup
object SwaggerSetup:
  
  def swaggerRoutes: HttpRoutes[IO] =
    val swaggerEndpoints = SwaggerInterpreter()
      .fromEndpoints[IO](
        List(
          TapirApi.listProductsEndpoint,
          TapirApi.getProductEndpoint,
          TapirApi.createProductEndpoint
        ),
        title = "Product API",
        version = "2.0.0"
      )
    
    Http4sServerInterpreter[IO]().toRoutes(swaggerEndpoints)
```

### Server Implementation

```scala
// src/main/scala/server/Server.scala
package server

import cats.effect.*
import org.http4s.ember.server.EmberServerBuilder
import org.http4s.server.middleware.*
import com.comcast.ip4s.*
import scala.concurrent.duration.*

object Server extends IOApp:
  
  def run(args: List[String]): IO[ExitCode] =
    for
      _      <- IO.println("Starting API server...")
      router <- AppRouter.make
      _      <- buildServer(router).useForever
    yield ExitCode.Success
  
  private def buildServer(router: AppRouter) =
    EmberServerBuilder
      .default[IO]
      .withHost(ipv4"0.0.0.0")
      .withPort(port"8080")
      .withHttpApp(
        withMiddlewares(router.routes.orNotFound)
      )
      .build
  
  private def withMiddlewares(app: cats.data.Kleisli[IO, org.http4s.Request[IO], org.http4s.Response[IO]]) =
    val corsConfig = CORS.policy
      .withAllowOriginAll
      .withAllowMethodsAll
      .withAllowHeadersAll
      .withMaxAge(1.hour)
    
    corsConfig(
      RequestId(
        GZip(
          app
        )
      )
    )
```

---

## ตัวอย่าง API Design แบบสมบูรณ์

### Complete Service Layer

```scala
// src/main/scala/services/ProductService.scala
package services

import cats.effect.IO
import cats.syntax.all.*
import domain.*
import api.*
import java.util.UUID
import java.time.Instant

trait ProductService:
  def listProducts(
    params: OffsetPaginationParams,
    filter: ProductFilterParams,
    sort: SortParams
  ): IO[PaginatedResponse[ProductResponseV2]]
  
  def getProduct(id: UUID): IO[Either[ApiError, ProductResponseV2]]
  
  def createProduct(
    request: CreateProductRequest,
    userId: UserId
  ): IO[Either[ApiError, ProductResponseV2]]
  
  def updateProduct(
    id: UUID,
    request: CreateProductRequest
  ): IO[Either[ApiError, ProductResponseV2]]
  
  def patchProduct(
    id: UUID,
    request: UpdateProductRequest
  ): IO[Either[ApiError, ProductResponseV2]]
  
  def deleteProduct(id: UUID): IO[Either[ApiError, Unit]]

class ProductServiceImpl(
  productRepo: ProductRepository[IO],
  categoryRepo: CategoryRepository[IO],
  eventBus: EventBus[IO]
) extends ProductService:
  
  def listProducts(
    params: OffsetPaginationParams,
    filter: ProductFilterParams,
    sort: SortParams
  ): IO[PaginatedResponse[ProductResponseV2]] =
    productRepo.findWithOffset(params, filter.toDomain).map { result =>
      PaginatedResponse(
        data = result.data.map(_.toResponseV2),
        pagination = result.pagination
      )
    }
  
  def getProduct(id: UUID): IO[Either[ApiError, ProductResponseV2]] =
    productRepo.findById(ProductId(id)).map {
      case Some(product) => Right(product.toResponseV2)
      case None => Left(NotFoundError("Product", id.toString))
    }
  
  def createProduct(
    request: CreateProductRequest,
    userId: UserId
  ): IO[Either[ApiError, ProductResponseV2]] =
    val validation = Validator.validateCreateProduct(request)
    
    if !validation.isValid then
      IO.pure(Left(ValidationError("request", validation.errors.mkString(", "))))
    else
      for
        categoryExists <- categoryRepo.exists(CategoryId(request.categoryId))
        result <- if !categoryExists then
          IO.pure(Left(NotFoundError("Category", request.categoryId.toString)))
        else
          createProductInternal(request, userId).map(Right(_))
      yield result
  
  private def createProductInternal(
    request: CreateProductRequest,
    userId: UserId
  ): IO[ProductResponseV2] =
    val now = Instant.now()
    val product = Product(
      id = ProductId(UUID.randomUUID()),
      name = request.name,
      description = request.description,
      price = request.price,
      categoryId = CategoryId(request.categoryId),
      stock = request.stock,
      createdAt = now,
      updatedAt = now
    )
    
    for
      saved  <- productRepo.save(product)
      _      <- eventBus.publish(ProductCreated(saved.id, userId, now))
    yield saved.toResponseV2
  
  // Extension methods for conversion
  extension (p: Product)
    def toResponseV2: ProductResponseV2 = ProductResponseV2(
      id = p.id.value.toString,
      name = p.name,
      description = p.description,
      price = p.price,
      categoryId = p.categoryId.value.toString,
      stock = p.stock,
      metadata = Map.empty,
      createdAt = p.createdAt.toString,
      updatedAt = p.updatedAt.toString
    )
  
  extension (f: ProductFilterParams)
    def toDomain: ProductFilter = ProductFilter(
      categoryId = f.category.map(id => CategoryId(UUID.fromString(id))),
      inStock = f.inStock
    )
```

### Rate Limiting

```scala
// src/main/scala/middleware/RateLimiting.scala
package middleware

import cats.effect.IO
import cats.effect.Ref
import org.http4s.*
import org.http4s.dsl.io.*
import java.time.Instant
import scala.concurrent.duration.*

case class RateLimitInfo(
  requests: Int,
  windowStart: Instant,
  limit: Int,
  window: FiniteDuration
)

class RateLimiter(
  limit: Int = 100,
  window: FiniteDuration = 1.minute
):
  
  private val store: IO[Ref[IO, Map[String, RateLimitInfo]]] =
    Ref.of[IO, Map[String, RateLimitInfo]](Map.empty)
  
  def middleware(routes: HttpRoutes[IO]): HttpRoutes[IO] =
    HttpRoutes { req =>
      val clientIp = req.remoteAddr.map(_.toString).getOrElse("unknown")
      
      for
        storeRef  <- cats.data.OptionT.liftF(store)
        allowed   <- cats.data.OptionT.liftF(checkAndUpdate(storeRef, clientIp))
        response  <- if allowed then routes(req)
                     else cats.data.OptionT.pure[IO](
                       Response[IO](Status.TooManyRequests)
                         .withHeaders(
                           Header.Raw(ci"X-RateLimit-Limit", limit.toString),
                           Header.Raw(ci"X-RateLimit-Remaining", "0"),
                           Header.Raw(ci"Retry-After", window.toSeconds.toString)
                         )
                     )
      yield response
    }
  
  private def checkAndUpdate(
    storeRef: Ref[IO, Map[String, RateLimitInfo]],
    clientIp: String
  ): IO[Boolean] =
    storeRef.modify { store =>
      val now = Instant.now()
      val info = store.getOrElse(
        clientIp,
        RateLimitInfo(0, now, limit, window)
      )
      
      val windowExpired = now.toEpochMilli - info.windowStart.toEpochMilli > window.toMillis
      
      if windowExpired then
        val newInfo = RateLimitInfo(1, now, limit, window)
        (store.updated(clientIp, newInfo), true)
      else if info.requests < limit then
        val newInfo = info.copy(requests = info.requests + 1)
        (store.updated(clientIp, newInfo), true)
      else
        (store, false)
    }
```

### Complete Application

```scala
// src/main/scala/Main.scala
import cats.effect.*
import cats.effect.std.Console
import org.typelevel.log4cats.slf4j.Slf4jLogger
import server.*

object Main extends IOApp:
  
  given logger: org.typelevel.log4cats.Logger[IO] =
    Slf4jLogger.getLogger[IO]
  
  def run(args: List[String]): IO[ExitCode] =
    for
      _ <- logger.info("Starting Scala REST API server")
      _ <- logger.info("API Documentation: http://localhost:8080/docs")
      _ <- Server.run(args)
    yield ExitCode.Success
```

---

## สรุป

การออกแบบ REST API ที่ดีใน Scala 3 ต้องคำนึงถึง:

1. **Naming Conventions**: ใช้ noun พหูพจน์สำหรับ resources
2. **HTTP Methods**: ใช้ method ที่เหมาะสมตาม semantics
3. **Status Codes**: ส่ง status code ที่ถูกต้องและมีความหมาย
4. **Pagination**: เลือก offset หรือ cursor-based ตามความเหมาะสม
5. **Filtering/Sorting**: ออกแบบ query parameters ที่ยืดหยุ่น
6. **HATEOAS**: ให้ links ที่จำเป็นใน response
7. **Versioning**: วางแผนการ versioning ตั้งแต่ต้น
8. **Documentation**: ใช้ Tapir เพื่อ generate OpenAPI spec อัตโนมัติ
9. **Error Handling**: ส่ง error ที่ชัดเจนและสม่ำเสมอ
10. **Rate Limiting**: ป้องกัน abuse ด้วย rate limiting

---

*[← ส่วนที่ 74: Testing Strategies](part-74-testing-strategies.md) | [ส่วนที่ 76: Database Patterns →](part-76-database-patterns.md)*
