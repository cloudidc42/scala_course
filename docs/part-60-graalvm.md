# Part 60: GraalVM Native Image

## สารบัญ
1. [GraalVM คืออะไร](#graalvm-คืออะไร)
2. [ประโยชน์ของ Native Image](#ประโยชน์ของ-native-image)
3. [การติดตั้งและตั้งค่า](#การติดตั้งและตั้งค่า)
4. [Build Configuration สำหรับ Scala](#build-configuration-สำหรับ-scala)
5. [Reflection Configuration](#reflection-configuration)
6. [Native Image กับ sbt-native-packager](#native-image-กับ-sbt-native-packager)
7. [ข้อจำกัดและ Workarounds](#ข้อจำกัดและ-workarounds)
8. [Performance Comparison: JVM vs Native](#performance-comparison-jvm-vs-native)
9. [ตัวอย่างแอปพลิเคชันครบถ้วน](#ตัวอย่างแอปพลิเคชันครบถ้วน)
10. [สรุป](#สรุป)

---

## GraalVM คืออะไร

GraalVM เป็น High-Performance JDK distribution จาก Oracle ที่รวมความสามารถสำคัญหลายอย่างไว้ด้วยกัน โดยฟีเจอร์ที่น่าสนใจที่สุดสำหรับ Scala developer คือ **Native Image** ซึ่งสามารถคอมไพล์ Java/Scala bytecode ให้เป็น native executable ได้โดยตรง

### สถาปัตยกรรม GraalVM

```
GraalVM Ecosystem:
┌─────────────────────────────────────────────────┐
│                   GraalVM JDK                   │
│  ┌────────────┐  ┌───────────┐  ┌───────────┐  │
│  │  JIT       │  │  Native   │  │Polyglot   │  │
│  │ Compiler   │  │  Image    │  │  Engine   │  │
│  └────────────┘  └───────────┘  └───────────┘  │
│       ↑                ↑               ↑        │
│   JVM Mode        AOT Mode        JS/Python/R   │
└─────────────────────────────────────────────────┘
```

### Ahead-of-Time (AOT) Compilation

แทนที่จะรัน bytecode ผ่าน JVM ในขณะ runtime, Native Image จะทำการ:

1. **Static Analysis** - วิเคราะห์โค้ดทั้งหมดตั้งแต่ entry point
2. **Dead Code Elimination** - ตัดโค้ดที่ไม่ถูกใช้งานออก
3. **AOT Compilation** - คอมไพล์เป็น machine code โดยตรง
4. **Linking** - รวม runtime library เข้าไปใน executable เดียว

```scala
// ตัวอย่างง่ายๆ: Hello World ที่จะถูก compile เป็น native
object HelloNative:
  def main(args: Array[String]): Unit =
    println("Hello from GraalVM Native Image!")
    println(s"Started at: ${java.time.Instant.now()}")
```

---

## ประโยชน์ของ Native Image

### 1. Instant Startup Time

JVM ปกติต้องใช้เวลา warm-up ก่อนจะทำงานได้เต็มที่ แต่ Native Image เริ่มทำงานได้ทันที:

```
Startup Time Comparison:
┌──────────────────┬──────────────┬────────────────┐
│   Application    │   JVM Mode   │  Native Image  │
├──────────────────┼──────────────┼────────────────┤
│ Hello World      │  ~150ms      │  ~5ms          │
│ Simple HTTP API  │  ~2,000ms    │  ~50ms         │
│ Spring Boot App  │  ~8,000ms    │  ~100ms        │
│ Scala CLI Tool   │  ~500ms      │  ~20ms         │
└──────────────────┴──────────────┴────────────────┘
```

### 2. ลดการใช้หน่วยความจำ

```scala
// โปรแกรมประมวลผลไฟล์ CSV
import scala.io.Source
import java.nio.file.{Files, Paths}

object CsvProcessor:
  case class Record(id: Int, name: String, value: Double)

  def parseRecord(line: String): Option[Record] =
    line.split(",").toList match
      case id :: name :: value :: Nil =>
        for
          i <- id.trim.toIntOption
          v <- value.trim.toDoubleOption
        yield Record(i, name.trim, v)
      case _ => None

  def processFile(path: String): List[Record] =
    Source.fromFile(path)
      .getLines()
      .drop(1) // skip header
      .flatMap(parseRecord)
      .toList

  def main(args: Array[String]): Unit =
    val path = args.headOption.getOrElse("data.csv")
    val records = processFile(path)
    println(s"Processed ${records.length} records")
    val total = records.map(_.value).sum
    println(f"Total value: $total%.2f")

/*
Memory Usage Comparison:
- JVM mode:    ~80MB RSS (Resident Set Size)
- Native mode: ~15MB RSS

Native Image รวม Garbage Collector ที่เล็กกว่าและไม่โหลด
class metadata ที่ไม่ถูกใช้งาน
*/
```

### 3. ไม่ต้องติดตั้ง JVM

```bash
# Native executable ทำงานได้เลยโดยไม่ต้องมี JVM
./my-scala-app --config app.yaml

# vs JVM mode ที่ต้องการ JVM
java -jar my-scala-app.jar --config app.yaml
```

### 4. เหมาะสำหรับ Containerization

```dockerfile
# Multi-stage Docker build
FROM ghcr.io/graalvm/native-image:22 AS builder
WORKDIR /app
COPY . .
RUN ./sbt nativeImage

# Final image ขนาดเล็กมาก
FROM debian:bullseye-slim
WORKDIR /app
COPY --from=builder /app/target/native-image/myapp .
ENTRYPOINT ["./myapp"]

# ขนาด image:
# JVM-based: ~400MB
# Native Image: ~50MB
```

---

## การติดตั้งและตั้งค่า

### ติดตั้ง GraalVM

```bash
# ใช้ SDKMAN (แนะนำ)
sdk install java 22.0.2-graal

# ตรวจสอบ version
java -version
# output: java version "22.0.2" 2024-07-16
#         Java(TM) SE Runtime Environment Oracle GraalVM 22.0.2+9.1

# ติดตั้ง native-image tool
gu install native-image

# ตรวจสอบ
native-image --version
```

### ติดตั้งด้วย Nix

```nix
# shell.nix
{ pkgs ? import <nixpkgs> {} }:
pkgs.mkShell {
  buildInputs = with pkgs; [
    graalvm-ce
    sbt
    zlib
    gcc
  ];
}
```

### ตั้งค่า PATH

```bash
# ~/.bashrc หรือ ~/.zshrc
export GRAALVM_HOME=/usr/lib/jvm/graalvm
export PATH=$GRAALVM_HOME/bin:$PATH
export JAVA_HOME=$GRAALVM_HOME
```

---

## Build Configuration สำหรับ Scala

### build.sbt พื้นฐาน

```scala
// project/plugins.sbt
addSbtPlugin("org.scala-native" % "sbt-scala-native" % "0.5.0")
// หรือสำหรับ GraalVM native image
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.10.0")
```

```scala
// build.sbt
import com.typesafe.sbt.packager.graalvmnativeimage.GraalVMNativeImagePlugin

lazy val root = project
  .in(file("."))
  .enablePlugins(GraalVMNativeImagePlugin)
  .settings(
    name         := "scala-native-demo",
    version      := "0.1.0",
    scalaVersion := "3.4.2",

    // GraalVM Native Image settings
    graalVMNativeImageOptions ++= Seq(
      "--no-fallback",              // ห้าม fallback เป็น JVM
      "--enable-https",             // เปิดใช้ HTTPS
      "-H:+ReportExceptionStackTraces",
      "--initialize-at-build-time",
      "-H:+StaticExecutableWithDynamicLibC", // สำหรับ Linux
    ),

    // dependencies
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-core"   % "2.10.0",
      "io.circe"      %% "circe-core"  % "0.14.7",
      "io.circe"      %% "circe-parser"% "0.14.7",
    )
  )
```

### การใช้งาน sbt commands

```bash
# Build native image
sbt graalvm-native-image:packageBin

# รัน
./target/graalvm-native-image/scala-native-demo

# Build สำหรับ Docker
sbt graalvm-native-image:packageBin docker:publishLocal
```

### Native Image Options ที่สำคัญ

```scala
graalVMNativeImageOptions ++= Seq(
  // ---- Compatibility ----
  "--no-fallback",           // ไม่ fallback เป็น JVM เมื่อ build ล้มเหลว
  "--allow-incomplete-classpath",  // อนุญาต missing classes

  // ---- Optimization ----
  "-O2",                     // Optimization level 2
  "--gc=G1",                 // ใช้ G1 GC (ต้องการ GraalVM Enterprise)
  "--gc=epsilon",            // No-op GC สำหรับ short-lived processes

  // ---- Debug ----
  "-H:+ReportExceptionStackTraces",
  "-H:+PrintAnalysisCallTree",  // แสดง call tree (ใช้ debug)

  // ---- Features ----
  "--enable-http",
  "--enable-https",
  "--enable-all-security-services",

  // ---- Static Analysis ----
  "--initialize-at-build-time=scala.runtime",
  "--initialize-at-run-time=io.netty",  // Netty ต้องการ runtime init
)
```

---

## Reflection Configuration

### ปัญหา Reflection ใน Native Image

Native Image ทำ closed-world assumption: มันจะรวมเฉพาะโค้ดที่ถูก reference อย่าง static เท่านั้น Reflection ทำให้ static analysis ยาก

```scala
// โค้ดนี้ทำงานปกติบน JVM แต่อาจมีปัญหาบน Native Image
val className = "com.example.MyClass"
val clazz = Class.forName(className)           // dynamic class loading
val instance = clazz.getDeclaredConstructor().newInstance()
val method = clazz.getMethod("process", classOf[String])
method.invoke(instance, "hello")
```

### วิธีแก้: Reflection Configuration File

สร้างไฟล์ `src/main/resources/META-INF/native-image/reflect-config.json`:

```json
[
  {
    "name": "com.example.MyClass",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredFields": true,
    "allPublicFields": true
  },
  {
    "name": "com.example.MyClass$Companion",
    "allDeclaredConstructors": true,
    "allDeclaredMethods": true
  },
  {
    "name": "scala.runtime.BoxedUnit",
    "allDeclaredFields": true
  }
]
```

### ใช้ @RegisterForReflection Annotation

```scala
import io.quarkus.runtime.annotations.RegisterForReflection

// สำหรับ Quarkus
@RegisterForReflection
case class UserData(id: Long, name: String, email: String)

// สำหรับ Micronaut
import io.micronaut.core.annotation.ReflectiveAccess

@ReflectiveAccess
case class ProductData(
  sku: String,
  title: String,
  price: BigDecimal
)
```

### ใช้ Tracing Agent เพื่อ Auto-generate Config

```bash
# รัน app ด้วย tracing agent เพื่อ record reflection usage
java -agentlib:native-image-agent=config-output-dir=src/main/resources/META-INF/native-image \
     -jar target/scala-3.4.2/myapp-assembly-0.1.0.jar

# agent จะสร้างไฟล์เหล่านี้อัตโนมัติ:
# - reflect-config.json
# - resource-config.json
# - proxy-config.json
# - jni-config.json
# - serialization-config.json
```

### Configuration สำหรับ Libraries ยอดนิยม

```json
// สำหรับ Circe (JSON library)
[
  {
    "name": "io.circe.Encoder$",
    "allDeclaredMethods": true,
    "allDeclaredFields": true
  },
  {
    "name": "io.circe.Decoder$",
    "allDeclaredMethods": true
  }
]
```

```json
// สำหรับ SLF4J + Logback
[
  {
    "name": "ch.qos.logback.classic.Logger",
    "allDeclaredConstructors": true,
    "allPublicMethods": true
  },
  {
    "name": "ch.qos.logback.core.ConsoleAppender",
    "allDeclaredConstructors": true,
    "allPublicMethods": true,
    "allDeclaredFields": true
  }
]
```

---

## Native Image กับ sbt-native-packager

### ตั้งค่าสมบูรณ์

```scala
// project/plugins.sbt
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.10.0")
```

```scala
// build.sbt
import com.typesafe.sbt.packager.graalvmnativeimage.GraalVMNativeImagePlugin
import NativePackagerHelper._

lazy val root = project
  .in(file("."))
  .enablePlugins(GraalVMNativeImagePlugin, DockerPlugin)
  .settings(
    name         := "http-api",
    version      := "1.0.0",
    scalaVersion := "3.4.2",
    organization := "com.example",

    // Main class
    Compile / mainClass := Some("com.example.Main"),

    // Native Image compilation options
    graalVMNativeImageOptions ++= Seq(
      "--no-fallback",
      "--enable-http",
      "--enable-https",
      "-H:Name=http-api",
      "-H:Class=com.example.Main",
      "-H:ReflectionConfigurationFiles=../../src/main/resources/META-INF/native-image/reflect-config.json",
      "-H:ResourceConfigurationFiles=../../src/main/resources/META-INF/native-image/resource-config.json",
      "--initialize-at-build-time",
      "--initialize-at-run-time=io.netty.channel.DefaultFileRegion",
      "--initialize-at-run-time=io.netty.channel.epoll.Native",
    ),

    // Docker settings (สำหรับ build ใน container)
    dockerBaseImage := "ghcr.io/graalvm/native-image:22",

    // Dependencies
    libraryDependencies ++= Seq(
      "com.softwaremill.sttp.tapir" %% "tapir-core"            % "1.10.0",
      "com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync" % "1.10.0",
      "io.circe"                    %% "circe-core"             % "0.14.7",
      "io.circe"                    %% "circe-generic"          % "0.14.7",
      "io.circe"                    %% "circe-parser"           % "0.14.7",
      "ch.qos.logback"               % "logback-classic"        % "1.5.6",
    )
  )
```

### Script สำหรับ Build

```bash
#!/bin/bash
# build-native.sh

set -euo pipefail

echo "==> Building native image..."
sbt graalvm-native-image:packageBin

BINARY="./target/graalvm-native-image/http-api"

echo "==> Binary size: $(du -sh $BINARY | cut -f1)"
echo "==> Testing startup time..."

time $BINARY &
PID=$!
sleep 0.1

# ทดสอบ endpoint
curl -s http://localhost:8080/health && echo ""

kill $PID
echo "==> Build complete!"
```

### Build ใน Docker (สำหรับ Linux production)

```dockerfile
# Dockerfile.native-build
FROM ghcr.io/graalvm/native-image:22 AS builder

# ติดตั้ง sbt
RUN curl -L https://github.com/sbt/sbt/releases/download/v1.10.0/sbt-1.10.0.tgz | \
    tar -xz -C /usr/local

ENV PATH="/usr/local/sbt/bin:$PATH"

WORKDIR /build
COPY build.sbt .
COPY project/ project/

# Cache dependencies
RUN sbt update

COPY src/ src/
RUN sbt graalvm-native-image:packageBin

# Final minimal image
FROM debian:bullseye-slim
RUN apt-get update && apt-get install -y libz-dev && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=builder /build/target/graalvm-native-image/http-api .
EXPOSE 8080
CMD ["./http-api"]
```

---

## ข้อจำกัดและ Workarounds

### 1. Dynamic Class Loading

```scala
// ❌ ไม่ทำงานบน Native Image โดยตรง
val plugin = Class.forName(s"com.example.plugins.$name")
               .getDeclaredConstructor()
               .newInstance()

// ✅ วิธีแก้: ใช้ ServiceLoader pattern
import java.util.ServiceLoader
import scala.jdk.CollectionConverters.*

trait Plugin:
  def name: String
  def execute(input: String): String

// สร้าง resource file: META-INF/services/com.example.Plugin
// เนื้อหา: com.example.plugins.PluginA
//          com.example.plugins.PluginB

val plugins = ServiceLoader.load(classOf[Plugin])
                .iterator()
                .asScala
                .toList
```

```json
// resource-config.json ต้องรวม service files
{
  "resources": {
    "includes": [
      { "pattern": "META-INF/services/.*" }
    ]
  }
}
```

### 2. Serialization

```scala
// ❌ Java Serialization มีปัญหาบน Native Image
val baos = new java.io.ByteArrayOutputStream()
val oos = new java.io.ObjectOutputStream(baos)
oos.writeObject(someObject)  // อาจล้มเหลว

// ✅ ใช้ Circe หรือ upickle แทน
import io.circe.generic.auto.*
import io.circe.syntax.*

case class Config(host: String, port: Int, debug: Boolean)

val config = Config("localhost", 8080, false)
val json = config.asJson.noSpaces
// "{\"host\":\"localhost\",\"port\":8080,\"debug\":false}"

// Deserialize
import io.circe.parser.decode
val decoded = decode[Config](json)
```

### 3. Invokedynamic และ Lambda

```scala
// Native Image รองรับ lambda ส่วนใหญ่แล้วใน GraalVM 22+
// แต่บาง pattern อาจต้องระวัง

// ✅ ทำงานได้ดี
val f: Int => Int = x => x * 2
val result = List(1, 2, 3).map(f)

// ✅ ทำงานได้ดีกับ method references
val doubled = List(1, 2, 3).map(_ * 2)

// ⚠️ String concatenation ด้วย + ใน Java interop
// บางกรณีต้องเพิ่ม --initialize-at-build-time
```

### 4. Thread-local และ Global State

```scala
// ⚠️ Global mutable state ต้องระวัง
object Registry:
  // ❌ อาจมีปัญหาถ้า initialize ขณะ build time
  private var entries = Map.empty[String, String]

  // ✅ ใช้ lazy val หรือ @volatile
  @volatile private var initialized = false
  private lazy val entries2 = Map.empty[String, String]
```

```scala
// Workaround: ใช้ RuntimeInitialized
// ใน native-image.properties:
// Args = --initialize-at-run-time=com.example.Registry
```

### 5. Resources และ Files

```scala
// ❌ ไฟล์ resource ต้องถูก include ใน native image
val stream = getClass.getResourceAsStream("/config.properties")

// ต้องเพิ่มใน resource-config.json:
// { "resources": { "includes": [{"pattern": "config.properties"}] } }
```

```json
{
  "resources": {
    "includes": [
      { "pattern": "application.conf" },
      { "pattern": "logback.xml" },
      { "pattern": ".*\\.properties" },
      { "pattern": "META-INF/.*" }
    ]
  }
}
```

---

## Performance Comparison: JVM vs Native

### Benchmark Setup

```scala
// BenchmarkApp.scala - ใช้สำหรับทดสอบ performance
object BenchmarkApp:
  import java.time.{Duration, Instant}

  def measureTime[A](name: String, iterations: Int)(f: => A): Unit =
    val start = Instant.now()
    var i = 0
    while i < iterations do
      f
      i += 1
    val elapsed = Duration.between(start, Instant.now())
    val perOp = elapsed.toNanos / iterations
    println(f"$name: total=${elapsed.toMillis}ms, per-op=${perOp}ns")

  // Benchmark 1: String processing
  def stringBenchmark(): Unit =
    val words = List.fill(1000)("hello world scala")
    measureTime("String concat", 10000) {
      words.map(_.toUpperCase).mkString(", ")
    }

  // Benchmark 2: Collection operations
  def collectionBenchmark(): Unit =
    val nums = (1 to 10000).toVector
    measureTime("Collection ops", 1000) {
      nums.filter(_ % 2 == 0).map(_ * 3).sum
    }

  // Benchmark 3: JSON parsing
  def jsonBenchmark(): Unit =
    import io.circe.parser.*
    val json = """{"id":1,"name":"test","values":[1,2,3,4,5]}"""
    measureTime("JSON parse", 10000) {
      parse(json)
    }

  def main(args: Array[String]): Unit =
    println(s"=== Benchmark on ${if args.contains("--native") then "Native" else "JVM"} ===")
    stringBenchmark()
    collectionBenchmark()
    jsonBenchmark()
```

### ผลลัพธ์ที่คาดหวัง

```
=== Startup Time ===
JVM (cold):    ~2000ms
JVM (warm):    ~200ms
Native:        ~20ms

=== Throughput (after warmup) ===
            JVM      Native
String:     fast      fast
JSON:       fast      ~20% slower
Math:       fast      comparable
HTTP:       fast      ~10% slower

=== Memory (RSS) ===
JVM:        120MB baseline
Native:     15MB baseline

=== Docker Image Size ===
JVM:        ~250MB (with JRE)
Native:     ~45MB (self-contained)
```

### เมื่อไหร่ควรใช้ Native Image?

```
ใช้ Native Image เมื่อ:
✅ CLI tools ที่ต้องการ startup เร็ว
✅ Serverless functions (AWS Lambda, Cloud Functions)
✅ Microservices ที่ deploy ใน container จำนวนมาก
✅ Edge computing
✅ ต้องการ memory footprint เล็ก

ยังใช้ JVM เมื่อ:
✅ Long-running services ที่ต้องการ JIT optimization
✅ Applications ที่ใช้ Reflection มาก
✅ Heavy dynamic class loading
✅ ต้องการ debugging tools เต็มรูปแบบ
✅ Team ไม่มีประสบการณ์ GraalVM
```

---

## ตัวอย่างแอปพลิเคชันครบถ้วน

### HTTP API ด้วย Tapir + Native Image

```scala
// src/main/scala/com/example/Main.scala
package com.example

import sttp.tapir.*
import sttp.tapir.server.netty.sync.NettySyncServer
import sttp.tapir.generic.auto.*
import sttp.tapir.json.circe.*
import io.circe.generic.auto.*
import org.slf4j.LoggerFactory

case class User(id: Long, name: String, email: String)
case class CreateUserRequest(name: String, email: String)
case class ErrorResponse(code: String, message: String)

object UserRepository:
  private var users = Map(
    1L -> User(1, "Alice", "alice@example.com"),
    2L -> User(2, "Bob", "bob@example.com")
  )
  private var nextId = 3L

  def findById(id: Long): Option[User] = users.get(id)
  def findAll(): List[User] = users.values.toList.sortBy(_.id)
  def create(req: CreateUserRequest): User =
    val user = User(nextId, req.name, req.email)
    users = users + (nextId -> user)
    nextId += 1
    user

object Endpoints:
  val getUser: Endpoint[Unit, Long, ErrorResponse, User, Any] =
    endpoint.get
      .in("users" / path[Long]("id"))
      .errorOut(jsonBody[ErrorResponse])
      .out(jsonBody[User])

  val listUsers: Endpoint[Unit, Unit, ErrorResponse, List[User], Any] =
    endpoint.get
      .in("users")
      .errorOut(jsonBody[ErrorResponse])
      .out(jsonBody[List[User]])

  val createUser: Endpoint[Unit, CreateUserRequest, ErrorResponse, User, Any] =
    endpoint.post
      .in("users")
      .in(jsonBody[CreateUserRequest])
      .errorOut(jsonBody[ErrorResponse])
      .out(jsonBody[User])
      .out(statusCode(sttp.model.StatusCode.Created))

  val health: Endpoint[Unit, Unit, Unit, String, Any] =
    endpoint.get.in("health").out(stringBody)

object Main:
  private val logger = LoggerFactory.getLogger("Main")

  def main(args: Array[String]): Unit =
    val port = sys.env.getOrElse("PORT", "8080").toInt
    val host = sys.env.getOrElse("HOST", "0.0.0.0")

    val routes = List(
      Endpoints.getUser.serverLogic { id =>
        Right(UserRepository.findById(id)
          .toRight(ErrorResponse("NOT_FOUND", s"User $id not found")))
          .flatten
      },
      Endpoints.listUsers.serverLogic { _ =>
        Right(UserRepository.findAll())
      },
      Endpoints.createUser.serverLogic { req =>
        Right(UserRepository.create(req))
      },
      Endpoints.health.serverLogic { _ =>
        Right("OK")
      }
    )

    logger.info(s"Starting server on $host:$port")
    val startTime = System.currentTimeMillis()

    NettySyncServer()
      .host(host)
      .port(port)
      .addEndpoints(routes)
      .startAndWait()
```

### build.sbt สำหรับตัวอย่างนี้

```scala
// build.sbt
lazy val root = project
  .in(file("."))
  .enablePlugins(GraalVMNativeImagePlugin)
  .settings(
    name         := "user-api",
    version      := "1.0.0",
    scalaVersion := "3.4.2",

    Compile / mainClass := Some("com.example.Main"),

    graalVMNativeImageOptions ++= Seq(
      "--no-fallback",
      "--enable-http",
      "--enable-https",
      "-H:+ReportExceptionStackTraces",
      "--initialize-at-build-time",
      "--initialize-at-run-time=io.netty",
    ),

    libraryDependencies ++= Seq(
      "com.softwaremill.sttp.tapir" %% "tapir-netty-server-sync" % "1.10.0",
      "com.softwaremill.sttp.tapir" %% "tapir-json-circe"        % "1.10.0",
      "io.circe"                    %% "circe-generic"           % "0.14.7",
      "ch.qos.logback"               % "logback-classic"         % "1.5.6",
    )
  )
```

### ทดสอบ

```bash
# Build
sbt graalvm-native-image:packageBin

# รัน
./target/graalvm-native-image/user-api &

# ทดสอบ
curl http://localhost:8080/health
# OK

curl http://localhost:8080/users
# [{"id":1,"name":"Alice","email":"alice@example.com"},{"id":2,...}]

curl -X POST http://localhost:8080/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com"}'
# {"id":3,"name":"Charlie","email":"charlie@example.com"}

curl http://localhost:8080/users/3
# {"id":3,"name":"Charlie","email":"charlie@example.com"}

# ดู startup time ใน log:
# INFO Main - Starting server on 0.0.0.0:8080
# Server started in ~35ms! (เทียบกับ JVM ~2000ms)
```

---

## สรุป

GraalVM Native Image เป็นเครื่องมือที่ทรงพลังสำหรับ Scala applications โดยเฉพาะใน use cases ที่ต้องการ:

- **Startup time เร็ว**: CLI tools, serverless, edge computing
- **Memory ต่ำ**: การ deploy จำนวนมากใน container
- **Distribution ง่าย**: ไม่ต้องติดตั้ง JVM

สิ่งสำคัญที่ต้องระวัง:
- ✅ วางแผน Reflection configuration ตั้งแต่เริ่มต้น
- ✅ ใช้ tracing agent เพื่อ auto-generate config
- ✅ ทดสอบบน native binary ตั้งแต่เนิ่นๆ
- ✅ Build ใน Docker เพื่อให้ได้ binary ที่ compatible กับ production
- ⚠️ Native Image ไม่เหมาะกับทุก application

---

*[← Part 59: Advanced Patterns](part-59-advanced-patterns.md) | [Part 61: Akka Actors →](part-61-akka-actors.md)*
