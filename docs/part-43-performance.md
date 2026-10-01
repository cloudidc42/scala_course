# Part 43: Performance Optimization

## สารบัญ
1. [JVM Performance Basics](#jvm-performance)
2. [Memory Management](#memory-management)
3. [Collection Performance](#collection-performance)
4. [Profiling](#profiling)
5. [Concurrency Performance](#concurrency-performance)
6. [Benchmarking with JMH](#benchmarking-jmh)

---

## JVM Performance Basics

### JVM Internals

```
JVM Performance layers:
1. JIT Compilation: hot code paths compiled to native
2. Garbage Collection: memory management overhead
3. Object allocation: heap pressure
4. Cache effects: CPU cache locality

Key JVM flags:
-Xms2g -Xmx4g                    # heap size
-XX:+UseG1GC                     # G1 garbage collector
-XX:MaxGCPauseMillis=200         # target max pause
-XX:+UseStringDeduplication      # de-dup strings
-XX:+PrintGCDetails              # log GC events
-XX:+PrintGCDateStamps           # timestamps in GC log
-Xss512k                         # stack size per thread
-XX:+OptimizeStringConcat        # concat optimization
-XX:+DoEscapeAnalysis            # stack allocation
```

### Scala-specific Optimizations

```scala
// 1. Avoid boxing: use @specialized or value classes
opaque type FastInt = Int          // no boxing
// vs
case class SlowInt(value: Int)     // allocates an object

// 2. Avoid closure capture of mutable state
var counter = 0
val badList = List(1, 2, 3).map { x => counter += 1; x }  // captures mutable var

// 3. Use Iterator for large collections (lazy, no intermediate collections)
val result = (1 to 1_000_000).iterator
  .filter(_ % 2 == 0)
  .map(_ * 3)
  .take(10)
  .toList
// Lazy: does NOT create intermediate List

// 4. Prefer pattern matching over isInstanceOf
// Fast:
def fast(x: Any): Int = x match
  case n: Int => n
  case _ => 0

// Slow:
def slow(x: Any): Int =
  if x.isInstanceOf[Int] then x.asInstanceOf[Int] else 0

// 5. Avoid repeated string concatenation in loops
// Bad: O(n^2) allocations
def badConcat(items: List[String]): String =
  items.foldLeft("") { (acc, s) => acc + s + ", " }

// Good: StringBuilder
def goodConcat(items: List[String]): String =
  val sb = new StringBuilder
  items.foreach { s => sb.append(s); sb.append(", ") }
  if sb.nonEmpty then sb.dropRight(2).toString else ""

// Better: mkString
def bestConcat(items: List[String]): String = items.mkString(", ")
```

---

## Memory Management

### Reducing Object Allocation

```scala
// 1. Use primitives when possible
// Bad: box to Integer
val scores: List[Int] = List(90, 85, 92, 88)  // boxing in generic collections

// Good: use specialized collections
import scala.collection.mutable
val fastScores: mutable.ArrayBuffer[Int] = mutable.ArrayBuffer(90, 85, 92, 88)

// 2. Value classes: zero-cost abstraction (compile away)
class Meters(val value: Double) extends AnyVal:
  def toKilometers: Double = value / 1000.0
  def +(other: Meters): Meters = Meters(value + other.value)

val d1 = Meters(1000.0)
val d2 = Meters(500.0)
val total = d1 + d2
// At runtime, total is just a Double, not a Meters object!

// 3. Reuse objects: object pools
class ObjectPool[A](create: () => A, maxSize: Int = 100):
  private val pool = new java.util.concurrent.LinkedBlockingQueue[A](maxSize)

  def acquire(): A =
    Option(pool.poll()).getOrElse(create())

  def release(obj: A): Unit =
    pool.offer(obj)  // silently drops if full

// 4. Avoid String.format, prefer interpolation
val name = "Alice"
val score = 95
// Bad:
val msg1 = String.format("Name: %s, Score: %d", name, score)
// Good:
val msg2 = s"Name: $name, Score: $score"
```

### GC-Friendly Code

```scala
// Avoid long-lived temporary objects
// Bad: creates many intermediate lists
def processBad(data: List[Int]): List[Int] =
  data
    .filter(_ > 0)        // new list
    .map(_ * 2)           // new list
    .filter(_ < 100)      // new list
    .sorted               // new list

// Good: single pass with view (lazy evaluation)
def processGood(data: List[Int]): List[Int] =
  data.view
    .filter(_ > 0)
    .map(_ * 2)
    .filter(_ < 100)
    .toList       // one allocation at the end
  // then sorted -- this still allocates

// For very hot paths: use while loops
def sumFast(data: Array[Int]): Long =
  var sum = 0L
  var i = 0
  while i < data.length do
    sum += data(i)
    i += 1
  sum

// vs functional (slightly slower due to closure overhead)
def sumFunctional(data: Array[Int]): Long =
  data.foldLeft(0L)(_ + _)
```

---

## Collection Performance

### Choosing the Right Collection

```scala
// Performance characteristics table:
// Operation       | Array  | List  | Vector | HashMap | TreeMap
// ----------------------------------------------------------------
// Random access   |  O(1)  | O(n)  |  O(1)  |   O(1)  |  O(log n)
// Prepend         |  O(n)  | O(1)  | O(log) |   -     |   -
// Append          |  O(1)  | O(n)  | O(log) |   -     |   -
// Lookup          |  O(n)  | O(n)  |  O(n)  |   O(1)  |  O(log n)
// Insert (mutable)|  O(n)  | O(1)  |   -    |   O(1)  |  O(log n)

// 1. Array vs List vs Vector
val arr    = Array(1, 2, 3, 4, 5)      // best for: random access, iteration
val lst    = List(1, 2, 3, 4, 5)       // best for: prepend, pattern matching
val vec    = Vector(1, 2, 3, 4, 5)     // best for: balanced (append+prepend+access)

// 2. HashMap vs SortedMap
val hm = Map("a" -> 1, "b" -> 2)         // O(1) lookup
val sm = scala.collection.SortedMap("a" -> 1, "b" -> 2)  // ordered, O(log n)

// 3. Set membership: HashSet is faster than List contains
val items = Set(1, 2, 3, 4, 5)
val hasThree = items.contains(3)  // O(1)
// vs
val list = List(1, 2, 3, 4, 5)
val hasThreeSlow = list.contains(3)  // O(n)

// 4. Mutable collections for hot paths
import scala.collection.mutable

def buildMap(pairs: List[(String, Int)]): Map[String, Int] =
  val m = mutable.HashMap.empty[String, Int]
  pairs.foreach { case (k, v) => m(k) = v }
  m.toMap  // convert to immutable at the end

// 5. LazyList (formerly Stream): infinite sequences
val naturals: LazyList[Int] = LazyList.from(1)
val first100Primes = naturals.filter(isPrime).take(100).toList

def isPrime(n: Int): Boolean =
  n > 1 && (2 to math.sqrt(n).toInt).forall(n % _ != 0)
```

---

## Profiling

### JVM Profiling Tools

```scala
// Simple timing
def time[A](name: String)(action: => A): A =
  val start = System.nanoTime()
  val result = action
  val elapsed = System.nanoTime() - start
  println(f"$name: ${elapsed / 1_000_000.0}%.2f ms")
  result

val result = time("sort 1M ints") {
  scala.util.Random.shuffle((1 to 1_000_000).toList).sorted
}

// Micro-benchmark helper (for development only)
def benchmark[A](name: String, iterations: Int = 1000)(action: => A): Unit =
  // Warmup
  (1 to 100).foreach(_ => action)

  // Measure
  val times = (1 to iterations).map { _ =>
    val t = System.nanoTime()
    action
    System.nanoTime() - t
  }

  val avg = times.sum / iterations
  val p95 = times.sorted.apply((iterations * 0.95).toInt)
  println(f"$name: avg=${avg/1000}µs, p95=${p95/1000}µs")

// Memory tracking
def trackMemory[A](name: String)(action: => A): A =
  val runtime = Runtime.getRuntime
  System.gc()
  val before = runtime.totalMemory() - runtime.freeMemory()
  val result = action
  System.gc()
  val after = runtime.totalMemory() - runtime.freeMemory()
  println(s"$name: memory delta = ${(after - before) / 1024}KB")
  result
```

---

## Concurrency Performance

### Parallelism Patterns

```scala
import scala.concurrent.{Future, ExecutionContext}
import scala.concurrent.duration.*

// Parallel collections for CPU-bound work
val data = (1 to 1_000_000).toVector

// Sequential
val seqResult = data.map(n => n * n).sum

// Parallel (uses all cores)
val parResult = data.par.map(n => n * n).sum

// Avoid unnecessary thread switches
// Bad: too many futures for cheap operations
val badFutures = (1 to 1000).map { i =>
  Future { i * i }  // overhead > computation
}

// Good: batch work
val batchedFutures = (1 to 1000).grouped(100).map { batch =>
  Future { batch.map(i => i * i) }
}

// Thread pool sizing
// CPU-bound: numCores threads
// IO-bound: more threads (100s), use async I/O instead

val cpuBound = ExecutionContext.fromExecutorService(
  java.util.concurrent.Executors.newFixedThreadPool(
    Runtime.getRuntime.availableProcessors()
  )
)

val ioBound = ExecutionContext.fromExecutorService(
  java.util.concurrent.Executors.newCachedThreadPool()
)
```

---

## Benchmarking with JMH

### JMH Setup

```scala
// build.sbt
addSbtPlugin("pl.project13.scala" % "sbt-jmh" % "0.4.6")

// Enable in project
enablePlugins(JmhPlugin)

// Writing benchmarks
package benchmarks

import org.openjdk.jmh.annotations.*
import java.util.concurrent.TimeUnit

@State(Scope.Thread)
@BenchmarkMode(Array(Mode.AverageTime))
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
class CollectionBenchmark:

  val data: List[Int] = (1 to 10_000).toList
  val arr: Array[Int] = (1 to 10_000).toArray
  val vec: Vector[Int] = (1 to 10_000).toVector

  @Benchmark
  def listAccess(): Int = data(5000)

  @Benchmark
  def arrayAccess(): Int = arr(5000)

  @Benchmark
  def vectorAccess(): Int = vec(5000)

  @Benchmark
  def listMap(): List[Int] = data.map(_ * 2)

  @Benchmark
  def arrayMap(): Array[Int] = arr.map(_ * 2)

  @Benchmark
  def listFold(): Int = data.foldLeft(0)(_ + _)

  @Benchmark
  def arrayWhile(): Int =
    var sum = 0
    var i = 0
    while i < arr.length do
      sum += arr(i)
      i += 1
    sum

// Run: sbt "jmh:run -i 10 -wi 5 -f 1 .*CollectionBenchmark.*"
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ JVM performance basics: JIT, GC, flags
- ✅ Reducing allocations: value classes, primitives
- ✅ GC-friendly code: views, while loops
- ✅ Collection performance: choosing the right collection
- ✅ Profiling: timing, memory tracking
- ✅ JMH benchmarking

---

*[← Part 42: DDD](part-42-ddd.md) | [Part 44: Security →](part-44-security.md)*
