# Part 16: Option และ Either เชิงลึก

## สารบัญ
1. [Option เชิงลึก](#option-เชิงลึก)
2. [Either เชิงลึก](#either-เชิงลึก)
3. [Try](#try)
4. [Combining Error Types](#combining-error-types)
5. [Practical Patterns](#practical-patterns)

---

## Option เชิงลึก

### Option API ครบถ้วน

```scala
val some: Option[Int] = Some(42)
val none: Option[Int] = None

// Extracting values
println(some.get)                  // 42 (throw if None)
println(some.getOrElse(0))         // 42
println(none.getOrElse(0))         // 0
println(some.orElse(Some(99)))     // Some(42)
println(none.orElse(Some(99)))     // Some(99)
println(some.fold("none")(_.toString))  // "42"
println(none.fold("none")(_.toString))  // "none"

// Transformations
println(some.map(_ * 2))           // Some(84)
println(none.map(_ * 2))           // None
println(some.flatMap(n => if n > 0 then Some(n) else None))  // Some(42)
println(some.filter(_ > 100))      // None
println(some.filter(_ < 100))      // Some(42)
println(some.filterNot(_ > 100))   // Some(42)

// Existence checks
println(some.isDefined)            // true
println(none.isEmpty)              // true
println(some.contains(42))         // true
println(some.exists(_ > 40))       // true
println(some.forall(_ > 40))       // true

// Conversion
println(some.toList)               // List(42)
println(none.toList)               // List()
println(some.toRight("error"))     // Right(42)
println(none.toRight("error"))     // Left(error)
println(some.toLeft("default"))    // Left(42)
println(none.toLeft("default"))    // Right(default)

// zip
println(Some(1).zip(Some("a")))    // Some((1,a))
println(Some(1).zip(None))         // None
```

### Pattern Matching กับ Option

```scala
def processOption[A, B](opt: Option[A])(f: A => B)(default: => B): B =
  opt match
    case Some(value) => f(value)
    case None        => default

// Chaining Options
case class Company(name: String, ceoId: Option[Int])
case class Person(id: Int, name: String, managerId: Option[Int])

val persons = Map(
  1 -> Person(1, "Alice", None),       // CEO
  2 -> Person(2, "Bob", Some(1)),       // reports to Alice
  3 -> Person(3, "Charlie", Some(2))    // reports to Bob
)

val companies = Map(
  "TechCorp" -> Company("TechCorp", Some(1))
)

def getCeoName(companyName: String): Option[String] =
  for
    company <- companies.get(companyName)
    ceoId   <- company.ceoId
    ceo     <- persons.get(ceoId)
  yield ceo.name

println(getCeoName("TechCorp"))    // Some(Alice)
println(getCeoName("Unknown"))     // None
```

### Option กับ Collections

```scala
val items = List(Some(1), None, Some(3), None, Some(5))

// flatten: ดึงเฉพาะ Some values
println(items.flatten)         // List(1, 3, 5)

// flatMap กับ Option
val strings = List("1", "abc", "3", "def", "5")
val numbers = strings.flatMap(_.toIntOption)
println(numbers)               // List(1, 3, 5)

// sequence: List[Option[A]] -> Option[List[A]]
def sequence[A](opts: List[Option[A]]): Option[List[A]] =
  opts.foldRight[Option[List[A]]](Some(Nil)) {
    case (Some(a), Some(acc)) => Some(a :: acc)
    case _ => None
  }

println(sequence(List(Some(1), Some(2), Some(3))))  // Some(List(1, 2, 3))
println(sequence(List(Some(1), None, Some(3))))      // None

// traverse
def traverse[A, B](list: List[A])(f: A => Option[B]): Option[List[B]] =
  list.foldRight[Option[List[B]]](Some(Nil)) { (a, acc) =>
    for
      b  <- f(a)
      bs <- acc
    yield b :: bs
  }

val result = traverse(List("1", "2", "3"))(_.toIntOption)
println(result)  // Some(List(1, 2, 3))
```

---

## Either เชิงลึก

### Either API

```scala
val right: Either[String, Int] = Right(42)
val left: Either[String, Int] = Left("error")

// Transformations (right-biased)
println(right.map(_ * 2))                // Right(84)
println(left.map(_ * 2))                 // Left(error)
println(right.flatMap(n => Right(n + 1))) // Right(43)
println(left.flatMap(n => Right(n + 1)))  // Left(error)

// Error transformation
println(right.left.map(_.toUpperCase))    // Right(42)
println(left.left.map(_.toUpperCase))     // Left(ERROR)

// fold
println(right.fold(e => s"Error: $e", n => s"Value: $n"))  // Value: 42
println(left.fold(e => s"Error: $e", n => s"Value: $n"))   // Error: error

// swap
println(right.swap)  // Left(42)
println(left.swap)   // Right(error)

// getOrElse
println(right.getOrElse(0))  // 42
println(left.getOrElse(0))   // 0

// toOption
println(right.toOption)  // Some(42)
println(left.toOption)   // None

// Predicates
println(right.isRight)  // true
println(left.isLeft)    // true
println(right.exists(_ > 40))   // true
println(right.forall(_ > 40))   // true
```

### Error Accumulation กับ Either

```scala
// Either short-circuits - ใช้ Validated สำหรับ accumulation

case class UserInput(name: String, age: String, email: String)
case class User(name: String, age: Int, email: String)

type Errors = List[String]
type Validated[A] = Either[Errors, A]

def validateName(name: String): Validated[String] =
  if name.nonEmpty && name.length >= 2
  then Right(name.trim)
  else Left(List(s"Invalid name: '$name'"))

def validateAge(ageStr: String): Validated[Int] =
  ageStr.toIntOption match
    case Some(age) if age >= 0 && age <= 150 => Right(age)
    case Some(age) => Left(List(s"Age out of range: $age"))
    case None => Left(List(s"Invalid age: '$ageStr'"))

def validateEmail(email: String): Validated[String] =
  if email.matches("""^[^@]+@[^@]+\.[^@]+$""")
  then Right(email.toLowerCase)
  else Left(List(s"Invalid email: '$email'"))

// Combine validations (accumulate errors)
def combineValidated[A, B, C](
  va: Validated[A],
  vb: Validated[B]
)(f: (A, B) => C): Validated[C] =
  (va, vb) match
    case (Right(a), Right(b))   => Right(f(a, b))
    case (Left(e1), Left(e2))   => Left(e1 ++ e2)
    case (Left(e), Right(_))    => Left(e)
    case (Right(_), Left(e))    => Left(e)

def validateUser(input: UserInput): Validated[User] =
  combineValidated(
    combineValidated(validateName(input.name), validateAge(input.age))(
      (name, age) => (name, age)
    ),
    validateEmail(input.email)
  ) { case ((name, age), email) => User(name, age, email) }

val valid = UserInput("Alice", "30", "alice@example.com")
val invalid = UserInput("", "abc", "not-an-email")

println(validateUser(valid))
// Right(User(Alice,30,alice@example.com))
println(validateUser(invalid))
// Left(List(Invalid name: '', Invalid age: 'abc', Invalid email: 'not-an-email'))
```

---

## Try

### Try API

```scala
import scala.util.{Try, Success, Failure}

// Try wraps exceptions
val success = Try(42)
val failure = Try(throw new RuntimeException("oops"))
val divByZero = Try(10 / 0)

println(success)    // Success(42)
println(failure)    // Failure(java.lang.RuntimeException: oops)
println(divByZero)  // Failure(java.lang.ArithmeticException: / by zero)

// Transformations
println(success.map(_ * 2))               // Success(84)
println(failure.map(_ * 2))               // Failure(...)
println(success.flatMap(n => Try(n / 2))) // Success(21)

// Recovering
val recovered = divByZero.recover {
  case _: ArithmeticException => 0
}
println(recovered)  // Success(0)

val recoveredWith = divByZero.recoverWith {
  case _: ArithmeticException => Try(-1)
}
println(recoveredWith)  // Success(-1)

// Convert
println(success.toOption)   // Some(42)
println(failure.toOption)   // None
println(success.toEither)   // Right(42)
println(failure.toEither)   // Left(RuntimeException: oops)

// fold
val result = divByZero.fold(
  ex => s"Error: ${ex.getMessage}",
  n => s"Result: $n"
)
println(result)  // Error: / by zero
```

---

## Combining Error Types

### Converting Between Option, Either, Try

```scala
// Option <-> Either
val opt: Option[Int] = Some(42)
val either: Either[String, Int] = opt.toRight("Value was None")
val optBack: Option[Int] = either.toOption

// Try -> Either
val tried: Try[Int] = Try(Integer.parseInt("42"))
val asEither: Either[Throwable, Int] = tried.toEither

// Lifting functions
def liftOption[A, B](f: A => B): A => Option[B] = a => Try(f(a)).toOption

val safeParse = liftOption[String, Int](_.toInt)
println(safeParse("42"))    // Some(42)
println(safeParse("abc"))   // None
```

---

## Practical Patterns

### Repository Pattern กับ Either

```scala
import scala.util.Try

sealed trait AppError
case class NotFound(id: String) extends AppError
case class ValidationError(msg: String) extends AppError
case class DatabaseError(msg: String, cause: Throwable) extends AppError

case class Product(id: String, name: String, price: Double)

// Repository trait
trait ProductRepository:
  def findById(id: String): Either[AppError, Product]
  def save(product: Product): Either[AppError, Product]
  def delete(id: String): Either[AppError, Unit]

// In-memory implementation
class InMemoryProductRepository extends ProductRepository:
  private var products = Map[String, Product]()

  def findById(id: String): Either[AppError, Product] =
    products.get(id).toRight(NotFound(id))

  def save(product: Product): Either[AppError, Product] =
    if product.name.isEmpty
    then Left(ValidationError("Product name cannot be empty"))
    else if product.price < 0
    then Left(ValidationError("Price cannot be negative"))
    else
      products = products.updated(product.id, product)
      Right(product)

  def delete(id: String): Either[AppError, Unit] =
    if products.contains(id)
    then
      products = products.removed(id)
      Right(())
    else Left(NotFound(id))

// Service layer
class ProductService(repo: ProductRepository):
  def getProductDetails(id: String): Either[AppError, String] =
    for product <- repo.findById(id)
    yield f"${product.name}: $$${product.price}%.2f"

  def applyDiscount(id: String, percent: Double): Either[AppError, Product] =
    for
      product     <- repo.findById(id)
      _           <- if percent < 0 || percent > 100
                     then Left(ValidationError(s"Invalid discount: $percent"))
                     else Right(())
      discounted  = product.copy(price = product.price * (1 - percent / 100))
      saved       <- repo.save(discounted)
    yield saved

val repo = InMemoryProductRepository()
val service = ProductService(repo)

repo.save(Product("P1", "Laptop", 1000.0))
repo.save(Product("P2", "Mouse", 25.0))

println(service.getProductDetails("P1"))   // Right(Laptop: $1000.00)
println(service.getProductDetails("P99"))  // Left(NotFound(P99))
println(service.applyDiscount("P1", 10))   // Right(Product(P1,Laptop,900.0))
println(service.applyDiscount("P1", -5))   // Left(ValidationError(Invalid discount: -5.0))
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Option API ครบถ้วน: map, flatMap, filter, fold, getOrElse
- ✅ Either API: right-biased operations
- ✅ Error Accumulation Pattern
- ✅ Try: recovering from exceptions
- ✅ Converting between Option, Either, Try
- ✅ Repository Pattern กับ Either

---

*[← Part 15: For Comprehensions](part-15-for-comprehensions.md) | [Part 17: Collections Advanced →](part-17-collections-advanced.md)*
