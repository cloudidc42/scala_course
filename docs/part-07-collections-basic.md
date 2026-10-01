# Part 07: Collections พื้นฐาน

## สารบัญ
1. [Collections Overview](#collections-overview)
2. [List](#list)
3. [Vector](#vector)
4. [Array](#array)
5. [Set](#set)
6. [Map](#map)
7. [Tuple](#tuple)
8. [Range](#range)
9. [Mutable vs Immutable](#mutable-vs-immutable)
10. [Collection Operations ครบถ้วน](#collection-operations)
11. [Performance Comparison](#performance-comparison)
12. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Collections Overview

### Scala Collection Hierarchy

```
Iterable
├── Seq (ordered, may have duplicates)
│   ├── IndexedSeq
│   │   ├── Vector (immutable, fast random access)
│   │   ├── Array (mutable, Java array)
│   │   └── ArraySeq (immutable wrapper)
│   └── LinearSeq
│       ├── List (immutable linked list)
│       ├── LazyList (lazy evaluation)
│       └── Queue
├── Set (no duplicates)
│   ├── HashSet (immutable)
│   ├── TreeSet (sorted)
│   └── LinkedHashSet (insertion order)
└── Map (key-value pairs)
    ├── HashMap (immutable)
    ├── TreeMap (sorted by key)
    └── LinkedHashMap (insertion order)
```

### Immutable vs Mutable

```scala
// Immutable (default - ใช้โดยไม่ต้อง import)
import scala.collection.immutable.*  // ไม่จำเป็น แต่ explicit

val list = List(1, 2, 3)        // immutable
val set = Set(1, 2, 3)          // immutable
val map = Map("a" -> 1)         // immutable

// Mutable (ต้อง import)
import scala.collection.mutable

val mList = mutable.ListBuffer(1, 2, 3)
val mSet = mutable.HashSet(1, 2, 3)
val mMap = mutable.HashMap("a" -> 1)
```

---

## List

### การสร้าง List

```scala
// วิธีต่างๆ ในการสร้าง List
val empty = List.empty[Int]  // ว่างเปล่า
val empty2 = Nil             // เหมือนกัน

val nums = List(1, 2, 3, 4, 5)
val strs = List("a", "b", "c")

// Cons operator ::
val cons = 1 :: 2 :: 3 :: Nil  // เหมือน List(1,2,3)
val prepend = 0 :: nums        // List(0, 1, 2, 3, 4, 5)

// List.fill
val zeros = List.fill(5)(0)           // List(0, 0, 0, 0, 0)
val chars = List.fill(3)('x')         // List(x, x, x)

// List.tabulate
val squares = List.tabulate(5)(i => i * i)  // List(0, 1, 4, 9, 16)
val table = List.tabulate(3, 3)((i, j) => i * 3 + j)
// List(List(0,1,2), List(3,4,5), List(6,7,8))

// List.range
val range = List.range(1, 10)        // List(1,2,3,4,5,6,7,8,9)
val rangeStep = List.range(0, 20, 3) // List(0,3,6,9,12,15,18)

// Converting from other types
val fromArray = Array(1, 2, 3).toList
val fromSet = Set(3, 1, 2).toList    // order not guaranteed
val fromRange = (1 to 5).toList
```

### List Operations

```scala
val list = List(1, 2, 3, 4, 5)

// Access
println(list.head)     // 1 (first element)
println(list.tail)     // List(2,3,4,5) (all but first)
println(list.last)     // 5 (last element)
println(list.init)     // List(1,2,3,4) (all but last)
println(list(2))       // 3 (index access, O(n)!)
println(list.apply(2)) // 3

// Size
println(list.length)   // 5
println(list.size)     // 5
println(list.isEmpty)  // false
println(list.nonEmpty) // true

// Search
println(list.contains(3))        // true
println(list.indexOf(3))         // 2
println(list.lastIndexOf(3))     // 2
println(list.find(_ > 3))        // Some(4)
println(list.exists(_ > 3))      // true
println(list.forall(_ > 0))      // true
println(list.count(_ % 2 == 0))  // 2

// Slicing
println(list.take(3))            // List(1,2,3)
println(list.drop(3))            // List(4,5)
println(list.slice(1, 4))        // List(2,3,4)
println(list.takeWhile(_ < 4))   // List(1,2,3)
println(list.dropWhile(_ < 4))   // List(4,5)
println(list.splitAt(3))         // (List(1,2,3), List(4,5))
println(list.span(_ < 4))        // (List(1,2,3), List(4,5))
```

### List Transformations

```scala
val nums = List(1, 2, 3, 4, 5)

// map
val doubled = nums.map(_ * 2)   // List(2,4,6,8,10)

// flatMap
val expanded = nums.flatMap(n => List(n, n * 10))
// List(1,10,2,20,3,30,4,40,5,50)

// filter
val evens = nums.filter(_ % 2 == 0)  // List(2,4)

// filterNot
val odds = nums.filterNot(_ % 2 == 0)  // List(1,3,5)

// collect (filter + map กับ partial function)
val strings = List("1", "two", "3", "four", "5")
val parsed = strings.collect { case s if s.forall(_.isDigit) => s.toInt }
// List(1, 3, 5)

// partition
val (pos, nonPos) = nums.partition(_ > 0)

// groupBy
val grouped = nums.groupBy(_ % 2)
// Map(0 -> List(2,4), 1 -> List(1,3,5))

// zip และ unzip
val names = List("Alice", "Bob", "Charlie")
val ages = List(30, 25, 35)
val zipped = names.zip(ages)  // List((Alice,30),(Bob,25),(Charlie,35))
val (names2, ages2) = zipped.unzip

// zipWithIndex
val indexed = names.zipWithIndex
// List((Alice,0),(Bob,1),(Charlie,2))

// sorted
val unsorted = List(3, 1, 4, 1, 5, 9, 2, 6)
println(unsorted.sorted)              // List(1,1,2,3,4,5,6,9)
println(unsorted.sortBy(-_))          // descending: List(9,6,5,4,3,2,1,1)
println(unsorted.sorted(Ordering[Int].reverse))  // List(9,6,5,4,3,2,1,1)

// distinct
val withDups = List(1, 2, 2, 3, 3, 3, 4)
println(withDups.distinct)            // List(1,2,3,4)
println(withDups.distinctBy(_ % 2))   // List(1,2)

// reverse
println(List(1,2,3).reverse)  // List(3,2,1)

// flatten
val nested = List(List(1,2), List(3,4), List(5,6))
println(nested.flatten)  // List(1,2,3,4,5,6)
```

### List Aggregations

```scala
val nums = List(1, 2, 3, 4, 5)

// Basic aggregations
println(nums.sum)         // 15
println(nums.product)     // 120
println(nums.min)         // 1
println(nums.max)         // 5
println(nums.minBy(-_))   // 5 (element with min -n)
println(nums.maxBy(-_))   // 1 (element with max -n)

// fold
val sumFold = nums.foldLeft(0)(_ + _)   // 15
val product = nums.foldLeft(1)(_ * _)   // 120

// scanLeft (running totals)
val running = nums.scanLeft(0)(_ + _)
// List(0, 1, 3, 6, 10, 15)

// reduce
val product2 = nums.reduce(_ * _)  // 120

// mkString
println(nums.mkString)        // 12345
println(nums.mkString(", "))  // 1, 2, 3, 4, 5
println(nums.mkString("[", ", ", "]"))  // [1, 2, 3, 4, 5]
```

### List Combination

```scala
val a = List(1, 2, 3)
val b = List(4, 5, 6)

// Concatenation
val c1 = a ++ b      // List(1,2,3,4,5,6)
val c2 = a ::: b     // List(1,2,3,4,5,6) (same for List)

// Prepend/Append
val prepended = 0 :: a       // List(0,1,2,3)
val appended = a :+ 4        // List(1,2,3,4)
val prepended2 = a.prepended(0)  // List(0,1,2,3)
val appended2 = a.appended(4)    // List(1,2,3,4)

// intersect, diff, union
val x = List(1, 2, 3, 4)
val y = List(3, 4, 5, 6)
println(x.intersect(y))  // List(3,4) (common elements)
println(x.diff(y))       // List(1,2) (in x but not y)
println((x ++ y).distinct)  // List(1,2,3,4,5,6) (union)
```

---

## Vector

```scala
// Vector: indexed sequence ที่ performant สำหรับ random access

val v = Vector(1, 2, 3, 4, 5)

// Random access O(log32 n) ≈ O(1)
println(v(0))   // 1
println(v(4))   // 5

// Append/Prepend ที่ไม่ต้อง copy ทั้ง collection
val v2 = v.appended(6)        // Vector(1,2,3,4,5,6)
val v3 = v.prepended(0)       // Vector(0,1,2,3,4,5)
val v4 = 0 +: v :+ 6          // Vector(0,1,2,3,4,5,6)

// Vector vs List:
// - List: ดีสำหรับ prepend (O(1)), head/tail operations
// - Vector: ดีสำหรับ random access, append, general use

// Vector สำหรับ large sequences
val big = Vector.fill(1000000)(0)
val bigUpdated = big.updated(500000, 42)  // O(log n) ไม่ใช่ O(n)!
```

---

## Array

```scala
// Array: Java array ที่ mutable
val arr = Array(1, 2, 3, 4, 5)
val arr2 = new Array[Int](5)  // Array(0,0,0,0,0)

// Mutable!
arr(0) = 10
println(arr.mkString(", "))  // 10, 2, 3, 4, 5

// Array operations (เหมือน Seq ส่วนใหญ่)
println(arr.length)  // 5
println(arr.sum)     // 24
println(arr.sorted.mkString(", "))  // 2, 3, 4, 5, 10

// Multi-dimensional array
val matrix = Array.ofDim[Int](3, 3)
for i <- 0 until 3; j <- 0 until 3 do
  matrix(i)(j) = i * 3 + j

for row <- matrix do
  println(row.mkString(" "))
// 0 1 2
// 3 4 5
// 6 7 8

// Array vs List:
// - Array: mutable, Java interop, primitive types (no boxing)
// - List: immutable, functional programming
```

---

## Set

```scala
// Set: collection ที่ไม่มี duplicate

// สร้าง
val empty = Set.empty[Int]
val s1 = Set(1, 2, 3, 4, 5)
val s2 = Set(3, 4, 5, 6, 7)

// เพิ่ม/ลบ element
val s3 = s1 + 6      // Set(1,2,3,4,5,6)
val s4 = s1 - 3      // Set(1,2,4,5)
val s5 = s1 ++ s2    // union: Set(1,2,3,4,5,6,7)
val s6 = s1 + 3      // Set(1,2,3,4,5) (ไม่เพิ่มเพราะมีแล้ว)

// Set operations
println(s1.contains(3))      // true
println(s1(3))               // true (เรียก contains)
println(s1.size)             // 5
println(s1.isEmpty)          // false

// Set algebra
val union = s1 | s2              // Set(1,2,3,4,5,6,7)
val intersection = s1 & s2      // Set(3,4,5)
val difference = s1 &~ s2       // Set(1,2) (in s1 but not s2)
val diff2 = s1 -- s2            // Set(1,2)
val symDiff = (s1 | s2) -- (s1 & s2)  // Set(1,2,6,7)

// Subsets
val subset = Set(1, 2)
println(subset.subsetOf(s1))   // true
println(s1.subsetOf(subset))   // false

// Sorted Set
import scala.collection.immutable.SortedSet
val sorted = SortedSet(5, 3, 1, 4, 2)
println(sorted)  // TreeSet(1, 2, 3, 4, 5) - sorted!

// BitSet (สำหรับ Int elements)
import scala.collection.immutable.BitSet
val bits = BitSet(1, 3, 5, 7)
println(bits)  // BitSet(1, 3, 5, 7)
```

---

## Map

### การสร้าง Map

```scala
// สร้าง Map
val empty = Map.empty[String, Int]
val m1 = Map("a" -> 1, "b" -> 2, "c" -> 3)
val m2 = Map(("a", 1), ("b", 2), ("c", 3))  // เหมือนกัน

// From sequences
val keys = List("x", "y", "z")
val values = List(10, 20, 30)
val fromLists = keys.zip(values).toMap
// Map(x -> 10, y -> 20, z -> 30)

// Map.from
val fromPairs = Map.from(List("a" -> 1, "b" -> 2))
```

### Map Access

```scala
val m = Map("alice" -> 30, "bob" -> 25, "charlie" -> 35)

// Access ที่อาจ throw exception
println(m("alice"))        // 30
// println(m("unknown"))  // ❌ NoSuchElementException

// Safe access
println(m.get("alice"))    // Some(30)
println(m.get("unknown"))  // None
println(m.getOrElse("unknown", 0))  // 0
println(m.withDefaultValue(0)("unknown"))  // 0

// ตรวจสอบ
println(m.contains("alice"))   // true
println(m.keys.toList)         // List(alice, bob, charlie)
println(m.values.toList)       // List(30, 25, 35)
println(m.keySet)              // Set(alice, bob, charlie)
println(m.size)                // 3
```

### Map Operations

```scala
val m = Map("alice" -> 30, "bob" -> 25)

// เพิ่ม/อัพเดท
val m2 = m + ("charlie" -> 35)     // เพิ่ม entry ใหม่
val m3 = m + ("alice" -> 31)       // อัพเดท alice
val m4 = m ++ Map("charlie" -> 35, "diana" -> 28)  // merge

// ลบ
val m5 = m - "alice"               // ลบ alice
val m6 = m -- List("alice", "bob") // ลบหลาย keys

// Transform
val doubled = m.view.mapValues(_ * 2).toMap
// Map(alice -> 60, bob -> 50)

val capitalized = m.map { case (k, v) => k.capitalize -> v }
// Map(Alice -> 30, Bob -> 25)

val filtered = m.filter { case (_, age) => age >= 28 }
// Map(alice -> 30)

// Filter keys/values
val byKey = m.filterKeys(_.startsWith("a")).toMap
// Map(alice -> 30)

// Fold over Map
val totalAge = m.values.sum   // 55
val names = m.keys.mkString(", ")  // alice, bob

// Convert
val list = m.toList   // List((alice,30),(bob,25))
val sorted = m.toList.sortBy(_._1)  // sorted by key
```

### Grouped Maps

```scala
val students = List(
  ("Alice", "Math", 95),
  ("Bob", "Math", 82),
  ("Alice", "Science", 88),
  ("Bob", "Science", 91),
  ("Charlie", "Math", 76)
)

// Group by student
val byStudent = students.groupBy(_._1)
  .view.mapValues(_.map { case (_, subject, score) => subject -> score }.toMap)
  .toMap

byStudent.foreach { case (name, scores) =>
  println(s"$name: ${scores.map { case (s, n) => s"$s=$n" }.mkString(", ")}")
}

// Nested Map operations
val avgBySubject = students
  .groupBy(_._2)
  .view.mapValues { group =>
    val scores = group.map(_._3)
    scores.sum.toDouble / scores.length
  }
  .toMap

avgBySubject.foreach { case (subject, avg) =>
  println(f"$subject average: $avg%.1f")
}
```

---

## Tuple

```scala
// Tuple: ordered collection ที่ fixed-size และมี type ต่างกันได้

// สร้าง
val pair: (Int, String) = (42, "hello")
val triple: (String, Int, Boolean) = ("Alice", 30, true)

// ผ่าน -> syntax (เฉพาะ Tuple2)
val mapEntry = "key" -> "value"

// Access
println(pair._1)   // 42
println(pair._2)   // hello
println(triple._1) // Alice
println(triple._3) // true

// Destructuring
val (num, str) = pair
val (name, age, active) = triple

// swap
val swapped = pair.swap  // (hello, 42)

// Tuple ใน collections
val coords = List((1, 2), (3, 4), (5, 6))
val sumCoords = coords.map { case (x, y) => x + y }
// List(3, 7, 11)

// unzip
val (xs, ys) = coords.unzip
println(xs)  // List(1, 3, 5)
println(ys)  // List(2, 4, 6)

// Tuple3 unzip3
val triples = List((1, "a", true), (2, "b", false))
val (ns, ss, bs) = triples.unzip3
```

---

## Range

```scala
// Range: lazy sequence ของ numbers

// Inclusive range (to)
val r1 = 1 to 10          // 1,2,...,10
val r2 = 1 to 10 by 2     // 1,3,5,7,9
val r3 = 10 to 1 by -1    // 10,9,...,1

// Exclusive range (until)
val r4 = 0 until 10       // 0,1,...,9
val r5 = 0 until 10 by 3  // 0,3,6,9

// Range operations
println(r1.contains(5))  // true
println(r1.min)          // 1
println(r1.max)          // 10
println(r1.sum)          // 55
println(r1.size)         // 10
println(r1.toList)       // List(1,2,...,10)

// Range ประหยัด memory มาก (lazy)
val huge = 1 to 1000000000  // ไม่ allocate memory สำหรับทุก element
println(huge.contains(500000000))  // true (O(1)!)
println(huge.sum)  // อาจ overflow! ใช้ BigInt แทน

// Char range
val letters = 'a' to 'z'
println(letters.toList.take(5))  // List(a, b, c, d, e)
println(letters.mkString)        // abcdefghijklmnopqrstuvwxyz
```

---

## Mutable vs Immutable

### Immutable Collections (default)

```scala
import scala.collection.immutable.*

// Immutable List
val list = List(1, 2, 3)
val newList = list :+ 4        // สร้าง list ใหม่

// Immutable Map
val map = Map("a" -> 1)
val newMap = map + ("b" -> 2)  // สร้าง map ใหม่

// Immutable Set
val set = Set(1, 2, 3)
val newSet = set + 4           // สร้าง set ใหม่
```

### Mutable Collections

```scala
import scala.collection.mutable

// ListBuffer: mutable list ที่ efficient
val lb = mutable.ListBuffer(1, 2, 3)
lb.append(4)         // เพิ่มท้าย
lb.prepend(0)        // เพิ่มหน้า
lb += 5              // append
lb ++= List(6, 7)    // appendAll
lb.remove(0)         // ลบ index 0
lb -= 4              // ลบ element 4
println(lb)          // ListBuffer(1, 2, 3, 5, 6, 7)

// ArrayBuffer: mutable, indexed
val ab = mutable.ArrayBuffer(1, 2, 3)
ab += 4
ab(0) = 10           // update by index
println(ab)          // ArrayBuffer(10, 2, 3, 4)

// HashMap: mutable
val hm = mutable.HashMap("a" -> 1)
hm("b") = 2          // add/update
hm += ("c" -> 3)
hm -= "a"
println(hm)          // HashMap(b -> 2, c -> 3)

// HashSet: mutable
val hs = mutable.HashSet(1, 2, 3)
hs += 4
hs -= 1
println(hs)          // HashSet(2, 3, 4)

// Queue
val q = mutable.Queue(1, 2, 3)
q.enqueue(4)
println(q.dequeue())  // 1
println(q)            // Queue(2, 3, 4)

// Stack
val s = mutable.Stack(1, 2, 3)
s.push(4)
println(s.pop())  // 4
println(s)        // Stack(1, 2, 3)
```

### แนวทางการเลือก

```scala
// หลักการเลือก:
// 1. ใช้ immutable เสมอถ้าเป็นไปได้ (thread-safe, easier to reason)
// 2. ใช้ mutable เมื่อ performance critical หรือ algorithm ต้องการ mutation

// ❌ หลีกเลี่ยง:
var list = List[Int]()
for i <- 1 to 1000 do
  list = list :+ i  // O(n) ต่อ operation = O(n^2) รวม

// ✅ ใช้ ListBuffer แทน:
val lb = mutable.ListBuffer[Int]()
for i <- 1 to 1000 do
  lb += i  // O(1) ต่อ operation

val result = lb.toList  // แปลงกลับเป็น immutable

// ✅ หรือใช้ functional:
val result2 = (1 to 1000).toList
```

---

## Collection Operations ครบถ้วน

### Operations ที่ใช้บ่อย

```scala
val nums = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// Querying
println(nums.head)           // 1
println(nums.last)           // 10
println(nums.headOption)     // Some(1)
println(nums.lastOption)     // Some(10)
println(nums.find(_ > 5))    // Some(6)
println(nums.findLast(_ < 5)) // Some(4)
println(nums.indexOf(5))      // 4
println(nums.count(_ % 2 == 0)) // 5

// Slicing
println(nums.take(3))           // List(1,2,3)
println(nums.drop(7))           // List(8,9,10)
println(nums.takeRight(3))      // List(8,9,10)
println(nums.dropRight(7))      // List(1,2,3)
println(nums.slice(2, 5))       // List(3,4,5)
println(nums.takeWhile(_ < 5))  // List(1,2,3,4)
println(nums.dropWhile(_ < 5))  // List(5,6,7,8,9,10)

// Transformation
println(nums.map(_ * 2))                    // doubled
println(nums.flatMap(n => List(n, -n)))     // each with negative
println(nums.filter(_ % 3 == 0))            // divisible by 3
println(nums.collect { case n if n > 7 => n * 10 }) // List(80,90,100)

// Grouping
println(nums.grouped(3).toList)  // List(List(1,2,3), List(4,5,6), List(7,8,9), List(10))
println(nums.sliding(3).toList)  // overlapping windows of size 3
println(nums.groupBy(_ % 3))     // Map(0 -> List(3,6,9), 1 -> List(1,4,7,10), 2 -> List(2,5,8))
println(nums.partition(_ % 2 == 0)) // (List(2,4,6,8,10), List(1,3,5,7,9))

// Aggregation
println(nums.sum)           // 55
println(nums.product)       // 3628800
println(nums.min)           // 1
println(nums.max)           // 10
println(nums.reduce(_ + _)) // 55
println(nums.foldLeft(0)(_ + _))  // 55
println(nums.scanLeft(0)(_ + _))  // running sum

// Sorting
val words = List("banana", "apple", "cherry", "date", "elderberry")
println(words.sorted)                     // alphabetical
println(words.sortBy(_.length))           // by length
println(words.sortWith(_.length > _.length)) // by length descending
println(words.sortBy(w => (-w.length, w))) // length desc, then alpha

// Set operations (เมื่อ convert เป็น Set)
val a = List(1, 2, 3, 4)
val b = List(3, 4, 5, 6)
println(a.intersect(b))     // List(3,4)
println(a.diff(b))          // List(1,2)
println((a ++ b).distinct)  // List(1,2,3,4,5,6)

// Flattening
val nested = List(List(1,2), List(3,4), List(5))
println(nested.flatten)  // List(1,2,3,4,5)

val nested2 = List(Some(1), None, Some(3), None, Some(5))
println(nested2.flatten)  // List(1,3,5)

// Zipping
val xs = List(1, 2, 3)
val ys = List("a", "b", "c")
println(xs.zip(ys))          // List((1,a),(2,b),(3,c))
println(xs.zipWithIndex)     // List((1,0),(2,1),(3,2))
println(xs.zipAll(List(1,2), 0, "z"))  // pad shorter list

// Combining
println(xs ++ ys)             // List(1,2,3,a,b,c)
println(xs.prependedAll(ys))  // List(a,b,c,1,2,3) (Scala 2.13+)
```

---

## Performance Comparison

```scala
// เปรียบเทียบ performance ของ collections ต่างๆ

// List: O(1) prepend, O(n) access by index
// Vector: O(log n) ≈ O(1) append/prepend/access
// Array: O(1) access, O(n) insertion
// HashMap: O(1) avg access
// TreeMap: O(log n) access, sorted

// ตัวอย่างการเลือก collection ที่ถูกต้อง:

// ✅ ใช้ List เมื่อ:
// - access ทาง head/tail
// - ส่วนใหญ่ prepend
// - pattern matching

val list = 1 :: 2 :: 3 :: Nil

// ✅ ใช้ Vector เมื่อ:
// - random access บ่อย
// - append บ่อย
// - general purpose sequential collection

val vector = Vector(1, 2, 3)
val updated = vector.updated(1, 99)  // efficient!

// ✅ ใช้ Array เมื่อ:
// - ต้องการ mutable
// - Java interop
// - primitive types (no boxing overhead)

val array = Array(1, 2, 3)
array(1) = 99  // in-place update

// ✅ ใช้ Map เมื่อ:
// - lookup by key
// - key-value relationships

val map = Map("alice" -> 30, "bob" -> 25)
val age = map("alice")  // O(1) avg

// Benchmark example
def timeIt[T](label: String)(block: => T): T =
  val start = System.nanoTime()
  val result = block
  val elapsed = (System.nanoTime() - start) / 1_000_000.0
  println(f"$label: $elapsed%.2f ms")
  result

val n = 100000

timeIt("List prepend") {
  var list = List[Int]()
  for i <- 1 to n do list = i :: list
}

timeIt("Vector append") {
  var vec = Vector[Int]()
  for i <- 1 to n do vec = vec :+ i
}

timeIt("ArrayBuffer append") {
  val ab = scala.collection.mutable.ArrayBuffer[Int]()
  for i <- 1 to n do ab += i
}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: List Operations

```scala
@main def listExercise(): Unit =
  val data = List(
    ("Alice", 85.0),
    ("Bob", 92.5),
    ("Charlie", 78.0),
    ("Diana", 95.5),
    ("Eve", 88.0),
    ("Frank", 72.0),
    ("Grace", 91.0)
  )

  // TODO:
  // 1. หาชื่อของนักเรียนที่ได้คะแนน >= 90
  // 2. คำนวณค่าเฉลี่ยของทุกคน
  // 3. จัด rank (1 = สูงสุด) ด้วย zipWithIndex หลัง sort
  // 4. หานักเรียนที่ได้คะแนนสูงสุดและต่ำสุด
  // 5. แสดง top 3 นักเรียน
```

### แบบฝึกหัดที่ 2: Map Operations

```scala
@main def mapExercise(): Unit =
  val inventory = Map(
    "apple" -> 50,
    "banana" -> 30,
    "cherry" -> 100,
    "date" -> 15,
    "elderberry" -> 5
  )

  // TODO:
  // 1. หา items ที่มีจำนวนน้อยกว่า 20
  // 2. เพิ่ม "fig" -> 40 เข้าไป
  // 3. อัพเดท "apple" เป็น 60
  // 4. คำนวณ total inventory
  // 5. แสดง inventory ที่ sorted by quantity (มากไปน้อย)
```

### แบบฝึกหัดที่ 3: Word Count

```scala
@main def wordCount(): Unit =
  val text = """
    To be or not to be that is the question
    Whether tis nobler in the mind to suffer
    The slings and arrows of outrageous fortune
    Or to take arms against a sea of troubles
  """.trim

  // TODO:
  // 1. split ข้อความเป็นคำ
  // 2. normalize (lowercase, ตัด whitespace)
  // 3. นับ frequency ของแต่ละคำ
  // 4. แสดง top 5 คำที่พบบ่อยที่สุด
  // 5. หาคำที่มีความยาวมากกว่า 4 ตัวอักษร
  // 6. แสดง unique words (sorted)
```

### แบบฝึกหัดที่ 4: Collection Transformations

```scala
@main def transformExercise(): Unit =
  case class Employee(
    name: String,
    department: String,
    salary: Double,
    yearsExp: Int
  )

  val employees = List(
    Employee("Alice", "Engineering", 95000, 5),
    Employee("Bob", "Marketing", 65000, 3),
    Employee("Charlie", "Engineering", 110000, 8),
    Employee("Diana", "HR", 55000, 2),
    Employee("Eve", "Engineering", 85000, 4),
    Employee("Frank", "Marketing", 72000, 6),
    Employee("Grace", "HR", 62000, 4)
  )

  // TODO:
  // 1. Group by department
  // 2. Average salary per department
  // 3. Highest paid in each department
  // 4. Employees with salary > average
  // 5. Total salary bill
  // 6. Department with most employees
```

**เฉลย แบบฝึกหัดที่ 3:**

```scala
@main def wordCount(): Unit =
  val text = """
    To be or not to be that is the question
    Whether tis nobler in the mind to suffer
    The slings and arrows of outrageous fortune
    Or to take arms against a sea of troubles
  """.trim

  val words = text.toLowerCase.split("\\s+").toList.filter(_.nonEmpty)
  val frequency = words.groupBy(identity).view.mapValues(_.length).toMap
  val sorted = frequency.toList.sortBy(-_._2)

  println("Top 5 words:")
  sorted.take(5).foreach { case (word, count) =>
    println(f"  $word%-15s: $count")
  }

  println(s"\nWords longer than 4 chars:")
  words.distinct.filter(_.length > 4).sorted.foreach(w => print(s"$w "))
  println()

  println(s"\nUnique words (${words.distinct.length}):")
  println(words.distinct.sorted.mkString(", "))
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Collection hierarchy ของ Scala
- ✅ List: operations ครบถ้วน
- ✅ Vector: indexed sequence ที่ performant
- ✅ Array: mutable Java array
- ✅ Set: unique elements และ set operations
- ✅ Map: key-value pairs และ operations
- ✅ Tuple: fixed-size heterogeneous collection
- ✅ Range: lazy numeric sequence
- ✅ Mutable vs Immutable collections
- ✅ Performance characteristics ของแต่ละ collection

## ขั้นตอนถัดไป

ใน [Part 08: การจัดการ String](part-08-strings.md) เราจะเรียนรู้:
- String operations เชิงลึก
- Regular expressions
- String parsing
- Text processing patterns

---

*[← Part 06: ฟังก์ชัน](part-06-functions.md) | [Part 08: การจัดการ String →](part-08-strings.md)*
