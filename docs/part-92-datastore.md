# ส่วนที่ 92: Advanced Database Operations

## สารบัญ

1. [NoSQL กับ MongoDB (Mongo4Cats)](#nosql-กับ-mongodb-mongo4cats)
2. [Cassandra กับ Phantom DSL](#cassandra-กับ-phantom-dsl)
3. [Elasticsearch กับ elastic4s](#elasticsearch-กับ-elastic4s)
4. [Neo4j Graph Database](#neo4j-graph-database)
5. [InfluxDB สำหรับ Time Series](#influxdb-สำหรับ-time-series)
6. [เกณฑ์การเลือก Database](#เกณฑ์การเลือก-database)
7. [Polyglot Persistence](#polyglot-persistence)
8. [สรุป](#สรุป)

---

## NoSQL กับ MongoDB (Mongo4Cats)

MongoDB เป็น document database ที่เหมาะกับข้อมูลที่มีโครงสร้างยืดหยุ่น Mongo4Cats ให้ type-safe API บน Cats Effect

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "io.github.kirill5k" %% "mongo4cats-core"    % "0.7.8",
  "io.github.kirill5k" %% "mongo4cats-circe"   % "0.7.8",
  "io.github.kirill5k" %% "mongo4cats-embedded" % "0.7.8" % Test
)
```

### Document Model

```scala
import mongo4cats.bson.ObjectId
import mongo4cats.circe.*
import io.circe.generic.auto.*
import java.time.Instant

case class Article(
  _id: ObjectId,
  title: String,
  content: String,
  author: String,
  tags: List[String],
  viewCount: Int,
  publishedAt: Instant,
  updatedAt: Instant
)

case class Comment(
  _id: ObjectId,
  articleId: ObjectId,
  author: String,
  body: String,
  createdAt: Instant,
  likes: Int
)

// MongoDB Repository
import cats.effect.IO
import mongo4cats.client.MongoClient
import mongo4cats.collection.MongoCollection
import mongo4cats.operations.{Filter, Sort, Update}

class ArticleRepository(collection: MongoCollection[IO, Article]):

  def insert(article: Article): IO[Unit] =
    collection.insertOne(article).void

  def findById(id: ObjectId): IO[Option[Article]] =
    collection.find(Filter.eq("_id", id)).first

  def findByAuthor(author: String): IO[List[Article]] =
    collection
      .find(Filter.eq("author", author))
      .sort(Sort.desc("publishedAt"))
      .all

  def findByTags(tags: List[String]): IO[List[Article]] =
    collection
      .find(Filter.in("tags", tags))
      .all

  def findPopular(minViews: Int, limit: Int): IO[List[Article]] =
    collection
      .find(Filter.gte("viewCount", minViews))
      .sort(Sort.desc("viewCount"))
      .limit(limit)
      .all

  def incrementViewCount(id: ObjectId): IO[Unit] =
    collection
      .updateOne(
        Filter.eq("_id", id),
        Update.inc("viewCount", 1).set("updatedAt", Instant.now())
      )
      .void

  def deleteById(id: ObjectId): IO[Boolean] =
    collection.deleteOne(Filter.eq("_id", id)).map(_.wasAcknowledged)

  def fullTextSearch(query: String): IO[List[Article]] =
    collection
      .find(Filter.text(query))
      .all

  def aggregate(pipeline: List[org.bson.Document]): IO[List[Article]] =
    collection.aggregate[Article](pipeline).all
```

### Aggregation Pipeline

```scala
import org.bson.Document
import mongo4cats.operations.Aggregate

class ArticleAnalytics(collection: MongoCollection[IO, Document]):

  def topAuthors(limit: Int): IO[List[Document]] =
    collection.aggregate[Document](
      List(
        Aggregate.group("$author", "totalArticles" -> Aggregate.sum(1), "totalViews" -> Aggregate.sum("$viewCount")),
        Aggregate.sort(Sort.desc("totalViews")),
        Aggregate.limit(limit)
      )
    ).all

  def viewsByDay: IO[List[Document]] =
    collection.aggregate[Document](
      List(
        Aggregate.project(Map(
          "day"       -> Map("$dateToString" -> Map("format" -> "%Y-%m-%d", "date" -> "$publishedAt")),
          "viewCount" -> 1
        )),
        Aggregate.group("$day", "totalViews" -> Aggregate.sum("$viewCount")),
        Aggregate.sort(Sort.asc("_id"))
      )
    ).all

  def tagCloud: IO[Map[String, Int]] =
    collection.aggregate[Document](
      List(
        Aggregate.unwind("$tags"),
        Aggregate.group("$tags", "count" -> Aggregate.sum(1)),
        Aggregate.sort(Sort.desc("count"))
      )
    ).all.map: docs =>
      docs.map(d => d.getString("_id") -> d.getInteger("count").toInt).toMap
```

### Main Setup

```scala
import cats.effect.*
import mongo4cats.client.MongoClient

object MongoApp extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    MongoClient.fromConnectionString[IO]("mongodb://localhost:27017").use: client =>
      for
        db         <- client.getDatabase("blog")
        collection <- db.getCollectionWithCodec[Article]("articles")
        repo        = ArticleRepository(collection)
        article     = Article(
          _id         = ObjectId(),
          title       = "Introduction to Scala",
          content     = "Scala is a powerful language...",
          author      = "Alice",
          tags        = List("scala", "programming", "functional"),
          viewCount   = 0,
          publishedAt = Instant.now(),
          updatedAt   = Instant.now()
        )
        _  <- repo.insert(article)
        _  <- repo.incrementViewCount(article._id)
        a  <- repo.findById(article._id)
        _  <- IO.println(s"Found: ${a.map(_.title)}")
      yield ExitCode.Success
```

---

## Cassandra กับ Phantom DSL

Apache Cassandra เหมาะกับ write-heavy workloads ที่ต้องการ high availability และ linear scalability

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.outworkers" %% "phantom-dsl"    % "2.59.0",
  "com.outworkers" %% "phantom-connectors" % "2.59.0"
)
```

### Table Definitions

```scala
import com.outworkers.phantom.dsl.*
import scala.concurrent.Future

// Case class
case class TimelineEvent(
  userId: UUID,
  eventTime: DateTime,
  eventType: String,
  payload: String
)

// Phantom Table
abstract class TimelineEvents extends Table[TimelineEvents, TimelineEvent]:
  override def tableName = "timeline_events"

  object userId extends UUIDColumn with PartitionKey
  object eventTime extends DateTimeColumn with ClusteringOrder with Descending
  object eventType extends StringColumn
  object payload extends StringColumn

  def getByUser(userId: UUID): Future[List[TimelineEvent]] =
    select.where(_.userId eqs userId).fetch()

  def getByUserSince(userId: UUID, since: DateTime): Future[List[TimelineEvent]] =
    select
      .where(_.userId eqs userId)
      .and(_.eventTime gte since)
      .fetch()

  def insertEvent(event: TimelineEvent): Future[ResultSet] =
    insert
      .value(_.userId, event.userId)
      .value(_.eventTime, event.eventTime)
      .value(_.eventType, event.eventType)
      .value(_.payload, event.payload)
      .ttl(90.days.toSeconds.toInt) // Auto-expire after 90 days
      .future()

// Connector
object CassandraConnector extends ContactPoint:
  val hosts = Seq("127.0.0.1")
  val keySpace = "myapp"
  lazy val connector = ContactPoint.local.noHeartbeat().keySpace(
    KeySpace(keySpace).ifNotExists().`with`(
      replication eqs SimpleStrategy.replication_factor(1)
    )
  )

// Database
class CassandraDatabase(override val connector: CassandraConnection)
    extends Database[CassandraDatabase](connector):
  object timelineEvents extends TimelineEvents with Connector

object CassandraDatabase extends CassandraDatabase(CassandraConnector.connector)
```

### Wide Row Pattern

```scala
// Wide rows: partition by user, cluster by time
// Optimized for "get all events for user X in time range"

abstract class SensorReadings extends Table[SensorReadings, SensorReading]:
  object sensorId extends UUIDColumn with PartitionKey
  object bucket extends IntColumn with PartitionKey       // e.g. YYYYMM for time bucketing
  object readingTime extends DateTimeColumn with ClusteringOrder with Descending
  object value extends DoubleColumn
  object unit extends StringColumn

  def getReadings(sensorId: UUID, bucket: Int, limit: Int): Future[List[SensorReading]] =
    select
      .where(_.sensorId eqs sensorId)
      .and(_.bucket eqs bucket)
      .limit(limit)
      .fetch()

  def getReadingsInRange(
    sensorId: UUID,
    bucket: Int,
    from: DateTime,
    to: DateTime
  ): Future[List[SensorReading]] =
    select
      .where(_.sensorId eqs sensorId)
      .and(_.bucket eqs bucket)
      .and(_.readingTime gte from)
      .and(_.readingTime lte to)
      .fetch()
```

---

## Elasticsearch กับ elastic4s

Elasticsearch เหมาะกับ full-text search, log analytics, และ complex queries

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.sksamuel.elastic4s" %% "elastic4s-client-esjava" % "8.11.5",
  "com.sksamuel.elastic4s" %% "elastic4s-effect-cats"   % "8.11.5",
  "com.sksamuel.elastic4s" %% "elastic4s-circe"         % "8.11.5"
)
```

### Index and Document Operations

```scala
import com.sksamuel.elastic4s.ElasticClient
import com.sksamuel.elastic4s.ElasticDsl.*
import com.sksamuel.elastic4s.cats.effect.instances.*
import com.sksamuel.elastic4s.circe.*
import cats.effect.IO
import io.circe.generic.auto.*

case class Product(
  id: String,
  name: String,
  description: String,
  category: String,
  price: Double,
  tags: List[String],
  inStock: Boolean
)

class ProductSearchService(client: ElasticClient):

  def createIndex: IO[Unit] =
    client.execute:
      createIndex("products").mapping(
        properties(
          textField("name").analyzer("standard"),
          textField("description").analyzer("standard"),
          keywordField("category"),
          doubleField("price"),
          keywordField("tags"),
          booleanField("inStock")
        )
      )
    .map(_ => ())

  def indexProduct(product: Product): IO[Unit] =
    client.execute:
      indexInto("products").id(product.id).doc(product)
    .map(_ => ())

  def bulkIndex(products: List[Product]): IO[Unit] =
    client.execute:
      bulk(
        products.map(p => indexInto("products").id(p.id).doc(p))*
      )
    .map(_ => ())

  def searchByName(query: String): IO[List[Product]] =
    client.execute:
      search("products").query(matchQuery("name", query))
    .map(_.result.to[Product].toList)

  def fullTextSearch(query: String): IO[List[Product]] =
    client.execute:
      search("products").query(
        multiMatchQuery(query)
          .fields("name^3", "description", "tags^2") // boost name and tags
      )
    .map(_.result.to[Product].toList)

  def filterByCategory(category: String, maxPrice: Double): IO[List[Product]] =
    client.execute:
      search("products").query(
        boolQuery()
          .must(termQuery("category", category))
          .filter(
            rangeQuery("price").lte(maxPrice),
            termQuery("inStock", true)
          )
      ).sortByFieldAsc("price")
    .map(_.result.to[Product].toList)

  def suggest(prefix: String): IO[List[String]] =
    client.execute:
      search("products").query(
        matchPhrasePrefixQuery("name", prefix)
      ).size(10)
    .map(_.result.hits.hits.flatMap(_.sourceAsMap.get("name").map(_.toString)).toList)

  def aggregateByCategory: IO[Map[String, Long]] =
    client.execute:
      search("products")
        .query(matchAllQuery())
        .aggs(termsAgg("by_category", "category").size(100))
        .size(0)
    .map: response =>
      response.result.aggs
        .terms("by_category")
        .buckets
        .map(b => b.key -> b.docCount)
        .toMap

  def deleteProduct(id: String): IO[Unit] =
    client.execute:
      deleteById("products", id)
    .map(_ => ())
```

### Advanced Search Features

```scala
  def complexSearch(
    query: Option[String],
    categories: List[String],
    minPrice: Option[Double],
    maxPrice: Option[Double],
    inStockOnly: Boolean,
    page: Int,
    size: Int
  ): IO[SearchResult[Product]] =
    val from = page * size
    val bq   = boolQuery()

    // Full text query
    query.foreach: q =>
      bq.must(multiMatchQuery(q).fields("name^3", "description"))

    // Category filter
    if categories.nonEmpty then
      bq.filter(termsQuery("category", categories))

    // Price range
    val priceRange = rangeQuery("price")
    minPrice.foreach(p => priceRange.gte(p))
    maxPrice.foreach(p => priceRange.lte(p))
    if minPrice.isDefined || maxPrice.isDefined then
      bq.filter(priceRange)

    // Stock filter
    if inStockOnly then
      bq.filter(termQuery("inStock", true))

    client.execute:
      search("products")
        .query(if query.isDefined || categories.nonEmpty then bq else matchAllQuery())
        .from(from)
        .size(size)
        .highlighting(highlight("name").preTags("<em>").postTags("</em>"))
        .aggs(
          termsAgg("categories", "category"),
          statsAgg("priceStats", "price")
        )
    .map: response =>
      SearchResult(
        items     = response.result.to[Product].toList,
        total     = response.result.totalHits,
        page      = page,
        pageSize  = size
      )

case class SearchResult[A](items: List[A], total: Long, page: Int, pageSize: Int):
  def totalPages: Long = Math.ceil(total.toDouble / pageSize).toLong
```

---

## Neo4j Graph Database

Neo4j เหมาะกับข้อมูลที่มีความสัมพันธ์ซับซ้อน เช่น social networks, recommendation engines, fraud detection

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.neo4j.driver" % "neo4j-java-driver" % "5.16.0"
)
```

### Graph Model and Queries

```scala
import org.neo4j.driver.*
import org.neo4j.driver.types.Node
import cats.effect.IO
import scala.jdk.CollectionConverters.*

case class Person(id: String, name: String, age: Int)
case class Movie(id: String, title: String, year: Int, genre: String)
case class ActedIn(role: String, salary: Option[Int])
case class Knows(since: Int, closeness: Int)

class Neo4jRepository(driver: Driver):

  private def withSession[A](f: Session => A): IO[A] =
    IO.delay:
      val session = driver.session()
      try f(session)
      finally session.close()

  def createPerson(person: Person): IO[Unit] =
    withSession: session =>
      session.run(
        "MERGE (p:Person {id: $id}) SET p.name = $name, p.age = $age",
        Values.parameters("id", person.id, "name", person.name, "age", person.age)
      )
      ()

  def createMovie(movie: Movie): IO[Unit] =
    withSession: session =>
      session.run(
        "MERGE (m:Movie {id: $id}) SET m.title = $title, m.year = $year, m.genre = $genre",
        Values.parameters("id", movie.id, "title", movie.title, "year", movie.year, "genre", movie.genre)
      )
      ()

  def actedIn(personId: String, movieId: String, role: String): IO[Unit] =
    withSession: session =>
      session.run(
        """MATCH (p:Person {id: $personId}), (m:Movie {id: $movieId})
           MERGE (p)-[r:ACTED_IN {role: $role}]->(m)""",
        Values.parameters("personId", personId, "movieId", movieId, "role", role)
      )
      ()

  def knows(personId1: String, personId2: String, since: Int): IO[Unit] =
    withSession: session =>
      session.run(
        """MATCH (p1:Person {id: $id1}), (p2:Person {id: $id2})
           MERGE (p1)-[:KNOWS {since: $since}]->(p2)""",
        Values.parameters("id1", personId1, "id2", personId2, "since", since)
      )
      ()

  // Find common friends (degree-2 connections)
  def commonFriends(person1: String, person2: String): IO[List[Person]] =
    withSession: session =>
      val result = session.run(
        """MATCH (p1:Person {id: $id1})-[:KNOWS]->(common)<-[:KNOWS]-(p2:Person {id: $id2})
           RETURN DISTINCT common""",
        Values.parameters("id1", person1, "id2", person2)
      )
      result.list().asScala.toList.map: record =>
        val node = record.get("common").asNode()
        Person(node.get("id").asString, node.get("name").asString, node.get("age").asInt)

  // Shortest path between two people
  def shortestPath(from: String, to: String): IO[List[String]] =
    withSession: session =>
      val result = session.run(
        """MATCH path = shortestPath(
             (start:Person {id: $from})-[:KNOWS*]-(end:Person {id: $to})
           )
           RETURN [n in nodes(path) | n.name] AS names""",
        Values.parameters("from", from, "to", to)
      )
      result.list().asScala.headOption
        .map(_.get("names").asList().asScala.map(_.toString).toList)
        .getOrElse(List.empty)

  // Recommendation: people you might know
  def recommendFriends(personId: String, limit: Int): IO[List[(Person, Int)]] =
    withSession: session =>
      val result = session.run(
        """MATCH (me:Person {id: $id})-[:KNOWS]->(friend)-[:KNOWS]->(foaf)
           WHERE NOT (me)-[:KNOWS]->(foaf) AND me <> foaf
           WITH foaf, count(*) AS mutualFriends
           ORDER BY mutualFriends DESC
           LIMIT $limit
           RETURN foaf, mutualFriends""",
        Values.parameters("id", personId, "limit", limit.toLong)
      )
      result.list().asScala.toList.map: record =>
        val node    = record.get("foaf").asNode()
        val person  = Person(node.get("id").asString, node.get("name").asString, node.get("age").asInt)
        val mutual  = record.get("mutualFriends").asInt
        (person, mutual)

  // Collaborative filtering recommendation
  def recommendMovies(personId: String, limit: Int): IO[List[Movie]] =
    withSession: session =>
      val result = session.run(
        """MATCH (me:Person {id: $id})-[:ACTED_IN|WATCHED]->(movie)<-[:WATCHED]-(similar)
           MATCH (similar)-[:WATCHED]->(rec)
           WHERE NOT (me)-[:WATCHED]->(rec)
           WITH rec, count(*) AS score
           ORDER BY score DESC
           LIMIT $limit
           RETURN rec""",
        Values.parameters("id", personId, "limit", limit.toLong)
      )
      result.list().asScala.toList.map: record =>
        val node = record.get("rec").asNode()
        Movie(
          node.get("id").asString,
          node.get("title").asString,
          node.get("year").asInt,
          node.get("genre").asString
        )
```

---

## InfluxDB สำหรับ Time Series

InfluxDB เหมาะกับ time series data เช่น metrics, logs, sensor data

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "com.influxdb" % "influxdb-client-java" % "7.0.0"
)
```

### Writing and Querying Time Series

```scala
import com.influxdb.client.*
import com.influxdb.client.domain.*
import com.influxdb.client.write.Point
import com.influxdb.query.FluxTable
import cats.effect.IO
import java.time.Instant
import scala.jdk.CollectionConverters.*

case class MetricPoint(
  measurement: String,
  tags: Map[String, String],
  fields: Map[String, Double],
  timestamp: Instant
)

case class ServerMetrics(
  host: String,
  cpuPercent: Double,
  memoryMB: Double,
  networkBytesIn: Long,
  networkBytesOut: Long,
  timestamp: Instant
)

class InfluxRepository(
  client: InfluxDBClient,
  org: String,
  bucket: String
):
  private val writeApi = client.getWriteApiBlocking
  private val queryApi = client.getQueryApi

  def writeMetrics(metrics: ServerMetrics): IO[Unit] =
    IO.delay:
      val point = Point
        .measurement("server_metrics")
        .addTag("host", metrics.host)
        .addField("cpu_percent", metrics.cpuPercent)
        .addField("memory_mb", metrics.memoryMB)
        .addField("network_bytes_in", metrics.networkBytesIn.toDouble)
        .addField("network_bytes_out", metrics.networkBytesOut.toDouble)
        .time(metrics.timestamp, WritePrecision.MS)
      writeApi.writePoint(bucket, org, point)

  def batchWrite(points: List[MetricPoint]): IO[Unit] =
    IO.delay:
      val influxPoints = points.map: mp =>
        val p = Point.measurement(mp.measurement).time(mp.timestamp, WritePrecision.MS)
        mp.tags.foreach((k, v) => p.addTag(k, v))
        mp.fields.foreach((k, v) => p.addField(k, v))
        p
      writeApi.writePoints(bucket, org, influxPoints.asJava)

  def queryCpuAverage(host: String, duration: String = "1h"): IO[List[(Instant, Double)]] =
    IO.delay:
      val flux =
        s"""from(bucket: "$bucket")
           |  |> range(start: -$duration)
           |  |> filter(fn: (r) => r._measurement == "server_metrics")
           |  |> filter(fn: (r) => r._field == "cpu_percent")
           |  |> filter(fn: (r) => r.host == "$host")
           |  |> aggregateWindow(every: 5m, fn: mean)
           |  |> yield(name: "mean")""".stripMargin

      queryApi.query(flux, org).asScala.toList
        .flatMap(_.getRecords.asScala)
        .map: record =>
          (record.getTime, record.getValue.asInstanceOf[Double])

  def anomalyDetection(host: String, windowMinutes: Int = 60): IO[List[Instant]] =
    IO.delay:
      val flux =
        s"""from(bucket: "$bucket")
           |  |> range(start: -24h)
           |  |> filter(fn: (r) => r._measurement == "server_metrics" and r.host == "$host")
           |  |> filter(fn: (r) => r._field == "cpu_percent")
           |  |> movingAverage(n: ${windowMinutes})
           |  |> map(fn: (r) => ({ r with isAnomaly: r._value > 90.0 }))
           |  |> filter(fn: (r) => r.isAnomaly == true)""".stripMargin

      queryApi.query(flux, org).asScala.toList
        .flatMap(_.getRecords.asScala)
        .map(_.getTime)

  def dashboardStats(duration: String = "24h"): IO[Map[String, Double]] =
    IO.delay:
      val flux =
        s"""from(bucket: "$bucket")
           |  |> range(start: -$duration)
           |  |> filter(fn: (r) => r._measurement == "server_metrics")
           |  |> group(columns: ["_field"])
           |  |> mean()""".stripMargin

      queryApi.query(flux, org).asScala.toList
        .flatMap(_.getRecords.asScala)
        .map: record =>
          record.getField -> record.getValue.asInstanceOf[Double]
        .toMap
```

### Retention Policy

```scala
  def createBucketWithRetention(name: String, retentionDays: Int): IO[Unit] =
    IO.delay:
      val bucketsApi    = client.getBucketsApi
      val orgId         = client.getOrganizationsApi.findOrganizations().asScala
                            .find(_.getName == org).map(_.getId).getOrElse("")
      val retentionRule = new BucketRetentionRules()
        .everySeconds((retentionDays * 86400).toLong)
        .`type`(BucketRetentionRules.TypeEnum.EXPIRE)
      val bucket = new Bucket()
        .name(name)
        .orgID(orgId)
        .addRetentionRulesItem(retentionRule)
      bucketsApi.createBucket(bucket)
      ()
```

---

## เกณฑ์การเลือก Database

### Decision Matrix

```scala
enum DataPattern:
  case Relational, Document, KeyValue, WideColumn, Graph, TimeSeries, Search

enum UseCase:
  case OLTP, Analytics, Caching, EventSourcing, FullTextSearch, GraphTraversal, TimeSeriesMetrics

case class DatabaseOption(
  name: String,
  patterns: Set[DataPattern],
  useCases: Set[UseCase],
  pros: List[String],
  cons: List[String],
  scalaLibrary: String
)

val databases: List[DatabaseOption] = List(
  DatabaseOption(
    name       = "PostgreSQL",
    patterns   = Set(DataPattern.Relational),
    useCases   = Set(UseCase.OLTP, UseCase.Analytics),
    pros       = List("ACID", "Rich SQL", "JSON support", "Mature ecosystem"),
    cons       = List("Vertical scaling limits", "Schema migrations"),
    scalaLibrary = "Doobie / Skunk"
  ),
  DatabaseOption(
    name       = "MongoDB",
    patterns   = Set(DataPattern.Document),
    useCases   = Set(UseCase.OLTP, UseCase.Analytics),
    pros       = List("Flexible schema", "Horizontal scaling", "Rich query language"),
    cons       = List("No ACID across collections", "Memory usage"),
    scalaLibrary = "Mongo4Cats / ReactiveMongo"
  ),
  DatabaseOption(
    name       = "Cassandra",
    patterns   = Set(DataPattern.WideColumn),
    useCases   = Set(UseCase.OLTP, UseCase.EventSourcing),
    pros       = List("Linear scalability", "High availability", "Write-optimized"),
    cons       = List("Limited query patterns", "Eventual consistency"),
    scalaLibrary = "Phantom DSL / Alpakka Cassandra"
  ),
  DatabaseOption(
    name       = "Elasticsearch",
    patterns   = Set(DataPattern.Search, DataPattern.Document),
    useCases   = Set(UseCase.FullTextSearch, UseCase.Analytics),
    pros       = List("Full-text search", "Aggregations", "Near real-time"),
    cons       = List("Not primary storage", "Resource intensive"),
    scalaLibrary = "elastic4s"
  ),
  DatabaseOption(
    name       = "Neo4j",
    patterns   = Set(DataPattern.Graph),
    useCases   = Set(UseCase.GraphTraversal),
    pros       = List("Native graph storage", "Cypher query language", "ACID"),
    cons       = List("Not for non-graph data", "Scaling challenges"),
    scalaLibrary = "Neo4j Java Driver"
  ),
  DatabaseOption(
    name       = "InfluxDB",
    patterns   = Set(DataPattern.TimeSeries),
    useCases   = Set(UseCase.TimeSeriesMetrics),
    pros       = List("Time-optimized storage", "Flux query language", "Downsampling"),
    cons       = List("Limited to time series", "Retention complexity"),
    scalaLibrary = "InfluxDB Java Client"
  )
)

def recommendDatabase(useCase: UseCase): List[DatabaseOption] =
  databases.filter(_.useCases.contains(useCase))

def selectByPattern(pattern: DataPattern): List[DatabaseOption] =
  databases.filter(_.patterns.contains(pattern))
```

---

## Polyglot Persistence

### Pattern: หลาย Database ร่วมกัน

```scala
import cats.effect.*
import cats.syntax.all.*

// แต่ละ domain ใช้ database ที่เหมาะสม
class EcommerceService(
  postgres: PostgresUserRepository,        // Users, Orders (ACID)
  mongo: MongoProductRepository,           // Products (flexible schema)
  redis: RedisCartService,                 // Shopping Cart (fast cache)
  elastic: ElasticSearchService,           // Product search
  neo4j: Neo4jRecommendationService,       // Recommendations (graph)
  influx: InfluxMetricsService             // Analytics (time series)
):
  def searchAndRecommend(userId: String, query: String): IO[SearchPage] =
    for
      // Parallel execution: search + user + recommendations
      (searchResults, user, recs) <- (
        elastic.searchProducts(query),
        postgres.findUser(userId),
        neo4j.recommendProducts(userId, limit = 10)
      ).parTupled

      // Get cart from Redis
      cart <- redis.getCart(userId)

      // Log search event for analytics
      _ <- influx.recordSearchEvent(userId, query, searchResults.length)
    yield SearchPage(
      results       = searchResults,
      recommendations = recs,
      cartItemCount = cart.items.length
    )

  def placeOrder(userId: String, items: List[CartItem]): IO[Either[String, Order]] =
    for
      // Validate user in PostgreSQL
      userOpt <- postgres.findUser(userId)
      user    <- IO.fromOption(userOpt)(new RuntimeException("User not found"))

      // Check product availability in MongoDB
      products <- items.traverse(i => mongo.findProduct(i.productId))

      // Create order in PostgreSQL (ACID)
      order <- postgres.createOrder(user, items)

      // Clear cart in Redis
      _ <- redis.clearCart(userId)

      // Update Neo4j: user purchased these products
      _ <- neo4j.recordPurchase(userId, items.map(_.productId))

      // Record metrics
      _ <- influx.recordOrderEvent(order.id, order.total)
    yield Right(order)
```

### Saga Pattern สำหรับ Distributed Transactions

```scala
// Saga: compensating transactions สำหรับ distributed consistency
sealed trait SagaStep[A]
case class DoStep[A](action: IO[A], compensate: A => IO[Unit]) extends SagaStep[A]

class Saga[A](private val steps: List[SagaStep[?]]):
  def run: IO[Either[String, Unit]] =
    runSteps(steps, List.empty)

  private def runSteps(
    remaining: List[SagaStep[?]],
    completed: List[IO[Unit]]
  ): IO[Either[String, Unit]] =
    remaining match
      case Nil => IO.pure(Right(()))
      case (step: DoStep[a]) :: rest =>
        step.action.attempt.flatMap:
          case Right(result) =>
            runSteps(rest, step.compensate(result) :: completed)
          case Left(err) =>
            // Run compensations in reverse order
            completed.sequence_ *> IO.pure(Left(err.getMessage))

object OrderSaga:
  def placeOrder(
    pg: PostgresRepository,
    mongo: MongoRepository,
    redis: RedisService,
    neo4j: Neo4jService
  )(userId: String, items: List[CartItem]): IO[Either[String, Order]] =
    for
      orderId <- IO(java.util.UUID.randomUUID().toString)
      result  <- Saga(List(
        DoStep(
          action     = pg.reserveInventory(items),
          compensate = _ => pg.releaseInventory(items)
        ),
        DoStep(
          action     = pg.createOrder(userId, orderId, items),
          compensate = _ => pg.cancelOrder(orderId)
        ),
        DoStep(
          action     = redis.clearCart(userId),
          compensate = _ => redis.restoreCart(userId, items)
        ),
        DoStep(
          action     = neo4j.recordPurchase(userId, items.map(_.productId)),
          compensate = _ => neo4j.removePurchase(userId, items.map(_.productId))
        )
      )).run
      order   <- result.traverse(_ => pg.findOrder(orderId).map(_.get))
    yield order

case class SearchPage(results: List[Product], recommendations: List[Product], cartItemCount: Int)
case class CartItem(productId: String, quantity: Int)
case class Order(id: String, userId: String, items: List[CartItem], total: BigDecimal)
```

---

## สรุป

Advanced Database Operations ใน Scala ครอบคลุม:

- **MongoDB / Mongo4Cats**: Document store สำหรับ flexible schema ด้วย type-safe Scala API
- **Cassandra / Phantom DSL**: Wide-column store สำหรับ write-heavy, high-availability workloads
- **Elasticsearch / elastic4s**: Full-text search engine ที่ทรงพลังด้วย DSL ที่อ่านง่าย
- **Neo4j**: Graph database สำหรับ relationship-heavy queries เช่น recommendations
- **InfluxDB**: Time series database สำหรับ metrics และ sensor data
- **Polyglot Persistence**: ใช้หลาย database ร่วมกันตาม use case ด้วย Saga pattern

---

*[← ส่วนที่ 91: Blockchain กับ Scala](part-91-blockchain.md) | [ส่วนที่ 93: Scala Interview Preparation →](part-93-interview-prep.md)*
