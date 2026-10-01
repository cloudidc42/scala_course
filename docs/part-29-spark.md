# Part 29: Apache Spark กับ Scala

## สารบัญ
1. [Spark Overview](#spark-overview)
2. [RDD API](#rdd-api)
3. [DataFrame API](#dataframe-api)
4. [Spark SQL](#spark-sql)
5. [Spark Streaming](#spark-streaming)
6. [MLlib](#mllib)

---

## Spark Overview

### Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core" % "3.5.0" % "provided",
  "org.apache.spark" %% "spark-sql"  % "3.5.0" % "provided",
  "org.apache.spark" %% "spark-mllib" % "3.5.0" % "provided",
  "org.apache.spark" %% "spark-streaming" % "3.5.0" % "provided"
)

// For local development, remove "provided"
// "provided" means Spark is on the cluster's classpath
```

### SparkSession

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.SparkConf

// Create SparkSession (entry point for Spark 2+)
val spark = SparkSession.builder()
  .appName("My Spark App")
  .master("local[*]")  // local mode, use all CPUs
  .config("spark.sql.shuffle.partitions", "4")
  .config("spark.driver.memory", "4g")
  .getOrCreate()

// Get SparkContext (for RDD operations)
val sc = spark.sparkContext
sc.setLogLevel("WARN")

// Stop session when done
// spark.stop()
```

---

## RDD API

### Creating RDDs

```scala
import org.apache.spark.rdd.RDD

// From collection
val rdd1: RDD[Int] = sc.parallelize(1 to 1000)
val rdd2: RDD[String] = sc.parallelize(List("a", "b", "c"))

// From files
val textRdd: RDD[String] = sc.textFile("hdfs://path/to/file.txt")
val localFile: RDD[String] = sc.textFile("file:///local/path/file.txt")

// From directory
val dirRdd: RDD[String] = sc.textFile("data/")

// Partitions
println(s"Partitions: ${rdd1.getNumPartitions}")
val repartitioned = rdd1.repartition(8)
```

### RDD Transformations

```scala
// map, filter, flatMap
val doubled = rdd1.map(_ * 2)
val evens = rdd1.filter(_ % 2 == 0)
val words = textRdd.flatMap(_.split("\\s+"))

// reduce, fold
val sum = rdd1.reduce(_ + _)
val product = rdd1.fold(1)(_ * _)

// groupBy
val grouped = rdd1.groupBy(_ % 5)  // group by remainder

// Key-value RDD operations
val pairs: RDD[(String, Int)] = sc.parallelize(
  List(("alice", 90), ("bob", 85), ("alice", 95), ("bob", 88))
)

// reduceByKey
val maxScores = pairs.reduceByKey(math.max)
// ("alice", 95), ("bob", 88)

// groupByKey (avoid when possible - use reduceByKey instead)
val grouped2 = pairs.groupByKey()
// ("alice", Iterable(90, 95)), ("bob", Iterable(85, 88))

// aggregateByKey
val stats = pairs.aggregateByKey((0, 0, 0))(
  // seqOp: combine value into accumulator
  (acc, score) => (acc._1 + score, math.max(acc._2, score), acc._3 + 1),
  // combOp: combine two accumulators
  (a1, a2) => (a1._1 + a2._1, math.max(a1._2, a2._2), a1._3 + a2._3)
).map { case (name, (sum, max, count)) =>
  (name, sum.toDouble / count, max)
}

// join
val departments: RDD[(String, String)] = sc.parallelize(
  List(("alice", "engineering"), ("bob", "marketing"))
)

val userDept = pairs.join(departments)
// ("alice", (90, "engineering")), ...
```

### RDD Actions

```scala
// collect: bring all data to driver
val result = rdd1.take(10).toList

// count
println(s"Count: ${rdd1.count()}")

// first, take
println(s"First: ${rdd1.first()}")
val top10 = rdd1.take(10)

// saveAsTextFile
doubled.saveAsTextFile("output/doubled")

// foreach
rdd1.foreach(n => if n % 100 == 0 then println(s"Milestone: $n"))

// Caching (persist in memory)
val cached = rdd1.cache()  // same as persist(MEMORY_ONLY)
// or
import org.apache.spark.storage.StorageLevel
val persisted = rdd1.persist(StorageLevel.MEMORY_AND_DISK)
```

---

## DataFrame API

### Creating DataFrames

```scala
import spark.implicits.*
import org.apache.spark.sql.{DataFrame, Dataset}
import org.apache.spark.sql.functions.*

// From case class collection
case class Employee(
  id: Int,
  name: String,
  department: String,
  salary: Double,
  age: Int
)

val employees = Seq(
  Employee(1, "Alice",   "Engineering", 95000, 30),
  Employee(2, "Bob",     "Marketing",   75000, 28),
  Employee(3, "Charlie", "Engineering", 105000, 35),
  Employee(4, "Diana",   "HR",          70000, 32),
  Employee(5, "Eve",     "Engineering", 88000, 27)
)

val df: DataFrame = employees.toDF()

// Schema
df.printSchema()
df.show()

// From CSV
val csvDf = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("employees.csv")

// From JSON
val jsonDf = spark.read
  .option("multiLine", "true")
  .json("employees.json")

// From Parquet (columnar format - recommended)
val parquetDf = spark.read.parquet("employees.parquet")
```

### DataFrame Operations

```scala
// Select columns
val names = df.select("name", "salary")
val computed = df.select(
  col("name"),
  col("salary"),
  (col("salary") * 1.1).as("new_salary"),
  upper(col("name")).as("upper_name")
)

// Filter (where)
val engineers = df.filter(col("department") === "Engineering")
val highPaid = df.where(col("salary") > 90000)

// GroupBy and aggregation
val deptStats = df.groupBy("department")
  .agg(
    count("*").as("headcount"),
    avg("salary").as("avg_salary"),
    max("salary").as("max_salary"),
    min("age").as("min_age")
  )
  .orderBy("department")

deptStats.show()

// Sort
val sorted = df.orderBy(col("salary").desc, col("name").asc)

// Join
case class Department(id: Int, name: String, location: String)
val depts = Seq(
  Department(1, "Engineering", "Bangkok"),
  Department(2, "Marketing", "Chiang Mai")
).toDF("dept_id", "dept_name", "location")

val empWithLoc = df
  .join(depts, df("department") === depts("dept_name"), "left")
  .select(df("name"), df("salary"), depts("location"))

// Window functions
import org.apache.spark.sql.expressions.Window

val window = Window.partitionBy("department").orderBy(col("salary").desc)

val ranked = df.withColumn("rank", rank().over(window))
  .withColumn("dense_rank", dense_rank().over(window))
  .withColumn("running_avg", avg("salary").over(
    Window.partitionBy("department").orderBy("salary")
      .rowsBetween(Window.unboundedPreceding, Window.currentRow)
  ))
```

---

## Spark SQL

### SQL Queries

```scala
// Register as temp view
df.createOrReplaceTempView("employees")

// SQL query
val result = spark.sql("""
  SELECT
    department,
    COUNT(*) as headcount,
    ROUND(AVG(salary), 2) as avg_salary,
    MAX(salary) as max_salary
  FROM employees
  GROUP BY department
  HAVING COUNT(*) >= 2
  ORDER BY avg_salary DESC
""")

result.show()

// Complex SQL
spark.sql("""
  WITH ranked AS (
    SELECT *,
      ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rn
    FROM employees
  )
  SELECT department, name, salary
  FROM ranked
  WHERE rn = 1
""").show()
```

---

## Spark Streaming

### Structured Streaming

```scala
import org.apache.spark.sql.streaming.*

// Read from socket (for testing)
val socketStream = spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()

// Read from Kafka
val kafkaStream = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "localhost:9092")
  .option("subscribe", "events")
  .load()

// Process streaming data
val words = socketStream
  .select(explode(split(col("value"), " ")).as("word"))

val wordCounts = words.groupBy("word").count()

// Write output
val query = wordCounts.writeStream
  .outputMode("complete")
  .format("console")
  .trigger(Trigger.ProcessingTime("5 seconds"))
  .start()

query.awaitTermination()

// Windowed aggregations
val windowed = words
  .withWatermark("timestamp", "10 minutes")
  .groupBy(
    window(col("timestamp"), "5 minutes", "1 minute"),
    col("word")
  )
  .count()
```

---

## MLlib

### Machine Learning

```scala
import org.apache.spark.ml.*
import org.apache.spark.ml.classification.*
import org.apache.spark.ml.feature.*
import org.apache.spark.ml.evaluation.*
import org.apache.spark.ml.Pipeline

// Load and prepare data
val data = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("iris.csv")

// Feature engineering pipeline
val indexer = new StringIndexer()
  .setInputCol("species")
  .setOutputCol("label")

val assembler = new VectorAssembler()
  .setInputCols(Array("sepal_length", "sepal_width", "petal_length", "petal_width"))
  .setOutputCol("features")

val scaler = new StandardScaler()
  .setInputCol("features")
  .setOutputCol("scaled_features")
  .setWithStd(true)
  .setWithMean(true)

// Classifier
val rf = new RandomForestClassifier()
  .setFeaturesCol("scaled_features")
  .setLabelCol("label")
  .setNumTrees(100)

// Pipeline
val pipeline = new Pipeline()
  .setStages(Array(indexer, assembler, scaler, rf))

// Train/test split
val Array(trainData, testData) = data.randomSplit(Array(0.8, 0.2), seed = 42)

// Train
val model = pipeline.fit(trainData)

// Evaluate
val predictions = model.transform(testData)
val evaluator = new MulticlassClassificationEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")
  .setMetricName("accuracy")

val accuracy = evaluator.evaluate(predictions)
println(f"Accuracy: ${accuracy * 100}%.2f%%")

// Save model
model.write.overwrite().save("models/iris-classifier")
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ SparkSession setup
- ✅ RDD API: transformations, actions, caching
- ✅ DataFrame API: select, filter, groupBy, join, window functions
- ✅ Spark SQL: queries, CTEs
- ✅ Structured Streaming: socket, Kafka, windowed aggregations
- ✅ MLlib: feature engineering, Random Forest, Pipeline

---

*[← Part 28: Circe JSON](part-28-circe-json.md) | [Part 30: ZIO →](part-30-zio.md)*
