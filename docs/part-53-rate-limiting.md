# Part 53: Rate Limiting และ Throttling

## สารบัญ

1. [ทำไมต้องมี Rate Limiting](#ทำไมต้องมี-rate-limiting)
2. [Token Bucket Algorithm](#token-bucket-algorithm)
3. [Sliding Window Counter](#sliding-window-counter)
4. [Fixed Window Counter](#fixed-window-counter)
5. [Distributed Rate Limiting ด้วย Redis](#distributed-rate-limiting-ด้วย-redis)
6. [http4s Middleware สำหรับ Rate Limiting](#http4s-middleware-สำหรับ-rate-limiting)
7. [Per-User และ Global Rate Limits](#per-user-และ-global-rate-limits)
8. [Rate Limit Headers](#rate-limit-headers)
9. [Testing Rate Limiters](#testing-rate-limiters)

---

## ทำไมต้องมี Rate Limiting

Rate limiting ปกป้อง API จากการใช้งานเกินขอบเขต:

```
ไม่มี Rate Limiting:                  มี Rate Limiting:
                                      
[Client 1] ──────── 1000 req/s ──▶   [Client 1] ─── 100 req/s ──▶
[Client 2] ──────── 5000 req/s ──▶   [Client 2] ─ ❌ 429 Too Many Requests
[Client 3] ─────────  500 req/s ──▶   [Client 3] ─── 100 req/s ──▶
                        │                              │
                   ┌────▼────┐                   ┌────▼────┐
                   │ Server  │                   │ Server  │
                   │ 💥 Down │                   │   OK   │
                   └─────────┘                   └─────────┘
```

### ประเภท Rate Limiting

| Algorithm | ลักษณะ | เหมาะกับ |
|-----------|--------|---------|
| Token Bucket | Smooth burst allowance | APIs ทั่วไป |
| Sliding Window | ความแม่นยำสูง | เมื่อต้องการ exact limits |
| Fixed Window | Simple, efficient | ระบบ count-based |
| Leaky Bucket | Constant rate output | Queuing systems |

### Dependencies

```scala
libraryDependencies ++= Seq(
  "org.typelevel"   %% "cats-effect"    % "3.5.4",
  "org.http4s"      %% "http4s-dsl"     % "0.23.27",
  "org.http4s"      %% "http4s-ember-server" % "0.23.27",
  "dev.profunktor"  %% "redis4cats-effects"  % "1.7.0",
  "dev.profunktor"  %% "redis4cats-log4cats" % "1.7.0",
  "io.github.valskalla" %% "odin-core"   % "0.13.0"
)
```

---

## Token Bucket Algorithm

Token Bucket เปรียบเหมือนถังที่มีโทเค็น โทเค็นเติมเข้ามาสม่ำเสมอ แต่ละ request ใช้โทเค็น 1 อัน ถ้าถังว่างก็ reject:

```
Token Bucket Visualization:

Time 0s:  [●●●●●●●●●●] ← Bucket full (10 tokens)
           ↓ 5 requests consumed
Time 1s:  [●●●●●·····] ← 5 tokens left
           ↑ 2 tokens added (rate = 2/s)
Time 2s:  [●●●●●●●····] ← 7 tokens
```

### In-Memory Token Bucket

```scala
import cats.effect.*
import cats.effect.Ref
import scala.concurrent.duration.*

case class TokenBucketConfig(
  capacity: Int,           // Maximum tokens
  refillRate: Int,         // Tokens added per second
  initialTokens: Int       // Starting tokens
)

class TokenBucket(
  config: TokenBucketConfig,
  tokens: Ref[IO, Double],
  lastRefillTime: Ref[IO, Long]
):

  // Try to acquire n tokens
  def tryAcquire(n: Int = 1): IO[Boolean] =
    for
      _       <- refill
      result  <- tokens.modify: current =>
                   if current >= n then (current - n, true)
                   else (current, false)
    yield result

  // Acquire tokens, waiting if necessary
  def acquire(n: Int = 1): IO[Unit] =
    tryAcquire(n).flatMap:
      case true  => IO.unit
      case false =>
        val waitTime = (n.toDouble / config.refillRate * 1000).toLong.millis
        IO.sleep(waitTime) >> acquire(n)

  // Refill tokens based on elapsed time
  private def refill: IO[Unit] =
    for
      now         <- IO.realTime.map(_.toMillis)
      lastRefill  <- lastRefillTime.getAndSet(now)
      elapsedSec   = (now - lastRefill).toDouble / 1000.0
      newTokens    = elapsedSec * config.refillRate
      _           <- tokens.update: current =>
                       math.min(config.capacity.toDouble, current + newTokens)
    yield ()

  def availableTokens: IO[Int] =
    tokens.get.map(_.toInt)

object TokenBucket:
  def create(config: TokenBucketConfig): IO[TokenBucket] =
    for
      tokens    <- Ref.of[IO, Double](config.initialTokens.toDouble)
      lastFill  <- IO.realTime.map(_.toMillis).flatMap(Ref.of[IO, Long])
    yield new TokenBucket(config, tokens, lastFill)

  def default: IO[TokenBucket] =
    create(TokenBucketConfig(
      capacity      = 100,
      refillRate    = 10,
      initialTokens = 100
    ))
```

### Token Bucket Usage

```scala
import cats.effect.*
import scala.concurrent.duration.*

object TokenBucketExample extends IOApp.Simple:

  def run: IO[Unit] =
    TokenBucket.create(
      TokenBucketConfig(capacity = 5, refillRate = 2, initialTokens = 5)
    ).flatMap: bucket =>

      val makeRequest = (i: Int) =>
        bucket.tryAcquire(1).flatMap:
          case true  => IO.println(s"Request $i: ✓ Allowed")
          case false => IO.println(s"Request $i: ✗ Rate limited")

      // Send 10 requests quickly
      (1 to 10).toList.traverse_(makeRequest)
```

---

## Sliding Window Counter

Sliding Window ติดตาม requests ภายใน time window ที่เลื่อนไปตามเวลา:

```
Sliding Window (size = 60s):

Timeline: ────────────────────────────────────────────────▶
           │               │               │
          t-60s           t-30s           t (now)
           │←────── window (60s) ─────────→│
           
          Requests in window: 45
          Limit: 100
          → ALLOWED
          
After 20 more requests:
          Requests in window: 65
          → ALLOWED
          
After 36 more requests:  
          Requests in window: 101
          → DENIED (429)
```

```scala
import cats.effect.*
import cats.effect.Ref
import scala.collection.mutable
import java.time.Instant

class SlidingWindowRateLimiter(
  windowSize: FiniteDuration,
  maxRequests: Int,
  timestamps: Ref[IO, Vector[Long]]
):

  def isAllowed: IO[Boolean] =
    IO.realTime.flatMap: now =>
      val nowMillis    = now.toMillis
      val windowStart  = nowMillis - windowSize.toMillis

      timestamps.modify: ts =>
        // Remove expired timestamps
        val relevant = ts.filter(_ > windowStart)
        if relevant.length < maxRequests then
          // Allow: add current timestamp
          (relevant :+ nowMillis, true)
        else
          // Deny: window is full
          (relevant, false)

  def requestsInWindow: IO[Int] =
    IO.realTime.flatMap: now =>
      val nowMillis   = now.toMillis
      val windowStart = nowMillis - windowSize.toMillis
      timestamps.get.map(_.count(_ > windowStart))

  def timeUntilNextSlot: IO[FiniteDuration] =
    IO.realTime.flatMap: now =>
      val nowMillis   = now.toMillis
      val windowStart = nowMillis - windowSize.toMillis
      timestamps.get.map: ts =>
        val relevant = ts.filter(_ > windowStart)
        if relevant.length < maxRequests then
          Duration.Zero
        else
          val oldest = relevant.min
          val wait   = (oldest + windowSize.toMillis) - nowMillis
          wait.max(0).millis

object SlidingWindowRateLimiter:
  def create(
    windowSize: FiniteDuration,
    maxRequests: Int
  ): IO[SlidingWindowRateLimiter] =
    Ref.of[IO, Vector[Long]](Vector.empty).map: timestamps =>
      new SlidingWindowRateLimiter(windowSize, maxRequests, timestamps)
```

---

## Fixed Window Counter

Simple แต่มีปัญหา "boundary burst":

```scala
import cats.effect.*
import cats.effect.Ref

class FixedWindowCounter(
  windowSize: FiniteDuration,
  maxRequests: Int,
  state: Ref[IO, (Long, Int)]  // (windowStart, count)
):

  def isAllowed: IO[Boolean] =
    IO.realTime.flatMap: now =>
      val nowMillis = now.toMillis

      state.modify: (windowStart, count) =>
        val windowEnd = windowStart + windowSize.toMillis

        if nowMillis >= windowEnd then
          // New window
          (nowMillis, 1) -> true
        else if count < maxRequests then
          // Same window, under limit
          (windowStart, count + 1) -> true
        else
          // Same window, over limit
          (windowStart, count) -> false

object FixedWindowCounter:
  def create(windowSize: FiniteDuration, maxRequests: Int): IO[FixedWindowCounter] =
    IO.realTime.flatMap: now =>
      Ref.of[IO, (Long, Int)]((now.toMillis, 0)).map: state =>
        new FixedWindowCounter(windowSize, maxRequests, state)
```

---

## Distributed Rate Limiting ด้วย Redis

สำหรับ multi-instance deployments ต้องใช้ Redis เป็น shared state:

### Redis Token Bucket

```scala
import cats.effect.*
import dev.profunktor.redis4cats.RedisCommands
import dev.profunktor.redis4cats.effects.*
import scala.concurrent.duration.*

class RedisTokenBucket(
  redis: RedisCommands[IO, String, String],
  keyPrefix: String = "rate_limit:token_bucket"
):

  // Lua script for atomic token bucket operation
  private val luaScript = """
    local key = KEYS[1]
    local capacity = tonumber(ARGV[1])
    local refillRate = tonumber(ARGV[2])
    local requested = tonumber(ARGV[3])
    local now = tonumber(ARGV[4])
    
    local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
    local tokens = tonumber(bucket[1]) or capacity
    local lastRefill = tonumber(bucket[2]) or now
    
    -- Refill tokens based on elapsed time
    local elapsed = (now - lastRefill) / 1000.0  -- Convert to seconds
    local newTokens = math.min(capacity, tokens + elapsed * refillRate)
    
    if newTokens >= requested then
      -- Allow request
      redis.call('HMSET', key, 
        'tokens', newTokens - requested,
        'last_refill', now
      )
      redis.call('EXPIRE', key, math.ceil(capacity / refillRate) + 10)
      return {1, math.floor(newTokens - requested)}
    else
      -- Deny request
      redis.call('HMSET', key, 'tokens', newTokens, 'last_refill', now)
      redis.call('EXPIRE', key, math.ceil(capacity / refillRate) + 10)
      return {0, math.floor(newTokens)}
    end
  """

  def tryAcquire(
    userId: String,
    capacity: Int = 100,
    refillRate: Int = 10,
    tokens: Int = 1
  ): IO[RateLimitResult] =
    for
      now    <- IO.realTime.map(_.toMillis)
      key     = s"$keyPrefix:$userId"
      result <- redis.eval(
                  luaScript,
                  List(key),
                  List(capacity.toString, refillRate.toString, tokens.toString, now.toString)
                )
    yield result match
      case List("1", remaining) =>
        RateLimitResult.Allowed(remaining.toInt)
      case List("0", remaining) =>
        val waitMs = ((tokens - remaining.toInt).toDouble / refillRate * 1000).toLong
        RateLimitResult.Denied(remaining.toInt, waitMs.millis)
      case _ =>
        RateLimitResult.Denied(0, 1.second)

sealed trait RateLimitResult
object RateLimitResult:
  case class Allowed(remainingTokens: Int) extends RateLimitResult
  case class Denied(remainingTokens: Int, retryAfter: FiniteDuration) extends RateLimitResult
```

### Redis Sliding Window

```scala
import cats.effect.*
import dev.profunktor.redis4cats.RedisCommands
import scala.concurrent.duration.*

class RedisSlidingWindow(
  redis: RedisCommands[IO, String, String]
):

  // Use sorted set: score = timestamp, member = unique request ID
  def isAllowed(
    key: String,
    windowSize: FiniteDuration,
    maxRequests: Int
  ): IO[RateLimitResult] =
    for
      now         <- IO.realTime.map(_.toMillis)
      windowStart  = now - windowSize.toMillis
      requestId    = s"$now-${java.util.UUID.randomUUID()}"

      // 1. Remove expired entries
      _           <- redis.zRemRangeByScore(key, ScoreRange.from(0).to(windowStart))

      // 2. Count current requests in window
      count       <- redis.zCard(key)

      result      <- if count < maxRequests then
                       // Allow: add new entry
                       redis.zAdd(key, ZArgs.NX, ScoreWithValue(Score(now.toDouble), requestId)) >>
                       redis.expire(key, windowSize).as:
                         RateLimitResult.Allowed((maxRequests - count - 1).toInt)
                     else
                       // Deny: get oldest entry to compute retry-after
                       redis.zRange(key, 0, 0).map: oldest =>
                         val retryAfterMs = oldest.headOption.fold(0L): ts =>
                           val oldestTime = ts.toLongOption.getOrElse(now)
                           (oldestTime + windowSize.toMillis) - now
                         RateLimitResult.Denied(0, retryAfterMs.max(0).millis)
    yield result
```

---

## http4s Middleware สำหรับ Rate Limiting

### Basic Rate Limit Middleware

```scala
import cats.effect.*
import cats.data.Kleisli
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.headers.*

object RateLimitMiddleware:

  def apply(
    rateLimiter: String => IO[RateLimitResult],
    extractKey: Request[IO] => String = defaultKeyExtractor
  ): HttpMiddleware[IO] =
    service => Kleisli: req =>
      val key = extractKey(req)
      rateLimiter(key).flatMap:
        case RateLimitResult.Allowed(remaining) =>
          service(req).map: resp =>
            resp
              .putHeaders(
                Header.Raw(ci"X-RateLimit-Remaining", remaining.toString),
                Header.Raw(ci"X-RateLimit-Limit", "100")
              )

        case RateLimitResult.Denied(remaining, retryAfter) =>
          TooManyRequests(
            s"Rate limit exceeded. Retry after ${retryAfter.toSeconds}s"
          ).map: resp =>
            resp.putHeaders(
              Header.Raw(ci"X-RateLimit-Remaining", "0"),
              Header.Raw(ci"Retry-After", retryAfter.toSeconds.toString),
              Header.Raw(ci"X-RateLimit-Reset", retryAfterEpoch(retryAfter).toString)
            )

  // Extract rate limit key from request
  private def defaultKeyExtractor(req: Request[IO]): String =
    req.headers
      .get(ci"X-Forwarded-For")
      .map(_.head.value)
      .orElse(req.remoteAddr.map(_.toString))
      .getOrElse("unknown")

  private def retryAfterEpoch(retryAfter: FiniteDuration): Long =
    (System.currentTimeMillis() + retryAfter.toMillis) / 1000

type HttpMiddleware[F[_]] = HttpRoutes[F] => HttpRoutes[F]
```

### Multi-Strategy Rate Limiter

```scala
import cats.effect.*
import cats.syntax.all.*
import org.http4s.*
import org.http4s.dsl.io.*

// Different strategies for different use cases
case class RateLimitStrategy(
  windowSize: FiniteDuration,
  maxRequests: Int,
  scope: LimitScope
)

enum LimitScope:
  case Global               // All requests
  case PerUser              // Per authenticated user
  case PerIp                // Per IP address
  case PerEndpoint(path: String)  // Per API endpoint

class MultiStrategyRateLimiter(
  strategies: List[RateLimitStrategy],
  redis: dev.profunktor.redis4cats.RedisCommands[IO, String, String]
):

  def check(req: Request[IO]): IO[Option[RateLimitResult.Denied]] =

    val checks: List[IO[Option[RateLimitResult.Denied]]] =
      strategies.map: strategy =>
        val key = buildKey(req, strategy)
        checkStrategy(key, strategy).map:
          case denied: RateLimitResult.Denied => Some(denied)
          case _: RateLimitResult.Allowed     => None

    // Check all strategies, return first denial
    checks.foldLeft(IO.pure(Option.empty[RateLimitResult.Denied])): (acc, check) =>
      acc.flatMap:
        case Some(denied) => IO.pure(Some(denied))  // Already denied
        case None         => check

  private def buildKey(req: Request[IO], strategy: RateLimitStrategy): String =
    strategy.scope match
      case LimitScope.Global =>
        s"rate_limit:global"
      case LimitScope.PerIp =>
        val ip = req.remoteAddr.map(_.toString).getOrElse("unknown")
        s"rate_limit:ip:$ip"
      case LimitScope.PerUser =>
        val userId = extractUserId(req).getOrElse("anonymous")
        s"rate_limit:user:$userId"
      case LimitScope.PerEndpoint(path) =>
        s"rate_limit:endpoint:${path.replace("/", "_")}"

  private def extractUserId(req: Request[IO]): Option[String] =
    req.headers.get(ci"X-User-Id").map(_.head.value)

  private def checkStrategy(key: String, strategy: RateLimitStrategy): IO[RateLimitResult] =
    val slidingWindow = new RedisSlidingWindow(redis)
    slidingWindow.isAllowed(key, strategy.windowSize, strategy.maxRequests)

// Middleware using multi-strategy
def multiStrategyMiddleware(
  rateLimiter: MultiStrategyRateLimiter
): HttpMiddleware[IO] =
  service => Kleisli: req =>
    rateLimiter.check(req).flatMap:
      case Some(denied) =>
        TooManyRequests(s"Rate limit exceeded").map:
          _.putHeaders(
            Header.Raw(ci"Retry-After", denied.retryAfter.toSeconds.toString)
          )
      case None =>
        service(req)
```

---

## Per-User และ Global Rate Limits

### Tiered Rate Limiting

```scala
import cats.effect.*
import cats.effect.Ref
import scala.concurrent.duration.*

// User tiers with different limits
enum UserTier:
  case Free
  case Basic
  case Premium
  case Enterprise

case class TierLimits(
  requestsPerMinute: Int,
  requestsPerHour: Int,
  requestsPerDay: Int,
  burstCapacity: Int
)

object TierLimits:
  val forTier: Map[UserTier, TierLimits] = Map(
    UserTier.Free       -> TierLimits(60, 1000, 10000, 10),
    UserTier.Basic      -> TierLimits(300, 5000, 50000, 50),
    UserTier.Premium    -> TierLimits(1000, 20000, 200000, 200),
    UserTier.Enterprise -> TierLimits(10000, 200000, 2000000, 2000)
  )

class TieredRateLimiter(
  redis: dev.profunktor.redis4cats.RedisCommands[IO, String, String]
):

  def checkAllLimits(userId: String, tier: UserTier): IO[RateLimitResult] =
    val limits = TierLimits.forTier.getOrElse(tier, TierLimits.forTier(UserTier.Free))
    val window = new RedisSlidingWindow(redis)

    for
      minuteResult <- window.isAllowed(s"rl:$userId:minute", 1.minute, limits.requestsPerMinute)
      hourResult   <- minuteResult match
                        case _: RateLimitResult.Denied => IO.pure(minuteResult)
                        case _ => window.isAllowed(s"rl:$userId:hour", 1.hour, limits.requestsPerHour)
      dayResult    <- hourResult match
                        case _: RateLimitResult.Denied => IO.pure(hourResult)
                        case _ => window.isAllowed(s"rl:$userId:day", 24.hours, limits.requestsPerDay)
    yield dayResult

  def getRemainingQuota(userId: String, tier: UserTier): IO[QuotaInfo] =
    val limits = TierLimits.forTier.getOrElse(tier, TierLimits.forTier(UserTier.Free))
    for
      minuteCount <- getCount(s"rl:$userId:minute", 1.minute)
      hourCount   <- getCount(s"rl:$userId:hour", 1.hour)
      dayCount    <- getCount(s"rl:$userId:day", 24.hours)
    yield QuotaInfo(
      perMinute = QuotaWindow(limits.requestsPerMinute, minuteCount),
      perHour   = QuotaWindow(limits.requestsPerHour, hourCount),
      perDay    = QuotaWindow(limits.requestsPerDay, dayCount)
    )

  private def getCount(key: String, window: FiniteDuration): IO[Int] =
    for
      now         <- IO.realTime.map(_.toMillis)
      windowStart  = now - window.toMillis
      _           <- redis.zRemRangeByScore(key, ScoreRange.from(0).to(windowStart))
      count       <- redis.zCard(key)
    yield count.toInt

case class QuotaWindow(limit: Int, used: Int):
  def remaining: Int = (limit - used).max(0)
  def percentUsed: Double = if limit > 0 then used.toDouble / limit * 100 else 0

case class QuotaInfo(
  perMinute: QuotaWindow,
  perHour: QuotaWindow,
  perDay: QuotaWindow
)
```

---

## Rate Limit Headers

### Standard Rate Limit Headers

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.headers.*
import scala.concurrent.duration.*

// Standard headers per RFC 6585 and draft-ietf-httpapi-ratelimit-headers
object RateLimitHeaders:

  // Headers for allowed response
  def allowedHeaders(
    limit: Int,
    remaining: Int,
    resetTime: Long,  // Unix timestamp
    policy: String = "default"
  ): List[Header.Raw] = List(
    Header.Raw(ci"X-RateLimit-Limit",     limit.toString),
    Header.Raw(ci"X-RateLimit-Remaining", remaining.toString),
    Header.Raw(ci"X-RateLimit-Reset",     resetTime.toString),
    Header.Raw(ci"X-RateLimit-Policy",    policy),
    Header.Raw(ci"X-RateLimit-Window",    "60")  // seconds
  )

  // Headers for denied response (429)
  def deniedHeaders(
    limit: Int,
    retryAfterSecs: Long,
    resetTime: Long
  ): List[Header.Raw] = List(
    Header.Raw(ci"X-RateLimit-Limit",     limit.toString),
    Header.Raw(ci"X-RateLimit-Remaining", "0"),
    Header.Raw(ci"X-RateLimit-Reset",     resetTime.toString),
    Header.Raw(ci"Retry-After",           retryAfterSecs.toString)
  )

  // Quota headers (for daily/monthly limits)
  def quotaHeaders(quota: QuotaInfo): List[Header.Raw] = List(
    Header.Raw(ci"X-RateLimit-Limit",     quota.perMinute.limit.toString),
    Header.Raw(ci"X-RateLimit-Remaining", quota.perMinute.remaining.toString),
    Header.Raw(ci"X-Quota-Limit-Day",     quota.perDay.limit.toString),
    Header.Raw(ci"X-Quota-Remaining-Day", quota.perDay.remaining.toString),
    Header.Raw(ci"X-Quota-Limit-Hour",    quota.perHour.limit.toString),
    Header.Raw(ci"X-Quota-Remaining-Hour",quota.perHour.remaining.toString)
  )

// Enhanced middleware with quota headers
def rateLimitWithQuotaMiddleware(
  rateLimiter: TieredRateLimiter,
  getUserTier: String => IO[UserTier]
): HttpMiddleware[IO] =
  service => Kleisli: req =>
    extractUserId(req) match
      case None =>
        Forbidden("Authentication required")

      case Some(userId) =>
        getUserTier(userId).flatMap: tier =>
          rateLimiter.checkAllLimits(userId, tier).flatMap:
            case RateLimitResult.Allowed(_) =>
              rateLimiter.getRemainingQuota(userId, tier).flatMap: quota =>
                service(req).map: resp =>
                  resp.putHeaders(RateLimitHeaders.quotaHeaders(quota)*)

            case RateLimitResult.Denied(_, retryAfter) =>
              val now       = System.currentTimeMillis() / 1000
              val resetTime = now + retryAfter.toSeconds
              val limits    = TierLimits.forTier.getOrElse(tier, TierLimits.forTier(UserTier.Free))

              TooManyRequests(s"Rate limit exceeded for tier $tier").map: resp =>
                resp.putHeaders(
                  RateLimitHeaders.deniedHeaders(limits.requestsPerMinute, retryAfter.toSeconds, resetTime)*
                )

def extractUserId(req: Request[IO]): Option[String] =
  req.headers.get(ci"X-User-Id").map(_.head.value)
```

---

## Complete Example: Rate-Limited API

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.ember.server.*
import com.comcast.ip4s.*
import scala.concurrent.duration.*

object RateLimitedApiServer extends IOApp:

  // API routes
  val apiRoutes: HttpRoutes[IO] = HttpRoutes.of[IO]:
    case GET -> Root / "api" / "products" =>
      Ok("""[{"id": 1, "name": "Product 1"}]""")

    case GET -> Root / "api" / "users" / userId =>
      Ok(s"""{"id": "$userId", "name": "User $userId"}""")

    case POST -> Root / "api" / "orders" =>
      Created("""{"orderId": "ord-123"}""")

  // Rate limit configuration
  val rateLimitStrategies = List(
    RateLimitStrategy(1.minute, 60, LimitScope.Global),       // 60 req/min globally
    RateLimitStrategy(1.minute, 10, LimitScope.PerIp),        // 10 req/min per IP
    RateLimitStrategy(1.hour, 500, LimitScope.PerUser)        // 500 req/hour per user
  )

  def run(args: List[String]): IO[ExitCode] =

    // Build Redis client
    import dev.profunktor.redis4cats.*
    import dev.profunktor.redis4cats.effect.Log.Stdout.*

    val redisResource = Redis[IO].utf8("redis://localhost")

    redisResource.use: redis =>
      val slidingWindow   = new RedisSlidingWindow(redis)
      val tieredLimiter   = new TieredRateLimiter(redis)

      // Simple rate limit middleware
      val simpleMiddleware: HttpMiddleware[IO] = service =>
        Kleisli: req =>
          val ip = req.remoteAddr.map(_.toString).getOrElse("unknown")
          slidingWindow.isAllowed(s"rl:ip:$ip", 1.minute, 60).flatMap:
            case RateLimitResult.Allowed(remaining) =>
              service(req).map:
                _.putHeaders(
                  Header.Raw(ci"X-RateLimit-Remaining", remaining.toString),
                  Header.Raw(ci"X-RateLimit-Limit", "60")
                )
            case RateLimitResult.Denied(_, retryAfter) =>
              TooManyRequests(s"Rate limit exceeded. Retry in ${retryAfter.toSeconds}s")

      // Apply middleware to routes
      val app: HttpApp[IO] = simpleMiddleware(apiRoutes).orNotFound

      EmberServerBuilder
        .default[IO]
        .withHost(ipv4"0.0.0.0")
        .withPort(port"8080")
        .withHttpApp(app)
        .build
        .use(_ => IO.never)
        .as(ExitCode.Success)
```

---

## Testing Rate Limiters

```scala
import cats.effect.*
import cats.effect.testing.scalatest.AsyncIOSpec
import org.scalatest.freespec.AsyncFreeSpec
import org.scalatest.matchers.should.Matchers
import scala.concurrent.duration.*

class RateLimiterSpec extends AsyncFreeSpec with AsyncIOSpec with Matchers:

  "TokenBucket" - {

    "should allow requests when tokens available" in {
      TokenBucket
        .create(TokenBucketConfig(5, 1, 5))
        .flatMap: bucket =>
          (1 to 5).toList.traverse(_ => bucket.tryAcquire())
        .asserting: results =>
          results.forall(identity) shouldBe true
    }

    "should deny requests when bucket is empty" in {
      TokenBucket
        .create(TokenBucketConfig(3, 1, 3))
        .flatMap: bucket =>
          for
            _ <- (1 to 3).toList.traverse(_ => bucket.tryAcquire())
            r <- bucket.tryAcquire()
          yield r
        .asserting: result =>
          result shouldBe false
    }

    "should refill tokens over time" in {
      TokenBucket
        .create(TokenBucketConfig(5, 5, 0))  // Empty bucket, 5 tokens/sec
        .flatMap: bucket =>
          for
            first     <- bucket.tryAcquire()  // Should fail - empty
            _         <- IO.sleep(300.millis)  // Wait for ~1.5 tokens
            second    <- bucket.tryAcquire()   // Should succeed
          yield (first, second)
        .asserting: (first, second) =>
          first shouldBe false
          second shouldBe true
    }
  }

  "SlidingWindowRateLimiter" - {

    "should allow requests within limit" in {
      SlidingWindowRateLimiter
        .create(1.minute, 10)
        .flatMap: limiter =>
          (1 to 10).toList.traverse(_ => limiter.isAllowed)
        .asserting: results =>
          results.forall(identity) shouldBe true
    }

    "should deny when limit exceeded" in {
      SlidingWindowRateLimiter
        .create(1.minute, 3)
        .flatMap: limiter =>
          (1 to 4).toList.traverse(_ => limiter.isAllowed)
        .asserting: results =>
          results.count(identity) shouldBe 3
          results.last shouldBe false
    }
  }
```

---

## สรุป

Rate Limiting เป็นส่วนสำคัญของทุก API ที่ให้บริการ:

| Algorithm | Memory | Precision | Burst Handling |
|-----------|--------|-----------|----------------|
| Token Bucket | O(1) | Medium | ดี (ถังสำรอง) |
| Sliding Window | O(N) | สูงมาก | ปานกลาง |
| Fixed Window | O(1) | ต่ำ (boundary issue) | ไม่ดี |

**Checklist สำหรับ Production:**

- ใช้ Redis สำหรับ distributed rate limiting ใน multi-instance setup
- ส่ง proper HTTP headers (`Retry-After`, `X-RateLimit-*`)
- Implement tiered limits ตาม user plan
- Log rate limit violations เพื่อ monitoring
- Set up alerts สำหรับ mass rate limit violations (อาจเป็น DDoS)
- Test rate limiter อย่างละเอียด รวมถึง edge cases

**Gotchas:**
- Redis cluster partitioning อาจทำให้ sliding window ไม่ accurate 100%
- ใช้ Lua scripts ใน Redis เพื่อ atomic operations
- Fixed window มีปัญหา "double the rate" ที่ window boundary

---

*[← Part 52: Outbox Pattern](part-52-outbox-pattern.md) | [Part 54: Caching Strategies →](part-54-caching-strategies.md)*
