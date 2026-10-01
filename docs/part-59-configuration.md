# ส่วนที่ 59: Configuration Management

## สารบัญ

1. [Configuration Management คืออะไร](#configuration-management-คืออะไร)
2. [pureconfig สำหรับ Type-safe Config](#pureconfig-สำหรับ-type-safe-config)
3. [ciris สำหรับ Effectful Config](#ciris-สำหรับ-effectful-config)
4. [Config Validation และ Loading](#config-validation-และ-loading)
5. [Environment-specific Configs](#environment-specific-configs)
6. [Secret Management](#secret-management)
7. [Hot Reloading Config](#hot-reloading-config)
8. [Config ใน cats-effect Applications](#config-ใน-cats-effect-applications)
9. [Testing กับ Config](#testing-กับ-config)
10. [Best Practices](#best-practices)
11. [สรุป](#สรุป)

---

## Configuration Management คืออะไร

Configuration Management คือกระบวนการจัดการ settings ของ application ที่อาจเปลี่ยนแปลงระหว่าง environments โดยไม่ต้องเปลี่ยน code

### ปัญหาที่ Configuration Management แก้ไข

```
ไม่มี config management:
  app.conf แบบ hardcode:
    host = "localhost"        <- ต้องเปลี่ยนตาม environment
    password = "1234"         <- security risk!
    maxConnections = 100      <- อาจต้องปรับตาม load
  
  ต้องเปลี่ยน code และ redeploy ทุกครั้งที่เปลี่ยน config

มี config management:
  Dev:  .env.dev   + application.conf
  Test: .env.test  + application-test.conf
  Prod: Secrets Manager + environment variables
  
  Code อ่าน config ตาม environment โดยอัตโนมัติ
```

### Config Management Approaches

```
1. Config Files (.conf, .yaml, .properties)
   └─ Typesafe Config (HOCON)
   └─ pureconfig

2. Environment Variables
   └─ ciris
   └─ sys.env

3. External Config Systems
   └─ AWS Parameter Store / Secrets Manager
   └─ HashiCorp Vault
   └─ Kubernetes ConfigMaps / Secrets

4. Feature Flags
   └─ LaunchDarkly
   └─ Split.io
```

### Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  // Config files
  "com.github.pureconfig" %% "pureconfig-core"       % "0.17.4",
  "com.github.pureconfig" %% "pureconfig-cats-effect" % "0.17.4",
  
  // Effectful config
  "is.cir"  %% "ciris"       % "3.3.0",
  "is.cir"  %% "ciris-enumeratum" % "3.3.0",
  
  // Validation
  "org.typelevel" %% "cats-core" % "2.10.0",
  
  // Encryption for secrets
  "com.bouncy-castle" % "bcprov-jdk18on" % "1.77",
  
  // HTTP for remote config
  "org.http4s" %% "http4s-ember-client" % "0.23.23"
)
```

---

## pureconfig สำหรับ Type-safe Config

### Basic pureconfig Setup

```scala
import pureconfig.*
import pureconfig.generic.derivation.default.*

// กำหนด Config case classes
case class DatabaseConfig(
  host: String,
  port: Int,
  name: String,
  username: String,
  password: String,
  maxConnections: Int,
  minConnections: Int,
  connectionTimeout: Int
) derives ConfigReader

case class ServerConfig(
  host: String,
  port: Int,
  readTimeout: Int,
  writeTimeout: Int,
  maxConnections: Int
) derives ConfigReader

case class CacheConfig(
  host: String,
  port: Int,
  maxKeys: Int,
  ttlSeconds: Int,
  password: Option[String]
) derives ConfigReader

case class AppConfig(
  database: DatabaseConfig,
  server: ServerConfig,
  cache: CacheConfig,
  logLevel: String,
  environment: String
) derives ConfigReader

// อ่าน config จากไฟล์
object ConfigLoader:
  def load(): Either[ConfigReaderFailures, AppConfig] =
    ConfigSource.default.load[AppConfig]
  
  def loadOrThrow(): AppConfig =
    ConfigSource.default.loadOrThrow[AppConfig]
  
  def loadFromFile(path: String): Either[ConfigReaderFailures, AppConfig] =
    ConfigSource.file(path).load[AppConfig]
```

### HOCON Config Files

```hocon
# src/main/resources/application.conf

database {
  host = "localhost"
  host = ${?DB_HOST}           # override ด้วย env var
  
  port = 5432
  port = ${?DB_PORT}
  
  name = "myapp"
  name = ${?DB_NAME}
  
  username = "postgres"
  username = ${?DB_USERNAME}
  
  password = ""
  password = ${?DB_PASSWORD}
  
  max-connections = 10
  min-connections = 2
  connection-timeout = 30000
}

server {
  host = "0.0.0.0"
  port = 8080
  port = ${?PORT}
  
  read-timeout = 30000
  write-timeout = 30000
  max-connections = 1000
}

cache {
  host = "localhost"
  host = ${?REDIS_HOST}
  
  port = 6379
  port = ${?REDIS_PORT}
  
  max-keys = 10000
  ttl-seconds = 3600
  
  password = null
  password = ${?REDIS_PASSWORD}
}

log-level = "INFO"
log-level = ${?LOG_LEVEL}

environment = "development"
environment = ${?ENVIRONMENT}
```

```hocon
# src/test/resources/application.conf
include "application.conf"

database {
  host = "localhost"
  name = "myapp_test"
  max-connections = 3
}

environment = "test"
log-level = "DEBUG"
```

### Custom ConfigReader

```scala
import pureconfig.*
import pureconfig.generic.derivation.default.*
import scala.concurrent.duration.*

// Custom reader สำหรับ Duration
given ConfigReader[FiniteDuration] =
  ConfigReader.fromString { s =>
    s.split(" ").toList match
      case amount :: unit :: Nil =>
        val n = amount.toLong
        unit.toLowerCase match
          case "ms" | "millis"  => Right(n.millis)
          case "s"  | "seconds" => Right(n.seconds)
          case "m"  | "minutes" => Right(n.minutes)
          case "h"  | "hours"   => Right(n.hours)
          case u => Left(ConfigReaderFailures(
            ConvertFailure(CannotConvert(s, "FiniteDuration", s"Unknown unit: $u"), None, "")
          ))
      case _ =>
        Left(ConfigReaderFailures(
          ConvertFailure(CannotConvert(s, "FiniteDuration", "Expected 'amount unit'"), None, "")
        ))
  }

// Custom reader สำหรับ URL
case class ServiceUrl(host: String, port: Int, path: String):
  def toUrl: String = s"http://$host:$port$path"

given ConfigReader[ServiceUrl] =
  ConfigReader.fromString { s =>
    val pattern = """http://([^:]+):(\d+)(/.*)""".r
    s match
      case pattern(host, port, path) =>
        Right(ServiceUrl(host, port.toInt, path))
      case _ =>
        Left(ConfigReaderFailures(
          ConvertFailure(CannotConvert(s, "ServiceUrl", "Expected http://host:port/path"), None, "")
        ))
  }

// ใช้ custom readers
case class ServiceConfig(
  timeout: FiniteDuration,
  apiUrl: ServiceUrl,
  retryDelay: FiniteDuration
) derives ConfigReader

// application.conf:
// service {
//   timeout = "30 s"
//   api-url = "http://api.example.com:8080/v1"
//   retry-delay = "500 ms"
// }
```

### Nested Config ที่ซับซ้อน

```scala
import pureconfig.*
import pureconfig.generic.derivation.default.*

// Enum สำหรับ config
enum LogLevel:
  case Debug, Info, Warn, Error

given ConfigReader[LogLevel] =
  ConfigReader.fromString {
    case "DEBUG" | "debug" => Right(LogLevel.Debug)
    case "INFO"  | "info"  => Right(LogLevel.Info)
    case "WARN"  | "warn"  => Right(LogLevel.Warn)
    case "ERROR" | "error" => Right(LogLevel.Error)
    case other => Left(ConfigReaderFailures(
      ConvertFailure(CannotConvert(other, "LogLevel", "Unknown log level"), None, "")
    ))
  }

// Nested configs
case class SslConfig(
  enabled: Boolean,
  certPath: Option[String],
  keyPath: Option[String],
  trustStorePath: Option[String]
) derives ConfigReader

case class KafkaConfig(
  bootstrapServers: List[String],
  groupId: String,
  autoOffsetReset: String,
  enableAutoCommit: Boolean,
  ssl: SslConfig,
  maxPollRecords: Int,
  sessionTimeoutMs: Int
) derives ConfigReader

case class MetricsConfig(
  enabled: Boolean,
  port: Int,
  path: String,
  prefix: String
) derives ConfigReader

case class FullAppConfig(
  server: ServerConfig,
  database: DatabaseConfig,
  kafka: KafkaConfig,
  metrics: MetricsConfig,
  logLevel: LogLevel,
  environment: String
) derives ConfigReader

@main def pureconfigDemo(): Unit =
  ConfigSource.default.load[FullAppConfig] match
    case Right(config) =>
      println(s"Loaded config for env: ${config.environment}")
      println(s"Database: ${config.database.host}:${config.database.port}")
      println(s"Server: ${config.server.host}:${config.server.port}")
    case Left(failures) =>
      failures.toList.foreach { f =>
        println(s"Config error: ${f.description}")
      }
      sys.exit(1)
```

---

## ciris สำหรับ Effectful Config

### ciris Basics

```scala
import cats.effect.*
import ciris.*

// ciris อ่าน config จาก environment variables
// และรองรับ validation, transformation แบบ functional

object CirisConfig:

  // อ่านจาก environment variables
  val host: ConfigValue[Effect, String] =
    env("APP_HOST").default("localhost")

  val port: ConfigValue[Effect, Int] =
    env("APP_PORT")
      .or(prop("app.port"))  // fallback to system property
      .default("8080")
      .as[Int]

  val logLevel: ConfigValue[Effect, String] =
    env("LOG_LEVEL").default("INFO")

  // Combine config values
  val appConfig: ConfigValue[Effect, (String, Int, String)] =
    (host, port, logLevel).parTupled

  def loadConfig: IO[(String, Int, String)] =
    appConfig.load[IO]
```

### ciris ที่ซับซ้อน

```scala
import cats.effect.*
import cats.syntax.all.*
import ciris.*
import scala.concurrent.duration.*

// กำหนด Config data classes
case class HttpConfig(host: String, port: Int, timeout: FiniteDuration)
case class DbConfig(url: String, maxPool: Int, password: Secret[String])
case class AppConfig(http: HttpConfig, db: DbConfig, debug: Boolean)

object CirisFullConfig:

  // Custom ciris ConfigDecoder สำหรับ FiniteDuration
  given ConfigDecoder[String, FiniteDuration] =
    ConfigDecoder[String, String].mapOption("FiniteDuration") { s =>
      s.split(" ").toList match
        case amount :: "s" :: Nil  => Some(amount.toLong.seconds)
        case amount :: "ms" :: Nil => Some(amount.toLong.millis)
        case amount :: "m" :: Nil  => Some(amount.toLong.minutes)
        case _                     => None
    }

  def httpConfig: ConfigValue[Effect, HttpConfig] =
    (
      env("HTTP_HOST").default("0.0.0.0"),
      env("HTTP_PORT").as[Int].default(8080),
      env("HTTP_TIMEOUT").as[FiniteDuration].default("30 s")
    ).parMapN(HttpConfig.apply)

  def dbConfig: ConfigValue[Effect, DbConfig] =
    (
      env("DATABASE_URL")
        .or(env("DB_URL"))
        .default("jdbc:postgresql://localhost:5432/myapp"),
      env("DB_MAX_POOL").as[Int].default(10),
      env("DB_PASSWORD").as[Secret[String]].default(Secret(""))
    ).parMapN(DbConfig.apply)

  def appConfig: ConfigValue[Effect, AppConfig] =
    (
      httpConfig,
      dbConfig,
      env("DEBUG").as[Boolean].default(false)
    ).parMapN(AppConfig.apply)

  // Load with IO
  def load: IO[AppConfig] =
    appConfig.load[IO]
    
  // Load with error handling
  def loadSafe: IO[Either[ConfigError, AppConfig]] =
    appConfig.attempt[IO]
```

### ciris Secret Values

```scala
import cats.effect.*
import ciris.*

object SecretConfig:

  // Secret[A] - wrapper ที่ป้องกันการ log ค่าออกมา
  def apiKey: ConfigValue[Effect, Secret[String]] =
    env("API_KEY").as[Secret[String]]

  def dbPassword: ConfigValue[Effect, Secret[String]] =
    env("DB_PASSWORD")
      .as[Secret[String]]
      .default(Secret(""))  // ไม่ log ค่า default

  // ใช้ Secret โดยไม่ expose ค่า
  def useSecret(secret: Secret[String]): IO[Unit] =
    // secret.value ดึงค่าออกมาได้
    IO.println(s"Using secret: ${secret}")  // แสดงแค่ Secret(<hidden>)
    // IO.println(s"Secret value: ${secret.value}")  // ไม่ควรทำ!

  def demo: IO[Unit] =
    for
      key <- apiKey.default(Secret("test-key")).load[IO]
      _ <- IO.println(s"API Key: $key")  // Secret(<redacted>)
      _ <- IO.println(s"API Key value: ${key.value}")  // actual value
    yield ()
```

---

## Config Validation และ Loading

### Validation ด้วย cats-effect

```scala
import cats.effect.*
import cats.data.{Validated, ValidatedNel}
import cats.syntax.all.*

case class ServerConf(host: String, port: Int)
case class DatabaseConf(url: String, maxPool: Int)
case class Config(server: ServerConf, db: DatabaseConf)

object ConfigValidator:

  type ValidationResult[A] = ValidatedNel[String, A]

  def validateHost(host: String): ValidationResult[String] =
    if host.nonEmpty && host.length <= 255 then host.validNel
    else s"Invalid host: '$host'".invalidNel

  def validatePort(port: Int): ValidationResult[Int] =
    if port >= 1 && port <= 65535 then port.validNel
    else s"Port must be 1-65535, got: $port".invalidNel

  def validateUrl(url: String): ValidationResult[String] =
    if url.startsWith("jdbc:") || url.startsWith("postgresql://") then
      url.validNel
    else
      s"Invalid database URL: $url".invalidNel

  def validateMaxPool(n: Int): ValidationResult[Int] =
    if n >= 1 && n <= 1000 then n.validNel
    else s"Max pool must be 1-1000, got: $n".invalidNel

  def validateServerConf(host: String, port: Int): ValidationResult[ServerConf] =
    (validateHost(host), validatePort(port)).mapN(ServerConf.apply)

  def validateDbConf(url: String, maxPool: Int): ValidationResult[DatabaseConf] =
    (validateUrl(url), validateMaxPool(maxPool)).mapN(DatabaseConf.apply)

  def loadAndValidate(): IO[Config] =
    val rawServer = (
      sys.env.getOrElse("HOST", "localhost"),
      sys.env.get("PORT").flatMap(_.toIntOption).getOrElse(8080)
    )
    val rawDb = (
      sys.env.getOrElse("DATABASE_URL", "jdbc:postgresql://localhost/myapp"),
      sys.env.get("DB_MAX_POOL").flatMap(_.toIntOption).getOrElse(10)
    )

    val validated = (
      validateServerConf.tupled(rawServer),
      validateDbConf.tupled(rawDb)
    ).mapN(Config.apply)

    validated match
      case Validated.Valid(config) =>
        IO.pure(config)
      case Validated.Invalid(errors) =>
        IO.raiseError(
          new IllegalArgumentException(
            s"Config validation failed:\n${errors.toList.map("  - " + _).mkString("\n")}"
          )
        )
```

### Config Loading Pipeline

```scala
import cats.effect.*
import pureconfig.*
import pureconfig.generic.derivation.default.*

// Step 1: โหลดจากไฟล์
// Step 2: Override ด้วย environment variables
// Step 3: Validate
// Step 4: Return typed config

object ConfigPipeline:
  
  case class RawConfig(
    host: String = "localhost",
    port: Int = 8080,
    dbUrl: String = "jdbc:postgresql://localhost/myapp",
    logLevel: String = "INFO"
  ) derives ConfigReader

  case class AppConfig(
    host: String,
    port: Int,
    dbUrl: String,
    logLevel: String
  )

  def loadConfig: IO[AppConfig] =
    for
      // Step 1: Load base config from file
      baseConfig <- IO.fromEither(
        ConfigSource.default.load[RawConfig]
          .left.map(e => new RuntimeException(e.prettyPrint()))
      )
      
      // Step 2: Apply environment variable overrides
      withEnvOverrides = baseConfig.copy(
        host = sys.env.getOrElse("HOST", baseConfig.host),
        port = sys.env.get("PORT").flatMap(_.toIntOption).getOrElse(baseConfig.port),
        dbUrl = sys.env.getOrElse("DATABASE_URL", baseConfig.dbUrl),
        logLevel = sys.env.getOrElse("LOG_LEVEL", baseConfig.logLevel)
      )
      
      // Step 3: Validate
      _ <- IO.raiseUnless(withEnvOverrides.port > 0)(
        new IllegalArgumentException(s"Invalid port: ${withEnvOverrides.port}")
      )
      _ <- IO.raiseUnless(withEnvOverrides.logLevel.nonEmpty)(
        new IllegalArgumentException("Log level cannot be empty")
      )
      
      // Step 4: Convert to typed config
      config = AppConfig(
        host = withEnvOverrides.host,
        port = withEnvOverrides.port,
        dbUrl = withEnvOverrides.dbUrl,
        logLevel = withEnvOverrides.logLevel
      )
      
      _ <- IO.println(s"Config loaded for ${config.host}:${config.port}")
    yield config
```

---

## Environment-specific Configs

### Multi-environment Configuration

```hocon
# src/main/resources/application.conf (base config)
app {
  name = "MyApp"
  version = "1.0.0"
}

database {
  driver = "org.postgresql.Driver"
  pool-size = 10
  connection-timeout = 30 seconds
}

server {
  host = "0.0.0.0"
  port = 8080
}
```

```hocon
# src/main/resources/application-dev.conf
include "application.conf"

database {
  url = "jdbc:postgresql://localhost:5432/myapp_dev"
  username = "dev_user"
  password = "dev_password"
  pool-size = 3
}

server {
  port = 8080
}

logging {
  level = "DEBUG"
  console = true
  file = false
}

features {
  debug-mode = true
  mock-external-apis = true
}
```

```hocon
# src/main/resources/application-prod.conf
include "application.conf"

database {
  url = ${DATABASE_URL}
  username = ${DB_USERNAME}
  password = ${DB_PASSWORD}
  pool-size = 20
  connection-timeout = 10 seconds
}

server {
  port = ${PORT}
}

logging {
  level = "WARN"
  console = false
  file = true
  file-path = "/var/log/myapp/app.log"
}

features {
  debug-mode = false
  mock-external-apis = false
}
```

### Environment-aware Config Loading

```scala
import cats.effect.*
import pureconfig.*
import pureconfig.generic.derivation.default.*

enum Environment:
  case Dev, Test, Staging, Prod

object EnvironmentConfig:

  def detectEnvironment(): Environment =
    sys.env.getOrElse("ENVIRONMENT", "dev").toLowerCase match
      case "dev" | "development"        => Environment.Dev
      case "test" | "testing"           => Environment.Test
      case "staging" | "stage"          => Environment.Staging
      case "prod" | "production"        => Environment.Prod
      case other                        =>
        println(s"Unknown environment '$other', defaulting to Dev")
        Environment.Dev

  def configFile(env: Environment): String =
    env match
      case Environment.Dev     => "application-dev.conf"
      case Environment.Test    => "application-test.conf"
      case Environment.Staging => "application-staging.conf"
      case Environment.Prod    => "application-prod.conf"

  def loadForEnvironment[A: ConfigReader]: IO[A] =
    IO {
      val env = detectEnvironment()
      println(s"Loading config for environment: $env")
      
      ConfigSource
        .resources(configFile(env))
        .withFallback(ConfigSource.default)
        .load[A]
    }.flatMap {
      case Right(config) => IO.pure(config)
      case Left(failures) =>
        IO.raiseError(new RuntimeException(s"Config load failed:\n${failures.prettyPrint()}"))
    }

// Feature flags
case class FeatureFlags(
  debugMode: Boolean,
  mockExternalApis: Boolean,
  enableBetaFeatures: Boolean,
  maintenanceMode: Boolean
) derives ConfigReader

object FeatureFlags:
  val default: FeatureFlags = FeatureFlags(
    debugMode = false,
    mockExternalApis = false,
    enableBetaFeatures = false,
    maintenanceMode = false
  )
  
  def forEnvironment(env: Environment): FeatureFlags =
    env match
      case Environment.Dev =>
        FeatureFlags(debugMode = true, mockExternalApis = true, 
                     enableBetaFeatures = true, maintenanceMode = false)
      case Environment.Test =>
        FeatureFlags(debugMode = true, mockExternalApis = true,
                     enableBetaFeatures = false, maintenanceMode = false)
      case Environment.Staging =>
        FeatureFlags(debugMode = false, mockExternalApis = false,
                     enableBetaFeatures = true, maintenanceMode = false)
      case Environment.Prod =>
        default
```

---

## Secret Management

### Vault Integration

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.ember.client.EmberClientBuilder
import org.http4s.circe.*
import io.circe.generic.auto.*
import io.circe.syntax.*

// Vault client สำหรับดึง secrets
case class VaultSecret(
  requestId: String,
  data: Map[String, String]
)

object VaultClient:
  
  def getSecret(
    vaultAddr: String,
    token: String,
    secretPath: String
  ): IO[Map[String, String]] =
    EmberClientBuilder
      .default[IO]
      .build
      .use { client =>
        val request = Request[IO](
          method = Method.GET,
          uri = Uri.unsafeFromString(s"$vaultAddr/v1/$secretPath"),
          headers = Headers(
            Header.Raw(ci"X-Vault-Token", token)
          )
        )
        
        client
          .expect[VaultSecret](request)(jsonOf[IO, VaultSecret])
          .map(_.data)
      }

// AWS Secrets Manager
object AwsSecretsManager:
  
  def getSecret(secretName: String): IO[Map[String, String]] =
    // ใช้ AWS SDK (simplified)
    IO {
      // In production, use aws-java-sdk-secretsmanager
      Map(
        "db_password" -> sys.env.getOrElse(s"${secretName}_DB_PASSWORD", ""),
        "api_key" -> sys.env.getOrElse(s"${secretName}_API_KEY", "")
      )
    }
```

### Encrypted Secrets ใน Config

```scala
import cats.effect.*
import java.util.Base64
import javax.crypto.{Cipher, SecretKeyFactory}
import javax.crypto.spec.{IvParameterSpec, PBEKeySpec, SecretKeySpec}

object SecretEncryption:
  
  private val ALGORITHM = "AES/CBC/PKCS5Padding"
  private val KEY_ALGORITHM = "AES"
  private val ITERATIONS = 65536
  private val KEY_LENGTH = 256
  
  def encrypt(plaintext: String, masterKey: String): IO[String] =
    IO {
      val salt = new Array[Byte](16)
      new java.security.SecureRandom().nextBytes(salt)
      
      val spec = new PBEKeySpec(masterKey.toCharArray, salt, ITERATIONS, KEY_LENGTH)
      val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
      val keyBytes = factory.generateSecret(spec).getEncoded
      val key = new SecretKeySpec(keyBytes, KEY_ALGORITHM)
      
      val iv = new Array[Byte](16)
      new java.security.SecureRandom().nextBytes(iv)
      val ivSpec = new IvParameterSpec(iv)
      
      val cipher = Cipher.getInstance(ALGORITHM)
      cipher.init(Cipher.ENCRYPT_MODE, key, ivSpec)
      val encrypted = cipher.doFinal(plaintext.getBytes("UTF-8"))
      
      val result = salt ++ iv ++ encrypted
      Base64.getEncoder.encodeToString(result)
    }
  
  def decrypt(ciphertext: String, masterKey: String): IO[String] =
    IO {
      val bytes = Base64.getDecoder.decode(ciphertext)
      val salt = bytes.slice(0, 16)
      val iv = bytes.slice(16, 32)
      val encrypted = bytes.slice(32, bytes.length)
      
      val spec = new PBEKeySpec(masterKey.toCharArray, salt, ITERATIONS, KEY_LENGTH)
      val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
      val keyBytes = factory.generateSecret(spec).getEncoded
      val key = new SecretKeySpec(keyBytes, KEY_ALGORITHM)
      
      val ivSpec = new IvParameterSpec(iv)
      val cipher = Cipher.getInstance(ALGORITHM)
      cipher.init(Cipher.DECRYPT_MODE, key, ivSpec)
      
      new String(cipher.doFinal(encrypted), "UTF-8")
    }

// Config ที่มีการเข้ารหัส secrets
object EncryptedConfigLoader:
  
  def loadSecrets(masterKey: String): IO[Map[String, String]] =
    val encryptedSecrets = Map(
      "db_password" -> sys.env.getOrElse("ENCRYPTED_DB_PASSWORD", ""),
      "api_key"     -> sys.env.getOrElse("ENCRYPTED_API_KEY", "")
    )
    
    encryptedSecrets
      .filter(_._2.nonEmpty)
      .toList
      .traverse { (key, encrypted) =>
        SecretEncryption.decrypt(encrypted, masterKey).map(key -> _)
      }
      .map(_.toMap)
```

### Environment Variables Safety

```scala
import cats.effect.*
import cats.data.{Validated, ValidatedNel}
import cats.syntax.all.*

// Type-safe env var reading
object EnvVars:
  
  type Result[A] = ValidatedNel[String, A]

  def required(name: String): Result[String] =
    sys.env.get(name) match
      case Some(v) if v.nonEmpty => v.validNel
      case Some(_)               => s"$name is set but empty".invalidNel
      case None                  => s"$name is required but not set".invalidNel

  def optional(name: String, default: String = ""): String =
    sys.env.getOrElse(name, default)

  def requiredInt(name: String): Result[Int] =
    required(name).andThen { v =>
      v.toIntOption match
        case Some(n) => n.validNel
        case None    => s"$name must be an integer, got: '$v'".invalidNel
    }

  def optionalInt(name: String, default: Int): Int =
    sys.env.get(name).flatMap(_.toIntOption).getOrElse(default)

  def requiredBoolean(name: String): Result[Boolean] =
    required(name).andThen { v =>
      v.toLowerCase match
        case "true" | "yes" | "1"  => true.validNel
        case "false" | "no" | "0"  => false.validNel
        case other => s"$name must be boolean (true/false), got: '$other'".invalidNel
    }

  // Load all required env vars
  def loadProductionConfig: Result[(String, Int, String, String)] =
    (
      required("DATABASE_URL"),
      requiredInt("PORT"),
      required("JWT_SECRET"),
      required("API_KEY")
    ).mapN((url, port, jwt, key) => (url, port, jwt, key))

// ใช้งาน
object SafeEnvLoading extends IOApp.Simple:
  def run: IO[Unit] =
    EnvVars.loadProductionConfig match
      case Validated.Valid((url, port, jwt, key)) =>
        IO.println(s"Config loaded: port=$port, url starts with: ${url.take(20)}...")
      case Validated.Invalid(errors) =>
        IO.println("Configuration errors:") >>
        errors.toList.traverse(e => IO.println(s"  ERROR: $e")).void >>
        IO.raiseError(new RuntimeException("Invalid configuration"))
```

---

## Hot Reloading Config

### Config Hot Reload ด้วย fs2

```scala
import cats.effect.*
import cats.effect.std.{Queue, Topic}
import fs2.*
import fs2.io.file.{Files, Path, WatchEvent}
import scala.concurrent.duration.*

// Config ที่ reloadable
case class ReloadableConfig(
  maxConnections: Int,
  timeout: Int,
  featureFlags: Map[String, Boolean]
)

object ConfigHotReload extends IOApp.Simple:

  // Watch config file สำหรับ changes
  def watchConfigFile(path: String): Stream[IO, Unit] =
    Files[IO]
      .watch(Path(path))
      .filter {
        case WatchEvent.Modified(_, _) => true
        case WatchEvent.Created(_, _)  => true
        case _                         => false
      }
      .void

  // Load config จาก file
  def loadConfig(path: String): IO[ReloadableConfig] =
    IO {
      // Simplified - ในจริงใช้ pureconfig
      ReloadableConfig(
        maxConnections = 10,
        timeout = 30,
        featureFlags = Map("featureA" -> true, "featureB" -> false)
      )
    }

  // Ref ที่ hold config ปัจจุบัน
  def makeConfigRef(configPath: String): Resource[IO, Ref[IO, ReloadableConfig]] =
    Resource.eval {
      for
        initial <- loadConfig(configPath)
        ref     <- Ref.of[IO, ReloadableConfig](initial)
      yield ref
    }

  // Hot reload loop
  def startHotReload(
    configPath: String,
    configRef: Ref[IO, ReloadableConfig]
  ): Resource[IO, Fiber[IO, Throwable, Unit]] =
    val reloadStream = watchConfigFile(configPath)
      .debounce(500.millis)  // debounce เพื่อหลีกเลี่ยงการ reload บ่อยเกินไป
      .evalMap { _ =>
        loadConfig(configPath)
          .flatMap(configRef.set)
          .flatTap(_ => IO.println("Config reloaded!"))
          .handleErrorWith(e => IO.println(s"Config reload failed: $e"))
      }

    Resource.make(
      reloadStream.compile.drain.start
    )(_.cancel)

  def run: IO[Unit] =
    (for
      configRef <- makeConfigRef("/tmp/app.conf")
      _         <- startHotReload("/tmp/app.conf", configRef)
    yield configRef).use { configRef =>
      // ทำงานต่อเนื่อง โดยอ่าน config ล่าสุดตลอดเวลา
      Stream
        .fixedDelay[IO](2.seconds)
        .evalMap { _ =>
          configRef.get.flatMap { config =>
            IO.println(s"Current config: maxConnections=${config.maxConnections}")
          }
        }
        .take(5)
        .compile
        .drain
    }
```

### Config Change Notification

```scala
import cats.effect.*
import cats.effect.std.Topic
import fs2.*
import scala.concurrent.duration.*

// Event ที่เกิดขึ้นเมื่อ config เปลี่ยน
case class ConfigChangeEvent(
  key: String,
  oldValue: String,
  newValue: String,
  timestamp: Long
)

class ConfigWithNotifications(
  private val ref: Ref[IO, Map[String, String]],
  private val topic: Topic[IO, ConfigChangeEvent]
):
  def get(key: String): IO[Option[String]] =
    ref.get.map(_.get(key))

  def set(key: String, value: String): IO[Unit] =
    ref.getAndUpdate(_ + (key -> value)).flatMap { oldConfig =>
      val oldValue = oldConfig.getOrElse(key, "")
      if oldValue != value then
        topic.publish1(ConfigChangeEvent(
          key = key,
          oldValue = oldValue,
          newValue = value,
          timestamp = System.currentTimeMillis()
        )).void
      else IO.unit
    }

  def subscribe: Stream[IO, ConfigChangeEvent] =
    topic.subscribe(100)

object ConfigWithNotifications:
  def make: IO[ConfigWithNotifications] =
    for
      ref   <- Ref.of[IO, Map[String, String]](Map.empty)
      topic <- Topic[IO, ConfigChangeEvent]
    yield new ConfigWithNotifications(ref, topic)

// ใช้งาน
object ConfigNotificationDemo extends IOApp.Simple:
  def run: IO[Unit] =
    ConfigWithNotifications.make.flatMap { config =>
      
      // Subscribe to changes
      val listener = config.subscribe
        .evalMap { event =>
          IO.println(s"Config changed: ${event.key} = '${event.oldValue}' -> '${event.newValue}'")
        }
        .take(3)
      
      // Make config changes
      val changes = Stream
        .iterate(0)(_ + 1)
        .take(4)
        .covary[IO]
        .evalMap { i =>
          IO.sleep(200.millis) >>
          config.set("featureFlag", if i % 2 == 0 then "true" else "false") >>
          IO.println(s"Updated config at iteration $i")
        }
      
      changes.concurrently(listener).compile.drain
    }
```

---

## Config ใน cats-effect Applications

### Proper Config Loading Pattern

```scala
import cats.effect.*
import pureconfig.*
import pureconfig.generic.derivation.default.*
import scala.concurrent.duration.*

// Full application config
case class AppConfig(
  server: ServerConfig,
  database: DatabaseConfig,
  cache: CacheConfig,
  features: FeatureConfig
) derives ConfigReader

case class FeatureConfig(
  enableUserRegistration: Boolean = true,
  enableEmailNotifications: Boolean = true,
  maxUploadSizeMb: Int = 10,
  rateLimitPerMinute: Int = 100
) derives ConfigReader

// Application entrypoint ที่ proper
object Main extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    // Load config first, fail fast if invalid
    val configResource = Resource.eval(
      IO.fromEither(
        ConfigSource.default.load[AppConfig]
          .left.map(e => new RuntimeException(e.prettyPrint()))
      ).flatTap { config =>
        IO.println(s"Starting application in ${sys.env.getOrElse("ENVIRONMENT", "dev")} mode")
      }
    )

    configResource
      .use { config =>
        Application.start(config)
      }
      .as(ExitCode.Success)
      .handleErrorWith { e =>
        IO.println(s"Fatal error: ${e.getMessage}").as(ExitCode.Error)
      }

// Application ที่รับ config เป็น dependency
object Application:
  def start(config: AppConfig): IO[Nothing] =
    for
      _ <- IO.println(s"Server starting on ${config.server.host}:${config.server.port}")
      _ <- IO.println(s"DB: ${config.database.host}:${config.database.port}")
      _ <- IO.never  // keep running
    yield ???
```

### Config Layer Pattern

```scala
import cats.effect.*

// Config Layer - inject config ผ่าน typeclass
trait HasConfig[F[_]]:
  def config: F[AppConfig]

// Service ที่ต้องการ config
class UserService[F[_]: Monad: HasConfig]:
  def maxUploadSize: F[Int] =
    HasConfig[F].config.map(_.features.maxUploadSizeMb)

// Concrete implementation
class IoUserService(appConfig: AppConfig) extends UserService[IO]
  with HasConfig[IO]:
  def config: IO[AppConfig] = IO.pure(appConfig)

// Wiring
object Wiring:
  def makeServices(config: AppConfig): (UserService[IO]) =
    val userService = new IoUserService(config)
    userService
```

---

## Testing กับ Config

### Test Config Setup

```scala
import cats.effect.*
import pureconfig.*
import pureconfig.generic.derivation.default.*

object TestConfig:
  
  // Config สำหรับ unit tests (in-memory)
  val unitTestConfig: AppConfig = AppConfig(
    server = ServerConfig("localhost", 0, 5000, 5000, 10),
    database = DatabaseConfig(
      host = "localhost",
      port = 5432,
      name = "test_db",
      username = "test",
      password = "test",
      maxConnections = 2,
      minConnections = 1,
      connectionTimeout = 5000
    ),
    cache = CacheConfig(
      host = "localhost",
      port = 6379,
      maxKeys = 100,
      ttlSeconds = 60,
      password = None
    ),
    features = FeatureConfig(
      enableUserRegistration = true,
      enableEmailNotifications = false,  // disable in tests
      maxUploadSizeMb = 1,
      rateLimitPerMinute = 1000          // higher limit for tests
    )
  )

  // Load test config
  def loadTestConfig: IO[AppConfig] =
    IO.fromEither(
      ConfigSource
        .resources("application-test.conf")
        .withFallback(ConfigSource.default)
        .load[AppConfig]
        .left.map(e => new RuntimeException(e.prettyPrint()))
    )

// Test helper
class ConfigSpec extends munit.CatsEffectSuite:
  
  test("config loads successfully"):
    TestConfig.loadTestConfig.map { config =>
      assertEquals(config.features.enableEmailNotifications, false)
      assert(config.database.maxConnections <= 5)
    }

  test("config has required fields"):
    TestConfig.loadTestConfig.map { config =>
      assert(config.server.port > 0)
      assert(config.database.name.nonEmpty)
    }

  test("environment-specific override works"):
    // Set test environment
    System.setProperty("ENVIRONMENT", "test")
    TestConfig.loadTestConfig.map { config =>
      assertEquals(config.features.enableEmailNotifications, false)
    }.guarantee(IO(System.clearProperty("ENVIRONMENT")).void)
```

---

## Best Practices

### Configuration Best Practices

```scala
// 1. ใช้ type-safe config (ไม่ใช้ string keys)
// BAD:
val host = config.getString("server.host")
val port = config.getInt("server.port")  // อาจ throw exception

// GOOD:
case class ServerConf(host: String, port: Int) derives ConfigReader
val serverConf = ConfigSource.default.load[ServerConf]

// 2. Fail fast ที่ startup
// ตรวจสอบ config ก่อนเริ่ม application
object SafeStart extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    ConfigSource.default.load[AppConfig] match
      case Left(failures) =>
        IO.println(s"FATAL: Config errors:\n${failures.prettyPrint()}").as(ExitCode.Error)
      case Right(config) =>
        startApplication(config).as(ExitCode.Success)
  
  def startApplication(config: AppConfig): IO[Unit] = IO.unit

// 3. ไม่เก็บ secrets ใน config files
// BAD: password = "my-secret-password"  <- in git!
// GOOD: password = ${DB_PASSWORD}       <- from environment

// 4. Validate config values
def validateConfig(config: AppConfig): Either[List[String], AppConfig] =
  val errors = scala.collection.mutable.ListBuffer[String]()
  
  if config.server.port < 1 || config.server.port > 65535 then
    errors += s"Invalid port: ${config.server.port}"
  if config.database.maxConnections < 1 then
    errors += s"DB max connections must be >= 1"
  if config.features.maxUploadSizeMb > 100 then
    errors += s"Max upload size should not exceed 100MB"
  
  if errors.isEmpty then Right(config)
  else Left(errors.toList)

// 5. Document config options
// scaladoc สำหรับ config case classes
/**
 * Server configuration
 *
 * @param host    Hostname to bind (default: "0.0.0.0")
 * @param port    Port to listen on. Must be 1-65535 (default: 8080)
 * @param timeout Request timeout in milliseconds (default: 30000)
 */
case class DocumentedServerConfig(
  host: String = "0.0.0.0",
  port: Int = 8080,
  timeout: Int = 30000
) derives ConfigReader
```

### Config Testing Strategy

```scala
class ConfigIntegrationSpec extends munit.CatsEffectSuite:
  
  // ทดสอบว่า config โหลดได้ใน test environment
  test("application config loads without errors"):
    IO.fromEither(
      ConfigSource.default.load[AppConfig]
        .left.map(e => new RuntimeException(e.prettyPrint()))
    ).map(_ => ())

  // ทดสอบ config validation
  test("config validation rejects invalid values"):
    val invalidConfig = TestConfig.unitTestConfig.copy(
      server = TestConfig.unitTestConfig.server.copy(port = -1)
    )
    
    val result = validateConfig(invalidConfig)
    assert(result.isLeft)
    assert(result.left.toOption.get.exists(_.contains("port")))

  // ทดสอบ default values
  test("default config has sensible values"):
    val config = TestConfig.unitTestConfig
    
    assert(config.server.port > 0)
    assert(config.database.maxConnections > 0)
    assert(config.features.rateLimitPerMinute > 0)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Configuration Management:

### สิ่งที่ได้เรียนรู้

| เครื่องมือ | วัตถุประสงค์ | เมื่อไหรใช้ |
|----------|-------------|-----------|
| pureconfig | Type-safe HOCON/properties loading | Config files ใน classpath |
| ciris | Effectful, functional config | Environment variables, secrets |
| HOCON override | Environment-specific config | Multi-environment deployments |
| Vault/AWS | Secret management | Production secrets |
| Hot reload | Dynamic config updates | Feature flags, rate limits |
| Validation | Config correctness | Fail fast at startup |

### Config Loading Flow ที่ดี

```
1. โหลด base config จาก application.conf
   ↓
2. Override ด้วย environment-specific file (application-prod.conf)
   ↓
3. Override sensitive values ด้วย environment variables
   ↓
4. โหลด secrets จาก Vault/AWS Secrets Manager
   ↓
5. Validate ทุก config values
   ↓
6. Fail fast หากมี error
   ↓
7. Start application ด้วย validated config
```

### Config Security Checklist

- ✅ ไม่เก็บ passwords/API keys ใน config files ที่อยู่ใน git
- ✅ ใช้ environment variables สำหรับ sensitive values
- ✅ ใช้ secrets manager สำหรับ production
- ✅ Log config ที่ startup (ยกเว้น secrets)
- ✅ Validate config ก่อน start application
- ✅ ใช้ Secret[A] wrapper เพื่อป้องกัน accidental logging
- ✅ Test config loading ใน CI/CD pipeline

---

*[← Part 58: Schema Evolution](part-58-schema-evolution.md) | [Part 60: Distributed Tracing →](part-60-distributed-tracing.md)*
