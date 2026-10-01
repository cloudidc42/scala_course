# Part 20: Futures และ Concurrency

## สารบัญ
1. [Future พื้นฐาน](#future-พื้นฐาน)
2. [Future Composition](#future-composition)
3. [Error Handling](#error-handling)
4. [ExecutionContext](#executioncontext)
5. [Promise](#promise)
6. [Practical Patterns](#practical-patterns)

---

## Future พื้นฐาน

### Creating Futures

```scala
import scala.concurrent.{Future, ExecutionContext, Await}
import scala.concurrent.duration.*
import scala.util.{Try, Success, Failure}

// ExecutionContext จำเป็นสำหรับ Future
import ExecutionContext.Implicits.global

// สร้าง Future
val f1: Future[Int] = Future(42)
val f2: Future[String] = Future("hello")
val f3: Future[Int] = Future {
  Thread.sleep(100)  // simulate work
  1 + 1
}

// Future ที่สำเร็จทันที
val immediate = Future.successful(100)

// Future ที่ fail ทันที
val failed = Future.failed(new RuntimeException("oops"))

// รอผลลัพธ์ (ไม่แนะนำในโค้ด production)
val result = Await.result(f1, 5.seconds)
println(result)  // 42
```

### Callbacks

```scala
val future = Future {
  Thread.sleep(50)
  42
}

// onComplete: ทำงานทั้ง Success และ Failure
future.onComplete {
  case Success(value) => println(s"Got: $value")
  case Failure(ex)    => println(s"Failed: ${ex.getMessage}")
}

// foreach: ทำงานเฉพาะ Success
future.foreach(v => println(s"Value: $v"))

// Scala 3 style
future.andThen {
  case Success(v) => println(s"Completed with: $v")
  case Failure(e) => println(s"Failed with: ${e.getMessage}")
}

Thread.sleep(200)  // ให้ time สำหรับ callbacks
```

---

## Future Composition

### map, flatMap, filter

```scala
// map: transform successful result
val doubled = Future(21).map(_ * 2)
println(Await.result(doubled, 5.seconds))  // 42

// flatMap: chain futures
def fetchData(id: Int): Future[String] = Future(s"data-$id")
def processData(data: String): Future[Int] = Future(data.length)

val pipeline = fetchData(42).flatMap(processData)
println(Await.result(pipeline, 5.seconds))  // 9

// for comprehension (sequential)
def getUserName(id: Int): Future[String] = Future(s"User$id")
def getUserEmail(name: String): Future[String] = Future(s"${name.toLowerCase}@email.com")
def sendWelcome(email: String): Future[Boolean] = Future {
  println(s"Sending welcome to $email")
  true
}

val result = for
  name  <- getUserName(1)
  email <- getUserEmail(name)
  sent  <- sendWelcome(email)
yield sent

println(Await.result(result, 5.seconds))  // true
```

### Parallel Futures

```scala
// สำคัญ: สร้าง futures ก่อน แล้วค่อย compose
// ถ้าสร้างใน for comprehension จะรัน sequential!

// Sequential (ช้า)
def slowOp(n: Int): Future[Int] = Future {
  Thread.sleep(100)
  n * 2
}

val sequential = for
  a <- slowOp(1)  // รอ 100ms
  b <- slowOp(2)  // รออีก 100ms
yield a + b        // รวม: ~200ms

// Parallel (เร็ว)
val fa = slowOp(1)  // เริ่มทันที
val fb = slowOp(2)  // เริ่มทันที (parallel)
val parallel = for
  a <- fa          // รอ fa
  b <- fb          // รอ fb (น่าจะเสร็จแล้ว)
yield a + b        // รวม: ~100ms

// Future.sequence: รวม List[Future[A]] เป็น Future[List[A]]
val futures = List(slowOp(1), slowOp(2), slowOp(3))
val combined = Future.sequence(futures)
println(Await.result(combined, 5.seconds))  // List(2, 4, 6)

// Future.traverse: map กับ Future
val numbers = List(1, 2, 3, 4, 5)
val processed = Future.traverse(numbers)(slowOp)
println(Await.result(processed, 5.seconds))  // List(2, 4, 6, 8, 10)
```

---

## Error Handling

### recover และ recoverWith

```scala
val riskyFuture = Future[Int] {
  if math.random() > 0.5 then 42
  else throw new RuntimeException("Random failure")
}

// recover: handle failure กับ value
val safe = riskyFuture.recover {
  case _: RuntimeException => -1
}

// recoverWith: handle failure กับ Future
val safeFuture = riskyFuture.recoverWith {
  case _: RuntimeException => Future.successful(0)
}

// transform: map both success and failure
val transformed = riskyFuture.transform(
  value => value * 2,
  ex => new RuntimeException(s"Wrapped: ${ex.getMessage}")
)

// fallbackTo: ลอง future แรก ถ้า fail ลอง future ที่สอง
val primary = Future.failed(new RuntimeException("primary failed"))
val fallback = Future.successful(99)
val withFallback = primary.fallbackTo(fallback)
println(Await.result(withFallback, 5.seconds))  // 99
```

### Future กับ Either

```scala
// Pattern: Future[Either[Error, A]]
type AsyncResult[A] = Future[Either[String, A]]

def divideAsync(a: Int, b: Int): AsyncResult[Double] = Future {
  if b == 0 then Left("Division by zero")
  else Right(a.toDouble / b)
}

def sqrtAsync(n: Double): AsyncResult[Double] = Future {
  if n < 0 then Left("Negative number")
  else Right(math.sqrt(n))
}

// Compose ด้วย flatMap
def computation(a: Int, b: Int): AsyncResult[Double] =
  for
    divResult <- divideAsync(a, b)
    sqrtResult <- divResult match
      case Right(n) => sqrtAsync(n)
      case Left(e)  => Future.successful(Left(e))
  yield sqrtResult

println(Await.result(computation(100, 4), 5.seconds))  // Right(5.0)
println(Await.result(computation(100, 0), 5.seconds))  // Left(Division by zero)
```

---

## ExecutionContext

### Custom ExecutionContext

```scala
import java.util.concurrent.Executors
import scala.concurrent.ExecutionContext

// CPU-bound work: ใช้ fixed thread pool
val cpuBound = ExecutionContext.fromExecutorService(
  Executors.newFixedThreadPool(Runtime.getRuntime.availableProcessors())
)

// IO-bound work: ใช้ cached thread pool
val ioBound = ExecutionContext.fromExecutorService(
  Executors.newCachedThreadPool()
)

// ใช้ specific ExecutionContext
val cpuTask = Future {
  // CPU intensive work
  (1 to 1000000).map(n => n.toLong * n).sum
}(cpuBound)

val ioTask = Future {
  // IO work (ใน real app จะเป็น actual IO)
  Thread.sleep(100)
  "data from database"
}(ioBound)

// Global ExecutionContext (default)
given ExecutionContext = ExecutionContext.global

// อย่าลืม shutdown
// cpuBound.shutdown()
// ioBound.shutdown()
```

---

## Promise

### Creating Promises

```scala
import scala.concurrent.Promise

// Promise: manually complete a Future
val promise = Promise[Int]()
val future = promise.future

// Complete from another thread
val thread = new Thread(() => {
  Thread.sleep(100)
  promise.success(42)
})
thread.start()

println(Await.result(future, 5.seconds))  // 42

// Failure
val failPromise = Promise[Int]()
failPromise.failure(new RuntimeException("failed"))
val result = Await.ready(failPromise.future, 5.seconds)
println(result.value)  // Some(Failure(...))

// trySuccess/tryFailure: thread-safe, ไม่ throw ถ้า already completed
val safePromise = Promise[Int]()
safePromise.trySuccess(1)   // true
safePromise.trySuccess(2)   // false (already completed)
```

### Timeout Pattern

```scala
def withTimeout[A](future: Future[A], timeout: FiniteDuration)
    (using ec: ExecutionContext): Future[A] =
  val timeoutFuture = Future {
    Thread.sleep(timeout.toMillis)
    throw new RuntimeException(s"Timeout after $timeout")
  }
  Future.firstCompletedOf(List(future, timeoutFuture))

// Callback-based with Promise
def callbackToFuture[A](register: (Try[A] => Unit) => Unit): Future[A] =
  val promise = Promise[A]()
  register(promise.complete)
  promise.future
```

---

## Practical Patterns

### Async Retry

```scala
def retry[A](maxRetries: Int, delay: FiniteDuration)(f: => Future[A])
    (using ec: ExecutionContext): Future[A] =
  f.recoverWith {
    case ex if maxRetries > 0 =>
      println(s"Retrying... ($maxRetries left) due to: ${ex.getMessage}")
      akka.pattern.after(delay)(retry(maxRetries - 1, delay)(f))
      // Without Akka:
      Future {
        Thread.sleep(delay.toMillis)
      }.flatMap(_ => retry(maxRetries - 1, delay)(f))
  }

// Simpler retry without delay
def simpleRetry[A](times: Int)(f: => Future[A])
    (using ec: ExecutionContext): Future[A] =
  f.recoverWith {
    case _ if times > 1 => simpleRetry(times - 1)(f)
  }

// Usage
var attempts = 0
val unreliable = Future {
  attempts += 1
  if attempts < 3 then throw new RuntimeException(s"Attempt $attempts failed")
  else s"Success on attempt $attempts"
}

// Note: retry needs a by-name parameter to re-execute
def flakyOp(): Future[String] = Future {
  attempts += 1
  if attempts < 3 then throw new RuntimeException(s"Attempt $attempts failed")
  else s"Success on attempt $attempts"
}

attempts = 0
val result = simpleRetry(5)(flakyOp())
println(Await.result(result, 10.seconds))
// Success on attempt 3
```

### Concurrent Rate Limiting

```scala
import java.util.concurrent.Semaphore

def rateLimited[A](maxConcurrent: Int)(futures: List[() => Future[A]])
    (using ec: ExecutionContext): Future[List[A]] =
  val semaphore = new Semaphore(maxConcurrent)

  val limitedFutures = futures.map { f =>
    Future {
      semaphore.acquire()
      try Await.result(f(), 30.seconds)  // simplified
      finally semaphore.release()
    }
  }

  Future.sequence(limitedFutures)

// Better: use akka-stream or fs2 for real rate limiting

// Batching pattern
def processBatch[A, B](items: List[A], batchSize: Int)(process: List[A] => Future[List[B]])
    (using ec: ExecutionContext): Future[List[B]] =
  val batches = items.grouped(batchSize).toList
  Future.sequence(batches.map(process)).map(_.flatten)
```

### Circuit Breaker Pattern

```scala
import java.util.concurrent.atomic.AtomicInteger

class CircuitBreaker(failureThreshold: Int, resetTimeout: Long):
  enum State { case Closed, Open, HalfOpen }

  @volatile private var state = State.Closed
  private val failureCount = AtomicInteger(0)
  @volatile private var lastFailureTime = 0L

  def call[A](f: => Future[A])(using ec: ExecutionContext): Future[A] =
    state match
      case State.Open =>
        if System.currentTimeMillis() - lastFailureTime > resetTimeout
        then
          state = State.HalfOpen
          attempt(f)
        else Future.failed(new RuntimeException("Circuit breaker is OPEN"))
      case _ => attempt(f)

  private def attempt[A](f: => Future[A])(using ec: ExecutionContext): Future[A] =
    f.map { result =>
      onSuccess()
      result
    }.recoverWith { case ex =>
      onFailure()
      Future.failed(ex)
    }

  private def onSuccess(): Unit =
    failureCount.set(0)
    state = State.Closed

  private def onFailure(): Unit =
    val count = failureCount.incrementAndGet()
    lastFailureTime = System.currentTimeMillis()
    if count >= failureThreshold then state = State.Open

// Usage
val breaker = CircuitBreaker(failureThreshold = 3, resetTimeout = 5000)

def callExternalService(): Future[String] = Future {
  if math.random() > 0.7 then "success"
  else throw new RuntimeException("Service unavailable")
}

(1 to 10).foreach { i =>
  val result = breaker.call(callExternalService())
  result.onComplete { t =>
    println(s"Call $i: ${t.map(_ => "ok").getOrElse("fail")}")
  }
  Thread.sleep(50)
}

Thread.sleep(1000)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Future: creating, callbacks, composition
- ✅ Parallel vs Sequential futures
- ✅ Error handling: recover, recoverWith, fallbackTo
- ✅ ExecutionContext: global, custom
- ✅ Promise: manual completion
- ✅ Patterns: retry, rate limiting, circuit breaker

---

*[← Part 19: Type Classes](part-19-type-classes.md) | [Part 21: Akka Actors →](part-21-akka-actors.md)*
