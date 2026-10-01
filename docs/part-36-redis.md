# Part 36: Redis กับ Scala

## สารบัญ
1. [Redis Overview](#redis-overview)
2. [Redis4Cats](#redis4cats)
3. [Data Structures](#data-structures)
4. [Pub/Sub](#pubsub)
5. [Caching Patterns](#caching-patterns)
6. [Session Management](#session-management)

---

## Redis Overview

### Dependencies

```scala
libraryDependencies ++= Seq(
  "dev.profunktor" %% "redis4cats-effects"  % "1.6.0",
  "dev.profunktor" %% "redis4cats-log4cats" % "1.6.0"
)
```

### Connection Setup

```scala
import cats.effect.{IO, IOApp, Resource}
import dev.profunktor.redis4cats.{Redis, RedisCommands}
import dev.profunktor.redis4cats.effect.Log.Stdout.*

// Basic connection
object RedisSetup:
  val redisResource: Resource[IO, RedisCommands[IO, String, String]] =
    Redis[IO].utf8("redis://localhost:6379")

  // With authentication
  val authRedis: Resource[IO, RedisCommands[IO, String, String]] =
    Redis[IO].utf8("redis://:password@localhost:6379/0")

  // Use in app
  def useRedis: IO[Unit] =
    redisResource.use { redis =>
      for
        _ <- redis.set("key", "value")
        v <- redis.get("key")
        _ <- IO.println(s"Got: $v")
      yield ()
    }
```

---

## Redis4Cats

### String Commands

```scala
import dev.profunktor.redis4cats.RedisCommands
import cats.effect.IO
import scala.concurrent.duration.*

def stringCommands(redis: RedisCommands[IO, String, String]): IO[Unit] =
  for
    // SET, GET
    _ <- redis.set("name", "Alice")
    v <- redis.get("name")
    _ <- IO.println(s"name = $v")  // Some(Alice)

    // SET with expiry
    _ <- redis.setEx("session", "token123", 1.hour)
    _ <- redis.pSetEx("temp", "data", 500.millis)

    // GET multiple keys
    vals <- redis.mGet(Set("name", "session", "missing"))
    _ <- IO.println(s"mGet = $vals")

    // SET if not exists (NX)
    _ <- redis.setNx("lock", "acquired")

    // Increment/Decrement
    _ <- redis.set("counter", "10")
    n1 <- redis.incr("counter")         // 11
    n2 <- redis.incrBy("counter", 5)    // 16
    n3 <- redis.decr("counter")         // 15
    _ <- IO.println(s"counter: $n1, $n2, $n3")

    // Append
    _ <- redis.set("greeting", "Hello")
    _ <- redis.append("greeting", ", World!")
    g <- redis.get("greeting")
    _ <- IO.println(g)  // Some(Hello, World!)

    // TTL
    ttl <- redis.ttl("session")
    _ <- IO.println(s"TTL: $ttl")

    // Delete
    _ <- redis.del("name", "counter")
  yield ()
```

---

## Data Structures

### Hash Commands

```scala
import dev.profunktor.redis4cats.RedisCommands
import cats.effect.IO

def hashCommands(redis: RedisCommands[IO, String, String]): IO[Unit] =
  for
    // HSET: set hash fields
    _ <- redis.hSet("user:1", Map(
      "name"  -> "Alice",
      "email" -> "alice@example.com",
      "age"   -> "30"
    ))

    // HGET: get single field
    name <- redis.hGet("user:1", "name")
    _ <- IO.println(s"name = $name")  // Some(Alice)

    // HMGET: get multiple fields
    fields <- redis.hmGet("user:1", "name", "email")
    _ <- IO.println(s"fields = $fields")

    // HGETALL: get all fields
    all <- redis.hGetAll("user:1")
    _ <- IO.println(s"all = $all")

    // HINCRBY: increment hash field
    _ <- redis.hSet("user:1", "score", "100")
    _ <- redis.hIncrBy("user:1", "score", 50)
    score <- redis.hGet("user:1", "score")
    _ <- IO.println(s"score = $score")  // Some(150)

    // HDEL: delete fields
    _ <- redis.hDel("user:1", "age")

    // HEXISTS
    exists <- redis.hExists("user:1", "name")
    _ <- IO.println(s"name exists: $exists")
  yield ()

// List Commands
def listCommands(redis: RedisCommands[IO, String, String]): IO[Unit] =
  for
    // LPUSH/RPUSH
    _ <- redis.lPush("mylist", "c", "b", "a")  // head: a, b, c
    _ <- redis.rPush("mylist", "d", "e")         // tail: d, e

    // LRANGE: get range
    items <- redis.lRange("mylist", 0, -1)  // all items
    _ <- IO.println(s"list = $items")

    // LPOP/RPOP
    head <- redis.lPop("mylist")
    tail <- redis.rPop("mylist")
    _ <- IO.println(s"head=$head, tail=$tail")

    // LLEN
    len <- redis.lLen("mylist")
    _ <- IO.println(s"length = $len")
  yield ()

// Set Commands
def setCommands(redis: RedisCommands[IO, String, String]): IO[Unit] =
  for
    // SADD
    _ <- redis.sAdd("tags", "scala", "functional", "jvm")

    // SMEMBERS
    members <- redis.sMembers("tags")
    _ <- IO.println(s"tags = $members")

    // SISMEMBER
    has <- redis.sIsMember("tags", "scala")
    _ <- IO.println(s"has scala: $has")

    // SCARD
    size <- redis.sCard("tags")
    _ <- IO.println(s"size = $size")

    // Set operations
    _ <- redis.sAdd("set1", "a", "b", "c")
    _ <- redis.sAdd("set2", "b", "c", "d")

    inter <- redis.sInter("set1", "set2")  // {b, c}
    union <- redis.sUnion("set1", "set2")  // {a, b, c, d}
    diff  <- redis.sDiff("set1", "set2")   // {a}
    _ <- IO.println(s"inter=$inter, union=$union, diff=$diff")
  yield ()
```

---

## Pub/Sub

### Publish/Subscribe Pattern

```scala
import dev.profunktor.redis4cats.PubSubCommands
import dev.profunktor.redis4cats.data.RedisChannel
import cats.effect.IO
import fs2.Stream

// Subscriber
def subscriber(redis: PubSubCommands[Stream[IO, *], String, String]): IO[Unit] =
  val channel = RedisChannel("news")

  redis.subscribe(channel)
    .evalMap { message =>
      IO.println(s"Received: $message")
    }
    .take(10)
    .compile.drain

// Publisher
def publisher(redis: RedisCommands[IO, String, String]): IO[Unit] =
  val messages = List("Breaking news!", "Tech update", "Sports score")
  IO.foreach(messages) { msg =>
    redis.publish("news", msg)
  }.void

// Pattern subscribe
def patternSubscriber(redis: PubSubCommands[Stream[IO, *], String, String]): IO[Unit] =
  redis.pSubscribe("events.*")  // subscribe to events.login, events.logout, etc.
    .evalMap { case (pattern, channel, msg) =>
      IO.println(s"[$channel] $msg")
    }
    .compile.drain
```

---

## Caching Patterns

### Cache-Aside Pattern

```scala
import cats.effect.IO
import dev.profunktor.redis4cats.RedisCommands
import scala.concurrent.duration.*
import io.circe.*
import io.circe.syntax.*
import io.circe.parser.*
import io.circe.generic.auto.*

case class UserProfile(id: Long, name: String, email: String, bio: String)

class UserCache(redis: RedisCommands[IO, String, String]):
  private def cacheKey(id: Long) = s"user:profile:$id"
  private val ttl = 30.minutes

  // Try cache first, then DB
  def get(id: Long)(fetchFromDb: Long => IO[Option[UserProfile]]): IO[Option[UserProfile]] =
    for
      cached <- redis.get(cacheKey(id))
      result <- cached match
        case Some(json) =>
          IO.fromEither(decode[UserProfile](json)).map(Some(_))
            .handleErrorWith(_ => fetchFromDb(id))
        case None =>
          fetchFromDb(id).flatMap {
            case Some(user) =>
              redis.setEx(cacheKey(id), user.asJson.noSpaces, ttl).as(Some(user))
            case None =>
              IO.pure(None)
          }
    yield result

  def invalidate(id: Long): IO[Unit] =
    redis.del(cacheKey(id)).void

  def invalidateAll(ids: List[Long]): IO[Unit] =
    redis.del(ids.map(cacheKey)*).void

// Write-through cache
class WriteThrough(redis: RedisCommands[IO, String, String]):
  def update(key: String, value: String)(
    updateDb: (String, String) => IO[Unit]
  ): IO[Unit] =
    for
      _ <- updateDb(key, value)       // update DB first
      _ <- redis.set(key, value)      // then update cache
    yield ()
```

---

## Session Management

### JWT Session with Redis

```scala
import dev.profunktor.redis4cats.RedisCommands
import cats.effect.IO
import scala.concurrent.duration.*
import java.util.UUID

case class Session(userId: Long, username: String, roles: List[String])

class SessionStore(redis: RedisCommands[IO, String, String]):
  private val prefix  = "session:"
  private val timeout = 24.hours

  private def key(token: String) = s"$prefix$token"

  def create(session: Session): IO[String] =
    val token = UUID.randomUUID().toString
    val data  = s"${session.userId}|${session.username}|${session.roles.mkString(",")}"
    redis.setEx(key(token), data, timeout).as(token)

  def get(token: String): IO[Option[Session]] =
    redis.get(key(token)).map {
      _.flatMap { data =>
        data.split("\\|") match
          case Array(id, name, roles) =>
            id.toLongOption.map { uid =>
              Session(uid, name, roles.split(",").toList)
            }
          case _ => None
      }
    }

  def refresh(token: String): IO[Boolean] =
    redis.expire(key(token), timeout).map(_.isDefined)

  def invalidate(token: String): IO[Unit] =
    redis.del(key(token)).void

  def invalidateUser(userId: Long): IO[Unit] =
    // In practice: track tokens per user in a Set
    IO.unit
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Redis4Cats setup และ connection
- ✅ String commands: SET, GET, INCR, TTL
- ✅ Hash commands: HSET, HGET, HGETALL
- ✅ List/Set commands
- ✅ Pub/Sub pattern
- ✅ Cache-Aside pattern
- ✅ Session management with JWT

---

*[← Part 35: Tapir](part-35-tapir.md) | [Part 37: Kafka →](part-37-kafka.md)*
