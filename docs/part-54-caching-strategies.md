# Part 54: Advanced Caching Strategies

## สารบัญ

1. [ทำไมต้อง Cache](#ทำไมต้อง-cache)
2. [Cache Patterns พื้นฐาน](#cache-patterns-พื้นฐาน)
3. [Multi-Level Cache (L1 + L2)](#multi-level-cache-l1--l2)
4. [Cache Invalidation Strategies](#cache-invalidation-strategies)
5. [Cache Stampede Prevention](#cache-stampede-prevention)
6. [Memoization ด้วย cats-effect Ref](#memoization-ด้วย-cats-effect-ref)
7. [TTL และ Eviction Policies](#ttl-และ-eviction-policies)
8. [ตัวอย่าง Production Cache](#ตัวอย่าง-production-cache)

---

## ทำไมต้อง Cache

Cache ลด latency และ load บน backend services:

```
ไม่มี Cache:
Request → DB Query (50ms) → Response     ← ทุก request ไปที่ DB

มี Cache:
Request → Cache Hit (1ms)  → Response    ← 99% ของ requests
Request → Cache Miss → DB (50ms) → Cache → Response  ← 1%

ประโยชน์:
  Response time: 50ms → 1ms (50x faster)
  DB load: 1000 req/s → 10 req/s (100x reduction)
  Cost: ลดค่า DB instance ได้มาก
```

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.typelevel"   %% "cats-effect"          % "3.5.4",
  "dev.profunktor"  %% "redis4cats-effects"   % "1.7.0",
  "dev.profunktor"  %% "redis4cats-log4cats"  % "1.7.0",
  "io.circe"        %% "circe-core"           % "0.14.9",
  "io.circe"        %% "circe-generic"        % "0.14.9",
  "io.circe"        %% "circe-parser"         % "0.14.9",
  "com.github.blemale" %% "scaffeine"         % "5.2.1",  // Caffeine wrapper
  "org.tpolecat"    %% "doobie-core"          % "1.0.0-RC4"
)
```

---

## Cache Patterns พื้นฐาน

### 1. Cache-Aside (Lazy Loading)

Application ควบคุม cache เอง อ่าน cache ก่อน ถ้า miss ค่อยไป source:

```scala
import cats.effect.*
import cats.syntax.all.*
import io.circe.*
import io.circe.parser.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*
import dev.profunktor.redis4cats.RedisCommands
import scala.concurrent.duration.*

case class Product(id: String, name: String, price: BigDecimal, stock: Int)
given Encoder[Product] = deriveEncoder
given Decoder[Product] = deriveDecoder

class CacheAsideRepository(
  redis: RedisCommands[IO, String, String],
  db: ProductDatabase,
  ttl: FiniteDuration = 5.minutes
):

  def findById(id: String): IO[Option[Product]] =
    val cacheKey = s"product:$id"

    redis.get(cacheKey).flatMap:
      case Some(json) =>
        // Cache HIT
        IO.pure(parse(json).flatMap(_.as[Product]).toOption)

      case None =>
        // Cache MISS: fetch from DB, populate cache
        db.findById(id).flatTap:
          case Some(product) =>
            redis.setEx(cacheKey, product.asJson.noSpaces, ttl)
          case None =>
            // Cache negative results too (prevent cache penetration)
            redis.setEx(cacheKey + ":null", "true", 30.seconds)

  def update(product: Product): IO[Unit] =
    val cacheKey = s"product:${product.id}"
    // Update DB first, then invalidate cache
    db.update(product) >>
    redis.del(cacheKey).void

// Cache-Aside is the most common pattern because:
// - Application has full control
// // - Cache failures are isolated from reads
// - Works well with read-heavy workloads
```

### 2. Write-Through Cache

เขียน cache และ database พร้อมกันเสมอ:

```scala
class WriteThroughCache(
  redis: RedisCommands[IO, String, String],
  db: ProductDatabase,
  ttl: FiniteDuration = 10.minutes
):

  // Write to BOTH cache and DB atomically (conceptually)
  def save(product: Product): IO[Unit] =
    val cacheKey = s"product:${product.id}"

    // Write to both - if DB fails, cache write is rolled back conceptually
    db.save(product)
      .flatMap(_ => redis.setEx(cacheKey, product.asJson.noSpaces, ttl))
      .handleErrorWith: e =>
        // If cache write fails, at least DB has the data
        IO.println(s"Cache write failed for ${product.id}: $e")

  def findById(id: String): IO[Option[Product]] =
    val cacheKey = s"product:$id"
    redis.get(cacheKey).flatMap:
      case Some(json) => IO.pure(parse(json).flatMap(_.as[Product]).toOption)
      case None       => db.findById(id)  // Should be rare with write-through

// Write-Through pros:
// - Cache always consistent with DB
// - No stale reads
// Write-Through cons:
// - Higher write latency (write to both)
// - May cache data that's never read (write amplification)
```

### 3. Write-Behind (Write-Back) Cache

เขียน cache ก่อน แล้วค่อย flush ไป DB แบบ async:

```scala
import cats.effect.*
import cats.effect.std.Queue
import fs2.*
import scala.concurrent.duration.*

class WriteBehindCache(
  redis: RedisCommands[IO, String, String],
  db: ProductDatabase,
  writeQueue: Queue[IO, Product],
  ttl: FiniteDuration = 10.minutes
):

  // Write to cache immediately, enqueue for DB write
  def save(product: Product): IO[Unit] =
    val cacheKey = s"product:${product.id}"
    redis.setEx(cacheKey, product.asJson.noSpaces, ttl) >>
    writeQueue.offer(product)  // Non-blocking, fast!

  def findById(id: String): IO[Option[Product]] =
    val cacheKey = s"product:$id"
    redis.get(cacheKey).flatMap:
      case Some(json) => IO.pure(parse(json).flatMap(_.as[Product]).toOption)
      case None       => db.findById(id)

  // Background worker: flush queue to DB in batches
  def backgroundFlusher: Stream[IO, Unit] =
    Stream
      .fromQueueUnterminated(writeQueue)
      .groupWithin(100, 500.millis)  // Batch up to 100 items or 500ms
      .evalMap: batch =>
        db.saveBatch(batch.toList)
          .handleErrorWith: e =>
            IO.println(s"Write-behind flush failed: $e") >>
            batch.toList.traverse_(writeQueue.offer)  // Re-queue on failure

object WriteBehindCache:
  def create(
    redis: RedisCommands[IO, String, String],
    db: ProductDatabase
  ): IO[WriteBehindCache] =
    Queue.bounded[IO, Product](1000).map: queue =>
      new WriteBehindCache(redis, db, queue)

// Write-Behind pros:
// - Very fast writes (just cache + queue)
// - Batch DB writes reduce load
// Write-Behind cons:
// - Risk of data loss if cache crashes before flush
// - Complex implementation
// - Eventual consistency
```

---

## Multi-Level Cache (L1 + L2)

### L1 = In-Memory (Caffeine) + L2 = Redis

```
Request ──▶ L1 Cache (JVM Heap)           Fast: 0.01ms
             │ miss
             ▼
            L2 Cache (Redis)              Medium: 1ms
             │ miss
             ▼
            Database                      Slow: 50ms+
```

```scala
import cats.effect.*
import cats.syntax.all.*
import com.github.blemale.scaffeine.*
import scala.concurrent.duration.*
import io.circe.*

// L1: In-memory cache using Caffeine
class CaffeineL1Cache[V](
  underlyingCache: AsyncLoadingCache[String, Option[V]]
):
  def get(key: String): IO[Option[V]] =
    IO.fromFuture(IO(underlyingCache.get(key).toScala))
      .map(identity)

  def put(key: String, value: V): IO[Unit] =
    IO(underlyingCache.put(key, scala.concurrent.Future.successful(Some(value))))

  def invalidate(key: String): IO[Unit] =
    IO(underlyingCache.synchronous().invalidate(key))

object CaffeineL1Cache:
  def create[V](
    maxSize: Int = 1000,
    expireAfterWrite: FiniteDuration = 1.minute,
    loader: String => IO[Option[V]]
  ): IO[CaffeineL1Cache[V]] =
    import scala.concurrent.ExecutionContext.Implicits.global

    IO:
      val cache = Scaffeine()
        .maximumSize(maxSize)
        .expireAfterWrite(expireAfterWrite)
        .buildAsyncFuture[String, Option[V]]: key =>
          import cats.effect.unsafe.implicits.global as ioRuntime
          loader(key).unsafeToFuture()(ioRuntime)

      new CaffeineL1Cache(cache)

// L2: Redis cache
class RedisL2Cache[V: Encoder: Decoder](
  redis: RedisCommands[IO, String, String],
  ttl: FiniteDuration = 10.minutes,
  keyPrefix: String = "cache"
):

  def get(key: String): IO[Option[V]] =
    redis.get(s"$keyPrefix:$key").map:
      _.flatMap(json => parse(json).flatMap(_.as[V]).toOption)

  def put(key: String, value: V): IO[Unit] =
    redis.setEx(s"$keyPrefix:$key", value.asJson.noSpaces, ttl)

  def delete(key: String): IO[Unit] =
    redis.del(s"$keyPrefix:$key").void

  def deleteByPattern(pattern: String): IO[Unit] =
    redis.keys(s"$keyPrefix:$pattern").flatMap: keys =>
      if keys.isEmpty then IO.unit
      else redis.del(keys.head, keys.tail*).void

// Multi-level cache composition
class MultiLevelCache[V: Encoder: Decoder](
  l1: CaffeineL1Cache[V],
  l2: RedisL2Cache[V],
  source: String => IO[Option[V]]
):

  def get(key: String): IO[Option[V]] =
    // Try L1 first (fastest)
    // Note: CaffeineL1Cache uses AsyncLoadingCache so it calls source on miss
    // Here we manually implement the chain for clarity:
    l1.get(key).flatMap:
      case Some(v) =>
        IO.pure(Some(v))  // L1 hit!

      case None =>
        // Try L2
        l2.get(key).flatMap:
          case Some(v) =>
            // L2 hit! Populate L1
            l1.put(key, v).as(Some(v))

          case None =>
            // Source hit! Populate both caches
            source(key).flatTap:
              case Some(v) =>
                l1.put(key, v) >> l2.put(key, v)
              case None =>
                IO.unit

  def invalidate(key: String): IO[Unit] =
    l1.invalidate(key) >> l2.delete(key)

  def invalidateAll(pattern: String): IO[Unit] =
    // L1 doesn't support pattern invalidation easily
    // Use tagged invalidation or version keys instead
    l2.deleteByPattern(pattern)
```

### Version-Based Cache Invalidation

```scala
import cats.effect.*
import dev.profunktor.redis4cats.RedisCommands
import scala.concurrent.duration.*

// Use version number as part of cache key
// Incrementing version invalidates all related entries
class VersionedCache[V: Encoder: Decoder](
  redis: RedisCommands[IO, String, String],
  ttl: FiniteDuration = 10.minutes
):

  def getVersion(namespace: String): IO[Long] =
    redis.get(s"version:$namespace")
      .map(_.flatMap(_.toLongOption).getOrElse(1L))

  def incrementVersion(namespace: String): IO[Long] =
    redis.incr(s"version:$namespace")

  def get(namespace: String, key: String): IO[Option[V]] =
    for
      version  <- getVersion(namespace)
      fullKey   = s"$namespace:v$version:$key"
      result   <- redis.get(fullKey).map:
                    _.flatMap(json => parse(json).flatMap(_.as[V]).toOption)
    yield result

  def put(namespace: String, key: String, value: V): IO[Unit] =
    for
      version <- getVersion(namespace)
      fullKey  = s"$namespace:v$version:$key"
      _       <- redis.setEx(fullKey, value.asJson.noSpaces, ttl)
    yield ()

  // Invalidate ENTIRE namespace by bumping version
  def invalidateNamespace(namespace: String): IO[Unit] =
    incrementVersion(namespace).void

// Usage
class ProductCacheService(
  versionedCache: VersionedCache[Product],
  db: ProductDatabase
):

  def findById(id: String): IO[Option[Product]] =
    versionedCache.get("products", id).flatMap:
      case Some(p) => IO.pure(Some(p))
      case None    =>
        db.findById(id).flatTap:
          case Some(p) => versionedCache.put("products", id, p)
          case None    => IO.unit

  // Invalidate all product caches at once
  def onProductCatalogUpdate(): IO[Unit] =
    versionedCache.invalidateNamespace("products")
```

---

## Cache Invalidation Strategies

### Event-Driven Invalidation

```scala
import cats.effect.*
import fs2.*
import fs2.kafka.*

// Listen to domain events and invalidate cache
class CacheInvalidationListener(
  multiLevelCache: MultiLevelCache[Product],
  kafkaSettings: ConsumerSettings[IO, String, String]
):

  def run: Stream[IO, Unit] =
    KafkaConsumer
      .stream(kafkaSettings)
      .subscribeTo("domain.Product.product_updated", "domain.Product.product_deleted")
      .records
      .evalMap: record =>
        handleEvent(record.record.topic, record.record.key, record.record.value)
          .flatMap(_ => record.offset.commit)

  private def handleEvent(
    topic: String,
    productId: String,
    payload: String
  ): IO[Unit] =
    topic match
      case "domain.Product.product_updated" =>
        IO.println(s"Cache invalidation: product $productId updated") >>
        multiLevelCache.invalidate(productId)

      case "domain.Product.product_deleted" =>
        IO.println(s"Cache invalidation: product $productId deleted") >>
        multiLevelCache.invalidate(productId)

      case _ =>
        IO.unit

// Tag-Based Invalidation
class TaggedCache(
  redis: RedisCommands[IO, String, String],
  ttl: FiniteDuration = 10.minutes
):
  // Associate cache entries with tags
  // Tag "category:electronics" → [product:1, product:5, product:12, ...]

  def putWithTags(key: String, value: String, tags: Set[String]): IO[Unit] =
    for
      // Store the value
      _ <- redis.setEx(s"cache:$key", value, ttl)
      // Add key to each tag's set
      _ <- tags.toList.traverse_(tag =>
             redis.sAdd(s"tag:$tag", key) >>
             redis.expire(s"tag:$tag", ttl + 1.minute)
           )
    yield ()

  // Invalidate all entries with a given tag
  def invalidateTag(tag: String): IO[Unit] =
    for
      keys <- redis.sMembers(s"tag:$tag")
      _    <- keys.toList.traverse_(key =>
                redis.del(s"cache:$key").void
              )
      _    <- redis.del(s"tag:$tag").void
    yield ()
```

---

## Cache Stampede Prevention

Cache Stampede (หรือ Thundering Herd) เกิดขึ้นเมื่อ cache entry หมดอายุและ requests จำนวนมากพยายาม rebuild cache พร้อมกัน:

```
Cache Stampede:

t=0: Cache key expires
t=0.001: Request 1 misses → starts DB query
t=0.002: Request 2 misses → starts DB query  ← STAMPEDE!
t=0.003: Request 3 misses → starts DB query  ← 
t=0.004: Request 4 misses → starts DB query  ←
...100 concurrent DB queries for the same data!
```

### Probabilistic Early Expiration

```scala
import cats.effect.*
import cats.effect.Ref
import scala.concurrent.duration.*
import scala.math.*

// XFetch algorithm: probabilistically refresh cache before it expires
class ProbabilisticCache[V: Encoder: Decoder](
  redis: RedisCommands[IO, String, String],
  beta: Double = 1.0  // Higher beta = more aggressive pre-fetching
):

  case class CachedValue[A](
    value: A,
    expiresAt: Long,    // Unix timestamp in millis
    computationTimeMs: Long  // How long it took to compute
  )

  def getOrCompute(
    key: String,
    ttl: FiniteDuration,
    compute: IO[V]
  ): IO[V] =
    redis.get(s"xfetch:$key").flatMap:
      case Some(json) =>
        io.circe.parser.parse(json)
          .flatMap(_.as[CachedValue[V]])
          .fold(
            _ => recompute(key, ttl, compute),
            cached =>
              val now = System.currentTimeMillis()
              // Should we pre-emptively refresh?
              if shouldRefresh(cached, now) then
                // Refresh in background, return cached value
                recompute(key, ttl, compute).start.void.as(cached.value)
              else
                IO.pure(cached.value)
          )

      case None =>
        recompute(key, ttl, compute)

  private def shouldRefresh(cached: CachedValue[V], now: Long): Boolean =
    // XFetch formula: now - delta * beta * log(random) > expiry - ttl
    val delta = cached.computationTimeMs.toDouble / 1000.0  // seconds
    val gap   = (cached.expiresAt - now) / 1000.0  // seconds until expiry
    val random = scala.util.Random.nextDouble()

    // Probability of refresh increases as expiry approaches
    -delta * beta * log(random) > gap

  private def recompute(key: String, ttl: FiniteDuration, compute: IO[V]): IO[V] =
    for
      start   <- IO.realTime.map(_.toMillis)
      value   <- compute
      end     <- IO.realTime.map(_.toMillis)
      cached   = CachedValue(value, System.currentTimeMillis() + ttl.toMillis, end - start)
      _       <- redis.setEx(s"xfetch:$key", cached.asJson.noSpaces, ttl + 1.minute)
    yield value

  given Encoder[CachedValue[V]] = Encoder.instance: cv =>
    io.circe.Json.obj(
      "value"             -> cv.value.asJson,
      "expiresAt"         -> cv.expiresAt.asJson,
      "computationTimeMs" -> cv.computationTimeMs.asJson
    )

  given Decoder[CachedValue[V]] = Decoder.instance: cursor =>
    for
      value  <- cursor.downField("value").as[V]
      exp    <- cursor.downField("expiresAt").as[Long]
      comp   <- cursor.downField("computationTimeMs").as[Long]
    yield CachedValue(value, exp, comp)
```

### Mutex-Based Stampede Prevention

```scala
import cats.effect.*
import cats.effect.std.Semaphore
import dev.profunktor.redis4cats.RedisCommands
import scala.concurrent.duration.*

class MutexCache[V: Encoder: Decoder](
  redis: RedisCommands[IO, String, String],
  localMutexes: Ref[IO, Map[String, Semaphore[IO]]],
  ttl: FiniteDuration = 10.minutes
):

  def getOrCompute(key: String, compute: IO[V]): IO[V] =
    redis.get(s"cache:$key").flatMap:
      case Some(json) =>
        io.circe.parser.parse(json)
          .flatMap(_.as[V])
          .fold(_ => IO.raiseError(new Exception("Deserialization failed")), IO.pure)

      case None =>
        // Get or create mutex for this key
        getOrCreateMutex(key).flatMap: mutex =>
          mutex.permit.use: _ =>
            // Re-check cache inside mutex (double-checked locking)
            redis.get(s"cache:$key").flatMap:
              case Some(json) =>
                io.circe.parser.parse(json)
                  .flatMap(_.as[V])
                  .fold(_ => compute.flatTap(v => cacheValue(key, v)), IO.pure)
              case None =>
                compute.flatTap(v => cacheValue(key, v))

  private def cacheValue(key: String, value: V): IO[Unit] =
    redis.setEx(s"cache:$key", value.asJson.noSpaces, ttl)

  private def getOrCreateMutex(key: String): IO[Semaphore[IO]] =
    localMutexes.get.flatMap: mutexMap =>
      mutexMap.get(key) match
        case Some(mutex) => IO.pure(mutex)
        case None =>
          Semaphore[IO](1).flatTap: mutex =>
            localMutexes.update(_ + (key -> mutex))

object MutexCache:
  def create[V: Encoder: Decoder](
    redis: RedisCommands[IO, String, String],
    ttl: FiniteDuration = 10.minutes
  ): IO[MutexCache[V]] =
    Ref.of[IO, Map[String, Semaphore[IO]]](Map.empty).map: mutexes =>
      new MutexCache(redis, mutexes, ttl)
```

---

## Memoization ด้วย cats-effect Ref

### Simple Memoization

```scala
import cats.effect.*
import cats.effect.Ref
import cats.syntax.all.*

// Memoize expensive computations
def memoize[A, B](f: A => IO[B]): IO[A => IO[B]] =
  Ref.of[IO, Map[A, B]](Map.empty).map: cache =>
    (a: A) =>
      cache.get.flatMap: map =>
        map.get(a) match
          case Some(b) => IO.pure(b)
          case None    =>
            f(a).flatTap: b =>
              cache.update(_ + (a -> b))

// Usage
object MemoizationExample extends IOApp.Simple:

  val expensiveComputation: Int => IO[Int] = n =>
    IO.sleep(1.second) >> IO.pure(n * n)

  def run: IO[Unit] =
    for
      memoized <- memoize(expensiveComputation)
      r1       <- memoized(5)   // Takes 1 second
      r2       <- memoized(5)   // Instant! (cached)
      r3       <- memoized(10)  // Takes 1 second
      _        <- IO.println(s"5² = $r1, 5² = $r2, 10² = $r3")
    yield ()
```

### Concurrent-Safe Memoization

```scala
import cats.effect.*
import cats.effect.Ref
import cats.effect.Deferred

// Using Deferred to prevent concurrent duplicate computations
def memoizeDeferred[A, B](f: A => IO[B]): IO[A => IO[B]] =
  Ref.of[IO, Map[A, Deferred[IO, Either[Throwable, B]]]](Map.empty).map: cache =>
    (a: A) =>
      for
        deferred  <- Deferred[IO, Either[Throwable, B]]
        action    <- cache.modify: map =>
                       map.get(a) match
                         case Some(existing) =>
                           (map, existing.get.rethrow)
                         case None =>
                           val computation =
                             f(a)
                               .attempt
                               .flatTap(result => deferred.complete(result))
                               .flatMap(IO.fromEither)
                           (map + (a -> deferred), computation)
        result    <- action
      yield result

// TTL-aware memoization
case class CacheEntry[A](value: A, expiresAt: Long)

def memoizeWithTTL[A, B](
  f: A => IO[B],
  ttl: FiniteDuration
): IO[A => IO[B]] =
  Ref.of[IO, Map[A, CacheEntry[B]]](Map.empty).map: cache =>
    (a: A) =>
      IO.realTime.flatMap: now =>
        val nowMillis = now.toMillis
        cache.get.flatMap: map =>
          map.get(a) match
            case Some(entry) if entry.expiresAt > nowMillis =>
              IO.pure(entry.value)  // Cache hit, not expired
            case _ =>
              f(a).flatTap: b =>
                val expiry = nowMillis + ttl.toMillis
                cache.update(_ + (a -> CacheEntry(b, expiry)))
```

---

## TTL และ Eviction Policies

### LRU Cache (Least Recently Used)

```scala
import cats.effect.*
import cats.effect.Ref
import scala.collection.mutable

class LRUCache[K, V](
  capacity: Int,
  store: Ref[IO, mutable.LinkedHashMap[K, V]]
):

  def get(key: K): IO[Option[V]] =
    store.modify: map =>
      map.get(key) match
        case Some(value) =>
          // Move to end (most recently used)
          map.remove(key)
          map.put(key, value)
          (map, Some(value))
        case None =>
          (map, None)

  def put(key: K, value: V): IO[Unit] =
    store.update: map =>
      if map.contains(key) then
        map.remove(key)
      else if map.size >= capacity then
        // Remove least recently used (first entry)
        val lruKey = map.keys.head
        map.remove(lruKey)
      map.put(key, value)
      map

  def size: IO[Int] = store.get.map(_.size)

object LRUCache:
  def create[K, V](capacity: Int): IO[LRUCache[K, V]] =
    Ref
      .of[IO, mutable.LinkedHashMap[K, V]](
        new mutable.LinkedHashMap[K, V](
          initialCapacity = capacity + 1,
          loadFactor = 1.0f
        )
      )
      .map(new LRUCache(capacity, _))
```

### TTL-Aware Eviction

```scala
import cats.effect.*
import cats.effect.Ref
import fs2.*
import java.time.Instant
import scala.concurrent.duration.*

class TtlCache[K, V](
  store: Ref[IO, Map[K, (V, Long)]],  // key -> (value, expiresAt)
  ttl: FiniteDuration
):

  def get(key: K): IO[Option[V]] =
    IO.realTime.flatMap: now =>
      store.get.map: map =>
        map.get(key).flatMap: (value, expiresAt) =>
          if expiresAt > now.toMillis then Some(value)
          else None

  def put(key: K, value: V): IO[Unit] =
    IO.realTime.flatMap: now =>
      store.update: map =>
        map + (key -> (value, now.toMillis + ttl.toMillis))

  def delete(key: K): IO[Unit] =
    store.update(_ - key)

  // Background eviction of expired entries
  def evictionStream: Stream[IO, Unit] =
    Stream
      .awakeEvery[IO](ttl / 2)  // Run cleanup at half TTL interval
      .evalMap(_ => evictExpired)

  private def evictExpired: IO[Int] =
    IO.realTime.flatMap: now =>
      store.modify: map =>
        val (expired, fresh) = map.partition((_, entry) => entry._2 <= now.toMillis)
        (fresh, expired.size)

  def size: IO[Int] = store.get.map(_.size)

object TtlCache:
  def create[K, V](ttl: FiniteDuration): IO[TtlCache[K, V]] =
    Ref.of[IO, Map[K, (V, Long)]](Map.empty).map(new TtlCache(_, ttl))
```

---

## ตัวอย่าง Production Cache

### Complete Cache Service

```scala
import cats.effect.*
import cats.effect.Ref
import cats.syntax.all.*
import dev.profunktor.redis4cats.RedisCommands
import io.circe.*
import io.circe.syntax.*
import io.circe.parser.*
import io.circe.generic.semiauto.*
import scala.concurrent.duration.*
import scala.math.*
import java.util.concurrent.atomic.AtomicLong

// Cache statistics
case class CacheStats(
  hits: Long,
  misses: Long,
  evictions: Long,
  size: Int
):
  def hitRate: Double =
    val total = hits + misses
    if total > 0 then hits.toDouble / total else 0.0

class CacheStatsCollector:
  private val hits      = new AtomicLong(0)
  private val misses    = new AtomicLong(0)
  private val evictions = new AtomicLong(0)

  def recordHit(): Unit      = hits.incrementAndGet()
  def recordMiss(): Unit     = misses.incrementAndGet()
  def recordEviction(): Unit = evictions.incrementAndGet()

  def getStats(cacheSize: Int): CacheStats =
    CacheStats(hits.get, misses.get, evictions.get, cacheSize)

  def reset(): Unit =
    hits.set(0); misses.set(0); evictions.set(0)

// Production-ready cache with all features
class ProductionCache[K, V: Encoder: Decoder](
  l1: LRUCache[K, V],
  redis: RedisCommands[IO, String, String],
  source: K => IO[Option[V]],
  stats: CacheStatsCollector,
  l2Ttl: FiniteDuration = 10.minutes,
  l1Ttl: FiniteDuration = 1.minute,
  keyPrefix: String = "cache"
):

  def get(key: K): IO[Option[V]] =
    // L1 check
    l1.get(key).flatMap:
      case Some(v) =>
        IO(stats.recordHit()) >> IO.pure(Some(v))

      case None =>
        // L2 check (Redis)
        val redisKey = s"$keyPrefix:$key"
        redis.get(redisKey).flatMap:
          case Some(json) =>
            val result = parse(json).flatMap(_.as[V]).toOption
            IO(stats.recordHit()) >>
            result.traverse_(v => l1.put(key, v)).as(result)

          case None =>
            // Source lookup
            IO(stats.recordMiss()) >>
            source(key).flatTap:
              case Some(v) =>
                l1.put(key, v) >>
                redis.setEx(redisKey, v.asJson.noSpaces, l2Ttl)
              case None =>
                // Cache negative results briefly
                redis.setEx(s"$redisKey:null", "1", 30.seconds)

  def invalidate(key: K): IO[Unit] =
    l1.delete(key) >>
    redis.del(s"$keyPrefix:$key").void

  def getStats: IO[CacheStats] =
    l1.size.map(stats.getStats)

  def logStats: IO[Unit] =
    getStats.flatMap: s =>
      IO.println(
        s"Cache Stats: hits=${s.hits} misses=${s.misses} " +
        s"hit_rate=${f"${s.hitRate * 100}%.1f"}% size=${s.size}"
      )

// Application service using the cache
class ProductService(
  cache: ProductionCache[String, Product],
  db: ProductDatabase
):

  def getProduct(id: String): IO[Option[Product]] =
    cache.get(id)

  def updateProduct(product: Product): IO[Unit] =
    db.save(product) >>
    cache.invalidate(product.id)

  def warmCache(ids: List[String]): IO[Unit] =
    ids.parTraverseN(10): id =>
      db.findById(id).flatMap:
        case Some(p) => cache.get(id).void  // This will cache it
        case None    => IO.unit
    .void
```

### Cache Warming Strategy

```scala
import cats.effect.*
import fs2.*
import scala.concurrent.duration.*

class CacheWarmer[K, V: Encoder: Decoder](
  cache: ProductionCache[K, V],
  db: ProductDatabase
):

  // Warm cache on startup
  def warmOnStartup(popularIds: List[String]): IO[Unit] =
    IO.println(s"Warming cache with ${popularIds.length} items...") >>
    popularIds
      .parTraverseN(20)(id => cache.get(id).void)  // This loads into cache
      .flatMap(_ => IO.println("Cache warming complete"))

  // Continuous refresh for hot keys
  def refreshHotKeys(
    getHotKeys: IO[List[String]],
    refreshInterval: FiniteDuration = 30.seconds
  ): Stream[IO, Unit] =
    Stream
      .awakeEvery[IO](refreshInterval)
      .evalMap: _ =>
        for
          hotKeys <- getHotKeys
          _       <- IO.println(s"Refreshing ${hotKeys.length} hot cache keys")
          _       <- hotKeys.parTraverseN(10): key =>
                       cache.invalidate(key) >>  // Force refresh
                       cache.get(key).void
        yield ()
```

---

## Testing Cache

```scala
import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.freespec.AsyncFreeSpec
import org.scalatest.matchers.should.Matchers
import scala.concurrent.duration.*

class CacheSpec extends AsyncFreeSpec with AsyncIOSpec with Matchers:

  "LRUCache" - {

    "should evict least recently used when at capacity" in {
      LRUCache.create[String, Int](3).flatMap: cache =>
        for
          _  <- cache.put("a", 1)
          _  <- cache.put("b", 2)
          _  <- cache.put("c", 3)
          _  <- cache.get("a")    // Access "a" to make it recent
          _  <- cache.put("d", 4)  // Should evict "b" (LRU)
          b  <- cache.get("b")
          a  <- cache.get("a")
        yield
          b shouldBe None    // "b" was evicted
          a shouldBe Some(1) // "a" still there
    }
  }

  "TtlCache" - {

    "should expire entries after TTL" in {
      TtlCache.create[String, String](100.millis).flatMap: cache =>
        for
          _       <- cache.put("key", "value")
          before  <- cache.get("key")
          _       <- IO.sleep(200.millis)
          after   <- cache.get("key")
        yield
          before shouldBe Some("value")
          after  shouldBe None  // Expired!
    }
  }

  "Memoization" - {

    "should call function only once for same input" in {
      Ref.of[IO, Int](0).flatMap: counter =>
        val expensive = (n: Int) => counter.update(_ + 1) >> IO.pure(n * 2)

        memoize(expensive).flatMap: memoized =>
          for
            r1    <- memoized(5)
            r2    <- memoized(5)
            r3    <- memoized(5)
            count <- counter.get
          yield
            r1 shouldBe 10
            r2 shouldBe 10
            r3 shouldBe 10
            count shouldBe 1  // Called only once!
    }
  }
```

---

## สรุป

Caching Strategies ที่เลือกใช้ควรขึ้นอยู่กับ use case:

| Pattern | เหมาะกับ | ข้อระวัง |
|---------|---------|---------|
| **Cache-Aside** | Read-heavy, ยอมรับ stale data | First request หลัง expiry จะช้า |
| **Write-Through** | ต้องการ strong consistency | Write latency สูงขึ้น |
| **Write-Behind** | Write-heavy, ยอมรับ eventual | Risk data loss on crash |
| **Multi-Level** | Low latency + large capacity | ความซับซ้อนสูง |

**Cache Invalidation Strategies:**
1. **TTL-based** - Simple แต่อาจ serve stale data
2. **Event-driven** - Accurate แต่ต้องมี event system
3. **Version-based** - Fast bulk invalidation

**Anti-Patterns ที่ต้องหลีกเลี่ยง:**
- Cache ข้อมูลที่เปลี่ยนบ่อยมากโดยไม่มีกลยุทธ์ invalidation ที่ดี
- ไม่ cache negative results (ทำให้ DB ถูกโจมตีด้วย non-existent keys)
- ไม่มี stampede prevention ในระบบ high-traffic
- Cache ข้อมูลที่ sensitive โดยไม่เข้ารหัส

**Checklist Production:**
- Monitor cache hit rate (เป้าหมาย > 90%)
- ตั้ง memory limits ที่เหมาะสมบน Caffeine/Redis
- ใช้ probabilistic early expiration หรือ mutex สำหรับ hot keys
- Implement cache warming สำหรับ cold start
- Log และ alert เมื่อ hit rate ต่ำผิดปกติ

---

*[← Part 53: Rate Limiting](part-53-rate-limiting.md) | [Part 55: Advanced Testing →](part-55-advanced-testing.md)*
