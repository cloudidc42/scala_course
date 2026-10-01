# ส่วนที่ 80: Scala Native

## สารบัญ

1. [Scala Native คืออะไร](#scala-native-คืออะไร)
2. [Setup และ Build Configuration](#setup-และ-build-configuration)
3. [Interop กับ C Libraries](#interop-กับ-c-libraries)
4. [Memory Management: Zones](#memory-management-zones)
5. [POSIX Bindings](#posix-bindings)
6. [Performance Benchmarks vs JVM](#performance-benchmarks-vs-jvm)
7. [Complete CLI Tool Example](#complete-cli-tool-example)
8. [การ Optimize และ Advanced Features](#การ-optimize-และ-advanced-features)
9. [สรุป](#สรุป)

---

## Scala Native คืออะไร

Scala Native เป็น compiler ที่แปลง Scala code ให้เป็น native binary โดยใช้ LLVM เป็น backend แทนที่จะ run บน JVM ทำให้ได้ประโยชน์หลายอย่าง:

- **Instant startup**: ไม่มี JVM warmup time
- **Low memory footprint**: ไม่ต้องโหลด JVM runtime
- **Direct C interop**: เรียกใช้ C libraries ได้โดยตรง
- **Predictable performance**: ไม่มี GC pause แบบ JVM
- **Small binary size**: deploy ง่าย

### เปรียบเทียบ Scala Native vs JVM vs Scala.js

```
Feature          | JVM          | Scala Native    | Scala.js
-----------------|--------------|-----------------|------------------
Startup time     | ~100-500ms   | <10ms           | Browser-dependent
Memory overhead  | ~50-100MB    | ~2-5MB          | N/A
C interop        | JNI (complex)| Direct (simple) | N/A
GC               | Stop-world   | Immix/Commix    | JS engine GC
Target           | Any OS+JVM   | Linux/macOS/Win | Browser/Node.js
Ecosystem        | Huge         | Growing         | Growing
```

### Use Cases ที่เหมาะสม

```scala
// Scala Native เหมาะสำหรับ:
// 1. CLI tools ที่ต้องการ instant startup
// 2. System programming ที่ต้องการ C interop
// 3. Embedded systems
// 4. Microservices ที่ต้องการ low memory
// 5. Scripts ที่รัน frequently
```

---

## Setup และ Build Configuration

### การติดตั้ง Prerequisites

```bash
# Ubuntu/Debian
sudo apt-get install clang libgc-dev libunwind-dev libre2-dev

# macOS
brew install llvm bdw-gc libunwind

# ตรวจสอบ version
clang --version
# clang version 14.0.0 or higher

# ติดตั้ง sbt
cs install sbt
```

### Project Structure

```
my-native-app/
├── build.sbt
├── project/
│   ├── plugins.sbt
│   └── build.properties
└── src/
    └── main/
        └── scala/
            └── Main.scala
```

### Build Configuration

```scala
// project/plugins.sbt
addSbtPlugin("org.scala-native" % "sbt-scala-native" % "0.4.17")
```

```scala
// project/build.properties
sbt.version=1.9.7
```

```scala
// build.sbt
import scala.scalanative.build._

ThisBuild / scalaVersion := "3.3.1"

lazy val root = project
  .in(file("."))
  .enablePlugins(ScalaNativePlugin)
  .settings(
    name := "my-native-app",
    version := "0.1.0",
    
    // Scala Native settings
    nativeConfig ~= {
      _.withLTO(LTO.thin)           // Link Time Optimization
       .withMode(Mode.releaseFast)  // releaseFast, releaseFull, debug
       .withGC(GC.commix)          // GC: none, boehm, immix, commix
    },
    
    // dependencies
    libraryDependencies ++= Seq(
      "com.lihaoyi" %%% "utest" % "0.8.1" % Test,
      "com.lihaoyi" %%% "os-lib" % "0.9.1",
      "com.lihaoyi" %%% "upickle" % "3.1.3"
    )
  )
```

### Build และ Run

```bash
# Compile
sbt nativeCompile

# Run
sbt run

# หรือ run binary โดยตรง
./target/scala-3.3.1/my-native-app-out

# Build สำหรับ release
sbt 'set nativeConfig ~= { _.withMode(Mode.releaseFull) }' nativeCompile
```

### GC Options อธิบาย

```scala
// GC Options
// none   - ไม่มี GC, memory leak แต่เร็วที่สุด
// boehm  - Conservative GC, เหมาะสำหรับ legacy code
// immix  - Modern precise GC, default
// commix - Concurrent GC, ลด pause time

// สำหรับ CLI tools อายุสั้น
nativeConfig ~= { _.withGC(GC.none) }

// สำหรับ long-running services
nativeConfig ~= { _.withGC(GC.commix) }
```

---

## Interop กับ C Libraries

Scala Native มี `@extern` annotation สำหรับ declare C functions และ types

### Basic C Interop

```scala
import scala.scalanative.unsafe.*
import scala.scalanative.libc.string.*
import scala.scalanative.libc.stdio.*
import scala.scalanative.libc.stdlib.*

// declare C function
@extern
def strlen(s: CString): CSize = extern

// declare C struct
type Point = CStruct2[CInt, CInt]

@main def run(): Unit =
  // allocate memory
  Zone { implicit z =>
    val str = toCString("Hello, World!")
    val len = strlen(str)
    println(s"Length: $len")  // 13
    
    // ใช้ printf
    val fmt = toCString("Hello from C: %d\n")
    printf(fmt, 42)
  }
```

### การ Bind ไปยัง External C Library

```scala
// สร้าง binding สำหรับ libcurl
import scala.scalanative.unsafe.*

@extern
@link("curl")
object LibCurl:
  type CurlHandle = Ptr[Byte]
  type CurlOption = CInt
  
  // CURL options
  val CURLOPT_URL: CurlOption       = 10002
  val CURLOPT_WRITEFUNCTION: CurlOption = 20011
  val CURLOPT_WRITEDATA: CurlOption = 10001
  
  def curl_easy_init(): CurlHandle = extern
  def curl_easy_setopt(handle: CurlHandle, option: CurlOption, param: CVoidPtr): CInt = extern
  def curl_easy_perform(handle: CurlHandle): CInt = extern
  def curl_easy_cleanup(handle: CurlHandle): Unit = extern
  def curl_easy_strerror(code: CInt): CString = extern

// Usage
def httpGet(url: String): String =
  import LibCurl.*
  
  Zone { implicit z =>
    val handle = curl_easy_init()
    if handle == null then
      throw new RuntimeException("Failed to init curl")
    
    var responseData = new StringBuilder()
    
    // callback function type
    type WriteCallback = CFuncPtr4[Ptr[Byte], CSize, CSize, Ptr[Byte], CSize]
    
    val callback: WriteCallback = CFuncPtr4.fromScalaFunction4 { 
      (data: Ptr[Byte], size: CSize, nmemb: CSize, _: Ptr[Byte]) =>
        val bytes = new Array[Byte]((size * nmemb).toInt)
        for i <- bytes.indices do
          bytes(i) = data(i)
        responseData.append(new String(bytes, "UTF-8"))
        size * nmemb
    }
    
    curl_easy_setopt(handle, CURLOPT_URL, toCString(url).asInstanceOf[CVoidPtr])
    curl_easy_setopt(handle, CURLOPT_WRITEFUNCTION, callback.rawptr.asInstanceOf[CVoidPtr])
    
    val code = curl_easy_perform(handle)
    if code != 0 then
      val errMsg = fromCString(curl_easy_strerror(code))
      throw new RuntimeException(s"Curl error: $errMsg")
    
    curl_easy_cleanup(handle)
    responseData.toString()
  }
```

### Working กับ C Structs

```scala
import scala.scalanative.unsafe.*
import scala.scalanative.unsigned.*

// C struct equivalent:
// struct Person {
//   char* name;
//   int age;
//   double score;
// };
type Person = CStruct3[CString, CInt, CDouble]

object PersonOps:
  extension (p: Ptr[Person])
    def name: CString = p._1
    def name_=(s: CString): Unit = p._1 = s
    def age: CInt = p._2
    def age_=(a: CInt): Unit = p._2 = a
    def score: CDouble = p._3
    def score_=(s: CDouble): Unit = p._3 = s

@main def structDemo(): Unit =
  import PersonOps.*
  
  Zone { implicit z =>
    val person = alloc[Person]()
    person.name = toCString("Alice")
    person.age = 30
    person.score = 95.5
    
    println(s"Name: ${fromCString(person.name)}")
    println(s"Age: ${person.age}")
    println(s"Score: ${person.score}")
  }
```

### Arrays ใน Native

```scala
import scala.scalanative.unsafe.*

@main def arrayDemo(): Unit =
  Zone { implicit z =>
    // allocate array of 10 ints
    val arr = alloc[CInt](10)
    
    // fill array
    for i <- 0 until 10 do
      arr(i) = i * i
    
    // print array
    for i <- 0 until 10 do
      print(s"arr[$i] = ${arr(i)}, ")
    println()
    
    // array of structs
    type Point = CStruct2[CDouble, CDouble]
    val points = alloc[Point](5)
    
    for i <- 0 until 5 do
      points(i)._1 = i.toDouble
      points(i)._2 = (i * 2).toDouble
    
    for i <- 0 until 5 do
      println(s"Point($i): (${points(i)._1}, ${points(i)._2})")
  }
```

---

## Memory Management: Zones

Zone คือ region-based memory management ที่ Scala Native ให้มา

### Zone Basics

```scala
import scala.scalanative.unsafe.*

@main def zoneDemo(): Unit =
  // Zone จัดการ memory lifecycle
  Zone { implicit z =>
    // Memory ถูก allocate ใน zone
    val str1 = toCString("Hello")
    val str2 = toCString("World")
    
    // str1, str2 valid ตลอด zone
    println(fromCString(str1))
    println(fromCString(str2))
    
    // Nested zones
    Zone { implicit z2 =>
      val innerStr = toCString("Inner")
      println(fromCString(innerStr))
    }
    // innerStr freed here
    
    // str1, str2 still valid
    println(fromCString(str1))
  }
  // str1, str2 freed here automatically
```

### Custom Zone Allocations

```scala
import scala.scalanative.unsafe.*
import scala.scalanative.libc.stdlib.*

// allocate memory ใน zone
def allocInZone[T](n: Int)(implicit z: Zone): Ptr[T] =
  z.alloc(n.toLong * sizeof[T])

// allocate และ initialize
def allocStruct[T <: CStruct[?]]()(implicit z: Zone): Ptr[T] =
  val ptr = z.alloc(sizeof[T])
  // zero-initialize
  libc.string.memset(ptr, 0, sizeof[T])
  ptr.asInstanceOf[Ptr[T]]

@main def customAlloc(): Unit =
  Zone { implicit z =>
    type Matrix = CStruct4[Ptr[CDouble], CInt, CInt, CInt] // data, rows, cols, stride
    
    val rows = 3
    val cols = 3
    val data = allocInZone[CDouble](rows * cols)
    
    // fill matrix
    for r <- 0 until rows; c <- 0 until cols do
      data(r * cols + c) = (r * cols + c).toDouble
    
    // print matrix
    for r <- 0 until rows do
      for c <- 0 until cols do
        print(f"${data(r * cols + c)}%6.1f ")
      println()
  }
```

### Zone Performance Tips

```scala
import scala.scalanative.unsafe.*

// หลีกเลี่ยง allocate ใน hot path
// แทนที่ใช้ stack allocation ด้วย stackalloc
@main def stackAllocDemo(): Unit =
  // stackalloc - allocate บน stack (เร็วมาก)
  val buf = stackalloc[Byte](256)
  val fmt = stackalloc[CChar](64)
  
  // ใช้ snprintf
  import scala.scalanative.libc.stdio.snprintf
  snprintf(buf, 256, c"Result: %d\n", 42)
  
  print(fromCString(buf))
  
  // stackalloc ไม่ต้องการ Zone
  // freed เมื่อออกจาก scope
```

---

## POSIX Bindings

Scala Native มี POSIX bindings built-in

### File System Operations

```scala
import scala.scalanative.posix.unistd.*
import scala.scalanative.posix.fcntl.*
import scala.scalanative.posix.stat.*
import scala.scalanative.unsafe.*
import scala.scalanative.libc.errno.*

// อ่านไฟล์ด้วย POSIX
def readFile(path: String): Either[String, String] =
  Zone { implicit z =>
    val fd = open(toCString(path), O_RDONLY)
    if fd < 0 then
      return Left(s"Cannot open file: ${fromCString(strerror(errno))}")
    
    // get file size
    val statBuf = alloc[stat]()
    fstat(fd, statBuf)
    val fileSize = statBuf.st_size.toInt
    
    // read content
    val buf = alloc[Byte](fileSize + 1)
    val bytesRead = read(fd, buf, fileSize.toULong)
    close(fd)
    
    if bytesRead < 0 then
      Left(s"Read error: ${fromCString(strerror(errno))}")
    else
      buf(bytesRead.toInt) = 0 // null terminate
      Right(fromCString(buf))
  }

// เขียนไฟล์ด้วย POSIX
def writeFile(path: String, content: String): Either[String, Unit] =
  Zone { implicit z =>
    val fd = open(
      toCString(path), 
      O_WRONLY | O_CREAT | O_TRUNC, 
      0o644.toUInt
    )
    if fd < 0 then
      return Left(s"Cannot create file: ${fromCString(strerror(errno))}")
    
    val bytes = toCString(content)
    val len = content.length
    val written = write(fd, bytes, len.toULong)
    close(fd)
    
    if written < 0 then
      Left(s"Write error: ${fromCString(strerror(errno))}")
    else
      Right(())
  }
```

### Process และ Signal Handling

```scala
import scala.scalanative.posix.unistd.*
import scala.scalanative.posix.signal.*
import scala.scalanative.unsafe.*

// fork และ exec
def runCommand(cmd: String, args: Seq[String]): Int =
  Zone { implicit z =>
    val pid = fork()
    
    if pid == 0 then
      // child process
      val argvArray = (cmd +: args :+ null).map(
        s => if s != null then toCString(s) else null
      ).toArray
      
      val argv = alloc[CString](argvArray.length)
      for (arg, i) <- argvArray.zipWithIndex do
        argv(i) = arg
      
      execvp(toCString(cmd), argv)
      // ถ้า exec ล้มเหลว
      sys.exit(1)
    else if pid > 0 then
      // parent process - รอ child
      val status = alloc[CInt]()
      waitpid(pid, status, 0)
      status(0)
    else
      -1 // fork failed
  }

// Signal handler
@main def signalDemo(): Unit =
  // install SIGINT handler
  val handler: CFuncPtr1[CInt, Unit] = CFuncPtr1.fromScalaFunction1 { 
    (sig: CInt) =>
      println(s"\nReceived signal $sig, cleaning up...")
      sys.exit(0)
  }
  
  signal(SIGINT, handler)
  
  println("Press Ctrl+C to exit...")
  while true do
    Thread.sleep(1000)
    println("Running...")
```

### Network Programming ด้วย POSIX Sockets

```scala
import scala.scalanative.posix.sys.socket.*
import scala.scalanative.posix.netinet.in.*
import scala.scalanative.posix.arpa.inet.*
import scala.scalanative.posix.unistd.*
import scala.scalanative.unsafe.*

// Simple TCP server
def createTcpServer(port: Int): Int =
  Zone { implicit z =>
    val serverFd = socket(AF_INET, SOCK_STREAM, 0)
    if serverFd < 0 then
      throw new RuntimeException("Failed to create socket")
    
    // allow reuse address
    val optval = alloc[CInt]()
    optval(0) = 1
    setsockopt(serverFd, SOL_SOCKET, SO_REUSEADDR, optval, sizeof[CInt].toUInt)
    
    // bind
    val addr = alloc[sockaddr_in]()
    addr.sin_family = AF_INET.toUShort
    addr.sin_addr.s_addr = INADDR_ANY
    addr.sin_port = htons(port.toUShort)
    
    if bind(serverFd, addr.asInstanceOf[Ptr[sockaddr]], sizeof[sockaddr_in].toUInt) < 0 then
      throw new RuntimeException("Failed to bind")
    
    if listen(serverFd, 10) < 0 then
      throw new RuntimeException("Failed to listen")
    
    println(s"Server listening on port $port")
    serverFd
  }
```

---

## Performance Benchmarks vs JVM

### Fibonacci Benchmark

```scala
// Fibonacci แบบ recursive (test computation speed)
def fibonacci(n: Int): Long =
  if n <= 1 then n.toLong
  else fibonacci(n - 1) + fibonacci(n - 2)

@main def fibBenchmark(): Unit =
  val n = 45
  val start = System.currentTimeMillis()
  val result = fibonacci(n)
  val elapsed = System.currentTimeMillis() - start
  
  println(s"fibonacci($n) = $result")
  println(s"Time: ${elapsed}ms")

// ผลลัพธ์โดยประมาณ:
// JVM (cold):    ~12,000ms
// JVM (warm):    ~8,000ms
// Scala Native:  ~5,000ms (consistent)
// C (gcc -O2):   ~4,500ms
```

### String Processing Benchmark

```scala
import scala.scalanative.unsafe.*

def countWords(text: String): Int =
  text.split("\\s+").length

@main def stringBenchmark(): Unit =
  val text = "The quick brown fox jumps over the lazy dog " * 10000
  val iterations = 1000
  
  val start = System.nanoTime()
  var total = 0L
  for _ <- 0 until iterations do
    total += countWords(text)
  val elapsed = (System.nanoTime() - start) / 1_000_000
  
  println(s"Total words: $total")
  println(s"Time: ${elapsed}ms for $iterations iterations")
  println(s"Average: ${elapsed.toDouble / iterations}ms per iteration")
```

### Memory Allocation Benchmark

```scala
import scala.scalanative.unsafe.*

// benchmark allocation ใน Zone vs heap
def benchmarkZoneAlloc(n: Int): Long =
  val start = System.nanoTime()
  for _ <- 0 until n do
    Zone { implicit z =>
      val arr = alloc[CInt](1000)
      for i <- 0 until 1000 do
        arr(i) = i
      arr(999) // prevent optimization
    }
  System.nanoTime() - start

def benchmarkHeapAlloc(n: Int): Long =
  val start = System.nanoTime()
  for _ <- 0 until n do
    val arr = new Array[Int](1000)
    for i <- 0 until 1000 do
      arr(i) = i
    arr(999) // prevent optimization
  System.nanoTime() - start

@main def memBenchmark(): Unit =
  val n = 10000
  val zoneTime = benchmarkZoneAlloc(n)
  val heapTime = benchmarkHeapAlloc(n)
  
  println(s"Zone allocation: ${zoneTime / 1_000_000}ms")
  println(s"Heap allocation: ${heapTime / 1_000_000}ms")
  println(s"Zone speedup: ${heapTime.toDouble / zoneTime}x")
```

### Benchmark Results สรุป

```
Benchmark              | JVM (warm) | Scala Native | Native/JVM
-----------------------|------------|--------------|------------
Startup time           | ~250ms     | ~5ms         | 50x faster
Fibonacci(45)          | ~8,000ms   | ~5,000ms     | 1.6x faster
String processing      | ~50ms      | ~45ms        | 1.1x faster
Memory allocation      | ~100ms     | ~40ms        | 2.5x faster
JSON parsing (small)   | ~2ms       | ~3ms         | 0.7x
Binary search (1M)     | ~1ms       | ~0.8ms       | 1.25x faster
File I/O (10MB)        | ~20ms      | ~15ms        | 1.3x faster
```

---

## Complete CLI Tool Example

สร้าง CLI tool สำหรับ process CSV files

### Project Setup

```scala
// build.sbt
import scala.scalanative.build.*

ThisBuild / scalaVersion := "3.3.1"

lazy val root = project
  .in(file("."))
  .enablePlugins(ScalaNativePlugin)
  .settings(
    name := "csv-processor",
    version := "1.0.0",
    
    nativeConfig ~= {
      _.withLTO(LTO.thin)
       .withMode(Mode.releaseFast)
       .withGC(GC.commix)
    },
    
    libraryDependencies ++= Seq(
      "com.lihaoyi" %%% "os-lib" % "0.9.1",
      "com.lihaoyi" %%% "upickle" % "3.1.3"
    )
  )
```

### Main Implementation

```scala
// src/main/scala/Main.scala
import scala.scalanative.unsafe.*
import scala.scalanative.libc.stdio.*
import scala.scalanative.posix.unistd.*

case class CsvRow(values: Map[String, String])

object CsvParser:
  def parse(content: String): (Seq[String], Seq[CsvRow]) =
    val lines = content.split("\n").filter(_.nonEmpty)
    if lines.isEmpty then return (Seq.empty, Seq.empty)
    
    val headers = parseRow(lines.head)
    val rows = lines.tail.map { line =>
      val values = parseRow(line)
      CsvRow(headers.zip(values).toMap)
    }
    
    (headers, rows.toSeq)
  
  private def parseRow(line: String): Seq[String] =
    var inQuotes = false
    var current = new StringBuilder()
    val fields = collection.mutable.ArrayBuffer[String]()
    
    for ch <- line do
      ch match
        case '"' => inQuotes = !inQuotes
        case ',' if !inQuotes =>
          fields += current.toString().trim
          current = new StringBuilder()
        case _ => current.append(ch)
    
    fields += current.toString().trim
    fields.toSeq

object Statistics:
  def mean(values: Seq[Double]): Double =
    if values.isEmpty then 0.0
    else values.sum / values.length
  
  def median(values: Seq[Double]): Double =
    if values.isEmpty then 0.0
    else
      val sorted = values.sorted
      val n = sorted.length
      if n % 2 == 0 then (sorted(n/2 - 1) + sorted(n/2)) / 2.0
      else sorted(n/2)
  
  def stddev(values: Seq[Double]): Double =
    if values.length < 2 then 0.0
    else
      val m = mean(values)
      val variance = values.map(x => (x - m) * (x - m)).sum / (values.length - 1)
      Math.sqrt(variance)
  
  def min(values: Seq[Double]): Double = if values.isEmpty then 0.0 else values.min
  def max(values: Seq[Double]): Double = if values.isEmpty then 0.0 else values.max

sealed trait Command
case class Describe(file: String, columns: Seq[String]) extends Command
case class Filter(file: String, column: String, value: String, output: String) extends Command
case class Aggregate(file: String, groupBy: String, aggCol: String, func: String) extends Command
case class ShowHelp() extends Command

object CliParser:
  def parse(args: Array[String]): Either[String, Command] =
    args.toList match
      case "describe" :: file :: rest =>
        Right(Describe(file, rest))
      case "filter" :: file :: "--column" :: col :: "--value" :: value :: "--output" :: out :: Nil =>
        Right(Filter(file, col, value, out))
      case "aggregate" :: file :: "--group-by" :: group :: "--column" :: col :: "--func" :: func :: Nil =>
        Right(Aggregate(file, group, col, func))
      case "help" :: _ | Nil =>
        Right(ShowHelp())
      case cmd :: _ =>
        Left(s"Unknown command: $cmd")

object CsvProcessor:
  def describe(file: String, columns: Seq[String]): Unit =
    val content = os.read(os.Path(file))
    val (headers, rows) = CsvParser.parse(content)
    
    val targetCols = if columns.isEmpty then headers else columns
    
    println(s"File: $file")
    println(s"Total rows: ${rows.length}")
    println(s"Columns: ${headers.mkString(", ")}")
    println()
    
    for col <- targetCols do
      val values = rows.flatMap(_.values.get(col))
      val numValues = values.flatMap(v => v.toDoubleOption)
      
      println(s"Column: $col")
      println(s"  Count: ${values.length}")
      println(s"  Non-null: ${values.count(_.nonEmpty)}")
      
      if numValues.nonEmpty then
        println(s"  Type: numeric")
        println(f"  Mean: ${Statistics.mean(numValues)}%.4f")
        println(f"  Median: ${Statistics.median(numValues)}%.4f")
        println(f"  Std Dev: ${Statistics.stddev(numValues)}%.4f")
        println(f"  Min: ${Statistics.min(numValues)}%.4f")
        println(f"  Max: ${Statistics.max(numValues)}%.4f")
      else
        println(s"  Type: text")
        val unique = values.distinct.length
        println(s"  Unique values: $unique")
        if unique <= 10 then
          println(s"  Values: ${values.distinct.take(10).mkString(", ")}")
      println()
  
  def filter(file: String, column: String, value: String, output: String): Unit =
    val content = os.read(os.Path(file))
    val (headers, rows) = CsvParser.parse(content)
    
    val filtered = rows.filter(_.values.get(column).contains(value))
    
    println(s"Filtered: ${filtered.length} / ${rows.length} rows")
    
    val csvContent = new StringBuilder()
    csvContent.append(headers.mkString(","))
    csvContent.append("\n")
    
    for row <- filtered do
      csvContent.append(headers.map(h => row.values.getOrElse(h, "")).mkString(","))
      csvContent.append("\n")
    
    os.write(os.Path(output), csvContent.toString())
    println(s"Written to: $output")
  
  def aggregate(file: String, groupBy: String, aggCol: String, func: String): Unit =
    val content = os.read(os.Path(file))
    val (_, rows) = CsvParser.parse(content)
    
    val grouped = rows.groupBy(_.values.getOrElse(groupBy, ""))
    
    println(f"$groupBy%-20s | $aggCol%-15s ($func)")
    println("-" * 45)
    
    for (group, groupRows) <- grouped.toSeq.sortBy(_._1) do
      val values = groupRows.flatMap(_.values.get(aggCol).flatMap(_.toDoubleOption))
      val result = func.toLowerCase match
        case "sum"   => values.sum
        case "mean"  => Statistics.mean(values)
        case "count" => values.length.toDouble
        case "min"   => Statistics.min(values)
        case "max"   => Statistics.max(values)
        case _       => Double.NaN
      
      println(f"$group%-20s | $result%-15.2f")

def showHelp(): Unit =
  println("""CSV Processor - Scala Native CLI Tool
    |
    |Usage:
    |  csv-processor describe <file> [columns...]
    |    Show statistics for CSV file
    |    
    |  csv-processor filter <file> --column <col> --value <val> --output <out>
    |    Filter rows where column equals value
    |    
    |  csv-processor aggregate <file> --group-by <col> --column <col> --func <func>
    |    Aggregate column grouped by another column
    |    Functions: sum, mean, count, min, max
    |
    |Examples:
    |  csv-processor describe data.csv
    |  csv-processor describe data.csv age salary
    |  csv-processor filter data.csv --column department --value Engineering --output eng.csv
    |  csv-processor aggregate data.csv --group-by department --column salary --func mean
    """.stripMargin)

@main def main(args: String*): Unit =
  CliParser.parse(args.toArray) match
    case Right(Describe(file, cols)) =>
      CsvProcessor.describe(file, cols)
    case Right(Filter(file, col, value, out)) =>
      CsvProcessor.filter(file, col, value, out)
    case Right(Aggregate(file, group, col, func)) =>
      CsvProcessor.aggregate(file, group, col, func)
    case Right(ShowHelp()) =>
      showHelp()
    case Left(err) =>
      println(s"Error: $err")
      showHelp()
      sys.exit(1)
```

### Test Data Generator

```scala
// src/test/scala/TestData.scala
import java.io.PrintWriter

object TestDataGenerator:
  def generateSalesData(rows: Int, file: String): Unit =
    val departments = Seq("Engineering", "Marketing", "Sales", "HR", "Finance")
    val random = new scala.util.Random(42)
    
    val pw = new PrintWriter(file)
    pw.println("id,name,department,salary,years_experience,performance_score")
    
    for i <- 1 to rows do
      val dept = departments(random.nextInt(departments.length))
      val salary = 50000 + random.nextInt(100000)
      val years = random.nextInt(20) + 1
      val score = (random.nextDouble() * 4 + 1).formatted("%.1f")
      pw.println(s"$i,Employee$i,$dept,$salary,$years,$score")
    
    pw.close()
    println(s"Generated $rows rows to $file")
```

---

## การ Optimize และ Advanced Features

### Link Time Optimization (LTO)

```scala
// build.sbt
nativeConfig ~= {
  _.withLTO(LTO.thin)    // เร็ว, ดีสำหรับ development
  // หรือ
  // _.withLTO(LTO.full)  // ช้ากว่า แต่ผลลัพธ์เล็กกว่า
}
```

### Optimization Modes

```scala
// build.sbt
nativeConfig ~= { c =>
  c.withMode(Mode.debug)        // สำหรับ development: fast compile, debug symbols
  // c.withMode(Mode.releaseFast)  // production: optimize แต่เร็ว
  // c.withMode(Mode.releaseFull)  // production: optimize เต็มที่
}
```

### Cross-compilation

```scala
// cross-compile สำหรับ multiple architectures
import scala.scalanative.build.*

lazy val root = project
  .enablePlugins(ScalaNativePlugin)
  .settings(
    // สำหรับ Linux x86_64
    nativeConfig ~= {
      _.withTargetTriple("x86_64-pc-linux-gnu")
    }
    
    // สำหรับ ARM64 (Apple Silicon / Raspberry Pi)
    // _.withTargetTriple("aarch64-apple-darwin")
    // _.withTargetTriple("aarch64-unknown-linux-gnu")
  )
```

### Embedding Resources

```scala
// อ่าน resource ที่ embed ไว้ใน binary
object Resources:
  def loadResource(name: String): String =
    // ใช้ os-lib สำหรับ read resources
    val stream = getClass.getResourceAsStream(s"/$name")
    if stream == null then
      throw new RuntimeException(s"Resource not found: $name")
    
    val bytes = stream.readAllBytes()
    new String(bytes, "UTF-8")

// ใน build.sbt เพิ่ม resources
Compile / unmanagedResources += {
  baseDirectory.value / "resources" / "config.json"
}
```

### Reflection และ Metadata

```scala
// Scala Native ไม่รองรับ runtime reflection เต็มรูปแบบ
// แต่ใช้ compile-time reflection ได้

import scala.compiletime.*

// Type-safe serialization ไม่ต้องใช้ reflection
trait Serializable[T]:
  def serialize(value: T): String
  def deserialize(s: String): Either[String, T]

case class Config(host: String, port: Int, debug: Boolean)

given Serializable[Config] with
  def serialize(c: Config): String =
    s"""{"host":"${c.host}","port":${c.port},"debug":${c.debug}}"""
  
  def deserialize(s: String): Either[String, Config] =
    // simple parser
    try
      val host = s.split(""""host":"""")(1).split("""""""")(0)
      val port = s.split(""""port":""")(1).split(",")(0).toInt
      val debug = s.contains(""""debug":true""")
      Right(Config(host, port, debug))
    catch
      case e: Exception => Left(s"Parse error: ${e.getMessage}")
```

---

## สรุป

Scala Native เป็นเครื่องมือทรงพลังที่ช่วยให้:

| Feature | ประโยชน์ |
|---------|---------|
| Native compilation | Instant startup, no JVM overhead |
| C interop | เข้าถึง native libraries โดยตรง |
| Zone memory | Predictable memory management |
| POSIX bindings | System programming ครบครัน |
| Small binary | Deploy ง่าย, container เล็ก |

### เมื่อไหร่ควรใช้ Scala Native

```
✓ CLI tools ที่รัน frequently
✓ System utilities และ scripts
✓ Embedded systems
✓ Performance-critical applications
✓ Applications ที่ต้องการ C interop
✗ Enterprise applications ที่มี JVM ecosystem
✗ Applications ที่ต้องการ Java reflection
✗ Applications ที่ใช้ Java-only libraries
```

### Next Steps

1. ทดลองสร้าง CLI tool ง่ายๆ ด้วย Scala Native
2. เรียนรู้การ bind C libraries
3. ศึกษา Zone memory management
4. Benchmark เปรียบเทียบกับ JVM

---

*[← ส่วนที่ 79: Scala.js](part-79-scala-js.md) | [ส่วนที่ 81: Functional Design Patterns →](part-81-fp-design.md)*
