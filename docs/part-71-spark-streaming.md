# ส่วนที่ 71: Spark Structured Streaming - การประมวลผลข้อมูลแบบ Real-time

## สารบัญ

1. [แนวคิด Streaming: Micro-batch vs Continuous](#streaming-concepts)
2. [อ่านข้อมูลจาก Kafka ด้วย Spark](#kafka-spark)
3. [Watermarks และการจัดการข้อมูลล่าช้า](#watermarks-late-data)
4. [Window Operations: Tumbling, Sliding, Session](#window-operations)
5. [Stateful Operations ด้วย flatMapGroupsWithState](#stateful-operations)
6. [Output Modes: Append, Complete, Update](#output-modes)
7. [Checkpointing และ Fault Tolerance](#checkpointing)
8. [Streaming Analytics Pipeline สมบูรณ์](#streaming-pipeline)
9. [สรุป](#summary)

---

## 1. แนวคิด Streaming: Micro-batch vs Continuous {#streaming-concepts}

### Micro-batch Processing

Structured Streaming ใช้ micro-batch model เป็นค่าเริ่มต้น โดยแต่ละ batch จะประมวลผลข้อมูลที่มาถึงในช่วงเวลาหนึ่ง

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-sql"              % "3.5.0",
  "org.apache.spark" %% "spark-sql-kafka-0-10"   % "3.5.0",
  "org.apache.spark" %% "spark-streaming"        % "3.5.0",
  "io.delta"         %% "delta-spark"            % "3.0.0"
)
```

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.streaming.{StreamingQuery, Trigger}
import java.util.concurrent.TimeUnit

object MicroBatchVsContinuous:
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("MicroBatchDemo")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits.*
    
    // Rate source สำหรับทดสอบ
    val stream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "1000")
      .option("numPartitions", "4")
      .load()
    
    val processedStream = stream
      .withColumn("category", (col("value") % 5).cast("string"))
      .withColumn("amount", (rand() * 1000).cast("double"))
    
    // === Trigger Types ===
    
    // 1. Default (500ms micro-batch)
    val defaultTrigger = processedStream.writeStream
      .format("console")
      .trigger(Trigger.ProcessingTime(0))  // as fast as possible
      .outputMode("append")
    
    // 2. Fixed interval micro-batch
    val fixedTrigger = processedStream.writeStream
      .format("console")
      .trigger(Trigger.ProcessingTime(5, TimeUnit.SECONDS))
      .outputMode("append")
    
    // 3. Once trigger (process all available data once)
    val onceTrigger = processedStream.writeStream
      .format("console")
      .trigger(Trigger.Once())
      .outputMode("append")
    
    // 4. Available Now (like Once but processes all batches)
    val availableNow = processedStream.writeStream
      .format("console")
      .trigger(Trigger.AvailableNow())
      .outputMode("append")
    
    // 5. Continuous processing (low latency, ~1ms)
    // หมายเหตุ: Continuous mode รองรับเฉพาะบาง operations
    val continuousStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "100")
      .load()
      .select(col("timestamp"), col("value"))
    
    // Continuous trigger (experimental)
    // val continuousTrigger = continuousStream.writeStream
    //   .format("console")
    //   .trigger(Trigger.Continuous(1, TimeUnit.SECONDS))
    //   .outputMode("append")
    //   .start()
    
    println("=== Micro-batch Processing Demo ===")
    
    // รัน fixed interval trigger
    val query: StreamingQuery = processedStream
      .groupBy(col("category"))
      .agg(
        count("*").as("count"),
        sum("amount").as("total")
      )
      .writeStream
      .format("console")
      .outputMode("complete")
      .trigger(Trigger.ProcessingTime(5, TimeUnit.SECONDS))
      .start()
    
    // แสดง streaming query status
    Thread.sleep(3000)
    println(s"Query Status: ${query.status}")
    println(s"Recent Progress: ${query.recentProgress.take(2).mkString("\n")}")
    
    query.awaitTermination(20000)
    spark.stop()
```

### Stream vs Batch Processing

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*

object StreamVsBatchProcessing:
  
  // ฟังก์ชันที่ใช้ได้ทั้ง batch และ streaming
  def processData(df: org.apache.spark.sql.DataFrame): org.apache.spark.sql.DataFrame =
    df
      .filter(col("amount") > 0)
      .withColumn("revenue_bucket",
        when(col("amount") < 100, "Small")
        .when(col("amount") < 500, "Medium")
        .otherwise("Large")
      )
      .withColumn("processed_at", current_timestamp())
  
  def runBatch(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Batch processing
    val batchData = Seq(
      ("T001", 150.0, "2024-01-01"),
      ("T002", 50.0,  "2024-01-01"),
      ("T003", 750.0, "2024-01-02")
    ).toDF("id", "amount", "date")
    
    val batchResult = processData(batchData)
    batchResult.show()
    batchResult.write.mode("overwrite").parquet("/tmp/batch_output")
  
  def runStreaming(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Streaming processing (ใช้ฟังก์ชันเดียวกัน)
    val streamData = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "10")
      .load()
      .withColumn("amount", (rand() * 1000).cast("double"))
      .withColumnRenamed("value", "id")
    
    val streamResult = processData(streamData)
    
    val query = streamResult.writeStream
      .format("parquet")
      .option("path", "/tmp/stream_output")
      .option("checkpointLocation", "/tmp/stream_checkpoint")
      .outputMode("append")
      .start()
    
    query.awaitTermination(30000)
```

---

## 2. อ่านข้อมูลจาก Kafka ด้วย Spark {#kafka-spark}

### Kafka Integration

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.types.*

object SparkKafkaIntegration:
  
  // Schema สำหรับ event data
  val eventSchema: StructType = new StructType()
    .add("eventId", StringType, nullable = false)
    .add("userId", StringType, nullable = false)
    .add("eventType", StringType, nullable = false)
    .add("productId", StringType, nullable = true)
    .add("amount", DoubleType, nullable = true)
    .add("timestamp", LongType, nullable = false)
    .add("metadata", MapType(StringType, StringType), nullable = true)
  
  def readFromKafka(spark: SparkSession): org.apache.spark.sql.DataFrame =
    spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "user-events")           // subscribe to one topic
      // .option("subscribePattern", "events-.*")    // subscribe with regex
      // .option("assign", """{"topic1":[0,1]}""")   // assign specific partitions
      .option("startingOffsets", "latest")           // หรือ "earliest", json offset
      .option("endingOffsets", "latest")             // สำหรับ batch only
      .option("maxOffsetsPerTrigger", 10000)          // จำกัด offsets per batch
      .option("failOnDataLoss", "false")             // ถ้า offset หาย ไม่หยุด
      .option("kafka.security.protocol", "PLAINTEXT") // SASL_SSL สำหรับ production
      .load()
  
  def parseKafkaMessages(kafkaDF: org.apache.spark.sql.DataFrame): 
      org.apache.spark.sql.DataFrame =
    kafkaDF
      // Kafka schema: key, value, topic, partition, offset, timestamp, timestampType
      .select(
        col("key").cast("string").as("key"),
        col("value").cast("string").as("json_value"),
        col("topic"),
        col("partition"),
        col("offset"),
        col("timestamp").as("kafka_timestamp")
      )
      // Parse JSON payload
      .withColumn("event", from_json(col("json_value"), eventSchema))
      .select(
        col("key"),
        col("topic"),
        col("partition"),
        col("offset"),
        col("kafka_timestamp"),
        col("event.*")
      )
      // แปลง timestamp จาก unix millis เป็น timestamp type
      .withColumn("event_time", 
        (col("timestamp") / 1000).cast("timestamp"))
      .drop("timestamp")
  
  def writeToKafka(
    processedDF: org.apache.spark.sql.DataFrame,
    outputTopic: String,
    checkpointPath: String
  ): org.apache.spark.sql.streaming.StreamingQuery =
    // แปลง back เป็น JSON สำหรับ Kafka
    val kafkaOutput = processedDF
      .select(
        col("userId").as("key"),
        to_json(struct("*")).as("value")
      )
    
    kafkaOutput.writeStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("topic", outputTopic)
      .option("checkpointLocation", checkpointPath)
      .outputMode("append")
      .start()
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("KafkaIntegration")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "8")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits.*
    
    // ในการทดสอบ ใช้ rate source แทน Kafka
    val mockKafkaData = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "100")
      .load()
      .select(
        col("value").cast("string").as("key"),
        to_json(struct(
          col("value").as("eventId"),
          (col("value") % 100).cast("string").as("userId"),
          array(lit("click"), lit("purchase"), lit("view"))
            .getItem((col("value") % 3).cast("int")).as("eventType"),
          (rand() * 500).as("amount"),
          (unix_timestamp() * 1000).as("timestamp")
        )).as("value"),
        lit("user-events").as("topic"),
        (col("value") % 4).cast("int").as("partition"),
        col("value").as("offset"),
        col("timestamp").as("kafka_timestamp")
      )
    
    // Parse และประมวลผล
    val parsedStream = mockKafkaData
      .withColumn("event", from_json(col("value"), eventSchema))
      .select(
        col("key"),
        col("topic"),
        col("partition"),
        col("offset"),
        col("kafka_timestamp"),
        col("event.userId"),
        col("event.eventType"),
        col("event.amount"),
        (col("kafka_timestamp").cast("long") * 1000).as("event_time")
      )
    
    // Real-time analytics
    val realtimeStats = parsedStream
      .withColumn("event_time_ts", col("kafka_timestamp"))
      .groupBy(
        window(col("event_time_ts"), "30 seconds", "10 seconds"),
        col("eventType")
      )
      .agg(
        count("*").as("event_count"),
        sum("amount").as("total_amount"),
        approx_count_distinct("userId").as("unique_users")
      )
    
    val query = realtimeStats.writeStream
      .format("console")
      .outputMode("update")
      .option("truncate", "false")
      .trigger(org.apache.spark.sql.streaming.Trigger.ProcessingTime("10 seconds"))
      .start()
    
    query.awaitTermination(60000)
    spark.stop()
```

### Multi-topic Processing

```scala
object MultiTopicProcessing:
  
  def processMultipleTopics(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // อ่านหลาย topics พร้อมกัน
    val multiTopicStream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribePattern", "events-.*")  // regex สำหรับหลาย topics
      .option("startingOffsets", "latest")
      .load()
    
    // Route ตาม topic
    val orderEvents = multiTopicStream
      .filter(col("topic") === "events-orders")
      .select(
        col("value").cast("string").as("payload"),
        col("timestamp")
      )
    
    val userEvents = multiTopicStream
      .filter(col("topic") === "events-users")
      .select(
        col("value").cast("string").as("payload"),
        col("timestamp")
      )
    
    // ประมวลผลแยกกัน (parallel queries)
    val orderQuery = orderEvents.writeStream
      .format("console")
      .outputMode("append")
      .queryName("order-processor")
      .start()
    
    val userQuery = userEvents.writeStream
      .format("console")
      .outputMode("append")
      .queryName("user-processor")
      .start()
    
    // รอทุก queries
    spark.streams.awaitAnyTermination()
```

---

## 3. Watermarks และการจัดการข้อมูลล่าช้า {#watermarks-late-data}

### Watermark Concepts

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.streaming.Trigger
import java.sql.Timestamp

object WatermarkDemo:
  
  case class Event(
    userId: String,
    eventType: String,
    amount: Double,
    eventTime: Timestamp
  )
  
  def demonstrateWatermarks(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // จำลองข้อมูลที่มาล่าช้า
    val eventStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "10")
      .load()
      .select(
        col("value").cast("string").as("userId"),
        (col("timestamp") - 
          // จำลองข้อมูลล่าช้าสุ่มระหว่าง 0-15 นาที
          expr("INTERVAL " + "(rand() * 15)".toString + " MINUTES")
        ).as("eventTime"),
        (rand() * 100).as("amount")
      )
    
    // Watermark: ยอมรับข้อมูลที่ล่าช้าได้ไม่เกิน 10 นาที
    val withWatermark = eventStream
      .withWatermark("eventTime", "10 minutes")
      .groupBy(
        window(col("eventTime"), "5 minutes"),
        col("userId")
      )
      .agg(
        count("*").as("event_count"),
        sum("amount").as("total_amount")
      )
    
    val query = withWatermark.writeStream
      .format("console")
      .outputMode("update")  // update mode รองรับ late data
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    query.awaitTermination(120000)
  
  def watermarkAndJoin(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Stream-Stream Join พร้อม Watermark
    val impressions = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "50")
      .load()
      .select(
        col("value").as("impressionId"),
        (col("value") % 1000).cast("string").as("adId"),
        col("timestamp").as("impressionTime")
      )
      .withWatermark("impressionTime", "10 minutes")
    
    val clicks = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "5")
      .load()
      .select(
        col("value").as("clickId"),
        // simulate click some impressions with delay
        ((col("value") * 13) % 1000).cast("string").as("adId"),
        // click happens 0-5 minutes after impression
        (col("timestamp") + expr("INTERVAL 2 MINUTES")).as("clickTime")
      )
      .withWatermark("clickTime", "10 minutes")
    
    // Stream-Stream Inner Join
    val clickThroughRate = impressions.join(
      clicks,
      impressions("adId") === clicks("adId") &&
      impressions("impressionTime") <= clicks("clickTime") &&
      impressions("impressionTime") >= 
        clicks("clickTime") - expr("INTERVAL 5 MINUTES"),
      joinType = "inner"
    )
    
    val query = clickThroughRate.writeStream
      .format("console")
      .outputMode("append")
      .option("truncate", "false")
      .start()
    
    query.awaitTermination(60000)
```

---

## 4. Window Operations {#window-operations}

### Tumbling, Sliding, และ Session Windows

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*

object WindowOperationsDemo:
  
  def demonstrateWindows(spark: SparkSession): Unit =
    import spark.implicits.*
    
    val sensorStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "100")
      .load()
      .select(
        (col("value") % 10).cast("string").as("sensorId"),
        col("timestamp").as("readingTime"),
        (rand() * 100 + 20).as("temperature"),
        (rand() * 1000 + 500).as("pressure")
      )
      .withWatermark("readingTime", "5 minutes")
    
    // === 1. Tumbling Window (ไม่ซ้อนทับ) ===
    val tumblingWindow = sensorStream
      .groupBy(
        window(col("readingTime"), "1 minute"),  // 1-min tumbling window
        col("sensorId")
      )
      .agg(
        avg("temperature").as("avg_temp"),
        max("temperature").as("max_temp"),
        min("temperature").as("min_temp"),
        count("*").as("reading_count")
      )
      .select(
        col("window.start").as("window_start"),
        col("window.end").as("window_end"),
        col("sensorId"),
        col("avg_temp"),
        col("max_temp"),
        col("min_temp"),
        col("reading_count")
      )
    
    // === 2. Sliding Window (ซ้อนทับได้) ===
    val slidingWindow = sensorStream
      .groupBy(
        window(
          col("readingTime"),
          "5 minutes",   // window duration
          "1 minute"     // slide duration (ทุก 1 นาที)
        ),
        col("sensorId")
      )
      .agg(
        avg("temperature").as("rolling_avg_temp"),
        stddev("temperature").as("temp_stddev"),
        percentile_approx(col("temperature"), lit(0.95)).as("p95_temp")
      )
    
    // === 3. Session Window (ตาม activity gap) ===
    val sessionWindow = sensorStream
      .groupBy(
        session_window(
          col("readingTime"),
          "2 minutes"  // session timeout - ถ้าไม่มีข้อมูล 2 นาที ปิด session
        ),
        col("sensorId")
      )
      .agg(
        count("*").as("session_events"),
        avg("temperature").as("session_avg_temp"),
        (max("readingTime").cast("long") - 
         min("readingTime").cast("long")).as("session_duration_ms")
      )
    
    // รัน tumbling window query
    val query = tumblingWindow.writeStream
      .format("console")
      .outputMode("update")
      .option("truncate", "false")
      .trigger(org.apache.spark.sql.streaming.Trigger.ProcessingTime("30 seconds"))
      .start()
    
    query.awaitTermination(90000)
  
  def advancedWindowAggregations(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Time-based aggregations ขั้นสูง
    val transactions = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "50")
      .load()
      .select(
        (col("value") % 100).cast("string").as("accountId"),
        col("timestamp").as("txTime"),
        (rand() * 10000).as("amount"),
        array(lit("credit"), lit("debit"))
          .getItem((col("value") % 2).cast("int")).as("txType")
      )
      .withWatermark("txTime", "10 minutes")
    
    // Multiple aggregations ในหน้าต่างเดียว
    val multiAgg = transactions
      .groupBy(
        window(col("txTime"), "5 minutes"),
        col("accountId")
      )
      .agg(
        sum(when(col("txType") === "credit", col("amount")).otherwise(0))
          .as("total_credits"),
        sum(when(col("txType") === "debit", col("amount")).otherwise(0))
          .as("total_debits"),
        count("*").as("tx_count"),
        approx_count_distinct("amount").as("unique_amounts"),
        // Detect anomalies
        (count("*") > 100 || sum("amount") > 50000).as("is_suspicious")
      )
    
    val query = multiAgg.writeStream
      .format("console")
      .outputMode("update")
      .option("truncate", "false")
      .start()
    
    query.awaitTermination(60000)
```

---

## 5. Stateful Operations ด้วย flatMapGroupsWithState {#stateful-operations}

### การจัดการ State ที่ซับซ้อน

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.streaming.{GroupState, GroupStateTimeout, OutputMode}
import java.sql.Timestamp

object StatefulOperationsDemo:
  
  // Input event
  case class UserEvent(
    userId: String,
    eventType: String,
    timestamp: Timestamp,
    amount: Double
  )
  
  // State ที่เก็บไว้สำหรับแต่ละ user
  case class UserState(
    userId: String,
    sessionStart: Long,
    lastEventTime: Long,
    eventCount: Int,
    totalAmount: Double,
    events: List[String]
  )
  
  // Output
  case class UserSession(
    userId: String,
    sessionId: String,
    sessionStart: Timestamp,
    sessionEnd: Timestamp,
    durationMinutes: Double,
    eventCount: Int,
    totalAmount: Double,
    eventSequence: String
  )
  
  // ฟังก์ชัน stateful update
  def updateUserState(
    userId: String,
    events: Iterator[UserEvent],
    state: GroupState[UserState]
  ): Iterator[UserSession] =
    
    val SESSION_TIMEOUT_MS = 30 * 60 * 1000L // 30 minutes
    
    // ตรวจสอบ timeout
    if state.hasTimedOut then
      val timedOutState = state.get
      state.remove()
      
      // Emit session เมื่อ timeout
      val session = UserSession(
        userId = timedOutState.userId,
        sessionId = s"${timedOutState.userId}_${timedOutState.sessionStart}",
        sessionStart = new Timestamp(timedOutState.sessionStart),
        sessionEnd = new Timestamp(timedOutState.lastEventTime),
        durationMinutes = (timedOutState.lastEventTime - timedOutState.sessionStart) / 60000.0,
        eventCount = timedOutState.eventCount,
        totalAmount = timedOutState.totalAmount,
        eventSequence = timedOutState.events.mkString(" -> ")
      )
      return Iterator(session)
    
    // ประมวลผล events ใหม่
    val newEvents = events.toList.sortBy(_.timestamp.getTime)
    
    if newEvents.isEmpty then
      return Iterator.empty
    
    val currentState = if state.exists then
      state.get
    else
      UserState(
        userId = userId,
        sessionStart = newEvents.head.timestamp.getTime,
        lastEventTime = newEvents.head.timestamp.getTime,
        eventCount = 0,
        totalAmount = 0.0,
        events = List.empty
      )
    
    // อัพเดท state
    val updatedState = newEvents.foldLeft(currentState) { (s, event) =>
      s.copy(
        lastEventTime = event.timestamp.getTime,
        eventCount = s.eventCount + 1,
        totalAmount = s.totalAmount + event.amount,
        events = s.events :+ event.eventType
      )
    }
    
    state.update(updatedState)
    
    // Set timeout สำหรับ session
    state.setEventTimeTimeout(
      new Timestamp(updatedState.lastEventTime + SESSION_TIMEOUT_MS)
    )
    
    // ไม่ emit ระหว่าง session ดำเนินอยู่
    Iterator.empty
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("StatefulOperationsDemo")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "4")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits.*
    
    // สร้าง event stream
    val eventStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "20")
      .load()
      .select(
        (col("value") % 50).cast("string").as("userId"),
        array(lit("login"), lit("view"), lit("click"), 
              lit("purchase"), lit("logout"))
          .getItem((col("value") % 5).cast("int")).as("eventType"),
        col("timestamp"),
        (rand() * 100).as("amount")
      )
      .as[UserEvent]
      .withWatermark("timestamp", "1 hour")
    
    // ใช้ flatMapGroupsWithState
    val sessions = eventStream
      .groupByKey(_.userId)
      .flatMapGroupsWithState(
        outputMode = OutputMode.Append(),
        timeoutConf = GroupStateTimeout.EventTimeTimeout()
      )(updateUserState)
    
    val query = sessions.writeStream
      .format("console")
      .outputMode("append")
      .option("truncate", "false")
      .start()
    
    query.awaitTermination(120000)
    spark.stop()
  
  // mapGroupsWithState - emit output ทุก batch
  def mapGroupsStateDemo(spark: SparkSession): Unit =
    import spark.implicits.*
    
    case class RunningAverage(
      sum: Double,
      count: Long,
      average: Double
    )
    
    val numberStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "10")
      .load()
      .select(
        (col("value") % 5).cast("string").as("groupId"),
        (rand() * 100).as("value")
      )
    
    val runningAvg = numberStream
      .groupByKey(row => row.getAs[String]("groupId"))
      .mapGroupsWithState(GroupStateTimeout.NoTimeout()) {
        case (groupId: String, 
              values: Iterator[org.apache.spark.sql.Row], 
              state: GroupState[RunningAverage]) =>
          
          val currentState = if state.exists then state.get
                             else RunningAverage(0.0, 0L, 0.0)
          
          val newValues = values.map(_.getAs[Double]("value")).toList
          val newSum = currentState.sum + newValues.sum
          val newCount = currentState.count + newValues.size
          val newAvg = if newCount > 0 then newSum / newCount else 0.0
          
          val updatedState = RunningAverage(newSum, newCount, newAvg)
          state.update(updatedState)
          
          (groupId, newAvg, newCount)
      }
    
    runningAvg.writeStream
      .format("console")
      .outputMode("update")
      .start()
      .awaitTermination(60000)
```

---

## 6. Output Modes: Append, Complete, Update {#output-modes}

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.streaming.{StreamingQuery, Trigger}

object OutputModesDemo:
  
  def demonstrateOutputModes(spark: SparkSession): Unit =
    import spark.implicits.*
    
    val baseStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "10")
      .load()
      .select(
        (col("value") % 5).cast("string").as("category"),
        col("timestamp"),
        (rand() * 100).as("amount")
      )
      .withWatermark("timestamp", "1 minute")
    
    // === 1. Append Mode ===
    // - เหมาะสำหรับ stateless operations หรือ windowed ops ที่มี watermark
    // - แต่ละ row ถูก emit เพียงครั้งเดียว
    // - ไม่รองรับ aggregations ที่ไม่มี watermark
    val appendQuery = baseStream
      .filter(col("amount") > 50)  // stateless
      .writeStream
      .format("console")
      .outputMode("append")
      .queryName("append-query")
      .start()
    
    // === 2. Complete Mode ===
    // - เขียน result table ทั้งหมดทุก trigger
    // - เหมาะสำหรับ aggregations ที่ต้องการ full result set
    val completeQuery = baseStream
      .groupBy("category")
      .agg(
        count("*").as("count"),
        sum("amount").as("total")
      )
      .writeStream
      .format("console")
      .outputMode("complete")
      .queryName("complete-query")
      .start()
    
    // === 3. Update Mode ===
    // - เขียนเฉพาะ rows ที่เปลี่ยนแปลงจาก last trigger
    // - ประหยัด I/O กว่า Complete mode
    // - รองรับ aggregations
    val updateQuery = baseStream
      .groupBy(
        window(col("timestamp"), "1 minute"),
        col("category")
      )
      .agg(
        count("*").as("count"),
        avg("amount").as("avg_amount")
      )
      .writeStream
      .format("console")
      .outputMode("update")
      .option("truncate", "false")
      .queryName("update-query")
      .start()
    
    // รอทุก queries
    Thread.sleep(60000)
    appendQuery.stop()
    completeQuery.stop()
    updateQuery.stop()
  
  def writeToMultipleSinks(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Foreachbatch - เขียนไปหลาย sinks
    val stream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "100")
      .load()
      .select(
        (col("value") % 10).cast("string").as("category"),
        (rand() * 1000).as("amount"),
        col("timestamp")
      )
    
    val aggregatedStream = stream
      .groupBy(
        window(col("timestamp"), "30 seconds"),
        col("category")
      )
      .agg(
        count("*").as("count"),
        sum("amount").as("total_amount"),
        avg("amount").as("avg_amount")
      )
    
    // foreachBatch: custom output logic
    val query = aggregatedStream.writeStream
      .foreachBatch { (batchDF: org.apache.spark.sql.DataFrame, batchId: Long) =>
        println(s"\n=== Batch $batchId ===")
        batchDF.show(truncate = false)
        
        // เขียนไป multiple destinations
        // 1. บันทึกลง Parquet
        batchDF.write
          .mode("append")
          .parquet(s"/tmp/streaming-output/batch=$batchId")
        
        // 2. เขียนลง Delta Lake (ถ้าใช้ Delta)
        // batchDF.write
        //   .format("delta")
        //   .mode("append")
        //   .save("/tmp/delta-table")
        
        // 3. เขียนลง JDBC database
        // batchDF.write
        //   .mode("append")
        //   .jdbc("jdbc:postgresql://host:5432/db", "table", props)
        
        println(s"Batch $batchId: ${batchDF.count()} rows processed")
      }
      .outputMode("update")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .option("checkpointLocation", "/tmp/foreach-checkpoint")
      .start()
    
    query.awaitTermination(120000)
```

---

## 7. Checkpointing และ Fault Tolerance {#checkpointing}

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.streaming.{StreamingQuery, Trigger}

object CheckpointingDemo:
  
  def productionStreamingSetup(spark: SparkSession): StreamingQuery =
    import spark.implicits.*
    
    val CHECKPOINT_PATH = "hdfs://namenode:9000/checkpoints/streaming-app"
    // หรือใช้ S3: "s3a://bucket/checkpoints/streaming-app"
    // หรือ local: "/tmp/checkpoints/streaming-app"
    
    val stream = spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "kafka:9092")
      .option("subscribe", "transactions")
      .option("startingOffsets", "latest")
      .load()
    
    val processedStream = stream
      .select(
        col("key").cast("string"),
        col("value").cast("string").as("json"),
        col("timestamp")
      )
      .withWatermark("timestamp", "10 minutes")
      .groupBy(
        window(col("timestamp"), "5 minutes"),
        col("key")
      )
      .count()
    
    processedStream.writeStream
      .format("delta")
      .outputMode("append")
      .option("checkpointLocation", CHECKPOINT_PATH)
      // Exactly-once semantics ต้องการ idempotent sink
      .option("path", "hdfs://namenode:9000/output/transactions")
      .trigger(Trigger.ProcessingTime("1 minute"))
      .start()
  
  def localCheckpointDemo(spark: SparkSession): Unit =
    import spark.implicits.*
    
    val checkpointDir = "/tmp/streaming-checkpoint-demo"
    
    val stream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "50")
      .load()
      .select(
        (col("value") % 5).cast("string").as("category"),
        col("timestamp"),
        (rand() * 100).as("amount")
      )
      .withWatermark("timestamp", "2 minutes")
    
    val result = stream
      .groupBy(
        window(col("timestamp"), "1 minute"),
        col("category")
      )
      .agg(
        count("*").as("count"),
        sum("amount").as("total")
      )
    
    println("=== Starting streaming query with checkpoint ===")
    println(s"Checkpoint location: $checkpointDir")
    
    val query = result.writeStream
      .format("console")
      .outputMode("update")
      .option("checkpointLocation", checkpointDir)
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("15 seconds"))
      .start()
    
    // Simulate query recovery
    Thread.sleep(45000)
    
    println("\n=== Query Status ===")
    println(s"Active: ${query.isActive}")
    println(s"ID: ${query.id}")
    println(s"Run ID: ${query.runId}")
    
    // ดู recent progress
    query.recentProgress.foreach { progress =>
      println(s"Batch: ${progress.batchId}, " +
              s"Input rows: ${progress.numInputRows}, " +
              s"Processing rate: ${progress.processedRowsPerSecond}")
    }
    
    query.stop()
    
    println("\n=== Restarting from checkpoint ===")
    // เมื่อ restart จะ resume จาก checkpoint อัตโนมัติ
    val resumedQuery = result.writeStream
      .format("console")
      .outputMode("update")
      .option("checkpointLocation", checkpointDir)  // ใช้ checkpoint เดิม
      .option("truncate", "false")
      .trigger(Trigger.ProcessingTime("15 seconds"))
      .start()
    
    resumedQuery.awaitTermination(60000)
  
  def exactlyOnceProcessing(spark: SparkSession): Unit =
    import spark.implicits.*
    
    // Exactly-once semantics ต้องการ:
    // 1. Idempotent sources (Kafka ด้วย offsets)
    // 2. Idempotent sinks (Delta Lake, idempotent writes)
    // 3. Checkpoint สำหรับ state recovery
    
    spark.readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", "localhost:9092")
      .option("subscribe", "orders")
      .option("startingOffsets", """{"orders":{"0":100,"1":200}}""")
      .load()
      .select(
        col("value").cast("string").as("order_json"),
        col("partition"),
        col("offset"),
        col("timestamp")
      )
      .writeStream
      .foreachBatch { (df: org.apache.spark.sql.DataFrame, batchId: Long) =>
        // Idempotent write: ใช้ batchId เป็น transaction identifier
        df.write
          .mode("append")
          // Delta Lake รองรับ exactly-once ด้วย ACID transactions
          .format("delta")
          .option("txnVersion", batchId.toString)
          .option("txnAppId", "my-streaming-app")
          .save("/tmp/delta-orders")
      }
      .option("checkpointLocation", "/tmp/kafka-checkpoint")
      .start()
```

---

## 8. Streaming Analytics Pipeline สมบูรณ์ {#streaming-pipeline}

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.sql.streaming.{GroupState, GroupStateTimeout, OutputMode, Trigger}
import org.apache.spark.sql.types.*
import java.sql.Timestamp

object CompleteStreamingPipeline:
  
  // Schemas
  val transactionSchema: StructType = StructType(Seq(
    StructField("transactionId", StringType, nullable = false),
    StructField("userId", StringType, nullable = false),
    StructField("merchantId", StringType, nullable = false),
    StructField("amount", DoubleType, nullable = false),
    StructField("currency", StringType, nullable = false),
    StructField("category", StringType, nullable = true),
    StructField("timestamp", LongType, nullable = false),
    StructField("location", StructType(Seq(
      StructField("country", StringType),
      StructField("city", StringType),
      StructField("lat", DoubleType),
      StructField("lon", DoubleType)
    )), nullable = true)
  ))
  
  // State สำหรับ fraud detection
  case class UserBehaviorState(
    userId: String,
    txCount: Int,
    totalAmount: Double,
    distinctMerchants: Set[String],
    distinctCountries: Set[String],
    lastTxTime: Long,
    velocityScore: Double
  )
  
  case class FraudAlert(
    transactionId: String,
    userId: String,
    amount: Double,
    alertType: String,
    score: Double,
    timestamp: Timestamp
  )
  
  // Fraud detection logic
  def detectFraud(
    userId: String,
    transactions: Iterator[org.apache.spark.sql.Row],
    state: GroupState[UserBehaviorState]
  ): Iterator[FraudAlert] =
    
    if state.hasTimedOut then
      state.remove()
      return Iterator.empty
    
    val txList = transactions.toList
    
    val currentState = state.getOption.getOrElse(
      UserBehaviorState(
        userId, 0, 0.0, Set.empty, Set.empty, 0L, 0.0
      )
    )
    
    val alerts = txList.flatMap { tx =>
      val txId = tx.getAs[String]("transactionId")
      val amount = tx.getAs[Double]("amount")
      val merchantId = tx.getAs[String]("merchantId")
      val txTime = tx.getAs[Long]("timestamp")
      val country = Option(tx.getAs[org.apache.spark.sql.Row]("location"))
        .map(_.getAs[String]("country")).getOrElse("Unknown")
      
      var txAlerts = List.empty[FraudAlert]
      
      // Rule 1: High velocity - มากกว่า 10 transactions ใน 5 นาที
      val recentTxRate = (currentState.txCount + 1).toDouble / 
        Math.max(1.0, (txTime - currentState.lastTxTime) / 60000.0)
      if recentTxRate > 10 then
        txAlerts = FraudAlert(txId, userId, amount, "HIGH_VELOCITY",
          Math.min(recentTxRate / 10, 1.0), new Timestamp(txTime)) :: txAlerts
      
      // Rule 2: Large amount anomaly - เกิน 10x ค่าเฉลี่ย
      val avgAmount = if currentState.txCount > 0 then 
                        currentState.totalAmount / currentState.txCount 
                      else amount
      if amount > avgAmount * 10 && currentState.txCount > 5 then
        txAlerts = FraudAlert(txId, userId, amount, "LARGE_AMOUNT",
          Math.min(amount / (avgAmount * 10), 1.0), new Timestamp(txTime)) :: txAlerts
      
      // Rule 3: Geographic anomaly - ใช้ใน 3+ ประเทศต่างกันใน 1 ชั่วโมง
      val newCountries = currentState.distinctCountries + country
      if newCountries.size >= 3 then
        txAlerts = FraudAlert(txId, userId, amount, "GEO_ANOMALY",
          Math.min(newCountries.size.toDouble / 3, 1.0), new Timestamp(txTime)) :: txAlerts
      
      txAlerts
    }
    
    // อัพเดท state
    if txList.nonEmpty then
      val latestTx = txList.maxBy(_.getAs[Long]("timestamp"))
      val updatedState = currentState.copy(
        txCount = currentState.txCount + txList.size,
        totalAmount = currentState.totalAmount + txList.map(_.getAs[Double]("amount")).sum,
        distinctMerchants = currentState.distinctMerchants ++ txList.map(_.getAs[String]("merchantId")),
        distinctCountries = currentState.distinctCountries ++ txList.flatMap { tx =>
          Option(tx.getAs[org.apache.spark.sql.Row]("location"))
            .map(_.getAs[String]("country"))
        },
        lastTxTime = latestTx.getAs[Long]("timestamp")
      )
      state.update(updatedState)
      state.setEventTimeTimeout(new Timestamp(updatedState.lastTxTime + 3600000L))
    
    alerts.iterator
  
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("StreamingAnalyticsPipeline")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "8")
      .config("spark.sql.streaming.stateStore.providerClass",
        "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider")
      .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits.*
    
    println("=== Starting Streaming Analytics Pipeline ===")
    
    // Mock transaction stream
    val transactionStream = spark.readStream
      .format("rate")
      .option("rowsPerSecond", "200")
      .load()
      .select(
        concat(lit("TX"), col("value").cast("string")).as("transactionId"),
        concat(lit("U"), (col("value") % 1000).cast("string")).as("userId"),
        concat(lit("M"), (col("value") % 100).cast("string")).as("merchantId"),
        (rand() * 500 + 10).as("amount"),
        lit("THB").as("currency"),
        array(lit("food"), lit("shopping"), lit("travel"), lit("entertainment"))
          .getItem((col("value") % 4).cast("int")).as("category"),
        (unix_timestamp() * 1000).as("timestamp"),
        struct(
          array(lit("TH"), lit("SG"), lit("JP"), lit("US"))
            .getItem((rand() * 4).cast("int")).as("country"),
          lit("Bangkok").as("city"),
          lit(13.75).as("lat"),
          lit(100.52).as("lon")
        ).as("location")
      )
      .withColumn("event_time",
        (col("timestamp") / 1000).cast("timestamp"))
      .withWatermark("event_time", "10 minutes")
    
    // === Query 1: Real-time Statistics ===
    val realtimeStats = transactionStream
      .groupBy(
        window(col("event_time"), "1 minute"),
        col("category")
      )
      .agg(
        count("*").as("tx_count"),
        sum("amount").as("total_amount"),
        avg("amount").as("avg_amount"),
        percentile_approx(col("amount"), lit(0.99)).as("p99_amount"),
        approx_count_distinct("userId").as("unique_users")
      )
    
    val statsQuery = realtimeStats.writeStream
      .format("console")
      .outputMode("update")
      .option("truncate", "false")
      .queryName("realtime-stats")
      .trigger(Trigger.ProcessingTime("30 seconds"))
      .start()
    
    // === Query 2: Fraud Detection ===
    val fraudAlerts = transactionStream
      .groupByKey(row => row.getAs[String]("userId"))
      .flatMapGroupsWithState(
        outputMode = OutputMode.Append(),
        timeoutConf = GroupStateTimeout.EventTimeTimeout()
      )(detectFraud)
    
    val fraudQuery = fraudAlerts.writeStream
      .format("console")
      .outputMode("append")
      .option("truncate", "false")
      .queryName("fraud-detection")
      .trigger(Trigger.ProcessingTime("15 seconds"))
      .start()
    
    // === Query 3: Merchant Analytics ===
    val merchantAnalytics = transactionStream
      .groupBy(
        window(col("event_time"), "5 minutes", "1 minute"),
        col("merchantId")
      )
      .agg(
        count("*").as("tx_count"),
        sum("amount").as("revenue"),
        approx_count_distinct("userId").as("customers")
      )
      .filter(col("tx_count") >= 5)
    
    val merchantQuery = merchantAnalytics.writeStream
      .format("console")
      .outputMode("update")
      .option("numRows", "10")
      .queryName("merchant-analytics")
      .trigger(Trigger.ProcessingTime("60 seconds"))
      .start()
    
    println(s"Active queries: ${spark.streams.active.map(_.name).mkString(", ")}")
    
    // Monitor all queries
    Thread.sleep(150000)
    
    spark.streams.active.foreach(_.stop())
    spark.stop()
```

---

## 9. สรุป {#summary}

### หัวข้อที่ครอบคลุมในบทนี้

1. **Micro-batch vs Continuous**: เข้าใจ trade-offs ระหว่างความหน่วงและ throughput
2. **Kafka Integration**: อ่านและเขียน Kafka ด้วย Structured Streaming
3. **Watermarks**: จัดการ late-arriving data อย่างถูกต้อง
4. **Window Operations**: ใช้ Tumbling, Sliding, Session windows
5. **Stateful Processing**: สร้าง complex state machines ด้วย `flatMapGroupsWithState`
6. **Output Modes**: เลือก Append, Complete, Update ตาม use case
7. **Fault Tolerance**: ใช้ checkpointing เพื่อ exactly-once semantics

### Production Checklist

```scala
// 1. กำหนด checkpoint location เสมอ
// 2. ตั้ง watermark สำหรับ stateful operations
// 3. จำกัด state size เพื่อหลีกเลี่ยง OOM
// 4. ใช้ RocksDB state store สำหรับ large state
// 5. Monitor query metrics อย่างสม่ำเสมอ
// 6. ทดสอบ recovery จาก checkpoint
// 7. ตั้ง alert สำหรับ query failure

val spark = SparkSession.builder()
  .config("spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider")
  .config("spark.sql.streaming.statefulOperator.checkCorrectness.enabled", "false")
  .getOrCreate()
```

---

*[← Part 70: Apache Spark Advanced](part-70-spark-advanced.md) | [Part 72: Functional Programming Patterns →](part-72-fp-patterns.md)*
