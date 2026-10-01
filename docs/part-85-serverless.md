# ตอนที่ 85: Serverless Scala

## สารบัญ

1. [แนวคิด Serverless และ FaaS](#แนวคิด-serverless-และ-faas)
2. [AWS Lambda กับ Scala](#aws-lambda-กับ-scala)
3. [Google Cloud Functions กับ Scala](#google-cloud-functions-กับ-scala)
4. [การ Optimize Cold Start](#การ-optimize-cold-start)
5. [Serverless Framework](#serverless-framework)
6. [API Gateway Integration](#api-gateway-integration)
7. [ตัวอย่าง Serverless API แบบครบถ้วน](#ตัวอย่าง-serverless-api-แบบครบถ้วน)
8. [สรุป](#สรุป)

---

## แนวคิด Serverless และ FaaS

Serverless computing เป็นรูปแบบการพัฒนาที่ผู้ใช้ไม่ต้องจัดการ Infrastructure เอง โดย Cloud Provider จะดูแลการจัดสรรและปรับขนาด Resources ให้อัตโนมัติ

### ข้อดีของ Serverless

- **ไม่ต้องจัดการ Server**: Cloud Provider ดูแลทั้งหมด
- **Scale อัตโนมัติ**: ปรับขนาดตามความต้องการ
- **จ่ายตามใช้**: คิดค่าบริการตามเวลาที่ Function ทำงานจริง
- **High Availability**: Built-in redundancy

### ข้อเสียของ Serverless

- **Cold Start**: Function ที่ไม่ได้ใช้งานนานจะใช้เวลาเริ่มต้นนาน
- **Stateless**: แต่ละ invocation ไม่มี State ร่วมกัน
- **Timeout Limits**: จำกัดเวลาทำงาน (เช่น AWS Lambda สูงสุด 15 นาที)
- **Vendor Lock-in**: ผูกกับ Cloud Provider เฉพาะราย

### FaaS (Function as a Service)

```
Request → API Gateway → Function → Response
                            ↓
                     Database/Storage
```

---

## AWS Lambda กับ Scala

AWS Lambda รองรับ Scala ผ่าน JVM Runtime (Java 17/21)

### การตั้งค่า build.sbt

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.1"
ThisBuild / organization := "com.example"

lazy val root = (project in file("."))
  .settings(
    name := "scala-lambda",
    libraryDependencies ++= Seq(
      "com.amazonaws"  % "aws-lambda-java-core"   % "1.2.3",
      "com.amazonaws"  % "aws-lambda-java-events"  % "3.11.3",
      "io.circe"      %% "circe-core"              % "0.14.6",
      "io.circe"      %% "circe-generic"           % "0.14.6",
      "io.circe"      %% "circe-parser"            % "0.14.6",
      "software.amazon.awssdk" % "dynamodb"        % "2.21.0",
      "org.slf4j"      % "slf4j-simple"            % "2.0.9"
    ),
    // สร้าง Fat JAR
    assembly / assemblyMergeStrategy := {
      case PathList("META-INF", xs @ _*) => MergeStrategy.discard
      case "reference.conf"              => MergeStrategy.concat
      case x                             => MergeStrategy.first
    }
  )
```

### Lambda Handler พื้นฐาน

```scala
package com.example.lambda

import com.amazonaws.services.lambda.runtime.{Context, RequestHandler}
import com.amazonaws.services.lambda.runtime.events.{APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent}
import io.circe.*
import io.circe.generic.auto.*
import io.circe.parser.*
import io.circe.syntax.*
import java.util.{Map as JMap}
import scala.jdk.CollectionConverters.*

// Model classes
case class User(id: String, name: String, email: String)
case class CreateUserRequest(name: String, email: String)
case class ApiResponse[A](success: Boolean, data: Option[A], error: Option[String])

// Lambda Handler
class UserHandler extends RequestHandler[APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent]:
  
  override def handleRequest(
    event: APIGatewayProxyRequestEvent,
    context: Context
  ): APIGatewayProxyResponseEvent =
    val logger = context.getLogger
    logger.log(s"Processing ${event.getHttpMethod} ${event.getPath}")
    
    try
      event.getHttpMethod match
        case "GET"  => handleGet(event, context)
        case "POST" => handlePost(event, context)
        case method => 
          createResponse(405, ApiResponse(false, None, Some(s"Method $method not allowed")).asJson.noSpaces)
    catch
      case e: Exception =>
        logger.log(s"Error: ${e.getMessage}")
        createResponse(500, ApiResponse(false, None, Some("Internal server error")).asJson.noSpaces)
  
  private def handleGet(event: APIGatewayProxyRequestEvent, ctx: Context): APIGatewayProxyResponseEvent =
    val userId = Option(event.getPathParameters)
      .flatMap(p => Option(p.get("id")))
    
    userId match
      case Some(id) =>
        // ดึงข้อมูล User จาก DynamoDB (ตัวอย่าง)
        val user = User(id, "John Doe", "john@example.com")
        createResponse(200, ApiResponse(true, Some(user), None).asJson.noSpaces)
      case None =>
        createResponse(400, ApiResponse(false, None, Some("User ID required")).asJson.noSpaces)
  
  private def handlePost(event: APIGatewayProxyRequestEvent, ctx: Context): APIGatewayProxyResponseEvent =
    val body = Option(event.getBody).getOrElse("{}")
    
    decode[CreateUserRequest](body) match
      case Right(req) =>
        val newUser = User(
          id = java.util.UUID.randomUUID().toString,
          name = req.name,
          email = req.email
        )
        // บันทึกลง DynamoDB
        createResponse(201, ApiResponse(true, Some(newUser), None).asJson.noSpaces)
      case Left(err) =>
        createResponse(400, ApiResponse(false, None, Some(s"Invalid request: $err")).asJson.noSpaces)
  
  private def createResponse(statusCode: Int, body: String): APIGatewayProxyResponseEvent =
    val response = new APIGatewayProxyResponseEvent()
    response.setStatusCode(statusCode)
    response.setBody(body)
    response.setHeaders(JMap.of(
      "Content-Type", "application/json",
      "Access-Control-Allow-Origin", "*"
    ))
    response
```

### การใช้งาน DynamoDB กับ Lambda

```scala
package com.example.lambda

import software.amazon.awssdk.services.dynamodb.DynamoDbClient
import software.amazon.awssdk.services.dynamodb.model.*
import scala.jdk.CollectionConverters.*

class DynamoDbRepository(tableName: String):
  
  // ใช้ Lazy initialization เพื่อ reuse connection ระหว่าง warm invocations
  private lazy val client = DynamoDbClient.builder().build()
  
  def getUser(userId: String): Option[User] =
    val request = GetItemRequest.builder()
      .tableName(tableName)
      .key(Map("id" -> AttributeValue.builder().s(userId).build()).asJava)
      .build()
    
    val response = client.getItem(request)
    
    if response.hasItem then
      val item = response.item()
      Some(User(
        id = item.get("id").s(),
        name = item.get("name").s(),
        email = item.get("email").s()
      ))
    else
      None
  
  def saveUser(user: User): Unit =
    val item = Map(
      "id"    -> AttributeValue.builder().s(user.id).build(),
      "name"  -> AttributeValue.builder().s(user.name).build(),
      "email" -> AttributeValue.builder().s(user.email).build()
    ).asJava
    
    val request = PutItemRequest.builder()
      .tableName(tableName)
      .item(item)
      .build()
    
    client.putItem(request)
  
  def deleteUser(userId: String): Unit =
    val request = DeleteItemRequest.builder()
      .tableName(tableName)
      .key(Map("id" -> AttributeValue.builder().s(userId).build()).asJava)
      .build()
    
    client.deleteItem(request)
  
  def listUsers(limit: Int = 10): List[User] =
    val request = ScanRequest.builder()
      .tableName(tableName)
      .limit(limit)
      .build()
    
    client.scan(request).items().asScala.toList.map: item =>
      User(
        id = item.get("id").s(),
        name = item.get("name").s(),
        email = item.get("email").s()
      )
```

### Lambda ด้วย Event Sources อื่นๆ

```scala
package com.example.lambda

import com.amazonaws.services.lambda.runtime.{Context, RequestHandler}
import com.amazonaws.services.lambda.runtime.events.{SQSEvent, S3Event}
import scala.jdk.CollectionConverters.*

// SQS Event Handler
class SqsMessageHandler extends RequestHandler[SQSEvent, Void]:
  
  override def handleRequest(event: SQSEvent, context: Context): Void =
    val records = event.getRecords.asScala.toList
    
    records.foreach: record =>
      val messageId = record.getMessageId
      val body = record.getBody
      
      context.getLogger.log(s"Processing message $messageId: $body")
      processMessage(body, context)
    
    null
  
  private def processMessage(body: String, ctx: Context): Unit =
    // ประมวลผล message
    ctx.getLogger.log(s"Processed: $body")

// S3 Event Handler
class S3EventHandler extends RequestHandler[S3Event, String]:
  
  override def handleRequest(event: S3Event, context: Context): String =
    val records = event.getRecords.asScala.toList
    
    val processed = records.map: record =>
      val bucket = record.getS3.getBucket.getName
      val key = record.getS3.getObject.getKey
      
      context.getLogger.log(s"Processing S3 object: s3://$bucket/$key")
      processS3Object(bucket, key, context)
    
    s"Processed ${processed.length} S3 events"
  
  private def processS3Object(bucket: String, key: String, ctx: Context): String =
    // ดึงและประมวลผล S3 object
    s"Processed s3://$bucket/$key"
```

---

## Google Cloud Functions กับ Scala

```scala
// build.sbt สำหรับ GCF
libraryDependencies ++= Seq(
  "com.google.cloud.functions" % "functions-framework-api" % "1.1.0",
  "io.circe" %% "circe-core"    % "0.14.6",
  "io.circe" %% "circe-generic" % "0.14.6",
  "io.circe" %% "circe-parser"  % "0.14.6"
)
```

```scala
package com.example.gcf

import com.google.cloud.functions.{HttpFunction, HttpRequest, HttpResponse}
import io.circe.*
import io.circe.generic.auto.*
import io.circe.parser.*
import io.circe.syntax.*

case class GreetingRequest(name: String)
case class GreetingResponse(message: String, timestamp: Long)

class HelloWorldFunction extends HttpFunction:
  
  override def service(request: HttpRequest, response: HttpResponse): Unit =
    // ตั้งค่า CORS headers
    response.appendHeader("Access-Control-Allow-Origin", "*")
    response.appendHeader("Content-Type", "application/json")
    
    request.getMethod match
      case "OPTIONS" =>
        // Preflight request
        response.appendHeader("Access-Control-Allow-Methods", "GET, POST")
        response.appendHeader("Access-Control-Allow-Headers", "Content-Type")
        response.setStatusCode(204)
      
      case "POST" =>
        val body = new String(request.getInputStream.readAllBytes())
        
        decode[GreetingRequest](body) match
          case Right(req) =>
            val greeting = GreetingResponse(
              message = s"Hello, ${req.name}! Welcome to Serverless Scala!",
              timestamp = System.currentTimeMillis()
            )
            response.setStatusCode(200)
            response.getWriter.write(greeting.asJson.noSpaces)
          
          case Left(err) =>
            response.setStatusCode(400)
            response.getWriter.write(s"""{"error": "Invalid request: $err"}""")
      
      case method =>
        response.setStatusCode(405)
        response.getWriter.write(s"""{"error": "Method $method not allowed"}""")
```

### GCF Background Functions (Pub/Sub)

```scala
package com.example.gcf

import com.google.cloud.functions.{BackgroundFunction, Context}
import java.util.Base64
import io.circe.parser.*

case class PubSubMessage(data: String, attributes: Map[String, String])
case class OrderEvent(orderId: String, status: String, amount: Double)

class OrderProcessor extends BackgroundFunction[PubSubMessage]:
  
  override def accept(message: PubSubMessage, context: Context): Unit =
    // Decode base64 message
    val decodedData = new String(Base64.getDecoder.decode(message.data))
    
    decode[OrderEvent](decodedData) match
      case Right(order) =>
        processOrder(order, context)
      case Left(err) =>
        System.err.println(s"Failed to decode message: $err")
  
  private def processOrder(order: OrderEvent, ctx: Context): Unit =
    println(s"Processing order ${order.orderId} with status ${order.status}")
    
    order.status match
      case "CREATED"   => handleOrderCreated(order)
      case "PAID"      => handleOrderPaid(order)
      case "SHIPPED"   => handleOrderShipped(order)
      case "DELIVERED" => handleOrderDelivered(order)
      case status      => println(s"Unknown status: $status")
  
  private def handleOrderCreated(order: OrderEvent): Unit =
    println(s"New order created: ${order.orderId} for $${order.amount}")
  
  private def handleOrderPaid(order: OrderEvent): Unit =
    println(s"Order ${order.orderId} has been paid")
  
  private def handleOrderShipped(order: OrderEvent): Unit =
    println(s"Order ${order.orderId} has been shipped")
  
  private def handleOrderDelivered(order: OrderEvent): Unit =
    println(s"Order ${order.orderId} has been delivered")
```

---

## การ Optimize Cold Start

Cold Start คือเวลาที่ใช้ในการ initialize Function ครั้งแรกหรือหลังจากที่ไม่ได้ใช้งานนาน

### สาเหตุของ Cold Start บน JVM

1. **JVM Startup**: JVM ต้องใช้เวลา start
2. **Class Loading**: โหลด Classes จำนวนมาก
3. **JIT Compilation**: Just-in-Time compilation
4. **Framework Initialization**: Spring, Akka ฯลฯ

### เทคนิคลด Cold Start

```scala
// 1. ใช้ Lazy initialization สำหรับ resources
object DatabaseConnection:
  // จะ initialize เมื่อใช้งานครั้งแรกเท่านั้น
  lazy val client = createDatabaseClient()
  
  private def createDatabaseClient() =
    println("Initializing database connection...")
    // สร้าง connection pool
    ???

// 2. Keep dependencies outside handler class (static initialization)
// ทำให้ reuse ได้ระหว่าง warm invocations
object StaticResources:
  val httpClient = java.net.http.HttpClient.newHttpClient()
  val objectMapper = com.fasterxml.jackson.databind.ObjectMapper()

// 3. ลด dependencies ให้น้อยที่สุด
// แทนที่จะใช้ framework ใหญ่ ใช้ library ขนาดเล็ก
```

### GraalVM Native Image

```scala
// build.sbt - เพิ่ม GraalVM Native Image support
enablePlugins(GraalVMNativeImagePlugin)

graalVMNativeImageOptions ++= Seq(
  "--no-fallback",
  "--static",
  "-H:+ReportExceptionStackTraces",
  "--initialize-at-build-time",
  "-H:ReflectionConfigurationFiles=reflect-config.json"
)
```

```json
// reflect-config.json - สำหรับ Circe/JSON serialization
[
  {
    "name": "com.example.lambda.User",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "com.example.lambda.CreateUserRequest",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  }
]
```

### Provisioned Concurrency

```yaml
# serverless.yml - ตั้งค่า Provisioned Concurrency
functions:
  api:
    handler: com.example.lambda.UserHandler
    provisionedConcurrency: 5  # รักษา 5 instances ให้ warm ตลอด
    memorySize: 512
    timeout: 30
```

### Custom Runtime สำหรับ Performance

```scala
// bootstrap script for Custom Runtime
package com.example.lambda

import java.net.URI
import java.net.http.{HttpClient, HttpRequest, HttpResponse}
import io.circe.*
import io.circe.parser.*

object CustomRuntime:
  
  private val runtimeApi = sys.env("AWS_LAMBDA_RUNTIME_API")
  private val client = HttpClient.newHttpClient()
  
  def run(): Unit =
    while true do
      val (requestId, event) = getNextInvocation()
      
      try
        val result = processEvent(event)
        sendResponse(requestId, result)
      catch
        case e: Exception =>
          sendError(requestId, e.getMessage)
  
  private def getNextInvocation(): (String, String) =
    val url = s"http://$runtimeApi/2018-06-01/runtime/invocation/next"
    val request = HttpRequest.newBuilder(URI.create(url)).GET().build()
    val response = client.send(request, HttpResponse.BodyHandlers.ofString())
    
    val requestId = response.headers()
      .firstValue("Lambda-Runtime-Aws-Request-Id")
      .orElse("unknown")
    
    (requestId, response.body())
  
  private def processEvent(event: String): String =
    // ประมวลผล event
    s"""{"processed": true, "input": $event}"""
  
  private def sendResponse(requestId: String, result: String): Unit =
    val url = s"http://$runtimeApi/2018-06-01/runtime/invocation/$requestId/response"
    val request = HttpRequest.newBuilder(URI.create(url))
      .POST(HttpRequest.BodyPublishers.ofString(result))
      .header("Content-Type", "application/json")
      .build()
    client.send(request, HttpResponse.BodyHandlers.discarding())
  
  private def sendError(requestId: String, error: String): Unit =
    val url = s"http://$runtimeApi/2018-06-01/runtime/invocation/$requestId/error"
    val body = s"""{"errorMessage": "$error", "errorType": "RuntimeError"}"""
    val request = HttpRequest.newBuilder(URI.create(url))
      .POST(HttpRequest.BodyPublishers.ofString(body))
      .header("Content-Type", "application/json")
      .build()
    client.send(request, HttpResponse.BodyHandlers.discarding())
  
  def main(args: Array[String]): Unit = run()
```

---

## Serverless Framework

Serverless Framework เป็น Tool ที่ช่วยในการ deploy และจัดการ Serverless applications

### การติดตั้ง

```bash
npm install -g serverless
```

### serverless.yml แบบละเอียด

```yaml
service: scala-serverless-api

frameworkVersion: '3'

provider:
  name: aws
  runtime: java17
  region: ap-southeast-1
  memorySize: 512
  timeout: 30
  
  # Environment variables
  environment:
    USERS_TABLE: ${self:service}-${opt:stage, 'dev'}-users
    LOG_LEVEL: INFO
  
  # IAM permissions
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:GetItem
            - dynamodb:PutItem
            - dynamodb:DeleteItem
            - dynamodb:Scan
            - dynamodb:Query
          Resource:
            - !GetAtt UsersTable.Arn
        - Effect: Allow
          Action:
            - logs:CreateLogGroup
            - logs:CreateLogStream
            - logs:PutLogEvents
          Resource: "*"

package:
  artifact: target/scala-3.3.1/scala-lambda-assembly-0.1.0.jar

functions:
  # CRUD Functions
  createUser:
    handler: com.example.lambda.CreateUserHandler
    events:
      - http:
          path: /users
          method: post
          cors: true
  
  getUser:
    handler: com.example.lambda.GetUserHandler
    events:
      - http:
          path: /users/{id}
          method: get
          cors: true
  
  updateUser:
    handler: com.example.lambda.UpdateUserHandler
    events:
      - http:
          path: /users/{id}
          method: put
          cors: true
  
  deleteUser:
    handler: com.example.lambda.DeleteUserHandler
    events:
      - http:
          path: /users/{id}
          method: delete
          cors: true
  
  listUsers:
    handler: com.example.lambda.ListUsersHandler
    events:
      - http:
          path: /users
          method: get
          cors: true
  
  # Scheduled function
  cleanupExpiredSessions:
    handler: com.example.lambda.CleanupHandler
    events:
      - schedule: rate(1 hour)
  
  # SQS triggered function
  processOrders:
    handler: com.example.lambda.OrderProcessor
    events:
      - sqs:
          arn: !GetAtt OrdersQueue.Arn
          batchSize: 10

resources:
  Resources:
    # DynamoDB Table
    UsersTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.environment.USERS_TABLE}
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        BillingMode: PAY_PER_REQUEST
    
    # SQS Queue
    OrdersQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ${self:service}-${opt:stage, 'dev'}-orders
        VisibilityTimeout: 60
        MessageRetentionPeriod: 86400

plugins:
  - serverless-offline  # สำหรับ local development
```

### Local Development ด้วย serverless-offline

```bash
# ติดตั้ง plugin
npm install --save-dev serverless-offline

# รัน locally
sbt assembly
serverless offline start
```

---

## API Gateway Integration

### Custom Authorizer

```scala
package com.example.lambda

import com.amazonaws.services.lambda.runtime.{Context, RequestHandler}
import io.circe.*
import io.circe.generic.auto.*
import io.circe.parser.*
import java.util.{Map as JMap, List as JList}
import scala.jdk.CollectionConverters.*

// Token Authorizer
class JwtAuthorizer extends RequestHandler[JMap[String, Any], JMap[String, Any]]:
  
  override def handleRequest(
    event: JMap[String, Any],
    context: Context
  ): JMap[String, Any] =
    val token = event.get("authorizationToken").toString
    val methodArn = event.get("methodArn").toString
    
    try
      val claims = validateJwt(token)
      generatePolicy(claims.userId, "Allow", methodArn, claims)
    catch
      case e: Exception =>
        context.getLogger.log(s"Auth failed: ${e.getMessage}")
        generatePolicy("user", "Deny", methodArn, JwtClaims("", "", List.empty))
  
  private def validateJwt(token: String): JwtClaims =
    // ตรวจสอบ JWT token (ใช้ library จริงในโปรดักชัน)
    val cleanToken = token.replace("Bearer ", "")
    // decode และ verify token
    JwtClaims(
      userId = "user123",
      email = "user@example.com",
      roles = List("user", "admin")
    )
  
  private def generatePolicy(
    principalId: String,
    effect: String,
    resource: String,
    claims: JwtClaims
  ): JMap[String, Any] =
    val statement = Map(
      "Action"   -> "execute-api:Invoke",
      "Effect"   -> effect,
      "Resource" -> resource
    ).asJava
    
    val policyDocument = Map(
      "Version"   -> "2012-10-17",
      "Statement" -> JList.of(statement)
    ).asJava
    
    Map(
      "principalId"    -> principalId,
      "policyDocument" -> policyDocument,
      "context"        -> Map(
        "userId" -> claims.userId,
        "email"  -> claims.email,
        "roles"  -> claims.roles.mkString(",")
      ).asJava
    ).asJava

case class JwtClaims(userId: String, email: String, roles: List[String])
```

### Request/Response Transformation

```scala
package com.example.lambda

import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent
import scala.jdk.CollectionConverters.*

// Helper functions สำหรับดึงข้อมูลจาก API Gateway event
extension (event: APIGatewayProxyRequestEvent)
  
  def pathParam(name: String): Option[String] =
    Option(event.getPathParameters)
      .flatMap(p => Option(p.get(name)))
  
  def queryParam(name: String): Option[String] =
    Option(event.getQueryStringParameters)
      .flatMap(p => Option(p.get(name)))
  
  def header(name: String): Option[String] =
    Option(event.getHeaders)
      .flatMap(h => Option(h.get(name)))
      .orElse(
        Option(event.getHeaders)
          .flatMap(h => Option(h.get(name.toLowerCase)))
      )
  
  def multiValueQueryParam(name: String): List[String] =
    Option(event.getMultiValueQueryStringParameters)
      .flatMap(p => Option(p.get(name)))
      .map(_.asScala.toList)
      .getOrElse(List.empty)
  
  def bodyAs[A: io.circe.Decoder]: Either[io.circe.Error, A] =
    io.circe.parser.decode[A](Option(event.getBody).getOrElse("{}"))
```

---

## ตัวอย่าง Serverless API แบบครบถ้วน

### โครงสร้างโปรเจกต์

```
scala-serverless-api/
├── build.sbt
├── project/
│   └── plugins.sbt
├── serverless.yml
├── src/
│   └── main/
│       └── scala/
│           └── com/example/
│               ├── model/
│               │   ├── User.scala
│               │   ├── Product.scala
│               │   └── Order.scala
│               ├── repository/
│               │   ├── UserRepository.scala
│               │   └── OrderRepository.scala
│               ├── service/
│               │   ├── UserService.scala
│               │   └── OrderService.scala
│               └── handler/
│                   ├── UserHandler.scala
│                   └── OrderHandler.scala
└── test/
    └── scala/
        └── com/example/
            └── handler/
                └── UserHandlerSpec.scala
```

### Model Layer

```scala
package com.example.model

import io.circe.{Encoder, Decoder}
import io.circe.generic.auto.*
import java.time.Instant

// User model
case class User(
  id: String,
  name: String,
  email: String,
  createdAt: Long = System.currentTimeMillis(),
  updatedAt: Long = System.currentTimeMillis()
)

// Product model
case class Product(
  id: String,
  name: String,
  price: Double,
  stock: Int,
  category: String
)

// Order model
case class Order(
  id: String,
  userId: String,
  items: List[OrderItem],
  totalAmount: Double,
  status: OrderStatus,
  createdAt: Long = System.currentTimeMillis()
)

case class OrderItem(
  productId: String,
  quantity: Int,
  unitPrice: Double
)

enum OrderStatus:
  case Pending, Confirmed, Shipped, Delivered, Cancelled

// Request/Response DTOs
case class CreateUserRequest(name: String, email: String)
case class UpdateUserRequest(name: Option[String], email: Option[String])
case class CreateOrderRequest(userId: String, items: List[OrderItemRequest])
case class OrderItemRequest(productId: String, quantity: Int)

case class PaginatedResponse[A](
  items: List[A],
  total: Int,
  page: Int,
  pageSize: Int,
  hasMore: Boolean
)

case class ErrorResponse(code: String, message: String, details: Option[String] = None)
```

### Repository Layer

```scala
package com.example.repository

import com.example.model.*
import software.amazon.awssdk.services.dynamodb.DynamoDbClient
import software.amazon.awssdk.services.dynamodb.model.*
import scala.jdk.CollectionConverters.*

class UserRepository(tableName: String):
  
  private lazy val client = DynamoDbClient.builder().build()
  
  def findById(id: String): Option[User] =
    val request = GetItemRequest.builder()
      .tableName(tableName)
      .key(Map("id" -> strAttr(id)).asJava)
      .build()
    
    val response = client.getItem(request)
    if response.hasItem then Some(itemToUser(response.item())) else None
  
  def findByEmail(email: String): Option[User] =
    val request = QueryRequest.builder()
      .tableName(tableName)
      .indexName("email-index")
      .keyConditionExpression("email = :email")
      .expressionAttributeValues(Map(":email" -> strAttr(email)).asJava)
      .build()
    
    val response = client.query(request)
    response.items().asScala.headOption.map(itemToUser)
  
  def save(user: User): User =
    val item = Map(
      "id"        -> strAttr(user.id),
      "name"      -> strAttr(user.name),
      "email"     -> strAttr(user.email),
      "createdAt" -> numAttr(user.createdAt.toString),
      "updatedAt" -> numAttr(user.updatedAt.toString)
    ).asJava
    
    val request = PutItemRequest.builder()
      .tableName(tableName)
      .item(item)
      .build()
    
    client.putItem(request)
    user
  
  def update(id: String, req: UpdateUserRequest): Option[User] =
    val updates = List(
      req.name.map(n => ("name", strAttr(n))),
      req.email.map(e => ("email", strAttr(e)))
    ).flatten
    
    if updates.isEmpty then return findById(id)
    
    val updateExpression = "SET " + updates.map((k, _) => s"$k = :$k").mkString(", ")
    val expressionValues = updates.map((k, v) => s":$k" -> v).toMap.asJava
    
    val request = UpdateItemRequest.builder()
      .tableName(tableName)
      .key(Map("id" -> strAttr(id)).asJava)
      .updateExpression(updateExpression)
      .expressionAttributeValues(expressionValues)
      .returnValues(ReturnValue.ALL_NEW)
      .build()
    
    val response = client.updateItem(request)
    if response.hasAttributes then Some(itemToUser(response.attributes())) else None
  
  def delete(id: String): Boolean =
    val request = DeleteItemRequest.builder()
      .tableName(tableName)
      .key(Map("id" -> strAttr(id)).asJava)
      .conditionExpression("attribute_exists(id)")
      .build()
    
    try
      client.deleteItem(request)
      true
    catch
      case _: ConditionalCheckFailedException => false
  
  def list(limit: Int = 20, lastKey: Option[String] = None): (List[User], Option[String]) =
    val builder = ScanRequest.builder()
      .tableName(tableName)
      .limit(limit)
    
    lastKey.foreach: key =>
      builder.exclusiveStartKey(Map("id" -> strAttr(key)).asJava)
    
    val response = client.scan(builder.build())
    val users = response.items().asScala.toList.map(itemToUser)
    val nextKey = Option(response.lastEvaluatedKey())
      .filter(_.nonEmpty)
      .flatMap(k => Option(k.get("id")).map(_.s()))
    
    (users, nextKey)
  
  private def strAttr(value: String) = AttributeValue.builder().s(value).build()
  private def numAttr(value: String) = AttributeValue.builder().n(value).build()
  
  private def itemToUser(item: java.util.Map[String, AttributeValue]): User =
    User(
      id = item.get("id").s(),
      name = item.get("name").s(),
      email = item.get("email").s(),
      createdAt = item.get("createdAt").n().toLong,
      updatedAt = item.get("updatedAt").n().toLong
    )
```

### Service Layer

```scala
package com.example.service

import com.example.model.*
import com.example.repository.UserRepository
import java.util.UUID

class UserService(userRepo: UserRepository):
  
  def createUser(req: CreateUserRequest): Either[ServiceError, User] =
    // Validate input
    val validations = List(
      validateName(req.name),
      validateEmail(req.email)
    ).collect { case Left(err) => err }
    
    if validations.nonEmpty then
      Left(ValidationError(validations.mkString(", ")))
    else
      // Check duplicate email
      userRepo.findByEmail(req.email) match
        case Some(_) => Left(ConflictError(s"Email ${req.email} already exists"))
        case None =>
          val user = User(
            id = UUID.randomUUID().toString,
            name = req.name.trim,
            email = req.email.toLowerCase.trim
          )
          Right(userRepo.save(user))
  
  def getUser(id: String): Either[ServiceError, User] =
    userRepo.findById(id) match
      case Some(user) => Right(user)
      case None       => Left(NotFoundError(s"User $id not found"))
  
  def updateUser(id: String, req: UpdateUserRequest): Either[ServiceError, User] =
    // Verify user exists
    userRepo.findById(id) match
      case None => Left(NotFoundError(s"User $id not found"))
      case Some(_) =>
        // Validate updates
        val emailValidation = req.email.map(validateEmail).getOrElse(Right(()))
        emailValidation match
          case Left(err) => Left(ValidationError(err))
          case Right(_) =>
            userRepo.update(id, req) match
              case Some(user) => Right(user)
              case None       => Left(InternalError("Update failed"))
  
  def deleteUser(id: String): Either[ServiceError, Unit] =
    if userRepo.delete(id) then Right(())
    else Left(NotFoundError(s"User $id not found"))
  
  def listUsers(page: Int, pageSize: Int): PaginatedResponse[User] =
    val (users, _) = userRepo.list(pageSize)
    PaginatedResponse(
      items = users,
      total = users.length,
      page = page,
      pageSize = pageSize,
      hasMore = users.length == pageSize
    )
  
  private def validateName(name: String): Either[String, Unit] =
    if name.trim.isEmpty then Left("Name cannot be empty")
    else if name.length > 100 then Left("Name too long (max 100 chars)")
    else Right(())
  
  private def validateEmail(email: String): Either[String, Unit] =
    val emailRegex = """^[^\s@]+@[^\s@]+\.[^\s@]+$""".r
    if emailRegex.matches(email) then Right(())
    else Left(s"Invalid email format: $email")

// Error types
sealed trait ServiceError
case class NotFoundError(message: String) extends ServiceError
case class ConflictError(message: String) extends ServiceError
case class ValidationError(message: String) extends ServiceError
case class InternalError(message: String) extends ServiceError
```

### Handler Layer

```scala
package com.example.handler

import com.amazonaws.services.lambda.runtime.{Context, RequestHandler}
import com.amazonaws.services.lambda.runtime.events.{APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent}
import com.example.model.*
import com.example.repository.UserRepository
import com.example.service.*
import io.circe.*
import io.circe.generic.auto.*
import io.circe.syntax.*
import java.util.{Map as JMap}

class UserHandler extends RequestHandler[APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent]:
  
  // Static initialization - shared across warm invocations
  private val tableName = sys.env.getOrElse("USERS_TABLE", "users")
  private val userRepo = UserRepository(tableName)
  private val userService = UserService(userRepo)
  
  override def handleRequest(
    event: APIGatewayProxyRequestEvent,
    context: Context
  ): APIGatewayProxyResponseEvent =
    
    val (method, path) = (event.getHttpMethod, event.getPath)
    
    context.getLogger.log(s"$method $path")
    
    try
      (method, path) match
        case ("POST", "/users")         => createUser(event)
        case ("GET", p) if p.matches("/users/[^/]+") => 
          getUser(event)
        case ("PUT", p) if p.matches("/users/[^/]+") => 
          updateUser(event)
        case ("DELETE", p) if p.matches("/users/[^/]+") => 
          deleteUser(event)
        case ("GET", "/users")          => listUsers(event)
        case _ =>
          errorResponse(404, "NOT_FOUND", "Endpoint not found")
    catch
      case e: Exception =>
        context.getLogger.log(s"Unhandled error: ${e.getMessage}")
        errorResponse(500, "INTERNAL_ERROR", "Internal server error")
  
  private def createUser(event: APIGatewayProxyRequestEvent): APIGatewayProxyResponseEvent =
    event.bodyAs[CreateUserRequest] match
      case Left(err) =>
        errorResponse(400, "INVALID_REQUEST", s"Invalid JSON: $err")
      case Right(req) =>
        userService.createUser(req) match
          case Right(user) => jsonResponse(201, user.asJson)
          case Left(ValidationError(msg)) => errorResponse(400, "VALIDATION_ERROR", msg)
          case Left(ConflictError(msg))   => errorResponse(409, "CONFLICT", msg)
          case Left(err)                  => errorResponse(500, "INTERNAL_ERROR", err.toString)
  
  private def getUser(event: APIGatewayProxyRequestEvent): APIGatewayProxyResponseEvent =
    event.pathParam("id") match
      case None => errorResponse(400, "BAD_REQUEST", "User ID required")
      case Some(id) =>
        userService.getUser(id) match
          case Right(user)             => jsonResponse(200, user.asJson)
          case Left(NotFoundError(msg)) => errorResponse(404, "NOT_FOUND", msg)
          case Left(err)               => errorResponse(500, "INTERNAL_ERROR", err.toString)
  
  private def updateUser(event: APIGatewayProxyRequestEvent): APIGatewayProxyResponseEvent =
    event.pathParam("id") match
      case None => errorResponse(400, "BAD_REQUEST", "User ID required")
      case Some(id) =>
        event.bodyAs[UpdateUserRequest] match
          case Left(err) => errorResponse(400, "INVALID_REQUEST", s"Invalid JSON: $err")
          case Right(req) =>
            userService.updateUser(id, req) match
              case Right(user)              => jsonResponse(200, user.asJson)
              case Left(NotFoundError(msg)) => errorResponse(404, "NOT_FOUND", msg)
              case Left(ValidationError(msg)) => errorResponse(400, "VALIDATION_ERROR", msg)
              case Left(err)                => errorResponse(500, "INTERNAL_ERROR", err.toString)
  
  private def deleteUser(event: APIGatewayProxyRequestEvent): APIGatewayProxyResponseEvent =
    event.pathParam("id") match
      case None => errorResponse(400, "BAD_REQUEST", "User ID required")
      case Some(id) =>
        userService.deleteUser(id) match
          case Right(_)                 => emptyResponse(204)
          case Left(NotFoundError(msg)) => errorResponse(404, "NOT_FOUND", msg)
          case Left(err)               => errorResponse(500, "INTERNAL_ERROR", err.toString)
  
  private def listUsers(event: APIGatewayProxyRequestEvent): APIGatewayProxyResponseEvent =
    val page     = event.queryParam("page").flatMap(_.toIntOption).getOrElse(1)
    val pageSize = event.queryParam("pageSize").flatMap(_.toIntOption).getOrElse(20)
    
    val result = userService.listUsers(page, pageSize)
    jsonResponse(200, result.asJson)
  
  private def jsonResponse(statusCode: Int, body: Json): APIGatewayProxyResponseEvent =
    val response = new APIGatewayProxyResponseEvent()
    response.setStatusCode(statusCode)
    response.setBody(body.noSpaces)
    response.setHeaders(JMap.of(
      "Content-Type",                 "application/json",
      "Access-Control-Allow-Origin",  "*",
      "Access-Control-Allow-Headers", "Content-Type, Authorization",
      "Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS"
    ))
    response
  
  private def errorResponse(statusCode: Int, code: String, message: String): APIGatewayProxyResponseEvent =
    jsonResponse(statusCode, ErrorResponse(code, message).asJson)
  
  private def emptyResponse(statusCode: Int): APIGatewayProxyResponseEvent =
    val response = new APIGatewayProxyResponseEvent()
    response.setStatusCode(statusCode)
    response
```

### Unit Tests

```scala
package com.example.handler

import com.amazonaws.services.lambda.runtime.events.APIGatewayProxyRequestEvent
import com.example.model.*
import com.example.repository.UserRepository
import com.example.service.UserService
import io.circe.generic.auto.*
import io.circe.parser.*
import org.scalatest.funsuite.AnyFunSuite
import org.scalatest.matchers.should.Matchers
import org.mockito.Mockito.*
import org.mockito.ArgumentMatchers.*
import java.util.Map as JMap

class UserHandlerSpec extends AnyFunSuite with Matchers:
  
  def makeEvent(method: String, path: String, body: String = null, 
                pathParams: JMap[String, String] = null): APIGatewayProxyRequestEvent =
    val event = new APIGatewayProxyRequestEvent()
    event.setHttpMethod(method)
    event.setPath(path)
    event.setBody(body)
    event.setPathParameters(pathParams)
    event
  
  test("POST /users - creates user successfully"):
    // สร้าง mock context
    val mockContext = mock(classOf[com.amazonaws.services.lambda.runtime.Context])
    val mockLogger = mock(classOf[com.amazonaws.services.lambda.runtime.LambdaLogger])
    when(mockContext.getLogger).thenReturn(mockLogger)
    
    val handler = new UserHandler()
    val event = makeEvent("POST", "/users", """{"name": "Alice", "email": "alice@example.com"}""")
    
    val response = handler.handleRequest(event, mockContext)
    
    response.getStatusCode shouldBe 201
    val body = decode[User](response.getBody)
    body.isRight shouldBe true
    body.map(_.name) shouldBe Right("Alice")
  
  test("GET /users/{id} - returns 404 for unknown user"):
    val mockContext = mock(classOf[com.amazonaws.services.lambda.runtime.Context])
    val mockLogger = mock(classOf[com.amazonaws.services.lambda.runtime.LambdaLogger])
    when(mockContext.getLogger).thenReturn(mockLogger)
    
    val handler = new UserHandler()
    val event = makeEvent("GET", "/users/unknown-id", pathParams = JMap.of("id", "unknown-id"))
    
    val response = handler.handleRequest(event, mockContext)
    response.getStatusCode shouldBe 404
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **แนวคิด Serverless**: FaaS, ข้อดีข้อเสีย และ use cases ที่เหมาะสม
2. **AWS Lambda กับ Scala**: การสร้าง Handler สำหรับ HTTP, SQS, และ S3 events
3. **Google Cloud Functions**: HTTP Functions และ Background Functions
4. **Cold Start Optimization**: Lazy initialization, GraalVM Native Image, Provisioned Concurrency
5. **Serverless Framework**: การ configure และ deploy serverless applications
6. **API Gateway**: Custom Authorizers, request/response handling
7. **Complete Example**: Architecture แบบ layered (Model, Repository, Service, Handler)

### เมื่อใช้ Serverless

- **เหมาะสม**: Event-driven processing, APIs ที่มี variable traffic, Microservices ขนาดเล็ก
- **ไม่เหมาะสม**: Long-running processes, Real-time applications ที่ต้องการ low latency, Applications ที่ต้องการ State

---

*[← ตอนที่ 84: Advanced Streaming](part-84-advanced-streaming.md) | [ตอนที่ 86: Type-Driven Development →](part-86-type-driven.md)*
