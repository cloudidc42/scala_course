# Part 39: gRPC กับ ScalaPB

## สารบัญ
1. [gRPC Overview](#grpc-overview)
2. [Protocol Buffers](#protocol-buffers)
3. [ScalaPB Setup](#scalapb-setup)
4. [Service Implementation](#service-implementation)
5. [Client Usage](#client-usage)
6. [Streaming RPCs](#streaming-rpcs)

---

## gRPC Overview

### แนวคิด gRPC

```
gRPC: Google Remote Procedure Call

Transport: HTTP/2
Serialization: Protocol Buffers (binary)
Code generation: from .proto files

Types of RPCs:
1. Unary:           client -> request -> server -> response
2. Server streaming: client -> request -> server -> stream of responses  
3. Client streaming: client -> stream of requests -> server -> response
4. Bidirectional:   client stream <-> server stream

Benefits:
- Strongly typed contracts
- Fast binary serialization
- Generated client/server code
- Streaming support
- Cross-language
```

### Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.grpc"               % "grpc-netty"             % "1.59.0",
  "com.thesamet.scalapb" %% "scalapb-runtime-grpc"   % scalapb.compiler.Version.scalapbVersion,
  "com.thesamet.scalapb" %% "scalapb-runtime"        % scalapb.compiler.Version.scalapbVersion % "protobuf"
)

// Add ScalaPB plugin
addSbtPlugin("com.thesamet" % "sbt-protoc" % "1.0.6")

libraryDependencies += "com.thesamet.scalapb" %% "compilerplugin" % "0.11.14"

// Generate Scala code from .proto files
Compile / PB.targets := Seq(
  scalapb.gen() -> (Compile / sourceManaged).value / "scalapb"
)
```

---

## Protocol Buffers

### Proto File

```protobuf
// src/main/protobuf/user.proto
syntax = "proto3";

package user.v1;

option java_package = "com.example.user.v1";
option java_multiple_files = true;

// Message definitions
message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  UserStatus status = 5;
  repeated string roles = 6;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  int32 age = 3;
}

message CreateUserResponse {
  User user = 1;
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message DeleteUserRequest {
  int64 id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

// Service definition
service UserService {
  // Unary RPCs
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);

  // Server streaming: stream users
  rpc ListUsers(ListUsersRequest) returns (stream GetUserResponse);

  // Client streaming: bulk create
  rpc BulkCreateUsers(stream CreateUserRequest) returns (CreateUserResponse);

  // Bidirectional streaming: chat
  rpc StreamUpdates(stream GetUserRequest) returns (stream GetUserResponse);
}
```

---

## ScalaPB Setup

### Generated Code

```scala
// ScalaPB generates these from .proto:
// - Case classes for messages
// - Enums as sealed traits
// - Service traits for client/server

// After generation you get:
// com.example.user.v1.User (case class)
// com.example.user.v1.UserStatus (sealed trait)
// com.example.user.v1.UserServiceGrpc (service)

// Using generated User:
val user = User(
  id = 1L,
  name = "Alice",
  email = "alice@example.com",
  age = 30,
  status = UserStatus.USER_STATUS_ACTIVE,
  roles = Seq("admin", "user")
)

// Protobuf serialization
val bytes = user.toByteArray
val fromBytes = User.parseFrom(bytes)
```

---

## Service Implementation

### Server Side

```scala
import io.grpc.*
import io.grpc.netty.NettyServerBuilder
import scala.concurrent.{Future, ExecutionContext}
import com.example.user.v1.*

// Implement generated service trait
class UserServiceImpl(implicit ec: ExecutionContext)
    extends UserServiceGrpc.UserService:

  private var users = Map(
    1L -> User(1L, "Alice", "alice@example.com", 30, UserStatus.USER_STATUS_ACTIVE),
    2L -> User(2L, "Bob", "bob@example.com", 25, UserStatus.USER_STATUS_ACTIVE)
  )
  private var nextId = 3L

  override def createUser(request: CreateUserRequest): Future[CreateUserResponse] =
    Future {
      val id = nextId
      nextId += 1
      val user = User(
        id = id,
        name = request.name,
        email = request.email,
        age = request.age,
        status = UserStatus.USER_STATUS_ACTIVE
      )
      users = users + (id -> user)
      CreateUserResponse(user = Some(user))
    }

  override def getUser(request: GetUserRequest): Future[GetUserResponse] =
    Future {
      users.get(request.id) match
        case Some(user) => GetUserResponse(user = Some(user))
        case None =>
          throw Status.NOT_FOUND
            .withDescription(s"User ${request.id} not found")
            .asRuntimeException()
    }

  override def deleteUser(request: DeleteUserRequest): Future[DeleteUserResponse] =
    Future {
      val existed = users.contains(request.id)
      users = users - request.id
      DeleteUserResponse(success = existed)
    }

  // Server streaming
  override def listUsers(
    request: ListUsersRequest,
    responseObserver: io.grpc.stub.StreamObserver[GetUserResponse]
  ): Unit =
    val filtered = users.values
      .filter(u => request.filter.isEmpty || u.name.contains(request.filter))
      .take(if request.pageSize > 0 then request.pageSize else 10)

    filtered.foreach { user =>
      responseObserver.onNext(GetUserResponse(user = Some(user)))
    }
    responseObserver.onCompleted()

// Start server
object UserServer extends App:
  import scala.concurrent.ExecutionContext.global

  val server = NettyServerBuilder
    .forPort(9090)
    .addService(UserServiceGrpc.bindService(
      new UserServiceImpl()(global),
      global
    ))
    .intercept(new LoggingInterceptor())
    .build()

  server.start()
  println("gRPC server started on port 9090")
  server.awaitTermination()

// Interceptor for logging
class LoggingInterceptor extends ServerInterceptor:
  override def interceptCall[ReqT, RespT](
    call: ServerCall[ReqT, RespT],
    headers: Metadata,
    next: ServerCallHandler[ReqT, RespT]
  ): ServerCall.Listener[ReqT] =
    println(s"Handling: ${call.getMethodDescriptor.getFullMethodName}")
    next.startCall(call, headers)
```

---

## Client Usage

### gRPC Client

```scala
import io.grpc.*
import io.grpc.netty.NettyChannelBuilder
import com.example.user.v1.*
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*

object UserClient:
  // Create channel
  val channel = NettyChannelBuilder
    .forAddress("localhost", 9090)
    .usePlaintext()  // use SSL/TLS in production
    .build()

  // Create stub (async)
  val stub = UserServiceGrpc.stub(channel)

  // Create blocking stub
  val blockingStub = UserServiceGrpc.blockingStub(channel)

  // Unary call
  def createUser(name: String, email: String, age: Int): Future[User] =
    stub.createUser(CreateUserRequest(name = name, email = email, age = age))
      .map(_.user.get)

  // Blocking call
  def getUserBlocking(id: Long): Option[User] =
    try
      Some(blockingStub.getUser(GetUserRequest(id = id)).user.get)
    catch
      case e: StatusRuntimeException if e.getStatus.getCode == Status.Code.NOT_FOUND =>
        None

  // Server streaming
  def listUsers(filter: String): Iterator[User] =
    blockingStub
      .listUsers(ListUsersRequest(filter = filter, pageSize = 100))
      .flatMap(_.user)

  // Error handling
  def safeGetUser(id: Long): Future[Either[String, User]] =
    stub.getUser(GetUserRequest(id = id))
      .map(resp => Right(resp.user.get))
      .recover {
        case e: StatusRuntimeException =>
          Left(s"gRPC error: ${e.getStatus.getCode} - ${e.getStatus.getDescription}")
        case e =>
          Left(s"Unexpected error: ${e.getMessage}")
      }

// Usage
import scala.concurrent.ExecutionContext.global
given ExecutionContext = global

val createFuture = UserClient.createUser("Charlie", "charlie@example.com", 28)
val created = Await.result(createFuture, 5.seconds)
println(s"Created: $created")

val users = UserClient.listUsers("").toList
println(s"Users: ${users.map(_.name).mkString(", ")}")
```

---

## Streaming RPCs

### Bidirectional Streaming

```scala
import io.grpc.stub.StreamObserver
import scala.concurrent.Promise
import com.example.user.v1.*

def streamingExample(): Unit =
  val responseFuture = Promise[List[User]]()
  val collectedUsers = collection.mutable.ListBuffer[User]()

  val requestObserver = UserClient.stub.streamUpdates(
    new StreamObserver[GetUserResponse]:
      override def onNext(response: GetUserResponse): Unit =
        response.user.foreach(collectedUsers += _)

      override def onError(t: Throwable): Unit =
        responseFuture.failure(t)

      override def onCompleted(): Unit =
        responseFuture.success(collectedUsers.toList)
  )

  // Send requests
  List(1L, 2L, 3L).foreach { id =>
    requestObserver.onNext(GetUserRequest(id = id))
  }
  requestObserver.onCompleted()

  import scala.concurrent.Await
  import scala.concurrent.duration.*
  val users = Await.result(responseFuture.future, 10.seconds)
  println(s"Streamed users: ${users.map(_.name)}")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ gRPC concepts: unary, server streaming, client streaming, bidirectional
- ✅ Protocol Buffers: message definitions, enums, services
- ✅ ScalaPB: code generation from .proto files
- ✅ Server implementation: service traits, interceptors
- ✅ Client usage: async stubs, blocking stubs
- ✅ Streaming RPCs

---

*[← Part 38: GraphQL](part-38-graphql.md) | [Part 40: Microservices →](part-40-microservices.md)*
