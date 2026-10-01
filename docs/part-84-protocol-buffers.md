# ส่วนที่ 84: Protocol Buffers and Binary Formats

## สารบัญ

1. [Protocol Buffers: ข้อดีเหนือ JSON](#protocol-buffers-ข้อดีเหนือ-json)
2. [ScalaPB Code Generation](#scalapb-code-generation)
3. [gRPC Service Definitions](#grpc-service-definitions)
4. [MessagePack กับ msgpack4s](#messagepack-กับ-msgpack4s)
5. [Avro กับ Vulcan](#avro-กับ-vulcan)
6. [Performance Comparison: JSON vs Protobuf vs Avro](#performance-comparison-json-vs-protobuf-vs-avro)
7. [Complete Serialization Example](#complete-serialization-example)
8. [สรุป](#สรุป)

---

## Protocol Buffers: ข้อดีเหนือ JSON

Protocol Buffers (Protobuf) คือ binary serialization format ที่ Google พัฒนาขึ้น

### ทำไมต้องใช้ Binary Formats

```
Feature           | JSON        | Protobuf    | Avro        | MessagePack
------------------|-------------|-------------|-------------|-------------
Size (typical)    | 100%        | 30-50%      | 40-60%      | 50-70%
Serialization     | Slow        | Fast        | Medium      | Fast
Deserialization   | Slow        | Fast        | Medium      | Fast
Schema            | None        | Required    | Required    | Optional
Human readable    | Yes         | No          | No          | No
Schema evolution  | Manual      | Built-in    | Built-in    | Limited
Language support  | Universal   | Wide        | Wide        | Wide
```

### ตัวอย่าง: JSON vs Protobuf

```json
// JSON: 180 bytes
{
  "user_id": "12345",
  "name": "Alice Smith",
  "email": "alice@example.com",
  "age": 30,
  "active": true,
  "scores": [95.5, 87.3, 92.1],
  "created_at": 1698765432
}
```

```protobuf
// Protobuf schema (user.proto)
syntax = "proto3";

message User {
  string user_id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  bool active = 5;
  repeated double scores = 6;
  int64 created_at = 7;
}

// Binary output: ~60-70 bytes (60% smaller)
```

### Protocol Buffers Encoding

```
Field encoding:
- Tag: (field_number << 3) | wire_type
- Wire types:
  0 = Varint (int32, int64, bool, enum)
  1 = 64-bit (double, fixed64)
  2 = Length-delimited (string, bytes, embedded messages)
  5 = 32-bit (float, fixed32)

ตัวอย่าง encoding ของ field name = "Alice":
- Tag: (2 << 3) | 2 = 0x12 (field 2, wire type 2)
- Length: 0x05 (5 bytes)
- Data: 0x41 0x6C 0x69 0x63 0x65 ("Alice" in UTF-8)
Total: 7 bytes (vs 14 bytes ใน JSON: "name":"Alice")
```

---

## ScalaPB Code Generation

### Setup

```scala
// project/plugins.sbt
addSbtPlugin("com.thesamet" % "sbt-protoc" % "1.0.6")

libraryDependencies += "com.thesamet.scalapb" %% "compilerplugin" % "0.11.13"
```

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.thesamet.scalapb" %% "scalapb-runtime" % scalapb.compiler.Version.scalapbVersion % "protobuf",
  "com.thesamet.scalapb" %% "scalapb-runtime-grpc" % scalapb.compiler.Version.scalapbVersion,
  "io.grpc" % "grpc-netty" % "1.58.0"
)

Compile / PB.targets := Seq(
  scalapb.gen() -> (Compile / sourceManaged).value / "scalapb"
)
```

### Proto Definition

```protobuf
// src/main/protobuf/user.proto
syntax = "proto3";

package com.example.user;

option java_package = "com.example.user";
option java_outer_classname = "UserProto";

import "google/protobuf/timestamp.proto";

// Enum types
enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_READER = 1;
  USER_ROLE_WRITER = 2;
  USER_ROLE_ADMIN = 3;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_BANNED = 3;
}

// Nested message
message Address {
  string street = 1;
  string city = 2;
  string country = 3;
  string postal_code = 4;
}

// Main message
message User {
  string id = 1;
  string email = 2;
  string display_name = 3;
  UserRole role = 4;
  UserStatus status = 5;
  Address address = 6;
  repeated string tags = 7;
  map<string, string> metadata = 8;
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
}

// Request/Response messages
message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  oneof result {
    User user = 1;
    string error_message = 2;
  }
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  UserStatus status_filter = 3;
  repeated string tag_filters = 4;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  bool has_next_page = 3;
}

message CreateUserRequest {
  string email = 1;
  string display_name = 2;
  UserRole role = 3;
  Address address = 4;
  repeated string tags = 5;
}

message UpdateUserRequest {
  string user_id = 1;
  optional string display_name = 2;
  optional UserRole role = 3;
  optional UserStatus status = 4;
  optional Address address = 5;
  repeated string add_tags = 6;
  repeated string remove_tags = 7;
}

message DeleteUserRequest {
  string user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}
```

### Using Generated Code

```scala
import com.example.user.*
import com.google.protobuf.timestamp.Timestamp
import java.time.Instant

// สร้าง User message
val address = Address(
  street = "123 Main St",
  city = "Bangkok",
  country = "Thailand",
  postalCode = "10110"
)

val now = Instant.now()
val timestamp = Timestamp(
  seconds = now.getEpochSecond,
  nanos = now.getNano
)

val user = User(
  id = "user-123",
  email = "alice@example.com",
  displayName = "Alice Smith",
  role = UserRole.USER_ROLE_WRITER,
  status = UserStatus.USER_STATUS_ACTIVE,
  address = Some(address),
  tags = Seq("premium", "verified"),
  metadata = Map("source" -> "web", "referral" -> "friend"),
  createdAt = Some(timestamp),
  updatedAt = Some(timestamp)
)

// Serialize to bytes
val bytes = user.toByteArray
println(s"Serialized size: ${bytes.length} bytes")

// Deserialize from bytes
val restored = User.parseFrom(bytes)
println(s"Deserialized: ${restored.displayName}")
println(s"Email: ${restored.email}")
println(s"Role: ${restored.role}")

// toProtoString (text format for debugging)
println(s"Text format:\n${user.toProtoString}")

// JSON encoding (via scalapb-json4s)
import scalapb.json4s.JsonFormat
val json = JsonFormat.toJsonString(user)
println(s"JSON:\n$json")

val fromJson = JsonFormat.fromJsonString[User](json)
```

### Schema Evolution

```protobuf
// user_v2.proto - backward compatible evolution
syntax = "proto3";

message User {
  string id = 1;
  string email = 2;
  string display_name = 3;
  UserRole role = 4;
  UserStatus status = 5;
  Address address = 6;
  repeated string tags = 7;
  map<string, string> metadata = 8;
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
  
  // New fields added in v2 (safe to add)
  string phone_number = 11;        // new field
  repeated string permissions = 12; // new field
  
  // removed field (NEVER reuse field number 13 if it existed)
  // reserved 13;
  // reserved "old_field_name";
}

// Schema evolution rules:
// ✓ Add new optional fields (safe)
// ✓ Remove fields (mark as reserved)
// ✓ Rename fields (doesn't affect binary)
// ✓ Add values to enum
// ✗ Change field types
// ✗ Reuse field numbers
// ✗ Change field numbers
```

---

## gRPC Service Definitions

### gRPC Service Definition

```protobuf
// src/main/protobuf/user_service.proto
syntax = "proto3";

package com.example.user.service;

import "user.proto";
import "google/protobuf/empty.proto";

option java_package = "com.example.user.service";

// gRPC service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  
  // Server streaming RPC
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming RPC
  rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkCreateResponse);
  
  // Bidirectional streaming RPC
  rpc SyncUsers(stream UserSyncRequest) returns (stream UserSyncResponse);
}

message BulkCreateResponse {
  int32 created_count = 1;
  int32 failed_count = 2;
  repeated string errors = 3;
}

message UserSyncRequest {
  oneof action {
    User upsert_user = 1;
    string delete_user_id = 2;
  }
}

message UserSyncResponse {
  string user_id = 1;
  bool success = 2;
  string message = 3;
}
```

### gRPC Server Implementation

```scala
import io.grpc.{Server, ServerBuilder, Status}
import scalapb.grpc.Grpc
import com.example.user.*
import com.example.user.service.*
import io.grpc.stub.StreamObserver
import scala.concurrent.{ExecutionContext, Future}

class UserServiceImpl(
  userRepo: UserRepository
)(using ec: ExecutionContext) extends UserServiceGrpc.UserService:
  
  // Unary: Get User
  override def getUser(request: GetUserRequest): Future[GetUserResponse] =
    userRepo.findById(request.userId).map {
      case Some(user) => GetUserResponse(GetUserResponse.Result.User(user))
      case None => GetUserResponse(GetUserResponse.Result.ErrorMessage(
        s"User ${request.userId} not found"
      ))
    }
  
  // Unary: Create User
  override def createUser(request: CreateUserRequest): Future[User] =
    val user = User(
      id = java.util.UUID.randomUUID().toString,
      email = request.email,
      displayName = request.displayName,
      role = request.role,
      status = UserStatus.USER_STATUS_ACTIVE,
      address = request.address,
      tags = request.tags
    )
    userRepo.save(user)
  
  // Unary: Update User
  override def updateUser(request: UpdateUserRequest): Future[User] =
    userRepo.findById(request.userId).flatMap {
      case None =>
        Future.failed(Status.NOT_FOUND
          .withDescription(s"User ${request.userId} not found")
          .asRuntimeException())
      case Some(existing) =>
        val updated = existing.copy(
          displayName = request.displayName.getOrElse(existing.displayName),
          role = request.role.getOrElse(existing.role),
          status = request.status.getOrElse(existing.status),
          address = request.address.orElse(existing.address),
          tags = (existing.tags.toSet ++ request.addTags.toSet -- request.removeTags.toSet).toSeq
        )
        userRepo.save(updated)
    }
  
  // Server streaming: List Users
  override def listUsers(
    request: ListUsersRequest,
    responseObserver: StreamObserver[User]
  ): Unit =
    userRepo.findAll(
      page = request.page,
      pageSize = request.pageSize,
      statusFilter = Some(request.statusFilter)
    ).foreach { users =>
      users.foreach(responseObserver.onNext)
      responseObserver.onCompleted()
    }
  
  // Client streaming: Bulk Create
  override def bulkCreateUsers(
    responseObserver: StreamObserver[BulkCreateResponse]
  ): StreamObserver[CreateUserRequest] =
    new StreamObserver[CreateUserRequest]:
      private var created = 0
      private var failed = 0
      private val errors = scala.collection.mutable.ArrayBuffer[String]()
      
      override def onNext(request: CreateUserRequest): Unit =
        try
          val user = User(
            id = java.util.UUID.randomUUID().toString,
            email = request.email,
            displayName = request.displayName
          )
          userRepo.save(user)
          created += 1
        catch
          case e: Exception =>
            failed += 1
            errors += e.getMessage
      
      override def onError(t: Throwable): Unit =
        responseObserver.onError(t)
      
      override def onCompleted(): Unit =
        responseObserver.onNext(BulkCreateResponse(
          createdCount = created,
          failedCount = failed,
          errors = errors.toSeq
        ))
        responseObserver.onCompleted()

// Server setup
object GrpcServer:
  def start(port: Int): Server =
    val server = ServerBuilder
      .forPort(port)
      .addService(UserServiceGrpc.bindService(
        new UserServiceImpl(InMemoryUserRepo()),
        scala.concurrent.ExecutionContext.global
      ))
      .build()
    
    server.start()
    println(s"gRPC server started on port $port")
    
    Runtime.getRuntime.addShutdownHook(new Thread(() => {
      println("Shutting down gRPC server...")
      server.shutdown()
    }))
    
    server
```

### gRPC Client

```scala
import io.grpc.{ManagedChannel, ManagedChannelBuilder}
import com.example.user.service.*
import scala.concurrent.{ExecutionContext, Future, Await}
import scala.concurrent.duration.*

class UserServiceClient(host: String, port: Int)(using ec: ExecutionContext):
  private val channel: ManagedChannel = ManagedChannelBuilder
    .forAddress(host, port)
    .usePlaintext()  // ใน production ใช้ TLS
    .build()
  
  private val stub: UserServiceGrpc.UserServiceStub = 
    UserServiceGrpc.stub(channel)
  
  private val blockingStub: UserServiceGrpc.UserServiceBlockingStub =
    UserServiceGrpc.blockingStub(channel)
  
  // Async call
  def getUser(userId: String): Future[GetUserResponse] =
    stub.getUser(GetUserRequest(userId))
  
  // Blocking call
  def getUserBlocking(userId: String): GetUserResponse =
    blockingStub.getUser(GetUserRequest(userId))
  
  // Streaming: iterate over results
  def listAllUsers(): Iterator[User] =
    blockingStub.listUsers(ListUsersRequest(
      page = 1,
      pageSize = 100
    ))
  
  def close(): Unit = channel.shutdown()

// Usage
@main def grpcClientExample(): Unit =
  given ec: scala.concurrent.ExecutionContext = 
    scala.concurrent.ExecutionContext.global
  
  val client = new UserServiceClient("localhost", 50051)
  
  // Create user
  val created = Await.result(
    client.stub.createUser(CreateUserRequest(
      email = "bob@example.com",
      displayName = "Bob Jones",
      role = UserRole.USER_ROLE_WRITER
    )),
    5.seconds
  )
  println(s"Created user: ${created.id}")
  
  // Get user
  val response = client.getUserBlocking(created.id)
  response.result match
    case GetUserResponse.Result.User(user) => println(s"Found: ${user.displayName}")
    case GetUserResponse.Result.ErrorMessage(msg) => println(s"Error: $msg")
    case _ => println("Unknown response")
  
  // List users (streaming)
  println("\nAll users:")
  client.listAllUsers().foreach { user =>
    println(s"  - ${user.displayName} (${user.email})")
  }
  
  client.close()
```

---

## MessagePack กับ msgpack4s

### MessagePack Overview

MessagePack คือ binary format ที่คล้าย JSON แต่เล็กกว่าและเร็วกว่า

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.msgpack" % "msgpack-core" % "0.9.6",
  "io.monix" %% "monix-reactive" % "3.4.1" // optional
)
```

### Basic MessagePack Usage

```scala
import org.msgpack.core.{MessagePack, MessageBufferPacker, MessageUnpacker}
import org.msgpack.value.*

// Packing data
def packUser(name: String, age: Int, scores: List[Double]): Array[Byte] =
  val packer = MessagePack.newDefaultBufferPacker()
  packer.packMapHeader(3)                    // map with 3 entries
  packer.packString("name"); packer.packString(name)
  packer.packString("age"); packer.packInt(age)
  packer.packString("scores")
  packer.packArrayHeader(scores.length)
  scores.foreach(packer.packDouble)
  packer.toByteArray

// Unpacking data
def unpackUser(bytes: Array[Byte]): (String, Int, List[Double]) =
  val unpacker = MessagePack.newDefaultUnpacker(bytes)
  val mapSize = unpacker.unpackMapHeader()   // should be 3
  
  var name = ""
  var age = 0
  var scores = List.empty[Double]
  
  for _ <- 0 until mapSize do
    val key = unpacker.unpackString()
    key match
      case "name" => name = unpacker.unpackString()
      case "age" => age = unpacker.unpackInt()
      case "scores" =>
        val arrSize = unpacker.unpackArrayHeader()
        scores = (0 until arrSize).map(_ => unpacker.unpackDouble()).toList
      case _ => unpacker.skipValue()
  
  (name, age, scores)

@main def msgpackDemo(): Unit =
  val packed = packUser("Alice", 30, List(95.5, 87.3, 92.1))
  println(s"Packed size: ${packed.length} bytes")
  
  val (name, age, scores) = unpackUser(packed)
  println(s"Name: $name, Age: $age, Scores: $scores")
  
  // เปรียบเทียบกับ JSON
  import upickle.default.*
  case class UserData(name: String, age: Int, scores: List[Double]) derives ReadWriter
  val user = UserData("Alice", 30, List(95.5, 87.3, 92.1))
  val jsonBytes = write(user).getBytes("UTF-8")
  println(s"JSON size: ${jsonBytes.length} bytes")
  println(s"Size reduction: ${(1 - packed.length.toDouble / jsonBytes.length) * 100}%%.1f%%")
```

### Type-Safe MessagePack ด้วย Custom Codecs

```scala
import org.msgpack.core.*

// Type class สำหรับ MessagePack serialization
trait MsgpackCodec[A]:
  def pack(packer: MessagePacker, value: A): Unit
  def unpack(unpacker: MessageUnpacker): A

object MsgpackCodec:
  def apply[A](using codec: MsgpackCodec[A]): MsgpackCodec[A] = codec
  
  given MsgpackCodec[String] with
    def pack(p: MessagePacker, v: String): Unit = p.packString(v)
    def unpack(u: MessageUnpacker): String = u.unpackString()
  
  given MsgpackCodec[Int] with
    def pack(p: MessagePacker, v: Int): Unit = p.packInt(v)
    def unpack(u: MessageUnpacker): Int = u.unpackInt()
  
  given MsgpackCodec[Double] with
    def pack(p: MessagePacker, v: Double): Unit = p.packDouble(v)
    def unpack(u: MessageUnpacker): Double = u.unpackDouble()
  
  given MsgpackCodec[Boolean] with
    def pack(p: MessagePacker, v: Boolean): Unit = p.packBoolean(v)
    def unpack(u: MessageUnpacker): Boolean = u.unpackBoolean()
  
  given [A](using codec: MsgpackCodec[A]): MsgpackCodec[List[A]] with
    def pack(p: MessagePacker, v: List[A]): Unit =
      p.packArrayHeader(v.length)
      v.foreach(codec.pack(p, _))
    def unpack(u: MessageUnpacker): List[A] =
      val size = u.unpackArrayHeader()
      (0 until size).map(_ => codec.unpack(u)).toList
  
  given [A](using codec: MsgpackCodec[A]): MsgpackCodec[Option[A]] with
    def pack(p: MessagePacker, v: Option[A]): Unit =
      v match
        case None => p.packNil()
        case Some(a) => codec.pack(p, a)
    def unpack(u: MessageUnpacker): Option[A] =
      if u.tryUnpackNil() then None
      else Some(codec.unpack(u))

// serialize/deserialize helpers
def serialize[A](value: A)(using codec: MsgpackCodec[A]): Array[Byte] =
  val packer = MessagePack.newDefaultBufferPacker()
  codec.pack(packer, value)
  packer.toByteArray

def deserialize[A](bytes: Array[Byte])(using codec: MsgpackCodec[A]): A =
  val unpacker = MessagePack.newDefaultUnpacker(bytes)
  codec.unpack(unpacker)
```

---

## Avro กับ Vulcan

### Apache Avro Overview

Avro เป็น data serialization framework ที่รองรับ schema evolution และ dynamic typing

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.sksamuel.avro4s" %% "avro4s-core" % "5.0.9",
  "org.apache.avro" % "avro" % "1.11.3",
  // สำหรับ Kafka integration
  "io.confluent" % "kafka-avro-serializer" % "7.5.0"
)

// Vulcan (functional Avro)
libraryDependencies += "com.github.fd4s" %% "vulcan" % "1.10.1"
```

### Avro ด้วย avro4s

```scala
import com.sksamuel.avro4s.*
import java.io.{ByteArrayOutputStream, ByteArrayInputStream}

// Case class จะถูก derive schema โดย automatic
case class Product(
  id: String,
  name: String,
  price: Double,
  category: String,
  tags: List[String],
  inStock: Boolean
)

// Derive schema automatically
val schema = AvroSchema[Product]
println(s"Schema:\n${schema.toString(true)}")

// Serialize
def serializeProduct(product: Product): Array[Byte] =
  val baos = new ByteArrayOutputStream()
  val avroOut = AvroOutputStream.data[Product].to(baos).build()
  avroOut.write(product)
  avroOut.flush()
  avroOut.close()
  baos.toByteArray

// Deserialize
def deserializeProduct(bytes: Array[Byte]): List[Product] =
  val bais = new ByteArrayInputStream(bytes)
  val avroIn = AvroInputStream.data[Product].from(bais).build(schema)
  val result = avroIn.iterator.toList
  avroIn.close()
  result

@main def avroDemo(): Unit =
  val products = List(
    Product("p1", "Laptop", 999.99, "Electronics", List("tech", "portable"), true),
    Product("p2", "Book", 29.99, "Education", List("learning"), true),
    Product("p3", "Headphones", 199.99, "Electronics", List("audio", "tech"), false)
  )
  
  val bytes = products.flatMap(p => serializeProduct(p)).toArray
  println(s"Serialized ${products.length} products: ${bytes.length} bytes")
  
  val restored = deserializeProduct(bytes)
  println(s"Deserialized ${restored.length} products")
  restored.foreach(p => println(s"  - ${p.name}: $${p.price}"))
```

### Vulcan: Functional Avro

```scala
import vulcan.*
import vulcan.generic.*

// Vulcan ใช้ typeclass approach
case class UserEvent(
  userId: String,
  eventType: String,
  timestamp: Long,
  properties: Map[String, String]
)

// derive Codec automatically
object UserEvent:
  given Codec[UserEvent] = Codec.derive[UserEvent]

// Manual codec definition
val userEventCodec: Codec[UserEvent] =
  Codec.record[UserEvent](
    name = "UserEvent",
    namespace = "com.example"
  ) { field =>
    (
      field("userId", _.userId),
      field("eventType", _.eventType),
      field("timestamp", _.timestamp),
      field("properties", _.properties)
    ).mapN(UserEvent.apply)
  }

// Encode
def encodeAvro[A](value: A)(using codec: Codec[A]): Either[AvroError, Array[Byte]] =
  codec.encode(value).map { avroValue =>
    val baos = new java.io.ByteArrayOutputStream()
    val encoder = org.apache.avro.io.EncoderFactory.get()
      .binaryEncoder(baos, null)
    val writer = new org.apache.avro.generic.GenericDatumWriter[Any](codec.schema.orNull)
    writer.write(avroValue, encoder)
    encoder.flush()
    baos.toByteArray
  }

// Schema registry integration
object SchemaRegistry:
  private val schemas = scala.collection.mutable.HashMap[String, org.apache.avro.Schema]()
  
  def register(name: String, schema: org.apache.avro.Schema): Int =
    schemas(name) = schema
    schemas.size
  
  def getSchema(name: String): Option[org.apache.avro.Schema] =
    schemas.get(name)
  
  // For Kafka Avro Serializer
  def getUrl: String = "http://schema-registry:8081"
```

---

## Performance Comparison: JSON vs Protobuf vs Avro

### Benchmark Implementation

```scala
import java.time.{Duration, Instant}
import scala.util.Random

case class BenchmarkData(
  id: String,
  name: String,
  values: List[Double],
  metadata: Map[String, String],
  timestamp: Long,
  active: Boolean
)

// เตรียม test data
def generateTestData(n: Int): List[BenchmarkData] =
  val rand = new Random(42)
  (1 to n).toList.map { i =>
    BenchmarkData(
      id = s"item-$i",
      name = s"Test Item $i",
      values = (1 to 10).map(_ => rand.nextDouble() * 100).toList,
      metadata = Map(
        "source" -> "test",
        "version" -> "1.0",
        "region" -> (if i % 2 == 0 then "us-east" else "eu-west")
      ),
      timestamp = System.currentTimeMillis(),
      active = i % 3 != 0
    )
  }

// Benchmark JSON (circe)
def benchmarkJson(data: List[BenchmarkData]): BenchmarkResult =
  import io.circe.generic.auto.*
  import io.circe.syntax.*
  import io.circe.parser.*
  
  val startSerialize = System.nanoTime()
  val jsonStrings = data.map(_.asJson.noSpaces)
  val serializedBytes = jsonStrings.map(_.getBytes("UTF-8"))
  val serializeMs = (System.nanoTime() - startSerialize) / 1_000_000.0
  
  val totalBytes = serializedBytes.map(_.length).sum
  
  val startDeserialize = System.nanoTime()
  val decoded = jsonStrings.map(decode[BenchmarkData])
  val deserializeMs = (System.nanoTime() - startDeserialize) / 1_000_000.0
  
  BenchmarkResult(
    format = "JSON",
    serializeMs = serializeMs,
    deserializeMs = deserializeMs,
    totalBytes = totalBytes,
    successCount = decoded.count(_.isRight)
  )

// Benchmark Protobuf (manual simulation)
def benchmarkProtobuf(data: List[BenchmarkData]): BenchmarkResult =
  // ใช้ manual packing ด้วย MessagePack เพื่อ simulate Protobuf performance
  import org.msgpack.core.MessagePack
  
  val startSerialize = System.nanoTime()
  val packed = data.map { item =>
    val packer = MessagePack.newDefaultBufferPacker()
    packer.packMapHeader(6)
    packer.packString("id"); packer.packString(item.id)
    packer.packString("name"); packer.packString(item.name)
    packer.packString("values")
    packer.packArrayHeader(item.values.length)
    item.values.foreach(packer.packDouble)
    packer.packString("metadata")
    packer.packMapHeader(item.metadata.size)
    item.metadata.foreach { (k, v) =>
      packer.packString(k); packer.packString(v)
    }
    packer.packString("timestamp"); packer.packLong(item.timestamp)
    packer.packString("active"); packer.packBoolean(item.active)
    packer.toByteArray
  }
  val serializeMs = (System.nanoTime() - startSerialize) / 1_000_000.0
  
  val totalBytes = packed.map(_.length).sum
  
  val startDeserialize = System.nanoTime()
  val unpacked = packed.map { bytes =>
    val unpacker = MessagePack.newDefaultUnpacker(bytes)
    val size = unpacker.unpackMapHeader()
    var id = ""; var name = ""
    for _ <- 0 until size do
      val key = unpacker.unpackString()
      key match
        case "id" => id = unpacker.unpackString()
        case "name" => name = unpacker.unpackString()
        case "values" =>
          val n = unpacker.unpackArrayHeader()
          (0 until n).foreach(_ => unpacker.unpackDouble())
        case "metadata" =>
          val n = unpacker.unpackMapHeader()
          (0 until n).foreach { _ =>
            unpacker.unpackString(); unpacker.unpackString()
          }
        case "timestamp" => unpacker.unpackLong()
        case "active" => unpacker.unpackBoolean()
        case _ => unpacker.skipValue()
    (id, name)
  }
  val deserializeMs = (System.nanoTime() - startDeserialize) / 1_000_000.0
  
  BenchmarkResult(
    format = "Binary",
    serializeMs = serializeMs,
    deserializeMs = deserializeMs,
    totalBytes = totalBytes,
    successCount = unpacked.length
  )

case class BenchmarkResult(
  format: String,
  serializeMs: Double,
  deserializeMs: Double,
  totalBytes: Int,
  successCount: Int
)

@main def runBenchmarks(): Unit =
  val n = 10000
  println(s"Benchmarking with $n items...\n")
  
  val data = generateTestData(n)
  
  // warm up
  benchmarkJson(data.take(100))
  benchmarkProtobuf(data.take(100))
  
  // actual benchmark
  val jsonResult = benchmarkJson(data)
  val binaryResult = benchmarkProtobuf(data)
  
  println(f"{'Format'%-12s} | ${"Serialize(ms)"}%-14s | ${"Deserialize(ms)"}%-16s | ${"Total bytes"}%-12s | ${"Per item (bytes)"}%-16s")
  println("-" * 80)
  
  for result <- List(jsonResult, binaryResult) do
    val perItem = result.totalBytes / n
    println(f"${result.format}%-12s | ${result.serializeMs}%-14.2f | ${result.deserializeMs}%-16.2f | ${result.totalBytes}%-12d | $perItem%-16d")
  
  println(s"\nSize reduction: ${(1 - binaryResult.totalBytes.toDouble / jsonResult.totalBytes) * 100}%.1f%%")
  println(s"Serialize speedup: ${jsonResult.serializeMs / binaryResult.serializeMs}%.2fx")
  println(s"Deserialize speedup: ${jsonResult.deserializeMs / binaryResult.deserializeMs}%.2fx")
```

---

## Complete Serialization Example

### Multi-Format Event System

```scala
import cats.effect.{IO, IOApp, ExitCode}

// Event types
sealed trait DomainEvent
case class UserRegistered(
  userId: String,
  email: String,
  timestamp: Long
) extends DomainEvent

case class OrderPlaced(
  orderId: String,
  userId: String,
  amount: Double,
  items: List[String],
  timestamp: Long
) extends DomainEvent

case class PaymentProcessed(
  paymentId: String,
  orderId: String,
  amount: Double,
  success: Boolean,
  timestamp: Long
) extends DomainEvent

// Generic serializer
trait EventSerializer[F[_]]:
  def serialize(event: DomainEvent): F[Array[Byte]]
  def deserialize(bytes: Array[Byte], eventType: String): F[DomainEvent]

// JSON serializer
class JsonEventSerializer extends EventSerializer[IO]:
  import io.circe.*
  import io.circe.generic.auto.*
  import io.circe.syntax.*
  import io.circe.parser.*
  
  def serialize(event: DomainEvent): IO[Array[Byte]] =
    IO {
      val json = event match
        case e: UserRegistered => e.asJson.noSpaces
        case e: OrderPlaced => e.asJson.noSpaces
        case e: PaymentProcessed => e.asJson.noSpaces
      json.getBytes("UTF-8")
    }
  
  def deserialize(bytes: Array[Byte], eventType: String): IO[DomainEvent] =
    IO.fromEither {
      val json = new String(bytes, "UTF-8")
      eventType match
        case "UserRegistered" => decode[UserRegistered](json).map(identity)
        case "OrderPlaced" => decode[OrderPlaced](json).map(identity)
        case "PaymentProcessed" => decode[PaymentProcessed](json).map(identity)
        case t => Left(DecodingFailure(s"Unknown event type: $t", Nil))
    }

// MessagePack serializer
class MsgpackEventSerializer extends EventSerializer[IO]:
  import org.msgpack.core.MessagePack
  
  def serialize(event: DomainEvent): IO[Array[Byte]] =
    IO {
      val packer = MessagePack.newDefaultBufferPacker()
      event match
        case e: UserRegistered =>
          packer.packMapHeader(3)
          packer.packString("userId"); packer.packString(e.userId)
          packer.packString("email"); packer.packString(e.email)
          packer.packString("timestamp"); packer.packLong(e.timestamp)
        
        case e: OrderPlaced =>
          packer.packMapHeader(5)
          packer.packString("orderId"); packer.packString(e.orderId)
          packer.packString("userId"); packer.packString(e.userId)
          packer.packString("amount"); packer.packDouble(e.amount)
          packer.packString("items")
          packer.packArrayHeader(e.items.length)
          e.items.foreach(packer.packString)
          packer.packString("timestamp"); packer.packLong(e.timestamp)
        
        case e: PaymentProcessed =>
          packer.packMapHeader(5)
          packer.packString("paymentId"); packer.packString(e.paymentId)
          packer.packString("orderId"); packer.packString(e.orderId)
          packer.packString("amount"); packer.packDouble(e.amount)
          packer.packString("success"); packer.packBoolean(e.success)
          packer.packString("timestamp"); packer.packLong(e.timestamp)
      
      packer.toByteArray
    }
  
  def deserialize(bytes: Array[Byte], eventType: String): IO[DomainEvent] =
    IO {
      val unpacker = MessagePack.newDefaultUnpacker(bytes)
      val mapSize = unpacker.unpackMapHeader()
      val fields = (0 until mapSize).map { _ =>
        val key = unpacker.unpackString()
        key -> unpacker.unpackValue()
      }.toMap
      
      eventType match
        case "UserRegistered" =>
          UserRegistered(
            userId = fields("userId").asStringValue().asString(),
            email = fields("email").asStringValue().asString(),
            timestamp = fields("timestamp").asIntegerValue().asLong()
          )
        case "OrderPlaced" =>
          import scala.jdk.CollectionConverters.*
          OrderPlaced(
            orderId = fields("orderId").asStringValue().asString(),
            userId = fields("userId").asStringValue().asString(),
            amount = fields("amount").asFloatValue().toDouble(),
            items = fields("items").asArrayValue().list().asScala
              .map(_.asStringValue().asString()).toList,
            timestamp = fields("timestamp").asIntegerValue().asLong()
          )
        case _ => throw new Exception(s"Unknown event type: $eventType")
    }

// Event bus ที่รองรับหลาย serialization formats
class MultiFormatEventBus(
  jsonSerializer: JsonEventSerializer,
  msgpackSerializer: MsgpackEventSerializer
):
  sealed trait SerializationFormat
  case object JsonFormat extends SerializationFormat
  case object MsgpackFormat extends SerializationFormat
  
  def publish(
    event: DomainEvent,
    format: SerializationFormat = JsonFormat
  ): IO[EventEnvelope] =
    for
      bytes <- format match
        case JsonFormat => jsonSerializer.serialize(event)
        case MsgpackFormat => msgpackSerializer.serialize(event)
      envelope = EventEnvelope(
        id = java.util.UUID.randomUUID().toString,
        eventType = event.getClass.getSimpleName,
        format = format.toString,
        payload = bytes,
        timestamp = System.currentTimeMillis()
      )
    yield envelope
  
  def consume(envelope: EventEnvelope): IO[DomainEvent] =
    envelope.format match
      case "JsonFormat" => 
        jsonSerializer.deserialize(envelope.payload, envelope.eventType)
      case "MsgpackFormat" =>
        msgpackSerializer.deserialize(envelope.payload, envelope.eventType)
      case f => IO.raiseError(new Exception(s"Unknown format: $f"))

case class EventEnvelope(
  id: String,
  eventType: String,
  format: String,
  payload: Array[Byte],
  timestamp: Long
)

// Main demo
object SerializationDemo extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    val bus = MultiFormatEventBus(
      new JsonEventSerializer(),
      new MsgpackEventSerializer()
    )
    
    val events = List(
      UserRegistered("u1", "alice@example.com", System.currentTimeMillis()),
      OrderPlaced("o1", "u1", 299.99, List("item1", "item2", "item3"), System.currentTimeMillis()),
      PaymentProcessed("p1", "o1", 299.99, true, System.currentTimeMillis())
    )
    
    for
      _ <- IO.println("=== Serialization Demo ===\n")
      
      // Publish with JSON
      jsonEnvelopes <- events.traverse(bus.publish(_, MultiFormatEventBus.JsonFormat))
      _ <- IO.println(s"JSON envelope sizes:")
      _ <- jsonEnvelopes.traverse(e => IO.println(s"  ${e.eventType}: ${e.payload.length} bytes"))
      
      // Publish with MessagePack
      msgpackEnvelopes <- events.traverse(bus.publish(_, MultiFormatEventBus.MsgpackFormat))
      _ <- IO.println(s"\nMessagePack envelope sizes:")
      _ <- msgpackEnvelopes.traverse(e => IO.println(s"  ${e.eventType}: ${e.payload.length} bytes"))
      
      // Size comparison
      jsonTotal = jsonEnvelopes.map(_.payload.length).sum
      msgpackTotal = msgpackEnvelopes.map(_.payload.length).sum
      _ <- IO.println(s"\nTotal JSON: $jsonTotal bytes")
      _ <- IO.println(s"Total MessagePack: $msgpackTotal bytes")
      _ <- IO.println(f"Reduction: ${(1 - msgpackTotal.toDouble / jsonTotal) * 100}%.1f%%")
      
      // Consume events
      _ <- IO.println("\n=== Consuming Events ===")
      consumed <- jsonEnvelopes.traverse(bus.consume)
      _ <- consumed.traverse(e => IO.println(s"  Consumed: ${e.getClass.getSimpleName}"))
    yield ExitCode.Success
```

---

## สรุป

Binary serialization formats มีข้อดีสำคัญสำหรับ production systems:

| Format | Best For | Schema Required | Language Support |
|--------|---------|----------------|-----------------|
| Protocol Buffers | gRPC, microservices | Yes | Wide |
| Avro | Kafka, data lakes | Yes | Wide |
| MessagePack | General purpose, JSON replacement | No | Very wide |
| Thrift | Facebook ecosystem | Yes | Wide |
| FlatBuffers | Games, mobile, zero-copy | Yes | Wide |

### เมื่อไหร่ใช้อะไร

```
Protobuf:
  ✓ Microservices communication
  ✓ gRPC services
  ✓ Performance-critical internal APIs

Avro:
  ✓ Kafka messages
  ✓ Schema registry
  ✓ Data pipelines (Spark, Flink)

MessagePack:
  ✓ JSON replacement ที่ต้องการ performance
  ✓ Caching
  ✓ WebSocket messages

JSON:
  ✓ Public APIs
  ✓ Human-readable configs
  ✓ Debugging
```

---

*[← ส่วนที่ 83: Machine Learning with Scala](part-83-ml-scala.md)*
