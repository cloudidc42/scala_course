# Part 31: Scala Macros และ Metaprogramming

## สารบัญ
1. [Macros Overview](#macros-overview)
2. [Inline Methods](#inline-methods)
3. [Compile-time Operations](#compile-time-operations)
4. [Quotes and Splices](#quotes-and-splices)
5. [Macro Examples](#macro-examples)

---

## Macros Overview

### แนวคิด Metaprogramming

```
Metaprogramming: code that manipulates code at compile time

Scala 3 Metaprogramming:
1. inline:     compile-time inlining
2. Quotes/Splices: typed code manipulation
3. Mirrors:    structural reflection
4. Macros:     arbitrary code generation

ประโยชน์:
- Zero-overhead abstractions
- Compile-time validation
- Auto-deriving type class instances
- DSL creation
```

---

## Inline Methods

### Basic Inline

```scala
// inline method: code expanded at call site
inline def square(n: Int): Int = n * n
inline def cube(n: Int): Int = n * n * n

// ทำงานเหมือน macro - code replaced at compile time
// square(5) compiles to: 5 * 5

// Inline กับ conditions
inline def log(msg: String): Unit =
  inline if DEBUG then println(s"[DEBUG] $msg")

// Inline parameter
inline def twice[A](inline f: => A): (A, A) = (f, f)

var count = 0
val (a, b) = twice { count += 1; count }
println(s"a=$a, b=$b, count=$count")  // a=1, b=2, count=2
// f is inlined, so called twice
```

### Inline Match

```scala
// Type-level matching at compile time
inline def typeDescription[T]: String =
  inline erasedValue[T] match
    case _: Int    => "integer"
    case _: String => "string"
    case _: Double => "double"
    case _         => "unknown"

println(typeDescription[Int])     // integer
println(typeDescription[String])  // string
println(typeDescription[Boolean]) // unknown

// Compile-time size checking
inline def head[T <: Tuple]: Any =
  inline erasedValue[T] match
    case _: (h *: t) => compiletime.summonInline[h.type]
    case _: EmptyTuple => compiletime.error("Cannot get head of empty tuple")
```

---

## Compile-time Operations

### compiletime Package

```scala
import scala.compiletime.*

// summonInline: summon given at compile time
def showAll[Tup <: Tuple]: List[String] =
  inline erasedValue[Tup] match
    case _: EmptyTuple => Nil
    case _: (h *: t) =>
      summonInline[Show[h]].show(???) :: showAll[t]

// constValue: extract compile-time constant
inline def intValue[N <: Int]: Int = constValue[N]
println(intValue[42])  // 42

// error: compile-time error message
inline def assertNonNegative(n: Int): Unit =
  inline if n < 0 then error(s"Expected non-negative, got $n")

// ops.int: compile-time integer arithmetic
import compiletime.ops.int.*
type Plus[A <: Int, B <: Int] = A + B
type Max2 = Max[3, 5]  // = 5 at type level
```

---

## Quotes and Splices

### Basic Quotes

```scala
import scala.quoted.*

// Quote: turn expression into AST representation
def printAst(using Quotes)(expr: Expr[Int]): Expr[String] =
  Expr(expr.show)

// Simple macro
inline def showExpr(inline expr: Int): String =
  ${ showExprImpl('expr) }

def showExprImpl(expr: Expr[Int])(using Quotes): Expr[String] =
  val code = expr.show
  Expr(s"Expression: $code = ${???}")  // simplified

// Power macro: generates code
inline def power(base: Int, inline exp: Int): Int =
  ${ powerImpl('base, exp) }

def powerImpl(base: Expr[Int], exp: Int)(using Quotes): Expr[Int] =
  import quotes.reflect.*
  if exp == 0 then '{ 1 }
  else if exp % 2 == 0 then '{
    val b = $base
    b * b * ${ powerImpl(base, exp - 2) }
  }
  else '{ $base * ${ powerImpl(base, exp - 1) } }

println(power(2, 10))  // 1024 (computed via repeated squaring)
```

---

## Macro Examples

### Auto-derive toString

```scala
import scala.quoted.*
import scala.deriving.*

inline def autoToString[T](using m: Mirror.ProductOf[T], t: T): String =
  ${ autoToStringImpl[T]('t) }

def autoToStringImpl[T: Type](t: Expr[T])(using Quotes): Expr[String] =
  import quotes.reflect.*
  val sym = TypeRepr.of[T].typeSymbol
  val fields = sym.caseFields

  val fieldExprs = fields.map { field =>
    val name = field.name
    val value = Select(t.asTerm, field).asExpr
    '{ ${ Expr(name) } + "=" + $value.toString }
  }

  fieldExprs.foldRight('{ "" }) { (expr, acc) =>
    '{ $expr + (if $acc.isEmpty then "" else ", " + $acc) }
  }
```

### Compile-time Validation

```scala
import scala.quoted.*

// Validate regex at compile time
inline def regex(inline pattern: String): java.util.regex.Pattern =
  ${ regexImpl('pattern) }

def regexImpl(pattern: Expr[String])(using Quotes): Expr[java.util.regex.Pattern] =
  import quotes.reflect.*
  val patternStr = pattern.valueOrAbort
  try
    java.util.regex.Pattern.compile(patternStr)  // validate
    '{ java.util.regex.Pattern.compile($pattern) }
  catch case e: java.util.regex.PatternSyntaxException =>
    report.errorAndAbort(s"Invalid regex: ${e.getMessage}")

// ใช้งาน
val emailRegex = regex("""^[^@]+@[^@]+\.[^@]+$""")  // validated at compile time
// regex("""[invalid""")  // compile error!
```

### Structural Logging

```scala
import scala.quoted.*

inline def log[T](inline value: T): T =
  ${ logImpl('value) }

def logImpl[T: Type](value: Expr[T])(using Quotes): Expr[T] =
  import quotes.reflect.*
  val name = value.show
  '{
    val result = $value
    println(s"${ ${ Expr(name) } } = $result")
    result
  }

// ใช้งาน
val x = log(1 + 2)        // prints: 1 + 2 = 3
val y = log(x * x)        // prints: x * x = 9
val z = log(List(1,2,3))  // prints: List(1,2,3) = List(1,2,3)
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ inline methods: compile-time inlining
- ✅ inline match: type-level pattern matching
- ✅ compiletime package: summonInline, constValue, error
- ✅ Quotes and Splices: typed AST manipulation
- ✅ Macro examples: auto toString, regex validation, structural logging

---

*[← Part 30: ZIO](part-30-zio.md) | [Part 32: Type System Advanced →](part-32-type-system-advanced.md)*
