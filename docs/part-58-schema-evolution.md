# ส่วนที่ 58: Schema Evolution และ Compatibility

## สารบัญ

1. [Schema Evolution คืออะไร](#schema-evolution-คืออะไร)
2. [Avro Schema Evolution Rules](#avro-schema-evolution-rules)
3. [Forward/Backward Compatibility](#forwardbackward-compatibility)
4. [JSON Schema Versioning](#json-schema-versioning)
5. [Database Schema Migration ด้วย Flyway](#database-schema-migration-ด้วย-flyway)
6. [API Versioning Strategies](#api-versioning-strategies)
7. [Schema Registry Patterns](#schema-registry-patterns)
8. [Scala Implementation ของ Schema Evolution](#scala-implementation-ของ-schema-evolution)
9. [Testing Schema Compatibility](#testing-schema-compatibility)
10. [Best Practices](#best-practices)
11. [สรุป](#สรุป)

---

## Schema Evolution คืออะไร

Schema Evolution คือกระบวนการเปลี่ยน schema ของข้อมูลในขณะที่ยังคง compatibility กับ producers และ consumers ที่ใช้ schema เวอร์ชันเก่าอยู่

### ทำไมต้องมี Schema Evolution?

```
ปัญหา: ระบบที่มีหลาย services ต้องการ schema ที่เปลี่ยนได้

Producer v1                  Consumer v2
(ใช้ schema v1)  ──────>  (ต้องการ schema v2)
                     |
              Kafka/Message Queue
                     |
Producer v2                  Consumer v1
(ใช้ schema v2)  ──────>  (ยังใช้ schema v1 อยู่)

ต้องรองรับทั้งสองกรณีนี้!
```

### ประเภทของ Schema Changes

```scala
// Schema เดิม
case class UserV1(
  id: Int,
  name: String,
  email: String
)

// Non-breaking changes (safe to evolve)
case class UserV2(
  id: Int,
  name: String,
  email: String,
  // เพิ่ม optional field
  phone: Option[String] = None,
  // เพิ่ม field ที่มี default value
  createdAt: Long = System.currentTimeMillis()
)

// Breaking changes (unsafe!)
case class UserV3_BREAKING(
  // ลบ field ที่จำเป็น - BREAKING!
  // id: Int,  // removed!
  fullName: String,  // rename จาก name - BREAKING!
  contactEmail: String  // rename จาก email - BREAKING!
)
```

### Dependencies สำหรับ Project

```scala
// build.sbt
libraryDependencies ++= Seq(
  // Avro
  "org.apache.avro"    % "avro"                % "1.11.3",
  "io.confluent"       % "kafka-schema-registry-client" % "7.5.0",
  
  // JSON Schema
  "io.circe"           %% "circe-core"          % "0.14.6",
  "io.circe"           %% "circe-generic"       % "0.14.6",
  "io.circe"           %% "circe-parser"        % "0.14.6",
  
  // Database Migration
  "org.flywaydb"       % "flyway-core"          % "9.22.3",
  "org.postgresql"     % "postgresql"           % "42.6.0",
  
  // HTTP API
  "org.http4s"         %% "http4s-dsl"          % "0.23.23",
  "org.http4s"         %% "http4s-circe"        % "0.23.23",
  
  // Schema validation
  "com.networknt"      % "json-schema-validator" % "1.0.87"
)
```

---

## Avro Schema Evolution Rules

### Avro Schema Basics

```json
// user_v1.avsc
{
  "type": "record",
  "name": "User",
  "namespace": "com.example",
  "doc": "User record version 1",
  "fields": [
    {"name": "id",    "type": "int"},
    {"name": "name",  "type": "string"},
    {"name": "email", "type": "string"}
  ]
}
```

### Schema Evolution Rules ใน Avro

```json
// user_v2.avsc - Backward compatible
{
  "type": "record",
  "name": "User",
  "namespace": "com.example",
  "doc": "User record version 2",
  "fields": [
    {"name": "id",    "type": "int"},
    {"name": "name",  "type": "string"},
    {"name": "email", "type": "string"},
    
    // OK: เพิ่ม optional field พร้อม default
    {"name": "phone", "type": ["null", "string"], "default": null},
    
    // OK: เพิ่ม field ที่มี default value
    {"name": "createdAt", "type": "long", "default": 0},
    
    // OK: เพิ่ม enum ที่มี default
    {
      "name": "status",
      "type": {
        "type": "enum",
        "name": "UserStatus",
        "symbols": ["ACTIVE", "INACTIVE", "PENDING"]
      },
      "default": "ACTIVE"
    }
  ]
}
```

### Scala Avro Integration

```scala
import org.apache.avro.Schema
import org.apache.avro.generic.{GenericData, GenericRecord}
import org.apache.avro.io.{DatumReader, DatumWriter, DecoderFactory, EncoderFactory}
import org.apache.avro.specific.{SpecificDatumReader, SpecificDatumWriter}
import java.io.{ByteArrayInputStream, ByteArrayOutputStream}

// Avro schema as strings
object AvroSchemas:
  val userV1Schema: String = """
    {
      "type": "record",
      "name": "User",
      "namespace": "com.example",
      "fields": [
        {"name": "id",    "type": "int"},
        {"name": "name",  "type": "string"},
        {"name": "email", "type": "string"}
      ]
    }
  """

  val userV2Schema: String = """
    {
      "type": "record",
      "name": "User",
      "namespace": "com.example",
      "fields": [
        {"name": "id",    "type": "int"},
        {"name": "name",  "type": "string"},
        {"name": "email", "type": "string"},
        {"name": "phone", "type": ["null", "string"], "default": null},
        {"name": "age",   "type": "int", "default": 0}
      ]
    }
  """

// Avro serializer/deserializer
object AvroCodec:
  def serialize(record: GenericRecord, schema: Schema): Array[Byte] =
    val baos = new ByteArrayOutputStream()
    val encoder = EncoderFactory.get().binaryEncoder(baos, null)
    val writer: DatumWriter[GenericRecord] = 
      new org.apache.avro.generic.GenericDatumWriter[GenericRecord](schema)
    writer.write(record, encoder)
    encoder.flush()
    baos.toByteArray

  def deserialize(
    bytes: Array[Byte], 
    writerSchema: Schema, 
    readerSchema: Schema
  ): GenericRecord =
    val bais = new ByteArrayInputStream(bytes)
    val decoder = DecoderFactory.get().binaryDecoder(bais, null)
    val reader: DatumReader[GenericRecord] = 
      new org.apache.avro.generic.GenericDatumReader[GenericRecord](writerSchema, readerSchema)
    reader.read(null, decoder)

// ทดสอบ schema evolution
object AvroEvolutionDemo:
  def demonstrate(): Unit =
    val v1Schema = new Schema.Parser().parse(AvroSchemas.userV1Schema)
    val v2Schema = new Schema.Parser().parse(AvroSchemas.userV2Schema)

    // สร้าง record ด้วย v1 schema
    val v1Record = new GenericData.Record(v1Schema)
    v1Record.put("id", 1)
    v1Record.put("name", "Alice")
    v1Record.put("email", "alice@example.com")

    // Serialize ด้วย v1
    val bytes = AvroCodec.serialize(v1Record, v1Schema)
    println(s"Serialized ${bytes.length} bytes with v1 schema")

    // Deserialize ด้วย v2 (backward compatible!)
    val v2Record = AvroCodec.deserialize(bytes, v1Schema, v2Schema)
    println(s"id: ${v2Record.get("id")}")
    println(s"name: ${v2Record.get("name")}")
    println(s"email: ${v2Record.get("email")}")
    println(s"phone: ${v2Record.get("phone")} (default from v2)")
    println(s"age: ${v2Record.get("age")} (default from v2)")
```

---

## Forward/Backward Compatibility

### ความหมายของ Compatibility

```
Backward Compatibility:
  ผู้อ่านใหม่ (v2) สามารถอ่านข้อมูลที่เขียนด้วย schema เก่า (v1) ได้

  Writer v1 ──── data ────> Reader v2 ✓

Forward Compatibility:
  ผู้อ่านเก่า (v1) สามารถอ่านข้อมูลที่เขียนด้วย schema ใหม่ (v2) ได้

  Writer v2 ──── data ────> Reader v1 ✓

Full Compatibility:
  ทั้ง backward และ forward compatible
```

### การตรวจสอบ Compatibility ใน Scala

```scala
import org.apache.avro.Schema
import org.apache.avro.SchemaCompatibility
import org.apache.avro.SchemaCompatibility.{SchemaCompatibilityType, SchemaPairCompatibility}

object CompatibilityChecker:
  
  def checkBackwardCompatibility(
    readerSchema: Schema,
    writerSchema: Schema
  ): (Boolean, String) =
    val result: SchemaPairCompatibility = 
      SchemaCompatibility.checkReaderWriterCompatibility(readerSchema, writerSchema)
    
    val isCompatible = result.getType == SchemaCompatibilityType.COMPATIBLE
    val message = if isCompatible then "Compatible" 
                  else result.getDescription
    (isCompatible, message)

  def checkForwardCompatibility(
    oldSchema: Schema,
    newSchema: Schema
  ): (Boolean, String) =
    // Forward: old reader can read new writer's data
    checkBackwardCompatibility(oldSchema, newSchema)

  def checkFullCompatibility(
    schemaV1: Schema,
    schemaV2: Schema
  ): (Boolean, Boolean, String) =
    val (backward, bMsg) = checkBackwardCompatibility(schemaV2, schemaV1)
    val (forward, fMsg)  = checkForwardCompatibility(schemaV1, schemaV2)
    (backward, forward, s"Backward: $bMsg, Forward: $fMsg")

// Schema evolution test cases
object SchemaCompatibilityTests:
  
  val baseSchema: String = """
    {
      "type": "record",
      "name": "Event",
      "fields": [
        {"name": "id",   "type": "string"},
        {"name": "type", "type": "string"},
        {"name": "data", "type": "string"}
      ]
    }
  """

  // ✓ Backward compatible - เพิ่ม optional field
  val addOptionalField: String = """
    {
      "type": "record",
      "name": "Event",
      "fields": [
        {"name": "id",        "type": "string"},
        {"name": "type",      "type": "string"},
        {"name": "data",      "type": "string"},
        {"name": "timestamp", "type": "long", "default": 0},
        {"name": "source",    "type": ["null", "string"], "default": null}
      ]
    }
  """

  // ✗ NOT backward compatible - ลบ required field
  val removeRequiredField: String = """
    {
      "type": "record",
      "name": "Event",
      "fields": [
        {"name": "id",   "type": "string"},
        {"name": "type", "type": "string"}
      ]
    }
  """

  // ✗ NOT backward compatible - เปลี่ยน type ที่ incompatible
  val changeFieldType: String = """
    {
      "type": "record",
      "name": "Event",
      "fields": [
        {"name": "id",   "type": "long"},
        {"name": "type", "type": "string"},
        {"name": "data", "type": "bytes"}
      ]
    }
  """

  def runTests(): Unit =
    val parser = new Schema.Parser()
    val base = parser.parse(baseSchema)
    val addOpt = parser.parse(addOptionalField)
    val removeReq = parser.parse(removeRequiredField)

    println("=== Schema Compatibility Tests ===\n")
    
    val (bc1, msg1) = CompatibilityChecker.checkBackwardCompatibility(addOpt, base)
    println(s"Add optional field (new reads old): $bc1 - $msg1")
    
    val (bc2, msg2) = CompatibilityChecker.checkBackwardCompatibility(removeReq, base)
    println(s"Remove required field: $bc2 - $msg2")
```

---

## JSON Schema Versioning

### JSON Schema Evolution Patterns

```scala
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.semiauto.*
import io.circe.parser.*

// Version 1 of User
case class UserV1(
  id: Int,
  name: String,
  email: String
)

// Version 2 of User - added fields
case class UserV2(
  id: Int,
  name: String,
  email: String,
  phone: Option[String] = None,
  age: Option[Int] = None
)

// Version 3 of User - renamed and restructured
case class UserV3(
  userId: String,      // id: Int -> userId: String
  fullName: String,    // name: String -> fullName: String  
  contacts: Contacts
)

case class Contacts(
  email: String,
  phone: Option[String] = None
)

// Schema versioning strategy
object JsonSchemaVersioning:
  
  // Strategy 1: Version field ใน JSON
  def addVersionField[A: Encoder](value: A, version: Int): Json =
    val baseJson = value.asJson
    baseJson.mapObject(_.add("_schemaVersion", version.asJson))

  // Strategy 2: Wrapper ที่มี version
  case class VersionedPayload[A](
    schemaVersion: Int,
    data: A
  )
  
  given [A: Encoder]: Encoder[VersionedPayload[A]] =
    Encoder.forProduct2("schemaVersion", "data")(p => (p.schemaVersion, p.data))
  
  given [A: Decoder]: Decoder[VersionedPayload[A]] =
    Decoder.forProduct2("schemaVersion", "data")(VersionedPayload(_, _))

  // Migration functions
  def migrateV1toV2(v1: UserV1): UserV2 =
    UserV2(
      id = v1.id,
      name = v1.name,
      email = v1.email,
      phone = None,
      age = None
    )

  def migrateV2toV3(v2: UserV2): UserV3 =
    UserV3(
      userId = v2.id.toString,
      fullName = v2.name,
      contacts = Contacts(
        email = v2.email,
        phone = v2.phone
      )
    )

  // Universal decoder ที่รองรับหลาย versions
  def decodeUser(json: String): Either[String, UserV3] =
    parse(json).left.map(_.message).flatMap { parsed =>
      // ตรวจสอบ version
      val version = parsed.hcursor.downField("_schemaVersion").as[Int].getOrElse(1)
      
      version match
        case 1 =>
          parsed.as[UserV1].left.map(_.message)
            .map(migrateV1toV2)
            .map(migrateV2toV3)
        case 2 =>
          parsed.as[UserV2].left.map(_.message)
            .map(migrateV2toV3)
        case 3 =>
          parsed.as[UserV3].left.map(_.message)
        case v =>
          Left(s"Unknown schema version: $v")
    }

// JSON Schema Validation
import scala.util.{Try, Success, Failure}

object JsonSchemaValidator:
  
  // Validate ว่า JSON ตรงกับ schema
  def validate(json: String, schema: String): Either[List[String], Unit] =
    // ใช้ com.networknt.schema หรือ implement เอง
    // ตัวอย่างนี้ simplified
    val errors = scala.collection.mutable.ListBuffer[String]()
    
    parse(json) match
      case Left(err) =>
        errors += s"Invalid JSON: ${err.message}"
      case Right(json) =>
        // ตรวจสอบ required fields
        val requiredFields = List("id", "name", "email")
        requiredFields.foreach { field =>
          if json.hcursor.downField(field).failed then
            errors += s"Missing required field: $field"
        }
    
    if errors.isEmpty then Right(())
    else Left(errors.toList)

@main def jsonVersioningDemo(): Unit =
  import io.circe.generic.auto.*
  
  // v1 data
  val v1User = UserV1(1, "Alice", "alice@example.com")
  val v1Json = JsonSchemaVersioning.addVersionField(v1User, 1).noSpaces
  println(s"V1 JSON: $v1Json")
  
  // v2 data
  val v2User = UserV2(2, "Bob", "bob@example.com", Some("+66812345678"), Some(25))
  val v2Json = JsonSchemaVersioning.addVersionField(v2User, 2).noSpaces
  println(s"V2 JSON: $v2Json")
  
  // Decode both to V3
  println("\nDecoding:")
  println(s"V1 -> V3: ${JsonSchemaVersioning.decodeUser(v1Json)}")
  println(s"V2 -> V3: ${JsonSchemaVersioning.decodeUser(v2Json)}")
```

---

## Database Schema Migration ด้วย Flyway

### Flyway Setup

```scala
import org.flywaydb.core.Flyway
import org.flywaydb.core.api.configuration.FluentConfiguration
import javax.sql.DataSource

object FlywayMigration:
  
  def migrate(dataSource: DataSource): Unit =
    val flyway = Flyway.configure()
      .dataSource(dataSource)
      .locations("classpath:db/migration")
      .baselineOnMigrate(true)
      .validateOnMigrate(true)
      .load()
    
    val result = flyway.migrate()
    println(s"Applied ${result.migrationsExecuted} migrations")
    println(s"Current version: ${result.targetSchemaVersion}")

  def validate(dataSource: DataSource): Unit =
    val flyway = Flyway.configure()
      .dataSource(dataSource)
      .locations("classpath:db/migration")
      .load()
    flyway.validate()  // throws if validation fails

  def info(dataSource: DataSource): Unit =
    val flyway = Flyway.configure()
      .dataSource(dataSource)
      .locations("classpath:db/migration")
      .load()
    
    val info = flyway.info()
    info.all().foreach { migration =>
      println(s"  ${migration.getVersion}: ${migration.getDescription} [${migration.getState}]")
    }
```

### Migration Files

```sql
-- src/main/resources/db/migration/V1__create_users.sql
CREATE TABLE users (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

```sql
-- src/main/resources/db/migration/V2__add_user_profile.sql
-- เพิ่ม optional columns (backward compatible)
ALTER TABLE users
    ADD COLUMN phone      VARCHAR(50),
    ADD COLUMN age        INTEGER,
    ADD COLUMN bio        TEXT,
    ADD COLUMN updated_at TIMESTAMP;

-- เพิ่ม table ใหม่
CREATE TABLE user_addresses (
    id          SERIAL PRIMARY KEY,
    user_id     INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type        VARCHAR(20) NOT NULL DEFAULT 'home',
    street      VARCHAR(255),
    city        VARCHAR(100),
    country     VARCHAR(100),
    is_primary  BOOLEAN NOT NULL DEFAULT false,
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_user_addresses_user_id ON user_addresses(user_id);
```

```sql
-- src/main/resources/db/migration/V3__add_status.sql
-- เพิ่ม status column พร้อม default
ALTER TABLE users
    ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'active';

-- เพิ่ม check constraint
ALTER TABLE users
    ADD CONSTRAINT chk_user_status
    CHECK (status IN ('active', 'inactive', 'pending', 'banned'));

-- Update existing records
UPDATE users SET status = 'active' WHERE status IS NULL;
```

```sql
-- src/main/resources/db/migration/V4__rename_columns.sql
-- CAREFUL: นี่คือ breaking change ถ้ายังมี code เก่า!
-- ควรทำเป็น 2 steps:
-- Step 1: เพิ่ม column ใหม่ พร้อม trigger copy ค่า
-- Step 2: หลัง deploy code ใหม่ทั้งหมดแล้ว ค่อยลบ column เก่า

-- V4: Add new columns (still keep old ones)
ALTER TABLE users
    ADD COLUMN full_name VARCHAR(255),
    ADD COLUMN contact_email VARCHAR(255);

-- Copy data to new columns
UPDATE users SET
    full_name = name,
    contact_email = email;

-- Create trigger to sync (during migration period)
CREATE OR REPLACE FUNCTION sync_user_columns()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.name IS NOT NULL AND NEW.full_name IS NULL THEN
        NEW.full_name = NEW.name;
    END IF;
    IF NEW.full_name IS NOT NULL AND NEW.name IS NULL THEN
        NEW.name = NEW.full_name;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_sync_user_columns
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_user_columns();
```

```sql
-- src/main/resources/db/migration/V5__cleanup_old_columns.sql
-- หลัง deploy ใหม่ทั้งหมดแล้ว ค่อยลบ column เก่า

-- ลบ trigger และ function
DROP TRIGGER IF EXISTS trigger_sync_user_columns ON users;
DROP FUNCTION IF EXISTS sync_user_columns();

-- ลบ column เก่า (ทำหลัง code ใหม่ deploy ทั้งหมดแล้ว)
ALTER TABLE users
    DROP COLUMN IF EXISTS name,
    DROP COLUMN IF EXISTS email;

-- Rename for clarity
ALTER TABLE users
    RENAME COLUMN full_name TO name;
ALTER TABLE users
    RENAME COLUMN contact_email TO email;
```

### Repeatable Migrations

```sql
-- src/main/resources/db/migration/R__create_views.sql
-- Repeatable migration - จะ re-run เมื่อไฟล์เปลี่ยน

CREATE OR REPLACE VIEW active_users AS
SELECT 
    u.id,
    u.name,
    u.email,
    u.status,
    u.created_at,
    COUNT(a.id) as address_count
FROM users u
LEFT JOIN user_addresses a ON a.user_id = u.id
WHERE u.status = 'active'
GROUP BY u.id, u.name, u.email, u.status, u.created_at;

CREATE OR REPLACE VIEW user_summary AS
SELECT 
    status,
    COUNT(*) as user_count,
    MIN(created_at) as first_registered,
    MAX(created_at) as last_registered
FROM users
GROUP BY status;
```

### Flyway ใน cats-effect

```scala
import cats.effect.*
import com.zaxxer.hikari.{HikariConfig, HikariDataSource}
import org.flywaydb.core.Flyway
import javax.sql.DataSource

object DatabaseSetup:

  def makeDataSource(
    host: String,
    port: Int,
    database: String,
    username: String,
    password: String,
    poolSize: Int = 10
  ): Resource[IO, HikariDataSource] =
    Resource.make(
      IO {
        val config = new HikariConfig()
        config.setJdbcUrl(s"jdbc:postgresql://$host:$port/$database")
        config.setUsername(username)
        config.setPassword(password)
        config.setMaximumPoolSize(poolSize)
        config.setMinimumIdle(2)
        config.setConnectionTimeout(30000)
        new HikariDataSource(config)
      }
    )(ds =>
      IO(ds.close())
    )

  def runMigrations(ds: DataSource): IO[Int] =
    IO {
      val flyway = Flyway.configure()
        .dataSource(ds)
        .locations("classpath:db/migration")
        .validateOnMigrate(true)
        .load()
      
      val result = flyway.migrate()
      result.migrationsExecuted
    }

  def appResource: Resource[IO, HikariDataSource] =
    for
      ds <- makeDataSource(
        host = sys.env.getOrElse("DB_HOST", "localhost"),
        port = sys.env.get("DB_PORT").flatMap(_.toIntOption).getOrElse(5432),
        database = sys.env.getOrElse("DB_NAME", "myapp"),
        username = sys.env.getOrElse("DB_USER", "postgres"),
        password = sys.env.getOrElse("DB_PASSWORD", "")
      )
      _ <- Resource.eval(
        runMigrations(ds).flatMap(n =>
          IO.println(s"Applied $n migrations")
        )
      )
    yield ds
```

---

## API Versioning Strategies

### URL-based Versioning

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import io.circe.generic.auto.*
import io.circe.syntax.*

// Domain models ต่าง version
case class UserResponseV1(id: Int, name: String, email: String)
case class UserResponseV2(id: Int, name: String, email: String, phone: Option[String])
case class UserResponseV3(
  id: String,
  fullName: String,
  emailAddress: String,
  phoneNumber: Option[String],
  createdAt: String
)

// Internal domain model
case class InternalUser(
  id: Int,
  name: String,
  email: String,
  phone: Option[String],
  createdAt: java.time.Instant
)

// Transformers
object UserTransformers:
  def toV1(u: InternalUser): UserResponseV1 =
    UserResponseV1(u.id, u.name, u.email)
  
  def toV2(u: InternalUser): UserResponseV2 =
    UserResponseV2(u.id, u.name, u.email, u.phone)
  
  def toV3(u: InternalUser): UserResponseV3 =
    UserResponseV3(
      id = u.id.toString,
      fullName = u.name,
      emailAddress = u.email,
      phoneNumber = u.phone,
      createdAt = u.createdAt.toString
    )

// Routes สำหรับแต่ละ version
object UserRoutes:
  
  // Sample data
  val users = Map(
    1 -> InternalUser(1, "Alice", "alice@example.com", Some("+66812345678"), java.time.Instant.now()),
    2 -> InternalUser(2, "Bob", "bob@example.com", None, java.time.Instant.now())
  )

  def v1Routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case GET -> Root / "api" / "v1" / "users" / IntVar(id) =>
      users.get(id) match
        case Some(user) => Ok(UserTransformers.toV1(user).asJson)
        case None       => NotFound()
    
    case GET -> Root / "api" / "v1" / "users" =>
      Ok(users.values.map(UserTransformers.toV1).asJson)
  }

  def v2Routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case GET -> Root / "api" / "v2" / "users" / IntVar(id) =>
      users.get(id) match
        case Some(user) => Ok(UserTransformers.toV2(user).asJson)
        case None       => NotFound()
    
    case GET -> Root / "api" / "v2" / "users" =>
      Ok(users.values.map(UserTransformers.toV2).asJson)
  }

  def v3Routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case GET -> Root / "api" / "v3" / "users" / IntVar(id) =>
      users.get(id) match
        case Some(user) => Ok(UserTransformers.toV3(user).asJson)
        case None       => NotFound()
    
    case GET -> Root / "api" / "v3" / "users" =>
      Ok(users.values.map(UserTransformers.toV3).asJson)
  }
```

### Header-based Versioning

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.headers.Accept
import io.circe.syntax.*
import io.circe.generic.auto.*

object HeaderVersioningRoutes:

  // Accept: application/vnd.api+json; version=2
  val VersionedAccept = "application/vnd.example"
  
  def extractVersion(request: Request[IO]): Int =
    request.headers
      .get[Accept]
      .flatMap { accept =>
        accept.values.collectFirst {
          case MediaRangeAndQValue(mediaRange, _) 
            if mediaRange.mainType == "application" =>
            mediaRange.extensions.find(_.name == "version")
              .flatMap(_.value.toIntOption)
        }.flatten
      }
      .getOrElse(1)  // default to v1

  def routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case req @ GET -> Root / "users" / IntVar(id) =>
      val version = extractVersion(req)
      UserRoutes.users.get(id) match
        case None => NotFound()
        case Some(user) =>
          version match
            case 1 => Ok(UserTransformers.toV1(user).asJson)
            case 2 => Ok(UserTransformers.toV2(user).asJson)
            case 3 => Ok(UserTransformers.toV3(user).asJson)
            case v => BadRequest(s"Unknown version: $v".asJson)
  }
```

### Deprecation Strategy

```scala
import cats.effect.*
import org.http4s.*
import org.http4s.dsl.io.*
import io.circe.generic.auto.*
import io.circe.syntax.*

// Middleware สำหรับ deprecation warnings
object DeprecationMiddleware:
  
  def addDeprecationWarning(
    version: Int,
    deprecatedVersions: Set[Int],
    sunsetDate: Map[Int, String]
  ): HttpRoutes[IO] => HttpRoutes[IO] =
    routes => HttpRoutes { req =>
      routes(req).map {
        case Status.Successful(response) if deprecatedVersions.contains(version) =>
          val withWarning = response
            .putHeaders(Header.Raw(
              ci"Deprecation",
              s"true"
            ))
            .putHeaders(Header.Raw(
              ci"Warning",
              s"""299 - "This API version ($version) is deprecated""""
            ))
          
          sunsetDate.get(version).fold(withWarning) { date =>
            withWarning.putHeaders(Header.Raw(ci"Sunset", date))
          }
        case other => other
      }
    }

  // Example usage
  val deprecatedRoutes = DeprecationMiddleware.addDeprecationWarning(
    version = 1,
    deprecatedVersions = Set(1),
    sunsetDate = Map(1 -> "2025-12-31")
  )(UserRoutes.v1Routes)
```

---

## Schema Registry Patterns

### In-memory Schema Registry

```scala
import cats.effect.*
import cats.effect.std.{Mutex, Ref}
import io.circe.*
import io.circe.syntax.*
import io.circe.generic.auto.*

case class SchemaInfo(
  id: Int,
  subject: String,
  version: Int,
  schema: String,
  compatibility: String
)

class SchemaRegistry(
  schemas: Ref[IO, Map[String, List[SchemaInfo]]],
  mutex: Mutex[IO]
):
  private var nextId = 0

  def register(subject: String, schema: String): IO[SchemaInfo] =
    mutex.lock.surround {
      schemas.modify { allSchemas =>
        val versions = allSchemas.getOrElse(subject, Nil)
        val newVersion = versions.length + 1
        nextId += 1
        val info = SchemaInfo(nextId, subject, newVersion, schema, "BACKWARD")
        val updated = allSchemas + (subject -> (versions :+ info))
        (updated, info)
      }
    }

  def getLatest(subject: String): IO[Option[SchemaInfo]] =
    schemas.get.map { allSchemas =>
      allSchemas.get(subject).flatMap(_.lastOption)
    }

  def getByVersion(subject: String, version: Int): IO[Option[SchemaInfo]] =
    schemas.get.map { allSchemas =>
      allSchemas.get(subject).flatMap(_.find(_.version == version))
    }

  def getById(id: Int): IO[Option[SchemaInfo]] =
    schemas.get.map { allSchemas =>
      allSchemas.values.flatten.find(_.id == id)
    }

  def listVersions(subject: String): IO[List[Int]] =
    schemas.get.map { allSchemas =>
      allSchemas.getOrElse(subject, Nil).map(_.version)
    }

  def listSubjects: IO[List[String]] =
    schemas.get.map(_.keys.toList.sorted)

  def checkCompatibility(subject: String, newSchema: String): IO[Boolean] =
    // Simplified - should use actual Avro/JSON Schema compatibility check
    getLatest(subject).map(_.isDefined)

object SchemaRegistry:
  def make: IO[SchemaRegistry] =
    for
      schemas <- Ref.of[IO, Map[String, List[SchemaInfo]]](Map.empty)
      mutex   <- Mutex[IO]
    yield new SchemaRegistry(schemas, mutex)

// Schema-aware Kafka producer/consumer pattern
case class SchemaAwareMessage[A](
  schemaId: Int,
  payload: A
)

object SchemaAwareCodec:
  def serialize[A: Encoder](
    registry: SchemaRegistry,
    subject: String,
    value: A
  ): IO[Array[Byte]] =
    registry.getLatest(subject).flatMap {
      case None =>
        IO.raiseError(new RuntimeException(s"No schema found for: $subject"))
      case Some(schemaInfo) =>
        val message = SchemaAwareMessage(schemaInfo.id, value.asJson)
        IO.pure(message.asJson.noSpaces.getBytes("UTF-8"))
    }

  def deserialize[A: Decoder](
    registry: SchemaRegistry,
    bytes: Array[Byte]
  ): IO[A] =
    IO {
      val json = new String(bytes, "UTF-8")
      io.circe.parser.decode[SchemaAwareMessage[Json]](json)
    }.flatMap {
      case Left(err) => IO.raiseError(err)
      case Right(message) =>
        IO.fromEither(message.payload.as[A])
    }

// Demo
object SchemaRegistryDemo extends IOApp.Simple:
  
  case class Event(id: String, eventType: String, timestamp: Long) derives Encoder.AsObject, Decoder

  def run: IO[Unit] =
    SchemaRegistry.make.flatMap { registry =>
      for
        // Register schema
        schemaV1 <- registry.register(
          "com.example.Event",
          """{"type":"record","name":"Event","fields":[
             {"name":"id","type":"string"},
             {"name":"eventType","type":"string"},
             {"name":"timestamp","type":"long"}
          ]}"""
        )
        _ <- IO.println(s"Registered schema: v${schemaV1.version}, id=${schemaV1.id}")

        // List schemas
        subjects <- registry.listSubjects
        _ <- IO.println(s"Subjects: $subjects")

        // Serialize and deserialize
        event = Event("evt-001", "user.created", System.currentTimeMillis())
        bytes <- SchemaAwareCodec.serialize(registry, "com.example.Event", event)
        _ <- IO.println(s"Serialized ${bytes.length} bytes")

        decoded <- SchemaAwareCodec.deserialize[Event](registry, bytes)
        _ <- IO.println(s"Decoded: $decoded")
        _ <- IO.println(s"Round-trip: ${decoded == event}")
      yield ()
    }
```

---

## Scala Implementation ของ Schema Evolution

### Type-safe Schema Migration

```scala
import cats.effect.*

// Type-safe migration ด้วย phantom types
sealed trait SchemaVersion
sealed trait V1 extends SchemaVersion
sealed trait V2 extends SchemaVersion
sealed trait V3 extends SchemaVersion

// Versioned data
case class Versioned[V <: SchemaVersion, A](value: A)

// Migration type class
trait Migration[From <: SchemaVersion, To <: SchemaVersion, A, B]:
  def migrate(from: Versioned[From, A]): Versioned[To, B]

object Migration:
  def apply[F <: SchemaVersion, T <: SchemaVersion, A, B](
    using m: Migration[F, T, A, B]
  ): Migration[F, T, A, B] = m

// Domain types
case class UserDataV1(id: Int, name: String, email: String)
case class UserDataV2(id: Int, name: String, email: String, phone: Option[String])
case class UserDataV3(userId: String, fullName: String, contacts: Map[String, String])

// Migration instances
given Migration[V1, V2, UserDataV1, UserDataV2] with
  def migrate(from: Versioned[V1, UserDataV1]): Versioned[V2, UserDataV2] =
    Versioned(UserDataV2(
      id = from.value.id,
      name = from.value.name,
      email = from.value.email,
      phone = None
    ))

given Migration[V2, V3, UserDataV2, UserDataV3] with
  def migrate(from: Versioned[V2, UserDataV2]): Versioned[V3, UserDataV3] =
    Versioned(UserDataV3(
      userId = from.value.id.toString,
      fullName = from.value.name,
      contacts = Map("email" -> from.value.email) ++
        from.value.phone.map("phone" -> _)
    ))

// Chain migrations
def migrateChain[A, B, C](
  data: Versioned[V1, A]
)(using
  m1: Migration[V1, V2, A, B],
  m2: Migration[V2, V3, B, C]
): Versioned[V3, C] =
  m2.migrate(m1.migrate(data))

@main def typeSafeMigrationDemo(): Unit =
  val v1Data = Versioned[V1, UserDataV1](
    UserDataV1(1, "Alice", "alice@example.com")
  )
  
  val v3Data = migrateChain(v1Data)
  println(s"Migrated: ${v3Data.value}")
```

---

## Testing Schema Compatibility

```scala
import munit.*
import io.circe.*
import io.circe.parser.*
import io.circe.generic.auto.*

class SchemaCompatibilitySpec extends FunSuite:

  // Test backward compatibility
  test("UserV2 can read UserV1 JSON"):
    val v1Json = """{"id":1,"name":"Alice","email":"alice@example.com"}"""
    
    // v1 JSON should be decodable as v2 (with defaults)
    val result = decode[UserResponseV2](v1Json)
    
    result match
      case Right(user) =>
        assertEquals(user.id, 1)
        assertEquals(user.name, "Alice")
        assertEquals(user.email, "alice@example.com")
        assertEquals(user.phone, None)  // default
      case Left(err) =>
        fail(s"Should have decoded successfully: $err")

  // Test forward compatibility
  test("UserV1 reader handles UserV2 JSON (ignores unknown fields)"):
    val v2Json = """{"id":1,"name":"Alice","email":"alice@example.com","phone":"+66812345"}"""
    
    // v2 JSON should be decodable as v1 (ignoring extra fields)
    val result = decode[UserResponseV1](v2Json)
    
    result match
      case Right(user) =>
        assertEquals(user.id, 1)
        assertEquals(user.name, "Alice")
      case Left(err) =>
        fail(s"Should have decoded with ignored fields: $err")

  // Test migration
  test("Migration V1 -> V3 is correct"):
    val v1 = UserResponseV1(1, "Alice", "alice@example.com")
    val v3 = UserTransformers.toV3(
      InternalUser(v1.id, v1.name, v1.email, None, java.time.Instant.now())
    )
    
    assertEquals(v3.id, "1")
    assertEquals(v3.fullName, "Alice")
    assertEquals(v3.emailAddress, "alice@example.com")
    assertEquals(v3.phoneNumber, None)

  // Test round-trip
  test("Schema V3 round-trip encode/decode"):
    val original = UserResponseV3(
      id = "123",
      fullName = "Bob Smith",
      emailAddress = "bob@example.com",
      phoneNumber = Some("+66898765432"),
      createdAt = "2024-01-01T00:00:00Z"
    )
    
    val json = original.asJson.noSpaces
    val decoded = decode[UserResponseV3](json)
    
    assertEquals(decoded, Right(original))

  // Property-based test for schema compatibility
  test("Any V1 data can be migrated to V3"):
    val testCases = List(
      UserResponseV1(1, "Alice", "alice@example.com"),
      UserResponseV1(2, "", ""),
      UserResponseV1(Int.MaxValue, "Long Name " * 10, "a@b.c")
    )
    
    testCases.foreach { v1 =>
      val internal = InternalUser(v1.id, v1.name, v1.email, None, java.time.Instant.now())
      val v3 = UserTransformers.toV3(internal)
      
      assertEquals(v3.id, v1.id.toString)
      assertEquals(v3.fullName, v1.name)
      assertEquals(v3.emailAddress, v1.email)
    }
```

---

## Best Practices

### Schema Evolution Guidelines

```scala
// 1. ALWAYS add default values to new required fields
// BAD:
case class EventBad(
  id: String,
  eventType: String,
  newField: String  // no default - BREAKING!
)

// GOOD:
case class EventGood(
  id: String,
  eventType: String,
  newField: String = ""  // has default - safe
)

// 2. NEVER remove required fields
// BAD: Remove 'eventType' - BREAKING!

// 3. ALWAYS use nullable types for new fields in Avro
// BAD in Avro: {"name": "newField", "type": "string"}  // no default
// GOOD: {"name": "newField", "type": ["null", "string"], "default": null}

// 4. Use Option for new fields in Scala
case class SafeModel(
  existingField: String,
  newOptional: Option[String] = None  // safe addition
)

// 5. Document deprecation
@deprecated("Use UserV3 instead. Will be removed in 2025-Q4.", "2.0.0")
case class UserV1Legacy(id: Int, name: String, email: String)

// 6. Maintain migration paths
object MigrationPaths:
  val supportedMigrations = Map(
    "1→2" -> "Add phone and age fields",
    "2→3" -> "Restructure contacts",
    "1→3" -> "Full migration via v2"
  )
```

### Schema Version Management Tool

```scala
import cats.effect.*

object SchemaVersionManager extends IOApp.Simple:
  
  case class SchemaChange(
    version: String,
    description: String,
    breaking: Boolean,
    migrationRequired: Boolean
  )
  
  val changelog = List(
    SchemaChange("2.0.0", "Add optional phone field", false, false),
    SchemaChange("2.1.0", "Add age field with default 0", false, false),
    SchemaChange("3.0.0", "Rename id to userId, restructure contacts", true, true),
    SchemaChange("3.1.0", "Add metadata object", false, false)
  )
  
  def printChangelog: IO[Unit] =
    IO.println("=== Schema Changelog ===") >>
    changelog.foldLeft(IO.unit) { (acc, change) =>
      acc >>
      IO.println(s"""
        |Version: ${change.version}
        |  Description: ${change.description}
        |  Breaking: ${change.breaking}
        |  Migration Required: ${change.migrationRequired}
        |""".stripMargin)
    }
  
  def checkForBreakingChanges(from: String, to: String): IO[List[SchemaChange]] =
    IO {
      changelog.filter { change =>
        change.breaking && change.version > from && change.version <= to
      }
    }
  
  def run: IO[Unit] =
    printChangelog >>
    checkForBreakingChanges("2.0.0", "3.1.0").flatMap { breaking =>
      if breaking.isEmpty then
        IO.println("No breaking changes found!")
      else
        IO.println(s"WARNING: ${breaking.size} breaking change(s) found:") >>
        breaking.traverse(c => IO.println(s"  - ${c.version}: ${c.description}")).void
    }
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ Schema Evolution และ Compatibility:

### สิ่งที่ได้เรียนรู้

| หัวข้อ | เนื้อหา |
|--------|---------|
| Avro Schema | การ define และ evolve schemas อย่างปลอดภัย |
| Compatibility | Backward/Forward/Full compatibility |
| JSON Schema | Versioning และ migration patterns |
| Flyway | Database schema migration แบบ version-controlled |
| API Versioning | URL-based, Header-based, deprecation strategies |
| Schema Registry | Central store สำหรับ schemas |
| Type-safe Migration | Phantom types สำหรับ compile-time safety |

### กฎที่ Safe สำหรับ Schema Evolution

**ทำได้ (Non-breaking):**
- ✅ เพิ่ม optional field พร้อม default value
- ✅ เพิ่ม enum value ใหม่ (ระวัง reader เก่าอาจไม่รู้จัก)
- ✅ เปลี่ยน type ที่ compatible (int → long)

**ห้ามทำ (Breaking):**
- ❌ ลบ field ที่จำเป็น
- ❌ เปลี่ยน type ที่ incompatible (string → int)
- ❌ เปลี่ยนชื่อ field โดยไม่มี migration plan

---

*[← Part 57: Effect Patterns](part-57-effect-patterns.md) | [Part 59: Configuration Management →](part-59-configuration.md)*
