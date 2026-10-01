# Part 01: แนะนำ Scala และการติดตั้ง

## สารบัญ
1. [Scala คืออะไร?](#scala-คืออะไร)
2. [ทำไมต้องเรียน Scala?](#ทำไมต้องเรียน-scala)
3. [Scala ถูกใช้ที่ไหนบ้าง?](#scala-ถูกใช้ที่ไหนบ้าง)
4. [การติดตั้ง JDK](#การติดตั้ง-jdk)
5. [การติดตั้ง Scala และ SBT](#การติดตั้ง-scala-และ-sbt)
6. [การติดตั้ง IDE](#การติดตั้ง-ide)
7. [โปรแกรมแรก: Hello World](#โปรแกรมแรก-hello-world)
8. [โครงสร้างโปรเจกต์ SBT](#โครงสร้างโปรเจกต์-sbt)
9. [Scala REPL](#scala-repl)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Scala คืออะไร?

Scala (Scalable Language) เป็นภาษาโปรแกรมมิ่งระดับสูงที่ผสานแนวคิดสองอย่างเข้าด้วยกัน:

- **Object-Oriented Programming (OOP)**: ทุกอย่างเป็น Object
- **Functional Programming (FP)**: ฟังก์ชันเป็น First-class citizen

### ประวัติโดยย่อ

| ปี | เหตุการณ์ |
|----|-----------|
| 2001 | Martin Odersky เริ่มพัฒนา Scala ที่ EPFL (Switzerland) |
| 2004 | Scala เวอร์ชัน 1.0 เปิดตัว |
| 2006 | Scala 2.0 ออกมาพร้อมฟีเจอร์ใหม่มากมาย |
| 2011 | Typesafe (ปัจจุบันคือ Lightbend) ก่อตั้ง |
| 2016 | Scala 2.12 รองรับ Java 8 lambdas |
| 2021 | Scala 3.0 เปิดตัว (ปรับปรุงครั้งใหญ่) |
| 2023 | Scala 3.3 LTS (Long Term Support) |

### Scala ทำงานอย่างไร?

```
โค้ด Scala (.scala)
        ↓
   Scala Compiler (scalac)
        ↓
   JVM Bytecode (.class)
        ↓
   Java Virtual Machine (JVM)
        ↓
   รันบนทุก Platform ที่มี JVM
```

สิ่งนี้ทำให้ Scala:
- รันได้บนทุก OS ที่มี JVM (Windows, macOS, Linux)
- ใช้ Java libraries ได้ทั้งหมด
- ทำงานร่วมกับโค้ด Java ได้โดยตรง

---

## ทำไมต้องเรียน Scala?

### 1. ภาษาที่ทรงพลังและกระชับ

เปรียบเทียบโค้ดเดียวกันระหว่าง Java และ Scala:

**Java:**
```java
// สร้าง Person class ใน Java
public class Person {
    private final String name;
    private final int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Person)) return false;
        Person other = (Person) obj;
        return name.equals(other.name) && age == other.age;
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }

    @Override
    public String toString() {
        return "Person(" + name + ", " + age + ")";
    }
}
```

**Scala:**
```scala
// สร้าง Person class ใน Scala - แค่บรรทัดเดียว!
case class Person(name: String, age: Int)
```

Scala สร้าง `equals`, `hashCode`, `toString`, `copy` และอื่นๆ ให้อัตโนมัติ!

### 2. ระบบ Type ที่แข็งแกร่ง

```scala
// Scala รู้ประเภทข้อมูลโดยอัตโนมัติ (Type Inference)
val name = "Alice"        // Scala รู้ว่าเป็น String
val age = 25              // Scala รู้ว่าเป็น Int
val scores = List(1, 2, 3) // Scala รู้ว่าเป็น List[Int]

// แต่ก็ยังตรวจสอบ type ตอน compile
val wrong: String = 42    // Error! ไม่สามารถใส่ Int ใน String ได้
```

### 3. Functional Programming ที่ทรงพลัง

```scala
val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

// หาผลรวมของเลขคู่ที่ยกกำลัง 2
val result = numbers
  .filter(_ % 2 == 0)    // [2, 4, 6, 8, 10]
  .map(x => x * x)        // [4, 16, 36, 64, 100]
  .sum                     // 220

println(result) // 220
```

### 4. Concurrency ที่ยอดเยี่ยม

```scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

// ทำงาน 3 อย่างพร้อมกัน
val f1 = Future { fetchDataFromDatabase() }
val f2 = Future { callExternalAPI() }
val f3 = Future { processLocalFile() }

// รอให้ทุกอย่างเสร็จ
for {
  data1 <- f1
  data2 <- f2
  data3 <- f3
} yield combineResults(data1, data2, data3)
```

### 5. ค่าตอบแทนสูง

ตาม Stack Overflow Survey และ various job boards:
- Scala developers เป็นหนึ่งในผู้ที่มีค่าตอบแทนสูงที่สุด
- ความต้องการในตลาดสูงมาก โดยเฉพาะ Data Engineering และ Backend

---

## Scala ถูกใช้ที่ไหนบ้าง?

### บริษัทชั้นนำที่ใช้ Scala

| บริษัท | การใช้งาน |
|--------|-----------|
| **Twitter/X** | Backend ทั้งหมด, real-time processing |
| **Netflix** | Recommendation engine, data pipelines |
| **LinkedIn** | Apache Kafka (เขียนด้วย Scala) |
| **Airbnb** | Data infrastructure |
| **Spotify** | Data pipelines, analytics |
| **Goldman Sachs** | Financial systems |
| **Morgan Stanley** | Trading systems |
| **Walmart** | E-commerce backend |
| **The Guardian** | News platform backend |
| **SoundCloud** | Streaming infrastructure |

### Use Cases หลัก

1. **Big Data Processing**
   - Apache Spark (เขียนด้วย Scala)
   - Apache Kafka (เขียนด้วย Scala)
   - Apache Flink มี Scala API

2. **Backend Web Development**
   - Play Framework
   - Akka HTTP
   - Http4s
   - ZIO HTTP

3. **Distributed Systems**
   - Akka/Pekko (Actor Model)
   - Lagom Framework

4. **Financial Technology**
   - Real-time trading systems
   - Risk calculation engines

5. **Machine Learning Infrastructure**
   - Breeze (numerical processing)
   - Spark MLlib

---

## การติดตั้ง JDK

Scala รันบน JVM ดังนั้นต้องติดตั้ง JDK ก่อน

### ตรวจสอบว่ามี JDK อยู่แล้วหรือไม่

```bash
java -version
```

ถ้าได้ output แบบนี้คือมีแล้ว:
```
openjdk version "17.0.9" 2023-10-17
OpenJDK Runtime Environment (build 17.0.9+9)
OpenJDK 64-Bit Server VM (build 17.0.9+9, mixed mode)
```

### การติดตั้ง JDK 17 (แนะนำ)

#### macOS

```bash
# ใช้ Homebrew
brew install openjdk@17

# เพิ่ม PATH
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

#### Ubuntu/Debian Linux

```bash
# อัพเดท package list
sudo apt update

# ติดตั้ง OpenJDK 17
sudo apt install openjdk-17-jdk

# ตรวจสอบ
java -version
```

#### Windows

1. ดาวน์โหลด OpenJDK 17 จาก https://adoptium.net/
2. รันไฟล์ติดตั้ง .msi
3. ติ๊ก "Set JAVA_HOME variable" และ "Add to PATH"
4. รีสตาร์ท Command Prompt/PowerShell
5. ตรวจสอบด้วย `java -version`

#### ใช้ SDKMAN (แนะนำสำหรับ Mac/Linux)

SDKMAN ช่วยจัดการหลาย version ของ JDK ได้สะดวก:

```bash
# ติดตั้ง SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ติดตั้ง JDK 17
sdk install java 17.0.9-tem

# ดู JDK ที่มีให้เลือก
sdk list java

# สลับ JDK version
sdk use java 17.0.9-tem
```

---

## การติดตั้ง Scala และ SBT

### วิธีที่ 1: ใช้ Coursier (แนะนำ)

Coursier เป็น launcher สำหรับ Scala ecosystem ที่ดีที่สุด

#### macOS/Linux

```bash
# ดาวน์โหลดและติดตั้ง Coursier
curl -fL https://github.com/coursier/launchers/raw/master/cs-x86_64-pc-linux.gz | gzip -d > cs
chmod +x cs
./cs setup

# หรือด้วย brew (macOS)
brew install coursier/formulas/coursier
cs setup
```

#### Windows (PowerShell)

```powershell
# ดาวน์โหลด cs launcher
Invoke-WebRequest -Uri "https://github.com/coursier/launchers/raw/master/cs-x86_64-pc-win32.zip" -OutFile "cs.zip"
Expand-Archive -Path "cs.zip" -DestinationPath "."
.\cs setup
```

หลัง `cs setup` จะได้:
- `scala` - Scala compiler/REPL
- `scalac` - Scala compiler
- `sbt` - Scala Build Tool
- `sbtn` - SBT native client (เร็วกว่า)

#### ตรวจสอบการติดตั้ง

```bash
scala -version
# Scala code runner version 3.3.1 -- Copyright 2002-2023, LAMP/EPFL

sbt -version
# sbt version in this project: 1.9.7
# sbt script version: 1.9.7
```

### วิธีที่ 2: ติดตั้งแยก

#### ติดตั้ง Scala

```bash
# macOS
brew install scala

# Ubuntu/Debian
sudo apt-get install scala

# หรือดาวน์โหลดจาก https://www.scala-lang.org/download/
```

#### ติดตั้ง SBT

```bash
# macOS
brew install sbt

# Ubuntu/Debian
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
sudo apt-get update
sudo apt-get install sbt

# Windows - ดาวน์โหลด MSI จาก https://www.scala-sbt.org/download.html
```

---

## การติดตั้ง IDE

### IntelliJ IDEA (แนะนำ)

IntelliJ IDEA เป็น IDE ที่ดีที่สุดสำหรับ Scala

1. **ดาวน์โหลด IntelliJ IDEA Community (ฟรี)**
   - ไปที่ https://www.jetbrains.com/idea/download/
   - เลือก Community Edition

2. **ติดตั้ง Scala Plugin**
   - เปิด IntelliJ IDEA
   - ไปที่ Settings/Preferences → Plugins
   - ค้นหา "Scala"
   - ติดตั้ง "Scala" plugin จาก JetBrains

3. **ตั้งค่า JDK**
   - File → Project Structure → Platform Settings → SDKs
   - คลิก + → Add JDK
   - เลือก path ที่ติดตั้ง JDK

### VS Code + Metals

สำหรับคนที่ชอบ VS Code:

1. ติดตั้ง VS Code
2. ไปที่ Extensions (Ctrl+Shift+X)
3. ค้นหาและติดตั้ง "Scala (Metals)"
4. เปิดโปรเจกต์ SBT → Metals จะแนะนำให้ import โปรเจกต์

### Neovim/Vim

```bash
# ใช้ nvim-metals สำหรับ Neovim
# ดูรายละเอียดที่ https://github.com/scalameta/nvim-metals
```

---

## โปรแกรมแรก: Hello World

### สร้างโปรเจกต์แรก

```bash
# สร้าง directory ใหม่
mkdir hello-scala
cd hello-scala

# สร้างโปรเจกต์ SBT ใหม่
sbt new scala/scala3.g8
```

SBT จะถามชื่อโปรเจกต์ ตอบ `hello-world`

### โครงสร้างไฟล์ที่ได้

```
hello-world/
├── build.sbt           ← ไฟล์ configuration หลัก
├── project/
│   ├── build.properties
│   └── plugins.sbt
└── src/
    ├── main/
    │   └── scala/
    │       └── Main.scala   ← ไฟล์โค้ดหลัก
    └── test/
        └── scala/
            └── MySuite.scala
```

### เปิดไฟล์ Main.scala

```scala
// src/main/scala/Main.scala

@main def hello(): Unit =
  println("Hello, World!")
  println("ยินดีต้อนรับสู่ Scala!")
```

### รันโปรแกรม

```bash
cd hello-world
sbt run
```

Output:
```
[info] welcome to sbt 1.9.7
[info] loading project definition from .../project
[info] loading settings for project hello-world from build.sbt
[info] set current project to hello-world
[info] compiling 1 Scala source to .../target/scala-3.3.1/classes
[info] running hello
Hello, World!
ยินดีต้อนรับสู่ Scala!
[success] Total time: 5 s
```

### Hello World แบบต่างๆ ใน Scala

```scala
// แบบที่ 1: @main annotation (Scala 3 - แนะนำ)
@main def hello(): Unit =
  println("Hello, World!")

// แบบที่ 2: object + main method (ทั้ง Scala 2 และ 3)
object HelloWorld {
  def main(args: Array[String]): Unit = {
    println("Hello, World!")
  }
}

// แบบที่ 3: extends App (Scala 2 - ไม่แนะนำใน Scala 3)
object HelloWorld extends App {
  println("Hello, World!")
}
```

---

## โครงสร้างโปรเจกต์ SBT

### build.sbt พื้นฐาน

```scala
// build.sbt

// ชื่อโปรเจกต์
name := "hello-world"

// เวอร์ชันโปรเจกต์
version := "0.1.0-SNAPSHOT"

// เวอร์ชัน Scala ที่ใช้
scalaVersion := "3.3.1"

// Dependencies
libraryDependencies ++= Seq(
  // Testing
  "org.scalameta" %% "munit" % "0.7.29" % Test
)
```

### project/build.properties

```properties
sbt.version=1.9.7
```

### SBT Commands ที่ใช้บ่อย

```bash
# คอมไพล์โค้ด
sbt compile

# รันโปรแกรม
sbt run

# รัน tests
sbt test

# เข้า SBT interactive mode
sbt

# ใน SBT shell:
> compile    # คอมไพล์
> run        # รัน
> test       # test
> console    # เปิด Scala REPL ใน context โปรเจกต์
> reload     # โหลด build.sbt ใหม่
> clean      # ลบ compiled files
> ~compile   # คอมไพล์อัตโนมัติเมื่อไฟล์เปลี่ยน (watch mode)
> ~test      # รัน test อัตโนมัติ
> exit       # ออกจาก SBT
```

### โครงสร้างโปรเจกต์แบบ Multi-module

```
my-project/
├── build.sbt
├── project/
│   ├── build.properties
│   └── plugins.sbt
├── core/                    ← module หลัก
│   └── src/
│       ├── main/scala/
│       └── test/scala/
├── api/                     ← module API
│   └── src/
│       ├── main/scala/
│       └── test/scala/
└── cli/                     ← module CLI
    └── src/
        ├── main/scala/
        └── test/scala/
```

```scala
// build.sbt สำหรับ multi-module

lazy val root = project
  .in(file("."))
  .aggregate(core, api, cli)
  .settings(
    name := "my-project",
    scalaVersion := "3.3.1"
  )

lazy val core = project
  .in(file("core"))
  .settings(
    name := "core",
    scalaVersion := "3.3.1",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-core" % "2.10.0"
    )
  )

lazy val api = project
  .in(file("api"))
  .dependsOn(core)
  .settings(
    name := "api",
    scalaVersion := "3.3.1"
  )

lazy val cli = project
  .in(file("cli"))
  .dependsOn(core)
  .settings(
    name := "cli",
    scalaVersion := "3.3.1"
  )
```

---

## Scala REPL

REPL (Read-Eval-Print Loop) ช่วยให้ทดลองโค้ดได้ทันทีโดยไม่ต้องสร้างไฟล์

### เปิด REPL

```bash
# เปิด Scala REPL
scala

# หรือใน SBT
sbt console
```

### การใช้งาน REPL พื้นฐาน

```scala
// พิมพ์โค้ดแล้วกด Enter
scala> 1 + 1
val res0: Int = 2

scala> "Hello" + " " + "World"
val res1: String = Hello World

scala> val x = 10
val x: Int = 10

scala> val y = 20
val y: Int = 20

scala> x + y
val res2: Int = 30

scala> println("ทดสอบ REPL")
ทดสอบ REPL

// กำหนดฟังก์ชัน
scala> def add(a: Int, b: Int): Int = a + b
def add(a: Int, b: Int): Int

scala> add(3, 4)
val res3: Int = 7

// ดู type ของค่า
scala> :type List(1, 2, 3)
List[Int]

// ออกจาก REPL
scala> :quit
// หรือกด Ctrl+D
```

### REPL Commands ที่มีประโยชน์

```
:help          - แสดง help
:type <expr>   - แสดง type ของ expression
:paste         - เข้า paste mode (สำหรับโค้ดหลายบรรทัด)
:load <file>   - โหลดไฟล์ .scala
:reset         - รีเซ็ต session
:quit          - ออกจาก REPL
```

### REPL Paste Mode

สำหรับโค้ดหลายบรรทัด:

```scala
scala> :paste
// Entering paste mode (ctrl-D to finish)

def factorial(n: Int): Int =
  if n <= 1 then 1
  else n * factorial(n - 1)

// กด Ctrl+D เพื่อ execute
// Exiting paste mode, now interpreting.

def factorial(n: Int): Int

scala> factorial(5)
val res0: Int = 120
```

---

## สคริปต์ Scala

Scala 3 รองรับการเขียนสคริปต์โดยตรง (Scripting):

### สร้างไฟล์ hello.sc

```scala
// hello.sc
println("Hello from Scala Script!")

val name = "นักเรียน Scala"
println(s"ยินดีต้อนรับ $name")

// คำนวณ
val numbers = 1 to 10
val sum = numbers.sum
println(s"ผลรวม 1-10 = $sum")
```

### รันสคริปต์

```bash
scala hello.sc
```

Output:
```
Hello from Scala Script!
ยินดีต้อนรับ นักเรียน Scala
ผลรวม 1-10 = 55
```

---

## โปรแกรมตัวอย่างเพิ่มเติม

### ตัวอย่าง 1: คำนวณ Fibonacci

```scala
// src/main/scala/Fibonacci.scala

@main def fibonacci(): Unit =
  // แบบ recursive
  def fib(n: Int): Int =
    if n <= 1 then n
    else fib(n - 1) + fib(n - 2)

  // แสดง fibonacci 10 ตัวแรก
  println("Fibonacci sequence:")
  for i <- 0 until 10 do
    print(s"fib($i) = ${fib(i)}, ")

  println()

  // แบบ tail recursive (ไม่เกิด stack overflow)
  def fibTail(n: Int, a: Int = 0, b: Int = 1): Int =
    if n == 0 then a
    else fibTail(n - 1, b, a + b)

  println(s"\nfib(30) = ${fibTail(30)}")
  println(s"fib(40) = ${fibTail(40)}")
```

### ตัวอย่าง 2: โปรแกรมรับ Input

```scala
// src/main/scala/InputExample.scala

@main def inputExample(): Unit =
  // รับชื่อจาก user
  print("กรุณาใส่ชื่อของคุณ: ")
  val name = scala.io.StdIn.readLine()

  // รับอายุ
  print("กรุณาใส่อายุของคุณ: ")
  val age = scala.io.StdIn.readLine().toInt

  // แสดงผล
  println(s"สวัสดี $name!")
  println(s"อายุ $age ปี")
  println(s"อีก ${65 - age} ปีจะเกษียณ")
```

### ตัวอย่าง 3: โปรแกรม Calculator

```scala
// src/main/scala/Calculator.scala

@main def calculator(): Unit =
  println("=== เครื่องคิดเลข Scala ===")

  def calculate(a: Double, op: String, b: Double): Option[Double] =
    op match
      case "+" => Some(a + b)
      case "-" => Some(a - b)
      case "*" => Some(a * b)
      case "/" =>
        if b != 0 then Some(a / b)
        else None
      case _ => None

  // ทดสอบ
  val operations = List(
    (10.0, "+", 5.0),
    (10.0, "-", 3.0),
    (4.0, "*", 6.0),
    (15.0, "/", 3.0),
    (10.0, "/", 0.0)
  )

  for (a, op, b) <- operations do
    calculate(a, op, b) match
      case Some(result) => println(s"$a $op $b = $result")
      case None         => println(s"$a $op $b = Error! (หารด้วยศูนย์)")
```

---

## ความแตกต่างระหว่าง Scala 2 และ Scala 3

| ฟีเจอร์ | Scala 2 | Scala 3 |
|---------|---------|---------|
| Main method | `object App extends App` | `@main def app()` |
| Braces | บังคับ `{ }` | Optional (indent-based) |
| Implicits | `implicit` | `given`/`using` |
| Type aliases | จำกัด | ปรับปรุงมาก |
| Enums | Sealed traits | `enum` keyword |
| Union Types | ไม่มี | มี (`A \| B`) |
| Intersection Types | ปรับปรุงแล้ว | ปรับปรุงแล้ว |
| Macros | Experimental | stable API |

ในหลักสูตรนี้จะเน้น **Scala 3** แต่จะระบุส่วนที่ต่างกันกับ Scala 2 เมื่อจำเป็น

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ติดตั้งและทดสอบ
1. ติดตั้ง JDK 17
2. ติดตั้ง Scala และ SBT ผ่าน Coursier
3. ตรวจสอบการติดตั้งด้วย `scala -version` และ `sbt -version`

### แบบฝึกหัดที่ 2: Hello World
1. สร้างโปรเจกต์ SBT ใหม่ชื่อ `my-first-scala`
2. แก้ไข Main.scala ให้แสดงข้อความ "สวัสดีโลก! ผมกำลังเรียน Scala"
3. รันโปรแกรม

### แบบฝึกหัดที่ 3: REPL
เปิด Scala REPL และลองคำสั่งต่อไปนี้:
```scala
// 1. คำนวณ
1 + 1
10 * 20
100 / 4
15 % 4    // เศษจากการหาร

// 2. String operations
"Hello".length
"Hello".toUpperCase
"Hello" + " World"
"Hello".reverse

// 3. List operations
List(1, 2, 3, 4, 5).sum
List(1, 2, 3, 4, 5).max
List(1, 2, 3, 4, 5).filter(_ > 3)
List(1, 2, 3).map(_ * 2)
```

### แบบฝึกหัดที่ 4: สคริปต์แรก
สร้างไฟล์ `my_info.sc` ที่แสดง:
- ชื่อของคุณ
- อายุของคุณ
- ภาษาโปรแกรมที่คุณรู้มาก่อนหน้า
- เหตุผลที่อยากเรียน Scala

### แบบฝึกหัดที่ 5: โปรแกรมคิดเลข
สร้างโปรแกรมที่:
1. แสดงตาราง multiplication table 1-10
2. หาผลรวมของตัวเลข 1-100
3. หาค่าเฉลี่ยของ `List(85, 92, 78, 90, 88, 95, 72, 88)`

**เฉลย แบบฝึกหัดที่ 5:**

```scala
@main def exercise5(): Unit =
  // ตาราง multiplication table
  println("=== ตาราง Multiplication ===")
  for i <- 1 to 10 do
    for j <- 1 to 10 do
      print(f"${i * j}%4d")
    println()

  // ผลรวม 1-100
  val sum = (1 to 100).sum
  println(s"\nผลรวม 1-100 = $sum")

  // ค่าเฉลี่ย
  val scores = List(85, 92, 78, 90, 88, 95, 72, 88)
  val average = scores.sum.toDouble / scores.length
  println(f"ค่าเฉลี่ยคะแนน = $average%.2f")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Scala คืออะไรและมีประวัติความเป็นมาอย่างไร
- ✅ ทำไมถึงควรเรียน Scala
- ✅ Scala ถูกใช้งานที่ไหนบ้างในโลกจริง
- ✅ การติดตั้ง JDK, Scala และ SBT
- ✅ การติดตั้งและตั้งค่า IDE
- ✅ เขียนและรันโปรแกรม Hello World แรก
- ✅ เข้าใจโครงสร้างโปรเจกต์ SBT
- ✅ ใช้งาน Scala REPL
- ✅ ความแตกต่างพื้นฐานระหว่าง Scala 2 และ 3

## ขั้นตอนถัดไป

ใน [Part 02: ไวยากรณ์พื้นฐาน](part-02-basic-syntax.md) เราจะเรียนรู้:
- โครงสร้างพื้นฐานของโปรแกรม Scala
- Comments
- Indentation-based syntax
- Expressions vs Statements
- การแสดงผลต่างๆ

---

*[← กลับหน้าหลัก](../README.md) | [Part 02: ไวยากรณ์พื้นฐาน →](part-02-basic-syntax.md)*
