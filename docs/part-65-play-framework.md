# Part 65: Play Framework - เฟรมเวิร์กสำหรับ Web Application

## สารบัญ

1. [แนะนำ Play Framework](#1-แนะนำ-play-framework)
2. [สถาปัตยกรรม MVC](#2-สถาปัตยกรรม-mvc)
3. [การตั้งค่าโปรเจกต์](#3-การตั้งค่าโปรเจกต์)
4. [Routes Configuration](#4-routes-configuration)
5. [Controllers และ Actions](#5-controllers-และ-actions)
6. [Play JSON](#6-play-json)
7. [Forms และ Validation](#7-forms-และ-validation)
8. [WebSocket Support](#8-websocket-support)
9. [Database Integration ด้วย Slick](#9-database-integration-ด้วย-slick)
10. [Complete REST API Example](#10-complete-rest-api-example)
11. [สรุป](#11-สรุป)

---

## 1. แนะนำ Play Framework

Play Framework คือ web application framework สำหรับ Scala (และ Java) ที่มีคุณสมบัติเด่น:

- **Reactive by design** - ออกแบบมาเพื่อ non-blocking I/O
- **Hot reload** - แก้โค้ดแล้ว reload ทันทีโดยไม่ต้อง restart
- **Type-safe** - ตรวจสอบ routing และ templates ที่ compile time
- **RESTful** - รองรับ REST API อย่างสมบูรณ์
- **Stateless** - ไม่มี session state บน server โดย default

### ทำไมต้องใช้ Play?

```
Traditional Web Framework:
Client -> Server (blocking thread) -> Database -> Response
         ┌──────────────────────────────────────────┐
         │ Thread blocked ขณะรอ DB response           │
         └──────────────────────────────────────────┘

Play Framework (Reactive):
Client -> Server (non-blocking) -> Database -> Response
         ┌──────────────────────────────────────────┐
         │ Thread ว่าง รับ request อื่นได้ทันที        │
         └──────────────────────────────────────────┘
```

### เปรียบเทียบกับ Frameworks อื่น

| Feature | Play | Spring Boot | Akka HTTP |
|---------|------|-------------|-----------|
| Language | Scala/Java | Java/Kotlin | Scala |
| Architecture | MVC | Various | Actor-based |
| Hot Reload | Yes | Partial | No |
| Learning Curve | Medium | Medium | High |
| Performance | High | Medium | Very High |

---

## 2. สถาปัตยกรรม MVC

Play ใช้สถาปัตยกรรม Model-View-Controller:

```
┌─────────────────────────────────────────────────────────┐
│                    Play Application                       │
│                                                           │
│  Request → Router → Controller → Model → Controller      │
│                          ↓                    ↓          │
│                       Service              View/JSON      │
│                          ↓                    ↓          │
│                      Database           Response          │
└─────────────────────────────────────────────────────────┘
```

### โครงสร้างโปรเจกต์

```
my-play-app/
├── app/
│   ├── controllers/         # Controller classes
│   │   ├── HomeController.scala
│   │   └── ApiController.scala
│   ├── models/              # Model/Domain classes
│   │   ├── User.scala
│   │   └── Product.scala
│   ├── services/            # Business logic
│   │   └── UserService.scala
│   └── views/               # Twirl templates
│       ├── index.scala.html
│       └── layout.scala.html
├── conf/
│   ├── application.conf     # Configuration
│   └── routes               # Route definitions
├── public/                  # Static assets
│   ├── css/
│   ├── js/
│   └── images/
├── test/                    # Test files
└── build.sbt                # Build configuration
```

---

## 3. การตั้งค่าโปรเจกต์

### build.sbt

```scala
// build.sbt
name := "scala-play-app"
version := "1.0.0"
scalaVersion := "3.3.1"

lazy val root = (project in file("."))
  .enablePlugins(PlayScala)
  .settings(
    libraryDependencies ++= Seq(
      // Play framework dependencies
      guice,                              // Dependency injection
      ws,                                 // HTTP client
      "com.typesafe.play" %% "play-json" % "2.10.0",
      
      // Database
      "com.typesafe.slick" %% "slick" % "3.4.1",
      "com.typesafe.slick" %% "slick-hikaricp" % "3.4.1",
      "org.playframework" %% "play-slick" % "5.1.0",
      "com.h2database" % "h2" % "2.2.224",
      
      // Testing
      "org.scalatestplus.play" %% "scalatestplus-play" % "7.0.0" % Test,
      
      // Logging
      "ch.qos.logback" % "logback-classic" % "1.4.11"
    )
  )
```

### plugins.sbt

```scala
// project/plugins.sbt
addSbtPlugin("com.typesafe.play" % "sbt-plugin" % "2.9.0")
addSbtPlugin("org.foundweekends" % "sbt-bintray" % "0.6.2")
```

### application.conf

```hocon
# conf/application.conf
play.http.secret.key = "your-secret-key-change-in-production"
play.http.secret.key = ${?APPLICATION_SECRET}

# Database configuration
slick.dbs.default {
  profile = "slick.jdbc.H2Profile$"
  db {
    driver = "org.h2.Driver"
    url = "jdbc:h2:mem:play;DB_CLOSE_DELAY=-1"
    user = "sa"
    password = ""
  }
}

# HTTP configuration
play.http.parser.maxMemoryBuffer = 1MB

# Allowed hosts (production should list specific hosts)
play.filters.hosts {
  allowed = ["localhost", "localhost:9000", ".example.com"]
}

# CORS configuration
play.filters.cors {
  allowedOrigins = ["http://localhost:3000"]
  allowedHttpMethods = ["GET", "POST", "PUT", "DELETE"]
  allowedHttpHeaders = ["Accept", "Content-Type", "Authorization"]
}

# Custom application settings
app {
  name = "Scala Play App"
  version = "1.0.0"
  max.items.per.page = 20
}
```

---

## 4. Routes Configuration

Routes file กำหนด URL patterns และ mapping ไปยัง Controller actions

### conf/routes

```
# conf/routes
# HTTP Method  URI Pattern           Controller#Action

# หน้าหลัก
GET     /                          controllers.HomeController.index()
GET     /about                     controllers.HomeController.about()

# User API
GET     /api/users                 controllers.UserController.list(page: Int ?= 1, size: Int ?= 20)
POST    /api/users                 controllers.UserController.create()
GET     /api/users/:id             controllers.UserController.getById(id: Long)
PUT     /api/users/:id             controllers.UserController.update(id: Long)
DELETE  /api/users/:id             controllers.UserController.delete(id: Long)

# Product API with optional query params
GET     /api/products              controllers.ProductController.search(q: Option[String], category: Option[String])
POST    /api/products              controllers.ProductController.create()
GET     /api/products/:id          controllers.ProductController.getById(id: Long)
PUT     /api/products/:id          controllers.ProductController.update(id: Long)
DELETE  /api/products/:id          controllers.ProductController.delete(id: Long)

# WebSocket
GET     /ws/chat                   controllers.ChatController.socket()

# Static assets
GET     /assets/*file              controllers.Assets.versioned(path="/public", file: Asset)
```

### การใช้ Path Variables

```scala
// Controller รับ path variable
def getById(id: Long) = Action.async {
  userService.findById(id).map {
    case Some(user) => Ok(Json.toJson(user))
    case None => NotFound(Json.obj("error" -> s"User $id not found"))
  }
}
```

### Regex Constraints ใน Routes

```
# กำหนด constraint ว่า id ต้องเป็นตัวเลข
GET     /api/users/$id<[0-9]+>     controllers.UserController.getById(id: Long)

# กำหนด constraint สำหรับ slug
GET     /posts/$slug<[a-z0-9-]+>   controllers.PostController.getBySlug(slug: String)
```

---

## 5. Controllers และ Actions

### Basic Controller

```scala
// app/controllers/HomeController.scala
package controllers

import javax.inject.*
import play.api.*
import play.api.mvc.*
import play.api.libs.json.*

@Singleton
class HomeController @Inject() (
  val controllerComponents: ControllerComponents
) extends BaseController:

  // Synchronous action
  def index() = Action {
    Ok(views.html.index("Welcome to Play!"))
  }

  // Action returning JSON
  def healthCheck() = Action {
    Ok(Json.obj(
      "status" -> "healthy",
      "timestamp" -> System.currentTimeMillis()
    ))
  }

  // Action with request body
  def echo() = Action(parse.json) { request =>
    Ok(request.body)
  }
```

### Async Controller

```scala
// app/controllers/UserController.scala
package controllers

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.mvc.*
import play.api.libs.json.*
import models.*
import services.UserService

@Singleton
class UserController @Inject() (
  userService: UserService,
  val controllerComponents: ControllerComponents
)(using ExecutionContext) extends BaseController:

  // List users with pagination
  def list(page: Int, size: Int) = Action.async {
    userService.findAll(page, size).map { users =>
      Ok(Json.toJson(users))
    }
  }

  // Get user by ID
  def getById(id: Long) = Action.async {
    userService.findById(id).map {
      case Some(user) => Ok(Json.toJson(user))
      case None       => NotFound(Json.obj(
        "error" -> s"User with id $id not found"
      ))
    }
  }

  // Create user
  def create() = Action.async(parse.json) { request =>
    request.body.validate[CreateUserRequest] match
      case JsSuccess(req, _) =>
        userService.create(req).map { user =>
          Created(Json.toJson(user))
            .withHeaders("Location" -> s"/api/users/${user.id}")
        }.recover {
          case e: IllegalArgumentException =>
            BadRequest(Json.obj("error" -> e.getMessage))
        }
      case JsError(errors) =>
        Future.successful(BadRequest(Json.obj(
          "error" -> "Invalid request body",
          "details" -> JsError.toJson(errors)
        )))
  }

  // Update user
  def update(id: Long) = Action.async(parse.json) { request =>
    request.body.validate[UpdateUserRequest] match
      case JsSuccess(req, _) =>
        userService.update(id, req).map {
          case Some(user) => Ok(Json.toJson(user))
          case None       => NotFound(Json.obj("error" -> s"User $id not found"))
        }
      case JsError(errors) =>
        Future.successful(BadRequest(Json.obj(
          "error" -> "Invalid request body",
          "details" -> JsError.toJson(errors)
        )))
  }

  // Delete user
  def delete(id: Long) = Action.async {
    userService.delete(id).map {
      case true  => NoContent
      case false => NotFound(Json.obj("error" -> s"User $id not found"))
    }
  }
```

### Action Composition

```scala
// app/actions/AuthAction.scala
package actions

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.mvc.*
import services.AuthService

// Request ที่มีข้อมูล user แล้ว
case class AuthenticatedRequest[A](
  user: User,
  request: Request[A]
) extends WrappedRequest[A](request)

// Action ที่ตรวจสอบ authentication
class AuthAction @Inject() (
  authService: AuthService,
  parser: BodyParsers.Default
)(using ExecutionContext) extends ActionBuilder[AuthenticatedRequest, AnyContent]:

  override def parser: BodyParser[AnyContent] = parser
  
  override protected def executionContext: ExecutionContext = summon[ExecutionContext]

  override def invokeBlock[A](
    request: Request[A],
    block: AuthenticatedRequest[A] => Future[Result]
  ): Future[Result] =
    request.headers.get("Authorization") match
      case Some(token) if token.startsWith("Bearer ") =>
        val bearerToken = token.drop(7)
        authService.validateToken(bearerToken).flatMap {
          case Some(user) =>
            block(AuthenticatedRequest(user, request))
          case None =>
            Future.successful(Results.Unauthorized(Json.obj(
              "error" -> "Invalid or expired token"
            )))
        }
      case _ =>
        Future.successful(Results.Unauthorized(Json.obj(
          "error" -> "Missing Authorization header"
        )))
```

### ใช้ AuthAction

```scala
// app/controllers/SecureController.scala
@Singleton
class SecureController @Inject() (
  authAction: AuthAction,
  val controllerComponents: ControllerComponents
)(using ExecutionContext) extends BaseController:

  // Action นี้ต้องการ authentication
  def profile() = authAction.async { request =>
    // request.user มีข้อมูล authenticated user
    Future.successful(Ok(Json.toJson(request.user)))
  }

  def updateProfile() = authAction.async(parse.json) { request =>
    // เข้าถึง user ที่ login อยู่
    val userId = request.user.id
    // ...
    Future.successful(Ok(Json.obj("message" -> "Profile updated")))
  }
```

### Error Handling

```scala
// app/ErrorHandler.scala
package app

import javax.inject.*
import play.api.*
import play.api.http.DefaultHttpErrorHandler
import play.api.mvc.*
import play.api.mvc.Results.*
import play.api.routing.Router
import scala.concurrent.*
import play.api.libs.json.Json

@Singleton
class ErrorHandler @Inject() (
  env: Environment,
  config: Configuration,
  sourceMapper: OptionalSourceMapper,
  router: Provider[Router]
) extends DefaultHttpErrorHandler(env, config, sourceMapper, router):

  override def onClientError(
    request: RequestHeader,
    statusCode: Int,
    message: String
  ): Future[Result] =
    Future.successful(
      Status(statusCode)(Json.obj(
        "error" -> message,
        "status" -> statusCode
      ))
    )

  override def onServerError(
    request: RequestHeader,
    exception: Throwable
  ): Future[Result] =
    Future.successful(
      InternalServerError(Json.obj(
        "error" -> "Internal server error",
        "message" -> exception.getMessage
      ))
    )
```

---

## 6. Play JSON

Play JSON เป็น library สำหรับ JSON parsing และ serialization

### Models และ JSON Format

```scala
// app/models/User.scala
package models

import play.api.libs.json.*
import play.api.libs.functional.syntax.*
import java.time.Instant

// Domain model
case class User(
  id: Long,
  email: String,
  name: String,
  role: UserRole,
  createdAt: Instant,
  isActive: Boolean = true
)

enum UserRole:
  case Admin, Editor, Viewer

// Request models
case class CreateUserRequest(
  email: String,
  name: String,
  role: UserRole = UserRole.Viewer
)

case class UpdateUserRequest(
  name: Option[String],
  role: Option[UserRole]
)

// JSON formats
object User:
  // Format สำหรับ UserRole enum
  given Format[UserRole] = Format(
    Reads {
      case JsString("admin")  => JsSuccess(UserRole.Admin)
      case JsString("editor") => JsSuccess(UserRole.Editor)
      case JsString("viewer") => JsSuccess(UserRole.Viewer)
      case other => JsError(s"Unknown role: $other")
    },
    Writes {
      case UserRole.Admin  => JsString("admin")
      case UserRole.Editor => JsString("editor")
      case UserRole.Viewer => JsString("viewer")
    }
  )

  // Format สำหรับ Instant
  given Format[Instant] = Format(
    Reads(_.validate[Long].map(Instant.ofEpochMilli)),
    Writes(i => JsNumber(i.toEpochMilli))
  )

  // Auto-derive format จาก case class
  given OFormat[User] = Json.format[User]

  // Formats for request models
  given OFormat[CreateUserRequest] = Json.format[CreateUserRequest]
  given OFormat[UpdateUserRequest] = Json.format[UpdateUserRequest]
```

### Custom JSON Reads/Writes

```scala
// Custom Reads ที่มี validation
val emailReads: Reads[String] = Reads[String] { json =>
  json.validate[String].flatMap { email =>
    if email.contains("@") then JsSuccess(email)
    else JsError("Invalid email format")
  }
}

// Custom Writes ที่ transform data
val userSummaryWrites: Writes[User] = Writes[User] { user =>
  Json.obj(
    "id"    -> user.id,
    "name"  -> user.name,
    "email" -> user.email
    // ไม่แสดง role และ createdAt
  )
}

// Combining Reads ด้วย functional syntax
val createUserReads: Reads[CreateUserRequest] = (
  (__ \ "email").read[String](emailReads) and
  (__ \ "name").read[String](Reads.minLength[String](2)) and
  (__ \ "role").readNullable[UserRole].map(_.getOrElse(UserRole.Viewer))
)(CreateUserRequest.apply)
```

### JSON Transformation

```scala
// Transform JSON
val transformer = (__ \ "user").json.pick  // extract nested object
val addField = __.json.update(
  (__ \ "timestamp").json.put(JsNumber(System.currentTimeMillis()))
)

// Prune sensitive fields
val removeSensitive = (__ \ "password").json.prune

// Apply transformations
val originalJson = Json.parse("""{"user": {"name": "Alice", "password": "secret"}}""")
val result = originalJson.transform(
  (__ \ "user").json.pick andThen
  (__ \ "password").json.prune
)
```

### Pagination Response

```scala
// Generic pagination wrapper
case class Page[T](
  items: Seq[T],
  page: Int,
  size: Int,
  total: Long
):
  def totalPages: Int = Math.ceil(total.toDouble / size).toInt
  def hasNext: Boolean = page < totalPages
  def hasPrev: Boolean = page > 1

object Page:
  given [T: Writes]: OWrites[Page[T]] = OWrites { page =>
    Json.obj(
      "items"      -> Json.toJson(page.items),
      "page"       -> page.page,
      "size"       -> page.size,
      "total"      -> page.total,
      "totalPages" -> page.totalPages,
      "hasNext"    -> page.hasNext,
      "hasPrev"    -> page.hasPrev
    )
  }
```

---

## 7. Forms และ Validation

### Play Forms

```scala
// app/forms/UserForms.scala
package forms

import play.api.data.*
import play.api.data.Forms.*
import play.api.data.validation.*
import models.*

object UserForms:

  // Custom validators
  val validEmail: Constraint[String] = Constraint("email.valid") { email =>
    if email.matches("""^[^@]+@[^@]+\.[^@]+$""") then Valid
    else Invalid(ValidationError("Invalid email address"))
  }

  val validPassword: Constraint[String] = Constraint("password.valid") { password =>
    val errors = List(
      Option.when(password.length < 8)("Password must be at least 8 characters"),
      Option.when(!password.exists(_.isUpper))("Password must contain uppercase letter"),
      Option.when(!password.exists(_.isDigit))("Password must contain a digit")
    ).flatten
    
    if errors.isEmpty then Valid
    else Invalid(errors.map(ValidationError(_)))
  }

  // Form definition
  val registrationForm: Form[RegistrationData] = Form(
    mapping(
      "email"     -> email.verifying(validEmail),
      "name"      -> nonEmptyText(minLength = 2, maxLength = 100),
      "password"  -> text.verifying(validPassword),
      "birthYear" -> number(min = 1900, max = 2024),
      "role"      -> optional(text).transform[UserRole](
        s => UserRole.valueOf(s.getOrElse("Viewer")),
        r => Some(r.toString.toLowerCase)
      )
    )(RegistrationData.apply)(r => Some((r.email, r.name, r.password, r.birthYear, r.role)))
  )

case class RegistrationData(
  email: String,
  name: String,
  password: String,
  birthYear: Int,
  role: UserRole
)
```

### ใช้ Forms ใน Controller

```scala
// app/controllers/RegistrationController.scala
@Singleton
class RegistrationController @Inject() (
  val controllerComponents: ControllerComponents,
  userService: UserService
)(using ExecutionContext) extends BaseController:

  // Handle GET - แสดง form
  def showForm() = Action {
    Ok(views.html.registration(UserForms.registrationForm))
  }

  // Handle POST - process form submission
  def submitForm() = Action.async { implicit request =>
    UserForms.registrationForm.bindFromRequest().fold(
      // Form มี errors
      formWithErrors =>
        Future.successful(
          BadRequest(views.html.registration(formWithErrors))
        ),
      // Form valid
      registrationData =>
        userService.register(registrationData).map { user =>
          Redirect(routes.HomeController.index())
            .flashing("success" -> s"Welcome, ${user.name}!")
        }.recover {
          case e: EmailAlreadyExistsException =>
            BadRequest(views.html.registration(
              UserForms.registrationForm.withError("email", "Email already exists")
            ))
        }
    )
  }
```

### JSON Validation ใน API

```scala
// Validate JSON body โดยตรง
def createUserApi() = Action.async(parse.json) { request =>
  val validation = for
    email  <- (request.body \ "email").validate[String]
    name   <- (request.body \ "name").validate[String]
    _      <- if email.contains("@") then JsSuccess(()) 
              else JsError("Invalid email")
    _      <- if name.length >= 2 then JsSuccess(())
              else JsError("Name too short")
  yield (email, name)

  validation match
    case JsSuccess((email, name), _) =>
      userService.create(email, name).map(user => Created(Json.toJson(user)))
    case JsError(errors) =>
      Future.successful(BadRequest(Json.obj(
        "errors" -> JsError.toJson(errors)
      )))
}
```

---

## 8. WebSocket Support

### Basic WebSocket

```scala
// app/controllers/ChatController.scala
package controllers

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.*
import play.api.mvc.*
import play.api.libs.streams.ActorFlow
import play.api.libs.json.*
import org.apache.pekko.actor.*
import org.apache.pekko.stream.Materializer

@Singleton
class ChatController @Inject() (
  val controllerComponents: ControllerComponents
)(using
  ExecutionContext,
  Materializer,
  ActorSystem
) extends BaseController:

  // WebSocket endpoint
  def socket(): WebSocket = WebSocket.accept[JsValue, JsValue] { request =>
    ActorFlow.actorRef { out =>
      ChatActor.props(out)
    }
  }

// Chat Actor
object ChatActor:
  def props(out: ActorRef): Props = Props(ChatActor(out))

class ChatActor(out: ActorRef) extends Actor:
  override def receive: Receive =
    case msg: JsValue =>
      val text = (msg \ "text").as[String]
      val response = Json.obj(
        "type"    -> "message",
        "text"    -> text,
        "echo"    -> s"Echo: $text",
        "time"    -> System.currentTimeMillis()
      )
      out ! response
    
    case other =>
      out ! Json.obj("error" -> s"Unknown message: $other")
```

### WebSocket ที่รองรับ Broadcasting

```scala
// app/actors/RoomActor.scala
object RoomActor:
  case class Join(userId: String, actorRef: ActorRef)
  case class Leave(userId: String)
  case class Broadcast(message: JsValue, senderId: String)
  case class DirectMessage(message: JsValue, recipientId: String)

class RoomActor extends Actor:
  private var members: Map[String, ActorRef] = Map.empty

  override def receive: Receive =
    case Join(userId, actorRef) =>
      members = members + (userId -> actorRef)
      broadcastSystemMessage(s"$userId joined the room")
    
    case Leave(userId) =>
      members = members - userId
      broadcastSystemMessage(s"$userId left the room")
    
    case Broadcast(message, senderId) =>
      members.values.foreach(_ ! message)
    
    case DirectMessage(message, recipientId) =>
      members.get(recipientId) match
        case Some(ref) => ref ! message
        case None      => sender() ! Json.obj("error" -> s"User $recipientId not found")

  private def broadcastSystemMessage(text: String): Unit =
    val msg = Json.obj(
      "type" -> "system",
      "text" -> text
    )
    members.values.foreach(_ ! msg)
```

### WebSocket ด้วย Flow

```scala
// WebSocket ที่ใช้ Flow (ไม่ต้องใช้ Actor)
def socketFlow(): WebSocket = WebSocket.accept[String, String] { request =>
  import org.apache.pekko.stream.scaladsl.*
  
  Flow[String]
    .map(msg => s"Echo: $msg")
    .keepAlive(30.seconds, () => "ping")
}

// WebSocket ที่มี authentication
def authenticatedSocket(): WebSocket = WebSocket.acceptOrResult[JsValue, JsValue] { request =>
  request.headers.get("Authorization") match
    case Some(token) =>
      authService.validateToken(token).map {
        case Some(user) =>
          Right(ActorFlow.actorRef(out => ChatActor.props(out, user)))
        case None =>
          Left(Forbidden(Json.obj("error" -> "Invalid token")))
      }
    case None =>
      Future.successful(Left(Unauthorized(Json.obj("error" -> "No token"))))
}
```

---

## 9. Database Integration ด้วย Slick

### Slick Models

```scala
// app/models/db/Tables.scala
package models.db

import slick.jdbc.H2Profile.api.*

// User table
class Users(tag: Tag) extends Table[UserRow](tag, "USERS"):
  def id        = column[Long]("ID", O.PrimaryKey, O.AutoInc)
  def email     = column[String]("EMAIL", O.Unique)
  def name      = column[String]("NAME")
  def role      = column[String]("ROLE", O.Default("viewer"))
  def createdAt = column[Long]("CREATED_AT")
  def isActive  = column[Boolean]("IS_ACTIVE", O.Default(true))

  def * = (id, email, name, role, createdAt, isActive) <> 
    (UserRow.apply, UserRow.unapply)

case class UserRow(
  id: Long = 0,
  email: String,
  name: String,
  role: String = "viewer",
  createdAt: Long = System.currentTimeMillis(),
  isActive: Boolean = true
)

object Tables:
  val users = TableQuery[Users]
```

### Repository Pattern

```scala
// app/repositories/UserRepository.scala
package repositories

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.db.slick.DatabaseConfigProvider
import slick.jdbc.JdbcProfile
import models.db.*
import models.*

@Singleton
class UserRepository @Inject() (
  dbConfigProvider: DatabaseConfigProvider
)(using ExecutionContext):

  private val dbConfig = dbConfigProvider.get[JdbcProfile]
  
  import dbConfig.*
  import profile.api.*

  private val users = Tables.users

  // Insert user
  def insert(user: UserRow): Future[UserRow] =
    val query = (users returning users.map(_.id) into { (row, id) =>
      row.copy(id = id)
    }) += user
    db.run(query)

  // Find by ID
  def findById(id: Long): Future[Option[UserRow]] =
    db.run(users.filter(_.id === id).result.headOption)

  // Find by email
  def findByEmail(email: String): Future[Option[UserRow]] =
    db.run(users.filter(_.email === email).result.headOption)

  // Find all with pagination
  def findAll(page: Int, size: Int): Future[Seq[UserRow]] =
    val offset = (page - 1) * size
    db.run(
      users
        .filter(_.isActive === true)
        .sortBy(_.id.desc)
        .drop(offset)
        .take(size)
        .result
    )

  // Count all active
  def countAll: Future[Int] =
    db.run(users.filter(_.isActive === true).length.result)

  // Update
  def update(id: Long, name: String, role: String): Future[Int] =
    db.run(
      users
        .filter(_.id === id)
        .map(u => (u.name, u.role))
        .update((name, role))
    )

  // Soft delete
  def softDelete(id: Long): Future[Int] =
    db.run(
      users
        .filter(_.id === id)
        .map(_.isActive)
        .update(false)
    )

  // Transaction: update + log
  def updateWithAudit(id: Long, name: String): Future[Unit] =
    val updateAction = users
      .filter(_.id === id)
      .map(_.name)
      .update(name)
    
    val logAction = // เพิ่ม audit log
      sqlu"INSERT INTO AUDIT_LOG(user_id, action) VALUES($id, 'UPDATE')"
    
    db.run((updateAction >> logAction).transactionally).map(_ => ())
```

---

## 10. Complete REST API Example

### User Service

```scala
// app/services/UserService.scala
package services

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import models.*
import repositories.UserRepository

@Singleton
class UserService @Inject() (
  userRepo: UserRepository
)(using ExecutionContext):

  def findAll(page: Int, size: Int): Future[Page[User]] =
    for
      rows  <- userRepo.findAll(page, size)
      total <- userRepo.countAll
    yield Page(
      items = rows.map(toUser),
      page  = page,
      size  = size,
      total = total
    )

  def findById(id: Long): Future[Option[User]] =
    userRepo.findById(id).map(_.map(toUser))

  def create(req: CreateUserRequest): Future[User] =
    userRepo.findByEmail(req.email).flatMap {
      case Some(_) =>
        Future.failed(new IllegalArgumentException(s"Email ${req.email} already exists"))
      case None =>
        val row = UserRow(
          email = req.email,
          name  = req.name,
          role  = req.role.toString.toLowerCase
        )
        userRepo.insert(row).map(toUser)
    }

  def update(id: Long, req: UpdateUserRequest): Future[Option[User]] =
    userRepo.findById(id).flatMap {
      case None => Future.successful(None)
      case Some(existing) =>
        val newName = req.name.getOrElse(existing.name)
        val newRole = req.role.map(_.toString.toLowerCase).getOrElse(existing.role)
        userRepo.update(id, newName, newRole).flatMap { _ =>
          userRepo.findById(id).map(_.map(toUser))
        }
    }

  def delete(id: Long): Future[Boolean] =
    userRepo.findById(id).flatMap {
      case None    => Future.successful(false)
      case Some(_) => userRepo.softDelete(id).map(_ > 0)
    }

  private def toUser(row: UserRow): User =
    User(
      id        = row.id,
      email     = row.email,
      name      = row.name,
      role      = UserRole.valueOf(row.role.capitalize),
      createdAt = java.time.Instant.ofEpochMilli(row.createdAt),
      isActive  = row.isActive
    )
```

### Complete Controller with all endpoints

```scala
// app/controllers/UserController.scala
package controllers

import javax.inject.*
import scala.concurrent.{ExecutionContext, Future}
import play.api.mvc.*
import play.api.libs.json.*
import models.*
import services.UserService

@Singleton
class UserController @Inject() (
  userService: UserService,
  val controllerComponents: ControllerComponents
)(using ExecutionContext) extends BaseController:

  def list(page: Int = 1, size: Int = 20) = Action.async {
    val safePage = Math.max(1, page)
    val safeSize = Math.min(100, Math.max(1, size))
    
    userService.findAll(safePage, safeSize).map { pageResult =>
      Ok(Json.obj(
        "data"  -> Json.toJson(pageResult),
        "links" -> Json.obj(
          "self" -> s"/api/users?page=$safePage&size=$safeSize",
          "next" -> Option.when(pageResult.hasNext)(s"/api/users?page=${safePage + 1}&size=$safeSize"),
          "prev" -> Option.when(pageResult.hasPrev)(s"/api/users?page=${safePage - 1}&size=$safeSize")
        )
      ))
    }.recover {
      case e => InternalServerError(Json.obj("error" -> e.getMessage))
    }
  }

  def getById(id: Long) = Action.async {
    userService.findById(id).map {
      case Some(user) => Ok(Json.toJson(user))
      case None => NotFound(Json.obj(
        "error" -> s"User $id not found",
        "code"  -> "USER_NOT_FOUND"
      ))
    }
  }

  def create() = Action.async(parse.json) { request =>
    request.body.validate[CreateUserRequest] match
      case JsSuccess(req, _) =>
        userService.create(req).map { user =>
          Created(Json.toJson(user))
            .withHeaders("Location" -> routes.UserController.getById(user.id).url)
        }.recover {
          case e: IllegalArgumentException =>
            Conflict(Json.obj(
              "error" -> e.getMessage,
              "code"  -> "EMAIL_EXISTS"
            ))
        }
      case JsError(errors) =>
        Future.successful(UnprocessableEntity(Json.obj(
          "error"  -> "Validation failed",
          "errors" -> JsError.toJson(errors)
        )))
  }

  def update(id: Long) = Action.async(parse.json) { request =>
    request.body.validate[UpdateUserRequest] match
      case JsSuccess(req, _) =>
        userService.update(id, req).map {
          case Some(user) => Ok(Json.toJson(user))
          case None => NotFound(Json.obj("error" -> s"User $id not found"))
        }
      case JsError(errors) =>
        Future.successful(UnprocessableEntity(Json.obj(
          "error"  -> "Validation failed",
          "errors" -> JsError.toJson(errors)
        )))
  }

  def delete(id: Long) = Action.async {
    userService.delete(id).map {
      case true  => NoContent
      case false => NotFound(Json.obj("error" -> s"User $id not found"))
    }
  }
```

### Module Setup (Dependency Injection)

```scala
// app/Module.scala
import com.google.inject.AbstractModule
import services.*
import repositories.*

class Module extends AbstractModule:
  override def configure(): Unit =
    bind(classOf[UserRepository]).asEagerSingleton()
    bind(classOf[UserService]).asEagerSingleton()
    // เพิ่ม bindings อื่นๆ ที่นี่
```

### Integration Tests

```scala
// test/controllers/UserControllerSpec.scala
package controllers

import org.scalatestplus.play.*
import org.scalatestplus.play.guice.*
import play.api.test.*
import play.api.test.Helpers.*
import play.api.libs.json.*

class UserControllerSpec extends PlaySpec with GuiceOneAppPerTest:

  "UserController GET /api/users" should {
    "return empty list initially" in {
      val request = FakeRequest(GET, "/api/users")
      val result  = route(app, request).get
      
      status(result) mustBe OK
      contentType(result) mustBe Some("application/json")
      
      val json = contentAsJson(result)
      (json \ "data" \ "items").as[Seq[JsValue]] mustBe empty
    }
  }

  "UserController POST /api/users" should {
    "create a new user" in {
      val body = Json.obj(
        "email" -> "test@example.com",
        "name"  -> "Test User"
      )
      val request = FakeRequest(POST, "/api/users")
        .withJsonBody(body)
      val result = route(app, request).get
      
      status(result) mustBe CREATED
      val json = contentAsJson(result)
      (json \ "email").as[String] mustBe "test@example.com"
    }

    "return 422 for invalid email" in {
      val body = Json.obj(
        "email" -> "not-an-email",
        "name"  -> "Test"
      )
      val request = FakeRequest(POST, "/api/users")
        .withJsonBody(body)
      val result = route(app, request).get
      
      status(result) mustBe UNPROCESSABLE_ENTITY
    }
  }
```

---

## 11. สรุป

Play Framework เป็น web framework ที่มีพลังสูงสำหรับ Scala:

### สิ่งที่เรียนรู้

| หัวข้อ | สรุป |
|--------|------|
| Architecture | MVC pattern กับ reactive design |
| Routing | Type-safe routes พร้อม constraints |
| Controllers | Action composition และ async handling |
| JSON | Play JSON สำหรับ serialization/deserialization |
| Forms | Built-in validation framework |
| WebSocket | Real-time communication ด้วย Actor |
| Database | Integration กับ Slick ORM |

### Best Practices

1. **ใช้ Dependency Injection** - Guice สำหรับ loose coupling
2. **Async everywhere** - ใช้ `Action.async` เสมอสำหรับ I/O operations
3. **Error handling** - Global error handler สำหรับ consistent responses
4. **Validation** - Validate input ทั้ง Forms และ JSON
5. **Testing** - Integration tests ด้วย `GuiceOneAppPerTest`

### ขั้นตอนต่อไป

- ศึกษา Slick อย่างลึกซึ้งใน Part 66
- ดู Play documentation ที่ https://www.playframework.com/

---

*[← Part 64: Monix](part-64-monix.md) | [Part 66: Slick Database →](part-66-slick.md)*
