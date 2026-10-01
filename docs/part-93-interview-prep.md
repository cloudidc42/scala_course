# ส่วนที่ 93: Scala Interview Preparation

## สารบัญ

1. [คำถาม Scala Interview ที่พบบ่อย](#คำถาม-scala-interview-ที่พบบ่อย)
2. [Algorithm Problems ใน Scala](#algorithm-problems-ใน-scala)
3. [Data Structures ใน Scala](#data-structures-ใน-scala)
4. [Functional Programming Questions](#functional-programming-questions)
5. [Concurrency Questions](#concurrency-questions)
6. [System Design Questions](#system-design-questions)
7. [Coding Challenges พร้อม Solutions](#coding-challenges-พร้อม-solutions)
8. [สรุป](#สรุป)

---

## คำถาม Scala Interview ที่พบบ่อย

### 1. อธิบายความแตกต่างระหว่าง `val`, `var`, และ `def`

```scala
// val: immutable value, evaluated once at definition
val x = 42          // x = 42 ตลอดไป
val greeting = "Hello"

// var: mutable variable, can be reassigned
var counter = 0
counter += 1        // ทำได้

// def: method/function, evaluated every time it's called
def now = System.currentTimeMillis()  // คำนวณใหม่ทุกครั้ง
def double(n: Int) = n * 2

// Lazy val: initialized on first access, then cached
lazy val expensiveResult = computeSomethingExpensive()
```

### 2. อธิบาย Case Class และความแตกต่างจาก Regular Class

```scala
// Case class: immutable by default, auto-generated equals/hashCode/toString/copy
case class Point(x: Double, y: Double)

val p1 = Point(1.0, 2.0)
val p2 = Point(1.0, 2.0)
println(p1 == p2)        // true (structural equality)
println(p1)              // Point(1.0, 2.0)

val p3 = p1.copy(y = 3.0) // Point(1.0, 3.0)

// Pattern matching works with case class
p1 match
  case Point(0, 0) => "origin"
  case Point(x, 0) => s"on x-axis at $x"
  case Point(x, y) => s"at ($x, $y)"

// Regular class: requires manual equals/hashCode
class RegularPoint(val x: Double, val y: Double)
val r1 = new RegularPoint(1.0, 2.0)
val r2 = new RegularPoint(1.0, 2.0)
println(r1 == r2)  // false (reference equality by default)
```

### 3. Trait vs Abstract Class

```scala
// Trait: can be mixed in multiple, no constructor parameters (in Scala 2; Scala 3 supports them)
trait Flyable:
  def fly(): String

trait Swimmable:
  def swim(): String

// Class can mix multiple traits
class Duck extends Flyable with Swimmable:
  def fly()  = "Duck flying"
  def swim() = "Duck swimming"

// Trait with constructor parameters (Scala 3)
trait Logger(prefix: String):
  def log(msg: String): Unit = println(s"[$prefix] $msg")

class Service extends Logger("SERVICE"):
  def process(): Unit = log("Processing...")

// Abstract class: can have constructor, single inheritance
abstract class Animal(val name: String):
  def sound: String
  def describe: String = s"$name says $sound"

class Dog(name: String) extends Animal(name):
  def sound = "Woof"
```

### 4. Option, Either, Try

```scala
// Option: value may or may not exist
def findUser(id: Int): Option[String] =
  if id > 0 then Some(s"User$id") else None

// Chaining with map/flatMap
val result = findUser(1)
  .map(_.toUpperCase)
  .getOrElse("Anonymous")

// Either: success (Right) or failure (Left)
def divide(a: Int, b: Int): Either[String, Double] =
  if b == 0 then Left("Division by zero")
  else Right(a.toDouble / b)

// Chaining Either
val calc =
  for
    r1 <- divide(10, 2)   // Right(5.0)
    r2 <- divide(r1.toInt, 2)  // Right(2.5)
  yield r1 + r2

// Try: success (Success) or exception (Failure)
import scala.util.{Try, Success, Failure}

def parseInt(s: String): Try[Int] = Try(s.toInt)

parseInt("42") match
  case Success(n) => println(s"Parsed: $n")
  case Failure(e) => println(s"Error: ${e.getMessage}")
```

### 5. Implicits / Given

```scala
// Scala 3: given/using instead of implicit
given Ordering[Int] = Ordering.Int  // already exists

case class Person(name: String, age: Int)

given personOrdering: Ordering[Person] =
  Ordering.by(_.age)

def sortPeople(people: List[Person])(using ord: Ordering[Person]): List[Person] =
  people.sorted

val people = List(Person("Alice", 30), Person("Bob", 25), Person("Charlie", 35))
val sorted = sortPeople(people)  // Uses given personOrdering

// Type class pattern
trait Show[A]:
  def show(a: A): String

given Show[Int]    = a => a.toString
given Show[String] = a => s"\"$a\""
given [A: Show, B: Show]: Show[(A, B)] = (a, b) =>
  s"(${summon[Show[A]].show(a)}, ${summon[Show[B]].show(b)})"

def print[A: Show](a: A): Unit =
  println(summon[Show[A]].show(a))
```

### 6. Higher-Order Functions

```scala
// Functions as first-class citizens
def applyTwice[A](f: A => A, x: A): A = f(f(x))

val double  = (x: Int) => x * 2
val addTen  = (x: Int) => x + 10

println(applyTwice(double, 3))   // 12
println(applyTwice(addTen, 5))   // 25

// Currying
def add(a: Int)(b: Int): Int = a + b
val add5 = add(5) _    // Partially applied
println(add5(3))        // 8

// Composing functions
val process = double andThen addTen  // first double, then addTen
println(process(3))  // (3*2)+10 = 16
```

---

## Algorithm Problems ใน Scala

### Binary Search

```scala
def binarySearch[A: Ordering](arr: Array[A], target: A): Int =
  @annotation.tailrec
  def search(low: Int, high: Int): Int =
    if low > high then -1
    else
      val mid = low + (high - low) / 2
      val cmp = Ordering[A].compare(arr(mid), target)
      if cmp == 0 then mid
      else if cmp < 0 then search(mid + 1, high)
      else search(low, mid - 1)
  search(0, arr.length - 1)

// Test
val arr = Array(1, 3, 5, 7, 9, 11, 13)
println(binarySearch(arr, 7))   // 3
println(binarySearch(arr, 4))   // -1
```

### Merge Sort

```scala
def mergeSort[A: Ordering](list: List[A]): List[A] =
  def merge(left: List[A], right: List[A]): List[A] =
    (left, right) match
      case (Nil, r)                                                  => r
      case (l, Nil)                                                  => l
      case (lh :: lt, rh :: _) if Ordering[A].lteq(lh, rh) => lh :: merge(lt, right)
      case (_, rh :: rt)                                             => rh :: merge(left, rt)

  list match
    case Nil | List(_) => list
    case _ =>
      val mid   = list.length / 2
      val left  = mergeSort(list.take(mid))
      val right = mergeSort(list.drop(mid))
      merge(left, right)

println(mergeSort(List(5, 2, 8, 1, 9, 3)))  // List(1, 2, 3, 5, 8, 9)
```

### Tree Operations

```scala
sealed trait Tree[+A]
case object Empty extends Tree[Nothing]
case class Node[A](value: A, left: Tree[A], right: Tree[A]) extends Tree[A]

object Tree:
  def insert[A: Ordering](tree: Tree[A], value: A): Tree[A] =
    tree match
      case Empty => Node(value, Empty, Empty)
      case Node(v, left, right) =>
        val cmp = Ordering[A].compare(value, v)
        if cmp < 0 then Node(v, insert(left, value), right)
        else if cmp > 0 then Node(v, left, insert(right, value))
        else tree  // duplicate

  def contains[A: Ordering](tree: Tree[A], target: A): Boolean =
    tree match
      case Empty => false
      case Node(v, left, right) =>
        val cmp = Ordering[A].compare(target, v)
        if cmp == 0 then true
        else if cmp < 0 then contains(left, target)
        else contains(right, target)

  def inOrder[A](tree: Tree[A]): List[A] =
    tree match
      case Empty           => Nil
      case Node(v, l, r)   => inOrder(l) ::: List(v) ::: inOrder(r)

  def height[A](tree: Tree[A]): Int =
    tree match
      case Empty         => 0
      case Node(_, l, r) => 1 + math.max(height(l), height(r))

  def fromList[A: Ordering](list: List[A]): Tree[A] =
    list.foldLeft[Tree[A]](Empty)(insert)

// Test
val bst = Tree.fromList(List(5, 3, 7, 1, 4, 6, 8))
println(Tree.inOrder(bst))         // List(1, 3, 4, 5, 6, 7, 8)
println(Tree.contains(bst, 4))     // true
println(Tree.height(bst))          // 3
```

### Dynamic Programming

```scala
// Fibonacci with memoization
def fibonacci(n: Int): Long =
  val memo = scala.collection.mutable.Map[Int, Long]()
  def fib(n: Int): Long =
    if n <= 1 then n.toLong
    else memo.getOrElseUpdate(n, fib(n - 1) + fib(n - 2))
  fib(n)

// Longest Common Subsequence
def lcs(s1: String, s2: String): Int =
  val m   = s1.length
  val n   = s2.length
  val dp  = Array.ofDim[Int](m + 1, n + 1)
  for
    i <- 1 to m
    j <- 1 to n
  do
    dp(i)(j) =
      if s1(i - 1) == s2(j - 1) then dp(i - 1)(j - 1) + 1
      else math.max(dp(i - 1)(j), dp(i)(j - 1))
  dp(m)(n)

// Knapsack Problem
def knapsack(weights: Array[Int], values: Array[Int], capacity: Int): Int =
  val n  = weights.length
  val dp = Array.ofDim[Int](n + 1, capacity + 1)
  for
    i <- 1 to n
    w <- 0 to capacity
  do
    dp(i)(w) =
      if weights(i - 1) > w then dp(i - 1)(w)
      else math.max(dp(i - 1)(w), dp(i - 1)(w - weights(i - 1)) + values(i - 1))
  dp(n)(capacity)

println(lcs("ABCBDAB", "BDCAB"))  // 4
println(knapsack(Array(2, 3, 4, 5), Array(3, 4, 5, 6), 8))  // 10
```

---

## Data Structures ใน Scala

### Immutable Stack

```scala
sealed trait Stack[+A]:
  def push[B >: A](elem: B): Stack[B] = StackNode(elem, this)
  def pop: Option[(A, Stack[A])]
  def peek: Option[A]
  def isEmpty: Boolean

case object EmptyStack extends Stack[Nothing]:
  def pop          = None
  def peek         = None
  val isEmpty      = true

case class StackNode[+A](head: A, tail: Stack[A]) extends Stack[A]:
  def pop          = Some((head, tail))
  def peek         = Some(head)
  val isEmpty      = false

// Usage
val s0 = EmptyStack
val s1 = s0.push(1).push(2).push(3)
s1.pop match
  case Some((top, rest)) => println(s"Top: $top")  // Top: 3
  case None              => println("Empty")
```

### Functional Queue

```scala
// O(1) amortized enqueue and dequeue
case class Queue[+A](inbox: List[A], outbox: List[A]):
  def enqueue[B >: A](elem: B): Queue[B] =
    Queue(elem :: inbox, outbox)

  def dequeue: Option[(A, Queue[A])] =
    outbox match
      case head :: tail => Some((head, Queue(inbox, tail)))
      case Nil =>
        inbox.reverse match
          case Nil          => None
          case head :: tail => Some((head, Queue(Nil, tail)))

  def peek: Option[A] = dequeue.map(_._1)
  def isEmpty: Boolean = inbox.isEmpty && outbox.isEmpty

object Queue:
  def empty[A]: Queue[A] = Queue(Nil, Nil)
  def apply[A](elems: A*): Queue[A] =
    elems.foldLeft(empty[A])((q, e) => q.enqueue(e))

// Test
val q = Queue(1, 2, 3)
q.dequeue match
  case Some((head, rest)) => println(s"Dequeued: $head")  // 1
  case None               => println("Empty")
```

### Trie (Prefix Tree)

```scala
case class TrieNode(
  children: Map[Char, TrieNode] = Map.empty,
  isEnd: Boolean = false
)

class Trie:
  private var root = TrieNode()

  def insert(word: String): Unit =
    def go(node: TrieNode, chars: List[Char]): TrieNode =
      chars match
        case Nil => node.copy(isEnd = true)
        case c :: rest =>
          val child    = node.children.getOrElse(c, TrieNode())
          val newChild = go(child, rest)
          node.copy(children = node.children + (c -> newChild))
    root = go(root, word.toList)

  def search(word: String): Boolean =
    def go(node: TrieNode, chars: List[Char]): Boolean =
      chars match
        case Nil    => node.isEnd
        case c :: rest =>
          node.children.get(c).exists(go(_, rest))
    go(root, word.toList)

  def startsWith(prefix: String): Boolean =
    def go(node: TrieNode, chars: List[Char]): Boolean =
      chars match
        case Nil    => true
        case c :: rest => node.children.get(c).exists(go(_, rest))
    go(root, prefix.toList)

  def wordsWithPrefix(prefix: String): List[String] =
    def findNode(node: TrieNode, chars: List[Char]): Option[TrieNode] =
      chars match
        case Nil    => Some(node)
        case c :: rest => node.children.get(c).flatMap(findNode(_, rest))
    def collect(node: TrieNode, current: String): List[String] =
      val own = if node.isEnd then List(current) else Nil
      own ++ node.children.flatMap((c, child) => collect(child, current + c)).toList
    findNode(root, prefix.toList).map(collect(_, prefix)).getOrElse(Nil)

// Test
val trie = Trie()
List("apple", "app", "apply", "application", "banana").foreach(trie.insert)
println(trie.search("app"))         // true
println(trie.startsWith("appl"))    // true
println(trie.wordsWithPrefix("app")) // List(app, apple, apply, application)
```

---

## Functional Programming Questions

### 1. Monad Laws

```scala
// Monad must satisfy three laws:
// 1. Left identity:   pure(a).flatMap(f) == f(a)
// 2. Right identity:  m.flatMap(pure)    == m
// 3. Associativity:   m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))

// Example with Option
val f: Int => Option[Int] = x => if x > 0 then Some(x * 2) else None
val g: Int => Option[Int] = x => if x < 100 then Some(x + 1) else None

val m = Option(5)

// Left identity
assert(Option(5).flatMap(f) == f(5))

// Right identity
assert(m.flatMap(Option(_)) == m)

// Associativity
assert(m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g)))
```

### 2. Functor, Applicative, Monad

```scala
import cats.{Functor, Applicative, Monad}
import cats.syntax.all.*

// Functor: map (shape-preserving transformation)
val opt: Option[Int] = Some(5)
opt.map(_ * 2)  // Some(10)

// Applicative: ap (apply function in context to value in context)
val optFn: Option[Int => Int] = Some(_ + 3)
Applicative[Option].ap(optFn)(opt)  // Some(8)

// Or with mapN
(Option(1), Option(2), Option(3)).mapN(_ + _ + _)  // Some(6)

// Monad: flatMap (chain computations in context)
def lookup(id: Int): Option[String] = Map(1 -> "Alice", 2 -> "Bob").get(id)
def getEmail(name: String): Option[String] = Map("Alice" -> "alice@ex.com").get(name)

val email = Option(1).flatMap(lookup).flatMap(getEmail)  // Some("alice@ex.com")

// Using for-comprehension (syntactic sugar for flatMap)
val email2 =
  for
    id    <- Option(1)
    name  <- lookup(id)
    email <- getEmail(name)
  yield email
```

### 3. Algebraic Data Types

```scala
// Sum type (OR)
sealed trait Shape
case class Circle(radius: Double) extends Shape
case class Rectangle(width: Double, height: Double) extends Shape
case class Triangle(base: Double, height: Double) extends Shape

// Product type (AND)
case class Point(x: Double, y: Double)

// Recursive ADT
sealed trait Expr
case class Num(n: Double) extends Expr
case class Add(l: Expr, r: Expr) extends Expr
case class Mul(l: Expr, r: Expr) extends Expr
case class Neg(e: Expr) extends Expr

def eval(expr: Expr): Double = expr match
  case Num(n)    => n
  case Add(l, r) => eval(l) + eval(r)
  case Mul(l, r) => eval(l) * eval(r)
  case Neg(e)    => -eval(e)

// (3 + 4) * (-2) = -14
val e = Mul(Add(Num(3), Num(4)), Neg(Num(2)))
println(eval(e))  // -14.0
```

### 4. Tail Recursion

```scala
// Non-tail recursive - stack overflow for large n
def factBad(n: Int): Long =
  if n <= 1 then 1L
  else n * factBad(n - 1)  // NOT in tail position

// Tail recursive with accumulator
@annotation.tailrec
def fact(n: Int, acc: Long = 1L): Long =
  if n <= 1 then acc
  else fact(n - 1, n * acc)

// Trampoline for mutual recursion
import cats.free.Trampoline
import cats.free.Trampoline.*

def isEven(n: Int): Trampoline[Boolean] =
  if n == 0 then done(true)
  else defer(isOdd(n - 1))

def isOdd(n: Int): Trampoline[Boolean] =
  if n == 0 then done(false)
  else defer(isEven(n - 1))

println(isEven(10000).run)  // true, no stack overflow
```

---

## Concurrency Questions

### Cats Effect Fibers

```scala
import cats.effect.*
import cats.effect.std.CountDownLatch
import scala.concurrent.duration.*

// Fiber: lightweight green thread
val program: IO[Unit] =
  for
    fiber1 <- IO.sleep(1.second).as("fiber1 done").start
    fiber2 <- IO.sleep(500.millis).as("fiber2 done").start
    r2     <- fiber2.join
    r1     <- fiber1.join
    _      <- IO.println(s"$r2 first, $r1 second")
  yield ()

// Racing two computations
val race: IO[Unit] =
  IO.race(
    IO.sleep(1.second).as("slow"),
    IO.sleep(100.millis).as("fast")
  ).flatMap(winner => IO.println(s"Winner: $winner"))

// Parallel execution
import cats.syntax.all.*
val parallel: IO[(Int, String, Boolean)] =
  (
    IO.sleep(100.millis) *> IO.pure(42),
    IO.sleep(200.millis) *> IO.pure("hello"),
    IO.sleep(50.millis)  *> IO.pure(true)
  ).parTupled  // runs all three concurrently
```

### Ref and Deferred

```scala
import cats.effect.Ref

// Ref: mutable reference safe for concurrent use
val counter: IO[Int] =
  for
    ref     <- Ref.of[IO, Int](0)
    fibers  <- List.fill(100)(ref.update(_ + 1).start).sequence
    _       <- fibers.traverse(_.join)
    result  <- ref.get
  yield result

// Deferred: one-shot promise
val communication: IO[Unit] =
  for
    deferred <- Deferred[IO, String]
    _        <- (IO.sleep(500.millis) *> deferred.complete("hello")).start
    message  <- deferred.get  // waits until complete is called
    _        <- IO.println(s"Received: $message")
  yield ()
```

### Question: Actor vs Fiber

```
Actor Model (Akka):
  - Message-passing concurrency
  - Mutable state encapsulated in actor
  - Location transparent (remote actors)
  - Supervision hierarchy
  - Good for: stateful, distributed systems

Fiber (Cats Effect / ZIO):
  - Direct-style concurrency
  - Structured concurrency (scoped lifetimes)
  - Purely functional
  - Cancellation and error propagation
  - Good for: functional, I/O-bound applications
```

---

## System Design Questions

### Design a URL Shortener

```scala
import cats.effect.*
import cats.effect.std.Random
import org.http4s.*

// Domain Model
opaque type ShortCode = String
object ShortCode:
  def apply(code: String): ShortCode = code
  extension (c: ShortCode) def value: String = c

case class ShortLink(
  code: ShortCode,
  originalUrl: String,
  createdAt: java.time.Instant,
  expiresAt: Option[java.time.Instant],
  clickCount: Int
)

// Service
trait UrlShortener[F[_]]:
  def shorten(url: String, customCode: Option[String] = None): F[Either[String, ShortLink]]
  def resolve(code: ShortCode): F[Option[ShortLink]]
  def trackClick(code: ShortCode): F[Unit]
  def getStats(code: ShortCode): F[Option[ShortLink]]

class UrlShortenerImpl[F[_]: Monad](
  repo: LinkRepository[F],
  rng: Random[F]
) extends UrlShortener[F]:
  private val Base62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"

  private def generateCode: F[ShortCode] =
    List.fill(7)(rng.nextIntBounded(62)).sequence
      .map(_.map(Base62.charAt).mkString)
      .map(ShortCode(_))

  def shorten(url: String, customCode: Option[String] = None): F[Either[String, ShortLink]] =
    customCode match
      case Some(code) =>
        repo.findByCode(ShortCode(code)).flatMap:
          case Some(_) => Monad[F].pure(Left("Code already taken"))
          case None    =>
            val link = ShortLink(ShortCode(code), url, java.time.Instant.now(), None, 0)
            repo.save(link).map(Right(_))
      case None =>
        generateCode.flatMap: code =>
          val link = ShortLink(code, url, java.time.Instant.now(), None, 0)
          repo.save(link).map(Right(_))

  def resolve(code: ShortCode): F[Option[ShortLink]] = repo.findByCode(code)

  def trackClick(code: ShortCode): F[Unit] = repo.incrementClicks(code)

  def getStats(code: ShortCode): F[Option[ShortLink]] = repo.findByCode(code)

// Capacity Estimation (100M links, 1B redirects/day)
// Read-heavy: read:write = 100:1
// Solution: CDN + Cache (Redis) for hot links
// Database: PostgreSQL with index on short_code
// Rate limiting: per IP, per user
```

### Design a Rate Limiter

```scala
import cats.effect.*
import cats.effect.std.MapRef
import java.time.Instant

// Token Bucket Algorithm
case class TokenBucket(tokens: Double, lastRefill: Long)

class RateLimiter(
  maxTokens: Double,
  refillRate: Double, // tokens per second
  store: Ref[IO, Map[String, TokenBucket]]
):
  def isAllowed(clientId: String): IO[Boolean] =
    store.modify: buckets =>
      val now    = Instant.now().toEpochMilli
      val bucket = buckets.getOrElse(clientId, TokenBucket(maxTokens, now))
      val elapsed = (now - bucket.lastRefill) / 1000.0
      val refilled = math.min(maxTokens, bucket.tokens + elapsed * refillRate)
      if refilled >= 1.0 then
        val updated = bucket.copy(tokens = refilled - 1.0, lastRefill = now)
        (buckets + (clientId -> updated), true)
      else
        (buckets, false)

object RateLimiter:
  def create(maxTokens: Double, refillRate: Double): IO[RateLimiter] =
    Ref.of[IO, Map[String, TokenBucket]](Map.empty)
      .map(new RateLimiter(maxTokens, refillRate, _))
```

---

## Coding Challenges พร้อม Solutions

### Challenge 1: Valid Parentheses

```scala
def isValid(s: String): Boolean =
  val matching = Map(')' -> '(', ']' -> '[', '}' -> '{')
  s.foldLeft(Option(List.empty[Char])):
    case (None, _) => None
    case (Some(stack), c) if "([{".contains(c) => Some(c :: stack)
    case (Some(Nil), c) if matching.contains(c) => None
    case (Some(top :: rest), c) if matching.contains(c) =>
      if matching(c) == top then Some(rest) else None
    case (stack, _) => stack
  .exists(_.isEmpty)

println(isValid("()[]{}"))  // true
println(isValid("([)]"))    // false
println(isValid("{[]}"))    // true
```

### Challenge 2: Two Sum

```scala
def twoSum(nums: Array[Int], target: Int): Array[Int] =
  @annotation.tailrec
  def go(i: Int, seen: Map[Int, Int]): Array[Int] =
    if i >= nums.length then Array.empty
    else
      val complement = target - nums(i)
      seen.get(complement) match
        case Some(j) => Array(j, i)
        case None    => go(i + 1, seen + (nums(i) -> i))
  go(0, Map.empty)

println(twoSum(Array(2, 7, 11, 15), 9).mkString("[", ", ", "]"))  // [0, 1]
```

### Challenge 3: Maximum Subarray (Kadane's Algorithm)

```scala
def maxSubArray(nums: Array[Int]): Int =
  nums.foldLeft((nums.head, nums.head)):
    case ((maxSoFar, currentMax), n) =>
      val newCurrent = math.max(n, currentMax + n)
      (math.max(maxSoFar, newCurrent), newCurrent)
  ._1

println(maxSubArray(Array(-2, 1, -3, 4, -1, 2, 1, -5, 4)))  // 6
```

### Challenge 4: Group Anagrams

```scala
def groupAnagrams(strs: Array[String]): List[List[String]] =
  strs.groupBy(_.sorted).values.map(_.toList).toList

println(groupAnagrams(Array("eat","tea","tan","ate","nat","bat")))
// List(List(eat, tea, ate), List(tan, nat), List(bat))
```

### Challenge 5: Balanced Binary Tree

```scala
sealed trait BTree[+A]
case object Leaf extends BTree[Nothing]
case class Branch[A](left: BTree[A], value: A, right: BTree[A]) extends BTree[A]

def isBalanced[A](tree: BTree[A]): Boolean =
  def heightOrNeg1(t: BTree[A]): Int =
    t match
      case Leaf => 0
      case Branch(l, _, r) =>
        val lh = heightOrNeg1(l)
        val rh = heightOrNeg1(r)
        if lh == -1 || rh == -1 || math.abs(lh - rh) > 1 then -1
        else 1 + math.max(lh, rh)
  heightOrNeg1(tree) != -1

val balanced = Branch(Branch(Leaf, 1, Leaf), 2, Branch(Leaf, 3, Leaf))
val unbalanced = Branch(Branch(Branch(Leaf, 1, Leaf), 2, Leaf), 3, Leaf)
println(isBalanced(balanced))    // true
println(isBalanced(unbalanced))  // false
```

### Challenge 6: Implement LRU Cache

```scala
import scala.collection.mutable

class LRUCache(capacity: Int):
  private val cache = mutable.LinkedHashMap.empty[Int, Int]

  def get(key: Int): Int =
    cache.get(key) match
      case None    => -1
      case Some(v) =>
        // Move to end (most recently used)
        cache.remove(key)
        cache.put(key, v)
        v

  def put(key: Int, value: Int): Unit =
    if cache.contains(key) then
      cache.remove(key)
    else if cache.size >= capacity then
      // Remove least recently used (first element)
      cache.remove(cache.head._1)
    cache.put(key, value)

// Test
val lru = LRUCache(3)
lru.put(1, 1)
lru.put(2, 2)
lru.put(3, 3)
println(lru.get(1))  // 1
lru.put(4, 4)        // evicts key 2
println(lru.get(2))  // -1 (evicted)
println(lru.get(3))  // 3
println(lru.get(4))  // 4
```

### Challenge 7: Stream Processing

```scala
import fs2.Stream
import cats.effect.IO

// Process a stream of events and compute rolling statistics
case class Event(userId: String, value: Double, timestamp: Long)
case class Stats(count: Int, sum: Double, min: Double, max: Double):
  def mean: Double = if count == 0 then 0.0 else sum / count
  def update(v: Double): Stats = Stats(count + 1, sum + v, math.min(min, v), math.max(max, v))

def processStream(events: Stream[IO, Event]): IO[Map[String, Stats]] =
  events
    .fold(Map.empty[String, Stats]):
      case (acc, event) =>
        val stats = acc.getOrElse(event.userId, Stats(0, 0, Double.MaxValue, Double.MinValue))
        acc + (event.userId -> stats.update(event.value))
    .compile
    .lastOrError

// Usage
val eventStream = Stream.emits[IO, Event](List(
  Event("alice", 10.0, 1000),
  Event("bob",   20.0, 1001),
  Event("alice", 15.0, 1002),
  Event("bob",   25.0, 1003),
  Event("alice", 5.0,  1004)
))

// processStream(eventStream).unsafeRunSync()
// => Map(alice -> Stats(3, 30.0, 5.0, 15.0), bob -> Stats(2, 45.0, 20.0, 25.0))
```

---

## สรุป

เตรียมตัวสัมภาษณ์ Scala ให้ครอบคลุม:

- **ภาษา Scala**: `val/var/def`, case class, trait, implicits/given, pattern matching
- **FP Concepts**: Option/Either/Try, higher-order functions, monads, type classes
- **Algorithms**: Binary search, sorting, tree traversal, dynamic programming
- **Data Structures**: Stack, Queue, Trie ด้วย immutable functional style
- **Concurrency**: Cats Effect fibers, Ref, Deferred, parallel execution
- **System Design**: URL shortener, rate limiter พร้อม capacity estimation
- **Coding Challenges**: LeetCode-style problems ด้วย idiomatic Scala

---

*[← ส่วนที่ 92: Advanced Database Operations](part-92-datastore.md) | [ส่วนที่ 94: Scala Best Practices →](part-94-best-practices.md)*
