# ส่วนที่ 70: Apache Spark Advanced - การใช้งาน Spark ขั้นสูง

## สารบัญ

1. [Spark Architecture: Driver, Executors, DAG](#spark-architecture)
2. [RDD vs DataFrame vs Dataset](#rdd-dataframe-dataset)
3. [Spark SQL และ Catalyst Optimizer](#spark-sql-catalyst)
4. [กลยุทธ์การแบ่ง Partition](#partitioning-strategies)
5. [Caching และ Persistence Levels](#caching-persistence)
6. [Broadcast Variables และ Accumulators](#broadcast-accumulators)
7. [Spark Streaming with Structured Streaming](#spark-streaming)
8. [ตัวอย่าง ML Pipeline สมบูรณ์](#ml-pipeline)
9. [สรุป](#summary)

---

## 1. Spark Architecture: Driver, Executors, DAG {#spark-architecture}

### ภาพรวมสถาปัตยกรรม Spark

Apache Spark ทำงานแบบ Master-Worker architecture โดยมีองค์ประกอบหลัก 3 ส่วน:

- **Driver Program**: กระบวนการหลักที่รัน `main()` และสร้าง SparkContext
- **Cluster Manager**: จัดการทรัพยากร (YARN, Mesos, Kubernetes, Standalone)
- **Executors**: กระบวนการที่รันบน Worker Nodes

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core"      % "3.5.0",
  "org.apache.spark" %% "spark-sql"       % "3.5.0",
  "org.apache.spark" %% "spark-mllib"     % "3.5.0",
  "org.apache.spark" %% "spark-streaming" % "3.5.0"
)
```

```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.SparkSession

object SparkArchitectureDemo:
  
  def main(args: Array[String]): Unit =
    // สร้าง SparkSession (entry point สำหรับ Spark 2.x+)
    val spark = SparkSession.builder()
      .appName("SparkArchitectureDemo")
      .master("local[*]")  // รัน local mode ด้วย cores ทั้งหมด
      .config("spark.executor.memory", "2g")
      .config("spark.executor.cores", "2")
      .config("spark.default.parallelism", "8")
      .getOrCreate()
    
    // SparkContext สำหรับ RDD operations
    val sc = spark.sparkContext
    
    // ดู configuration
    println(s"App Name: ${sc.appName}")
    println(s"Master: ${sc.master}")
    println(s"Default Parallelism: ${sc.defaultParallelism}")
    
    // ดู DAG Scheduler info
    val conf = spark.sparkContext.getConf
    conf.getAll.foreach { case (k, v) => 
      if k.contains("spark.sql") then println(s"$k = $v")
    }
    
    spark.stop()
```

### DAG (Directed Acyclic Graph)

Spark แปลง transformations เป็น DAG ก่อน execute:

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*

object DAGDemo:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("DAGDemo")
      .master("local[*]")
      .getOrCreate()
    
    import spark.implicits.*
    
    // Stage 1: อ่านข้อมูล (narrow transformation)
    val df1 = spark.range(1, 1000000)
      .withColumn("value", rand())
      .withColumn("category", (col("id") % 5).cast("string"))
    
    // Stage 2: Filter + Map (narrow transformations - รวมเป็น stage เดียว)
    val df2 = df1
      .filter(col("value") > 0.5)
      .withColumn("doubled", col("value") * 2)
    
    // Stage 3: GroupBy (wide transformation - สร้าง shuffle stage ใหม่)
    val df3 = df2.groupBy("category")
      .agg(
        count("*").as("count"),
        avg("doubled").as("avg_doubled"),
        sum("doubled").as("sum_doubled")
      )
    
    // แสดง execution plan
    df3.explain(mode = "extended")
    
    // ดู physical plan
    df3.explain("codegen")
    
    df3.show()
    spark.stop()
```

### การเข้าใจ Stages และ Tasks

```scala
object StagesAndTasksDemo:
  
  def demonstrateStages(spark: SparkSession): Unit =
    import spark.implicits.*
    
    val sc = spark.sparkContext
    
    // RDD เพื่อแสดง stages อย่างชัดเจน
    val rdd1 = sc.parallelize(1 to 100, numSlices = 4)
    
    // Narrow transformations (ไม่ต้อง shuffle)
    val rdd2 = rdd1
      .map(_ * 2)           // Stage 0
      .filter(_ % 3 == 0)   // Stage 0 (ต่อเนื่อง)
    
    // Wide transformation (ต้อง shuffle - สร้าง stage ใหม่)
    val rdd3 = rdd2.groupBy(_ % 10)  // Stage 1 (shuffle)
    
    // ดู partitions
    println(s"rdd1 partitions: ${rdd1.getNumPartitions}")
    println(s"rdd2 partitions: ${rdd2.getNumPartitions}")
    println(s"rdd3 partitions: ${rdd3.getNumPartitions}")
    
    // Collect results
    val result = rdd3.mapValues(_.toList).collect()
    result.take(3).foreach(println)
```

---

## 2. RDD vs DataFrame vs Dataset {#rdd-dataframe-dataset}

### การเปรียบเทียบ API ทั้งสาม

| Feature | RDD | DataFrame | Dataset |
|---------|-----|-----------|---------|
| Type Safety | Compile-time | Runtime | Compile-time |
| Optimization | No catalyst | Catalyst optimizer | Catalyst optimizer |
| Performance | Baseline | Better | Better |
| Schema | None | Yes | Yes |
| API Style | Functional | SQL-like | Both |

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.types.*

case class Employee(
  id: Int,
  name: String,
  department: String,
  salary: Double,
  age: Int
)

object RDDDataFrameDatasetComparison:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("ComparisonDemo")
      .master("local[*]")
      .getOrCreate()
    
    import spark.implicits.*
    
    // สร้างข้อมูลตัวอย่าง
    val employees = Seq(
      Employee(1, "Alice",   "Engineering", 95000.0, 30),
      Employee(2, "Bob",     "Marketing",   75000.0, 35),
      Employee(3, "Charlie", "Engineering", 105000.0, 28),
      Employee(4, "Diana",   "HR",          65000.0, 42),
      Employee(5, "Eve",     "Engineering", 115000.0, 33),
      Employee(6, "Frank",   "Marketing",   80000.0, 29),
      Employee(7, "Grace",   "HR",          70000.0, 38),
      Employee(8, "Henry",   "Engineering", 90000.0, 25)
    )
    
    // === RDD API ===
    println("=== RDD API ===")
    val rdd = spark.sparkContext.parallelize(employees)
    
    val rddResult = rdd
      .filter(_.department == "Engineering")
      .map(e => (e.department, e.salary))
      .groupByKey()
      .mapValues(salaries => salaries.sum / salaries.size)
    
    rddResult.collect().foreach { case (dept, avgSalary) =>
      println(f"$dept: $avgSalary%.2f")
    }
    
    // === DataFrame API ===
    println("\n=== DataFrame API ===")
    val df = employees.toDF()
    
    val dfResult = df
      .filter(col("department") === "Engineering")
      .groupBy("department")
      .agg(avg("salary").as("avg_salary"))
    
    dfResult.show()
    
    // === Dataset API (Type-safe) ===
    println("=== Dataset API ===")
    val ds = employees.toDS()
    
    val dsResult = ds
      .filter(_.department == "Engineering")
      .groupByKey(_.department)
      .agg(
        org.apache.spark.sql.functions.avg($"salary").as("avg_salary")
          .as[Double]
      )
    
    dsResult.show()
    
    spark.stop()
```

### Dataset การดำเนินการขั้นสูง

```scala
import org.apache.spark.sql.{Dataset, SparkSession}
import org.apache.spark.sql.functions.*

object DatasetAdvanced:
  
  case class Order(
    orderId: String,
    customerId: String,
    productId: String,
    quantity: Int,
    price: Double,
    orderDate: String
  )
  
  case class Customer(
    customerId: String,
    name: String,
    city: String,
    tier: String
  )
  
  case class OrderSummary(
    customerId: String,
    customerName: String,
    city: String,
    totalOrders: Long,
    totalRevenue: Double,
    avgOrderValue: Double
  )
  
  def analyzeOrders(
    orders: Dataset[Order],
    customers: Dataset[Customer]
  )(implicit spark: SparkSession): Dataset[OrderSummary] =
    import spark.implicits.*
    
    orders
      .withColumn("revenue", col("quantity") * col("price"))
      .groupBy("customerId")
      .agg(
        count("orderId").as("totalOrders"),
        sum("revenue").as("totalRevenue"),
        avg("revenue").as("avgOrderValue")
      )
      .join(customers, "customerId")
      .select(
        col("customerId"),
        col("name").as("customerName"),
        col("city"),
        col("totalOrders"),
        col("totalRevenue"),
        col("avgOrderValue")
      )
      .as[OrderSummary]
      .orderBy(col("totalRevenue").desc)
  
  def main(args: Array[String]): Unit =
    implicit val spark: SparkSession = SparkSession.builder()
      .appName("DatasetAdvanced")
      .master("local[*]")
      .getOrCreate()
    
    import spark.implicits.*
    
    val orders = Seq(
      Order("O001", "C001", "P001", 2, 150.0, "2024-01-15"),
      Order("O002", "C002", "P002", 1, 300.0, "2024-01-16"),
      Order("O003", "C001", "P003", 3, 75.0,  "2024-01-17"),
      Order("O004", "C003", "P001", 5, 150.0, "2024-01-18"),
      Order("O005", "C002", "P004", 2, 200.0, "2024-01-19")
    ).toDS()
    
    val customers = Seq(
      Customer("C001", "Alice Johnson", "Bangkok",  "Gold"),
      Customer("C002", "Bob Smith",     "Chiang Mai","Silver"),
      Customer("C003", "Carol White",   "Phuket",    "Bronze")
    ).toDS()
    
    val summary = analyzeOrders(orders, customers)
    summary.show(truncate = false)
    
    spark.stop()
```

---

## 3. Spark SQL และ Catalyst Optimizer {#spark-sql-catalyst}

### Catalyst Optimizer Pipeline

Catalyst optimizer ทำงานผ่าน 4 ขั้นตอน:
1. **Analysis**: แก้ไข unresolved attributes และ relations
2. **Logical Optimization**: ใช้ rules เช่น predicate pushdown, constant folding
3. **Physical Planning**: เลือก physical operators
4. **Code Generation**: สร้าง Java bytecode

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*

object CatalystOptimizerDemo:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("CatalystDemo")
      .master("local[*]")
      // เปิด Adaptive Query Execution
      .config("spark.sql.adaptive.enabled", "true")
      .config("spark.sql.adaptive.coalescePartitions.enabled", "true")
      .getOrCreate()
    
    import spark.implicits.*
    
    // สร้างตาราง temp
    val salesDF = (1 to 100000).map { i =>
      (i, s"Product_${i % 100}", s"Region_${i % 5}", 
       (math.random * 1000).toLong, 
       s"2024-${(i % 12) + 1}-${(i % 28) + 1}")
    }.toDF("id", "product", "region", "amount", "date")
    
    salesDF.createOrReplaceTempView("sales")
    
    // Query ที่ Catalyst จะ optimize
    val query = """
      SELECT 
        region,
        product,
        SUM(amount) as total_amount,
        COUNT(*) as num_transactions,
        AVG(amount) as avg_amount
      FROM sales
      WHERE amount > 100
        AND region IN ('Region_0', 'Region_1', 'Region_2')
      GROUP BY region, product
      HAVING SUM(amount) > 5000
      ORDER BY total_amount DESC
      LIMIT 20
    """
    
    val result = spark.sql(query)
    
    // แสดง optimized plan
    println("=== Analyzed Logical Plan ===")
    result.explain("extended")
    
    // ตัวอย่าง Predicate Pushdown
    val optimizedQuery = salesDF
      .filter(col("amount") > 100)        // pushdown ก่อน join
      .filter(col("region").isin("Region_0", "Region_1"))
      .groupBy("region", "product")
      .agg(sum("amount").as("total"))
      .filter(col("total") > 5000)
    
    println("\n=== Physical Plan with Cost-Based Optimizer ===")
    optimizedQuery.explain("cost")
    
    result.show(5)
    spark.stop()
```

### Spark SQL UDFs และ UDAFs

```scala
import org.apache.spark.sql.{SparkSession, Row}
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.expressions.{UserDefinedFunction, Aggregator}
import org.apache.spark.sql.{Encoder, Encoders}

// Type-safe UDAF ด้วย Aggregator
case class AverageBuffer(sum: Double, count: Long)

class TypeSafeAverage extends Aggregator[Double, AverageBuffer, Double]:
  
  def zero: AverageBuffer = AverageBuffer(0.0, 0L)
  
  def reduce(b: AverageBuffer, a: Double): AverageBuffer =
    AverageBuffer(b.sum + a, b.count + 1)
  
  def merge(b1: AverageBuffer, b2: AverageBuffer): AverageBuffer =
    AverageBuffer(b1.sum + b2.sum, b1.count + b2.count)
  
  def finish(reduction: AverageBuffer): Double =
    if reduction.count == 0 then 0.0
    else reduction.sum / reduction.count
  
  def bufferEncoder: Encoder[AverageBuffer] = 
    Encoders.product[AverageBuffer]
  
  def outputEncoder: Encoder[Double] = 
    Encoders.scalaDouble

object UDFAndUDAFDemo:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("UDFDemo")
      .master("local[*]")
      .getOrCreate()
    
    import spark.implicits.*
    
    // UDF: ฟังก์ชันปกติ
    val categorizeAge: Int => String = age =>
      if age < 25 then "Young"
      else if age < 40 then "Middle"
      else "Senior"
    
    val categorizeAgeUDF = udf(categorizeAge)
    spark.udf.register("categorize_age", categorizeAge)
    
    // UDF สำหรับ Thai text processing
    val normalizeThaiText: String => String = text =>
      text.trim
        .toLowerCase
        .replaceAll("[^ก-๙a-z0-9\\s]", "")
    
    val normalizeUDF = udf(normalizeThaiText)
    
    val employeeDF = Seq(
      (1, "Alice", 25, 95000.0),
      (2, "Bob", 42, 75000.0),
      (3, "Charlie", 33, 105000.0),
      (4, "Diana", 55, 65000.0)
    ).toDF("id", "name", "age", "salary")
    
    val result = employeeDF
      .withColumn("age_category", categorizeAgeUDF(col("age")))
      .withColumn("salary_bucket", 
        when(col("salary") < 70000, "Low")
        .when(col("salary") < 90000, "Medium")
        .otherwise("High")
      )
    
    result.show()
    
    // ลงทะเบียน UDAF
    val typeSafeAvg = new TypeSafeAverage()
    val avgFunc = udaf(typeSafeAvg)
    
    val avgResult = employeeDF
      .agg(avgFunc(col("salary")).as("custom_avg_salary"))
    
    avgResult.show()
    
    // ใช้ SQL
    spark.sql("""
      SELECT 
        categorize_age(age) as category,
        COUNT(*) as count,
        AVG(salary) as avg_salary
      FROM employees
      GROUP BY categorize_age(age)
    """.stripMargin.replace("employees", "emp"))
    
    employeeDF.createOrReplaceTempView("emp")
    spark.sql(
      "SELECT categorize_age(age) as cat, COUNT(*) as cnt FROM emp GROUP BY cat"
    ).show()
    
    spark.stop()
```

---

## 4. กลยุทธ์การแบ่ง Partition {#partitioning-strategies}

### การทำความเข้าใจ Partitioning

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.Partitioner

object PartitioningStrategies:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("PartitioningDemo")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "8")
      .getOrCreate()
    
    import spark.implicits.*
    val sc = spark.sparkContext
    
    // === Hash Partitioning ===
    println("=== Hash Partitioning ===")
    val data = (1 to 100).map(i => (i % 10, i))
    val rdd = sc.parallelize(data)
    
    val hashPartitioned = rdd.partitionBy(
      new org.apache.spark.HashPartitioner(4)
    )
    println(s"Hash partitions: ${hashPartitioned.getNumPartitions}")
    hashPartitioned.mapPartitionsWithIndex { (idx, iter) =>
      iter.map(x => (idx, x))
    }.take(5).foreach(println)
    
    // === Range Partitioning ===
    println("\n=== Range Partitioning ===")
    val rangePartitioned = rdd.sortByKey(ascending = true, numPartitions = 4)
    println(s"Range partitions: ${rangePartitioned.getNumPartitions}")
    
    // === Custom Partitioner ===
    println("\n=== Custom Partitioner ===")
    
    class DepartmentPartitioner(numParts: Int) extends Partitioner:
      val departments = Map(
        "Engineering" -> 0,
        "Marketing"   -> 1,
        "HR"          -> 2,
        "Finance"     -> 3
      )
      
      override def numPartitions: Int = numParts
      
      override def getPartition(key: Any): Int =
        key match
          case dept: String => departments.getOrElse(dept, 0) % numParts
          case _ => 0
    
    val empData = Seq(
      ("Engineering", "Alice", 95000),
      ("Marketing", "Bob", 75000),
      ("HR", "Charlie", 65000),
      ("Engineering", "Diana", 105000),
      ("Finance", "Eve", 85000)
    )
    
    val empRDD = sc.parallelize(empData)
      .map { case (dept, name, salary) => (dept, (name, salary)) }
      .partitionBy(new DepartmentPartitioner(4))
    
    empRDD.mapPartitionsWithIndex { (idx, iter) =>
      iter.map { case (dept, info) => s"Partition $idx: $dept -> $info" }
    }.collect().foreach(println)
    
    // === DataFrame Partitioning ===
    println("\n=== DataFrame Partitioning ===")
    val salesDF = (1 to 10000).map { i =>
      (i, s"Region_${i % 5}", s"Product_${i % 20}", 
       (math.random * 1000).toInt)
    }.toDF("id", "region", "product", "amount")
    
    // Repartition (shuffle)
    val repartitioned = salesDF.repartition(4, col("region"))
    println(s"Repartitioned: ${repartitioned.rdd.getNumPartitions} partitions")
    
    // Coalesce (no shuffle - ลด partitions เท่านั้น)
    val coalesced = salesDF.coalesce(2)
    println(s"Coalesced: ${coalesced.rdd.getNumPartitions} partitions")
    
    // ดู partition distribution
    repartitioned.groupBy(spark_partition_id())
      .count()
      .orderBy("spark_partition_id()")
      .show()
    
    spark.stop()
```

### Skew Handling และ Salting

```scala
object SkewHandling:
  
  def handleSkewedJoin(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // จำลอง skewed data
    val hotKey = "Popular_Product"
    val orders = (1 to 100000).map { i =>
      val product = if i % 100 < 95 then hotKey  // 95% skew
                    else s"Product_$i"
      (i, product, (math.random * 100).toInt)
    }.toDF("orderId", "product", "quantity")
    
    val products = Seq(
      (hotKey, "Electronics", 999.0),
      ("Product_1", "Books", 29.0),
      ("Product_2", "Clothing", 59.0)
    ).toDF("product", "category", "price")
    
    // วิธี 1: Salting Technique
    println("=== Salting Technique ===")
    val saltFactor = 10
    
    val saltedOrders = orders
      .withColumn("salt", (rand() * saltFactor).cast("int"))
      .withColumn("salted_product", 
        concat(col("product"), lit("_"), col("salt")))
    
    val replicatedProducts = products
      .withColumn("salt", explode(array((0 until saltFactor).map(lit): _*)))
      .withColumn("salted_product",
        concat(col("product"), lit("_"), col("salt")))
    
    val saltedJoin = saltedOrders
      .join(replicatedProducts, "salted_product")
      .drop("salt", "salted_product")
    
    saltedJoin.groupBy("category")
      .agg(sum("quantity").as("total_qty"))
      .show()
    
    // วิธี 2: Adaptive Query Execution (AQE)
    spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
    spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
    spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256m")
    
    // AQE จะจัดการ skew โดยอัตโนมัติ
    val normalJoin = orders.join(products, "product")
    normalJoin.explain("extended")
```

---

## 5. Caching และ Persistence Levels {#caching-persistence}

### Storage Levels

```scala
import org.apache.spark.storage.StorageLevel
import org.apache.spark.sql.SparkSession

object CachingAndPersistence:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("CachingDemo")
      .master("local[*]")
      .config("spark.executor.memory", "4g")
      .getOrCreate()
    
    import spark.implicits.*
    
    // สร้างข้อมูลใหญ่
    val largeDF = (1 to 1000000).map { i =>
      (i, s"User_${i % 1000}", s"Product_${i % 500}",
       (math.random * 100).toInt, 
       (math.random * 1000.0))
    }.toDF("id", "user", "product", "quantity", "price")
    
    // Storage Levels
    println("=== Storage Levels ===")
    
    // MEMORY_ONLY: เก็บใน JVM heap เป็น Java objects
    val memoryOnly = largeDF.persist(StorageLevel.MEMORY_ONLY)
    
    // MEMORY_AND_DISK: เก็บใน memory ถ้าเต็มเก็บที่ disk
    val memAndDisk = largeDF.persist(StorageLevel.MEMORY_AND_DISK)
    
    // MEMORY_ONLY_SER: เก็บแบบ serialized (ประหยัด memory กว่า แต่ช้ากว่า)
    val memOnlySer = largeDF.persist(StorageLevel.MEMORY_ONLY_SER)
    
    // MEMORY_AND_DISK_SER: เก็บแบบ serialized, spill ไป disk ถ้าเต็ม
    val memAndDiskSer = largeDF.persist(StorageLevel.MEMORY_AND_DISK_SER)
    
    // OFF_HEAP: เก็บนอก JVM heap (ต้องการ Tungsten memory management)
    // val offHeap = largeDF.persist(StorageLevel.OFF_HEAP)
    
    // การทดสอบประสิทธิภาพ
    def benchmark(name: String, df: org.apache.spark.sql.DataFrame): Unit =
      val start = System.currentTimeMillis()
      df.count() // trigger first computation
      val firstRun = System.currentTimeMillis() - start
      
      val start2 = System.currentTimeMillis()
      df.count() // use cached
      val secondRun = System.currentTimeMillis() - start2
      
      println(f"$name: first=$firstRun%dms, cached=$secondRun%dms, speedup=${firstRun.toDouble/secondRun}%.1fx")
    
    benchmark("MEMORY_ONLY", memoryOnly)
    benchmark("MEMORY_AND_DISK", memAndDisk)
    
    // DataFrame cache API
    val cachedDF = largeDF.cache() // เหมือน MEMORY_AND_DISK
    
    // ดู cached tables
    spark.catalog.cacheTable("large_view")
    
    // Unpersist เมื่อไม่ใช้แล้ว
    memoryOnly.unpersist()
    memAndDisk.unpersist()
    memOnlySer.unpersist()
    memAndDiskSer.unpersist()
    cachedDF.unpersist()
    
    spark.stop()
  
  def adaptiveCaching(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Cache เฉพาะ DataFrame ที่ใช้หลายครั้ง
    val rawData = spark.range(1, 100000)
      .withColumn("category", (col("id") % 10).cast("string"))
      .withColumn("value", rand())
    
    // ตรวจสอบ storage info
    val cached = rawData.cache()
    cached.count() // materialize cache
    
    val storageInfo = spark.sparkContext.getRDDStorageInfo
    storageInfo.foreach { info =>
      println(s"RDD ${info.id}: ${info.memSize} bytes in memory, " +
              s"${info.diskSize} bytes on disk")
    }
    
    cached.unpersist()
```

---

## 6. Broadcast Variables และ Accumulators {#broadcast-accumulators}

### Broadcast Variables

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.broadcast.Broadcast
import scala.collection.Map

object BroadcastAndAccumulators:
  
  def broadcastDemo(spark: SparkSession): Unit =
    import spark.implicits.*
    val sc = spark.sparkContext
    
    // === Broadcast Variables ===
    println("=== Broadcast Variables ===")
    
    // Map ขนาดเล็กที่ใช้บ่อย - broadcast เพื่อหลีกเลี่ยง shuffle
    val productCatalog: Map[String, (String, Double)] = Map(
      "P001" -> ("Laptop", 999.0),
      "P002" -> ("Phone",  599.0),
      "P003" -> ("Tablet", 399.0),
      "P004" -> ("Watch",  299.0),
      "P005" -> ("Earbuds", 149.0)
    )
    
    // Broadcast the lookup table
    val broadcastCatalog: Broadcast[Map[String, (String, Double)]] =
      sc.broadcast(productCatalog)
    
    val orders = sc.parallelize(Seq(
      ("O001", "P001", 2),
      ("O002", "P003", 1),
      ("O003", "P002", 3),
      ("O004", "P005", 5),
      ("O005", "P001", 1)
    ))
    
    // ใช้ broadcast variable ใน transformation
    val orderDetails = orders.map { case (orderId, productId, qty) =>
      val (name, price) = broadcastCatalog.value.getOrElse(
        productId, ("Unknown", 0.0)
      )
      (orderId, productId, name, qty, price, qty * price)
    }
    
    orderDetails.collect().foreach { case (oid, pid, name, qty, price, total) =>
      println(f"$oid | $pid | $name | $qty | $price%.2f | $total%.2f")
    }
    
    // Broadcast ใน DataFrame API
    val productDF = productCatalog.toSeq
      .map { case (id, (name, price)) => (id, name, price) }
      .toDF("productId", "name", "price")
    
    val orderDF = Seq(
      ("O001", "P001", 2), ("O002", "P003", 1)
    ).toDF("orderId", "productId", "qty")
    
    // Broadcast join - บอก Spark ให้ใช้ broadcast join
    import org.apache.spark.sql.functions.broadcast
    val joinResult = orderDF.join(
      broadcast(productDF), "productId"
    )
    joinResult.show()
    
    // Destroy broadcast เมื่อไม่ใช้
    broadcastCatalog.destroy()
    
    // === Accumulators ===
    println("\n=== Accumulators ===")
    
    val errorCount = sc.longAccumulator("Error Count")
    val totalRevenue = sc.doubleAccumulator("Total Revenue")
    
    // Custom Accumulator
    import org.apache.spark.util.AccumulatorV2
    
    class SetAccumulator[T] extends AccumulatorV2[T, Set[T]]:
      private var _set = Set.empty[T]
      
      def isZero: Boolean = _set.isEmpty
      def copy(): SetAccumulator[T] = 
        val newAcc = new SetAccumulator[T]
        newAcc._set = _set
        newAcc
      def reset(): Unit = _set = Set.empty[T]
      def add(v: T): Unit = _set += v
      def merge(other: AccumulatorV2[T, Set[T]]): Unit = 
        _set ++= other.value
      def value: Set[T] = _set
    
    val uniqueProducts = new SetAccumulator[String]
    sc.register(uniqueProducts, "Unique Products")
    
    // ใช้ accumulators ใน RDD actions
    val transactionData = sc.parallelize(Seq(
      ("O001", "P001", 2, 999.0, "success"),
      ("O002", "P003", 1, 399.0, "error"),
      ("O003", "P002", 3, 599.0, "success"),
      ("O004", "INVALID", 1, -1.0, "error"),
      ("O005", "P001", 1, 999.0, "success")
    ))
    
    val validOrders = transactionData.filter { case (_, _, _, amount, status) =>
      if status == "error" then
        errorCount.add(1)
        false
      else
        totalRevenue.add(amount)
        true
    }
    
    validOrders.foreach { case (oid, pid, qty, price, _) =>
      uniqueProducts.add(pid)
    }
    
    println(s"Error Count: ${errorCount.value}")
    println(f"Total Revenue: ${totalRevenue.value}%.2f")
    println(s"Unique Products: ${uniqueProducts.value}")
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("BroadcastAccumulatorDemo")
      .master("local[*]")
      .getOrCreate()
    
    broadcastDemo(spark)
    spark.stop()
```

---

## 7. Spark Streaming with Structured Streaming {#spark-streaming}

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.streaming.{StreamingQuery, Trigger}
import java.util.concurrent.TimeUnit

object StructuredStreamingDemo:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("StructuredStreamingDemo")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits.*
    
    // อ่าน streaming data จาก socket (สำหรับ demo)
    val socketStream = spark.readStream
      .format("socket")
      .option("host", "localhost")
      .option("port", 9999)
      .load()
    
    // แปลงข้อมูล
    val wordCounts = socketStream
      .as[String]
      .flatMap(_.split("\\s+"))
      .groupBy("value")
      .count()
      .orderBy(col("count").desc)
    
    // อ่านจาก Rate source (สำหรับทดสอบ)
    val rateStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "100")
      .load()
    
    // ประมวลผล rate stream
    val processedStream = rateStream
      .withColumn("category", (col("value") % 5).cast("string"))
      .withColumn("amount", (rand() * 1000).cast("double"))
      .groupBy(
        window(col("timestamp"), "10 seconds", "5 seconds"),
        col("category")
      )
      .agg(
        count("*").as("count"),
        sum("amount").as("total_amount"),
        avg("amount").as("avg_amount")
      )
    
    // Output ไป console
    val query: StreamingQuery = processedStream.writeStream
      .format("console")
      .outputMode("update")
      .option("truncate", "false")
      .option("numRows", "20")
      .trigger(Trigger.ProcessingTime(10, TimeUnit.SECONDS))
      .start()
    
    // รอ 60 วินาที แล้วหยุด
    query.awaitTermination(60000)
    
    spark.stop()
```

---

## 8. ตัวอย่าง ML Pipeline สมบูรณ์ {#ml-pipeline}

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.ml.{Pipeline, PipelineModel}
import org.apache.spark.ml.classification.{RandomForestClassifier, LogisticRegression}
import org.apache.spark.ml.evaluation.{BinaryClassificationEvaluator, MulticlassClassificationEvaluator}
import org.apache.spark.ml.feature.*
import org.apache.spark.ml.tuning.{CrossValidator, ParamGridBuilder}
import org.apache.spark.sql.functions.*

object CompleteMLPipeline:
  
  case class CustomerChurn(
    customerId: String,
    tenure: Int,
    monthlyCharges: Double,
    totalCharges: Double,
    contract: String,
    internetService: String,
    techSupport: String,
    paymentMethod: String,
    numTickets: Int,
    churn: Int  // 0 = No, 1 = Yes
  )
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("MLPipelineDemo")
      .master("local[*]")
      .getOrCreate()
    
    import spark.implicits.*
    
    // สร้างข้อมูล synthetic
    val random = new scala.util.Random(42)
    val data = (1 to 5000).map { i =>
      val tenure = random.nextInt(72) + 1
      val monthlyCharges = 20.0 + random.nextDouble() * 80.0
      val contract = Seq("Month-to-month", "One year", "Two year")(random.nextInt(3))
      val internetService = Seq("DSL", "Fiber optic", "No")(random.nextInt(3))
      val techSupport = Seq("Yes", "No")(random.nextInt(2))
      val numTickets = random.nextInt(10)
      
      // Churn probability based on features
      val churnProb = 
        (if contract == "Month-to-month" then 0.3 else 0.0) +
        (if monthlyCharges > 70 then 0.2 else 0.0) +
        (if numTickets > 5 then 0.2 else 0.0) +
        (if tenure < 12 then 0.15 else 0.0)
      
      val churn = if random.nextDouble() < churnProb then 1 else 0
      
      CustomerChurn(
        s"C${i.toString.padTo(5, '0')}",
        tenure, monthlyCharges, monthlyCharges * tenure,
        contract, internetService, techSupport,
        Seq("Credit card", "Bank transfer", "Electronic check")(random.nextInt(3)),
        numTickets, churn
      )
    }.toDS().toDF()
    
    println("=== Dataset Statistics ===")
    data.describe("tenure", "monthlyCharges", "numTickets").show()
    data.groupBy("churn").count().show()
    
    // แบ่ง train/test
    val Array(trainDF, testDF) = data.randomSplit(Array(0.8, 0.2), seed = 42)
    
    // === Feature Engineering ===
    
    // String Indexers สำหรับ categorical columns
    val contractIndexer = new StringIndexer()
      .setInputCol("contract")
      .setOutputCol("contractIndex")
      .setHandleInvalid("keep")
    
    val internetIndexer = new StringIndexer()
      .setInputCol("internetService")
      .setOutputCol("internetIndex")
      .setHandleInvalid("keep")
    
    val techSupportIndexer = new StringIndexer()
      .setInputCol("techSupport")
      .setOutputCol("techSupportIndex")
      .setHandleInvalid("keep")
    
    val paymentIndexer = new StringIndexer()
      .setInputCol("paymentMethod")
      .setOutputCol("paymentIndex")
      .setHandleInvalid("keep")
    
    // One-Hot Encoding
    val encoder = new OneHotEncoder()
      .setInputCols(Array("contractIndex", "internetIndex", 
                          "techSupportIndex", "paymentIndex"))
      .setOutputCols(Array("contractVec", "internetVec",
                           "techSupportVec", "paymentVec"))
    
    // Normalization
    val scaler = new StandardScaler()
      .setInputCol("numericFeatures")
      .setOutputCol("scaledFeatures")
      .setWithMean(true)
      .setWithStd(true)
    
    // Assembly ก่อน scale
    val assemblerForScale = new VectorAssembler()
      .setInputCols(Array("tenure", "monthlyCharges", 
                          "totalCharges", "numTickets"))
      .setOutputCol("numericFeatures")
    
    // Final Assembler
    val finalAssembler = new VectorAssembler()
      .setInputCols(Array("scaledFeatures", "contractVec",
                          "internetVec", "techSupportVec", "paymentVec"))
      .setOutputCol("features")
      .setHandleInvalid("keep")
    
    // === Model Definition ===
    val rf = new RandomForestClassifier()
      .setLabelCol("churn")
      .setFeaturesCol("features")
      .setNumTrees(100)
      .setMaxDepth(10)
      .setSeed(42)
    
    val lr = new LogisticRegression()
      .setLabelCol("churn")
      .setFeaturesCol("features")
      .setMaxIter(100)
      .setRegParam(0.01)
    
    // === Pipeline ===
    val pipeline = new Pipeline().setStages(Array(
      contractIndexer, internetIndexer,
      techSupportIndexer, paymentIndexer,
      encoder, assemblerForScale, scaler,
      finalAssembler, rf
    ))
    
    // === Hyperparameter Tuning ===
    val paramGrid = new ParamGridBuilder()
      .addGrid(rf.numTrees, Array(50, 100, 200))
      .addGrid(rf.maxDepth, Array(5, 10, 15))
      .build()
    
    val evaluator = new BinaryClassificationEvaluator()
      .setLabelCol("churn")
      .setRawPredictionCol("rawPrediction")
      .setMetricName("areaUnderROC")
    
    val cv = new CrossValidator()
      .setEstimator(pipeline)
      .setEvaluator(evaluator)
      .setEstimatorParamMaps(paramGrid)
      .setNumFolds(3)
      .setParallelism(2)
    
    println("=== Training Model ===")
    val cvModel = cv.fit(trainDF)
    
    // === Evaluation ===
    println("=== Evaluating Model ===")
    val predictions = cvModel.transform(testDF)
    
    val auc = evaluator.evaluate(predictions)
    println(f"AUC-ROC: $auc%.4f")
    
    val multiEval = new MulticlassClassificationEvaluator()
      .setLabelCol("churn")
      .setPredictionCol("prediction")
    
    val accuracy = multiEval.setMetricName("accuracy").evaluate(predictions)
    val f1 = multiEval.setMetricName("f1").evaluate(predictions)
    val precision = multiEval.setMetricName("weightedPrecision").evaluate(predictions)
    val recall = multiEval.setMetricName("weightedRecall").evaluate(predictions)
    
    println(f"Accuracy:  $accuracy%.4f")
    println(f"F1-Score:  $f1%.4f")
    println(f"Precision: $precision%.4f")
    println(f"Recall:    $recall%.4f")
    
    // Feature Importance
    val bestModel = cvModel.bestModel.asInstanceOf[PipelineModel]
    val rfModel = bestModel.stages.last
      .asInstanceOf[org.apache.spark.ml.classification.RandomForestClassificationModel]
    
    println("\n=== Feature Importance (Top 10) ===")
    rfModel.featureImportances.toArray.zipWithIndex
      .sortBy(-_._1)
      .take(10)
      .foreach { case (importance, idx) =>
        println(f"Feature $idx: $importance%.4f")
      }
    
    // Confusion Matrix
    println("\n=== Confusion Matrix ===")
    predictions
      .groupBy("churn", "prediction")
      .count()
      .orderBy("churn", "prediction")
      .show()
    
    // บันทึก Model
    val modelPath = "/tmp/churn_model"
    cvModel.write.overwrite().save(modelPath)
    println(s"Model saved to $modelPath")
    
    spark.stop()
```

---

## 9. สรุป {#summary}

### สิ่งที่ได้เรียนรู้ในบทนี้

1. **Spark Architecture**: เข้าใจ Driver, Executors, DAG Scheduler และวิธีที่ Spark จัดการ stages
2. **API Levels**: เลือกใช้ RDD, DataFrame, หรือ Dataset ตามความต้องการ
3. **Catalyst Optimizer**: เข้าใจ optimization pipeline และวิธีเขียน query ที่มีประสิทธิภาพ
4. **Partitioning**: จัดการ partitions อย่างเหมาะสมเพื่อประสิทธิภาพสูงสุด
5. **Caching**: เลือก storage level ที่เหมาะสมกับ use case
6. **Broadcast & Accumulators**: ใช้เครื่องมือเหล่านี้อย่างถูกต้อง
7. **Structured Streaming**: สร้าง real-time processing pipeline
8. **ML Pipeline**: สร้าง end-to-end machine learning workflow

### Best Practices

```scala
// 1. ใช้ DataFrame/Dataset แทน RDD เมื่อเป็นไปได้
// 2. Cache ข้อมูลที่ใช้หลายครั้ง
// 3. ตรวจสอบ execution plan ก่อน submit job ใหญ่
// 4. ใช้ broadcast join สำหรับ small table
// 5. จัดการ skew ด้วย salting หรือ AQE
// 6. ตั้งค่า partition จำนวนที่เหมาะสม (2-3x จำนวน cores)
// 7. ใช้ Kryo serialization สำหรับ performance ที่ดีขึ้น

val spark = SparkSession.builder()
  .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
  .config("spark.kryo.registrationRequired", "false")
  .getOrCreate()
```

---

*[← Part 69: Apache Kafka](part-69-kafka.md) | [Part 71: Spark Structured Streaming →](part-71-spark-streaming.md)*
