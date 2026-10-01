# Part 22: Akka Streams

## สารบัญ
1. [Stream Fundamentals](#stream-fundamentals)
2. [Sources, Flows, Sinks](#sources-flows-sinks)
3. [Stream Operations](#stream-operations)
4. [Error Handling](#error-handling)
5. [Practical Examples](#practical-examples)

---

## Stream Fundamentals

### แนวคิด Akka Streams

```
Akka Streams:
- Source[Out, Mat]  - produces elements
- Flow[In, Out, Mat] - transforms elements
- Sink[In, Mat]     - consumes elements
- RunnableGraph    - Source ~> Flow ~> Sink (complete graph)

Backpressure: consumer controls speed of producer
ทำให้ไม่มี OutOfMemoryError จาก fast producer + slow consumer
```

### SBT Dependencies

```scala
libraryDependencies += "com.typesafe.akka" %% "akka-stream" % "2.8.0"
```

---

## Sources, Flows, Sinks

### Basic Stream

```scala
import akka.actor.typed.ActorSystem
import akka.actor.typed.scaladsl.Behaviors
import akka.stream.*
import akka.stream.scaladsl.*
import scala.concurrent.{Future, Await}
import scala.concurrent.duration.*

given system: ActorSystem[?] = ActorSystem(Behaviors.empty, "stream-system")
given ec: scala.concurrent.ExecutionContext = system.executionContext

// Source: produces elements
val source: Source[Int, NotUsed] = Source(1 to 10)

// Flow: transforms elements
val flow: Flow[Int, Int, NotUsed] = Flow[Int].map(_ * 2)

// Sink: consumes elements
val sink: Sink[Int, Future[Done]] = Sink.foreach[Int](println)

// Run the stream
val runnable: RunnableGraph[Future[Done]] = source.via(flow).to(sink)
val done: Future[Done] = runnable.run()

Await.ready(done, 10.seconds)

// Shorter syntax
val result: Future[Done] =
  Source(1 to 10)
    .map(_ * 2)
    .filter(_ > 10)
    .runForeach(println)

Await.ready(result, 10.seconds)
// 12 14 16 18 20
```

### Sources

```scala
// Various Source types

// From collection
val fromList = Source(List(1, 2, 3, 4, 5))

// From single value
val single = Source.single("hello")

// Repeat value
val repeated = Source.repeat(42).take(5)

// Tick (periodic)
import scala.concurrent.duration.*
val ticking = Source.tick(1.second, 1.second, "tick").take(5)

// Future
val fromFuture = Source.future(Future.successful(42))

// Unfold (like LazyList.unfold)
val fibonacci = Source.unfold((0, 1)) { case (a, b) =>
  Some(((b, a + b), a))
}

val first10Fibs = fibonacci.take(10).runWith(Sink.seq)
println(Await.result(first10Fibs, 5.seconds))
// Vector(0, 1, 1, 2, 3, 5, 8, 13, 21, 34)
```

### Sinks

```scala
// Various Sink types

// Foreach
val foreachSink = Sink.foreach[Int](println)

// Collect into sequence
val seqSink: Sink[Int, Future[Seq[Int]]] = Sink.seq[Int]

// Fold (reduce to single value)
val sumSink: Sink[Int, Future[Int]] = Sink.fold(0)(_ + _)

// Head (first element)
val headSink: Sink[Int, Future[Int]] = Sink.head[Int]

// Last element
val lastSink: Sink[Int, Future[Int]] = Sink.last[Int]

// Count elements
val countSink: Sink[Any, Future[Int]] = Sink.fold(0)((acc, _) => acc + 1)

// Ignore all elements
val ignoreSink: Sink[Any, Future[Done]] = Sink.ignore

// Usage
val sum = Source(1 to 100).runWith(sumSink)
println(Await.result(sum, 5.seconds))  // 5050
```

---

## Stream Operations

### Transformations

```scala
val numbers = Source(1 to 20)

// map
numbers.map(_ * 2)

// filter
numbers.filter(_ % 2 == 0)

// collect (partial function)
numbers.collect { case n if n % 3 == 0 => s"Multiple of 3: $n" }

// flatMapConcat: expand each element to sub-stream (sequential)
numbers.take(5).flatMapConcat(n => Source(1 to n))
  .runWith(Sink.seq)
// Vector(1, 1, 2, 1, 2, 3, 1, 2, 3, 4, 1, 2, 3, 4, 5)

// flatMapMerge: parallel expansion
numbers.take(3).flatMapMerge(2, n => Source(1 to n).map(n -> _))

// grouped: batch elements
numbers.grouped(5).runWith(Sink.seq)
// Vector(Vector(1,2,3,4,5), Vector(6,7,8,9,10), ...)

// sliding: rolling window
numbers.sliding(3, 1).runWith(Sink.seq)
// Vector(Vector(1,2,3), Vector(2,3,4), Vector(3,4,5), ...)

// throttle: rate limiting
numbers.throttle(5, 1.second)
  .runForeach(n => println(s"$n at ${System.currentTimeMillis()}"))
```

### Combining Streams

```scala
// merge: combine multiple sources
val src1 = Source(List(1, 3, 5))
val src2 = Source(List(2, 4, 6))
val merged = Source.combine(src1, src2)(Merge(_))
// Order not guaranteed with Merge

// concat: append streams sequentially
val concatenated = src1.concat(src2)
// List(1, 3, 5, 2, 4, 6)

// zip: pair elements
val zipped = src1.zip(src2)
val result = Await.result(zipped.runWith(Sink.seq), 5.seconds)
println(result)  // Vector((1,2),(3,4),(5,6))

// zipWith: combine with function
val summed = src1.zipWith(src2)(_ + _)
println(Await.result(summed.runWith(Sink.seq), 5.seconds))
// Vector(3, 7, 11)

// broadcast: split one source to multiple sinks
val broadcast = RunnableGraph.fromGraph(GraphDSL.createGraph(
  Sink.seq[Int],
  Sink.seq[Int]
) { implicit builder => (sumSink, productSink) =>
  import GraphDSL.Implicits.*
  val bcast = builder.add(Broadcast[Int](2))
  Source(1 to 5) ~> bcast.in
  bcast.out(0) ~> Flow[Int].filter(_ % 2 == 0) ~> sumSink
  bcast.out(1) ~> Flow[Int].filter(_ % 2 != 0) ~> productSink
  ClosedShape
})
```

---

## Error Handling

```scala
// recoverWith: replace failed stream with another
val withRecovery = Source(List(1, 2, 0, 4, 5))
  .map(n => 100 / n)
  .recover { case _: ArithmeticException => -1 }
  .runWith(Sink.seq)

println(Await.result(withRecovery, 5.seconds))
// Vector(100, 50, -1, 25, 20)

// restart: restart source on failure
import akka.stream.scaladsl.RestartSource
import akka.stream.RestartSettings

var counter = 0
val unreliable = RestartSource.withBackoff(
  RestartSettings(
    minBackoff = 100.millis,
    maxBackoff = 1.second,
    randomFactor = 0.2
  )
) { () =>
  counter += 1
  Source(List(1, 2, 3))
    .map { n =>
      if counter < 3 && n == 3 then throw new RuntimeException("Oops!")
      else n
    }
}
```

---

## Practical Examples

### File Processing

```scala
import akka.stream.scaladsl.*
import akka.util.ByteString
import java.nio.file.Paths

// Read file line by line
val filePath = Paths.get("data.csv")

val csvProcessing = FileIO.fromPath(filePath)
  .via(Framing.delimiter(ByteString("\n"), maximumFrameLength = 1024, allowTruncation = true))
  .map(_.utf8String.trim)
  .filter(_.nonEmpty)
  .filter(!_.startsWith("#"))  // skip comments
  .map(_.split(","))
  .map(fields => Map(
    "name"  -> fields(0),
    "age"   -> fields(1),
    "score" -> fields(2)
  ))
  .runWith(Sink.seq)
```

### HTTP Client Streaming

```scala
// Process large HTTP responses as streams
// (requires akka-http dependency)

// Conceptual example
import akka.http.scaladsl.Http
import akka.http.scaladsl.model.*

def downloadAndProcess(url: String): Future[Done] =
  Http().singleRequest(HttpRequest(uri = url)).flatMap {
    case HttpResponse(StatusCodes.OK, _, entity, _) =>
      entity.dataBytes
        .via(Framing.delimiter(ByteString("\n"), 1024))
        .map(_.utf8String)
        .filter(_.nonEmpty)
        .map(line => processLine(line))
        .runForeach(println)
    case resp =>
      Future.failed(new RuntimeException(s"HTTP ${resp.status}"))
  }

def processLine(line: String): String = line.toUpperCase
```

### Kafka Consumer (Alpakka)

```scala
// Using Alpakka Kafka connector
// libraryDependencies += "com.typesafe.akka" %% "akka-stream-kafka" % "4.0.0"

import akka.kafka.*
import akka.kafka.scaladsl.*
import org.apache.kafka.clients.consumer.ConsumerConfig
import org.apache.kafka.common.serialization.StringDeserializer

val consumerSettings = ConsumerSettings(system, new StringDeserializer, new StringDeserializer)
  .withBootstrapServers("localhost:9092")
  .withGroupId("my-group")
  .withProperty(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")

val kafkaSource = Consumer
  .plainSource(consumerSettings, Subscriptions.topics("my-topic"))

kafkaSource
  .map(record => record.value())
  .filter(_.nonEmpty)
  .map(json => parseJson(json))
  .grouped(100)
  .mapAsync(4)(batch => saveToDB(batch))
  .runWith(Sink.ignore)

def parseJson(s: String): Map[String, String] = Map()
def saveToDB(batch: Seq[Map[String, String]]): Future[Done] =
  Future.successful(Done)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Stream concepts: Source, Flow, Sink, backpressure
- ✅ Creating Sources: collection, tick, unfold
- ✅ Stream operations: map, filter, flatMap, grouped, sliding
- ✅ Combining streams: merge, concat, zip, broadcast
- ✅ Error handling: recover, restart
- ✅ Practical: file processing, HTTP, Kafka

---

*[← Part 21: Akka Actors](part-21-akka-actors.md) | [Part 23: Testing in Scala →](part-23-testing.md)*
