# Part 19: Type Classes Pattern เชิงลึก

## สารบัญ
1. [Type Class Design](#type-class-design)
2. [Common Type Classes](#common-type-classes)
3. [Deriving Type Classes](#deriving-type-classes)
4. [Coherence and Orphans](#coherence-and-orphans)
5. [Practical Applications](#practical-applications)

---

## Type Class Design

### Type Class Structure

```scala
// Type Class มี 3 ส่วน:
// 1. Type class trait
// 2. Instances (given)
// 3. Interface (methods/extensions ที่ user ใช้)

// 1. Type class
trait Show[A]:
  def show(a: A): String

// 2. Instances
object Show:
  // Summoner
  def apply[A: Show]: Show[A] = summon[Show[A]]

  // Basic instances
  given Show[Int] with
    def show(n: Int): String = n.toString

  given Show[Double] with
    def show(d: Double): String = f"$d%.6f"

  given Show[String] with
    def show(s: String): String = s""""$s""""

  given Show[Boolean] with
    def show(b: Boolean): String = if b then "true" else "false"

  // Derived instances
  given [A: Show]: Show[Option[A]] with
    def show(opt: Option[A]): String = opt match
      case Some(a) => s"Some(${Show[A].show(a)})"
      case None    => "None"

  given [A: Show]: Show[List[A]] with
    def show(list: List[A]): String =
      list.map(Show[A].show).mkString("List(", ", ", ")")

  given [A: Show, B: Show]: Show[(A, B)] with
    def show(pair: (A, B)): String =
      s"(${Show[A].show(pair._1)}, ${Show[B].show(pair._2)})"

// 3. Interface
extension [A: Show](a: A)
  def show: String = Show[A].show(a)
  def println(): Unit = Predef.println(a.show)

// Usage
println(42.show)                          // 42
println("hello".show)                    // "hello"
println(List(1, 2, 3).show)             // List(1, 2, 3)
println((true, "world").show)           // (true, "world")
println(Some(List(1, 2)).show)          // Some(List(1, 2))
```

---

## Common Type Classes

### Ordering Type Class

```scala
// Scala's built-in Ordering type class

case class Person(name: String, age: Int, salary: Double)

// Custom Ordering
given Ordering[Person] = Ordering.by(_.name)

val people = List(
  Person("Charlie", 35, 75000),
  Person("Alice", 30, 90000),
  Person("Bob", 25, 60000)
)

println(people.sorted)
// sorted by name: Alice, Bob, Charlie

// Multi-field ordering
val byAgeDesc = Ordering.by[Person, Int](_.age).reverse
val bySalaryAsc = Ordering.by[Person, Double](_.salary)

// Compound ordering
val byAgeThenSalary = Ordering.by((p: Person) => (-p.age, p.salary))

println(people.sorted(byAgeThenSalary).map(p => s"${p.name}(${p.age})"))
// Charlie(35), Alice(30), Bob(25)

// Min/max with custom ordering
println(people.maxBy(_.salary))  // Alice
println(people.minBy(_.age))     // Bob
println(people.sorted(byAgeDesc).head)  // Charlie
```

### Numeric Type Class

```scala
// Scala's built-in Numeric type class

def statistics[A: Numeric](numbers: Seq[A]): (A, A, Double) =
  val n = summon[Numeric[A]]
  val sum = numbers.reduce(n.plus)
  val max = numbers.reduce(n.max)
  val avg = n.toDouble(sum) / numbers.length
  (sum, max, avg)

val ints = Seq(1, 2, 3, 4, 5)
val doubles = Seq(1.5, 2.5, 3.5)

println(statistics(ints))     // (15, 5, 3.0)
println(statistics(doubles))  // (7.5, 3.5, 2.5)

// Generic arithmetic
def normalize[A: Numeric](values: Seq[A]): Seq[Double] =
  val n = summon[Numeric[A]]
  val min = values.min
  val max = values.max
  val range = n.toDouble(n.minus(max, min))
  if range == 0 then values.map(_ => 0.0)
  else values.map(v => (n.toDouble(v) - n.toDouble(min)) / range)

println(normalize(Seq(1, 5, 3, 7, 2)))
// Seq(0.0, 0.667, 0.333, 1.0, 0.167)
```

### Hashing Type Class

```scala
trait Hash[A] extends Eq[A]:
  def hash(a: A): Int

trait Eq[A]:
  def eqv(x: A, y: A): Boolean

object Hash:
  given Hash[Int] with
    def hash(n: Int): Int = n.hashCode
    def eqv(x: Int, y: Int): Boolean = x == y

  given Hash[String] with
    def hash(s: String): Int = s.hashCode
    def eqv(x: String, y: String): Boolean = x == y

  given [A: Hash, B: Hash]: Hash[(A, B)] with
    def hash(pair: (A, B)): Int =
      31 * summon[Hash[A]].hash(pair._1) + summon[Hash[B]].hash(pair._2)
    def eqv(x: (A, B), y: (A, B)): Boolean =
      summon[Hash[A]].eqv(x._1, y._1) && summon[Hash[B]].eqv(x._2, y._2)

// Generic HashMap using type class
class TypedHashMap[K: Hash, V]:
  private val underlying = scala.collection.mutable.HashMap[Int, List[(K, V)]]()

  def put(k: K, v: V): Unit =
    val h = summon[Hash[K]].hash(k)
    val bucket = underlying.getOrElse(h, Nil)
    val filtered = bucket.filter { case (bk, _) => !summon[Hash[K]].eqv(bk, k) }
    underlying(h) = (k, v) :: filtered

  def get(k: K): Option[V] =
    val h = summon[Hash[K]].hash(k)
    underlying.getOrElse(h, Nil)
      .find { case (bk, _) => summon[Hash[K]].eqv(bk, k) }
      .map(_._2)
```

---

## Deriving Type Classes

### Scala 3 Deriving

```scala
// Scala 3 supports automatic derivation for some type classes
import scala.deriving.*
import scala.compiletime.*

// Case class กับ derived
case class Point(x: Double, y: Double) derives Show, Eq, Ordering

// Manually derived Show สำหรับ case classes
inline def summonAll[T <: Tuple]: List[Show[?]] =
  inline erasedValue[T] match
    case _: EmptyTuple => Nil
    case _: (t *: ts) => summon[Show[t]] :: summonAll[ts]

// สมมติว่ามี derivation mechanism
trait Eq[A]:
  def eqv(x: A, y: A): Boolean

object Eq:
  given Eq[Int] with
    def eqv(x: Int, y: Int): Boolean = x == y
  given Eq[String] with
    def eqv(x: String, y: String): Boolean = x == y
  given [A: Eq, B: Eq]: Eq[(A, B)] with
    def eqv(x: (A, B), y: (A, B)): Boolean =
      summon[Eq[A]].eqv(x._1, y._1) && summon[Eq[B]].eqv(x._2, y._2)
```

---

## Practical Applications

### Serialization Framework

```scala
// JSON Codec type class
trait JsonDecoder[A]:
  def decode(json: String): Either[String, A]

trait JsonEncoder[A]:
  def encode(a: A): String

trait JsonCodec[A] extends JsonEncoder[A] with JsonDecoder[A]

object JsonEncoder:
  given JsonEncoder[Int] with
    def encode(n: Int): String = n.toString

  given JsonEncoder[String] with
    def encode(s: String): String =
      "\"" + s.replace("\\", "\\\\").replace("\"", "\\\"") + "\""

  given JsonEncoder[Boolean] with
    def encode(b: Boolean): String = b.toString

  given JsonEncoder[Double] with
    def encode(d: Double): String = d.toString

  given [A: JsonEncoder]: JsonEncoder[Option[A]] with
    def encode(opt: Option[A]): String = opt match
      case Some(a) => summon[JsonEncoder[A]].encode(a)
      case None    => "null"

  given [A: JsonEncoder]: JsonEncoder[List[A]] with
    def encode(list: List[A]): String =
      list.map(summon[JsonEncoder[A]].encode).mkString("[", ",", "]")

  given [K: JsonEncoder, V: JsonEncoder]: JsonEncoder[Map[K, V]] with
    def encode(map: Map[K, V]): String =
      val kEnc = summon[JsonEncoder[K]]
      val vEnc = summon[JsonEncoder[V]]
      map.map { case (k, v) => s"${kEnc.encode(k)}:${vEnc.encode(v)}" }
        .mkString("{", ",", "}")

extension [A: JsonEncoder](a: A)
  def toJson: String = summon[JsonEncoder[A]].encode(a)

// Usage
case class User(name: String, age: Int, active: Boolean)

// Manual encoder for User
given JsonEncoder[User] with
  def encode(u: User): String =
    Map(
      "\"name\"" -> u.name.toJson,
      "\"age\"" -> u.age.toJson,
      "\"active\"" -> u.active.toJson
    ).map { case (k, v) => s"$k:$v" }.mkString("{", ",", "}")

val user = User("Alice", 30, true)
println(user.toJson)
// {"name":"Alice","age":30,"active":true}

val users = List(User("Alice", 30, true), User("Bob", 25, false))
println(users.toJson)
```

### Validation Framework

```scala
// Applicative Validation Type Class

trait Validate[A]:
  def validate(a: A): Either[List[String], A]

object Validate:
  def apply[A: Validate]: Validate[A] = summon[Validate[A]]

  def check[A](pred: A => Boolean, msg: A => String): Validate[A] =
    a => if pred(a) then Right(a) else Left(List(msg(a)))

  def and[A](v1: Validate[A], v2: Validate[A]): Validate[A] =
    a => (v1.validate(a), v2.validate(a)) match
      case (Right(_), Right(_)) => Right(a)
      case (Left(e1), Left(e2)) => Left(e1 ++ e2)
      case (Left(e), Right(_))  => Left(e)
      case (Right(_), Left(e))  => Left(e)

// DSL for building validators
case class Validator[A](private val underlying: Validate[A]):
  def and(other: Validate[A]): Validator[A] =
    Validator(Validate.and(underlying, other))
  def validate(a: A): Either[List[String], A] = underlying.validate(a)

object V:
  def minLength(n: Int): Validate[String] =
    Validate.check(_.length >= n, s => s"Too short: '$s' (min $n chars)")

  def maxLength(n: Int): Validate[String] =
    Validate.check(_.length <= n, s => s"Too long: '$s' (max $n chars)")

  def matches(regex: String): Validate[String] =
    Validate.check(_.matches(regex), s => s"'$s' doesn't match $regex")

  def min(n: Int): Validate[Int] =
    Validate.check(_ >= n, v => s"$v < min $n")

  def max(n: Int): Validate[Int] =
    Validate.check(_ <= n, v => s"$v > max $n")

// Combining validators
val nameValidator = Validator(V.minLength(2))
  .and(V.maxLength(50))
  .and(V.matches("[a-zA-Z ]+"))

val ageValidator = Validator(V.min(0)).and(V.max(150))

println(nameValidator.validate("Alice"))    // Right(Alice)
println(nameValidator.validate(""))        // Left(List(Too short: '' ...))
println(nameValidator.validate("Alice123")) // Left(List('Alice123' doesn't match ...))
println(ageValidator.validate(25))          // Right(25)
println(ageValidator.validate(-5))          // Left(List(-5 < min 0))
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Type Class Design: trait + given instances + interface
- ✅ Common Type Classes: Show, Ordering, Numeric, Hash, Eq
- ✅ Type Class Derivation
- ✅ Practical Applications: Serialization, Validation frameworks
- ✅ Coherence และ Orphan instances

---

*[← Part 18: Implicits](part-18-implicits.md) | [Part 20: Futures และ Concurrency →](part-20-futures-concurrency.md)*
