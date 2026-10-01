# Part 34: fs2 Streams

## สารบัญ
1. [fs2 Overview](#fs2-overview)
2. [Stream Basics](#stream-basics)
3. [Effectful Streams](#effectful-streams)
4. [Concurrency](#concurrency)
5. [Error Handling](#error-handling)
6. [Real-World Examples](#real-world-examples)

---

## fs2 Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "co.fs2" %% "fs2-core" % "3.9.4",
  "co.fs2" %% "fs2-io"   % "3.9.4"  // for file/network I/O
)
```

### Core Types

```
Stream[F[_], O]:
  - F: effect type (IO, Task, etc.)
  - O: output element type
  - F[Unit] for side effects only

Pure streams: Stream[Pure, O] (no effects)
IO streams:   Stream[IO, O]

Operations:
  - map, flatMap, filter
  - take, drop, takeWhile
  - through: Stream[F,A] => Stream[F,B] (Pipe)
  - merge, zip, interleave
  - chunks: efficient batching
```

---

## Stream Basics

### Creating Streams

```scala
import fs2.Stream
import cats.effect.IO

// Pure streams (no effects)
val s1: Stream[Pure, Int] = Stream(1, 2, 3, 4, 5)
val s2: Stream[Pure, Int] = Stream.range(1, 100)
val s3: Stream[Pure, Int] = Stream.iterate(0)(_ + 1)  // infinite!
val s4: Stream[Pure, String] = Stream.emits(List("a", "b", "c"))

// Unfold: generate from state
val fibonacci: Stream[Pure, BigInt] =
  Stream.unfold((BigInt(0), BigInt(1))) { case (a, b) =>
    Some((a, (b, a + b)))
  }

println(fibonacci.take(10).toList)
// List(0, 1, 1, 2, 3, 5, 8, 13, 21, 34)

// Effectful streams
val ioStream: Stream[IO, Int] = Stream.eval(IO(42))
val fromIO: Stream[IO, Long] = Stream.eval(IO(System.currentTimeMillis()))
```

### Stream Operations

```scala
import fs2.Stream
import cats.effect.IO

val numbers = Stream.range(1, 20)

// Basic transforms
val evens    = numbers.filter(_ % 2 == 0)
val doubled  = numbers.map(_ * 2)
val strings  = numbers.map(_.toString)

// flatMap
val expanded = Stream(1, 2, 3).flatMap(n => Stream.range(0, n))
// 0, 0, 1, 0, 1, 2

// take/drop
val first5   = numbers.take(5)
val dropped5 = numbers.drop(5)
val while10  = numbers.takeWhile(_ <= 10)

// fold/compile
val sum: IO[Int] = numbers.compile.fold(0)(_ + _)
val lst: IO[List[Int]] = numbers.take(5).compile.toList
val cnt: IO[Long] = numbers.compile.count
val drn: IO[Unit] = numbers.map(println).compile.drain

// chunk operations
val chunked = numbers.chunkN(3)  // Stream of Chunk[Int], size 3
val unchunked = chunked.flatMap(Stream.chunk)  // back to Stream[Int]

// scan: running aggregate
val runningSum = numbers.scan(0)(_ + _)
// 0, 1, 3, 6, 10, 15, 21, 28, 36, 45, ...
```

---

## Effectful Streams

### IO Streams

```scala
import fs2.Stream
import cats.effect.{IO, Resource}
import java.io.*

// Stream of IO effects
val effectStream: Stream[IO, String] = Stream(1, 2, 3)
  .evalMap(n => IO(s"Processing $n"))

// Interleave effects with data
val withSideEffects = Stream.range(1, 6)
  .evalTap(n => IO.println(s"About to process $n"))
  .map(_ * 10)
  .evalTap(n => IO.println(s"Result: $n"))

// Resource streams (auto-close)
def readLines(path: String): Stream[IO, String] =
  Stream.resource(Resource.fromAutoCloseable(IO(new BufferedReader(new FileReader(path)))))
    .flatMap { reader =>
      Stream.unfoldEval(reader) { r =>
        IO(Option(r.readLine()).map(line => (line, r)))
      }
    }

// Write to file
def writeLines(path: String)(lines: Stream[IO, String]): IO[Unit] =
  Stream.resource(Resource.fromAutoCloseable(IO(new PrintWriter(new FileWriter(path)))))
    .flatMap { writer =>
      lines.evalMap(line => IO(writer.println(line)))
    }
    .compile.drain
```

### Pipes

```scala
import fs2.{Stream, Pipe}
import cats.effect.IO

// Pipe: transforms Stream[F, I] to Stream[F, O]
def parseInts: Pipe[IO, String, Int] = stream =>
  stream.flatMap { s =>
    s.toIntOption match
      case Some(n) => Stream.emit(n)
      case None    => Stream.empty
  }

def filterPositive: Pipe[IO, Int, Int] = _.filter(_ > 0)

def stringify[A]: Pipe[IO, A, String] = _.map(_.toString)

// Compose pipes with `through`
val pipeline = Stream("1", "abc", "3", "-2", "5")
  .through(parseInts)
  .through(filterPositive)
  .through(stringify)

val result: IO[List[String]] = pipeline.compile.toList
// List(1, 3, 5)

// Stateful pipe using scan
def runningAverage: Pipe[IO, Double, Double] = stream =>
  stream.scan((0.0, 0)) { case ((sum, count), x) =>
    (sum + x, count + 1)
  }.drop(1).map { case (sum, count) => sum / count }

// Batch pipe
def batch[A](size: Int): Pipe[IO, A, List[A]] = stream =>
  stream.chunkN(size).map(_.toList)
```

---

## Concurrency

### Concurrent Streams

```scala
import fs2.Stream
import cats.effect.{IO, Ref}
import scala.concurrent.duration.*

// Merge: interleave two streams
val slow = Stream.iterate(0)(_ + 1).covary[IO]
  .evalTap(_ => IO.sleep(200.millis)).take(5)

val fast = Stream.iterate(100)(_ + 1).covary[IO]
  .evalTap(_ => IO.sleep(50.millis)).take(10)

val merged = slow.merge(fast)
// elements from both streams as they arrive

// Concurrent flatMap (parJoin)
val concurrent = Stream(1, 2, 3).covary[IO].parEvalMap(3) { n =>
  IO.sleep(n.millis) *> IO.pure(n * 10)
}

// Queue-based producer/consumer
import fs2.concurrent.Queue

def producerConsumer: IO[Unit] =
  for
    queue    <- Queue.bounded[IO, Option[Int]](10)
    producer = Stream.range(1, 20)
      .evalMap(n => queue.offer(Some(n)))
      .onFinalize(queue.offer(None))
    consumer = Stream.fromQueueNoneTerminated(queue)
      .evalMap(n => IO.println(s"Consumed: $n"))
    _        <- producer.concurrently(consumer).compile.drain
  yield ()

// Channel (broadcast)
import fs2.concurrent.Channel

def broadcast: IO[Unit] =
  for
    ch      <- Channel.bounded[IO, String](10)
    sender  = Stream("msg1", "msg2", "msg3")
      .evalMap(ch.send)
      .onFinalize(ch.close.void)
    receiver = ch.stream.evalMap(IO.println)
    _       <- sender.concurrently(receiver.take(3)).compile.drain
  yield ()
```

---

## Error Handling

### Error Recovery

```scala
import fs2.Stream
import cats.effect.IO

// handleErrorWith: recover from errors
val risky = Stream(1, 2, 0, 4, 5).evalMap { n =>
  IO(10 / n)  // will throw on 0
}

val safe = risky.handleErrorWith { e =>
  Stream.eval(IO.println(s"Error: ${e.getMessage}")) *> Stream.empty
}

// attempt: convert errors to Either
val attempted: Stream[IO, Either[Throwable, Int]] = risky.attempt

// Last resort
val withFallback = risky.recoverWith {
  case _: ArithmeticException => Stream(0)
}

// Retry with delays
import scala.concurrent.duration.*

def retryStream[A](stream: Stream[IO, A], maxRetries: Int): Stream[IO, A] =
  stream.handleErrorWith { e =>
    if maxRetries > 0 then
      Stream.eval(IO.sleep(1.second)) >> retryStream(stream, maxRetries - 1)
    else
      Stream.raiseError[IO](e)
  }
```

---

## Real-World Examples

### File Processing Pipeline

```scala
import fs2.{Stream, Pipe}
import fs2.io.file.{Files, Path}
import cats.effect.IO

// Read CSV file and process
def processCsv(path: String): IO[Map[String, Double]] =
  Files[IO].readAll(Path(path))
    .through(fs2.text.utf8.decode)
    .through(fs2.text.lines)
    .filter(_.nonEmpty)
    .filter(!_.startsWith("#"))  // skip comments
    .drop(1)  // skip header
    .map(_.split(",").toList)
    .collect { case name :: score :: _ =>
      name.trim -> score.trim.toDouble
    }
    .compile
    .to(Map)

// Write results
def writeCsv(path: String, data: Map[String, Double]): IO[Unit] =
  Stream.emits(data.toList)
    .map { case (name, score) => s"$name,$score\n" }
    .through(fs2.text.utf8.encode)
    .through(Files[IO].writeAll(Path(path)))
    .compile
    .drain
```

### Kafka-like Consumer

```scala
import fs2.Stream
import cats.effect.{IO, Ref}
import scala.concurrent.duration.*

case class Message(topic: String, key: String, value: String, offset: Long)

// Simulated message source
def messageSource(topic: String): Stream[IO, Message] =
  Stream.iterate(0L)(_ + 1)
    .covary[IO]
    .evalMap { offset =>
      IO.sleep(100.millis) *>
      IO.pure(Message(topic, s"key-$offset", s"value-$offset", offset))
    }

// Process messages with at-least-once semantics
def processMessages(
  messages: Stream[IO, Message],
  handler: Message => IO[Unit]
): IO[Unit] =
  messages
    .evalMap { msg =>
      handler(msg)
        .handleErrorWith(e =>
          IO.println(s"Error processing ${msg.offset}: ${e.getMessage}")
        )
    }
    .compile.drain

// Usage
val consumer = processMessages(
  messageSource("orders").take(10),
  msg => IO.println(s"Processing: ${msg.value}")
)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Stream creation: emits, range, unfold, eval
- ✅ Stream operations: map, filter, flatMap, scan
- ✅ Effectful streams: IO effects, resource management
- ✅ Pipes: composable stream transformations
- ✅ Concurrency: merge, parEvalMap, Queue, Channel
- ✅ Error handling: handleErrorWith, attempt, retry
- ✅ Real-world: CSV processing, message consumers

---

*[← Part 33: Cats Library](part-33-cats.md) | [Part 35: Tapir API →](part-35-tapir.md)*
