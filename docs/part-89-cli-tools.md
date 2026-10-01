# ตอนที่ 89: Building CLI Tools with Scala

## สารบัญ

1. [decline Library สำหรับ Command Parsing](#decline-library-สำหรับ-command-parsing)
2. [Subcommands และ Options](#subcommands-และ-options)
3. [การอ่านจาก stdin](#การอ่านจาก-stdin)
4. [Progress Bars และ Colored Output](#progress-bars-และ-colored-output)
5. [Configuration File Support](#configuration-file-support)
6. [Exit Codes และ Error Handling](#exit-codes-และ-error-handling)
7. [Complete CLI Tool Example](#complete-cli-tool-example)
8. [สรุป](#สรุป)

---

## decline Library สำหรับ Command Parsing

decline เป็น library สำหรับ parsing command-line arguments แบบ type-safe และ composable

### การตั้งค่า build.sbt

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.1"

lazy val root = (project in file("."))
  .settings(
    name := "scala-cli-tool",
    libraryDependencies ++= Seq(
      // Command line parsing
      "com.monovore" %% "decline"        % "2.4.1",
      "com.monovore" %% "decline-effect" % "2.4.1",
      
      // Cats Effect
      "org.typelevel" %% "cats-effect"   % "3.5.2",
      
      // Config file support
      "com.typesafe"   % "config"        % "1.4.3",
      "io.circe"      %% "circe-core"    % "0.14.6",
      "io.circe"      %% "circe-generic" % "0.14.6",
      "io.circe"      %% "circe-parser"  % "0.14.6",
      "io.circe"      %% "circe-yaml"    % "0.15.1",
      
      // Logging
      "org.typelevel" %% "log4cats-slf4j" % "2.6.0",
      "org.slf4j"      % "slf4j-simple"   % "2.0.9"
    ),
    
    // Package as native executable (optional with GraalVM)
    assembly / mainClass := Some("com.example.cli.Main"),
    assembly / assemblyJarName := "mycli.jar"
  )
```

### พื้นฐาน decline

```scala
package com.example.cli

import com.monovore.decline.*
import cats.syntax.all.*

// Opt - optional arguments
val verboseOpt: Opts[Boolean] =
  Opts.flag("verbose", "Enable verbose output", short = "v").orFalse

val outputOpt: Opts[String] =
  Opts.option[String]("output", "Output file path", short = "o")
    .withDefault("output.txt")

val countOpt: Opts[Int] =
  Opts.option[Int]("count", "Number of items", short = "n")
    .validate("Count must be positive")(_ > 0)
    .withDefault(10)

// Argument - positional arguments
val inputArg: Opts[String] =
  Opts.argument[String]("input-file")

val filesArg: Opts[List[String]] =
  Opts.arguments[String]("files").orEmpty

// สร้าง Command พื้นฐาน
case class GreetConfig(name: String, count: Int, loud: Boolean)

val greetCommand: Command[GreetConfig] = Command(
  name   = "greet",
  header = "Greet someone"
):
  val nameArg  = Opts.argument[String]("name")
  val countOpt = Opts.option[Int]("count", "Times to greet", short = "n").withDefault(1)
  val loudOpt  = Opts.flag("loud", "Use uppercase", short = "l").orFalse
  
  (nameArg, countOpt, loudOpt).mapN(GreetConfig.apply)

// การใช้งาน
@main def simpleMain(args: String*): Unit =
  greetCommand.parse(args.toSeq) match
    case Right(config) =>
      val greeting = s"Hello, ${config.name}!"
      val output   = if config.loud then greeting.toUpperCase else greeting
      (1 to config.count).foreach(_ => println(output))
    
    case Left(help) =>
      System.err.println(help)
      sys.exit(1)
```

---

## Subcommands และ Options

### Multi-Command CLI

```scala
package com.example.cli

import com.monovore.decline.*
import com.monovore.decline.effect.*
import cats.effect.*
import cats.syntax.all.*

// =============================================================
// Command configs
// =============================================================

case class GlobalConfig(
  verbose: Boolean,
  configFile: Option[String],
  outputFormat: OutputFormat
)

enum OutputFormat:
  case Text, Json, Csv

// Subcommand configs
case class ListConfig(
  global: GlobalConfig,
  filter: Option[String],
  limit: Int,
  sortBy: String
)

case class CreateConfig(
  global: GlobalConfig,
  name: String,
  tags: List[String],
  description: Option[String]
)

case class DeleteConfig(
  global: GlobalConfig,
  id: String,
  force: Boolean
)

case class ExportConfig(
  global: GlobalConfig,
  outputPath: String,
  format: OutputFormat,
  since: Option[String]
)

// =============================================================
// Opts definitions
// =============================================================

object Opts:
  import com.monovore.decline.{Opts => DeclOpts}
  
  val verboseOpt: DeclOpts[Boolean] =
    DeclOpts.flag("verbose", "Enable verbose logging", short = "v").orFalse
  
  val configOpt: DeclOpts[Option[String]] =
    DeclOpts.option[String]("config", "Config file path", short = "c").orNone
  
  val formatOpt: DeclOpts[OutputFormat] =
    DeclOpts.option[String]("format", "Output format (text|json|csv)", short = "f")
      .mapValidated: s =>
        s.toLowerCase match
          case "text" => cats.data.Validated.valid(OutputFormat.Text)
          case "json" => cats.data.Validated.valid(OutputFormat.Json)
          case "csv"  => cats.data.Validated.valid(OutputFormat.Csv)
          case other  => cats.data.Validated.invalidNel(s"Unknown format: $other")
      .withDefault(OutputFormat.Text)
  
  val globalOpts: DeclOpts[GlobalConfig] =
    (verboseOpt, configOpt, formatOpt).mapN(GlobalConfig.apply)

// =============================================================
// Subcommand definitions
// =============================================================

val listSubcmd: Command[ListConfig] = Command(
  name   = "list",
  header = "List all items"
):
  val filterOpt = com.monovore.decline.Opts
    .option[String]("filter", "Filter by name pattern", short = "f").orNone
  val limitOpt = com.monovore.decline.Opts
    .option[Int]("limit", "Max items to show", short = "l")
    .validate("Limit must be positive")(_ > 0)
    .withDefault(20)
  val sortOpt = com.monovore.decline.Opts
    .option[String]("sort", "Sort field", short = "s").withDefault("name")
  
  (Opts.globalOpts, filterOpt, limitOpt, sortOpt).mapN(ListConfig.apply)

val createSubcmd: Command[CreateConfig] = Command(
  name   = "create",
  header = "Create a new item"
):
  val nameArg = com.monovore.decline.Opts.argument[String]("name")
  val tagsOpt = com.monovore.decline.Opts
    .options[String]("tag", "Add tag (can repeat)", short = "t").orEmpty
  val descOpt = com.monovore.decline.Opts
    .option[String]("description", "Item description", short = "d").orNone
  
  (Opts.globalOpts, nameArg, tagsOpt, descOpt).mapN(CreateConfig.apply)

val deleteSubcmd: Command[DeleteConfig] = Command(
  name   = "delete",
  header = "Delete an item"
):
  val idArg    = com.monovore.decline.Opts.argument[String]("id")
  val forceOpt = com.monovore.decline.Opts.flag("force", "Skip confirmation", short = "f").orFalse
  
  (Opts.globalOpts, idArg, forceOpt).mapN(DeleteConfig.apply)

val exportSubcmd: Command[ExportConfig] = Command(
  name   = "export",
  header = "Export data to file"
):
  val outputArg = com.monovore.decline.Opts.argument[String]("output-path")
  val formatOpt = Opts.formatOpt
  val sinceOpt  = com.monovore.decline.Opts
    .option[String]("since", "Export items since date (YYYY-MM-DD)").orNone
  
  (Opts.globalOpts, outputArg, formatOpt, sinceOpt).mapN(ExportConfig.apply)

// =============================================================
// Main command with subcommands
// =============================================================

sealed trait AppCommand
case class ListCmd(config: ListConfig) extends AppCommand
case class CreateCmd(config: CreateConfig) extends AppCommand
case class DeleteCmd(config: DeleteConfig) extends AppCommand
case class ExportCmd(config: ExportConfig) extends AppCommand

val mainCommand: Command[AppCommand] = Command(
  name   = "mycli",
  header = "My CLI Tool - Manage your data from the command line"
):
  import com.monovore.decline.Opts as DeclOpts
  
  DeclOpts.subcommands(
    listSubcmd.map(ListCmd.apply),
    createSubcmd.map(CreateCmd.apply),
    deleteSubcmd.map(DeleteCmd.apply),
    exportSubcmd.map(ExportCmd.apply)
  )
```

### Main Application ด้วย decline-effect

```scala
package com.example.cli

import com.monovore.decline.effect.CommandIOApp
import com.monovore.decline.*
import cats.effect.*
import cats.syntax.all.*

object Main extends CommandIOApp(
  name    = "mycli",
  header  = "My CLI Tool",
  version = "1.0.0"
):
  
  def main: Opts[IO[ExitCode]] =
    import com.monovore.decline.{Opts => DeclOpts}
    
    DeclOpts.subcommands(
      listSubcmd.map(handleList),
      createSubcmd.map(handleCreate),
      deleteSubcmd.map(handleDelete),
      exportSubcmd.map(handleExport)
    )
  
  private def handleList(config: ListConfig): IO[ExitCode] =
    for
      _ <- logVerbose(config.global, s"Listing items (limit=${config.limit}, sort=${config.sortBy})")
      items <- fetchItems(config.filter, config.limit, config.sortBy)
      _ <- printItems(items, config.global.outputFormat)
    yield ExitCode.Success
  
  private def handleCreate(config: CreateConfig): IO[ExitCode] =
    for
      _ <- logVerbose(config.global, s"Creating item: ${config.name}")
      item <- createItem(config.name, config.tags, config.description)
      _ <- printSuccess(s"Created item with ID: ${item.id}")
    yield ExitCode.Success
  
  private def handleDelete(config: DeleteConfig): IO[ExitCode] =
    for
      confirmed <- if config.force then IO.pure(true)
                   else promptConfirmation(s"Delete item ${config.id}? [y/N] ")
      result <- if confirmed then
                  deleteItem(config.id).map(_ => ExitCode.Success)
                else
                  IO.println("Deletion cancelled.") >> IO.pure(ExitCode.Success)
    yield result
  
  private def handleExport(config: ExportConfig): IO[ExitCode] =
    for
      _ <- logVerbose(config.global, s"Exporting to ${config.outputPath}")
      items <- fetchItems(None, Int.MaxValue, "id")
      _ <- exportItems(items, config.outputPath, config.format)
      _ <- printSuccess(s"Exported ${items.length} items to ${config.outputPath}")
    yield ExitCode.Success
  
  // Helper functions
  private def logVerbose(global: GlobalConfig, msg: String): IO[Unit] =
    if global.verbose then IO.println(s"[DEBUG] $msg") else IO.unit
  
  private def promptConfirmation(prompt: String): IO[Boolean] =
    IO.print(prompt) >>
    IO.readLine.map(_.trim.toLowerCase == "y")
  
  private def printSuccess(msg: String): IO[Unit] =
    IO.println(s"✓ $msg")  // สามารถใช้ colored output ได้
  
  // Placeholder implementations
  private def fetchItems(filter: Option[String], limit: Int, sortBy: String) = IO.pure(List.empty[Item])
  private def createItem(name: String, tags: List[String], desc: Option[String]) = IO.pure(Item("new-id", name))
  private def deleteItem(id: String) = IO.unit
  private def printItems(items: List[Item], format: OutputFormat) = IO.unit
  private def exportItems(items: List[Item], path: String, format: OutputFormat) = IO.unit

case class Item(id: String, name: String)
```

---

## การอ่านจาก stdin

```scala
package com.example.cli

import cats.effect.*
import fs2.*
import fs2.io.*
import java.io.{BufferedReader, InputStreamReader}

object StdinReader:
  
  // อ่านทีละบรรทัด
  def readLines(): IO[List[String]] =
    IO:
      val reader = new BufferedReader(new InputStreamReader(System.in))
      Iterator
        .continually(reader.readLine())
        .takeWhile(_ != null)
        .toList
  
  // อ่านแบบ streaming
  def streamLines(): Stream[IO, String] =
    fs2.io.stdin[IO](4096)
      .through(fs2.text.utf8.decode)
      .through(fs2.text.lines)
  
  // ตรวจสอบว่ามี stdin data หรือเปล่า (piped input)
  def hasStdinData(): IO[Boolean] =
    IO(System.in.available() > 0 || !System.console().isInstanceOf[java.io.Console])
      .handleError(_ => false)

// CLI ที่รับ input จาก pipe หรือ argument
def processInput(
  inputFile: Option[String],
  processLine: String => IO[String]
): IO[Unit] =
  val lines: IO[List[String]] = inputFile match
    case Some(path) =>
      IO:
        val source = scala.io.Source.fromFile(path)
        try source.getLines().toList
        finally source.close()
    
    case None =>
      StdinReader.readLines()
  
  lines.flatMap: ls =>
    ls.traverse(processLine).flatMap: results =>
      results.traverse_(IO.println)
```

---

## Progress Bars และ Colored Output

### ANSI Color Codes

```scala
package com.example.cli

// ANSI color codes
object Colors:
  val Reset   = "\u001B[0m"
  val Bold    = "\u001B[1m"
  val Dim     = "\u001B[2m"
  
  // Foreground colors
  val Black   = "\u001B[30m"
  val Red     = "\u001B[31m"
  val Green   = "\u001B[32m"
  val Yellow  = "\u001B[33m"
  val Blue    = "\u001B[34m"
  val Magenta = "\u001B[35m"
  val Cyan    = "\u001B[36m"
  val White   = "\u001B[37m"
  
  // Bright colors
  val BrightRed    = "\u001B[91m"
  val BrightGreen  = "\u001B[92m"
  val BrightYellow = "\u001B[93m"
  val BrightBlue   = "\u001B[94m"
  val BrightCyan   = "\u001B[96m"
  val BrightWhite  = "\u001B[97m"
  
  // Background colors
  val BgRed    = "\u001B[41m"
  val BgGreen  = "\u001B[42m"
  val BgYellow = "\u001B[43m"
  val BgBlue   = "\u001B[44m"

// Color helper functions
def success(msg: String): String = s"${Colors.BrightGreen}✓${Colors.Reset} $msg"
def error(msg: String): String   = s"${Colors.BrightRed}✗${Colors.Reset} $msg"
def warning(msg: String): String = s"${Colors.BrightYellow}⚠${Colors.Reset} $msg"
def info(msg: String): String    = s"${Colors.BrightBlue}ℹ${Colors.Reset} $msg"
def bold(msg: String): String    = s"${Colors.Bold}$msg${Colors.Reset}"
def dim(msg: String): String     = s"${Colors.Dim}$msg${Colors.Reset}"

// Check if terminal supports colors
def supportsColor: Boolean =
  Option(System.getenv("TERM")).exists(_ != "dumb") &&
  Option(System.getenv("NO_COLOR")).isEmpty &&
  System.console() != null

def colored(msg: String, color: String): String =
  if supportsColor then s"$color$msg${Colors.Reset}"
  else msg
```

### Progress Bar

```scala
package com.example.cli

import cats.effect.*
import cats.effect.std.Queue
import scala.concurrent.duration.*

class ProgressBar(
  total: Int,
  width: Int = 40,
  prefix: String = "Progress"
):
  
  private var current = 0
  
  def update(n: Int = 1): IO[Unit] = IO:
    current = math.min(current + n, total)
    render()
  
  def complete(): IO[Unit] = IO:
    current = total
    render()
    println() // New line after completion
  
  def withMessage(msg: String): IO[Unit] = IO:
    render(Some(msg))
  
  private def render(msg: Option[String] = None): Unit =
    val percentage = if total > 0 then (current.toDouble / total * 100).toInt else 0
    val filled     = if total > 0 then (current.toDouble / total * width).toInt else 0
    val empty      = width - filled
    
    val bar     = "█" * filled + "░" * empty
    val msgStr  = msg.map(m => s" $m").getOrElse("")
    val line    = s"\r${Colors.BrightBlue}$prefix${Colors.Reset} [$bar] $percentage% ($current/$total)$msgStr"
    
    print(line)
    System.out.flush()

// Spinner
class Spinner(message: String):
  private val frames = Array("⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏")
  private var frameIdx = 0
  
  def tick(): IO[Unit] = IO:
    val frame = frames(frameIdx % frames.length)
    frameIdx += 1
    print(s"\r${Colors.Cyan}$frame${Colors.Reset} $message")
    System.out.flush()
  
  def stop(finalMessage: String): IO[Unit] = IO:
    println(s"\r${success(finalMessage)}${" " * 20}")

// Usage example
def longRunningTask(items: List[String]): IO[List[String]] =
  val bar = ProgressBar(items.length, prefix = "Processing")
  
  items.zipWithIndex.traverse: (item, idx) =>
    IO.sleep(50.milliseconds) >>
    bar.update(1) >>
    bar.withMessage(s"Processing: $item") >>
    IO.pure(item.toUpperCase)
  .flatTap(_ => bar.complete())
```

### Table Printer

```scala
package com.example.cli

object TablePrinter:
  
  case class Column(name: String, width: Int, align: Alignment = Alignment.Left)
  
  enum Alignment:
    case Left, Right, Center
  
  def print(
    columns: List[Column],
    rows: List[List[String]],
    withBorder: Boolean = true
  ): String =
    val header = formatRow(columns.map(_.name), columns)
    val separator = columns.map(c => "─" * c.width).mkString("┼", "─┼─", "─")
    val dataRows = rows.map(formatRow(_, columns))
    
    if withBorder then
      val top    = columns.map(c => "─" * c.width).mkString("┌", "─┬─", "─┐")
      val bottom = columns.map(c => "─" * c.width).mkString("└", "─┴─", "─┘")
      val lines  = List(top, header, separator) ++ dataRows ++ List(bottom)
      lines.mkString("\n")
    else
      (List(header, separator) ++ dataRows).mkString("\n")
  
  private def formatRow(values: List[String], columns: List[Column]): String =
    val cells = values.zip(columns).map: (value, col) =>
      formatCell(value, col.width, col.align)
    cells.mkString("│ ", " │ ", " │")
  
  private def formatCell(value: String, width: Int, align: Alignment): String =
    val truncated = if value.length > width then value.take(width - 3) + "..." else value
    align match
      case Alignment.Left   => truncated.padTo(width, ' ')
      case Alignment.Right  => truncated.reverse.padTo(width, ' ').reverse
      case Alignment.Center =>
        val padding = width - truncated.length
        val leftPad = padding / 2
        val rightPad = padding - leftPad
        " " * leftPad + truncated + " " * rightPad

// Usage
val result = TablePrinter.print(
  columns = List(
    TablePrinter.Column("ID", 8),
    TablePrinter.Column("Name", 20),
    TablePrinter.Column("Status", 10),
    TablePrinter.Column("Created", 12, TablePrinter.Alignment.Right)
  ),
  rows = List(
    List("001", "Alice Johnson", "active", "2024-01-15"),
    List("002", "Bob Smith", "inactive", "2024-02-20"),
    List("003", "Charlie Brown", "active", "2024-03-10")
  )
)
```

---

## Configuration File Support

### YAML/TOML Config

```scala
package com.example.cli

import io.circe.*
import io.circe.generic.auto.*
import io.circe.parser.*
import cats.effect.*
import java.nio.file.{Files, Paths}

// Config model
case class AppConfig(
  apiUrl: String = "https://api.example.com",
  apiKey: Option[String] = None,
  timeout: Int = 30,
  maxRetries: Int = 3,
  outputFormat: String = "text",
  colorOutput: Boolean = true,
  database: DatabaseConfig = DatabaseConfig()
)

case class DatabaseConfig(
  host: String = "localhost",
  port: Int = 5432,
  name: String = "mydb",
  user: String = "postgres",
  password: Option[String] = None
)

class ConfigLoader:
  
  private val configLocations = List(
    ".mycli.json",
    ".config/mycli/config.json",
    s"${System.getProperty("user.home")}/.mycli.json",
    s"${System.getProperty("user.home")}/.config/mycli/config.json"
  )
  
  def load(explicitPath: Option[String] = None): IO[AppConfig] =
    val paths = explicitPath.toList ++ configLocations
    
    findFirstExists(paths).flatMap:
      case None       => IO.pure(AppConfig()) // Default config
      case Some(path) => loadFromFile(path)
  
  private def findFirstExists(paths: List[String]): IO[Option[String]] =
    IO:
      paths.find(p => Files.exists(Paths.get(p)))
  
  private def loadFromFile(path: String): IO[AppConfig] =
    IO:
      val content = new String(Files.readAllBytes(Paths.get(path)))
      decode[AppConfig](content) match
        case Right(config) => config
        case Left(err)     => throw new RuntimeException(s"Failed to parse config $path: $err")
  
  def save(config: AppConfig, path: String): IO[Unit] =
    IO:
      import io.circe.syntax.*
      val json = config.asJson.spaces2
      Files.writeString(Paths.get(path), json)
  
  // Merge command line overrides with config file
  def mergeWithOverrides(
    base: AppConfig,
    apiUrl: Option[String] = None,
    apiKey: Option[String] = None,
    timeout: Option[Int] = None,
    outputFormat: Option[String] = None
  ): AppConfig = base.copy(
    apiUrl       = apiUrl.getOrElse(base.apiUrl),
    apiKey       = apiKey.orElse(base.apiKey),
    timeout      = timeout.getOrElse(base.timeout),
    outputFormat = outputFormat.getOrElse(base.outputFormat)
  )

// Environment variables support
object EnvConfig:
  
  def loadFromEnv(): Map[String, String] =
    val envVars = Map(
      "MYCLI_API_URL"    -> "apiUrl",
      "MYCLI_API_KEY"    -> "apiKey",
      "MYCLI_TIMEOUT"    -> "timeout",
      "MYCLI_OUTPUT_FMT" -> "outputFormat"
    )
    
    envVars.flatMap: (envVar, configKey) =>
      Option(System.getenv(envVar)).map(configKey -> _)
  
  def applyToConfig(config: AppConfig): AppConfig =
    val env = loadFromEnv()
    config.copy(
      apiUrl       = env.getOrElse("apiUrl", config.apiUrl),
      apiKey       = env.get("apiKey").orElse(config.apiKey),
      timeout      = env.get("timeout").flatMap(_.toIntOption).getOrElse(config.timeout),
      outputFormat = env.getOrElse("outputFormat", config.outputFormat)
    )
```

---

## Exit Codes และ Error Handling

### Exit Code Management

```scala
package com.example.cli

import cats.effect.*

// Custom exit codes
object ExitCodes:
  val Success         = 0
  val GeneralError    = 1
  val MisuseError     = 2  // Command misuse
  val CannotExecute   = 126
  val NotFound        = 127
  
  // Application-specific codes
  val NotFound2        = 10
  val AuthError        = 11
  val NetworkError     = 12
  val ParseError       = 13
  val PermissionDenied = 14

// Error hierarchy
sealed trait CliError extends Throwable:
  def message: String
  def exitCode: Int
  override def getMessage: String = message

case class NotFoundError(message: String) extends CliError:
  val exitCode = ExitCodes.NotFound2

case class AuthenticationError(message: String) extends CliError:
  val exitCode = ExitCodes.AuthError

case class NetworkError(message: String) extends CliError:
  val exitCode = ExitCodes.NetworkError

case class ParseError(message: String) extends CliError:
  val exitCode = ExitCodes.ParseError

case class UserCancelledError() extends CliError:
  val message = "Operation cancelled by user"
  val exitCode = ExitCodes.Success  // User cancel is not an error

// Error handler
class ErrorHandler:
  
  def handle(error: Throwable): IO[ExitCode] =
    error match
      case e: CliError =>
        printError(e.message) >> IO.pure(ExitCode(e.exitCode))
      
      case e: java.io.FileNotFoundException =>
        printError(s"File not found: ${e.getMessage}") >>
        IO.pure(ExitCode(ExitCodes.NotFound2))
      
      case e: java.net.ConnectException =>
        printError(s"Connection failed: ${e.getMessage}") >>
        IO.pure(ExitCode(ExitCodes.NetworkError))
      
      case e: SecurityException =>
        printError(s"Permission denied: ${e.getMessage}") >>
        IO.pure(ExitCode(ExitCodes.PermissionDenied))
      
      case e =>
        printError(s"Unexpected error: ${e.getMessage}") >>
        IO.pure(ExitCode(ExitCodes.GeneralError))
  
  private def printError(msg: String): IO[Unit] =
    IO(System.err.println(error(msg)))

// App with error handling
def runWithErrorHandling(app: IO[ExitCode]): IO[ExitCode] =
  val handler = new ErrorHandler()
  app.handleErrorWith(handler.handle)
```

---

## Complete CLI Tool Example

### File Manager CLI

```scala
package com.example.cli.filemanager

import com.monovore.decline.*
import com.monovore.decline.effect.*
import cats.effect.*
import cats.syntax.all.*
import java.nio.file.*
import java.nio.file.attribute.BasicFileAttributes
import scala.jdk.CollectionConverters.*

// =============================================================
// Commands
// =============================================================

sealed trait FmCommand
case class ListFiles(path: String, recursive: Boolean, showHidden: Boolean, sortBy: String) extends FmCommand
case class CopyFile(src: String, dest: String, overwrite: Boolean) extends FmCommand
case class DeleteFile(path: String, recursive: Boolean, force: Boolean) extends FmCommand
case class SearchFiles(dir: String, pattern: String, maxDepth: Int) extends FmCommand
case class FileStats(path: String, detailed: Boolean) extends FmCommand

// =============================================================
// Parser
// =============================================================

val listCmd: Command[FmCommand] = Command("ls", "List files and directories"):
  import com.monovore.decline.{Opts => O}
  val pathArg   = O.argument[String]("path").withDefault(".")
  val recurOpt  = O.flag("recursive", "List recursively", short = "r").orFalse
  val hiddenOpt = O.flag("all", "Show hidden files", short = "a").orFalse
  val sortOpt   = O.option[String]("sort", "Sort by: name|size|time", short = "s").withDefault("name")
  (pathArg, recurOpt, hiddenOpt, sortOpt).mapN(ListFiles.apply)

val copyCmd: Command[FmCommand] = Command("cp", "Copy file or directory"):
  import com.monovore.decline.{Opts => O}
  val srcArg       = O.argument[String]("source")
  val destArg      = O.argument[String]("destination")
  val overwriteOpt = O.flag("force", "Overwrite if exists", short = "f").orFalse
  (srcArg, destArg, overwriteOpt).mapN(CopyFile.apply)

val deleteCmd: Command[FmCommand] = Command("rm", "Delete file or directory"):
  import com.monovore.decline.{Opts => O}
  val pathArg   = O.argument[String]("path")
  val recurOpt  = O.flag("recursive", "Delete directory recursively", short = "r").orFalse
  val forceOpt  = O.flag("force", "No confirmation prompt", short = "f").orFalse
  (pathArg, recurOpt, forceOpt).mapN(DeleteFile.apply)

val searchCmd: Command[FmCommand] = Command("find", "Search for files"):
  import com.monovore.decline.{Opts => O}
  val dirArg      = O.argument[String]("directory").withDefault(".")
  val patternOpt  = O.option[String]("name", "File name pattern (glob)", short = "n").withDefault("*")
  val depthOpt    = O.option[Int]("max-depth", "Max search depth", short = "d").withDefault(Int.MaxValue)
  (dirArg, patternOpt, depthOpt).mapN(SearchFiles.apply)

val statsCmd: Command[FmCommand] = Command("stat", "Show file statistics"):
  import com.monovore.decline.{Opts => O}
  val pathArg      = O.argument[String]("path")
  val detailedOpt  = O.flag("detailed", "Show detailed info", short = "d").orFalse
  (pathArg, detailedOpt).mapN(FileStats.apply)

// =============================================================
// File operations
// =============================================================

object FileOperations:
  
  case class FileInfo(
    name: String,
    path: String,
    isDirectory: Boolean,
    size: Long,
    lastModified: Long,
    permissions: String
  )
  
  def listFiles(
    path: String,
    recursive: Boolean,
    showHidden: Boolean,
    sortBy: String
  ): IO[List[FileInfo]] = IO:
    val dir = Paths.get(path)
    
    if !Files.exists(dir) then
      throw NotFoundError(s"Path not found: $path")
    
    val stream = if recursive then
      Files.walk(dir)
    else
      Files.list(dir)
    
    val files = stream.iterator().asScala
      .filter: p =>
        showHidden || !p.getFileName.toString.startsWith(".")
      .map: p =>
        val attrs = Files.readAttributes(p, classOf[BasicFileAttributes])
        FileInfo(
          name          = p.getFileName.toString,
          path          = p.toString,
          isDirectory   = Files.isDirectory(p),
          size          = if Files.isDirectory(p) then -1L else attrs.size(),
          lastModified  = attrs.lastModifiedTime().toMillis,
          permissions   = getPermissions(p)
        )
      .toList
    
    sortBy match
      case "size" => files.sortBy(_.size)
      case "time" => files.sortBy(-_.lastModified)
      case _      => files.sortBy(_.name)
  
  def copyFile(src: String, dest: String, overwrite: Boolean): IO[Unit] = IO:
    val srcPath  = Paths.get(src)
    val destPath = Paths.get(dest)
    
    if !Files.exists(srcPath) then
      throw NotFoundError(s"Source not found: $src")
    
    val options = if overwrite then
      Array(StandardCopyOption.REPLACE_EXISTING)
    else Array.empty[CopyOption]
    
    Files.copy(srcPath, destPath, options*)
  
  def deleteFile(path: String, recursive: Boolean, force: Boolean): IO[Unit] =
    val p = Paths.get(path)
    
    if !Files.exists(p) then
      IO.raiseError(NotFoundError(s"Path not found: $path"))
    else if Files.isDirectory(p) && !recursive then
      IO.raiseError(ParseError(s"$path is a directory, use --recursive to delete"))
    else if !force then
      IO.print(s"Delete '$path'? [y/N] ") >>
      IO.readLine.flatMap:
        case "y" | "Y" => doDelete(p, recursive)
        case _         => IO.raiseError(UserCancelledError())
    else
      doDelete(p, recursive)
  
  private def doDelete(path: Path, recursive: Boolean): IO[Unit] = IO:
    if recursive && Files.isDirectory(path) then
      Files.walk(path)
        .sorted(java.util.Comparator.reverseOrder())
        .forEach(Files.delete)
    else
      Files.delete(path)
  
  def searchFiles(dir: String, pattern: String, maxDepth: Int): IO[List[String]] = IO:
    val dirPath = Paths.get(dir)
    val matcher = FileSystems.getDefault.getPathMatcher(s"glob:**/$pattern")
    
    Files.walk(dirPath, maxDepth)
      .iterator().asScala
      .filter(matcher.matches)
      .map(_.toString)
      .toList
  
  def getStats(path: String): IO[Map[String, String]] = IO:
    val p     = Paths.get(path)
    val attrs = Files.readAttributes(p, classOf[BasicFileAttributes])
    
    Map(
      "Path"       -> p.toAbsolutePath.toString,
      "Type"       -> (if Files.isDirectory(p) then "Directory" else "File"),
      "Size"       -> formatSize(attrs.size()),
      "Created"    -> java.util.Date(attrs.creationTime().toMillis).toString,
      "Modified"   -> java.util.Date(attrs.lastModifiedTime().toMillis).toString,
      "Permissions" -> getPermissions(p)
    )
  
  private def getPermissions(path: Path): String =
    try
      val perms = Files.getPosixFilePermissions(path)
      import java.nio.file.attribute.PosixFilePermission.*
      
      def p(perm: java.nio.file.attribute.PosixFilePermission, ch: Char) =
        if perms.contains(perm) then ch else '-'
      
      s"${p(OWNER_READ, 'r')}${p(OWNER_WRITE, 'w')}${p(OWNER_EXECUTE, 'x')}" +
      s"${p(GROUP_READ, 'r')}${p(GROUP_WRITE, 'w')}${p(GROUP_EXECUTE, 'x')}" +
      s"${p(OTHERS_READ, 'r')}${p(OTHERS_WRITE, 'w')}${p(OTHERS_EXECUTE, 'x')}"
    catch
      case _ => "rwxrwxrwx" // Windows fallback
  
  def formatSize(bytes: Long): String =
    if bytes < 0 then "-"
    else if bytes < 1024 then s"${bytes}B"
    else if bytes < 1024 * 1024 then f"${bytes / 1024.0}%.1fKB"
    else if bytes < 1024 * 1024 * 1024 then f"${bytes / (1024.0 * 1024)}%.1fMB"
    else f"${bytes / (1024.0 * 1024 * 1024)}%.2fGB"

// =============================================================
// Output formatters
// =============================================================

object OutputFormatter:
  
  def formatFileList(files: List[FileOperations.FileInfo]): String =
    if files.isEmpty then
      dim("(empty)")
    else
      val columns = List(
        TablePrinter.Column("Type", 3),
        TablePrinter.Column("Name", 30),
        TablePrinter.Column("Size", 10, TablePrinter.Alignment.Right),
        TablePrinter.Column("Modified", 20)
      )
      
      val rows = files.map: f =>
        val typeStr = if f.isDirectory then colored("dir", Colors.Cyan) else "file"
        val sizeStr = FileOperations.formatSize(f.size)
        val dateStr = new java.util.Date(f.lastModified).toString.take(20)
        val nameStr = if f.isDirectory then colored(f.name, Colors.Bold) else f.name
        List(typeStr, nameStr, sizeStr, dateStr)
      
      TablePrinter.print(columns, rows)

// =============================================================
// Main CLI Application
// =============================================================

object FileManagerApp extends CommandIOApp(
  name    = "fm",
  header  = "Scala File Manager CLI",
  version = "1.0.0"
):
  
  def main: com.monovore.decline.Opts[IO[ExitCode]] =
    import com.monovore.decline.{Opts => O}
    
    O.subcommands(
      listCmd.map(handleCommand),
      copyCmd.map(handleCommand),
      deleteCmd.map(handleCommand),
      searchCmd.map(handleCommand),
      statsCmd.map(handleCommand)
    )
  
  private def handleCommand(cmd: FmCommand): IO[ExitCode] =
    runWithErrorHandling:
      cmd match
        case ListFiles(path, recursive, showHidden, sortBy) =>
          for
            files <- FileOperations.listFiles(path, recursive, showHidden, sortBy)
            _ <- IO.println(OutputFormatter.formatFileList(files))
            _ <- IO.println(info(s"${files.length} items in '$path'"))
          yield ExitCode.Success
        
        case CopyFile(src, dest, overwrite) =>
          FileOperations.copyFile(src, dest, overwrite) >>
          IO.println(success(s"Copied '$src' to '$dest'")) >>
          IO.pure(ExitCode.Success)
        
        case DeleteFile(path, recursive, force) =>
          FileOperations.deleteFile(path, recursive, force) >>
          IO.println(success(s"Deleted '$path'")) >>
          IO.pure(ExitCode.Success)
        
        case SearchFiles(dir, pattern, maxDepth) =>
          FileOperations.searchFiles(dir, pattern, maxDepth).flatMap: results =>
            if results.isEmpty then
              IO.println(warning(s"No files matching '$pattern' found in '$dir'")) >>
              IO.pure(ExitCode.Success)
            else
              results.traverse_(IO.println) >>
              IO.println(info(s"Found ${results.length} matching files")) >>
              IO.pure(ExitCode.Success)
        
        case FileStats(path, detailed) =>
          FileOperations.getStats(path).flatMap: stats =>
            val lines = stats.map: (key, value) =>
              s"${bold(key.padTo(12, ' '))} $value"
            lines.toList.traverse_(IO.println) >>
            IO.pure(ExitCode.Success)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **decline Library**: การสร้าง type-safe command-line parsers
2. **Subcommands**: จัดการ commands หลายอย่างใน application เดียว
3. **Options และ Arguments**: ประเภทต่างๆ ของ CLI inputs
4. **stdin Reading**: อ่าน input จาก pipe และ keyboard
5. **Colored Output**: ใช้ ANSI codes ทำ terminal output สวยงาม
6. **Progress Bars**: แสดง progress ของ long-running tasks
7. **Table Printer**: แสดงข้อมูลในรูปแบบตาราง
8. **Config Files**: จัดการ configuration ด้วย JSON
9. **Exit Codes**: จัดการ exit codes อย่างถูกต้อง
10. **Error Handling**: Error hierarchy ที่มี exit codes
11. **Complete Example**: File Manager CLI ที่ใช้งานได้จริง

### Best Practices สำหรับ CLI Tools

- **ใช้ type-safe parser** (decline) แทน manual string parsing
- **ตรวจสอบ stdin** เพื่อรองรับ piping
- **เคารพ NO_COLOR** environment variable
- **ให้ exit codes ที่ถูกต้อง** สำหรับ scripting
- **เพิ่ม --help ที่อธิบายชัดเจน** สำหรับทุก command
- **รองรับ config files** เพื่อหลีกเลี่ยงการพิมพ์ซ้ำ
- **ใช้ progress indicators** สำหรับ long operations
- **Write to stderr** สำหรับ errors และ logs

---

*[← ตอนที่ 88: Web Scraping with Scala](part-88-web-scraping.md) | [ตอนที่ 90: Advanced Patterns →](part-90-advanced-patterns.md)*
