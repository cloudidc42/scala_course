# Part 25: Play Framework

## สารบัญ
1. [Play Framework Overview](#overview)
2. [Routes and Controllers](#routes-and-controllers)
3. [JSON API](#json-api)
4. [Database Integration](#database-integration)
5. [Authentication](#authentication)
6. [WebSockets](#websockets)

---

## Overview

### สร้าง Play Project

```bash
# ใช้ sbt new
sbt new playframework/play-scala-seed.g8

# หรือ download template
# https://www.playframework.com/

# Project structure:
# app/
#   controllers/
#   models/
#   views/
#   services/
# conf/
#   application.conf
#   routes
# public/
#   css/
#   js/
#   images/
# test/
# build.sbt
```

### build.sbt สำหรับ Play

```scala
name := "my-play-app"
version := "1.0.0"

lazy val root = (project in file(".")).enablePlugins(PlayScala)

scalaVersion := "3.3.1"

libraryDependencies ++= Seq(
  guice,
  "org.scalatestplus.play" %% "scalatestplus-play" % "7.0.1" % Test,
  "com.typesafe.play"      %% "play-slick"         % "5.1.0",
  "com.typesafe.play"      %% "play-slick-evolutions" % "5.1.0",
  "com.h2database"          %  "h2"                % "2.2.224",
  "io.circe"               %% "circe-core"         % "0.14.6",
  "io.circe"               %% "circe-generic"      % "0.14.6"
)
```

---

## Routes and Controllers

### conf/routes

```
# conf/routes

GET    /                          controllers.HomeController.index
GET    /health                    controllers.HomeController.health

# Users API
GET    /api/users                 controllers.UserController.list
POST   /api/users                 controllers.UserController.create
GET    /api/users/:id             controllers.UserController.get(id: Long)
PUT    /api/users/:id             controllers.UserController.update(id: Long)
DELETE /api/users/:id             controllers.UserController.delete(id: Long)

# Search with query parameter
GET    /api/users/search          controllers.UserController.search(q: String, limit: Int ?= 10)

# Static assets
GET    /assets/*file              controllers.Assets.at(file)
```

### Controller

```scala
// app/controllers/UserController.scala
package controllers

import javax.inject.*
import play.api.*
import play.api.mvc.*
import play.api.libs.json.*
import models.*
import services.*
import scala.concurrent.{ExecutionContext, Future}

case class CreateUserRequest(name: String, email: String, age: Int)
case class UpdateUserRequest(name: Option[String], email: Option[String])

object UserJsonFormats:
  implicit val createUserFormat: OFormat[CreateUserRequest] =
    Json.format[CreateUserRequest]
  implicit val updateUserFormat: OFormat[UpdateUserRequest] =
    Json.format[UpdateUserRequest]
  implicit val userFormat: OFormat[User] =
    Json.format[User]

@Singleton
class UserController @Inject()(
  cc: ControllerComponents,
  userService: UserService
)(using ec: ExecutionContext) extends AbstractController(cc):

  import UserJsonFormats.*

  def list: Action[AnyContent] = Action.async {
    userService.findAll().map { users =>
      Ok(Json.toJson(users))
    }
  }

  def get(id: Long): Action[AnyContent] = Action.async {
    userService.findById(id).map {
      case Some(user) => Ok(Json.toJson(user))
      case None       => NotFound(Json.obj("error" -> s"User $id not found"))
    }
  }

  def create: Action[JsValue] = Action.async(parse.json) { request =>
    request.body.validate[CreateUserRequest] match
      case JsSuccess(req, _) =>
        userService.create(req.name, req.email, req.age).map { user =>
          Created(Json.toJson(user))
        }
      case JsError(errors) =>
        Future.successful(BadRequest(Json.obj(
          "error" -> "Invalid request",
          "details" -> JsError.toJson(errors)
        )))
  }

  def update(id: Long): Action[JsValue] = Action.async(parse.json) { request =>
    request.body.validate[UpdateUserRequest] match
      case JsSuccess(req, _) =>
        userService.update(id, req.name, req.email).map {
          case Some(user) => Ok(Json.toJson(user))
          case None       => NotFound(Json.obj("error" -> s"User $id not found"))
        }
      case JsError(errors) =>
        Future.successful(BadRequest(JsError.toJson(errors)))
  }

  def delete(id: Long): Action[AnyContent] = Action.async {
    userService.delete(id).map { deleted =>
      if deleted then NoContent
      else NotFound(Json.obj("error" -> s"User $id not found"))
    }
  }

  def search(q: String, limit: Int): Action[AnyContent] = Action.async {
    userService.search(q, limit).map { users =>
      Ok(Json.toJson(users))
    }
  }
```

---

## JSON API

### JSON Reads/Writes/Format

```scala
// app/models/User.scala
package models

import play.api.libs.json.*
import play.api.libs.functional.syntax.*

case class User(
  id: Option[Long],
  name: String,
  email: String,
  age: Int,
  role: UserRole,
  createdAt: java.time.Instant
)

sealed trait UserRole
object UserRole:
  case object Admin extends UserRole
  case object Regular extends UserRole
  case object Guest extends UserRole

  implicit val reads: Reads[UserRole] = Reads[UserRole] { json =>
    json.validate[String].flatMap {
      case "admin"   => JsSuccess(Admin)
      case "regular" => JsSuccess(Regular)
      case "guest"   => JsSuccess(Guest)
      case other     => JsError(s"Unknown role: $other")
    }
  }

  implicit val writes: Writes[UserRole] = Writes[UserRole] { role =>
    JsString(role.toString.toLowerCase)
  }

object User:
  // Custom Reads with validation
  implicit val reads: Reads[User] = (
    (__ \ "id").readNullable[Long] and
    (__ \ "name").read[String](Reads.minLength[String](2)) and
    (__ \ "email").read[String](Reads.email) and
    (__ \ "age").read[Int](Reads.min(0) keepAnd Reads.max(150)) and
    (__ \ "role").read[UserRole] and
    (__ \ "createdAt").read[java.time.Instant]
  )(User.apply)

  implicit val writes: OWrites[User] = Json.writes[User]

  // Transformation
  val publicView: Reads[JsObject] =
    (__ \ "id").read[Long].map(id => Json.obj("id" -> id)) and
    (__ \ "name").read[String].map(n => Json.obj("name" -> n)) reduce
```

---

## Database Integration

### Slick ORM

```scala
// app/models/UserTable.scala
package models

import slick.jdbc.H2Profile.api.*
import java.time.Instant

class UserTable(tag: Tag) extends Table[User](tag, "users"):
  def id        = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name      = column[String]("name", O.Length(255))
  def email     = column[String]("email", O.Unique, O.Length(255))
  def age       = column[Int]("age")
  def role      = column[String]("role", O.Default("regular"))
  def createdAt = column[Instant]("created_at")

  def * = (id.?, name, email, age, role, createdAt).mapTo[UserRow]

case class UserRow(
  id: Option[Long],
  name: String,
  email: String,
  age: Int,
  role: String,
  createdAt: Instant
)

// app/repositories/UserRepository.scala
package repositories

import javax.inject.*
import play.api.db.slick.*
import slick.jdbc.JdbcProfile
import models.*
import scala.concurrent.{ExecutionContext, Future}

@Singleton
class UserRepository @Inject()(
  protected val dbConfigProvider: DatabaseConfigProvider
)(using ec: ExecutionContext) extends HasDatabaseConfigProvider[JdbcProfile]:

  import profile.api.*

  private val users = TableQuery[UserTable]

  def findAll(): Future[Seq[UserRow]] = db.run(users.result)

  def findById(id: Long): Future[Option[UserRow]] =
    db.run(users.filter(_.id === id).result.headOption)

  def create(user: UserRow): Future[UserRow] =
    db.run((users returning users.map(_.id) into ((u, id) => u.copy(id = Some(id)))) += user)

  def update(id: Long, name: Option[String], email: Option[String]): Future[Int] =
    val query = users.filter(_.id === id)
    val updates = for
      existing <- query.result.headOption
    yield existing.map { u =>
      users.filter(_.id === id).update(u.copy(
        name = name.getOrElse(u.name),
        email = email.getOrElse(u.email)
      ))
    }.getOrElse(DBIO.successful(0))

    db.run(updates.flatten)

  def delete(id: Long): Future[Int] =
    db.run(users.filter(_.id === id).delete)

  def search(query: String): Future[Seq[UserRow]] =
    db.run(users.filter { u =>
      u.name.toLowerCase.like(s"%${query.toLowerCase}%") ||
      u.email.toLowerCase.like(s"%${query.toLowerCase}%")
    }.result)
```

---

## Authentication

### JWT Authentication

```scala
// app/services/AuthService.scala
package services

import javax.inject.*
import pdi.jwt.{JwtAlgorithm, JwtClaim, JwtJson}
import play.api.Configuration
import scala.concurrent.{ExecutionContext, Future}
import java.time.Instant

case class TokenPayload(userId: Long, email: String, role: String)

@Singleton
class AuthService @Inject()(config: Configuration):
  private val secret = config.get[String]("jwt.secret")
  private val expiry  = config.get[Long]("jwt.expiry.seconds")

  def generateToken(userId: Long, email: String, role: String): String =
    val claim = JwtClaim(
      content = s"""{"userId":$userId,"email":"$email","role":"$role"}""",
      expiration = Some(Instant.now().plusSeconds(expiry).getEpochSecond),
      issuedAt = Some(Instant.now().getEpochSecond)
    )
    JwtJson.encode(claim, secret, JwtAlgorithm.HS256)

  def validateToken(token: String): Option[TokenPayload] =
    JwtJson.decodeJson(token, secret, Seq(JwtAlgorithm.HS256)).toOption.flatMap { json =>
      (
        (json \ "userId").asOpt[Long],
        (json \ "email").asOpt[String],
        (json \ "role").asOpt[String]
      ).mapN(TokenPayload.apply)
    }

// Authenticated Action
class AuthenticatedRequest[A](val payload: TokenPayload, request: Request[A])
  extends WrappedRequest[A](request)

class AuthenticatedAction @Inject()(
  parser: BodyParsers.Default,
  authService: AuthService
)(using ec: ExecutionContext) extends ActionBuilder[AuthenticatedRequest, AnyContent]:

  def parser = parser
  protected def executionContext = ec

  def invokeBlock[A](request: Request[A], block: AuthenticatedRequest[A] => Future[Result]) =
    request.headers.get("Authorization") match
      case Some(auth) if auth.startsWith("Bearer ") =>
        val token = auth.drop(7)
        authService.validateToken(token) match
          case Some(payload) => block(AuthenticatedRequest(payload, request))
          case None          => Future.successful(Results.Unauthorized(Json.obj("error" -> "Invalid token")))
      case _ =>
        Future.successful(Results.Unauthorized(Json.obj("error" -> "Authorization header required")))
```

---

## WebSockets

```scala
// app/controllers/ChatController.scala
package controllers

import javax.inject.*
import play.api.mvc.*
import play.api.libs.streams.ActorFlow
import akka.actor.*
import akka.stream.Materializer

@Singleton
class ChatController @Inject()(cc: ControllerComponents)
    (using actorSystem: ActorSystem, mat: Materializer) extends AbstractController(cc):

  def chat(roomId: String): WebSocket = WebSocket.accept[String, String] { request =>
    ActorFlow.actorRef { out =>
      ChatActor.props(out, roomId)
    }
  }

object ChatActor:
  def props(out: ActorRef, roomId: String): Props =
    Props(new ChatActor(out, roomId))

class ChatActor(out: ActorRef, roomId: String) extends Actor:
  override def preStart(): Unit =
    ChatRoom.subscribe(roomId, self)

  def receive: Receive =
    case msg: String =>
      ChatRoom.broadcast(roomId, s"[$roomId] $msg")

  override def postStop(): Unit =
    ChatRoom.unsubscribe(roomId, self)

// Simple in-memory chat room
object ChatRoom:
  private var rooms = Map[String, Set[ActorRef]]()

  def subscribe(roomId: String, actor: ActorRef): Unit =
    rooms = rooms.updated(roomId, rooms.getOrElse(roomId, Set.empty) + actor)

  def unsubscribe(roomId: String, actor: ActorRef): Unit =
    val updated = rooms.getOrElse(roomId, Set.empty) - actor
    if updated.isEmpty then rooms = rooms - roomId
    else rooms = rooms.updated(roomId, updated)

  def broadcast(roomId: String, msg: String): Unit =
    rooms.getOrElse(roomId, Set.empty).foreach(_ ! msg)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Play Framework setup และ project structure
- ✅ Routes และ Controllers
- ✅ JSON API ด้วย Play JSON
- ✅ Database integration กับ Slick
- ✅ JWT Authentication
- ✅ WebSockets กับ Akka Actors

---

*[← Part 24: SBT](part-24-sbt.md) | [Part 26: HTTP4s →](part-26-http4s.md)*
