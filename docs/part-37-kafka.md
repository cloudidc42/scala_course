# Part 37: Apache Kafka กับ Scala

## สารบัญ
1. [Kafka Overview](#kafka-overview)
2. [fs2-kafka](#fs2-kafka)
3. [Producer](#producer)
4. [Consumer](#consumer)
5. [Streams Processing](#streams-processing)
6. [Schema Registry](#schema-registry)

---

## Kafka Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "com.github.fd4s" %% "fs2-kafka"                  % "3.2.0",
  "com.github.fd4s" %% "fs2-kafka-vulcan"            % "3.2.0",  // for Avro
  "io.confluent"     % "kafka-schema-registry-client" % "7.5.1"
)
```

### Core Concepts

```
Kafka Architecture:
- Topic: named stream of records
- Partition: ordered, immutable sequence
- Producer: writes records to topics
- Consumer: reads records from topics
- Consumer Group: scales consumption
- Broker: Kafka server
- Offset: position in partition

Record = (Key, Value, Timestamp, Headers)

Delivery guarantees:
- At most once:  may lose messages
- At least once: may duplicate messages
- Exactly once:  transactional (Kafka 0.11+)
```

---

## fs2-kafka

### Basic Setup

```scala
import cats.effect.IO
import fs2.kafka.*
import org.apache.kafka.common.serialization.*

// Serializers for basic types
val stringSerializer: Serializer[IO, String] =
  Serializer.lift(IO.delay(new StringSerializer()))

val stringDeserializer: Deserializer[IO, String] =
  Deserializer.lift(IO.delay(new StringDeserializer()))

// Producer settings
val producerSettings: ProducerSettings[IO, String, String] =
  ProducerSettings[IO, String, String](
    keySerializer   = Serializer[IO, String],
    valueSerializer = Serializer[IO, String]
  )
  .withBootstrapServers("localhost:9092")
  .withAcks(Acks.All)
  .withRetries(3)
  .withProperty("enable.idempotence", "true")

// Consumer settings
val consumerSettings: ConsumerSettings[IO, String, String] =
  ConsumerSettings[IO, String, String](
    keyDeserializer   = Deserializer[IO, String],
    valueDeserializer = Deserializer[IO, String]
  )
  .withBootstrapServers("localhost:9092")
  .withGroupId("my-consumer-group")
  .withAutoOffsetReset(AutoOffsetReset.Earliest)
  .withEnableAutoCommit(false)
```

---

## Producer

### Sending Messages

```scala
import cats.effect.IO
import fs2.kafka.*
import fs2.Stream

// Simple producer
def produceMessages(
  topic: String,
  messages: List[(String, String)]
): IO[Unit] =
  KafkaProducer.stream(producerSettings)
    .flatMap { producer =>
      Stream.emits(messages).evalMap { case (key, value) =>
        val record = ProducerRecord(topic, key, value)
        producer.produce(ProducerRecords.one(record)).flatten
      }
    }
    .compile.drain

// Produce with metadata headers
def produceWithHeaders(
  topic: String,
  key: String,
  value: String,
  headers: Map[String, String]
): IO[RecordMetadata] =
  KafkaProducer.resource(producerSettings).use { producer =>
    val kafkaHeaders = Headers(
      headers.map { case (k, v) =>
        Header(k, v.getBytes("UTF-8"))
      }.toSeq*
    )
    val record = ProducerRecord(topic, key, value)
      .withHeaders(kafkaHeaders)
    producer.produce(ProducerRecords.one(record)).flatten.map(_.passthrough)
  }

// Batch producer
def batchProduce[A](
  topic: String,
  items: List[A],
  toRecord: A => ProducerRecord[String, String]
): IO[Unit] =
  KafkaProducer.stream(producerSettings)
    .flatMap { producer =>
      Stream.emits(items.map(toRecord))
        .map(r => ProducerRecords.one(r))
        .through(KafkaProducer.pipe(producerSettings))
    }
    .compile.drain

// Transactional producer
def transactionalProduce(messages: List[(String, String)]): IO[Unit] =
  TransactionalKafkaProducer
    .stream(
      TransactionalProducerSettings(
        "my-transactional-id",
        producerSettings
      )
    )
    .flatMap { producer =>
      Stream.eval {
        val records = messages.map { case (k, v) =>
          ProducerRecord("my-topic", k, v)
        }
        producer.produce(TransactionalProducerRecords(
          CommittableProducerRecords(records, CommittableOffset.empty)
        ))
      }
    }
    .compile.drain
```

---

## Consumer

### Consuming Messages

```scala
import cats.effect.IO
import fs2.kafka.*
import fs2.Stream
import scala.concurrent.duration.*

// Simple consumer
def consumeMessages(topic: String): Stream[IO, Unit] =
  KafkaConsumer.stream(consumerSettings)
    .subscribeTo(topic)
    .records
    .evalMap { committable =>
      val record = committable.record
      IO.println(s"[${record.partition}:${record.offset}] ${record.key} => ${record.value}")
        .flatMap(_ => committable.offset.commit)
    }

// Consumer with manual offset management
def consumeWithOffsets(topic: String): IO[Unit] =
  KafkaConsumer.stream(consumerSettings)
    .subscribeTo(topic)
    .records
    .mapAsync(25)(committable => processRecord(committable).as(committable.offset))
    .through(commitBatchWithin(500, 15.seconds))
    .compile.drain

def processRecord(committable: CommittableConsumerRecord[IO, String, String]): IO[Unit] =
  IO.println(s"Processing: ${committable.record.value}")

// Partitioned consumer (one stream per partition)
def partitionedConsumer(topic: String): IO[Unit] =
  KafkaConsumer.stream(consumerSettings)
    .subscribeTo(topic)
    .partitionedRecords
    .mapAsync(Int.MaxValue) { partitionStream =>
      partitionStream
        .evalMap { committable =>
          IO.println(s"Partition: ${committable.record.partition} - ${committable.record.value}")
            .as(committable.offset)
        }
        .through(commitBatchWithin(100, 5.seconds))
        .compile.drain
    }
    .compile.drain
```

---

## Streams Processing

### Event Processing Pipeline

```scala
import cats.effect.IO
import fs2.kafka.*
import fs2.Stream
import io.circe.*
import io.circe.syntax.*
import io.circe.parser.*
import io.circe.generic.auto.*

case class OrderEvent(orderId: String, userId: String, amount: Double, status: String)
case class Notification(userId: String, message: String)

// Process orders and produce notifications
def orderProcessor: IO[Unit] =
  val orderConsumer = ConsumerSettings[IO, String, String](
    keyDeserializer = Deserializer[IO, String],
    valueDeserializer = Deserializer[IO, String]
  )
  .withBootstrapServers("localhost:9092")
  .withGroupId("order-processor")
  .withAutoOffsetReset(AutoOffsetReset.Earliest)

  val notifProducer = ProducerSettings[IO, String, String](
    keySerializer = Serializer[IO, String],
    valueSerializer = Serializer[IO, String]
  )
  .withBootstrapServers("localhost:9092")

  KafkaConsumer.stream(orderConsumer)
    .subscribeTo("orders")
    .records
    .mapAsync(10) { committable =>
      // Parse order event
      decode[OrderEvent](committable.record.value) match
        case Right(event) if event.status == "completed" =>
          val notification = Notification(
            event.userId,
            s"Order ${event.orderId} completed! Amount: ${event.amount}"
          )
          IO.pure((Some(notification), committable.offset))
        case _ =>
          IO.pure((None, committable.offset))
    }
    .flatMap { case (maybeNotif, offset) =>
      maybeNotif match
        case Some(notif) =>
          Stream.eval(IO.pure(
            ProducerRecord("notifications", notif.userId, notif.asJson.noSpaces)
          ).map(r => (r, offset)))
        case None =>
          Stream.eval(IO.pure((None, offset))).collect { case (Some(r), o) => (r, o) }
    }
    .compile.drain
```

### Aggregations

```scala
import cats.effect.{IO, Ref}
import fs2.kafka.*
import fs2.Stream

// Windowed aggregation
def windowedCount(windowSizeMs: Long = 60000): IO[Unit] =
  Ref.of[IO, Map[String, Int]](Map.empty).flatMap { counts =>
    KafkaConsumer.stream(consumerSettings)
      .subscribeTo("events")
      .records
      .groupWithin(100, 5.seconds)  // batch every 5 seconds or 100 records
      .evalMap { chunk =>
        val updates = chunk.toList
          .groupBy(_.record.key)
          .view.mapValues(_.size)
          .toMap
        counts.update(m => m ++ updates.map { case (k, v) =>
          k -> (m.getOrElse(k, 0) + v)
        }) *> counts.get.flatMap(c => IO.println(s"Counts: $c"))
      }
      .compile.drain
  }
```

---

## Schema Registry

### Avro with Vulcan

```scala
import cats.effect.IO
import fs2.kafka.*
import fs2.kafka.vulcan.*
import vulcan.Codec

// Define Avro codec
case class Event(
  id: String,
  eventType: String,
  data: Map[String, String]
)

object Event:
  given Codec[Event] = Codec.record[Event](
    name = "Event",
    namespace = "com.example"
  ) { field =>
    (
      field("id", _.id),
      field("eventType", _.eventType),
      field("data", _.data)
    ).mapN(Event.apply)
  }

// Setup with Schema Registry
def avroProducer: IO[Unit] =
  val avroSettings = AvroSettings[IO](
    SchemaRegistryClientSettings[IO]("http://localhost:8081")
  )

  val producerSettings = ProducerSettings[IO, String, Event](
    keySerializer = Serializer[IO, String],
    valueSerializer = avroSerializer[IO, Event](avroSettings).forValue
  )
  .withBootstrapServers("localhost:9092")

  KafkaProducer.stream(producerSettings)
    .flatMap { producer =>
      val record = ProducerRecord(
        "events",
        "key1",
        Event("evt-1", "user_login", Map("userId" -> "123"))
      )
      Stream.eval(producer.produce(ProducerRecords.one(record)).flatten)
    }
    .compile.drain
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Kafka concepts: topics, partitions, offsets
- ✅ fs2-kafka producer: simple, headers, transactional
- ✅ fs2-kafka consumer: commit offsets, partitioned
- ✅ Event processing pipelines
- ✅ Windowed aggregations
- ✅ Schema Registry กับ Avro/Vulcan

---

*[← Part 36: Redis](part-36-redis.md) | [Part 38: GraphQL →](part-38-graphql.md)*
