# Part 08: การจัดการ String

## สารบัญ
1. [String Operations เชิงลึก](#string-operations)
2. [Regular Expressions](#regular-expressions)
3. [String Parsing](#string-parsing)
4. [Text Processing Patterns](#text-processing-patterns)
5. [Internationalization (i18n)](#internationalization)
6. [StringBuilder และ Performance](#stringbuilder)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## String Operations เชิงลึก

### String เป็น Sequence of Chars

```scala
val str = "Hello, Scala!"

// String เป็น Seq[Char] - ทำ operations ได้เหมือน collection
println(str.length)          // 13
println(str.head)            // H
println(str.last)            // !
println(str(0))              // H
println(str.charAt(0))       // H

// Iterate
for ch <- str do print(s"$ch-")
// H-e-l-l-o-,-  -S-c-a-l-a-!-

// map/filter ทำงานกับ String
val upperLetters = str.filter(_.isLetter).map(_.toUpper)
println(upperLetters)  // HELLOSCALA

// count
println(str.count(_.isLetter))     // 10
println(str.count(_.isUpperCase))  // 2 (H, S)
```

### String Methods ครบถ้วน

```scala
val s = "  Hello, World!  "

// Whitespace
println(s.trim)             // "Hello, World!"
println(s.strip)            // "Hello, World!" (Unicode-aware)
println(s.stripLeading)     // "Hello, World!  "
println(s.stripTrailing)    // "  Hello, World!"

// Case
println("hello".toUpperCase)       // HELLO
println("HELLO".toLowerCase)       // hello
println("hello world".capitalize)  // Hello world (only first char)

// Check
println("Hello".startsWith("He"))  // true
println("Hello".endsWith("lo"))    // true
println("Hello".contains("ell"))   // true
println("".isEmpty)                // true
println("  ".isBlank)              // true (Java 11+)
println("abc".forall(_.isLetter))  // true
println("123".forall(_.isDigit))   // true
println("abc123".exists(_.isDigit)) // true

// Search
val text = "Hello, World! Hello, Scala!"
println(text.indexOf("Hello"))          // 0
println(text.lastIndexOf("Hello"))      // 14
println(text.indexOf("Hello", 5))       // 14 (start from index 5)
println(text.indexOfSlice("World"))     // 7
println(text.count('l' == _))           // 5

// Comparison
println("abc".compareTo("abd"))         // negative (-1)
println("abc".compareToIgnoreCase("ABC"))  // 0
println("abc" < "abd")                  // true (lexicographic)

// Padding
println("42".padTo(5, ' '))            // "42   "
println("42".padTo(5, '0'))            // "42000"
println("hello".padTo(3, ' '))         // "hello" (no truncation)

// Repeat
println("abc" * 3)                      // abcabcabc
println("=-" * 20)                      // =-=-=-=-=-=-...
```

### String Splitting

```scala
val csv = "Alice,30,Engineer,Bangkok"
val tsv = "Alice\t30\tEngineer\tBangkok"
val multiSpace = "Alice   30   Engineer   Bangkok"

// split ด้วย literal
println(csv.split(",").toList)
// List(Alice, 30, Engineer, Bangkok)

// split ด้วย regex
println(multiSpace.split("\\s+").toList)
// List(Alice, 30, Engineer, Bangkok)

// split กับ limit
println("a:b:c:d:e".split(":", 3).toList)
// List(a, b, c:d:e) (max 3 parts)

// splitAt (index)
val (first, rest) = "Hello, World!".splitAt(5)
println(first)  // Hello
println(rest)   // , World!

// partition
val (digits, nonDigits) = "abc123def456".partition(_.isDigit)
println(digits)     // 123456
println(nonDigits)  // abcdef

// lines (split by newlines)
val multiline = "line1\nline2\nline3"
println(multiline.linesIterator.toList)
// List(line1, line2, line3)
```

### String Joining

```scala
val parts = List("Hello", "World", "Scala")

// mkString
println(parts.mkString)          // HelloWorldScala
println(parts.mkString(" "))     // Hello World Scala
println(parts.mkString(", "))    // Hello, World, Scala
println(parts.mkString("[", ", ", "]"))  // [Hello, World, Scala]

// String.join (Java 8+)
println(String.join(", ", parts*))  // Hello, World, Scala
println(String.join("-", "a", "b", "c"))  // a-b-c

// Array join
val arr = Array("x", "y", "z")
println(arr.mkString("+"))  // x+y+z
```

---

## Regular Expressions

### Regex Basics

```scala
import scala.util.matching.Regex

// สร้าง Regex
val emailPattern = """[\w.]+@[\w.]+\.[a-z]{2,}""".r
val phonePattern = """\d{3}-\d{3}-\d{4}""".r
val ipPattern = """(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})""".r

// Test match
val email = "alice@example.com"
println(emailPattern.matches(email))  // true

val badEmail = "not-an-email"
println(emailPattern.matches(badEmail))  // false
```

### Finding Matches

```scala
val text = "Contact us at alice@example.com or bob@test.org"
val emailPattern = """[\w.]+@[\w.]+\.[a-z]{2,}""".r

// findFirstIn: หา match แรก
emailPattern.findFirstIn(text) match
  case Some(email) => println(s"Found: $email")
  case None => println("No email found")
// Found: alice@example.com

// findAllIn: หาทุก match
val emails = emailPattern.findAllIn(text).toList
println(emails)  // List(alice@example.com, bob@test.org)

// findFirstMatchIn: match object พร้อม position
emailPattern.findFirstMatchIn(text) match
  case Some(m) =>
    println(s"Match: ${m.group(0)}")
    println(s"Start: ${m.start}, End: ${m.end}")
  case None => ()

// findAllMatchIn: ทุก match objects
for m <- emailPattern.findAllMatchIn(text) do
  println(s"Found '${m.group(0)}' at position ${m.start}")
```

### Capture Groups

```scala
// Groups ด้วย ()
val datePattern = """(\d{4})-(\d{2})-(\d{2})""".r

val date = "2024-01-15"
date match
  case datePattern(year, month, day) =>
    println(s"Year: $year, Month: $month, Day: $day")
  case _ =>
    println("Not a date")

// Named groups
val namedDate = """(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})""".r

namedDate.findFirstMatchIn("Today is 2024-01-15") match
  case Some(m) =>
    println(s"Year: ${m.group("year")}")
    println(s"Month: ${m.group("month")}")
    println(s"Day: ${m.group("day")}")
  case None => ()

// Extracting multiple groups
val logPattern = """(\d{4}-\d{2}-\d{2}) (\d{2}:\d{2}:\d{2}) \[(\w+)\] (.+)""".r

val logLine = "2024-01-15 10:30:45 [ERROR] Connection failed"
logLine match
  case logPattern(date, time, level, message) =>
    println(s"Date: $date")
    println(s"Time: $time")
    println(s"Level: $level")
    println(s"Message: $message")
  case _ => println("Invalid log format")
```

### Replacing with Regex

```scala
val text = "Hello World 123 Foo 456"

// replaceAll: replace ทุก match
val noNumbers = text.replaceAll("""\d+""", "#")
println(noNumbers)  // Hello World # Foo #

// replaceFirst: replace match แรก
val firstReplaced = text.replaceFirst("""\d+""", "#")
println(firstReplaced)  // Hello World # Foo 456

// Regex.replaceAllIn
val pattern = """\d+""".r
val result = pattern.replaceAllIn(text, m => s"[${m.group(0)}]")
println(result)  // Hello World [123] Foo [456]

// replaceAllIn กับ function
val words = "hello world scala"
val capitalized = """(\w+)""".r.replaceAllIn(words, m => m.group(1).capitalize)
println(capitalized)  // Hello World Scala

// replaceFirstIn
val first = """(\w+)""".r.replaceFirstIn(words, m => m.group(1).toUpperCase)
println(first)  // HELLO world scala
```

### Common Regex Patterns

```scala
object RegexPatterns:
  val Email = """^[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}$""".r
  val Phone = """^\+?[\d\s\-()]{10,}$""".r
  val Url = """^https?://[\w\-.]+(:\d+)?(/[\w\-./?%&=]*)?$""".r
  val IPv4 = """^(\d{1,3}\.){3}\d{1,3}$""".r
  val Date = """^\d{4}-\d{2}-\d{2}$""".r
  val CreditCard = """^\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}$""".r
  val ZipCode = """^\d{5}(-\d{4})?$""".r
  val HexColor = """^#[0-9A-Fa-f]{6}$""".r
  val Username = """^[a-zA-Z0-9_]{3,20}$""".r
  val Password = """^(?=.*[A-Z])(?=.*[a-z])(?=.*\d).{8,}$""".r

def validate(value: String, pattern: scala.util.matching.Regex): Boolean =
  pattern.matches(value)

// ทดสอบ
println(validate("alice@example.com", RegexPatterns.Email))  // true
println(validate("not-email", RegexPatterns.Email))          // false
println(validate("#FF5733", RegexPatterns.HexColor))         // true
println(validate("2024-01-15", RegexPatterns.Date))          // true
```

---

## String Parsing

### Parsing ประเภทข้อมูล

```scala
import scala.util.{Try, Success, Failure}

// Safe parsing
def parseInt(s: String): Option[Int] = s.toIntOption
def parseDouble(s: String): Option[Double] = s.toDoubleOption
def parseLong(s: String): Option[Long] = s.toLongOption
def parseBoolean(s: String): Option[Boolean] =
  s.toLowerCase match
    case "true" | "yes" | "1" => Some(true)
    case "false" | "no" | "0" => Some(false)
    case _ => None

println(parseInt("42"))     // Some(42)
println(parseInt("abc"))    // None
println(parseDouble("3.14")) // Some(3.14)
println(parseBoolean("yes")) // Some(true)
println(parseBoolean("maybe")) // None

// Try-based parsing
def parseIntTry(s: String): Try[Int] = Try(s.toInt)
def parseDoubleTry(s: String): Try[Double] = Try(s.toDouble)

parseIntTry("42") match
  case Success(n) => println(s"Parsed: $n")
  case Failure(e) => println(s"Error: ${e.getMessage}")
```

### CSV Parsing

```scala
case class Person(name: String, age: Int, city: String)

object CSVParser:
  def parseLine(line: String, delimiter: String = ","): List[String] =
    line.split(delimiter).map(_.trim).toList

  def parsePerson(line: String): Option[Person] =
    parseLine(line) match
      case List(name, ageStr, city) =>
        ageStr.toIntOption.map(age => Person(name, age, city))
      case _ => None

  def parseCSV(content: String): List[Person] =
    content.linesIterator
      .drop(1)  // skip header
      .flatMap(parsePerson)
      .toList

// ทดสอบ
val csv = """name,age,city
Alice,30,Bangkok
Bob,25,Chiang Mai
Charlie,abc,Phuket
Diana,28,Bangkok"""

val people = CSVParser.parseCSV(csv)
people.foreach(println)
// Person(Alice,30,Bangkok)
// Person(Bob,25,Chiang Mai)
// Person(Diana,28,Bangkok)  (Charlie skip เพราะ age invalid)
```

### JSON-like Parsing (ไม่ใช้ library)

```scala
// Simple key=value parsing
def parseConfig(content: String): Map[String, String] =
  content.linesIterator
    .map(_.trim)
    .filterNot(line => line.isEmpty || line.startsWith("#"))
    .flatMap { line =>
      line.split("=", 2) match
        case Array(key, value) => Some(key.trim -> value.trim)
        case _ => None
    }
    .toMap

val configContent = """
# Database config
db.host = localhost
db.port = 5432
db.name = myapp

# App config
app.name = MyApp
app.debug = true
"""

val config = parseConfig(configContent)
println(config)
// Map(db.host -> localhost, db.port -> 5432, ...)
```

### Query String Parsing

```scala
def parseQueryString(query: String): Map[String, List[String]] =
  if query.isEmpty then Map.empty
  else
    query.split("&")
      .toList
      .map(param =>
        param.split("=", 2) match
          case Array(key, value) => (
            java.net.URLDecoder.decode(key, "UTF-8"),
            java.net.URLDecoder.decode(value, "UTF-8")
          )
          case Array(key) => (key, "")
          case _ => ("", "")
      )
      .filter(_._1.nonEmpty)
      .groupBy(_._1)
      .view.mapValues(_.map(_._2))
      .toMap

val query = "name=Alice&age=30&tags=scala&tags=java&city=Bangkok%20Thailand"
val params = parseQueryString(query)

params.foreach { case (key, values) =>
  println(s"$key: ${values.mkString(", ")}")
}
```

---

## Text Processing Patterns

### Word Tokenization

```scala
object Tokenizer:
  def tokenize(text: String): List[String] =
    text.toLowerCase
      .replaceAll("[^a-z0-9\\s]", " ")
      .split("\\s+")
      .filter(_.nonEmpty)
      .toList

  def wordFrequency(text: String): Map[String, Int] =
    tokenize(text)
      .groupBy(identity)
      .view.mapValues(_.length)
      .toMap

  def topN(text: String, n: Int): List[(String, Int)] =
    wordFrequency(text)
      .toList
      .sortBy(-_._2)
      .take(n)

// ทดสอบ
val article = """
Scala is a powerful programming language. Scala runs on the JVM.
The language combines object-oriented and functional programming.
Many companies use Scala for big data and backend services.
"""

println("Top 5 words:")
Tokenizer.topN(article, 5).foreach { case (word, count) =>
  println(f"  $word%-20s: $count")
}
```

### Template Engine

```scala
object SimpleTemplate:
  def render(template: String, variables: Map[String, String]): String =
    val pattern = """\{\{(\w+)\}\}""".r
    pattern.replaceAllIn(template, m =>
      variables.getOrElse(m.group(1), m.group(0))
    )

// ทดสอบ
val template = """
Dear {{name}},

Thank you for registering at {{site}}.
Your username is: {{username}}
Your email is: {{email}}

Best regards,
{{site}} Team
"""

val vars = Map(
  "name" -> "Alice",
  "site" -> "MyApp",
  "username" -> "alice123",
  "email" -> "alice@example.com"
)

println(SimpleTemplate.render(template, vars))
```

### Text Formatting

```scala
object TextFormatter:
  def wordWrap(text: String, maxWidth: Int): String =
    val words = text.split("\\s+")
    val lines = scala.collection.mutable.ArrayBuffer[String]()
    val currentLine = new StringBuilder()

    for word <- words do
      if currentLine.nonEmpty && currentLine.length + 1 + word.length > maxWidth then
        lines += currentLine.toString
        currentLine.clear()
        currentLine.append(word)
      else
        if currentLine.nonEmpty then currentLine.append(" ")
        currentLine.append(word)

    if currentLine.nonEmpty then lines += currentLine.toString
    lines.mkString("\n")

  def centerAlign(text: String, width: Int, fill: Char = ' '): String =
    val padding = width - text.length
    if padding <= 0 then text
    else
      val leftPad = padding / 2
      val rightPad = padding - leftPad
      s"${fill.toString * leftPad}$text${fill.toString * rightPad}"

  def table(headers: List[String], rows: List[List[String]]): String =
    val allRows = headers :: rows
    val colWidths = (0 until headers.length).map { i =>
      allRows.map(row => if i < row.length then row(i).length else 0).max
    }.toList

    def formatRow(row: List[String]): String =
      row.zipWithIndex.map { case (cell, i) =>
        cell.padTo(colWidths(i), ' ')
      }.mkString(" | ")

    val separator = colWidths.map("-" * _).mkString("-+-")

    (formatRow(headers) :: separator :: rows.map(formatRow)).mkString("\n")

// ทดสอบ
val longText = "Scala is a general-purpose programming language that combines object-oriented and functional programming in one concise language."
println(TextFormatter.wordWrap(longText, 40))

println()
println(TextFormatter.centerAlign("Scala Course", 40, '='))

println()
val data = List(
  List("Alice", "30", "Engineer"),
  List("Bob", "25", "Designer"),
  List("Charlie", "35", "Manager")
)
println(TextFormatter.table(List("Name", "Age", "Role"), data))
```

### Levenshtein Distance (String Similarity)

```scala
def levenshtein(s1: String, s2: String): Int =
  val dp = Array.tabulate(s1.length + 1, s2.length + 1)((i, j) =>
    if i == 0 then j
    else if j == 0 then i
    else 0
  )

  for
    i <- 1 to s1.length
    j <- 1 to s2.length
  do
    dp(i)(j) =
      if s1(i-1) == s2(j-1) then dp(i-1)(j-1)
      else 1 + math.min(dp(i-1)(j), math.min(dp(i)(j-1), dp(i-1)(j-1)))

  dp(s1.length)(s2.length)

def similarity(s1: String, s2: String): Double =
  val maxLen = math.max(s1.length, s2.length)
  if maxLen == 0 then 1.0
  else 1.0 - levenshtein(s1, s2).toDouble / maxLen

// ทดสอบ
println(levenshtein("kitten", "sitting"))  // 3
println(levenshtein("scala", "scalable"))  // 4

println(f"Similarity: ${similarity("scala", "scala")}%.2f")   // 1.00
println(f"Similarity: ${similarity("scala", "scaler")}%.2f")  // 0.67
println(f"Similarity: ${similarity("hello", "world")}%.2f")   // 0.40
```

---

## Internationalization

### Unicode Support

```scala
// Scala String รองรับ Unicode เต็มที่
val thai = "สวัสดีโลก"
val japanese = "こんにちは世界"
val arabic = "مرحبا بالعالم"
val emoji = "Hello 🌍! 🎉"

println(thai.length)    // 9 (นับ characters)
println(emoji.length)   // 13 (แต่ emoji อาจเป็น 2 chars!)

// String.codePointCount สำหรับนับ Unicode code points ที่ถูกต้อง
println(emoji.codePointCount(0, emoji.length))  // 10

// Unicode categories
"Hello123!@#".foreach { c =>
  val category = Character.getType(c).toChar match
    case Character.UPPERCASE_LETTER => "UPPER"
    case Character.LOWERCASE_LETTER => "LOWER"
    case Character.DECIMAL_DIGIT_NUMBER => "DIGIT"
    case _ => "OTHER"
  print(s"$c:$category ")
}
println()

// Thai string operations
val thaiText = "สวัสดีชาวโลก"
println(thaiText.length)  // 12
println(thaiText.take(3)) // สวั
println(thaiText.reverse) // กลโวชีดัสว
```

### String Normalization

```scala
import java.text.Normalizer

// Unicode normalization forms
// NFC: Canonical Decomposition, Canonical Composition
// NFD: Canonical Decomposition
// NFKC: Compatibility Decomposition, Canonical Composition
// NFKD: Compatibility Decomposition

val accented = "café"  // might be stored as c-a-f-e + combining accent
val normalized = Normalizer.normalize(accented, Normalizer.Form.NFC)

println(accented.length)    // 5 (อาจต่างกัน)
println(normalized.length)  // 4 (normalized)

// Remove accents
def removeAccents(s: String): String =
  Normalizer.normalize(s, Normalizer.Form.NFD)
    .replaceAll("[\\p{InCombiningDiacriticalMarks}]", "")

println(removeAccents("café"))     // cafe
println(removeAccents("naïve"))    // naive
println(removeAccents("résumé"))   // resume
```

### Locale-aware Operations

```scala
import java.util.Locale

// Case conversion กับ Locale (ภาษาบางภาษามีกฎพิเศษ)
val s = "istanbul"
println(s.toUpperCase)                      // ISTANBUL
println(s.toUpperCase(new Locale("tr")))    // İSTANBUL (Turkish i)

// String comparison กับ Locale
import java.text.Collator

val collator = Collator.getInstance(new Locale("th", "TH"))
val thaiWords = List("แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง")
val sortedThai = thaiWords.sortWith((a, b) => collator.compare(a, b) < 0)
println(sortedThai.mkString(", "))

// Number formatting
import java.text.NumberFormat

val thaiFormat = NumberFormat.getInstance(new Locale("th", "TH"))
val usFormat = NumberFormat.getInstance(Locale.US)

println(thaiFormat.format(1234567.89))  // ๑,๒๓๔,๕๖๗.๘๙ (Thai numerals!)
println(usFormat.format(1234567.89))    // 1,234,567.89
```

---

## StringBuilder และ Performance

### เมื่อไหรควรใช้ StringBuilder

```scala
// ❌ String concatenation ใน loop (สร้าง string ใหม่ทุกครั้ง = O(n^2))
def badConcat(n: Int): String =
  var result = ""
  for i <- 1 to n do
    result += i.toString + ", "  // สร้าง string ใหม่ทุก iteration!
  result

// ✅ StringBuilder (O(n))
def goodConcat(n: Int): String =
  val sb = new StringBuilder()
  for i <- 1 to n do
    sb.append(i)
    sb.append(", ")
  sb.toString()

// ✅ หรือ functional (อ่านง่ายกว่า, performance ดี)
def functionalConcat(n: Int): String =
  (1 to n).mkString(", ")
```

### StringBuilder API

```scala
val sb = new StringBuilder()

// Append
sb.append("Hello")
sb.append(", ")
sb.append("World")
sb.append("!")
println(sb)  // Hello, World!

// Insert
sb.insert(5, " Beautiful")
println(sb)  // Hello Beautiful, World!

// Delete
sb.delete(5, 15)  // ลบ " Beautiful"
println(sb)  // Hello, World!

// Replace
sb.replace(7, 12, "Scala")
println(sb)  // Hello, Scala!

// Reverse
sb.reverse
println(sb)  // !alacS ,olleH

// StringBuilder is mutable - method chaining
val result = new StringBuilder()
  .append("Scala ")
  .append("3")
  .append(" is ")
  .append("awesome!")
  .toString

println(result)  // Scala 3 is awesome!

// แปลงกลับ
val str = sb.toString
val sbFromStr = new StringBuilder("Hello")
```

### String Pool และ Memory

```scala
// String interning (String pool)
val s1 = "hello"
val s2 = "hello"
val s3 = new String("hello")

println(s1 == s2)    // true (value equality)
println(s1 eq s2)    // true (same reference - interned)
println(s1 eq s3)    // false (different object)

// intern: ใส่ string เข้า pool
val s4 = s3.intern()
println(s1 eq s4)   // true (interned, same reference now)

// ในทางปฏิบัติ:
// - String literals อัตโนมัติ interned
// - String ที่สร้างด้วย new หรือ concatenation อาจไม่ interned
// - ใช้ == สำหรับ comparison ไม่ใช้ eq
```

### String Performance Tips

```scala
// 1. ใช้ mkString แทน ++
val list = (1 to 1000).toList
val str1 = list.mkString(", ")  // ✅ ดี
// val str2 = list.map(_.toString).reduce(_ + ", " + _)  // ❌ แย่

// 2. ใช้ interpolation แทน concatenation
val name = "Alice"
val age = 30
val good = s"$name is $age years old"      // ✅
// val bad = name + " is " + age + " years old"  // ❌ (แต่ Scala optimize ได้)

// 3. StringBuilder สำหรับ loop
def buildHTML(items: List[String]): String =
  val sb = new StringBuilder("<ul>\n")
  for item <- items do
    sb.append(s"  <li>$item</li>\n")
  sb.append("</ul>")
  sb.toString

// 4. precompile regex
val emailRegex = """[\w.]+@[\w.]+\.[a-z]{2,}""".r  // compile ครั้งเดียว
def isEmail(s: String): Boolean = emailRegex.matches(s)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: String Analysis

```scala
@main def stringAnalysis(): Unit =
  val paragraph = """
    In the beginning God created the heavens and the earth.
    Now the earth was formless and empty, darkness was over the surface of the deep,
    and the Spirit of God was hovering over the waters.
  """.trim

  // TODO:
  // 1. นับจำนวน words, sentences, characters
  // 2. หา average word length
  // 3. หาคำที่ยาวที่สุด
  // 4. นับ vowels (a,e,i,o,u) และ consonants
  // 5. แสดง word frequency (top 5)
```

### แบบฝึกหัดที่ 2: Email Validator

```scala
@main def emailValidator(): Unit =
  val emails = List(
    "alice@example.com",
    "bob.smith@gmail.com",
    "charlie+tag@domain.co.uk",
    "invalid-email",
    "@nodomain.com",
    "noatsign.com",
    "user@",
    "user@domain",
    "user.name+tag@domain.org"
  )

  // TODO: สร้าง email validator ที่ครอบคลุม
  // กฎ:
  // - มี @ หนึ่งตัว
  // - local part (ก่อน @): ตัวอักษร, ตัวเลข, ., +, -, _
  // - domain: ตัวอักษร, ตัวเลข, -, .
  // - TLD (หลัง . สุดท้าย): ตัวอักษร 2-6 ตัว

  def isValidEmail(email: String): Boolean = ???

  emails.foreach { email =>
    val valid = if isValidEmail(email) then "✅" else "❌"
    println(s"$valid $email")
  }
```

### แบบฝึกหัดที่ 3: CSV Processor

```scala
@main def csvProcessor(): Unit =
  val csvData = """
name,age,salary,department
Alice,30,95000,Engineering
Bob,25,65000,Marketing
Charlie,35,110000,Engineering
Diana,28,72000,HR
Eve,32,98000,Engineering
Frank,27,68000,Marketing
""".trim

  // TODO:
  // 1. Parse CSV
  // 2. Average salary per department
  // 3. Employees sorted by salary (desc)
  // 4. Youngest employee per department
  // 5. Export summary as CSV string
```

### แบบฝึกหัดที่ 4: Log Parser

```scala
@main def logParser(): Unit =
  val logs = """
2024-01-15 10:00:01 [INFO] Server started on port 8080
2024-01-15 10:00:05 [DEBUG] Loading configuration
2024-01-15 10:01:23 [INFO] User alice logged in
2024-01-15 10:02:45 [ERROR] Database connection failed: timeout
2024-01-15 10:02:46 [WARN] Retrying connection (attempt 1/3)
2024-01-15 10:02:47 [WARN] Retrying connection (attempt 2/3)
2024-01-15 10:02:48 [INFO] Database connection restored
2024-01-15 10:05:12 [INFO] User bob logged in
2024-01-15 10:10:00 [ERROR] Out of memory error in worker thread
""".trim

  case class LogEntry(date: String, time: String, level: String, message: String)

  // TODO:
  // 1. Parse แต่ละ log line เป็น LogEntry
  // 2. นับ log entries per level
  // 3. แสดงเฉพาะ ERROR และ WARN entries
  // 4. หา time ของ first และ last entry
```

**เฉลย แบบฝึกหัดที่ 2:**

```scala
def isValidEmail(email: String): Boolean =
  val pattern = """^[a-zA-Z0-9.+_-]+@[a-zA-Z0-9-]+(\.[a-zA-Z0-9-]+)*\.[a-zA-Z]{2,6}$""".r
  pattern.matches(email)
```

**เฉลย แบบฝึกหัดที่ 4:**

```scala
case class LogEntry(date: String, time: String, level: String, message: String)

val logPattern = """(\d{4}-\d{2}-\d{2}) (\d{2}:\d{2}:\d{2}) \[(\w+)\] (.+)""".r

val entries = logs.linesIterator.flatMap { line =>
  line match
    case logPattern(date, time, level, msg) =>
      Some(LogEntry(date, time, level, msg))
    case _ => None
}.toList

// Count by level
val counts = entries.groupBy(_.level).view.mapValues(_.length).toMap
println("Log counts:")
counts.toList.sorted.foreach { case (level, count) =>
  println(s"  $level: $count")
}

// Errors and warnings
println("\nErrors and Warnings:")
entries.filter(e => e.level == "ERROR" || e.level == "WARN")
  .foreach(e => println(s"  [${e.level}] ${e.time} - ${e.message}"))

// Time range
println(s"\nFirst entry: ${entries.head.time}")
println(s"Last entry: ${entries.last.time}")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ String เป็น Seq[Char] และ operations ครบถ้วน
- ✅ Regular expressions กับ Scala
- ✅ String parsing สำหรับ CSV, config, query strings
- ✅ Text processing patterns
- ✅ Unicode และ Internationalization support
- ✅ StringBuilder และ performance optimization

## ขั้นตอนถัดไป

ใน [Part 09: Classes และ Objects](part-09-classes-objects.md) เราจะเรียนรู้:
- Class definitions ครบถ้วน
- Object (companion objects)
- Constructors
- Inheritance
- Abstract classes

---

*[← Part 07: Collections พื้นฐาน](part-07-collections-basic.md) | [Part 09: Classes และ Objects →](part-09-classes-objects.md)*
