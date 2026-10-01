# Part 24: SBT Build Tool เชิงลึก

## สารบัญ
1. [SBT Fundamentals](#sbt-fundamentals)
2. [Multi-Module Projects](#multi-module-projects)
3. [Custom Tasks and Settings](#custom-tasks-and-settings)
4. [Plugins](#plugins)
5. [Publishing](#publishing)

---

## SBT Fundamentals

### build.sbt ครบถ้วน

```scala
// build.sbt

// Project metadata
name    := "my-scala-app"
version := "1.0.0"
organization := "com.example"
scalaVersion := "3.3.1"

// Compiler options
scalacOptions ++= Seq(
  "-deprecation",           // warn on deprecated APIs
  "-feature",               // warn on language features
  "-unchecked",             // warn on unchecked casts
  "-Xfatal-warnings",       // treat warnings as errors
  "-explain",               // explain errors (Scala 3)
  "-source:future",         // future Scala features
)

// Dependencies
libraryDependencies ++= Seq(
  // Core
  "org.typelevel"    %% "cats-core"        % "2.10.0",
  "org.typelevel"    %% "cats-effect"      % "3.5.2",

  // Akka
  "com.typesafe.akka" %% "akka-actor-typed" % "2.8.5",
  "com.typesafe.akka" %% "akka-stream"      % "2.8.5",
  "com.typesafe.akka" %% "akka-http"        % "10.5.3",

  // Databases
  "org.tpolecat"     %% "doobie-core"      % "1.0.0-RC4",
  "org.tpolecat"     %% "doobie-postgres"  % "1.0.0-RC4",
  "io.getquill"      %% "quill-jdbc-zio"   % "4.7.3",

  // JSON
  "io.circe"         %% "circe-core"       % "0.14.6",
  "io.circe"         %% "circe-generic"    % "0.14.6",
  "io.circe"         %% "circe-parser"     % "0.14.6",

  // Logging
  "ch.qos.logback"    %  "logback-classic" % "1.4.11",
  "com.typesafe.scala-logging" %% "scala-logging" % "3.9.5",

  // Test
  "org.scalatest"   %% "scalatest"         % "3.2.17" % Test,
  "org.scalacheck"  %% "scalacheck"        % "1.17.0" % Test,
  "org.mockito"     %% "mockito-scala"     % "1.17.30" % Test
)

// Resolvers
resolvers ++= Seq(
  Resolver.mavenCentral,
  "Sonatype OSS Snapshots" at "https://oss.sonatype.org/content/repositories/snapshots"
)

// Assembly settings (fat JAR)
assembly / mainClass := Some("com.example.Main")
assembly / assemblyMergeStrategy := {
  case PathList("META-INF", xs @ _*) => MergeStrategy.discard
  case "reference.conf"              => MergeStrategy.concat
  case x                             =>
    val defaultStrategy = (assembly / assemblyMergeStrategy).value
    defaultStrategy(x)
}

// Fork JVM for tests
Test / fork := true
Test / javaOptions ++= Seq(
  "-Xms512m",
  "-Xmx2g"
)

// Test options
Test / testOptions += Tests.Argument("-oDF")
```

### SBT Commands

```bash
# เริ่ม SBT interactive
sbt

# คำสั่งใน SBT shell
compile            # compile ทุก source
test               # run ทุก tests
run                # run main class
run arg1 arg2      # run with arguments
clean              # delete target/
reload             # reload build.sbt changes
update             # update dependencies
console            # Scala REPL with project classpath
~compile           # watch mode: recompile on changes
~test              # watch mode: retest on changes

# Run specific test
testOnly com.example.UserServiceSpec

# Run specific test method
testOnly com.example.UserServiceSpec -- -z "should return user"

# Dependency tree
dependencyTree
dependencyBrowseGraph

# Show version
show version
show scalaVersion

# Package
package           # create JAR
assembly          # create fat JAR (requires sbt-assembly)

# Publish
publishLocal      # publish to local ~/.ivy2
publish           # publish to remote repository
```

---

## Multi-Module Projects

### Project Structure

```
my-project/
├── build.sbt
├── project/
│   ├── build.properties
│   └── plugins.sbt
├── core/
│   └── src/main/scala/...
├── api/
│   └── src/main/scala/...
├── server/
│   └── src/main/scala/...
└── client/
    └── src/main/scala/...
```

### build.sbt for Multi-Module

```scala
// Root project
lazy val root = (project in file("."))
  .aggregate(core, api, server, client)
  .settings(
    name := "my-project",
    // Don't publish root
    publish / skip := true
  )

// Common settings
lazy val commonSettings = Seq(
  organization := "com.example",
  scalaVersion := "3.3.1",
  scalacOptions ++= Seq("-deprecation", "-feature"),
  libraryDependencies ++= Seq(
    "org.scalatest" %% "scalatest" % "3.2.17" % Test
  )
)

// Core module: no dependencies
lazy val core = (project in file("core"))
  .settings(
    commonSettings,
    name := "my-project-core",
    libraryDependencies ++= Seq(
      "org.typelevel" %% "cats-core" % "2.10.0"
    )
  )

// API module: depends on core
lazy val api = (project in file("api"))
  .dependsOn(core)
  .settings(
    commonSettings,
    name := "my-project-api",
    libraryDependencies ++= Seq(
      "io.circe" %% "circe-core" % "0.14.6"
    )
  )

// Server module: depends on core and api
lazy val server = (project in file("server"))
  .dependsOn(core, api)
  .settings(
    commonSettings,
    name := "my-project-server",
    libraryDependencies ++= Seq(
      "com.typesafe.akka" %% "akka-http" % "10.5.3"
    )
  )

// Client module: depends on api only
lazy val client = (project in file("client"))
  .dependsOn(api)
  .settings(
    commonSettings,
    name := "my-project-client"
  )
```

---

## Custom Tasks and Settings

### Defining Custom Tasks

```scala
// build.sbt

// Custom setting
val greeting = settingKey[String]("A greeting message")
greeting := "Hello, SBT!"

// Custom task
val sayHello = taskKey[Unit]("Says hello")
sayHello := println(greeting.value)

// Task with dependencies
val generateVersion = taskKey[File]("Generate version file")
generateVersion := {
  val file = (Compile / sourceManaged).value / "Version.scala"
  val content = s"""
    |package com.example
    |object BuildInfo {
    |  val version = "${version.value}"
    |  val scalaVersion = "${scalaVersion.value}"
    |  val buildTime = "${java.time.Instant.now()}"
    |}
  """.stripMargin
  IO.write(file, content)
  file
}

// Add generated source to compilation
Compile / sourceGenerators += generateVersion.taskValue

// Input task (takes user input)
val greetUser = inputKey[Unit]("Greet a specific user")
greetUser := {
  import sbt.complete.DefaultParsers.*
  val name = spaceDelimited("<name>").parsed.headOption.getOrElse("World")
  println(s"Hello, $name!")
}
```

---

## Plugins

### Common SBT Plugins

```scala
// project/plugins.sbt

// Assembly (fat JAR)
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "2.1.3")

// Native packaging (Docker, RPM, Debian)
addSbtPlugin("com.github.sbt" % "sbt-native-packager" % "1.9.16")

// Code formatting
addSbtPlugin("org.scalameta" % "sbt-scalafmt" % "2.5.2")

// Code coverage
addSbtPlugin("org.scoverage" % "sbt-scoverage" % "2.0.9")

// Wartremover (static analysis)
addSbtPlugin("org.wartremover" % "sbt-wartremover" % "3.1.5")

// Dependency updates
addSbtPlugin("com.timushev.sbt" % "sbt-updates" % "0.6.4")

// Build info
addSbtPlugin("com.eed3si9n" % "sbt-buildinfo" % "0.11.0")
```

### Docker with sbt-native-packager

```scala
// build.sbt
enablePlugins(DockerPlugin, JavaAppPackaging)

Docker / packageName := "my-app"
Docker / version := version.value
dockerBaseImage := "eclipse-temurin:17-jre"
dockerExposedPorts ++= Seq(8080)

dockerCommands ++= Seq(
  Cmd("USER", "daemon")
)

// Build Docker image
// sbt docker:publishLocal
// sbt docker:publish
```

---

## Publishing

### Maven Central Publishing

```scala
// build.sbt
publishTo := sonatypePublishToBundle.value
sonatypeCredentialHost := "s01.oss.sonatype.org"

// POM settings
pomExtra := (
  <url>https://github.com/example/project</url>
  <licenses>
    <license>
      <name>Apache 2</name>
      <url>https://www.apache.org/licenses/LICENSE-2.0</url>
    </license>
  </licenses>
  <developers>
    <developer>
      <id>username</id>
      <name>Your Name</name>
      <url>https://github.com/username</url>
    </developer>
  </developers>
  <scm>
    <url>git@github.com:example/project.git</url>
    <connection>scm:git:git@github.com:example/project.git</connection>
  </scm>
)

// project/plugins.sbt
addSbtPlugin("com.github.sbt" % "sbt-pgp" % "2.2.1")
addSbtPlugin("org.xerial.sbt" % "sbt-sonatype" % "3.9.21")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ build.sbt ครบถ้วน: metadata, dependencies, compiler options
- ✅ SBT commands ที่ใช้บ่อย
- ✅ Multi-module projects
- ✅ Custom tasks and settings
- ✅ Common plugins: assembly, docker, scalafmt
- ✅ Publishing to Maven Central

---

*[← Part 23: Testing](part-23-testing.md) | [Part 25: Play Framework →](part-25-play-framework.md)*
