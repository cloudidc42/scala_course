# Part 44: Security ใน Scala

## สารบัญ
1. [Security Overview](#security-overview)
2. [Authentication](#authentication)
3. [Authorization](#authorization)
4. [Input Validation](#input-validation)
5. [Cryptography](#cryptography)
6. [Secure API Design](#secure-api-design)

---

## Security Overview

### OWASP Top 10 ใน Scala

```
Web Application Security (OWASP Top 10):
1. Injection (SQL, LDAP, OS command)
2. Broken Authentication
3. Sensitive Data Exposure
4. XML External Entities (XXE)
5. Broken Access Control
6. Security Misconfiguration
7. Cross-Site Scripting (XSS)
8. Insecure Deserialization
9. Using Components with Known Vulnerabilities
10. Insufficient Logging & Monitoring

Scala/FP advantages:
- Immutability prevents state mutation bugs
- Type system catches many errors at compile time
- Pure functions easier to audit and test
- Strong types prevent injection
```

---

## Authentication

### JWT Authentication

```scala
import cats.effect.IO
import io.circe.generic.auto.*
import io.circe.syntax.*
import io.circe.parser.*
import java.util.Base64
import javax.crypto.Mac
import javax.crypto.spec.SecretKeySpec
import java.time.Instant

// JWT claims
case class JwtClaims(
  sub: String,        // subject (user id)
  iat: Long,          // issued at
  exp: Long,          // expiration
  roles: List[String] = Nil
)

object JWT:
  private def hmac256(secret: String, data: String): String =
    val mac = Mac.getInstance("HmacSHA256")
    mac.init(SecretKeySpec(secret.getBytes("UTF-8"), "HmacSHA256"))
    Base64.getUrlEncoder.withoutPadding.encodeToString(mac.doFinal(data.getBytes("UTF-8")))

  private def base64Encode(s: String): String =
    Base64.getUrlEncoder.withoutPadding.encodeToString(s.getBytes("UTF-8"))

  private def base64Decode(s: String): String =
    new String(Base64.getUrlDecoder.decode(s), "UTF-8")

  def create(claims: JwtClaims, secret: String): String =
    val header = """{"alg":"HS256","typ":"JWT"}"""
    val h = base64Encode(header)
    val p = base64Encode(claims.asJson.noSpaces)
    val sig = hmac256(secret, s"$h.$p")
    s"$h.$p.$sig"

  def verify(token: String, secret: String): Either[String, JwtClaims] =
    token.split("\\.") match
      case Array(h, p, sig) =>
        val expectedSig = hmac256(secret, s"$h.$p")
        if sig != expectedSig then
          Left("Invalid signature")
        else
          decode[JwtClaims](base64Decode(p)).left.map(_.getMessage).flatMap { claims =>
            if claims.exp < Instant.now().getEpochSecond
            then Left("Token expired")
            else Right(claims)
          }
      case _ => Left("Invalid token format")

// Usage
val secret = "super-secret-key-change-in-production"
val claims = JwtClaims(
  sub   = "user-123",
  iat   = Instant.now().getEpochSecond,
  exp   = Instant.now().plusSeconds(3600).getEpochSecond,
  roles = List("user", "admin")
)

val token = JWT.create(claims, secret)
println(s"Token: $token")

val verified = JWT.verify(token, secret)
println(s"Verified: $verified")
```

### Password Hashing

```scala
import cats.effect.IO

// Use BCrypt for password hashing
// build.sbt: "org.mindrot" % "jbcrypt" % "0.4"
import org.mindrot.jbcrypt.BCrypt

object PasswordService:
  // Cost factor: higher = slower (more secure)
  private val COST = 12

  def hash(plaintext: String): IO[String] = IO {
    BCrypt.hashpw(plaintext, BCrypt.gensalt(COST))
  }

  def verify(plaintext: String, hashed: String): IO[Boolean] = IO {
    BCrypt.checkpw(plaintext, hashed)
  }

// Usage
val program = for
  hash  <- PasswordService.hash("my-secure-password")
  valid <- PasswordService.verify("my-secure-password", hash)
  wrong <- PasswordService.verify("wrong-password", hash)
  _     <- IO.println(s"Valid: $valid, Wrong: $wrong")
yield ()
```

---

## Authorization

### Role-Based Access Control

```scala
// RBAC
enum Permission:
  case Read, Write, Delete, Admin

case class Role(name: String, permissions: Set[Permission])

object Roles:
  val Guest  = Role("guest",  Set(Permission.Read))
  val User   = Role("user",   Set(Permission.Read, Permission.Write))
  val Mod    = Role("mod",    Set(Permission.Read, Permission.Write, Permission.Delete))
  val Admin  = Role("admin",  Permission.values.toSet)

case class AuthContext(userId: String, roles: List[Role]):
  def hasPermission(p: Permission): Boolean =
    roles.exists(_.permissions.contains(p))

  def requirePermission(p: Permission): Either[String, Unit] =
    if hasPermission(p) then Right(())
    else Left(s"Missing permission: $p")

// Secured operations
class ResourceService(auth: AuthContext):
  def read(id: String): Either[String, String] =
    for
      _ <- auth.requirePermission(Permission.Read)
    yield s"Resource $id content"

  def write(id: String, content: String): Either[String, Unit] =
    for
      _ <- auth.requirePermission(Permission.Write)
    yield println(s"Writing to $id: $content")

  def delete(id: String): Either[String, Unit] =
    for
      _ <- auth.requirePermission(Permission.Delete)
    yield println(s"Deleted $id")

// http4s middleware for auth
import org.http4s.*
import cats.effect.IO

def authMiddleware(secret: String): HttpRoutes[IO] => HttpRoutes[IO] = routes =>
  HttpRoutes.of[IO] { req =>
    req.headers.get(ci"Authorization").map(_.head.value) match
      case Some(bearer) if bearer.startsWith("Bearer ") =>
        val token = bearer.drop(7)
        JWT.verify(token, secret) match
          case Right(claims) =>
            val ctx = AuthContext(
              userId = claims.sub,
              roles  = claims.roles.flatMap {
                case "admin" => Some(Roles.Admin)
                case "user"  => Some(Roles.User)
                case "guest" => Some(Roles.Guest)
                case _       => None
              }
            )
            routes(req.withAttribute(AuthContext.key, ctx))
          case Left(err) =>
            IO.pure(Response(Status.Unauthorized).withEntity(s"Auth error: $err"))
      case _ =>
        IO.pure(Response(Status.Unauthorized).withEntity("Missing Authorization header"))
  }

object AuthContext:
  val key = org.http4s.AttributeKey[AuthContext]
```

---

## Input Validation

### Sanitization and Validation

```scala
import cats.data.ValidatedNel
import cats.syntax.validated.*

// Validated input
object Validators:
  type V[A] = ValidatedNel[String, A]

  def nonEmpty(field: String, value: String): V[String] =
    if value.trim.nonEmpty then value.trim.validNel
    else s"$field cannot be empty".invalidNel

  def minLength(field: String, value: String, min: Int): V[String] =
    if value.length >= min then value.validNel
    else s"$field must be at least $min characters".invalidNel

  def maxLength(field: String, value: String, max: Int): V[String] =
    if value.length <= max then value.validNel
    else s"$field must be at most $max characters".invalidNel

  def emailFormat(value: String): V[String] =
    val pattern = """^[^@\s]+@[^@\s]+\.[^@\s]{2,}$""".r
    if pattern.matches(value) then value.validNel
    else "Invalid email format".invalidNel

  def positiveInt(field: String, value: Int): V[Int] =
    if value > 0 then value.validNel
    else s"$field must be positive".invalidNel

  def range(field: String, value: Int, min: Int, max: Int): V[Int] =
    if value >= min && value <= max then value.validNel
    else s"$field must be between $min and $max".invalidNel

  // Prevent SQL injection: validate with whitelist
  def safeIdentifier(value: String): V[String] =
    val safe = """^[a-zA-Z][a-zA-Z0-9_]{0,63}$""".r
    if safe.matches(value) then value.validNel
    else s"Invalid identifier: $value".invalidNel

  // Sanitize HTML to prevent XSS
  def sanitizeHtml(html: String): String =
    html
      .replace("&", "&amp;")
      .replace("<", "&lt;")
      .replace(">", "&gt;")
      .replace("\"", "&quot;")
      .replace("'", "&#x27;")

// Usage
case class RegisterForm(username: String, email: String, age: Int, bio: String)

def validateRegisterForm(
  username: String,
  email: String,
  age: Int,
  bio: String
): ValidatedNel[String, RegisterForm] =
  import cats.syntax.apply.*
  (
    Validators.nonEmpty("Username", username)
      .andThen(u => Validators.minLength("Username", u, 3))
      .andThen(u => Validators.maxLength("Username", u, 20))
      .andThen(Validators.safeIdentifier),
    Validators.nonEmpty("Email", email)
      .andThen(Validators.emailFormat),
    Validators.range("Age", age, 13, 120),
    Validators.maxLength("Bio", Validators.sanitizeHtml(bio), 500).validNel
  ).mapN(RegisterForm.apply)
```

---

## Cryptography

### Encryption and Hashing

```scala
import javax.crypto.{Cipher, KeyGenerator, SecretKey}
import javax.crypto.spec.{GCMParameterSpec, SecretKeySpec}
import java.security.SecureRandom
import java.util.Base64

object Crypto:
  private val AES_GCM_ALGORITHM = "AES/GCM/NoPadding"
  private val KEY_SIZE = 256
  private val IV_SIZE = 12  // GCM recommended IV size
  private val TAG_SIZE = 128  // GCM auth tag size in bits
  private val rng = new SecureRandom()

  // Generate secure random key
  def generateKey(): SecretKey =
    val kg = KeyGenerator.getInstance("AES")
    kg.init(KEY_SIZE, rng)
    kg.generateKey()

  // Encrypt with AES-GCM (authenticated encryption)
  def encrypt(plaintext: String, key: SecretKey): String =
    val iv = new Array[Byte](IV_SIZE)
    rng.nextBytes(iv)

    val cipher = Cipher.getInstance(AES_GCM_ALGORITHM)
    cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(TAG_SIZE, iv))

    val encrypted = cipher.doFinal(plaintext.getBytes("UTF-8"))
    val combined = iv ++ encrypted
    Base64.getEncoder.encodeToString(combined)

  // Decrypt
  def decrypt(ciphertext: String, key: SecretKey): Either[String, String] =
    try
      val combined = Base64.getDecoder.decode(ciphertext)
      val iv = combined.take(IV_SIZE)
      val encrypted = combined.drop(IV_SIZE)

      val cipher = Cipher.getInstance(AES_GCM_ALGORITHM)
      cipher.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(TAG_SIZE, iv))

      val plaintext = cipher.doFinal(encrypted)
      Right(new String(plaintext, "UTF-8"))
    catch case e: Exception =>
      Left(s"Decryption failed: ${e.getMessage}")

  // Secure hash with SHA-256
  def hash(data: String): String =
    val digest = java.security.MessageDigest.getInstance("SHA-256")
    val bytes = digest.digest(data.getBytes("UTF-8"))
    bytes.map("%02x".format(_)).mkString

  // HMAC-SHA256 for message authentication
  def hmac(message: String, secret: String): String =
    val mac = javax.crypto.Mac.getInstance("HmacSHA256")
    mac.init(new SecretKeySpec(secret.getBytes("UTF-8"), "HmacSHA256"))
    val bytes = mac.doFinal(message.getBytes("UTF-8"))
    bytes.map("%02x".format(_)).mkString

// Secure secrets management
object Secrets:
  def fromEnv(name: String): Either[String, String] =
    Option(System.getenv(name)).toRight(s"Missing environment variable: $name")

  // Never log secrets!
  def mask(secret: String): String =
    if secret.length <= 4 then "***"
    else secret.take(2) + "*" * (secret.length - 4) + secret.takeRight(2)
```

---

## Secure API Design

### Security Headers and Best Practices

```scala
import org.http4s.*
import org.http4s.headers.*
import cats.effect.IO
import cats.data.Kleisli

// Security headers middleware
val securityHeaders: HttpRoutes[IO] => HttpRoutes[IO] = routes =>
  HttpRoutes.of[IO] { req =>
    routes(req).map { resp =>
      resp
        .putHeaders(
          Header.Raw(ci"X-Content-Type-Options", "nosniff"),
          Header.Raw(ci"X-Frame-Options", "DENY"),
          Header.Raw(ci"X-XSS-Protection", "1; mode=block"),
          Header.Raw(ci"Strict-Transport-Security", "max-age=31536000; includeSubDomains"),
          Header.Raw(ci"Content-Security-Policy",
            "default-src 'self'; script-src 'self'; style-src 'self'"),
          Header.Raw(ci"Referrer-Policy", "no-referrer"),
          Header.Raw(ci"Permissions-Policy", "geolocation=(), microphone=()")
        )
        // Remove server header (information disclosure)
        .removeHeader(ci"Server")
    }
  }

// Rate limiting
import cats.effect.Ref
import scala.concurrent.duration.*

class RateLimiter(
  counts: Ref[IO, Map[String, (Int, Long)]],
  maxRequests: Int = 100,
  windowMs: Long = 60000
):
  def check(clientIp: String): IO[Boolean] =
    val now = System.currentTimeMillis()
    counts.modify { m =>
      m.get(clientIp) match
        case Some((count, windowStart)) if now - windowStart < windowMs =>
          if count >= maxRequests then
            (m, false)  // rate limited
          else
            (m + (clientIp -> (count + 1, windowStart)), true)
        case _ =>
          (m + (clientIp -> (1, now)), true)
    }

// CORS configuration
import org.http4s.server.middleware.CORS

val corsPolicy = CORS.policy
  .withAllowOriginHost(Set("https://myapp.com", "https://api.myapp.com"))
  .withAllowMethodsIn(Set(Method.GET, Method.POST, Method.PUT, Method.DELETE))
  .withAllowHeadersIn(Set(ci"Authorization", ci"Content-Type"))
  .withMaxAge(1.day)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ JWT authentication: create, verify tokens
- ✅ Password hashing: BCrypt
- ✅ RBAC: Role-Based Access Control
- ✅ Input validation: preventing injection, XSS
- ✅ AES-GCM encryption, SHA-256 hashing, HMAC
- ✅ Security headers, rate limiting, CORS

---

*[← Part 43: Performance](part-43-performance.md) | [Part 45: Logging and Monitoring →](part-45-logging.md)*
