# ส่วนที่ 72: Functional Programming Patterns - รูปแบบการเขียนโปรแกรมเชิงฟังก์ชัน

## สารบัญ

1. [Recursion และ Tail-call Optimization](#recursion-tco)
2. [Trampolining เพื่อ Stack Safety](#trampolining)
3. [Continuation-Passing Style (CPS)](#cps)
4. [Church Encoding](#church-encoding)
5. [Zipper Pattern สำหรับ Tree Traversal](#zipper-pattern)
6. [Lenses ด้วย Monocle](#lenses-monocle)
7. [Prisms และ Traversals](#prisms-traversals)
8. [ตัวอย่างการ Refactor สมบูรณ์](#refactoring-example)
9. [สรุป](#summary)

---

## 1. Recursion และ Tail-call Optimization {#recursion-tco}

### Recursion พื้นฐานและปัญหา Stack Overflow

```scala
// build.sbt
libraryDependencies ++= Seq(
  "dev.optics"  %% "monocle-core"  % "3.2.0",
  "dev.optics"  %% "monocle-macro" % "3.2.0",
  "org.typelevel" %% "cats-core"   % "2.10.0",
  "org.typelevel" %% "cats-effect" % "3.5.4"
)
```

```scala
object RecursionDemo:
  
  // Naive recursion - StackOverflow สำหรับ n ใหญ่
  def factorial(n: BigInt): BigInt =
    if n <= 1 then BigInt(1)
    else n * factorial(n - 1)
  
  // Tail-recursive version
  def factorialTR(n: BigInt, acc: BigInt = BigInt(1)): BigInt =
    if n <= 1 then acc
    else factorialTR(n - 1, n * acc)
  
  // Scala annotation เพื่อ verify tail recursion
  import scala.annotation.tailrec
  
  @tailrec
  def factorialAnnotated(n: BigInt, acc: BigInt = BigInt(1)): BigInt =
    if n <= 1 then acc
    else factorialAnnotated(n - 1, n * acc)
  
  // Fibonacci - naive (exponential time)
  def fibNaive(n: Int): Long =
    if n <= 1 then n.toLong
    else fibNaive(n - 1) + fibNaive(n - 2)
  
  // Fibonacci - tail recursive
  @tailrec
  def fibTR(n: Int, a: Long = 0, b: Long = 1): Long =
    if n == 0 then a
    else fibTR(n - 1, b, a + b)
  
  // Fibonacci - memoized
  def fibMemo(n: Int): Long =
    val memo = scala.collection.mutable.Map[Int, Long]()
    def go(k: Int): Long =
      if k <= 1 then k.toLong
      else memo.getOrElseUpdate(k, go(k - 1) + go(k - 2))
    go(n)
  
  // List operations - tail recursive
  @tailrec
  def sumList(list: List[Int], acc: Int = 0): Int = list match
    case Nil     => acc
    case h :: t  => sumList(t, acc + h)
  
  @tailrec
  def reverseList[A](list: List[A], acc: List[A] = Nil): List[A] = list match
    case Nil     => acc
    case h :: t  => reverseList(t, h :: acc)
  
  // Mutual recursion - ต้องใช้ trampoline
  def isEvenNaive(n: Int): Boolean =
    if n == 0 then true
    else isOddNaive(n - 1)
  
  def isOddNaive(n: Int): Boolean =
    if n == 0 then false
    else isEvenNaive(n - 1)
  
  // Tree traversal - tail recursive ด้วย explicit stack
  sealed trait Tree[+A]
  case class Leaf[A](value: A) extends Tree[A]
  case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
  
  def treeSum(tree: Tree[Int]): Int =
    @tailrec
    def go(remaining: List[Tree[Int]], acc: Int): Int = remaining match
      case Nil => acc
      case Leaf(v) :: rest => go(rest, acc + v)
      case Branch(l, r) :: rest => go(l :: r :: rest, acc)
    go(List(tree), 0)
  
  def treeToList[A](tree: Tree[A]): List[A] =
    @tailrec
    def go(remaining: List[Tree[A]], acc: List[A]): List[A] = remaining match
      case Nil => acc.reverse
      case Leaf(v) :: rest => go(rest, v :: acc)
      case Branch(l, r) :: rest => go(l :: r :: rest, acc)
    go(List(tree), Nil)
  
  // ทดสอบ
  def main(args: Array[String]): Unit =
    println(s"factorial(20) = ${factorialAnnotated(20)}")
    println(s"fib(50) = ${fibTR(50)}")
    println(s"sumList = ${sumList((1 to 10000).toList)}")
    
    val tree: Tree[Int] = Branch(
      Branch(Leaf(1), Leaf(2)),
      Branch(Leaf(3), Branch(Leaf(4), Leaf(5)))
    )
    println(s"treeSum = ${treeSum(tree)}")
    println(s"treeToList = ${treeToList(tree)}")
    
    // Performance comparison
    val start1 = System.nanoTime()
    (1 to 1000000).foldLeft(0)(_ + _)
    val iterTime = (System.nanoTime() - start1) / 1000000.0
    
    val start2 = System.nanoTime()
    sumList((1 to 1000000).toList)
    val recTime = (System.nanoTime() - start2) / 1000000.0
    
    println(f"Iterative: $iterTime%.2fms, Tail Recursive: $recTime%.2fms")
```

---

## 2. Trampolining เพื่อ Stack Safety {#trampolining}

### Trampoline Pattern

```scala
object TrampolineDemo:
  
  // Trampoline ADT
  sealed trait Trampoline[+A]:
    final def run: A = Trampoline.run(this)
  
  case class Done[A](value: A) extends Trampoline[A]
  case class More[A](next: () => Trampoline[A]) extends Trampoline[A]
  case class FlatMap[A, B](sub: Trampoline[A], f: A => Trampoline[B]) 
    extends Trampoline[B]
  
  object Trampoline:
    def done[A](a: A): Trampoline[A] = Done(a)
    def more[A](thunk: => Trampoline[A]): Trampoline[A] = More(() => thunk)
    
    @scala.annotation.tailrec
    def run[A](t: Trampoline[A]): A = t match
      case Done(v) => v
      case More(next) => run(next())
      case FlatMap(sub, f) => sub match
        case Done(v) => run(f(v))
        case More(next) => run(FlatMap(next(), f))
        case FlatMap(sub2, g) => 
          run(FlatMap(sub2, (x: Any) => FlatMap(g(x), f)))
    
    extension [A](t: Trampoline[A])
      def flatMap[B](f: A => Trampoline[B]): Trampoline[B] = FlatMap(t, f)
      def map[B](f: A => B): Trampoline[B] = flatMap(a => Done(f(a)))
  
  // Mutual recursion ด้วย Trampoline
  def isEven(n: Int): Trampoline[Boolean] =
    if n == 0 then Trampoline.done(true)
    else Trampoline.more(isOdd(n - 1))
  
  def isOdd(n: Int): Trampoline[Boolean] =
    if n == 0 then Trampoline.done(false)
    else Trampoline.more(isEven(n - 1))
  
  // Fibonacci ด้วย Trampoline
  def fibTrampoline(n: Int): Trampoline[Long] =
    if n <= 1 then Trampoline.done(n.toLong)
    else 
      for
        a <- Trampoline.more(fibTrampoline(n - 1))
        b <- Trampoline.more(fibTrampoline(n - 2))
      yield a + b
  
  // Stack-safe tree fold
  sealed trait Tree[+A]
  case class Leaf[A](value: A) extends Tree[A]
  case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A]
  
  def foldTree[A, B](tree: Tree[A], zero: B)(f: (B, A) => B): Trampoline[B] =
    tree match
      case Leaf(v) => Trampoline.done(f(zero, v))
      case Node(left, right) =>
        for
          leftResult  <- Trampoline.more(foldTree(left, zero)(f))
          rightResult <- Trampoline.more(foldTree(right, leftResult)(f))
        yield rightResult
  
  def main(args: Array[String]): Unit =
    // ทดสอบ mutual recursion ที่ deep มาก
    println(s"isEven(1000000) = ${isEven(1000000).run}")
    println(s"isOdd(999999) = ${isOdd(999999).run}")
    
    // Tree fold
    def buildTree(depth: Int): Tree[Int] =
      if depth == 0 then Leaf(1)
      else Node(buildTree(depth - 1), buildTree(depth - 1))
    
    val deepTree = buildTree(15)  // 2^15 = 32768 leaves
    val sum = foldTree(deepTree, 0)(_ + _).run
    println(s"Sum of deep tree: $sum")  // should be 32768
  
  // cats-effect IO เป็น built-in trampoline
  import cats.effect.IO
  
  def stackSafeRecursion(): IO[Unit] =
    def go(n: Int): IO[Int] =
      if n <= 0 then IO.pure(0)
      else IO.defer(go(n - 1)).map(_ + 1)  // defer = trampoline
    
    go(1000000).map(result => println(s"Count: $result"))
```

---

## 3. Continuation-Passing Style (CPS) {#cps}

### CPS Transformations

```scala
object CPSDemo:
  
  // Direct style
  def addDirect(x: Int, y: Int): Int = x + y
  
  // CPS style
  def addCPS[R](x: Int, y: Int)(k: Int => R): R = k(x + y)
  
  // Factorial ใน CPS
  def factorialCPS[R](n: Int)(k: BigInt => R): R =
    if n <= 0 then k(BigInt(1))
    else factorialCPS(n - 1) { result => k(n * result) }
  
  // Fibonacci ใน CPS
  def fibCPS[R](n: Int)(k: Long => R): R =
    if n <= 1 then k(n.toLong)
    else fibCPS(n - 1) { a =>
      fibCPS(n - 2) { b =>
        k(a + b)
      }
    }
  
  // CPS ด้วย Continuation monad
  case class Cont[R, A](run: (A => R) => R):
    def map[B](f: A => B): Cont[R, B] =
      Cont { k => run(a => k(f(a))) }
    
    def flatMap[B](f: A => Cont[R, B]): Cont[R, B] =
      Cont { k => run(a => f(a).run(k)) }
  
  object Cont:
    def pure[R, A](a: A): Cont[R, A] = Cont { k => k(a) }
    
    def callCC[R, A, B](f: (A => Cont[R, B]) => Cont[R, A]): Cont[R, A] =
      Cont { k => f(a => Cont { _ => k(a) }).run(k) }
  
  // ตัวอย่างการใช้ Cont monad
  def computeWithCont(): Unit =
    // Early exit pattern ด้วย callCC
    def findFirst[R](list: List[Int], pred: Int => Boolean): Cont[R, Option[Int]] =
      Cont.callCC[R, Option[Int], Nothing] { exit =>
        list.foldLeft(Cont.pure[R, Option[Int]](None)) { (acc, elem) =>
          if pred(elem) then exit(Some(elem))
          else acc
        }
      }
    
    val result = findFirst[Option[Int]](List(1, 3, 5, 8, 11), _ % 2 == 0)
      .run(identity)
    println(s"First even: $result")  // Some(8)
    
    // Chaining computations
    val computation = for
      x <- Cont.pure[Int, Int](10)
      y <- Cont.pure[Int, Int](20)
      z = x + y
    yield z * 2
    
    println(s"Result: ${computation.run(identity)}")  // 60
  
  // CPS สำหรับ async simulation
  type Callback[A] = A => Unit
  
  def fetchUser(userId: Int)(callback: Callback[String]): Unit =
    // Simulate async operation
    callback(s"User_$userId")
  
  def fetchOrders(userId: String)(callback: Callback[List[String]]): Unit =
    callback(List(s"Order1_$userId", s"Order2_$userId"))
  
  def processOrders(orders: List[String])(callback: Callback[Int]): Unit =
    callback(orders.length)
  
  // CPS chaining (callback hell)
  def processUserCPS(userId: Int)(done: Callback[String]): Unit =
    fetchUser(userId) { user =>
      fetchOrders(user) { orders =>
        processOrders(orders) { count =>
          done(s"User $user has $count orders")
        }
      }
    }
  
  def main(args: Array[String]): Unit =
    println(addCPS(3, 4)(identity))
    println(factorialCPS(10)(identity))
    println(fibCPS(20)(identity))
    computeWithCont()
    processUserCPS(42)(println)
```

---

## 4. Church Encoding {#church-encoding}

### การ Encode ข้อมูลด้วยฟังก์ชัน

```scala
object ChurchEncodingDemo:
  
  // Church Numerals - ตัวเลขในรูปแบบฟังก์ชัน
  type Church[A] = (A => A) => A => A
  
  // Zero: λf.λx.x
  def zero[A]: Church[A] = f => x => x
  
  // Successor: λn.λf.λx.f(nfx)
  def succ[A](n: Church[A]): Church[A] = f => x => f(n(f)(x))
  
  // สร้างตัวเลข
  def one[A]: Church[A]   = succ(zero)
  def two[A]: Church[A]   = succ(one)
  def three[A]: Church[A] = succ(two)
  
  // แปลง Church numeral เป็น Int
  def churchToInt(n: Church[Int]): Int = n(_ + 1)(0)
  
  // บวก: λm.λn.λf.λx.mf(nfx)
  def add[A](m: Church[A], n: Church[A]): Church[A] = 
    f => x => m(f)(n(f)(x))
  
  // คูณ: λm.λn.λf.m(nf)
  def mult[A](m: Church[A], n: Church[A]): Church[A] = 
    f => m(n(f))
  
  // ยกกำลัง: λm.λn.nm
  def pow[A](base: Church[Church[A]], exp: Church[Church[A] => Church[A]]): Church[A] =
    exp(base)
  
  // Church Booleans
  type ChurchBool[A] = A => A => A
  
  def churchTrue[A]: ChurchBool[A]  = t => f => t
  def churchFalse[A]: ChurchBool[A] = t => f => f
  
  def churchAnd[A](p: ChurchBool[A], q: ChurchBool[A]): ChurchBool[A] =
    t => f => p(q(t)(f))(f)
  
  def churchOr[A](p: ChurchBool[A], q: ChurchBool[A]): ChurchBool[A] =
    t => f => p(t)(q(t)(f))
  
  def churchNot[A](p: ChurchBool[A]): ChurchBool[A] =
    t => f => p(f)(t)
  
  def churchToBool(p: ChurchBool[Boolean]): Boolean = p(true)(false)
  
  // Church Pairs
  type ChurchPair[A, B, R] = (A => B => R) => R
  
  def churchPair[A, B, R](a: A)(b: B): ChurchPair[A, B, R] = 
    f => f(a)(b)
  
  def churchFst[A, B, R](p: ChurchPair[A, B, A]): A = p(a => b => a)
  def churchSnd[A, B, R](p: ChurchPair[A, B, B]): B = p(a => b => b)
  
  // Church Lists
  // Nil: λc.λn.n
  // Cons: λh.λt.λc.λn.c h (t c n)
  type ChurchList[A, R] = (A => R => R) => R => R
  
  def churchNil[A, R]: ChurchList[A, R] = c => n => n
  
  def churchCons[A, R](head: A)(tail: ChurchList[A, R]): ChurchList[A, R] =
    c => n => c(head)(tail(c)(n))
  
  def churchToList[A](l: ChurchList[A, List[A]]): List[A] =
    l(h => t => h :: t)(Nil)
  
  def main(args: Array[String]): Unit =
    // ทดสอบ Church numerals
    println(s"zero = ${churchToInt(zero)}")
    println(s"one = ${churchToInt(one)}")
    println(s"two = ${churchToInt(two)}")
    println(s"three = ${churchToInt(three)}")
    
    val four = add(two, two)
    println(s"2+2 = ${churchToInt(four)}")
    
    val six = mult(two, three)
    println(s"2*3 = ${churchToInt(six)}")
    
    // Church booleans
    println(s"true AND false = ${churchToBool(churchAnd(churchTrue, churchFalse))}")
    println(s"true OR false = ${churchToBool(churchOr(churchTrue, churchFalse))}")
    println(s"NOT true = ${churchToBool(churchNot(churchTrue))}")
    
    // Church lists
    val myList = churchCons(1)(churchCons(2)(churchCons(3)(churchNil)))
    println(s"Church list = ${churchToList(myList)}")
```

---

## 5. Zipper Pattern สำหรับ Tree Traversal {#zipper-pattern}

### Tree Zipper

```scala
object ZipperDemo:
  
  // Tree Definition
  sealed trait Tree[+A]:
    def map[B](f: A => B): Tree[B] = this match
      case Leaf(v)    => Leaf(f(v))
      case Node(l, r) => Node(l.map(f), r.map(f))
  
  case class Leaf[A](value: A)               extends Tree[A]
  case class Node[A](left: Tree[A], right: Tree[A]) extends Tree[A]
  
  // Breadcrumb - บันทึกเส้นทางที่ผ่านมา
  sealed trait Breadcrumb[+A]
  case class LeftCrumb[A](right: Tree[A])  extends Breadcrumb[A]
  case class RightCrumb[A](left: Tree[A])  extends Breadcrumb[A]
  
  // Zipper = (current focus, breadcrumb trail)
  case class TreeZipper[A](focus: Tree[A], trail: List[Breadcrumb[A]]):
    
    // เดินไปซ้าย
    def goLeft: Option[TreeZipper[A]] = focus match
      case Node(l, r) => Some(TreeZipper(l, LeftCrumb(r) :: trail))
      case Leaf(_)    => None
    
    // เดินไปขวา
    def goRight: Option[TreeZipper[A]] = focus match
      case Node(l, r) => Some(TreeZipper(r, RightCrumb(l) :: trail))
      case Leaf(_)    => None
    
    // เดินขึ้น (กลับ)
    def goUp: Option[TreeZipper[A]] = trail match
      case Nil => None
      case LeftCrumb(r)  :: rest => Some(TreeZipper(Node(focus, r), rest))
      case RightCrumb(l) :: rest => Some(TreeZipper(Node(l, focus), rest))
    
    // ไปยัง root
    def top: TreeZipper[A] = trail match
      case Nil => this
      case _   => goUp.get.top
    
    // แก้ไข node ปัจจุบัน
    def modify(f: A => A): TreeZipper[A] = focus match
      case Leaf(v) => this.copy(focus = Leaf(f(v)))
      case Node(l, r) => this.copy(focus = Node(l.map(f), r.map(f)))
    
    // แทนที่ subtree ปัจจุบัน
    def replace(newTree: Tree[A]): TreeZipper[A] =
      this.copy(focus = newTree)
    
    // ดูว่าอยู่ที่ root หรือเปล่า
    def isRoot: Boolean = trail.isEmpty
    
    // ดูว่าอยู่ที่ leaf หรือเปล่า
    def isLeaf: Boolean = focus.isInstanceOf[Leaf[_]]
    
    // ได้ tree กลับมา
    def toTree: Tree[A] = top.focus
  
  object TreeZipper:
    def fromTree[A](tree: Tree[A]): TreeZipper[A] = TreeZipper(tree, Nil)
  
  // List Zipper
  case class ListZipper[A](before: List[A], focus: A, after: List[A]):
    
    def goNext: Option[ListZipper[A]] = after match
      case Nil    => None
      case h :: t => Some(ListZipper(focus :: before, h, t))
    
    def goPrev: Option[ListZipper[A]] = before match
      case Nil    => None
      case h :: t => Some(ListZipper(t, h, focus :: after))
    
    def modify(f: A => A): ListZipper[A] = copy(focus = f(focus))
    
    def insert(elem: A): ListZipper[A] = 
      ListZipper(before, elem, focus :: after)
    
    def delete: Option[ListZipper[A]] = after match
      case h :: t => Some(ListZipper(before, h, t))
      case Nil => before match
        case h :: t => Some(ListZipper(t, h, Nil))
        case Nil => None
    
    def toList: List[A] = before.reverse ++ (focus :: after)
    
    def toStart: ListZipper[A] = before match
      case Nil => this
      case _   => goPrev.get.toStart
    
    def toEnd: ListZipper[A] = after match
      case Nil => this
      case _   => goNext.get.toEnd
  
  object ListZipper:
    def fromList[A](list: List[A]): Option[ListZipper[A]] = list match
      case Nil    => None
      case h :: t => Some(ListZipper(Nil, h, t))
  
  def main(args: Array[String]): Unit =
    // สร้าง tree
    val tree: Tree[Int] = Node(
      Node(Leaf(1), Leaf(2)),
      Node(Leaf(3), Node(Leaf(4), Leaf(5)))
    )
    
    println("=== Tree Zipper Demo ===")
    val zipper = TreeZipper.fromTree(tree)
    
    // เดิน tree
    val result = for
      z1 <- zipper.goRight            // ไปขวา
      z2 <- z1.goLeft                 // ไปซ้าย
      z3 = z2.modify(_ * 10)          // แก้ไข (3 -> 30)
      z4 <- z3.goUp                   // กลับขึ้น
      z5 = z4.toTree                  // ได้ tree กลับ
    yield z5
    
    println(s"Modified tree: $result")
    
    // ใช้ zipper เพื่อ search และ update
    def updateLeaf[A](zipper: TreeZipper[A], pred: A => Boolean, f: A => A): Tree[A] =
      zipper.focus match
        case Leaf(v) if pred(v) => zipper.modify(f).toTree
        case Leaf(_)            => zipper.toTree
        case Node(_, _) =>
          val leftResult = zipper.goLeft.map(z => 
            TreeZipper.fromTree(updateLeaf(z, pred, f))
          )
          val rightResult = zipper.goRight.map(z =>
            TreeZipper.fromTree(updateLeaf(z, pred, f))
          )
          val newLeft = leftResult.map(_.focus).getOrElse(
            zipper.focus.asInstanceOf[Node[A]].left
          )
          val newRight = rightResult.map(_.focus).getOrElse(
            zipper.focus.asInstanceOf[Node[A]].right
          )
          Node(newLeft, newRight)
    
    val updatedTree = updateLeaf(zipper, _ == 3, _ * 100)
    println(s"Tree with 3 -> 300: $updatedTree")
    
    // List Zipper
    println("\n=== List Zipper Demo ===")
    val listZipper = ListZipper.fromList(List(1, 2, 3, 4, 5)).get
    
    val modified = for
      z1 <- listZipper.goNext
      z2 <- z1.goNext
      z3 = z2.modify(_ * 10)  // 3 -> 30
      z4 <- z3.goNext
      z5 = z4.insert(99)      // insert 99 before 4
    yield z5.toList
    
    println(s"Modified list: $modified")  // List(1, 2, 30, 99, 4, 5)
```

---

## 6. Lenses ด้วย Monocle {#lenses-monocle}

### Lens, Optional, Prism

```scala
import monocle.{Lens, Optional, Prism, Traversal}
import monocle.macros.{GenLens, GenPrism}
import monocle.syntax.all.*

object LensDemo:
  
  // Domain model
  case class Address(
    street: String,
    city: String,
    country: String,
    zipCode: String
  )
  
  case class Contact(
    email: String,
    phone: Option[String],
    address: Address
  )
  
  case class Person(
    id: Int,
    firstName: String,
    lastName: String,
    age: Int,
    contact: Contact
  )
  
  case class Company(
    name: String,
    ceo: Person,
    employees: List[Person],
    headquarters: Address
  )
  
  def demonstrateLens(): Unit =
    // สร้าง Lenses ด้วย GenLens macro
    val personContact: Lens[Person, Contact] = GenLens[Person](_.contact)
    val contactAddress: Lens[Contact, Address] = GenLens[Contact](_.address)
    val addressCity: Lens[Address, String] = GenLens[Address](_.city)
    val personAge: Lens[Person, Int] = GenLens[Person](_.age)
    
    // Compose lenses
    val personCity: Lens[Person, String] = 
      personContact.andThen(contactAddress).andThen(addressCity)
    
    val alice = Person(
      id = 1,
      firstName = "Alice",
      lastName = "Johnson",
      age = 30,
      contact = Contact(
        email = "alice@example.com",
        phone = Some("+66-81-234-5678"),
        address = Address("123 Main St", "Bangkok", "Thailand", "10110")
      )
    )
    
    // อ่านด้วย lens
    println(s"City: ${personCity.get(alice)}")         // Bangkok
    println(s"Age: ${personAge.get(alice)}")           // 30
    
    // อัพเดทด้วย lens
    val movedAlice = personCity.replace("Chiang Mai")(alice)
    println(s"New city: ${personCity.get(movedAlice)}") // Chiang Mai
    
    // modify (apply function)
    val olderAlice = personAge.modify(_ + 1)(alice)
    println(s"New age: ${personAge.get(olderAlice)}")   // 31
    
    // ใช้ focus syntax
    val companyLens = Company(
      name = "TechCorp",
      ceo = alice,
      employees = List(alice),
      headquarters = alice.contact.address
    )
    
    // Fluent lens composition
    val updatedCompany = companyLens
      .focus(_.ceo.contact.address.city).replace("Phuket")
      .focus(_.ceo.age).modify(_ + 5)
      .focus(_.headquarters.zipCode).replace("83000")
    
    println(s"CEO city: ${updatedCompany.ceo.contact.address.city}")  // Phuket
    println(s"CEO age: ${updatedCompany.ceo.age}")                     // 35
  
  def demonstrateOptionalAndTraversal(): Unit =
    case class Config(
      host: String,
      port: Int,
      credentials: Option[Credentials],
      features: List[Feature]
    )
    
    case class Credentials(username: String, password: String)
    case class Feature(name: String, enabled: Boolean, priority: Int)
    
    // Optional - lens สำหรับค่าที่อาจไม่มี
    val configCredentials = monocle.Optional[Config, Credentials](
      c => c.credentials
    )(creds => c => c.copy(credentials = Some(creds)))
    
    val credUsername = GenLens[Credentials](_.username)
    
    val configUsername = configCredentials.andThen(credUsername)
    
    val config = Config(
      host = "localhost",
      port = 8080,
      credentials = Some(Credentials("admin", "secret")),
      features = List(
        Feature("darkMode", enabled = true, priority = 1),
        Feature("beta", enabled = false, priority = 2),
        Feature("analytics", enabled = true, priority = 3)
      )
    )
    
    // Optional: safe get (returns Option)
    println(s"Username: ${configUsername.getOption(config)}")  // Some(admin)
    
    val noCredConfig = config.copy(credentials = None)
    println(s"No creds: ${configUsername.getOption(noCredConfig)}")  // None
    
    // Traversal - lens สำหรับหลาย elements
    val featuresTraversal: Traversal[Config, Feature] = 
      monocle.Traversal.fromTraverse[List, Feature]
        .compose(monocle.Lens[Config, List[Feature]](_.features)(fs => c => c.copy(features = fs)))
    
    // แก้ไขทุก features
    val allEnabled = config
      .focus(_.features).each.focus(_.enabled).replace(true)
    
    println(s"All enabled: ${allEnabled.features.map(_.enabled)}")
    
    // filter และ modify
    val highPriorityEnabled = config.focus(_.features)
      .each
      .filterIndex[Int](_ < 2)  // index 0 และ 1
      .focus(_.enabled)
      .replace(false)
    
    println(s"Features: ${highPriorityEnabled.features}")
  
  def main(args: Array[String]): Unit =
    demonstrateLens()
    demonstrateOptionalAndTraversal()
```

---

## 7. Prisms และ Traversals {#prisms-traversals}

```scala
import monocle.{Prism, Traversal}
import monocle.syntax.all.*

object PrismsAndTraversalsDemo:
  
  // ADT สำหรับ demo
  sealed trait Json
  case class JNull()                          extends Json
  case class JBool(value: Boolean)            extends Json
  case class JNum(value: Double)              extends Json
  case class JStr(value: String)              extends Json
  case class JArr(values: List[Json])         extends Json
  case class JObj(fields: Map[String, Json])  extends Json
  
  // Prisms สำหรับ JSON ADT
  val jNull: Prism[Json, Unit] = Prism[Json, Unit] {
    case JNull() => Some(())
    case _       => None
  }(_ => JNull())
  
  val jBool: Prism[Json, Boolean] = Prism[Json, Boolean] {
    case JBool(b) => Some(b)
    case _        => None
  }(JBool.apply)
  
  val jNum: Prism[Json, Double] = Prism[Json, Double] {
    case JNum(n) => Some(n)
    case _       => None
  }(JNum.apply)
  
  val jStr: Prism[Json, String] = Prism[Json, String] {
    case JStr(s) => Some(s)
    case _       => None
  }(JStr.apply)
  
  val jArr: Prism[Json, List[Json]] = Prism[Json, List[Json]] {
    case JArr(vs) => Some(vs)
    case _        => None
  }(JArr.apply)
  
  val jObj: Prism[Json, Map[String, Json]] = Prism[Json, Map[String, Json]] {
    case JObj(fs) => Some(fs)
    case _        => None
  }(JObj.apply)
  
  def demonstratePrisms(): Unit =
    val json: Json = JObj(Map(
      "name"   -> JStr("Alice"),
      "age"    -> JNum(30.0),
      "active" -> JBool(true),
      "scores" -> JArr(List(JNum(95), JNum(87), JNum(92)))
    ))
    
    // ใช้ Prism
    println(s"Is string: ${jStr.getOption(JStr("hello"))}")  // Some(hello)
    println(s"Is num: ${jNum.getOption(JStr("hello"))}")      // None
    
    // Compose Prisms ด้วย Optional
    val objFields = jObj.andThen(
      monocle.Lens[Map[String, Json], Option[Json]](
        _.get("name")
      )(opt => m => opt.fold(m - "name")(v => m + ("name" -> v)))
        .andThen(monocle.Optional.some[Json])
        .andThen(jStr)
    )
    
    // ทำงานกับ JSON path
    def getField(json: Json, key: String): Option[Json] = json match
      case JObj(fields) => fields.get(key)
      case _            => None
    
    def setField(json: Json, key: String, value: Json): Json = json match
      case JObj(fields) => JObj(fields + (key -> value))
      case _            => json
    
    // JSON path traversal
    def deepGet(json: Json, path: List[String]): Option[Json] = path match
      case Nil     => Some(json)
      case h :: t  => getField(json, h).flatMap(child => deepGet(child, t))
    
    println(deepGet(json, List("name")))    // Some(JStr(Alice))
    println(deepGet(json, List("scores")))  // Some(JArr(...))
    println(deepGet(json, List("missing"))) // None
  
  // Traversal สำหรับ transform nested structures
  def demonstrateTraversals(): Unit =
    import cats.implicits.*
    
    val data = List(
      JObj(Map("value" -> JNum(10.0), "label" -> JStr("A"))),
      JObj(Map("value" -> JNum(20.0), "label" -> JStr("B"))),
      JObj(Map("value" -> JNum(30.0), "label" -> JStr("C")))
    )
    
    // Traversal ผ่านทุก elements ใน list
    val listTraversal = Traversal.fromTraverse[List, Json]
    
    // นับทุก JNum ใน nested structure
    def countNums(json: Json): Int = json match
      case JNum(_)    => 1
      case JArr(vs)   => vs.map(countNums).sum
      case JObj(fs)   => fs.values.map(countNums).sum
      case _          => 0
    
    // เพิ่มค่า numeric ทั้งหมดด้วย 100
    def addToNums(json: Json, delta: Double): Json = json match
      case JNum(n)    => JNum(n + delta)
      case JArr(vs)   => JArr(vs.map(addToNums(_, delta)))
      case JObj(fs)   => JObj(fs.map { case (k, v) => k -> addToNums(v, delta) })
      case other      => other
    
    val dataJson = JArr(data)
    println(s"Num count: ${countNums(dataJson)}")  // 3
    println(s"After +100: ${addToNums(dataJson, 100)}")
    
    // ใช้ Optics ที่ซับซ้อนขึ้น
    val numericValues: Traversal[Json, Double] = new Traversal[Json, Double]:
      def modifyA[F[_]: cats.Applicative](f: Double => F[Double])(json: Json): F[Json] =
        json match
          case JNum(n) => cats.Applicative[F].map(f(n))(JNum.apply)
          case JArr(vs) =>
            cats.Applicative[F].map(
              vs.traverse(v => modifyA(f)(v))
            )(JArr.apply)
          case JObj(fs) =>
            cats.Applicative[F].map(
              fs.toList.traverse { case (k, v) => 
                cats.Applicative[F].map(modifyA(f)(v))(k -> _)
              }.map(_.toMap)
            )(JObj.apply)
          case other => cats.Applicative[F].pure(other)
    
    // ตรวจสอบค่าทั้งหมด
    val allNums = numericValues.getAll(dataJson)
    println(s"All nums: $allNums")  // List(10.0, 20.0, 30.0)
    
    // Multiply all nums by 2
    val doubled = numericValues.modify(_ * 2)(dataJson)
    println(s"Doubled: ${numericValues.getAll(doubled)}")  // List(20.0, 40.0, 60.0)
  
  def main(args: Array[String]): Unit =
    demonstratePrisms()
    demonstrateTraversals()
```

---

## 8. ตัวอย่างการ Refactor สมบูรณ์ {#refactoring-example}

### จาก OOP สู่ Functional Style

```scala
// === Before: Imperative/OOP Style ===
object BeforeRefactoring:
  
  import scala.collection.mutable
  
  class OrderProcessor:
    private val orders = mutable.ListBuffer[Map[String, Any]]()
    private val errors = mutable.ListBuffer[String]()
    
    def processOrder(data: Map[String, Any]): Option[Map[String, Any]] =
      // Mutation, mutable state, exceptions
      try
        val id = data("id").asInstanceOf[String]
        val amount = data("amount").asInstanceOf[Double]
        val items = data("items").asInstanceOf[List[Map[String, Any]]]
        
        if amount <= 0 then
          errors += s"Invalid amount for order $id"
          return None
        
        if items.isEmpty then
          errors += s"No items in order $id"
          return None
        
        var total = 0.0
        val processedItems = mutable.ListBuffer[Map[String, Any]]()
        
        for item <- items do
          val price = item("price").asInstanceOf[Double]
          val qty = item("quantity").asInstanceOf[Int]
          val itemTotal = price * qty
          total += itemTotal
          processedItems += Map(
            "name" -> item("name"),
            "total" -> itemTotal
          )
        
        val result = Map(
          "id" -> id,
          "total" -> total,
          "items" -> processedItems.toList,
          "status" -> "processed"
        )
        orders += result
        Some(result)
      catch
        case e: Exception =>
          errors += s"Error: ${e.getMessage}"
          None
    
    def getErrors: List[String] = errors.toList
    def getOrders: List[Map[String, Any]] = orders.toList

// === After: Functional Style ===
object AfterRefactoring:
  
  import cats.data.{EitherT, ValidatedNel}
  import cats.implicits.*
  
  // Pure domain types
  case class OrderId(value: String) extends AnyVal
  case class Money(amount: BigDecimal) extends AnyVal:
    def +(other: Money): Money = Money(amount + other.amount)
    def *(qty: Quantity): Money = Money(amount * qty.value)
  
  case class Quantity(value: Int) extends AnyVal
  case class ProductName(value: String) extends AnyVal
  
  case class OrderItem(
    name: ProductName,
    price: Money,
    quantity: Quantity
  ):
    def total: Money = price * quantity
  
  case class Order(
    id: OrderId,
    items: List[OrderItem]
  ):
    def total: Money = items.map(_.total).reduce(_ + _)
  
  case class ProcessedOrder(
    id: OrderId,
    items: List[OrderItem],
    total: Money,
    status: String
  )
  
  // Error types
  sealed trait OrderError
  case class InvalidAmount(msg: String)  extends OrderError
  case class EmptyOrder(id: OrderId)     extends OrderError
  case class ParseError(msg: String)     extends OrderError
  case class ValidationError(errors: List[OrderError]) extends OrderError
  
  // Validation
  def validateOrder(order: Order): ValidatedNel[OrderError, Order] =
    val validItems = order.items.toNel
      .toValidNel(EmptyOrder(order.id): OrderError)
      .void
    
    val validAmounts = order.items.traverse { item =>
      if item.price.amount > 0 && item.quantity.value > 0 then
        item.validNel[OrderError]
      else
        (InvalidAmount(s"Invalid item: ${item.name}"): OrderError).invalidNel
    }.void
    
    (validItems, validAmounts).mapN { (_, _) => order }
  
  // Pure processing function
  def processOrder(order: Order): Either[OrderError, ProcessedOrder] =
    validateOrder(order).toEither
      .leftMap(errors => ValidationError(errors.toList))
      .map { validOrder =>
        ProcessedOrder(
          id = validOrder.id,
          items = validOrder.items,
          total = validOrder.total,
          status = "processed"
        )
      }
  
  // Process multiple orders
  def processOrders(orders: List[Order]): (List[OrderError], List[ProcessedOrder]) =
    orders.partitionMap(processOrder)
  
  // กำหนด transformations ด้วย Lenses และ Prisms
  import monocle.syntax.all.*
  
  def applyDiscount(order: ProcessedOrder, discountRate: BigDecimal): ProcessedOrder =
    val discount = Money(order.total.amount * discountRate)
    val newTotal = Money(order.total.amount - discount.amount)
    order.copy(total = newTotal)
  
  // Main program
  def runExample(): Unit =
    val orders = List(
      Order(
        OrderId("O001"),
        List(
          OrderItem(ProductName("Laptop"), Money(999.0), Quantity(2)),
          OrderItem(ProductName("Mouse"), Money(29.0), Quantity(3))
        )
      ),
      Order(
        OrderId("O002"),
        List.empty  // Invalid: empty items
      ),
      Order(
        OrderId("O003"),
        List(
          OrderItem(ProductName("Keyboard"), Money(-10.0), Quantity(1))  // Invalid: negative price
        )
      )
    )
    
    val (errors, processed) = processOrders(orders)
    
    println("=== Processed Orders ===")
    processed.foreach { order =>
      val withDiscount = applyDiscount(order, BigDecimal("0.10"))
      println(f"${order.id.value}: total=${order.total.amount}%.2f, " +
              f"after-10%%=${withDiscount.total.amount}%.2f")
    }
    
    println("\n=== Errors ===")
    errors.foreach(println)
  
  def main(args: Array[String]): Unit =
    runExample()
    
    // เปรียบเทียบความอ่านง่าย
    println("\n=== Functional Pipeline ===")
    val pipeline = List(
      Order(OrderId("O004"), List(OrderItem(ProductName("Book"), Money(25.0), Quantity(5)))),
      Order(OrderId("O005"), List(OrderItem(ProductName("Pen"), Money(5.0), Quantity(10))))
    )
    .map(processOrder)
    .collect { case Right(order) => order }
    .map(applyDiscount(_, BigDecimal("0.05")))
    .sortBy(-_.total.amount)
    
    pipeline.foreach { order =>
      println(s"${order.id.value}: ${order.total.amount}")
    }
```

---

## 9. สรุป {#summary}

### Patterns ที่ได้เรียนในบทนี้

1. **Tail Recursion**: ใช้ `@tailrec` annotation เพื่อป้องกัน StackOverflow
2. **Trampolining**: แก้ปัญหา mutual recursion และ deep recursion
3. **CPS**: เข้าใจ continuation และใช้ `callCC` สำหรับ early exit
4. **Church Encoding**: encode data structures ด้วยฟังก์ชัน
5. **Zipper**: navigate และ update immutable structures อย่างมีประสิทธิภาพ
6. **Lenses**: compose optics สำหรับ nested data manipulation
7. **Prisms**: optics สำหรับ sum types (ADTs)
8. **Refactoring**: เปลี่ยนจาก imperative เป็น functional style

### เมื่อไหร่ควรใช้อะไร

```scala
// Recursion + @tailrec: สำหรับ simple recursive algorithms
@tailrec def sum(list: List[Int], acc: Int = 0): Int = ...

// Trampoline: mutual recursion หรือ recursion ที่ลึกมาก
def isEven(n: Int): Trampoline[Boolean] = ...

// Lens: nested immutable data updates
val newCity = personCity.replace("Bangkok")(person)

// Prism: pattern matching บน ADTs
val maybeNum = jNum.getOption(jsonValue)

// Traversal: bulk operations บน collections ใน structures
val doubled = numericValues.modify(_ * 2)(json)
```

---

*[← Part 71: Spark Structured Streaming](part-71-spark-streaming.md) | [Part 73: Reactive Architecture →](part-73-reactive-architecture.md)*
