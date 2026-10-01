# Part 14: Higher-Order Functions เชิงลึก

## สารบัญ
1. [HOF ทบทวน](#hof-ทบทวน)
2. [Collection Operations เชิงลึก](#collection-operations-เชิงลึก)
3. [Partial Application และ Currying](#partial-application-และ-currying)
4. [Trampolining](#trampolining)
5. [Lazy Evaluation](#lazy-evaluation)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## HOF ทบทวน

### Functions เป็น First-Class Values

```scala
// Function types
val double: Int => Int = _ * 2
val add: (Int, Int) => Int = _ + _
val greet: String => String = name => s"Hello, $name!"

// Passing functions
def applyTwice[A](f: A => A, a: A): A = f(f(a))
println(applyTwice(double, 3))  // 12

// Returning functions
def multiplier(factor: Int): Int => Int = _ * factor
val triple = multiplier(3)
val quadruple = multiplier(4)
println(triple(5))      // 15
println(quadruple(5))   // 20

// Functions in data structures
val transformations: List[Int => Int] = List(
  _ + 1,
  _ * 2,
  n => n * n,
  _ - 5
)

val applyAll = transformations.foldLeft(identity[Int] _)(_ andThen _)
println(applyAll(3))  // ((3+1)*2)^2-5 = 59
```

---

## Collection Operations เชิงลึก

### map, flatMap, filter

```scala
val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// map: transform every element
val squared = nums.map(n => n * n)
println(squared)  // List(1, 4, 9, 16, 25, 36, 49, 64, 81, 100)

// flatMap: map + flatten
val nestedLists = List(List(1, 2), List(3, 4), List(5, 6))
val flattened = nestedLists.flatMap(identity)
println(flattened)  // List(1, 2, 3, 4, 5, 6)

// Generating pairs
val pairs = (1 to 3).flatMap(x => (1 to 3).map(y => (x, y)))
println(pairs)  // Vector((1,1),(1,2),(1,3),(2,1),...,(3,3))

// filter + map
val evenSquares = nums.filter(_ % 2 == 0).map(n => n * n)
println(evenSquares)  // List(4, 16, 36, 64, 100)

// collect = partial function (filter + map)
val oddSquares = nums.collect {
  case n if n % 2 != 0 => n * n
}
println(oddSquares)  // List(1, 9, 25, 49, 81)
```

### fold, reduce, scan

```scala
val words = List("Scala", "is", "awesome", "and", "powerful")

// foldLeft: สะสมผลจากซ้าย
val concatenated = words.foldLeft("")((acc, w) =>
  if acc.isEmpty then w else s"$acc $w")
println(concatenated)  // Scala is awesome and powerful

// foldRight: สะสมผลจากขวา
val reversed = words.foldRight(List.empty[String])((w, acc) => acc :+ w)
println(reversed)

// reduce: ต้องการ non-empty list
val maxLength = words.map(_.length).reduce(math.max)
println(maxLength)  // 8 ("powerful")

// scan: สะสม intermediate results
val runningSum = (1 to 5).scan(0)(_ + _)
println(runningSum)  // Vector(0, 1, 3, 6, 10, 15)

val runningProduct = (1 to 5).scanLeft(1)(_ * _)
println(runningProduct)  // Vector(1, 1, 2, 6, 24, 120)
```

### groupBy, partition, span, splitAt

```scala
val people = List(
  ("Alice", 30, "Engineering"),
  ("Bob", 25, "Marketing"),
  ("Charlie", 35, "Engineering"),
  ("Diana", 28, "Marketing"),
  ("Eve", 32, "Engineering")
)

// groupBy
val byDepartment = people.groupBy(_._3)
byDepartment.foreach { case (dept, members) =>
  println(s"$dept: ${members.map(_._1).mkString(", ")}")
}
// Engineering: Alice, Charlie, Eve
// Marketing: Bob, Diana

// partition: split into two lists by predicate
val (seniors, juniors) = people.partition(_._2 >= 30)
println(s"Seniors: ${seniors.map(_._1)}")  // List(Alice, Charlie, Eve)
println(s"Juniors: ${juniors.map(_._1)}")  // List(Bob, Diana)

// span: take while + drop while
val nums = List(1, 2, 3, 4, 5, 1, 2)
val (before, from) = nums.span(_ < 4)
println(before)  // List(1, 2, 3)
println(from)    // List(4, 5, 1, 2)

// splitAt
val (left, right) = nums.splitAt(3)
println(left)   // List(1, 2, 3)
println(right)  // List(4, 5, 1, 2)
```

### zip, unzip, zipWithIndex

```scala
val names = List("Alice", "Bob", "Charlie")
val scores = List(95, 87, 92)
val grades = List("A", "B+", "A")

// zip
val nameScore = names.zip(scores)
println(nameScore)  // List((Alice,95),(Bob,87),(Charlie,92))

// zip with multiple lists using zip+zip
val combined = names.zip(scores).zip(grades).map {
  case ((name, score), grade) => (name, score, grade)
}
println(combined)  // List((Alice,95,A),(Bob,87,B+),(Charlie,92,A))

// unzip
val (ns, ss) = nameScore.unzip
println(ns)  // List(Alice, Bob, Charlie)
println(ss)  // List(95, 87, 92)

// zipWithIndex
names.zipWithIndex.foreach { case (name, i) =>
  println(s"${i + 1}. $name")
}
// 1. Alice
// 2. Bob
// 3. Charlie

// zipAll (pad with default if lengths differ)
val long = List(1, 2, 3, 4, 5)
val short = List("a", "b", "c")
println(long.zipAll(short, 0, "-"))
// List((1,a),(2,b),(3,c),(4,-),(5,-))
```

---

## Partial Application และ Currying

### Partial Application

```scala
// Partial application: บาง arguments

def add(x: Int, y: Int): Int = x + y
val add5 = add(5, _: Int)  // partial application
println(add5(3))   // 8
println(add5(10))  // 15

// Method to partially applied function
def pow(base: Int, exp: Int): Int =
  math.pow(base, exp).toInt

val square = pow(_: Int, 2)
val cube = pow(_: Int, 3)
val powerOf2 = pow(2, _: Int)

println(square(4))    // 16
println(cube(3))      // 27
println(powerOf2(8))  // 256

// Practical use
val logDebug = println _  // method to function
val numbers = List(1, 2, 3, 4, 5)
numbers.foreach(logDebug)
```

### Currying

```scala
// Curried function: multiple argument lists

def multiply(x: Int)(y: Int): Int = x * y

val double = multiply(2)    // Int => Int
val triple = multiply(3)    // Int => Int

println(double(5))   // 10
println(triple(5))   // 15

// Practical currying examples

// Logger with context
def logWithContext(level: String)(context: String)(msg: String): Unit =
  println(s"[$level][$context] $msg")

val debugLog = logWithContext("DEBUG")
val infoLog  = logWithContext("INFO")
val authDebug = debugLog("auth")
val paymentInfo = infoLog("payment")

authDebug("User logged in")      // [DEBUG][auth] User logged in
paymentInfo("Payment processed") // [INFO][payment] Payment processed

// Configuration-based functions
def httpRequest(method: String)(url: String)(body: Option[String]): String =
  s"$method $url ${body.getOrElse("")}"

val get = httpRequest("GET")
val post = httpRequest("POST")
val postToApi = post("https://api.example.com")

println(get("https://api.example.com/users")(None))
println(postToApi(Some("""{"name":"Alice"}""")))
```

### Currying กับ Collections

```scala
// Curried functions ทำงานดีกับ HOFs

def between(min: Int)(max: Int)(n: Int): Boolean =
  n >= min && n <= max

val isPositive = between(1)(Int.MaxValue) _
val isTeen = between(13)(19) _
val isAdult = between(18)(64) _

val ages = List(5, 13, 18, 25, 17, 30, 65)
println(ages.filter(isPositive))  // All positive
println(ages.filter(isTeen))      // List(13, 17)
println(ages.filter(isAdult))     // List(18, 25, 30)

// Function factory pattern
def validator[A](checks: (A => Boolean)*): A => Boolean =
  a => checks.forall(_(a))

val validateAge = validator[Int](
  _ >= 0,
  _ <= 150,
  _ != 13  // superstition
)

println(validateAge(25))   // true
println(validateAge(-1))   // false
println(validateAge(13))   // false
```

---

## Trampolining

### Tail Recursion กับ Trampoline

```scala
import scala.annotation.tailrec

// ปัญหา: mutual recursion ไม่ใช่ tail-recursive
def isEven(n: Int): Boolean =
  if n == 0 then true else isOdd(n - 1)

def isOdd(n: Int): Boolean =
  if n == 0 then false else isEven(n - 1)

// isEven(100000) จะ StackOverflow!

// แก้ด้วย Trampoline
sealed trait Trampoline[+A]:
  @tailrec
  final def run: A = this match
    case Done(a) => a
    case More(f) => f().run

case class Done[A](a: A) extends Trampoline[A]
case class More[A](f: () => Trampoline[A]) extends Trampoline[A]

def isEvenT(n: Int): Trampoline[Boolean] =
  if n == 0 then Done(true) else More(() => isOddT(n - 1))

def isOddT(n: Int): Trampoline[Boolean] =
  if n == 0 then Done(false) else More(() => isEvenT(n - 1))

println(isEvenT(100000).run)  // true - ไม่ StackOverflow
println(isOddT(100001).run)   // true

// Trampoline กับ FlatMap
sealed trait Bounce[+A]:
  def map[B](f: A => B): Bounce[B] = this.flatMap(a => Return(f(a)))
  def flatMap[B](f: A => Bounce[B]): Bounce[B] = Cont(this, f)

  @tailrec
  final def step: Either[() => Bounce[A], A] = this match
    case Return(a)     => Right(a)
    case Suspend(f)    => Left(f)
    case Cont(sub, f) => sub match
      case Return(a)    => f(a).step
      case Suspend(g)   => Left(() => Cont(g(), f))
      case Cont(sub2, g) => Cont(sub2, x => Cont(g(x), f)).step

  def run: A =
    @tailrec def go(bounce: Bounce[A]): A =
      bounce.step match
        case Right(a) => a
        case Left(f)  => go(f())
    go(this)

case class Return[A](a: A) extends Bounce[A]
case class Suspend[A](f: () => Bounce[A]) extends Bounce[A]
case class Cont[A, B](sub: Bounce[A], f: A => Bounce[B]) extends Bounce[B]
```

---

## Lazy Evaluation

### lazy val

```scala
// lazy val: evaluate เมื่อใช้ครั้งแรก
class ExpensiveResource:
  lazy val connection: String =
    println("Connecting...")
    Thread.sleep(100)  // simulate expensive operation
    "connection-established"

  lazy val data: List[Int] =
    println("Loading data...")
    (1 to 1000000).toList

val resource = ExpensiveResource()
println("Resource created")   // No connection yet
println(resource.connection)  // Connecting... connection-established
println(resource.connection)  // connection-established (no reconnect)
println(resource.data.length) // Loading data... 1000000
```

### LazyList (Infinite Lists)

```scala
// LazyList: elements computed on demand

// Natural numbers
val naturals: LazyList[Int] = LazyList.iterate(0)(_ + 1)
println(naturals.take(5).toList)  // List(0, 1, 2, 3, 4)

// Fibonacci
val fibs: LazyList[BigInt] =
  LazyList.unfold((BigInt(0), BigInt(1))) { case (a, b) =>
    Some((a, (b, a + b)))
  }
println(fibs.take(10).toList)
// List(0, 1, 1, 2, 3, 5, 8, 13, 21, 34)

// Primes (Sieve of Eratosthenes)
def sieve(numbers: LazyList[Int]): LazyList[Int] =
  numbers.head #:: sieve(numbers.tail.filter(_ % numbers.head != 0))

val primes = sieve(LazyList.iterate(2)(_ + 1))
println(primes.take(10).toList)
// List(2, 3, 5, 7, 11, 13, 17, 19, 23, 29)

// Processing infinite stream
val evenSquares = naturals
  .filter(_ % 2 == 0)
  .map(n => n * n)
  .take(5)
  .toList
println(evenSquares)  // List(0, 4, 16, 36, 64)
```

### by-name Parameters

```scala
// by-name: parameter ไม่ถูก evaluate จนกว่าจะถูกใช้

def ifTrue[A](condition: Boolean, thenBranch: => A, elseBranch: => A): A =
  if condition then thenBranch else elseBranch

// thenBranch ไม่ถูก evaluate ถ้า condition เป็น false
val result = ifTrue(
  false,
  { println("This won't print"); 1 },  // ไม่ถูกเรียก
  { println("This will print"); 2 }     // ถูกเรียก
)
println(result)  // 2

// retry function
def retry[A](times: Int)(action: => A): Option[A] =
  def loop(remaining: Int): Option[A] =
    if remaining <= 0 then None
    else
      try Some(action)  // action evaluate ทุกครั้ง
      catch case _: Exception =>
        println(s"Retrying... ($remaining left)")
        loop(remaining - 1)
  loop(times)

var attempt = 0
val result2 = retry(3) {
  attempt += 1
  if attempt < 3 then throw new RuntimeException("Not ready")
  else "success"
}
println(result2)  // Some(success)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Lazy Stream Processing

```scala
// สร้าง data pipeline ด้วย LazyList

// 1. Generate infinite stream of random numbers
val rand = new scala.util.Random(42)
val randomStream: LazyList[Double] = LazyList.continually(rand.nextDouble())

// 2. Process: filter, transform, aggregate
def movingAverage(stream: LazyList[Double], window: Int): LazyList[Double] =
  stream.sliding(window)
    .map(_.sum / window)
    .to(LazyList)

// TODO: implement
// val result = randomStream
//   .filter(_ > 0.5)          // keep only > 0.5
//   .map(_ * 100)             // scale to 0-100
//   .take(1000)               // first 1000 elements
//   .toList
// val avg = movingAverage(result.to(LazyList), 10).take(5).toList
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Higher-Order Functions เชิงลึก
- ✅ Collection Operations: fold, scan, groupBy, zip
- ✅ Partial Application
- ✅ Currying กับ Multiple Parameter Lists
- ✅ Trampolining สำหรับ Mutual Recursion
- ✅ Lazy Evaluation: lazy val, LazyList, by-name parameters

---

*[← Part 13: Functional Programming](part-13-functional-programming.md) | [Part 15: For Comprehensions →](part-15-for-comprehensions.md)*
