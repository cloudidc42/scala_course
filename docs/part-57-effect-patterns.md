# ส่วนที่ 57: Advanced Effect Patterns

## สารบัญ

1. [Resource Management ด้วย Resource[F, A]](#resource-management-ด้วย-resourcef-a)
2. [Bracket และ Guarantee](#bracket-และ-guarantee)
3. [Racing Effects ด้วย IO.race](#racing-effects-ด้วย-iorace)
4. [Timeout และ Cancellation](#timeout-และ-cancellation)
5. [Fiber-based Concurrency](#fiber-based-concurrency)
6. [Semaphore สำหรับ Concurrency Control](#semaphore-สำหรับ-concurrency-control)
7. [Mutex และ Exclusive Access](#mutex-และ-exclusive-access)
8. [CountDownLatch และ Synchronization](#countdown-latch-และ-synchronization)
9. [Deferred และ Promise Patterns](#deferred-และ-promise-patterns)
10. [Complete Concurrent Example](#complete-concurrent-example)
11. [สรุป](#สรุป)

---

## Resource Management ด้วย Resource[F, A]

`Resource[F, A]` คือ abstraction สำหรับจัดการ resources ที่ต้องการ acquire และ release อย่างปลอดภัย แม้ว่าจะเกิด exception

### Resource Fundamentals

```scala
import cats.effect.*
import cats.syntax.all.*
import scala.concurrent.duration.*

object ResourceBasics extends IOApp.Simple:

  // สร้าง Resource อย่างง่าย
  def dbConnection(id: Int): Resource[IO, String] =
    Resource.make(
      IO.println(s"Opening connection $id") >> IO.pure(s"conn-$id")
    )(conn =>
      IO.println(s"Closing connection: $conn")
    )

  // ใช้ Resource
  def basicUsage: IO[Unit] =
    dbConnection(1).use { conn =>
      IO.println(s"Using connection: $conn") >>
      IO.sleep(100.millis)
    }

  // Resource จะ release แม้ว่าจะเกิด error
  def safeCleanup: IO[Unit] =
    dbConnection(2).use { conn =>
      IO.println(s"Using $conn") >>
      IO.raiseError(new RuntimeException("Something went wrong!"))
    }.handleErrorWith(err =>
      IO.println(s"Caught error: ${err.getMessage}")
    )

  // Compose หลาย Resources
  def composedResources: IO[Unit] =
    (
      dbConnection(1),
      dbConnection(2),
      dbConnection(3)
    ).tupled.use { (c1, c2, c3) =>
      IO.println(s"Using all connections: $c1, $c2, $c3")
    }

  def run: IO[Unit] =
    IO.println("=== Basic Usage ===") >>
    basicUsage >>
    IO.println("\n=== Safe Cleanup (with error) ===") >>
    safeCleanup >>
    IO.println("\n=== Composed Resources ===") >>
    composedResources
```

### Resource ที่ซับซ้อน

```scala
import cats.effect.*
import cats.syntax.all.*
import scala.concurrent.duration.*

// Database connection pool
case class ConnectionPool(size: Int, host: String)

object ConnectionPool:
  def make(host: String, poolSize: Int): Resource[IO, ConnectionPool] =
    Resource.make(
      IO.println(s"Initializing connection pool: $poolSize connections to $host") >>
      IO.pure(ConnectionPool(poolSize, host))
    )(pool =>
      IO.println(s"Draining connection pool to ${pool.host}")
    )

// HTTP Client
case class HttpClient(baseUrl: String)

object HttpClient:
  def make(baseUrl: String): Resource[IO, HttpClient] =
    Resource.make(
      IO.println(s"Creating HTTP client for: $baseUrl") >>
      IO.pure(HttpClient(baseUrl))
    )(client =>
      IO.println(s"Closing HTTP client for: ${client.baseUrl}")
    )

// Application context ที่รวม resources ทั้งหมด
case class AppContext(
  dbPool: ConnectionPool,
  httpClient: HttpClient,
  cacheConn: String
)

object AppContext:
  def make: Resource[IO, AppContext] =
    for
      db     <- ConnectionPool.make("db.example.com", 10)
      http   <- HttpClient.make("https://api.example.com")
      cache  <- Resource.make(
        IO.println("Connecting to Redis...") >> IO.pure("redis://localhost:6379")
      )(c =>
        IO.println(s"Disconnecting from Redis: $c")
      )
    yield AppContext(db, http, cache)

object ComplexResourceDemo extends IOApp.Simple:

  def run: IO[Unit] =
    AppContext.make.use { ctx =>
      IO.println(s"""
        |Application started with:
        |  DB Pool: ${ctx.dbPool.size} connections to ${ctx.dbPool.host}
        |  HTTP Client: ${ctx.httpClient.baseUrl}
        |  Cache: ${ctx.cacheConn}
        |
        |Processing requests...
        |""".stripMargin) >>
      IO.sleep(200.millis) >>
      IO.println("Done processing, shutting down gracefully...")
    }
```

### Resource.fromAutoCloseable

```scala
import cats.effect.*
import java.io.*
import java.nio.file.*

object AutoCloseableResource extends IOApp.Simple:

  // ใช้กับ Java AutoCloseable resources
  def readFile(path: String): Resource[IO, BufferedReader] =
    Resource.fromAutoCloseable(
      IO(new BufferedReader(new FileReader(path)))
    )

  def writeFile(path: String): Resource[IO, BufferedWriter] =
    Resource.fromAutoCloseable(
      IO(new BufferedWriter(new FileWriter(path)))
    )

  // Copy file ด้วย Resource
  def copyFile(src: String, dst: String): IO[Int] =
    (readFile(src), writeFile(dst)).tupled.use { (reader, writer) =>
      def loop(count: Int): IO[Int] =
        IO(reader.readLine()).flatMap {
          case null => IO.pure(count)
          case line =>
            IO(writer.write(line + "\n")) >> loop(count + 1)
        }
      loop(0)
    }

  def run: IO[Unit] =
    // Create test file
    IO(Files.write(
      Paths.get("/tmp/test-input.txt"),
      "Line 1\nLine 2\nLine 3\n".getBytes
    )) >>
    copyFile("/tmp/test-input.txt", "/tmp/test-output.txt").flatMap { lines =>
      IO.println(s"Copied $lines lines")
    }
```

---

## Bracket และ Guarantee

### Bracket Pattern

```scala
import cats.effect.*
import scala.concurrent.duration.*

object BracketDemo extends IOApp.Simple:

  // bracket = acquire + use + release
  def bracketExample: IO[Unit] =
    IO.bracket(
      // Acquire
      IO.println("Acquiring resource...") >> IO.pure("my-resource")
    )(
      // Use
      resource =>
        IO.println(s"Using $resource") >>
        IO.sleep(100.millis)
    )(
      // Release (always called)
      resource =>
        IO.println(s"Releasing $resource")
    )

  // bracketCase - รู้ว่า use ออกด้วย exit case อะไร
  def bracketCaseExample: IO[Unit] =
    IO.bracketCase(
      IO.println("Acquiring...") >> IO.pure("resource")
    )(
      resource =>
        IO.println(s"Using $resource") >>
        IO.raiseError(new RuntimeException("Use failed!"))
    )(
      (resource, exitCase) =>
        exitCase match
          case Outcome.Succeeded(fa) =>
            IO.println(s"Released $resource after success")
          case Outcome.Errored(e) =>
            IO.println(s"Released $resource after error: ${e.getMessage}")
          case Outcome.Canceled() =>
            IO.println(s"Released $resource after cancellation")
    )

  def run: IO[Unit] =
    IO.println("=== Bracket ===") >>
    bracketExample >>
    IO.println("\n=== BracketCase (with error) ===") >>
    bracketCaseExample.handleErrorWith(e => IO.println(s"Handled: $e"))
```

### Guarantee Patterns

```scala
import cats.effect.*
import scala.concurrent.duration.*

object GuaranteeDemo extends IOApp.Simple:

  // guarantee - รัน finalizer เสมอ
  def guaranteeExample: IO[Int] =
    IO(42)
      .guarantee(
        IO.println("Cleanup always runs!")
      )

  // guaranteeCase - รัน finalizer และรู้ exit case
  def guaranteeCaseExample: IO[Unit] =
    IO.sleep(50.millis)
      .as("result")
      .guaranteeCase {
        case Outcome.Succeeded(fa) =>
          fa.flatMap(r => IO.println(s"Success with: $r"))
        case Outcome.Errored(e) =>
          IO.println(s"Error: ${e.getMessage}")
        case Outcome.Canceled() =>
          IO.println("Cancelled!")
      }

  // onError - รัน action เฉพาะเมื่อเกิด error
  def onErrorExample: IO[Int] =
    IO.raiseError[Int](new RuntimeException("oops"))
      .onError(e => IO.println(s"Error occurred: ${e.getMessage}"))
      .handleError(_ => -1)

  // onCancel - รัน action เมื่อถูก cancel
  def onCancelExample: IO[Unit] =
    IO.sleep(5.seconds)
      .onCancel(IO.println("I was cancelled!"))

  def run: IO[Unit] =
    IO.println("=== Guarantee ===") >>
    guaranteeExample.flatMap(n => IO.println(s"Result: $n")) >>
    IO.println("\n=== GuaranteeCase ===") >>
    guaranteeCaseExample >>
    IO.println("\n=== OnError ===") >>
    onErrorExample.flatMap(n => IO.println(s"Result: $n")) >>
    IO.println("\n=== OnCancel (will timeout) ===") >>
    onCancelExample.timeoutTo(100.millis, IO.println("Timed out"))
```

---

## Racing Effects ด้วย IO.race

### Race Patterns

```scala
import cats.effect.*
import scala.concurrent.duration.*

object RaceDemo extends IOApp.Simple:

  // race - ใช้ผลลัพธ์จาก effect แรกที่เสร็จ
  def basicRace: IO[Unit] =
    IO.race(
      IO.sleep(200.millis) >> IO.pure("slow"),
      IO.sleep(100.millis) >> IO.pure("fast")
    ).flatMap {
      case Left(s)  => IO.println(s"Left won: $s")
      case Right(s) => IO.println(s"Right won: $s")
    }

  // racePair - ได้ทั้งผลลัพธ์และ fiber ที่แพ้
  def racePairDemo: IO[Unit] =
    IO.racePair(
      IO.sleep(300.millis).as("slow-result"),
      IO.sleep(150.millis).as("fast-result")
    ).flatMap {
      case Left((outcome, loserFiber)) =>
        // left won
        loserFiber.cancel >>
        IO.println(s"Left outcome: $outcome")
      case Right((loserFiber, outcome)) =>
        // right won  
        loserFiber.cancel >>
        IO.println(s"Right won: $outcome")
    }

  // race สำหรับ fallback pattern
  def fallbackRace: IO[String] =
    val primary = IO.sleep(2.seconds) >> IO.pure("primary-response")
    val fallback = IO.sleep(500.millis) >> IO.pure("fallback-response")
    
    IO.race(primary, fallback).map {
      case Left(r)  => r
      case Right(r) => r
    }

  // Timeout ด้วย race
  def withTimeout[A](effect: IO[A], timeout: FiniteDuration): IO[Option[A]] =
    IO.race(effect, IO.sleep(timeout)).map {
      case Left(result) => Some(result)
      case Right(_)     => None
    }

  def run: IO[Unit] =
    IO.println("=== Basic Race ===") >>
    basicRace >>
    IO.println("\n=== Race Pair ===") >>
    racePairDemo >>
    IO.println("\n=== Fallback Race ===") >>
    fallbackRace.flatMap(r => IO.println(s"Got: $r")) >>
    IO.println("\n=== Timeout Race ===") >>
    withTimeout(IO.sleep(300.millis) >> IO.pure(42), 500.millis)
      .flatMap(r => IO.println(s"Result: $r")) >>
    withTimeout(IO.sleep(1.second) >> IO.pure(42), 200.millis)
      .flatMap(r => IO.println(s"Timed out: $r"))
```

### Real-world Race Patterns

```scala
import cats.effect.*
import scala.concurrent.duration.*

// Multi-source data fetching
object MultiSourceFetch extends IOApp.Simple:

  // จำลอง API calls จาก multiple sources
  def fetchFromPrimary(key: String): IO[String] =
    IO.sleep(200.millis) >> IO.pure(s"primary:$key")

  def fetchFromReplica(key: String): IO[String] =
    IO.sleep(100.millis) >> IO.pure(s"replica:$key")

  def fetchFromCache(key: String): IO[Option[String]] =
    IO.sleep(10.millis) >> IO.pure(Some(s"cache:$key"))

  // Fetch fastest available result
  def fetchFastest(key: String): IO[String] =
    IO.race(
      fetchFromPrimary(key),
      fetchFromReplica(key)
    ).map {
      case Left(r)  => r
      case Right(r) => r
    }

  // Try cache first, then race primary vs replica
  def fetchWithCacheFirst(key: String): IO[String] =
    fetchFromCache(key).flatMap {
      case Some(cached) => IO.pure(cached)
      case None         => fetchFastest(key)
    }

  // Hedged requests (duplicate to reduce tail latency)
  def hedgedFetch(key: String, hedgeAfter: FiniteDuration): IO[String] =
    IO.race(
      fetchFromPrimary(key),
      IO.sleep(hedgeAfter) >> fetchFromReplica(key)
    ).map {
      case Left(r)  => r
      case Right(r) => r
    }

  def run: IO[Unit] =
    for
      r1 <- fetchFastest("user:123")
      _ <- IO.println(s"Fastest: $r1")
      
      r2 <- fetchWithCacheFirst("product:456")
      _ <- IO.println(s"Cache-first: $r2")
      
      r3 <- hedgedFetch("order:789", 50.millis)
      _ <- IO.println(s"Hedged: $r3")
    yield ()
```

---

## Timeout และ Cancellation

### Timeout Patterns

```scala
import cats.effect.*
import scala.concurrent.duration.*

object TimeoutDemo extends IOApp.Simple:

  // timeout - raise TimeoutException
  def timeoutBasic: IO[Unit] =
    IO.sleep(2.seconds)
      .as("done")
      .timeout(500.millis)
      .flatMap(r => IO.println(s"Result: $r"))
      .handleErrorWith {
        case _: java.util.concurrent.TimeoutException =>
          IO.println("Operation timed out!")
        case e =>
          IO.println(s"Other error: $e")
      }

  // timeoutTo - fallback value on timeout
  def timeoutWithFallback: IO[String] =
    IO.sleep(2.seconds)
      .as("slow-result")
      .timeoutTo(500.millis, IO.pure("timeout-fallback"))

  // timeout ที่ nested
  def nestedTimeouts: IO[Unit] =
    val inner = IO.sleep(300.millis).as("inner-result")
      .timeout(500.millis)  // inner timeout: 500ms
    
    inner
      .timeout(1.second)  // outer timeout: 1s
      .flatMap(r => IO.println(s"Got: $r"))

  // Cancellation-aware operations
  def cancellationAware: IO[Unit] =
    val longOperation =
      IO.println("Starting long operation...") >>
      IO.sleep(10.seconds) >>
      IO.println("Long operation done")
    
    val cancellable =
      longOperation
        .onCancel(IO.println("Long operation was cancelled!"))
    
    // Cancel ด้วย timeout
    cancellable
      .timeoutTo(500.millis, IO.println("Fell back after timeout"))

  def run: IO[Unit] =
    IO.println("=== Timeout Basic ===") >>
    timeoutBasic >>
    IO.println("\n=== Timeout With Fallback ===") >>
    timeoutWithFallback.flatMap(r => IO.println(s"Result: $r")) >>
    IO.println("\n=== Nested Timeouts ===") >>
    nestedTimeouts >>
    IO.println("\n=== Cancellation Aware ===") >>
    cancellationAware
```

### Graceful Shutdown

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*
import scala.concurrent.duration.*

object GracefulShutdown extends IOApp.Simple:

  // Shutdown hook ด้วย SignallingRef
  def serverWithGracefulShutdown: IO[Unit] =
    for
      shutdownSignal <- cats.effect.std.Deferred[IO, Unit]
      
      // Server fiber
      serverFiber <- (
        IO.println("Server starting...") >>
        IO.sleep(100.millis) >>
        IO.println("Server ready, accepting connections") >>
        shutdownSignal.get >>  // รอจนกว่าจะได้รับ shutdown signal
        IO.println("Server shutting down gracefully...")
      ).start
      
      // Simulate shutdown after 500ms
      _ <- IO.sleep(500.millis)
      _ <- IO.println("Sending shutdown signal...")
      _ <- shutdownSignal.complete(())
      _ <- serverFiber.join
      _ <- IO.println("Server stopped")
    yield ()

  // Resource-safe server
  def managedServer: Resource[IO, Unit] =
    Resource.make(
      IO.println("Server starting...") >> IO.pure(())
    )(_ =>
      IO.println("Server stopping gracefully...") >>
      IO.sleep(100.millis) >>  // รอให้ requests ที่ค้างอยู่เสร็จก่อน
      IO.println("Server stopped")
    )

  def run: IO[Unit] =
    IO.println("=== Graceful Shutdown with Deferred ===") >>
    serverWithGracefulShutdown >>
    IO.println("\n=== Managed Server with Resource ===") >>
    managedServer.use(_ =>
      IO.println("Server running...") >>
      IO.sleep(200.millis)
    )
```

---

## Fiber-based Concurrency

### Fiber Fundamentals

```scala
import cats.effect.*
import scala.concurrent.duration.*

object FiberDemo extends IOApp.Simple:

  // start - เริ่ม fiber ใหม่
  def basicFiber: IO[Unit] =
    for
      fiber <- IO.sleep(200.millis).as("fiber-result").start
      _ <- IO.println("Main thread continues...")
      result <- fiber.joinWithNever  // รอผลลัพธ์
      _ <- IO.println(s"Fiber result: $result")
    yield ()

  // หลาย fibers พร้อมกัน
  def multipleFibers: IO[Unit] =
    for
      f1 <- (IO.sleep(300.millis) >> IO.pure(1)).start
      f2 <- (IO.sleep(200.millis) >> IO.pure(2)).start
      f3 <- (IO.sleep(100.millis) >> IO.pure(3)).start
      
      r3 <- f3.joinWithNever  // เสร็จก่อน
      r2 <- f2.joinWithNever
      r1 <- f1.joinWithNever
      
      _ <- IO.println(s"Results: $r1, $r2, $r3")
    yield ()

  // Fiber lifecycle management
  def fiberLifecycle: IO[Unit] =
    for
      fiber <- (
        IO.println("Fiber: starting work") >>
        IO.sleep(1.second) >>
        IO.println("Fiber: done") >>
        IO.pure("result")
      ).start
      
      _ <- IO.sleep(200.millis)
      _ <- IO.println("Main: cancelling fiber")
      _ <- fiber.cancel
      
      outcome <- fiber.join
      _ <- outcome match
        case Outcome.Succeeded(fa) => fa.flatMap(r => IO.println(s"Succeeded: $r"))
        case Outcome.Errored(e)    => IO.println(s"Errored: $e")
        case Outcome.Canceled()    => IO.println("Canceled!")
    yield ()

  // parMapN - ใช้บ่อยที่สุด
  def parallelMap: IO[Unit] =
    (
      IO.sleep(100.millis) >> IO.pure("a"),
      IO.sleep(200.millis) >> IO.pure("b"),
      IO.sleep(150.millis) >> IO.pure("c")
    ).parMapN { (a, b, c) =>
      IO.println(s"Results: $a, $b, $c")
    }.flatten

  def run: IO[Unit] =
    IO.println("=== Basic Fiber ===") >>
    basicFiber >>
    IO.println("\n=== Multiple Fibers ===") >>
    multipleFibers >>
    IO.println("\n=== Fiber Lifecycle ===") >>
    fiberLifecycle >>
    IO.println("\n=== Parallel Map ===") >>
    parallelMap
```

### Fiber Supervision

```scala
import cats.effect.*
import cats.effect.std.Queue
import scala.concurrent.duration.*

// Worker Pool ด้วย Fibers
object WorkerPool extends IOApp.Simple:

  case class Job(id: Int, work: IO[String])

  def makeWorkerPool(
    workerCount: Int,
    jobQueue: Queue[IO, Option[Job]]
  ): IO[List[Fiber[IO, Throwable, Unit]]] =
    List.range(0, workerCount).traverse { workerId =>
      val worker = for
        _ <- IO.println(s"Worker $workerId starting")
        _ <- fs2.Stream
          .fromQueueNoneTerminated(jobQueue)
          .evalMap { job =>
            IO.println(s"Worker $workerId processing job ${job.id}") >>
            job.work.flatMap(result =>
              IO.println(s"Worker $workerId completed job ${job.id}: $result")
            )
          }
          .compile
          .drain
        _ <- IO.println(s"Worker $workerId stopping")
      yield ()
      
      worker.start
    }

  def run: IO[Unit] =
    for
      queue <- Queue.bounded[IO, Option[Job]](20)
      
      // Enqueue jobs
      _ <- List.range(1, 11).traverse { i =>
        val job = Job(i, IO.sleep((i * 10 % 100).millis) >> IO.pure(s"result-$i"))
        queue.offer(Some(job))
      }
      
      // Signal completion to all workers
      _ <- List.range(0, 3).traverse(_ => queue.offer(None))
      
      // Start workers
      workers <- makeWorkerPool(3, queue)
      
      // Wait for all workers to finish
      _ <- workers.traverse(_.join)
      
      _ <- IO.println("All jobs completed!")
    yield ()
```

---

## Semaphore สำหรับ Concurrency Control

```scala
import cats.effect.*
import cats.effect.std.Semaphore
import scala.concurrent.duration.*

object SemaphoreDemo extends IOApp.Simple:

  // Basic semaphore - จำกัด concurrent access
  def basicSemaphore: IO[Unit] =
    for
      sem <- Semaphore[IO](3)  // อนุญาตสูงสุด 3 concurrent
      
      _ <- List.range(1, 11).parTraverse { i =>
        sem.permit.use { _ =>
          IO.println(s"Task $i: acquired permit") >>
          IO.sleep((100 + i * 20 % 300).millis) >>
          IO.println(s"Task $i: releasing permit")
        }
      }
    yield ()

  // Rate limiter ด้วย Semaphore
  class RateLimiter(sem: Semaphore[IO], refillInterval: FiniteDuration):
    def acquire: IO[Unit] = sem.acquire
    def release: IO[Unit] = sem.release
    
    def usePermit[A](action: IO[A]): IO[A] =
      sem.permit.use(_ => action)

  def makeRateLimiter(
    permits: Int,
    refillInterval: FiniteDuration
  ): Resource[IO, RateLimiter] =
    Resource.eval(Semaphore[IO](permits)).flatMap { sem =>
      // Refill permits periodically
      val refiller = fs2.Stream
        .fixedDelay[IO](refillInterval)
        .evalMap { _ =>
          sem.count.flatMap { count =>
            val toAdd = permits - count
            if toAdd > 0 then sem.releaseN(toAdd)
            else IO.unit
          }
        }
        .compile
        .drain
        .start
      
      Resource.make(
        refiller.map(_ => new RateLimiter(sem, refillInterval))
      )(_ => IO.unit)
    }

  // Semaphore สำหรับ database connection pool
  def connectionPoolDemo: IO[Unit] =
    Semaphore[IO](5).flatMap { pool =>  // max 5 connections
      List.range(1, 16).parTraverse { requestId =>
        pool.permit.use { _ =>
          IO.println(s"Request $requestId: got connection") >>
          IO.sleep((50 + requestId * 10 % 200).millis) >>
          IO.println(s"Request $requestId: released connection")
        }
      }.void
    }

  def run: IO[Unit] =
    IO.println("=== Basic Semaphore (max 3 concurrent) ===") >>
    basicSemaphore >>
    IO.println("\n=== DB Connection Pool (max 5) ===") >>
    connectionPoolDemo
```

---

## Mutex และ Exclusive Access

```scala
import cats.effect.*
import cats.effect.std.Mutex
import scala.concurrent.duration.*

object MutexDemo extends IOApp.Simple:

  // Mutex - exclusive access (semaphore ที่มี 1 permit)
  def basicMutex: IO[Unit] =
    for
      mutex <- Mutex[IO]
      counter <- Ref.of[IO, Int](0)
      
      // ไม่มี mutex - race condition!
      unsafeIncrement = counter.get.flatMap { n =>
        IO.sleep(1.millis) >> counter.set(n + 1)  // read-modify-write race!
      }
      
      // มี mutex - thread-safe
      safeIncrement = mutex.lock.surround {
        counter.get.flatMap { n =>
          IO.sleep(1.millis) >> counter.set(n + 1)
        }
      }
      
      _ <- IO.println("Running 10 concurrent increments with mutex:")
      _ <- List.range(1, 11).parTraverse(_ => safeIncrement)
      result <- counter.get
      _ <- IO.println(s"Final count: $result (expected: 10)")
    yield ()

  // Mutex สำหรับ cache
  class SafeCache[K, V](
    store: Ref[IO, Map[K, V]],
    mutex: Mutex[IO]
  ):
    def get(key: K): IO[Option[V]] =
      store.get.map(_.get(key))
    
    def put(key: K, value: V): IO[Unit] =
      mutex.lock.surround {
        store.update(_ + (key -> value))
      }
    
    def getOrFetch(key: K)(fetch: IO[V]): IO[V] =
      get(key).flatMap {
        case Some(v) => IO.pure(v)
        case None =>
          mutex.lock.surround {
            // Double-check inside mutex
            get(key).flatMap {
              case Some(v) => IO.pure(v)
              case None =>
                fetch.flatMap(v => put(key, v) >> IO.pure(v))
            }
          }
      }

  object SafeCache:
    def make[K, V]: IO[SafeCache[K, V]] =
      for
        store <- Ref.of[IO, Map[K, V]](Map.empty)
        mutex <- Mutex[IO]
      yield new SafeCache(store, mutex)

  def cacheDemo: IO[Unit] =
    SafeCache.make[String, Int].flatMap { cache =>
      val fetchCount = Ref.of[IO, Int](0)
      
      fetchCount.flatMap { counter =>
        val expensiveFetch = (key: String) =>
          counter.updateAndGet(_ + 1).flatMap { count =>
            IO.sleep(100.millis) >>
            IO.println(s"  Fetching $key (fetch #$count)") >>
            IO.pure(key.length)
          }
        
        IO.println("Cache demo (concurrent reads):") >>
        List("hello", "world", "hello", "scala", "world", "hello")
          .parTraverse { key =>
            cache.getOrFetch(key)(expensiveFetch(key)).flatMap { value =>
              IO.println(s"  key='$key' -> value=$value")
            }
          }.void >>
        counter.get.flatMap(n => IO.println(s"Total fetches: $n (should be 3)"))
      }
    }

  def run: IO[Unit] =
    IO.println("=== Basic Mutex ===") >>
    basicMutex >>
    IO.println("\n=== Thread-safe Cache ===") >>
    cacheDemo
```

---

## CountDownLatch และ Synchronization

```scala
import cats.effect.*
import cats.effect.std.CountDownLatch
import scala.concurrent.duration.*

object CountDownLatchDemo extends IOApp.Simple:

  // CountDownLatch - รอจนกว่า N events จะเกิดขึ้น
  def basicLatch: IO[Unit] =
    for
      latch <- CountDownLatch[IO](3)
      
      // 3 workers ที่ต้องเสร็จก่อน main thread จะดำเนินการต่อ
      workers = List.range(1, 4).map { i =>
        IO.sleep((i * 100).millis) >>
        IO.println(s"Worker $i done, counting down") >>
        latch.release
      }
      
      // Start all workers
      _ <- workers.parTraverse(w => w.start)
      
      _ <- IO.println("Waiting for all workers to complete...")
      _ <- latch.await  // block จนกว่า count จะถึง 0
      _ <- IO.println("All workers done! Proceeding...")
    yield ()

  // Barrier pattern ด้วย CountDownLatch
  def barrierDemo: IO[Unit] =
    for
      ready  <- CountDownLatch[IO](5)  // รอให้ทุก worker พร้อม
      done   <- CountDownLatch[IO](5)  // รอให้ทุก worker เสร็จ
      
      workers = List.range(1, 6).map { i =>
        for
          _ <- IO.sleep((i * 30).millis)  // initialize time
          _ <- IO.println(s"Worker $i ready")
          _ <- ready.release
          _ <- ready.await  // รอให้ทุกคนพร้อม
          _ <- IO.println(s"Worker $i starting work")
          _ <- IO.sleep((i * 50 % 200).millis)  // do work
          _ <- IO.println(s"Worker $i finished")
          _ <- done.release
        yield ()
      }
      
      _ <- IO.println("Starting barrier demo:")
      _ <- workers.parTraverse(_.start)
      _ <- done.await
      _ <- IO.println("All workers completed their work!")
    yield ()

  def run: IO[Unit] =
    IO.println("=== CountDownLatch ===") >>
    basicLatch >>
    IO.println("\n=== Barrier Demo ===") >>
    barrierDemo
```

---

## Deferred และ Promise Patterns

```scala
import cats.effect.*
import cats.effect.std.Deferred
import scala.concurrent.duration.*

object DeferredDemo extends IOApp.Simple:

  // Deferred - single-use channel สำหรับส่ง value ระหว่าง fibers
  def basicDeferred: IO[Unit] =
    for
      deferred <- Deferred[IO, Int]
      
      // Producer fiber
      producerFiber <- (
        IO.sleep(300.millis) >>
        IO.println("Producer: completing deferred with 42") >>
        deferred.complete(42)
      ).start
      
      // Consumer รอ value
      _ <- IO.println("Consumer: waiting for value...")
      value <- deferred.get
      _ <- IO.println(s"Consumer: got value $value")
      _ <- producerFiber.join
    yield ()

  // Promise-like pattern
  class Promise[A](deferred: Deferred[IO, Either[Throwable, A]]):
    def resolve(value: A): IO[Boolean] =
      deferred.complete(Right(value))
    
    def reject(error: Throwable): IO[Boolean] =
      deferred.complete(Left(error))
    
    def future: IO[A] =
      deferred.get.flatMap {
        case Right(value) => IO.pure(value)
        case Left(error)  => IO.raiseError(error)
      }

  object Promise:
    def make[A]: IO[Promise[A]] =
      Deferred[IO, Either[Throwable, A]].map(new Promise(_))

  def promiseDemo: IO[Unit] =
    for
      promise <- Promise.make[String]
      
      // Async computation
      _ <- (
        IO.sleep(200.millis) >>
        promise.resolve("Hello from async!")
      ).start
      
      // Wait for result
      result <- promise.future
      _ <- IO.println(s"Promise resolved: $result")
    yield ()

  // One-shot initialization pattern
  class LazyInit[A](init: IO[A]):
    private val deferred = Deferred.unsafe[IO, A]
    private val initialized = Ref.unsafe[IO, Boolean](false)
    
    def get: IO[A] =
      initialized.get.flatMap {
        case true  => deferred.get
        case false =>
          initialized.compareAndSet(false, true).flatMap { acquired =>
            if acquired then
              init.flatMap(a => deferred.complete(a) >> IO.pure(a))
            else
              deferred.get
          }
      }

  def run: IO[Unit] =
    IO.println("=== Basic Deferred ===") >>
    basicDeferred >>
    IO.println("\n=== Promise Pattern ===") >>
    promiseDemo
```

---

## Complete Concurrent Example

### Distributed Task Processing System

```scala
import cats.effect.*
import cats.effect.std.*
import cats.syntax.all.*
import fs2.*
import scala.concurrent.duration.*

// Domain
case class Task(id: Int, priority: Int, payload: String)
case class TaskResult(taskId: Int, result: String, worker: Int, duration: Long)
case class WorkerStats(
  workerId: Int,
  tasksProcessed: Int,
  totalDuration: Long,
  errors: Int
)

object DistributedTaskProcessor extends IOApp.Simple:

  // ===== Worker Pool =====
  def makeWorker(
    workerId: Int,
    taskQueue: Queue[IO, Option[Task]],
    resultQueue: Queue[IO, TaskResult],
    stats: Ref[IO, Map[Int, WorkerStats]]
  ): IO[Unit] =
    Stream
      .fromQueueNoneTerminated(taskQueue, chunkSize = 1)
      .evalMap { task =>
        val start = System.currentTimeMillis()
        
        // Process task (with random failure simulation)
        val process =
          if task.id % 7 == 0 then
            IO.raiseError(new RuntimeException(s"Task ${task.id} failed!"))
          else
            IO.sleep((50 + task.priority * 10).millis) >>
            IO.pure(s"processed:${task.payload.toUpperCase}")

        process
          .attempt
          .flatMap { result =>
            val duration = System.currentTimeMillis() - start
            
            result match
              case Right(output) =>
                resultQueue.offer(TaskResult(task.id, output, workerId, duration)) >>
                stats.update { m =>
                  val ws = m.getOrElse(workerId, WorkerStats(workerId, 0, 0, 0))
                  m + (workerId -> ws.copy(
                    tasksProcessed = ws.tasksProcessed + 1,
                    totalDuration = ws.totalDuration + duration
                  ))
                }
              case Left(error) =>
                IO.println(s"Worker $workerId: Error processing task ${task.id}: ${error.getMessage}") >>
                stats.update { m =>
                  val ws = m.getOrElse(workerId, WorkerStats(workerId, 0, 0, 0))
                  m + (workerId -> ws.copy(errors = ws.errors + 1))
                }
          }
      }
      .compile
      .drain

  // ===== Result Processor =====
  def makeResultProcessor(
    resultQueue: Queue[IO, TaskResult],
    processedCount: Ref[IO, Int],
    expectedCount: Int
  ): IO[Unit] =
    Stream
      .fromQueueUnterminated(resultQueue)
      .evalMap { result =>
        IO.println(s"Result: task=${result.taskId}, worker=${result.worker}, duration=${result.duration}ms") >>
        processedCount.updateAndGet(_ + 1)
      }
      .takeWhile(_ < expectedCount)
      .compile
      .drain

  // ===== Monitoring =====
  def makeMonitor(
    stats: Ref[IO, Map[Int, WorkerStats]],
    processedCount: Ref[IO, Int],
    stopSignal: Deferred[IO, Unit]
  ): IO[Unit] =
    Stream
      .fixedDelay[IO](500.millis)
      .evalMap { _ =>
        for
          allStats  <- stats.get
          processed <- processedCount.get
          _ <- IO.println(s"""
            |--- Monitor Update (${System.currentTimeMillis()}ms) ---
            |Tasks processed: $processed
            |Worker stats:
            |${allStats.values.map(ws =>
              s"  Worker ${ws.workerId}: ${ws.tasksProcessed} tasks, " +
              s"avg=${if ws.tasksProcessed > 0 then ws.totalDuration/ws.tasksProcessed else 0}ms, " +
              s"errors=${ws.errors}"
            ).mkString("\n")}
            |""".stripMargin)
        yield ()
      }
      .interruptWhen(stopSignal.get.as(true))
      .compile
      .drain

  // ===== Main =====
  def run: IO[Unit] =
    val numWorkers = 4
    val numTasks = 30

    for
      taskQueue  <- Queue.bounded[IO, Option[Task]](50)
      resultQueue <- Queue.bounded[IO, TaskResult](100)
      stats      <- Ref.of[IO, Map[Int, WorkerStats]](Map.empty)
      processed  <- Ref.of[IO, Int](0)
      stopSignal <- Deferred[IO, Unit]

      _ <- IO.println(s"Starting task processor with $numWorkers workers, $numTasks tasks\n")

      // Start workers
      workerFibers <- List.range(1, numWorkers + 1).traverse { id =>
        makeWorker(id, taskQueue, resultQueue, stats).start
      }

      // Start result processor
      resultFiber <- makeResultProcessor(resultQueue, processed, numTasks).start

      // Start monitor
      monitorFiber <- makeMonitor(stats, processed, stopSignal).start

      // Enqueue tasks
      _ <- List.range(1, numTasks + 1).traverse { i =>
        taskQueue.offer(Some(Task(i, i % 5 + 1, s"task-payload-$i")))
      }

      // Signal workers to stop
      _ <- List.range(0, numWorkers).traverse(_ => taskQueue.offer(None))

      // Wait for result processor
      _ <- resultFiber.join

      // Stop monitoring
      _ <- stopSignal.complete(())
      _ <- monitorFiber.cancel

      // Wait for workers
      _ <- workerFibers.traverse(_.join)

      // Final stats
      finalStats <- stats.get
      _ <- IO.println("\n=== Final Statistics ===")
      _ <- finalStats.values.toList.sortBy(_.workerId).traverse { ws =>
        IO.println(s"""
          |Worker ${ws.workerId}:
          |  Tasks processed: ${ws.tasksProcessed}
          |  Errors: ${ws.errors}
          |  Avg duration: ${if ws.tasksProcessed > 0 then ws.totalDuration/ws.tasksProcessed else 0}ms
          |""".stripMargin)
      }
    yield ()
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Advanced Effect Patterns ใน cats-effect:

### สรุปเครื่องมือที่เรียนรู้

| เครื่องมือ | วัตถุประสงค์ | เมื่อไหรใช้ |
|----------|-------------|-----------|
| `Resource[F, A]` | Lifecycle management | จัดการ resources ที่ต้อง cleanup |
| `IO.bracket` | Acquire-use-release | Low-level resource management |
| `IO.guarantee` | Finalizer | ทำ cleanup เสมอ |
| `IO.race` | Racing effects | First result wins |
| `IO.timeout` | Time limiting | ป้องกัน hanging operations |
| `Fiber` | Lightweight threads | Concurrent work |
| `Semaphore` | Access limiting | จำกัด concurrent access |
| `Mutex` | Exclusive access | Thread-safe mutations |
| `CountDownLatch` | Synchronization | รอ N events |
| `Deferred` | Single-value channel | Communication between fibers |

### Best Practices

1. **ใช้ Resource แทน bracket** - Resource compose ได้ดีกว่า
2. **ใช้ parMapN / parTraverse** แทน manual fibers - สะดวกกว่า
3. **Handle cancellation** - ใช้ `onCancel` สำหรับ cleanup
4. **Bound ทุก Queue** - ป้องกัน memory overflow
5. **Test timeouts** - ใช้ `IO.sleep` ใน tests
6. **Avoid shared mutable state** - ใช้ `Ref` หรือ `Deferred` แทน

---

*[← Part 56: Typeclass Derivation](part-56-typeclass-derivation.md) | [Part 58: Schema Evolution →](part-58-schema-evolution.md)*
