# ส่วนที่ 102 (BONUS): Recursion Schemes ใน Scala

> **BONUS CONTENT** - เนื้อหาขั้นสูงเกี่ยวกับ Recursion Schemes สำหรับการจัดการ recursive data structures อย่างสง่างาม

---

## สารบัญ

1. [บทนำ: ปัญหาของ Recursion ทั่วไป](#บทนำ)
2. [Fixed Point Types: Fix[F]](#fixed-point-types)
3. [Algebra และ Coalgebra](#algebra-และ-coalgebra)
4. [Catamorphism (fold)](#catamorphism)
5. [Anamorphism (unfold)](#anamorphism)
6. [Hylomorphism](#hylomorphism)
7. [Paramorphism](#paramorphism)
8. [Apomorphism](#apomorphism)
9. [Histomorphism](#histomorphism)
10. [Futumorphism](#futumorphism)
11. [ตัวอย่าง: JSON Evaluator](#json-evaluator)
12. [สรุป](#สรุป)

---

## บทนำ

Recursion Schemes เป็นวิธีการ abstract recursion ออกจาก data structure เพื่อให้โค้ดเป็น modular และ reusable มากขึ้น

### ปัญหาของ Recursion ทั่วไป

```scala
// Naive recursive evaluation - recursion กระจายทั่วทุกที่
sealed trait Expr
case class Num(value: Int) extends Expr
case class Add(left: Expr, right: Expr) extends Expr
case class Mul(left: Expr, right: Expr) extends Expr
case class Neg(expr: Expr) extends Expr

// Evaluator ต้องจัดการ recursion เอง
def eval(expr: Expr): Int = expr match
  case Num(n)    => n
  case Add(l, r) => eval(l) + eval(r)
  case Mul(l, r) => eval(l) * eval(r)
  case Neg(e)    => -eval(e)

// Printer ก็ต้องทำเหมือนกัน
def print(expr: Expr): String = expr match
  case Num(n)    => n.toString
  case Add(l, r) => s"(${print(l)} + ${print(r)})"
  case Mul(l, r) => s"(${print(l)} * ${print(r)})"
  case Neg(e)    => s"-${print(e)}"

// ปัญหา: recursion ซ้ำๆ และ stack overflow สำหรับ deep structures
```

### แนวทางแก้ไข: แยก Recursion ออกจาก Logic

```scala
// แยก recursive structure ออกมาเป็น type parameter
sealed trait ExprF[A]  // A แทน "recursive call result"
case class NumF[A](value: Int) extends ExprF[A]
case class AddF[A](left: A, right: A) extends ExprF[A]
case class MulF[A](left: A, right: A) extends ExprF[A]
case class NegF[A](expr: A) extends ExprF[A]

// ExprF เป็น Functor
given Functor[ExprF] with
  def map[A, B](fa: ExprF[A])(f: A => B): ExprF[B] = fa match
    case NumF(n)    => NumF(n)
    case AddF(l, r) => AddF(f(l), f(r))
    case MulF(l, r) => MulF(f(l), f(r))
    case NegF(e)    => NegF(f(e))
```

---

## Fixed Point Types

Fix[F] คือ type ที่เป็น "fixed point" ของ type constructor F

### การนิยาม Fix

```scala
// Fix[F] = F[Fix[F]]
// นี่คือ infinite type เราใช้ newtype เพื่อ break the cycle
case class Fix[F[_]](unfix: F[Fix[F]])

// Alias สำหรับความสะดวก
type Expr2 = Fix[ExprF]

// Smart constructors
def num(n: Int): Expr2 = Fix(NumF(n))
def add(l: Expr2, r: Expr2): Expr2 = Fix(AddF(l, r))
def mul(l: Expr2, r: Expr2): Expr2 = Fix(MulF(l, r))
def neg(e: Expr2): Expr2 = Fix(NegF(e))

// สร้าง expression: (2 + 3) * -(4)
val expr: Expr2 = mul(add(num(2), num(3)), neg(num(4)))
```

### Mu และ Nu

```scala
// Fix เหมาะสำหรับ finite structures
// Mu = Least Fixed Point (finite, inductive)
// Nu = Greatest Fixed Point (potentially infinite, coinductive)

// Mu[F]: ใช้สำหรับ recursive types ที่ finite
newtype Mu[F[_]] = Mu { def fold[A](alg: F[A] => A): A }

// Nu[F]: ใช้สำหรับ corecursive types ที่อาจ infinite
case class Nu[F[_]](head: Any, tail: Any => F[Any])
```

---

## Algebra และ Coalgebra

### Algebra

```scala
// Algebra[F, A] = F[A] => A
// "วิธีคำนวณ A จาก F[A]"
type Algebra[F[_], A] = F[A] => A

// ตัวอย่าง Algebras สำหรับ ExprF
val evalAlg: Algebra[ExprF, Int] = {
  case NumF(n)    => n
  case AddF(l, r) => l + r
  case MulF(l, r) => l * r
  case NegF(e)    => -e
}

val printAlg: Algebra[ExprF, String] = {
  case NumF(n)    => n.toString
  case AddF(l, r) => s"($l + $r)"
  case MulF(l, r) => s"($l * $r)"
  case NegF(e)    => s"-$e"
}

val countAlg: Algebra[ExprF, Int] = {
  case NumF(_)    => 1
  case AddF(l, r) => l + r + 1
  case MulF(l, r) => l + r + 1
  case NegF(e)    => e + 1
}
```

### Coalgebra

```scala
// Coalgebra[F, A] = A => F[A]
// "วิธีสร้าง F[A] จาก A" (unfolding)
type Coalgebra[F[_], A] = A => F[A]

// ตัวอย่าง Coalgebra สำหรับสร้าง List structure
sealed trait ListF[+A, +R]
case object NilF extends ListF[Nothing, Nothing]
case class ConsF[A, R](head: A, tail: R) extends ListF[A, R]

type MyListF[A] = [R] =>> ListF[A, R]  // Higher-kinded alias

// Coalgebra สำหรับสร้าง range
def rangeCoalg(end: Int): Coalgebra[[R] =>> ListF[Int, R], Int] =
  n => if n >= end then NilF else ConsF(n, n + 1)
```

---

## Catamorphism

Catamorphism (cata) คือ "fold" - ทำลาย structure และสร้าง summary value

### การนิยาม

```scala
// cata :: Functor f => (f a -> a) -> Fix f -> a
def cata[F[_]: Functor, A](alg: Algebra[F, A])(fix: Fix[F]): A =
  alg(fix.unfix.map(cata(alg)))
// อ่าน: แกะ Fix -> map cata เข้าไปใน children -> apply algebra
```

### ตัวอย่างการใช้งาน

```scala
// Evaluate expression
val result = cata(evalAlg)(expr)
println(s"Result: $result")  // Result: -20 (= (2+3) * -4)

// Print expression
val printed = cata(printAlg)(expr)
println(s"Expr: $printed")  // Expr: ((2 + 3) * -4)

// Count nodes
val nodeCount = cata(countAlg)(expr)
println(s"Nodes: $nodeCount")  // Nodes: 7

// Depth
val depthAlg: Algebra[ExprF, Int] = {
  case NumF(_)    => 0
  case AddF(l, r) => 1 + (l max r)
  case MulF(l, r) => 1 + (l max r)
  case NegF(e)    => 1 + e
}

val depth = cata(depthAlg)(expr)
println(s"Depth: $depth")  // Depth: 3
```

### Catamorphism สำหรับ List

```scala
// Nat สำหรับแสดง Natural Numbers
sealed trait NatF[+A]
case object ZeroF extends NatF[Nothing]
case class SuccF[A](pred: A) extends NatF[A]

given Functor[NatF] with
  def map[A, B](fa: NatF[A])(f: A => B): NatF[B] = fa match
    case ZeroF    => ZeroF
    case SuccF(n) => SuccF(f(n))

type Nat = Fix[NatF]

val zero: Nat = Fix(ZeroF)
def succ(n: Nat): Nat = Fix(SuccF(n))

// Build 3
val three: Nat = succ(succ(succ(zero)))

// Convert to Int
val toIntAlg: Algebra[NatF, Int] = {
  case ZeroF    => 0
  case SuccF(n) => n + 1
}

println(cata(toIntAlg)(three))  // 3
```

### Catamorphism สำหรับ Tree

```scala
sealed trait TreeF[+A, +R]
case class LeafF[A](value: A) extends TreeF[A, Nothing]
case class BranchF[A, R](left: R, value: A, right: R) extends TreeF[A, R]

type BinaryTree[A] = Fix[[R] =>> TreeF[A, R]]

given [A]: Functor[[R] =>> TreeF[A, R]] with
  def map[X, Y](fa: TreeF[A, X])(f: X => Y): TreeF[A, Y] = fa match
    case LeafF(v)       => LeafF(v)
    case BranchF(l, v, r) => BranchF(f(l), v, f(r))

// Sum all values in tree
def sumTreeAlg[A: Numeric]: Algebra[[R] =>> TreeF[A, R], A] = {
  case LeafF(v)       => v
  case BranchF(l, v, r) => implicitly[Numeric[A]].plus(implicitly[Numeric[A]].plus(l, v), r)
}
```

---

## Anamorphism

Anamorphism (ana) คือ "unfold" - สร้าง structure จาก seed value

### การนิยาม

```scala
// ana :: Functor f => (a -> f a) -> a -> Fix f
def ana[F[_]: Functor, A](coalg: Coalgebra[F, A])(seed: A): Fix[F] =
  Fix(coalg(seed).map(ana(coalg)))
// อ่าน: apply coalgebra -> wrap ใน Fix -> map ana เข้าไปใน children
```

### ตัวอย่างการใช้งาน

```scala
// Unfold: สร้าง countdown timer
val countdownCoalg: Coalgebra[ExprF, Int] =
  n => if n <= 0 then NumF(0) else AddF(n, n - 1)  // simplified

// สร้าง Fibonacci sequence
sealed trait StreamF[+A, +R]
case class StreamConsF[A, R](head: A, tail: R) extends StreamF[A, R]

given [A]: Functor[[R] =>> StreamF[A, R]] with
  def map[X, Y](fa: StreamF[A, X])(f: X => Y): StreamF[A, Y] = fa match
    case StreamConsF(h, t) => StreamConsF(h, f(t))

// Fibonacci Coalgebra
val fibCoalg: Coalgebra[[R] =>> StreamF[Long, R], (Long, Long)] =
  (a, b) => StreamConsF(a, (b, a + b))

// Take n elements from stream
def take[A](n: Int)(stream: Fix[[R] =>> StreamF[A, R]]): List[A] =
  if n <= 0 then Nil
  else stream.unfix match
    case StreamConsF(h, t) => h :: take(n - 1)(t)

val fibStream = ana(fibCoalg)((0L, 1L))
println(take(10)(fibStream))
// List(0, 1, 1, 2, 3, 5, 8, 13, 21, 34)
```

### Range ด้วย Anamorphism

```scala
// สร้าง List จาก Range
sealed trait ListF2[+A, +R]
case object NilF2 extends ListF2[Nothing, Nothing]
case class ConsF2[A, R](head: A, tail: R) extends ListF2[A, R]

given [A]: Functor[[R] =>> ListF2[A, R]] with
  def map[X, Y](fa: ListF2[A, X])(f: X => Y): ListF2[A, Y] = fa match
    case NilF2         => NilF2
    case ConsF2(h, t)  => ConsF2(h, f(t))

def rangeCoalg2(end: Int): Coalgebra[[R] =>> ListF2[Int, R], Int] =
  n => if n >= end then NilF2 else ConsF2(n, n + 1)

// Convert to List
val toListAlg: Algebra[[R] =>> ListF2[Int, R], List[Int]] = {
  case NilF2         => Nil
  case ConsF2(h, t)  => h :: t
}

// สร้าง List(0..9) ด้วย ana แล้ว fold ด้วย cata
val fixList = ana[[R] =>> ListF2[Int, R], Int](rangeCoalg2(10))(0)
val result = cata[[R] =>> ListF2[Int, R], List[Int]](toListAlg)(fixList)
println(result)  // List(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
```

---

## Hylomorphism

Hylomorphism = Anamorphism แล้วตาม Catamorphism (build then fold)

### การนิยาม

```scala
// hylo :: Functor f => (f b -> b) -> (a -> f a) -> a -> b
def hylo[F[_]: Functor, A, B](alg: Algebra[F, B])(coalg: Coalgebra[F, A])(seed: A): B =
  alg(coalg(seed).map(hylo(alg)(coalg)))
// ไม่สร้าง intermediate Fix! - efficient กว่า ana แล้ว cata
```

### Merge Sort ด้วย Hylomorphism

```scala
// Merge Sort เป็น Hylomorphism ที่สวยงาม
sealed trait TreeF2[+A, +R]
case class LeafF2[A](values: List[A]) extends TreeF2[A, Nothing]
case class BranchF2[A, R](left: R, right: R) extends TreeF2[A, R]

given [A]: Functor[[R] =>> TreeF2[A, R]] with
  def map[X, Y](fa: TreeF2[A, X])(f: X => Y): TreeF2[A, Y] = fa match
    case LeafF2(vs)     => LeafF2(vs)
    case BranchF2(l, r) => BranchF2(f(l), f(r))

// Split: divide list into two halves (Coalgebra)
def splitCoalg[A]: Coalgebra[[R] =>> TreeF2[A, R], List[A]] = list =>
  if list.length <= 1 then LeafF2(list)
  else
    val (left, right) = list.splitAt(list.length / 2)
    BranchF2(left, right)

// Merge: combine two sorted lists (Algebra)
def mergeAlg[A: Ordering]: Algebra[[R] =>> TreeF2[A, R], List[A]] = {
  case LeafF2(vs)     => vs.sorted
  case BranchF2(l, r) => mergeSorted(l, r)
}

def mergeSorted[A: Ordering](xs: List[A], ys: List[A]): List[A] =
  (xs, ys) match
    case (Nil, _)   => ys
    case (_, Nil)   => xs
    case (x :: xt, y :: yt) =>
      if implicitly[Ordering[A]].lteq(x, y) then x :: mergeSorted(xt, ys)
      else y :: mergeSorted(xs, yt)

// Merge Sort = hylo
def mergeSort[A: Ordering](list: List[A]): List[A] =
  hylo[[R] =>> TreeF2[A, R], List[A], List[A]](mergeAlg[A])(splitCoalg[A])(list)

println(mergeSort(List(5, 2, 8, 1, 9, 3)))
// List(1, 2, 3, 5, 8, 9)
```

### Factorial ด้วย Hylomorphism

```scala
// build จาก n ลงมาถึง 0 แล้ว multiply
val factCoalg: Coalgebra[ExprF, Int] =
  n => if n <= 0 then NumF(1) else MulF(n, n - 1)

val mulAlg: Algebra[ExprF, Int] = {
  case NumF(n)    => n
  case MulF(l, r) => l * r
  case AddF(l, r) => l + r
  case NegF(e)    => -e
}

def factorial(n: Int): Int = hylo(mulAlg)(factCoalg)(n)
println(factorial(5))  // 120
```

---

## Paramorphism

Paramorphism (para) คือ catamorphism ที่มี access ถึง original substructure ด้วย

### นิยาม

```scala
// RAlgebra[F, A] = F[(Fix[F], A)] => A
// เราเห็นทั้ง "processed result" และ "original structure"
type RAlgebra[F[_], A] = F[(Fix[F], A)] => A

def para[F[_]: Functor, A](ralg: RAlgebra[F, A])(fix: Fix[F]): A =
  ralg(fix.unfix.map(child => (child, para(ralg)(child))))
```

### ตัวอย่าง: Pretty Print พร้อม Context

```scala
// Pretty print ที่รู้ว่า parent คืออะไร
val prettyPrintAlg: RAlgebra[ExprF, String] = {
  case NumF(n) => n.toString
  case NegF((Fix(NegF(_)), result)) => result  // double neg = simplify
  case NegF((_, result)) => s"-$result"
  case AddF((_, l), (_, r)) => s"$l + $r"
  case MulF((Fix(AddF(_, _)), l), (_, r)) => s"($l) * $r"  // ใส่วงเล็บถ้า left เป็น Add
  case MulF((_, l), (Fix(AddF(_, _)), r)) => s"$l * ($r)"  // ใส่วงเล็บถ้า right เป็น Add
  case MulF((_, l), (_, r)) => s"$l * $r"
}
```

### Fibonacci ด้วย Paramorphism

```scala
// Fibonacci ต้องการ access ถึง n-2 terms
// Paramorphism ช่วยได้!
val fibAlg: RAlgebra[NatF, Long] = {
  case ZeroF => 0L
  case SuccF((Fix(ZeroF), _)) => 1L  // fib(1) = 1
  case SuccF((Fix(SuccF((_, prevResult))), result)) => result + prevResult
  // fib(n) = fib(n-1) + fib(n-2)
}

def fibonacci(n: Int): Long =
  para(fibAlg)(buildNat(n))

def buildNat(n: Int): Fix[NatF] =
  if n <= 0 then Fix(ZeroF) else Fix(SuccF(buildNat(n - 1)))

// Test
(0 to 10).foreach(n => print(s"${fibonacci(n)} "))
// 0 1 1 2 3 5 8 13 21 34 55
```

---

## Apomorphism

Apomorphism (apo) คือ anamorphism ที่สามารถ "short-circuit" ได้

### นิยาม

```scala
// RCoalgebra[F, A] = A => F[Either[Fix[F], A]]
// Left = หยุด (ใช้ existing structure)
// Right = ดำเนินต่อ (unfold ต่อ)
type RCoalgebra[F[_], A] = A => F[Either[Fix[F], A]]

def apo[F[_]: Functor, A](rcoalg: RCoalgebra[F, A])(seed: A): Fix[F] =
  Fix(rcoalg(seed).map {
    case Left(fix)  => fix    // short circuit
    case Right(a)   => apo(rcoalg)(a)  // continue unfolding
  })
```

### ตัวอย่าง: Insert ใน Sorted List

```scala
// Insert element ใน sorted list
def insertCoalg(x: Int): RCoalgebra[[R] =>> ListF2[Int, R], List[Int]] = {
  case Nil    => ConsF2(x, Left(Fix(NilF2)))  // insert at end
  case h :: t =>
    if x <= h then ConsF2(x, Left(Fix(ConsF2(h, ???))))  // insert before h
    else ConsF2(h, Right(t))  // continue
}
// simplified version:
def insertInSorted(x: Int)(list: List[Int]): List[Int] =
  list match
    case Nil    => List(x)
    case h :: t => if x <= h then x :: list else h :: insertInSorted(x)(t)
```

---

## Histomorphism

Histomorphism (histo) คือ catamorphism ที่มี access ถึง history ของการ compute ทั้งหมด

### นิยาม

```scala
// เราใช้ Cofree เป็น "history"
case class Cofree[F[_], A](head: A, tail: F[Cofree[F, A]])

type CVAlgebra[F[_], A] = F[Cofree[F, A]] => A

def histo[F[_]: Functor, A](cvalg: CVAlgebra[F, A])(fix: Fix[F]): A =
  def toHistory(fix: Fix[F]): Cofree[F, A] =
    val tail = fix.unfix.map(toHistory)
    Cofree(cvalg(tail), tail)
  toHistory(fix).head
```

### Fibonacci ด้วย Histomorphism (Efficient)

```scala
// Histomorphism ทำให้ Fibonacci เป็น O(n) เพราะ access ถึง cache ได้
val efficientFibAlg: CVAlgebra[NatF, Long] = {
  case ZeroF => 0L
  case SuccF(Cofree(a, ZeroF)) => 1L  // fib(1) = 1
  case SuccF(Cofree(fib_n_1, SuccF(Cofree(fib_n_2, _)))) =>
    fib_n_1 + fib_n_2  // fib(n) = fib(n-1) + fib(n-2)
}
```

---

## Futumorphism

Futumorphism (futu) คือ anamorphism ที่สามารถ "look ahead" ได้

### นิยาม

```scala
// ใช้ Free Monad เป็น "future"
// CVCoalgebra[F, A] = A => F[Free[F, A]]
type CVCoalgebra[F[_], A] = A => F[cats.free.Free[F, A]]

// futu สร้าง structure หลายขั้นพร้อมกัน
```

---

## JSON Evaluator

ตัวอย่างสมบูรณ์: สร้าง JSON DSL ด้วย Recursion Schemes

### นิยาม JSON ADT

```scala
// JSON F-Algebra (parameterized)
sealed trait JsonF[+R]
case object JsonNullF extends JsonF[Nothing]
case class JsonBoolF(value: Boolean) extends JsonF[Nothing]
case class JsonNumberF(value: Double) extends JsonF[Nothing]
case class JsonStringF(value: String) extends JsonF[Nothing]
case class JsonArrayF[R](elements: List[R]) extends JsonF[R]
case class JsonObjectF[R](fields: List[(String, R)]) extends JsonF[R]

given Functor[JsonF] with
  def map[A, B](fa: JsonF[A])(f: A => B): JsonF[B] = fa match
    case JsonNullF            => JsonNullF
    case JsonBoolF(b)         => JsonBoolF(b)
    case JsonNumberF(n)       => JsonNumberF(n)
    case JsonStringF(s)       => JsonStringF(s)
    case JsonArrayF(elems)    => JsonArrayF(elems.map(f))
    case JsonObjectF(fields)  => JsonObjectF(fields.map((k, v) => (k, f(v))))

type Json = Fix[JsonF]

// Smart constructors
val jsonNull: Json = Fix(JsonNullF)
def jsonBool(b: Boolean): Json = Fix(JsonBoolF(b))
def jsonNum(n: Double): Json = Fix(JsonNumberF(n))
def jsonStr(s: String): Json = Fix(JsonStringF(s))
def jsonArr(elems: Json*): Json = Fix(JsonArrayF(elems.toList))
def jsonObj(fields: (String, Json)*): Json = Fix(JsonObjectF(fields.toList))
```

### JSON Algebra: Printer

```scala
val jsonPrintAlg: Algebra[JsonF, String] = {
  case JsonNullF          => "null"
  case JsonBoolF(b)       => b.toString
  case JsonNumberF(n)     =>
    if n == n.toLong then n.toLong.toString else n.toString
  case JsonStringF(s)     => s""""$s""""
  case JsonArrayF(elems)  => elems.mkString("[", ", ", "]")
  case JsonObjectF(fields) =>
    fields.map((k, v) => s""""$k": $v""").mkString("{", ", ", "}")
}

def jsonPrint(json: Json): String = cata(jsonPrintAlg)(json)
```

### JSON Algebra: Size Counter

```scala
val jsonSizeAlg: Algebra[JsonF, Int] = {
  case JsonNullF            => 1
  case JsonBoolF(_)         => 1
  case JsonNumberF(_)       => 1
  case JsonStringF(_)       => 1
  case JsonArrayF(elems)    => 1 + elems.sum
  case JsonObjectF(fields)  => 1 + fields.map(_._2).sum
}

def jsonSize(json: Json): Int = cata(jsonSizeAlg)(json)
```

### JSON Algebra: Depth

```scala
val jsonDepthAlg: Algebra[JsonF, Int] = {
  case JsonNullF            => 0
  case JsonBoolF(_)         => 0
  case JsonNumberF(_)       => 0
  case JsonStringF(_)       => 0
  case JsonArrayF(elems)    => 1 + (if elems.isEmpty then 0 else elems.max)
  case JsonObjectF(fields)  => 1 + (if fields.isEmpty then 0 else fields.map(_._2).max)
}
```

### JSON Algebra: Schema Inference

```scala
sealed trait JsonSchema
case object NullSchema extends JsonSchema
case object BoolSchema extends JsonSchema
case object NumberSchema extends JsonSchema
case object StringSchema extends JsonSchema
case class ArraySchema(elemSchema: JsonSchema) extends JsonSchema
case class ObjectSchema(fields: Map[String, JsonSchema]) extends JsonSchema
case object UnknownSchema extends JsonSchema

// Semigroup สำหรับ merge schemas
given Semigroup[JsonSchema] with
  def combine(x: JsonSchema, y: JsonSchema): JsonSchema = (x, y) match
    case (NullSchema, _) | (_, NullSchema) => UnknownSchema  // nullable
    case (a, b) if a == b => a
    case _ => UnknownSchema

val jsonSchemaAlg: Algebra[JsonF, JsonSchema] = {
  case JsonNullF            => NullSchema
  case JsonBoolF(_)         => BoolSchema
  case JsonNumberF(_)       => NumberSchema
  case JsonStringF(_)       => StringSchema
  case JsonArrayF(Nil)      => ArraySchema(UnknownSchema)
  case JsonArrayF(schemas)  =>
    ArraySchema(schemas.reduce((a, b) => Semigroup[JsonSchema].combine(a, b)))
  case JsonObjectF(fields)  =>
    ObjectSchema(fields.toMap)
}
```

### JSON Coalgebra: Parser (simplified)

```scala
// แปลง String เป็น Json structure
// (Simplified - real JSON parser ซับซ้อนกว่านี้มาก)
sealed trait ParseResult[+A]
case class ParseSuccess[A](value: A, remaining: String) extends ParseResult[A]
case class ParseFailure(message: String) extends ParseResult[Nothing]
```

### JSON Transformation: Hylomorphism

```scala
// Transform JSON: แปลง number เป็น string ใน object values
def transformNumbersCoalg: Coalgebra[JsonF, Json] = json =>
  json.unfix match
    case JsonObjectF(fields) =>
      JsonObjectF(fields.map { (k, v) => k -> v })
    case other => other

def numberToStringAlg: Algebra[JsonF, Json] = {
  case JsonObjectF(fields) =>
    Fix(JsonObjectF(fields.map { (k, v) =>
      v.unfix match
        case JsonNumberF(n) => (k, jsonStr(n.toString))
        case _              => (k, v)
    }))
  case other => Fix(other)
}
```

### ตัวอย่างการใช้งานทั้งหมด

```scala
// สร้าง JSON document
val document = jsonObj(
  "name"    -> jsonStr("Alice"),
  "age"     -> jsonNum(30),
  "active"  -> jsonBool(true),
  "scores"  -> jsonArr(jsonNum(95), jsonNum(87), jsonNum(92)),
  "address" -> jsonObj(
    "street" -> jsonStr("123 Main St"),
    "city"   -> jsonStr("Bangkok"),
    "zip"    -> jsonStr("10110")
  ),
  "notes"   -> jsonNull
)

// Print
println(jsonPrint(document))
/* Output:
{"name": "Alice", "age": 30, "active": true, "scores": [95, 87, 92], 
 "address": {"street": "123 Main St", "city": "Bangkok", "zip": "10110"}, 
 "notes": null}
*/

// Metrics
println(s"Size: ${jsonSize(document)}")   // Size: 13
println(s"Depth: ${cata(jsonDepthAlg)(document)}")  // Depth: 2

// Schema
val schema = cata(jsonSchemaAlg)(document)
println(schema)
// ObjectSchema(Map(name -> StringSchema, age -> NumberSchema, ...))
```

---

## Matryoshka Library

Matryoshka เป็น library สำหรับ Recursion Schemes ใน Scala

```scala
// build.sbt
// libraryDependencies += "com.slamdata" %% "matryoshka-core" % "0.21.3"

import matryoshka._
import matryoshka.data._
import matryoshka.implicits._

// ด้วย Matryoshka เราใช้ได้โดยตรง
val evalResult: Int = expr.cata(evalAlg)
val printed: String = expr.cata(printAlg)

// Matryoshka ยังมี:
// - Scheme.ghylo: generalized hylomorphism
// - ZipAlgebra: combine algebras
// - GAlgebra: generalized algebra
```

### ตัวอย่างกับ Matryoshka

```scala
// Optimize expression tree
def optimize: Algebra[ExprF, Fix[ExprF]] = {
  case AddF(Fix(NumF(0)), x)  => x           // 0 + x = x
  case AddF(x, Fix(NumF(0)))  => x           // x + 0 = x
  case MulF(Fix(NumF(1)), x)  => x           // 1 * x = x
  case MulF(x, Fix(NumF(1)))  => x           // x * 1 = x
  case MulF(Fix(NumF(0)), _)  => Fix(NumF(0)) // 0 * x = 0
  case MulF(_, Fix(NumF(0)))  => Fix(NumF(0)) // x * 0 = 0
  case NegF(Fix(NegF(x)))     => x           // --x = x
  case other                  => Fix(other)
}

// Apply optimizations (can run multiple passes)
def optimizeExpr(expr: Fix[ExprF]): Fix[ExprF] =
  cata(optimize)(expr)

// Test
val redundant = mul(num(1), add(num(2), num(0)))
println(jsonPrint(redundant))  // custom print
val optimized = optimizeExpr(redundant)
println(s"Before: ${cata(printAlg)(redundant)}")   // (1 * (2 + 0))
println(s"After: ${cata(printAlg)(optimized)}")    // 2
```

---

## สรุป

| Scheme | Type | Desc | ใช้เมื่อ |
|--------|------|------|---------|
| `cata` | `F[A] => A` | fold (destroy) | สรุป structure |
| `ana` | `A => F[A]` | unfold (build) | สร้าง structure |
| `hylo` | `F[B] => B, A => F[A]` | build then fold | pipeline ที่ไม่เก็บ intermediate |
| `para` | `F[(Fix[F], A)] => A` | fold + original | ต้องการ context จาก parent |
| `apo` | `A => F[Either[Fix[F], A]]` | unfold + short-circuit | early termination |
| `histo` | `F[Cofree[F, A]] => A` | fold + history | ต้องการ memoized subtree |
| `futu` | `A => F[Free[F, A]]` | unfold + lookahead | สร้างหลาย steps พร้อมกัน |

### Recursion Schemes ช่วยอะไร

1. **Separation of Concerns**: Logic แยกจาก Recursion
2. **Reusability**: Algebra/Coalgebra ใช้ซ้ำได้
3. **Composability**: Combine multiple algebras ด้วย `ZipAlgebra`
4. **Termination**: รับประกัน termination สำหรับ finite structures
5. **Performance**: Hylo ไม่สร้าง intermediate structure

---

*[← BONUS ส่วนที่ 101: Abstract Algebra และ Category Theory](part-101-bonus-fp-algebra.md) | [BONUS ส่วนที่ 103: Optics กับ Monocle →](part-103-bonus-optics.md)*
