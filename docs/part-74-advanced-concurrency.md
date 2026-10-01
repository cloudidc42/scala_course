# ส่วนที่ 74: Advanced Concurrency - การเขียนโปรแกรมแบบขนานขั้นสูง

## สารบัญ

1. [JVM Memory Model](#jvm-memory-model)
2. [Volatile, Synchronized, AtomicReference](#volatile-synchronized-atomic)
3. [Lock-free Data Structures](#lock-free-structures)
4. [STM (Software Transactional Memory) ด้วย cats-effect](#stm-cats-effect)
5. [Parallel Computation ด้วย IO.parSequence](#parallel-computation)
6. [cats-effect 3 Fiber Scheduler](#fiber-scheduler)
7. [Worker Pools และ Task Queues](#worker-pools)
8. [ตัวอย่าง High-Concurrency สมบูรณ์](#high-concurrency-example)
9. [สรุป](#summary)

---

## 1. JVM Memory Model {#jvm-memory-model}

### Java Memory Model (JMM) พื้นฐาน

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.typelevel" %% "cats-effect"       % "3.5.4",
  "org.typelevel" %% "cats-effect-std"   % "3.5.4",
  "org.typelevel" %% "cats-effect-kernel"% "3.5.4",
  "co.fs2"        %% "fs2-core"          % "3.10.2",
  "org.typelevel" %% "cats-stm"          % "0.13.4"
)
```

```scala
import java.util.concurrent.atomic.*
import scala.concurrent.{Future, ExecutionContext}
import scala.concurrent.ExecutionContext.Implicits.global

object JVMMemoryModelDemo:
  
  // === Happens-Before Relationship ===
  
  // ปัญหา: Data race
  class UnsafeCounter:
    private var count = 0  // NOT thread-safe
    
    def increment(): Unit = count += 1  // read-modify-write: NOT atomic
    def get(): Int = count
  
  // ทดสอบ data race
  def demonstrateDataRace(): Unit =
    val counter = new UnsafeCounter()
    val threads = (1 to 10).map { _ =>
      new Thread(() => {
        (1 to 1000).foreach(_ => counter.increment())
      })
    }
    
    threads.foreach(_.start())
    threads.foreach(_.join())
    
    // ผลลัพธ์จะน้อยกว่า 10000 เนื่องจาก data race
    println(s"Expected: 10000, Got: ${counter.get()}")
  
  // === Memory Visibility ===
  
  class VisibilityProblem:
    // ไม่มี visibility guarantee
    private var flag = false
    private var data = 0
    
    def writer(): Unit =
      data = 42       // write data
      flag = true     // signal
    
    def reader(): Unit =
      while !flag do {}  // spin wait (อาจ loop forever)
      println(s"Data: $data")  // อาจเห็น data = 0 ถ้าไม่มี memory barrier
  
  // === Cache Coherency ===
  // L1 cache per core -> L2 -> L3 -> Main Memory
  // Cache line = 64 bytes ปกติ
  
  // False sharing - performance killer
  class FalseSharing:
    // สองตัวแปรอยู่ใน cache line เดียวกัน
    @volatile var counter1 = 0L  // อาจอยู่ใน cache line เดียวกับ counter2
    @volatile var counter2 = 0L
  
  // แก้ด้วย padding
  class PaddedCounters:
    @volatile var counter1 = 0L
    // padding เพื่อให้แต่ละตัวอยู่ cache line ต่างกัน
    var p1, p2, p3, p4, p5, p6, p7 = 0L  // 7 * 8 = 56 bytes padding
    @volatile var counter2 = 0L
    var q1, q2, q3, q4, q5, q6, q7 = 0L
  
  // Benchmark false sharing vs padded
  def benchmarkFalseSharing(): Unit =
    val iterations = 100_000_000
    
    // With false sharing
    val shared = new FalseSharing()
    val start1 = System.nanoTime()
    val t1 = new Thread(() => (0 until iterations).foreach(_ => shared.counter1 += 1))
    val t2 = new Thread(() => (0 until iterations).foreach(_ => shared.counter2 += 1))
    t1.start(); t2.start(); t1.join(); t2.join()
    val time1 = (System.nanoTime() - start1) / 1e6
    
    // Without false sharing
    val padded = new PaddedCounters()
    val start2 = System.nanoTime()
    val t3 = new Thread(() => (0 until iterations).foreach(_ => padded.counter1 += 1))
    val t4 = new Thread(() => (0 until iterations).foreach(_ => padded.counter2 += 1))
    t3.start(); t4.start(); t3.join(); t4.join()
    val time2 = (System.nanoTime() - start2) / 1e6
    
    println(f"False sharing: $time1%.0fms")
    println(f"Padded: $time2%.0fms")
    println(f"Speedup: ${time1/time2:.1f}x")
  
  def main(args: Array[String]): Unit =
    demonstrateDataRace()
    benchmarkFalseSharing()
```

---

## 2. Volatile, Synchronized, AtomicReference {#volatile-synchronized-atomic}

### Thread-safe Primitives

```scala
import java.util.concurrent.atomic.*
import java.util.concurrent.locks.*
import scala.annotation.tailrec

object ThreadSafetyDemo:
  
  // === @volatile ===
  // Guarantees: visibility (not atomicity)
  class VolatileFlag:
    @volatile private var running = true
    @volatile private var data: Option[String] = None
    
    def stop(): Unit =
      running = false
    
    def setData(value: String): Unit =
      data = Some(value)
    
    def run(): Unit =
      while running do
        data.foreach { d =>
          println(s"Processing: $d")
          data = None
        }
        Thread.sleep(1)
  
  // === synchronized ===
  class SynchronizedCounter:
    private var count = 0
    
    def increment(): Unit = synchronized {
      count += 1
    }
    
    def decrement(): Unit = synchronized {
      count -= 1
    }
    
    def get(): Int = synchronized {
      count
    }
    
    // Double-checked locking pattern (DCL)
    @volatile private var instance: Option[String] = None
    
    def getInstance(): String = instance match
      case Some(v) => v
      case None => synchronized {
        instance match
          case Some(v) => v
          case None =>
            val newInstance = "Singleton"
            instance = Some(newInstance)
            newInstance
      }
  
  // === AtomicReference ===
  class AtomicStack[A]:
    import scala.jdk.CollectionConverters.*
    
    private val stack = new AtomicReference[List[A]](List.empty)
    
    @tailrec
    final def push(elem: A): Unit =
      val current = stack.get()
      val next = elem :: current
      if !stack.compareAndSet(current, next) then
        push(elem)  // retry if CAS fails
    
    @tailrec
    final def pop(): Option[A] =
      val current = stack.get()
      current match
        case Nil => None
        case head :: tail =>
          if stack.compareAndSet(current, tail) then Some(head)
          else pop()  // retry
    
    def peek(): Option[A] = stack.get().headOption
    def size(): Int = stack.get().size
    def toList: List[A] = stack.get()
  
  // === ReentrantLock ===
  class ReadWriteCache[K, V](maxSize: Int = 1000):
    private val cache = scala.collection.mutable.LinkedHashMap[K, V]()
    private val lock = new ReentrantReadWriteLock()
    private val readLock = lock.readLock()
    private val writeLock = lock.writeLock()
    
    def get(key: K): Option[V] =
      readLock.lock()
      try cache.get(key)
      finally readLock.unlock()
    
    def put(key: K, value: V): Unit =
      writeLock.lock()
      try
        if cache.size >= maxSize then
          cache.remove(cache.keys.head)
        cache.put(key, value)
      finally writeLock.unlock()
    
    def getOrCompute(key: K)(compute: => V): V =
      get(key) match
        case Some(v) => v
        case None =>
          val v = compute
          put(key, v)
          v
    
    def size(): Int =
      readLock.lock()
      try cache.size
      finally readLock.unlock()
  
  // === Atomic Operations - Compare and Swap ===
  object CASDemo:
    
    val counter = new AtomicLong(0)
    val ref = new AtomicReference[String]("initial")
    
    // CAS-based increment
    @tailrec
    def incrementAndGet(): Long =
      val current = counter.get()
      val next = current + 1
      if counter.compareAndSet(current, next) then next
      else incrementAndGet()
    
    // Update with transformation
    def updateRef(transform: String => String): String =
      var current = ref.get()
      var updated = transform(current)
      while !ref.compareAndSet(current, updated) do
        current = ref.get()
        updated = transform(current)
      updated
    
    def main(): Unit =
      // Concurrent increments
      val threads = (1 to 10).map { _ =>
        new Thread(() => (1 to 10000).foreach(_ => incrementAndGet()))
      }
      threads.foreach(_.start())
      threads.foreach(_.join())
      println(s"Final count: ${counter.get()}")  // Exactly 100000
      
      // Atomic reference update
      val result = updateRef(s => s + "_modified")
      println(s"Updated ref: $result")
  
  def main(args: Array[String]): Unit =
    CASDemo.main()
    
    val stack = new AtomicStack[Int]()
    val pushThreads = (1 to 5).map { t =>
      new Thread(() => (1 to 100).foreach(i => stack.push(t * 100 + i)))
    }
    pushThreads.foreach(_.start())
    pushThreads.foreach(_.join())
    println(s"Stack size: ${stack.size()}")  // 500
    
    val cache = new ReadWriteCache[String, Int]()
    val writeThread = new Thread(() => (1 to 1000).foreach(i => cache.put(s"key$i", i)))
    val readThreads = (1 to 5).map { _ =>
      new Thread(() => (1 to 200).foreach(i => cache.get(s"key$i")))
    }
    writeThread.start()
    readThreads.foreach(_.start())
    writeThread.join()
    readThreads.foreach(_.join())
    println(s"Cache size: ${cache.size()}")
```

---

## 3. Lock-free Data Structures {#lock-free-structures}

```scala
import java.util.concurrent.atomic.*

object LockFreeDataStructures:
  
  // === Lock-free Queue (Michael-Scott Queue) ===
  class LockFreeQueue[A]:
    private class Node[A](val value: Option[A]):
      val next = new AtomicReference[Node[A]](null)
    
    private val head = new AtomicReference[Node[A]](new Node(None))  // sentinel
    private val tail = new AtomicReference[Node[A]](head.get())
    
    import scala.annotation.tailrec
    
    def enqueue(value: A): Unit =
      val newNode = new Node(Some(value))
      
      @tailrec
      def tryEnqueue(): Unit =
        val currentTail = tail.get()
        val tailNext = currentTail.next.get()
        
        if currentTail == tail.get() then
          if tailNext == null then
            if currentTail.next.compareAndSet(null, newNode) then
              tail.compareAndSet(currentTail, newNode)
              // success
            else
              tryEnqueue()  // retry
          else
            tail.compareAndSet(currentTail, tailNext)  // advance tail
            tryEnqueue()
      
      tryEnqueue()
    
    @tailrec
    final def dequeue(): Option[A] =
      val currentHead = head.get()
      val currentTail = tail.get()
      val headNext = currentHead.next.get()
      
      if currentHead == head.get() then
        if currentHead == currentTail then
          if headNext == null then None  // empty
          else
            tail.compareAndSet(currentTail, headNext)  // advance tail
            dequeue()
        else
          val value = headNext.value
          if head.compareAndSet(currentHead, headNext) then value
          else dequeue()  // retry
      else dequeue()
    
    def isEmpty: Boolean = head.get().next.get() == null
  
  // === Treiber Stack (Lock-free) ===
  class TreiberStack[A]:
    private val top = new AtomicReference[List[A]](Nil)
    
    import scala.annotation.tailrec
    
    @tailrec
    final def push(elem: A): Unit =
      val current = top.get()
      if !top.compareAndSet(current, elem :: current) then push(elem)
    
    @tailrec
    final def pop(): Option[A] = top.get() match
      case Nil => None
      case h :: t =>
        if top.compareAndSet(h :: t, t) then Some(h)
        else pop()
    
    def peek: Option[A] = top.get().headOption
  
  // === Lock-free HashMap ===
  class ConcurrentHashMap[K, V](initialCapacity: Int = 16):
    import java.util.concurrent.ConcurrentHashMap as JConcurrentHashMap
    
    private val map = new JConcurrentHashMap[K, V](initialCapacity)
    
    def put(key: K, value: V): Option[V] = 
      Option(map.put(key, value))
    
    def get(key: K): Option[V] = 
      Option(map.get(key))
    
    def computeIfAbsent(key: K)(compute: K => V): V =
      map.computeIfAbsent(key, compute(_))
    
    def updateValue(key: K)(update: V => V): Option[V] =
      var result: Option[V] = None
      map.computeIfPresent(key, (k, v) => {
        val newV = update(v)
        result = Some(newV)
        newV
      })
      result
    
    def atomicPutIfAbsent(key: K, value: V): Boolean =
      map.putIfAbsent(key, value) == null
    
    def size: Int = map.size()
    def toMap: Map[K, V] = 
      import scala.jdk.CollectionConverters.*
      map.asScala.toMap
  
  def main(args: Array[String]): Unit =
    // ทดสอบ lock-free queue
    val queue = new LockFreeQueue[Int]()
    val producers = (1 to 4).map { id =>
      new Thread(() => (1 to 1000).foreach(i => queue.enqueue(id * 1000 + i)))
    }
    
    var consumed = new AtomicInteger(0)
    val consumers = (1 to 2).map { _ =>
      new Thread(() => {
        var running = true
        while running do
          queue.dequeue() match
            case Some(_) => consumed.incrementAndGet()
            case None    => 
              Thread.sleep(1)
              if consumed.get() >= 4000 then running = false
      })
    }
    
    producers.foreach(_.start())
    consumers.foreach(_.start())
    producers.foreach(_.join())
    Thread.sleep(100)
    consumers.foreach(_.interrupt())
    
    println(s"Consumed: ${consumed.get()}")  // Should be close to 4000
    
    // ทดสอบ concurrent hashmap
    val concMap = new ConcurrentHashMap[String, AtomicInteger]()
    val threads = (1 to 10).map { _ =>
      new Thread(() => {
        (1 to 1000).foreach { i =>
          val key = s"key${i % 100}"
          val counter = concMap.computeIfAbsent(key)(_ => new AtomicInteger(0))
          counter.incrementAndGet()
        }
      })
    }
    threads.foreach(_.start())
    threads.foreach(_.join())
    
    val total = concMap.toMap.values.map(_.get()).sum
    println(s"Total increments: $total")  // Should be 10000
```

---

## 4. STM (Software Transactional Memory) ด้วย cats-effect {#stm-cats-effect}

```scala
import cats.effect.*
import cats.effect.std.Semaphore
import cats.syntax.all.*
import cats.effect.unsafe.implicits.global

object STMWithCatsEffect:
  
  // === cats-effect Ref (mutable reference) ===
  def demonstrateRef(): IO[Unit] =
    for
      // สร้าง thread-safe mutable reference
      counter <- IO.ref(0)
      
      // Concurrent increments
      fibers <- (1 to 1000).toList.parTraverse { _ =>
        counter.update(_ + 1)
      }
      
      result <- counter.get
      _ <- IO.println(s"Counter: $result")  // Should be 1000
      
      // Atomic modify with result
      oldValue <- counter.getAndUpdate(_ * 2)
      newValue <- counter.get
      _ <- IO.println(s"Old: $oldValue, New: $newValue")
      
      // modify with custom logic
      _ <- counter.modify { n =>
        val newN = n + 100
        (newN, s"Updated from $n to $newN")
      }.flatMap(IO.println)
    yield ()
  
  // === cats-stm - Software Transactional Memory ===
  import io.github.timwspence.cats.stm.*
  
  def demonstrateSTM(): IO[Unit] =
    STM.runtime[IO].flatMap { implicit stm =>
      import stm.*
      
      // สร้าง transactional variables
      for
        accountA <- TVar.of(1000.0)
        accountB <- TVar.of(500.0)
        
        // Atomic transfer
        def transfer(from: TVar[Double], to: TVar[Double], amount: Double): STM[Unit] =
          for
            fromBalance <- from.get
            _ <- STM.check(fromBalance >= amount)  // retry if insufficient
            _ <- from.modify(_ - amount)
            _ <- to.modify(_ + amount)
          yield ()
        
        // Run concurrent transfers
        _ <- List(
          stm.commit(transfer(accountA, accountB, 200.0)),
          stm.commit(transfer(accountB, accountA, 100.0)),
          stm.commit(transfer(accountA, accountB, 50.0))
        ).parSequence
        
        balanceA <- stm.commit(accountA.get)
        balanceB <- stm.commit(accountB.get)
        _ <- IO.println(s"Account A: $balanceA")  // 1000 - 200 + 100 - 50 = 850
        _ <- IO.println(s"Account B: $balanceB")  // 500 + 200 - 100 + 50 = 650
        _ <- IO.println(s"Total: ${balanceA + balanceB}")  // 1500 (unchanged)
      yield ()
    }
  
  // === Deferred - promise-like ===
  def demonstrateDeferred(): IO[Unit] =
    for
      deferred <- IO.deferred[String]
      
      // Producer fiber
      producer <- IO.sleep(scala.concurrent.duration.DurationInt(100).millis)
        .flatMap(_ => deferred.complete("Hello from producer!"))
        .start
      
      // Consumer - waits for value
      result <- deferred.get
      _ <- IO.println(s"Got: $result")
      _ <- producer.join
    yield ()
  
  // === Mutex ด้วย Semaphore ===
  def demonstrateMutex(): IO[Unit] =
    for
      mutex <- Semaphore[IO](1)
      counter <- IO.ref(0)
      
      // Critical section ด้วย mutex
      fibers <- (1 to 100).toList.parTraverse { _ =>
        mutex.permit.use { _ =>
          // เฉพาะ 1 fiber ที่สามารถรัน section นี้พร้อมกัน
          counter.update(_ + 1)
        }
      }
      
      result <- counter.get
      _ <- IO.println(s"Mutex counter: $result")  // 100
    yield ()
  
  // === Bounded Semaphore (rate limiting) ===
  def demonstrateBoundedSemaphore(): IO[Unit] =
    for
      // อนุญาตสูงสุด 3 concurrent operations
      sem <- Semaphore[IO](3)
      results <- (1 to 10).toList.parTraverse { i =>
        sem.permit.use { _ =>
          for
            _ <- IO.println(s"Starting operation $i")
            _ <- IO.sleep(scala.concurrent.duration.DurationInt(100).millis)
            _ <- IO.println(s"Completed operation $i")
          yield i * 2
        }
      }
      _ <- IO.println(s"All results: $results")
    yield ()
  
  def main(args: Array[String]): Unit =
    val program = for
      _ <- IO.println("=== Ref Demo ===")
      _ <- demonstrateRef()
      _ <- IO.println("\n=== Deferred Demo ===")
      _ <- demonstrateDeferred()
      _ <- IO.println("\n=== Mutex Demo ===")
      _ <- demonstrateMutex()
      _ <- IO.println("\n=== Bounded Semaphore Demo ===")
      _ <- demonstrateBoundedSemaphore()
    yield ()
    
    program.unsafeRunSync()
```

---

## 5. Parallel Computation ด้วย IO.parSequence {#parallel-computation}

```scala
import cats.effect.*
import cats.syntax.all.*
import scala.concurrent.duration.*

object ParallelComputationDemo:
  
  // Simulate expensive computation
  def compute(id: Int, delay: Int): IO[Int] =
    IO.sleep(delay.millis) >> IO(id * id)
  
  def demonstrateParallelism(): IO[Unit] =
    val tasks = (1 to 10).toList.map(i => compute(i, i * 50))
    
    for
      // Sequential - รวม ~5500ms
      start1 <- IO.realTime
      seqResults <- tasks.sequence
      end1 <- IO.realTime
      _ <- IO.println(f"Sequential: ${(end1 - start1).toMillis}ms, results: ${seqResults.sum}")
      
      // Parallel - รวม ~500ms (longest task)
      start2 <- IO.realTime
      parResults <- tasks.parSequence
      end2 <- IO.realTime
      _ <- IO.println(f"Parallel: ${(end2 - start2).toMillis}ms, results: ${parResults.sum}")
    yield ()
  
  // parMapN - รัน N fibers พร้อมกัน แล้วรวมผลลัพธ์
  def demonstrateParMapN(): IO[Unit] =
    val fetchUser: IO[String] = IO.sleep(100.millis) >> IO.pure("Alice")
    val fetchOrders: IO[List[String]] = IO.sleep(150.millis) >> IO.pure(List("O1", "O2"))
    val fetchRecommendations: IO[List[String]] = IO.sleep(80.millis) >> IO.pure(List("P1", "P2", "P3"))
    
    for
      start <- IO.realTime
      // รันทั้ง 3 พร้อมกัน
      (user, orders, recs) <- (fetchUser, fetchOrders, fetchRecommendations).parMapN {
        (u, o, r) => (u, o, r)
      }
      end <- IO.realTime
      _ <- IO.println(f"Fetched in ${(end - start).toMillis}ms")
      _ <- IO.println(s"User: $user, Orders: $orders, Recs: $recs")
    yield ()
  
  // parTraverseN - จำกัดจำนวน concurrent fibers
  def demonstrateParTraverseN(): IO[Unit] =
    val items = (1 to 20).toList
    
    for
      results <- items.parTraverseN(5) { i =>  // สูงสุด 5 fibers พร้อมกัน
        IO.sleep(50.millis) >> IO(i * 2)
      }
      _ <- IO.println(s"Results: ${results.take(5)}...")
    yield ()
  
  // Racing - รันหลาย tasks แข่งกัน เอาที่ชนะก่อน
  def demonstrateRacing(): IO[Unit] =
    val fast = IO.sleep(100.millis) >> IO.pure("Fast won!")
    val slow = IO.sleep(500.millis) >> IO.pure("Slow won!")
    val failed = IO.sleep(50.millis) >> IO.raiseError[String](new Exception("Failed!"))
    
    // race - เอา winner, cancel loser
    for
      result1 <- IO.race(fast, slow)
      _ <- IO.println(s"Race 1: $result1")
      
      // raceN - หลาย tasks
      result2 <- (fast, slow, IO.sleep(200.millis) >> IO.pure("Medium won!")).parMapN {
        (a, b, c) => (a, b, c)
      }
      _ <- IO.println(s"All done: $result2")
    yield ()
  
  // Parallel aggregation
  def parallelWordCount(texts: List[String]): IO[Map[String, Int]] =
    texts.parTraverse { text =>
      IO {
        text.split("\\s+").groupMapReduce(identity)(_ => 1)(_ + _)
      }
    }.map { maps =>
      maps.foldLeft(Map.empty[String, Int]) { (acc, m) =>
        m.foldLeft(acc) { case (a, (k, v)) =>
          a + (k -> (a.getOrElse(k, 0) + v))
        }
      }
    }
  
  def main(args: Array[String]): Unit =
    val program = for
      _ <- IO.println("=== Sequential vs Parallel ===")
      _ <- demonstrateParallelism()
      _ <- IO.println("\n=== parMapN ===")
      _ <- demonstrateParMapN()
      _ <- IO.println("\n=== parTraverseN ===")
      _ <- demonstrateParTraverseN()
      _ <- IO.println("\n=== Racing ===")
      _ <- demonstrateRacing()
      _ <- IO.println("\n=== Parallel Word Count ===")
      texts = List(
        "the quick brown fox jumps over the lazy dog",
        "the fox is quick and the dog is lazy",
        "brown dogs jump over quick foxes"
      )
      wc <- parallelWordCount(texts)
      _ <- IO.println(s"Word count: ${wc.toList.sortBy(-_._2).take(5)}")
    yield ()
    
    program.unsafeRunSync()
```

---

## 6. cats-effect 3 Fiber Scheduler {#fiber-scheduler}

```scala
import cats.effect.*
import cats.effect.std.*
import cats.syntax.all.*
import scala.concurrent.duration.*

object FiberSchedulerDemo:
  
  // === Fibers - lightweight threads ===
  def demonstrateFibers(): IO[Unit] =
    for
      // spawn fiber
      fiber1 <- IO.println("Fiber 1 started")
        .flatMap(_ => IO.sleep(100.millis))
        .flatMap(_ => IO.println("Fiber 1 done"))
        .start
      
      fiber2 <- IO.println("Fiber 2 started")
        .flatMap(_ => IO.sleep(50.millis))
        .flatMap(_ => IO.println("Fiber 2 done"))
        .start
      
      _ <- IO.println("Main fiber continues")
      
      // รอ fibers
      _ <- fiber2.join
      _ <- fiber1.join
      _ <- IO.println("All fibers joined")
    yield ()
  
  // === Fiber cancellation ===
  def demonstrateCancellation(): IO[Unit] =
    val longRunning = IO.println("Long task started") >>
      IO.sleep(10.seconds) >>
      IO.println("Long task done")  // จะไม่รันถ้า cancelled
    
    for
      fiber <- longRunning.start
      _ <- IO.sleep(100.millis)
      _ <- fiber.cancel
      _ <- IO.println("Fiber cancelled")
      outcome <- fiber.join
      _ <- outcome match
        case Outcome.Succeeded(_) => IO.println("Succeeded (unexpected)")
        case Outcome.Errored(e)   => IO.println(s"Errored: $e")
        case Outcome.Canceled()   => IO.println("Confirmed: fiber was cancelled")
    yield ()
  
  // === Fiber pool control ===
  def demonstrateFiberPool(): IO[Unit] =
    // cats-effect ใช้ work-stealing thread pool โดยอัตโนมัติ
    // จำนวน threads = จำนวน CPU cores
    
    val cpuBound = IO.blocking {
      // CPU-intensive work - รันบน blocking thread pool
      (1 to 1000000).sum
    }
    
    val ioBound = IO.interruptible {
      // I/O operation ที่อาจ block - รันบน blocking pool แต่ interruptible
      Thread.sleep(100)
      "I/O result"
    }
    
    for
      // CPU-intensive tasks
      cpuResults <- (1 to 4).toList.parTraverse { _ => cpuBound }
      _ <- IO.println(s"CPU results: ${cpuResults.sum}")
      
      // I/O tasks (blocking)
      ioResult <- ioBound
      _ <- IO.println(s"IO result: $ioResult")
    yield ()
  
  // === Custom execution context ===
  def withCustomRuntime(): IO[Unit] =
    // cats-effect runtime configuration
    import cats.effect.unsafe.{IORuntime, IORuntimeConfig}
    
    // Default runtime ใช้ available processors
    val numThreads = Runtime.getRuntime.availableProcessors()
    println(s"Available processors: $numThreads")
    
    IO.println(s"Running on fiber scheduler with $numThreads threads")
  
  // === Cooperative scheduling ===
  def demonstrateCooperativeScheduling(): IO[Unit] =
    def countDown(n: Int): IO[Unit] =
      if n <= 0 then IO.println("Done!")
      else 
        IO.println(s"Counting: $n") >>
        IO.cede >>           // เปิดโอกาสให้ fiber อื่นรัน (yield)
        countDown(n - 1)
    
    for
      f1 <- countDown(5).start
      f2 <- countDown(5).start
      _ <- f1.join
      _ <- f2.join
    yield ()
  
  // === Fiber local state ===
  def demonstrateFiberLocal(): IO[Unit] =
    // FiberLocal - state per fiber (like ThreadLocal but for fibers)
    for
      local <- FiberLocal[IO].make[Int](0)
      
      fiber1 <- (
        local.set(1) >>
        IO.sleep(50.millis) >>
        local.get.flatMap(v => IO.println(s"Fiber 1: $v"))
      ).start
      
      fiber2 <- (
        local.set(2) >>
        IO.sleep(25.millis) >>
        local.get.flatMap(v => IO.println(s"Fiber 2: $v"))
      ).start
      
      _ <- fiber1.join
      _ <- fiber2.join
      
      mainValue <- local.get
      _ <- IO.println(s"Main: $mainValue")  // 0 (unchanged)
    yield ()
  
  def main(args: Array[String]): Unit =
    val program = for
      _ <- IO.println("=== Fibers ===")
      _ <- demonstrateFibers()
      _ <- IO.println("\n=== Cancellation ===")
      _ <- demonstrateCancellation()
      _ <- IO.println("\n=== Cooperative Scheduling ===")
      _ <- demonstrateCooperativeScheduling()
      _ <- IO.println("\n=== Fiber Local ===")
      _ <- demonstrateFiberLocal()
    yield ()
    
    program.unsafeRunSync()
```

---

## 7. Worker Pools และ Task Queues {#worker-pools}

```scala
import cats.effect.*
import cats.effect.std.*
import cats.syntax.all.*
import fs2.*
import scala.concurrent.duration.*

object WorkerPoolsDemo:
  
  // === Queue-based Worker Pool ===
  def workerPool[A, B](
    workers: Int,
    process: A => IO[B]
  ): Resource[IO, Queue[IO, A] => Stream[IO, B]] =
    Resource.eval {
      Queue.bounded[IO, Option[A]](workers * 10).map { queue =>
        (input: Queue[IO, A]) =>
          val worker = Stream.repeatEval(queue.take).unNoneTerminate.evalMap(process)
          Stream.range(0, workers)
            .map(_ => worker)
            .parJoin(workers)
      }
    }
  
  // Simple worker pool ด้วย Semaphore
  def boundedWorkerPool[A, B](
    concurrency: Int
  )(process: A => IO[B]): List[A] => IO[List[B]] = { tasks =>
    Semaphore[IO](concurrency).flatMap { sem =>
      tasks.parTraverse { task =>
        sem.permit.use(_ => process(task))
      }
    }
  }
  
  // === Async Queue ===
  def demonstrateQueue(): IO[Unit] =
    for
      // Unbounded queue
      queue <- Queue.unbounded[IO, Int]
      
      // Producer
      producer <- Stream.range(1, 101)
        .evalMap(i => queue.offer(i) >> IO.sleep(1.millis))
        .compile
        .drain
        .start
      
      // Consumer ด้วย Stream
      results <- Stream.fromQueueUnterminated(queue, limit = 100)
        .take(100)
        .fold(0)(_ + _)
        .compile
        .toList
      
      _ <- producer.join
      _ <- IO.println(s"Sum: ${results.headOption.getOrElse(0)}")  // 5050
    yield ()
  
  // === Circuit Breaker ด้วย Ref ===
  sealed trait CBState
  case object Closed   extends CBState
  case object Open     extends CBState
  case object HalfOpen extends CBState
  
  class CircuitBreaker(
    state: Ref[IO, CBState],
    failures: Ref[IO, Int],
    maxFailures: Int,
    resetTimeout: FiniteDuration
  ):
    
    def call[A](effect: IO[A]): IO[A] =
      state.get.flatMap {
        case Open =>
          IO.raiseError(new Exception("Circuit breaker is OPEN"))
        
        case HalfOpen | Closed =>
          effect.attempt.flatMap {
            case Right(value) =>
              // reset on success
              state.set(Closed) >> failures.set(0) >> IO.pure(value)
            
            case Left(error) =>
              failures.updateAndGet(_ + 1).flatMap { failCount =>
                if failCount >= maxFailures then
                  state.set(Open) >>
                  // auto-reset after timeout
                  (IO.sleep(resetTimeout) >> state.set(HalfOpen)).start.void >>
                  IO.raiseError(error)
                else
                  IO.raiseError(error)
              }
          }
      }
    
    def getState: IO[CBState] = state.get
  
  object CircuitBreaker:
    def make(maxFailures: Int, resetTimeout: FiniteDuration): IO[CircuitBreaker] =
      for
        state    <- IO.ref[CBState](Closed)
        failures <- IO.ref(0)
      yield new CircuitBreaker(state, failures, maxFailures, resetTimeout)
  
  // === Rate Limiter ===
  class RateLimiter(
    permits: Semaphore[IO],
    maxPerSecond: Int
  ):
    def acquire: IO[Unit] =
      permits.acquire >>
      IO.sleep((1000.0 / maxPerSecond).millis).start.void
    
    def withRateLimit[A](effect: IO[A]): IO[A] =
      acquire >> effect
  
  object RateLimiter:
    def make(maxPerSecond: Int): IO[RateLimiter] =
      Semaphore[IO](maxPerSecond).map(new RateLimiter(_, maxPerSecond))
  
  def demonstrateWorkerPool(): IO[Unit] =
    val process: Int => IO[Int] = n =>
      IO.sleep(50.millis) >> IO(n * n)
    
    val pooledProcess = boundedWorkerPool[Int, Int](4)(process)
    
    for
      start <- IO.realTime
      results <- pooledProcess((1 to 20).toList)
      end <- IO.realTime
      _ <- IO.println(f"Processed 20 items in ${(end - start).toMillis}ms")
      _ <- IO.println(s"Results: ${results.take(5)}...")
    yield ()
  
  def demonstrateCircuitBreaker(): IO[Unit] =
    for
      cb <- CircuitBreaker.make(maxFailures = 3, resetTimeout = 1.second)
      
      // Normal calls succeed
      r1 <- cb.call(IO.pure("success1"))
      _ <- IO.println(s"Call 1: $r1")
      
      // Simulate failures
      _ <- (1 to 3).toList.traverse { i =>
        cb.call(IO.raiseError[String](new Exception(s"Failure $i")))
          .handleErrorWith(e => IO.println(s"Error $i: ${e.getMessage}"))
      }
      
      state <- cb.getState
      _ <- IO.println(s"CB State: $state")  // Open
      
      // Call while open - should fail fast
      _ <- cb.call(IO.pure("should fail"))
        .handleErrorWith(e => IO.println(s"Rejected: ${e.getMessage}"))
    yield ()
  
  def main(args: Array[String]): Unit =
    val program = for
      _ <- IO.println("=== Worker Pool ===")
      _ <- demonstrateWorkerPool()
      _ <- IO.println("\n=== Circuit Breaker ===")
      _ <- demonstrateCircuitBreaker()
      _ <- IO.println("\n=== Queue Demo ===")
      _ <- demonstrateQueue()
    yield ()
    
    program.unsafeRunSync()
```

---

## 8. ตัวอย่าง High-Concurrency สมบูรณ์ {#high-concurrency-example}

```scala
import cats.effect.*
import cats.effect.std.*
import cats.syntax.all.*
import fs2.*
import scala.concurrent.duration.*

object HighConcurrencySystem:
  
  // === Domain ===
  case class Request(id: String, userId: String, payload: String)
  case class Response(requestId: String, result: String, processingTimeMs: Long)
  
  // === Metrics ===
  case class Metrics(
    totalRequests: Long,
    successCount: Long,
    errorCount: Long,
    totalProcessingMs: Long
  ):
    def avgProcessingMs: Double = 
      if successCount > 0 then totalProcessingMs.toDouble / successCount else 0.0
  
  // === High-throughput Request Processor ===
  class RequestProcessor(
    workers: Int,
    maxQueueSize: Int,
    rateLimit: Int  // requests per second
  ):
    
    def process(requests: Stream[IO, Request]): IO[Metrics] =
      for
        metricsRef <- IO.ref(Metrics(0, 0, 0, 0))
        queue <- Queue.bounded[IO, Request](maxQueueSize)
        rateSem <- Semaphore[IO](rateLimit)
        
        // Producer: enqueue requests with rate limiting
        producer = requests
          .evalTap(_ => rateSem.acquire)
          .evalMap(req => queue.offer(req))
          .compile
          .drain
          .onFinalizeCase {
            case _ => IO.sleep(100.millis)  // wait for workers to drain
          }
        
        // Workers: process requests concurrently
        workerStream = Stream
          .range(0, workers)
          .flatMap { workerId =>
            Stream.fromQueueUnterminated(queue, limit = 1000)
              .evalMap { req =>
                val start = System.currentTimeMillis()
                
                // Simulate processing with occasional errors
                val process = if scala.util.Random.nextDouble() < 0.05 then
                  IO.raiseError[String](new Exception("Random failure"))
                else
                  IO.sleep(10.millis) >> IO.pure(s"Processed by worker $workerId")
                
                process.attempt.flatMap { result =>
                  val duration = System.currentTimeMillis() - start
                  result match
                    case Right(r) =>
                      metricsRef.update(m => m.copy(
                        totalRequests = m.totalRequests + 1,
                        successCount = m.successCount + 1,
                        totalProcessingMs = m.totalProcessingMs + duration
                      )) >> IO.pure(Response(req.id, r, duration))
                    case Left(e) =>
                      metricsRef.update(m => m.copy(
                        totalRequests = m.totalRequests + 1,
                        errorCount = m.errorCount + 1
                      )) >> IO.pure(Response(req.id, s"Error: ${e.getMessage}", duration))
                }
              }
          }
          .parJoin(workers)
        
        // Rate limiter refresher
        rateLimiter = Stream
          .awakeEvery[IO](1.second)
          .evalMap(_ => rateSem.releaseN(rateLimit.toLong))
          .compile
          .drain
        
        // รัน producer, workers, และ rate limiter พร้อมกัน
        _ <- (
          producer,
          workerStream.compile.drain,
          rateLimiter
        ).parTupled.void
        
        metrics <- metricsRef.get
      yield metrics
  
  // === HTTP-like request handling ===
  sealed trait HttpMethod
  case object GET  extends HttpMethod
  case object POST extends HttpMethod
  case object PUT  extends HttpMethod
  
  case class HttpRequest(
    method: HttpMethod,
    path: String,
    headers: Map[String, String],
    body: Option[String]
  )
  
  case class HttpResponse(
    status: Int,
    headers: Map[String, String],
    body: String
  )
  
  type Handler = HttpRequest => IO[HttpResponse]
  type Middleware = Handler => Handler
  
  // Middleware compositions
  def loggingMiddleware: Middleware = handler => req =>
    for
      start    <- IO.realTime
      response <- handler(req)
      end      <- IO.realTime
      _        <- IO.println(
        f"${req.method} ${req.path} -> ${response.status} " +
        f"[${(end - start).toMillis}ms]"
      )
    yield response
  
  def rateLimitingMiddleware(maxRps: Int): IO[Middleware] =
    Semaphore[IO](maxRps).map { sem => handler => req =>
      sem.permit.use(_ => handler(req))
    }
  
  def authMiddleware(validTokens: Set[String]): Middleware = handler => req =>
    req.headers.get("Authorization") match
      case Some(token) if validTokens.contains(token) =>
        handler(req)
      case _ =>
        IO.pure(HttpResponse(401, Map.empty, "Unauthorized"))
  
  def circuitBreakerMiddleware(cb: CircuitBreaker): Middleware = handler => req =>
    cb.call(handler(req))
      .handleErrorWith { e =>
        IO.pure(HttpResponse(503, Map.empty, s"Service unavailable: ${e.getMessage}"))
      }
  
  // Route handlers
  def userHandler(userId: String): IO[HttpResponse] =
    IO.sleep(10.millis) >> IO.pure(
      HttpResponse(200, Map("Content-Type" -> "application/json"),
        s"""{"id": "$userId", "name": "User $userId"}""")
    )
  
  def productHandler(productId: String): IO[HttpResponse] =
    IO.sleep(5.millis) >> IO.pure(
      HttpResponse(200, Map("Content-Type" -> "application/json"),
        s"""{"id": "$productId", "price": 99.99}""")
    )
  
  def router(req: HttpRequest): IO[HttpResponse] = req.path match
    case path if path.startsWith("/users/") =>
      userHandler(path.drop("/users/".length))
    case path if path.startsWith("/products/") =>
      productHandler(path.drop("/products/".length))
    case _ =>
      IO.pure(HttpResponse(404, Map.empty, "Not Found"))
  
  def main(args: Array[String]): Unit =
    val program = for
      _ <- IO.println("=== High-Concurrency Request Processor ===")
      
      // สร้าง stream of requests
      requestStream = Stream
        .range(1, 201)
        .map { i =>
          Request(
            id = s"REQ-$i",
            userId = s"U${i % 10}",
            payload = s"Payload for request $i"
          )
        }
        .covary[IO]
      
      processor = new RequestProcessor(
        workers = 8,
        maxQueueSize = 50,
        rateLimit = 100  // 100 req/s
      )
      
      metrics <- processor.process(requestStream)
      
      _ <- IO.println(s"\n=== Metrics ===")
      _ <- IO.println(s"Total Requests: ${metrics.totalRequests}")
      _ <- IO.println(s"Success: ${metrics.successCount}")
      _ <- IO.println(s"Errors: ${metrics.errorCount}")
      _ <- IO.println(f"Avg Processing: ${metrics.avgProcessingMs}%.1fms")
      
      _ <- IO.println("\n=== Middleware Pipeline ===")
      
      rateLimitMW <- rateLimitingMiddleware(10)
      
      val tokens = Set("token123", "token456")
      
      val pipeline = loggingMiddleware
        .compose(authMiddleware(tokens))
        .compose(rateLimitMW)
      
      val handler = pipeline(router)
      
      // Test requests
      requests = List(
        HttpRequest(GET, "/users/U001", Map("Authorization" -> "token123"), None),
        HttpRequest(GET, "/products/P001", Map("Authorization" -> "token456"), None),
        HttpRequest(GET, "/unknown", Map("Authorization" -> "token123"), None),
        HttpRequest(GET, "/users/U002", Map.empty, None)  // No auth
      )
      
      responses <- requests.parTraverse(handler)
      
      _ <- responses.zip(requests).traverse { case (resp, req) =>
        IO.println(f"${req.method} ${req.path} -> ${resp.status}: ${resp.body.take(50)}")
      }
    yield ()
    
    program.unsafeRunSync()
```

---

## 9. สรุป {#summary}

### สิ่งที่ได้เรียนรู้ในบทนี้

1. **JVM Memory Model**: เข้าใจ happens-before, cache coherency, false sharing
2. **Thread Safety Primitives**: `@volatile`, `synchronized`, `AtomicReference`
3. **Lock-free Structures**: Queue, Stack ด้วย CAS operations
4. **STM**: Software Transactional Memory ด้วย cats-stm
5. **Parallel IO**: `parSequence`, `parMapN`, `parTraverseN`
6. **Fibers**: lightweight threads ด้วย cats-effect
7. **Worker Pools**: จัดการ concurrency ด้วย Semaphore และ Queue
8. **Production Patterns**: Circuit breaker, rate limiter, middleware

### Concurrency Decision Guide

```scala
// เลือกเครื่องมือตามลักษณะงาน:

// 1. Simple shared mutable state
val counter = new java.util.concurrent.atomic.AtomicLong(0)

// 2. Thread-safe collections
val map = new java.util.concurrent.ConcurrentHashMap[K, V]()

// 3. Functional concurrent state
val ref: IO[Ref[IO, A]] = IO.ref(initialValue)

// 4. Multiple concurrent updates (transactional)
// ใช้ cats-stm สำหรับ atomic multi-variable updates

// 5. Parallel processing
items.parTraverseN(parallelism)(process)

// 6. Bounded concurrency
Semaphore[IO](maxConcurrent).flatMap(sem => sem.permit.use(_ => task))

// 7. Producer-Consumer
Queue.bounded[IO, A](capacity)

// 8. Fan-out fan-in
Stream.emits(tasks).parEvalMapUnordered(n)(process)
```

---

*[← Part 73: Reactive Architecture](part-73-reactive-architecture.md) | [Part 75: Production Best Practices →](part-75-production.md)*
