# Part 17: Collections Advanced

## สารบัญ
1. [Collection Hierarchy เชิงลึก](#collection-hierarchy-เชิงลึก)
2. [Immutable vs Mutable](#immutable-vs-mutable)
3. [Performance Characteristics](#performance-characteristics)
4. [Custom Collections](#custom-collections)
5. [Parallel Collections](#parallel-collections)
6. [Views](#views)

---

## Collection Hierarchy เชิงลึก

```scala
// Scala Collection Hierarchy (simplified):
//
// Iterable[A]
// ├── Seq[A]           - ordered, indexed
// │   ├── IndexedSeq[A] - fast random access
// │   │   ├── Vector[A]
// │   │   ├── Array[A]
// │   │   └── ArraySeq[A]
// │   └── LinearSeq[A]  - fast head/tail
// │       ├── List[A]
// │       └── LazyList[A]
// ├── Set[A]           - unique elements
// │   ├── HashSet[A]
// │   ├── TreeSet[A]   - sorted
// │   └── BitSet
// └── Map[K, V]        - key-value pairs
//     ├── HashMap[K, V]
//     ├── TreeMap[K, V] - sorted by key
//     └── SortedMap[K, V]

// Creating collections
val list    = List(1, 2, 3, 4, 5)
val vector  = Vector(1, 2, 3, 4, 5)
val set     = Set(1, 2, 3, 4, 5)
val map     = Map("a" -> 1, "b" -> 2)

// Generic operations (work on any Seq)
def sumOf[A: Numeric](seq: Seq[A]): A =
  val num = summon[Numeric[A]]
  seq.foldLeft(num.zero)(num.plus)

println(sumOf(list))    // 15
println(sumOf(vector))  // 15
println(sumOf(Array(1.0, 2.0, 3.0)))  // 6.0
```

---

## Immutable vs Mutable

### Immutable Collections

```scala
// List: linked list, fast prepend/head/tail
val lst = 1 :: 2 :: 3 :: Nil

// Prepend O(1)
val lst2 = 0 :: lst     // List(0, 1, 2, 3)
// Append O(n)
val lst3 = lst :+ 4     // List(1, 2, 3, 4)
// Concat O(n)
val lst4 = lst ++ lst3  // List(1, 2, 3, 1, 2, 3, 4)

// Vector: O(log n) access and update
val vec = Vector(1, 2, 3, 4, 5)
val vec2 = vec.updated(2, 99)   // Vector(1, 2, 99, 4, 5) - O(log n)
val vec3 = vec :+ 6             // O(log n)

// Map: O(log n) lookup and update
val m = Map("a" -> 1, "b" -> 2)
val m2 = m + ("c" -> 3)           // add entry
val m3 = m - "a"                  // remove entry
val m4 = m.updated("a", 99)       // update entry
val merged = m ++ Map("c" -> 3, "d" -> 4)  // merge

// Set
val s1 = Set(1, 2, 3, 4, 5)
val s2 = Set(3, 4, 5, 6, 7)
println(s1 | s2)    // union: Set(1,2,3,4,5,6,7)
println(s1 & s2)    // intersection: Set(3,4,5)
println(s1 &~ s2)   // difference: Set(1,2)
println(s1 -- s2)   // same as difference
```

### Mutable Collections

```scala
import scala.collection.mutable

// When to use mutable:
// 1. Performance critical code
// 2. Builder pattern
// 3. Local state in a function

// ArrayBuffer: dynamic array
val buf = mutable.ArrayBuffer[Int](1, 2, 3)
buf += 4          // append
buf.prepend(0)    // prepend
buf.insert(2, 99) // insert at index
buf.remove(2)     // remove at index
println(buf)      // ArrayBuffer(0, 1, 2, 3, 4)

// ListBuffer: for building lists
val lb = mutable.ListBuffer[String]()
lb += "scala"
lb += "is"
lb += "great"
val immutableList = lb.toList  // convert when done
println(immutableList)

// HashMap
val hm = mutable.HashMap[String, Int]()
hm("one") = 1
hm.update("two", 2)
hm.getOrElseUpdate("three", 3)  // add if not exists
println(hm)

// Builder pattern (using mutable internally, returning immutable)
def buildList(n: Int): List[Int] =
  val builder = mutable.ListBuffer[Int]()
  var i = 0
  while i < n do
    builder += i * i
    i += 1
  builder.toList

println(buildList(5))  // List(0, 1, 4, 9, 16)
```

---

## Performance Characteristics

### Big-O Comparison

```scala
// Performance table:
// Operation   | List  | Vector | Array | Set   | Map
// ============|=======|========|=======|=======|=======
// access(i)   | O(n)  | O(logn)| O(1)  | N/A   | N/A
// prepend     | O(1)  | O(logn)| O(n)  | N/A   | N/A
// append      | O(n)  | O(logn)| O(n)  | N/A   | N/A
// insert      | O(n)  | O(logn)| O(n)  | N/A   | N/A
// head        | O(1)  | O(logn)| O(1)  | N/A   | N/A
// tail        | O(1)  | O(logn)| O(n)  | N/A   | N/A
// length      | O(n)  | O(1)   | O(1)  | O(1)  | O(1)
// lookup      | O(n)  | O(n)   | O(n)  | O(1)  | O(1)
// add         | N/A   | N/A    | N/A   | O(1)  | O(1)
// remove      | N/A   | N/A    | N/A   | O(1)  | O(1)

// Benchmarking example
def time[A](name: String)(f: => A): A =
  val start = System.nanoTime()
  val result = f
  val end = System.nanoTime()
  println(f"$name: ${(end - start) / 1e6}%.2f ms")
  result

val n = 100000

// List vs Vector append
time("List append") {
  var list = List.empty[Int]
  for i <- 0 until n do list = list :+ i
  list.length
}

time("Vector append") {
  var vec = Vector.empty[Int]
  for i <- 0 until n do vec = vec :+ i
  vec.length
}

time("ArrayBuffer append") {
  val buf = scala.collection.mutable.ArrayBuffer.empty[Int]
  for i <- 0 until n do buf += i
  buf.length
}
```

### Choosing the Right Collection

```scala
// Decision guide

// Use List when:
// - Frequent head access or pattern matching
// - Building by prepending
// - Recursive algorithms

// Use Vector when:
// - Random access needed
// - Building by appending
// - Large collections with mixed operations

// Use Set when:
// - Membership testing
// - Removing duplicates

// Use Map when:
// - Key-value lookups
// - Grouping data

// Use Array when:
// - Java interop needed
// - Maximum performance with primitives
// - Fixed size

// Example: word frequency
def wordFrequency(text: String): Map[String, Int] =
  text.toLowerCase
    .split("""\W+""")
    .filter(_.nonEmpty)
    .foldLeft(Map.empty[String, Int]) { (map, word) =>
      map.updated(word, map.getOrElse(word, 0) + 1)
    }

val text = "to be or not to be that is the question"
val freq = wordFrequency(text)
println(freq.toList.sortBy(-_._2).take(5))
// List((be,2),(to,2),(or,1),(not,1),(that,1))
```

---

## Custom Collections

### Implementing a Custom Seq

```scala
// Simple Ring Buffer (circular queue)
class RingBuffer[A](capacity: Int) extends Seq[A]:
  private val buffer = new Array[Any](capacity)
  private var start = 0
  private var end = 0
  private var count = 0

  def enqueue(a: A): RingBuffer[A] =
    if count < capacity then
      buffer(end) = a
      end = (end + 1) % capacity
      count += 1
    else
      // overwrite oldest
      buffer(end) = a
      end = (end + 1) % capacity
      start = (start + 1) % capacity
    this

  def dequeue(): Option[A] =
    if count == 0 then None
    else
      val item = buffer(start).asInstanceOf[A]
      start = (start + 1) % capacity
      count -= 1
      Some(item)

  def apply(i: Int): A =
    if i < 0 || i >= count then throw IndexOutOfBoundsException(i.toString)
    buffer((start + i) % capacity).asInstanceOf[A]

  def iterator: Iterator[A] = new Iterator[A]:
    private var idx = 0
    def hasNext: Boolean = idx < count
    def next(): A =
      val item = buffer((start + idx) % capacity).asInstanceOf[A]
      idx += 1
      item

  def length: Int = count

val ring = RingBuffer[Int](3)
ring.enqueue(1).enqueue(2).enqueue(3)
println(ring.toList)   // List(1, 2, 3)
ring.enqueue(4)         // overwrites 1
println(ring.toList)   // List(2, 3, 4)
println(ring.sum)       // 9 (uses Seq's sum)
```

---

## Parallel Collections

```scala
// Parallel collections: easy parallelism
val numbers = (1 to 1000000).toVector

// Sequential
val seqTime = System.nanoTime()
val seqSum = numbers.map(n => n.toLong * n).sum
val seqDuration = System.nanoTime() - seqTime

// Parallel
val parTime = System.nanoTime()
val parSum = numbers.par.map(n => n.toLong * n).sum
val parDuration = System.nanoTime() - parTime

println(s"Sequential: ${seqDuration / 1e6}ms")
println(s"Parallel:   ${parDuration / 1e6}ms")
println(s"Same result: ${seqSum == parSum}")

// Be careful: parallel collections need thread-safe operations
// Avoid mutable state in parallel operations!

// Good: pure transformations
val squares = numbers.par.map(n => n * n)

// Bad: shared mutable state (race condition!)
var badCount = 0
numbers.par.foreach { n =>
  if n % 2 == 0 then badCount += 1  // ❌ race condition
}

// Good: use aggregate instead
val goodCount = numbers.par.aggregate(0)(
  (acc, n) => if n % 2 == 0 then acc + 1 else acc,
  _ + _
)
println(s"Even count: $goodCount")
```

---

## Views

```scala
// Views: lazy transformation (no intermediate collections)

val nums = (1 to 1000000)

// Without view: creates intermediate collections
val result1 = nums
  .filter(_ % 2 == 0)    // creates 500000 element collection
  .map(_ * 3)             // creates 500000 element collection
  .take(5)                // List of 5 elements

// With view: lazy, no intermediate collections
val result2 = nums.view
  .filter(_ % 2 == 0)    // lazy filter
  .map(_ * 3)             // lazy map
  .take(5)                // evaluate only first 5
  .toList

println(result1 == result2)  // true, but result2 is faster

// Views กับ Strings
val str = "Hello, World!"
val upper = str.view.map(_.toUpper).mkString
println(upper)  // HELLO, WORLD!

// Sliding window ด้วย view
val data = Vector(1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0)
val movingAvg = data.sliding(3)
  .map(window => window.sum / window.length)
  .toVector
println(movingAvg)  // Vector(2.0, 3.0, 4.0, 5.0, 6.0)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Collection Hierarchy: Seq, IndexedSeq, LinearSeq, Set, Map
- ✅ Immutable collections: List, Vector, Set, Map
- ✅ Mutable collections: ArrayBuffer, ListBuffer, HashMap
- ✅ Performance characteristics และการเลือก collection
- ✅ Custom Collections
- ✅ Parallel Collections
- ✅ Views (lazy evaluation)

---

*[← Part 16: Option และ Either](part-16-option-either.md) | [Part 18: Implicits และ Given →](part-18-implicits.md)*
