# ส่วนที่ 83: Machine Learning with Scala

## สารบัญ

1. [Breeze สำหรับ Numerical Computing](#breeze-สำหรับ-numerical-computing)
2. [Matrix Operations และ Linear Algebra](#matrix-operations-และ-linear-algebra)
3. [Apache Spark MLlib](#apache-spark-mllib)
4. [Feature Engineering](#feature-engineering)
5. [Model Evaluation](#model-evaluation)
6. [Serving ML Models ผ่าน HTTP](#serving-ml-models-ผ่าน-http)
7. [Complete ML Pipeline](#complete-ml-pipeline)
8. [สรุป](#สรุป)

---

## Breeze สำหรับ Numerical Computing

Breeze คือ numerical processing library สำหรับ Scala ที่คล้ายกับ NumPy ใน Python

### Setup Dependencies

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.scalanlp" %% "breeze" % "2.1.0",
  "org.scalanlp" %% "breeze-viz" % "2.1.0",  // optional: visualization
  "org.scalanlp" %% "breeze-natives" % "2.1.0" // BLAS/LAPACK optimization
)
```

### Vectors และ Basic Operations

```scala
import breeze.linalg.*
import breeze.numerics.*
import breeze.stats.*

// Create vectors
val v1 = DenseVector(1.0, 2.0, 3.0, 4.0, 5.0)
val v2 = DenseVector(2.0, 3.0, 4.0, 5.0, 6.0)

// Basic arithmetic
val sum = v1 + v2       // DenseVector(3.0, 5.0, 7.0, 9.0, 11.0)
val diff = v2 - v1      // DenseVector(1.0, 1.0, 1.0, 1.0, 1.0)
val scaled = v1 * 2.0   // DenseVector(2.0, 4.0, 6.0, 8.0, 10.0)

// Dot product
val dot = v1 dot v2   // 70.0

// Norm (magnitude)
val norm2 = norm(v1)    // L2 norm ≈ 7.416
val norm1 = norm(v1, 1) // L1 norm = 15.0

// Elementwise operations
val sqrtV = sqrt(v1)    // elementwise square root
val expV = exp(v1)      // elementwise exponential
val absV = abs(v1 - 3.0) // absolute value

// Slicing
val slice = v1(1 to 3)  // DenseVector(2.0, 3.0, 4.0)
val every2 = v1(0 to 4 by 2) // DenseVector(1.0, 3.0, 5.0)

// Statistics
val meanV = mean(v1)   // 3.0
val stdV = stddev(v1)  // ≈ 1.581
val maxV = max(v1)     // 5.0
val minV = min(v1)     // 1.0

println(s"v1: $v1")
println(s"dot product: $dot")
println(s"norm: $norm2")
println(s"mean: $meanV, std: $stdV")
```

### Sparse Vectors

```scala
import breeze.linalg.*

// Sparse vector สำหรับ high-dimensional data (เช่น text features)
val sparseV = SparseVector(1000)(
  0 -> 1.0,
  42 -> 2.5,
  99 -> 0.7,
  500 -> 3.2
)

println(s"Dimension: ${sparseV.length}")   // 1000
println(s"Active elements: ${sparseV.activeSize}") // 4
println(s"Value at 42: ${sparseV(42)}")    // 2.5

// Convert between dense and sparse
val dense = sparseV.toDenseVector
val sparse2 = dense.toSparseVector

// Dot product ระหว่าง dense และ sparse
val denseSmall = DenseVector.ones[Double](1000)
val dotResult = denseSmall dot sparseV // efficient computation
```

---

## Matrix Operations และ Linear Algebra

### Matrix Creation และ Operations

```scala
import breeze.linalg.*
import breeze.numerics.*

// Create matrices
val m1 = DenseMatrix(
  (1.0, 2.0, 3.0),
  (4.0, 5.0, 6.0),
  (7.0, 8.0, 9.0)
)

val m2 = DenseMatrix.eye[Double](3) // identity matrix

// Matrix arithmetic
val mSum = m1 + m1        // element-wise addition
val mScale = m1 * 2.0     // scalar multiplication
val mProd = m1 * m2       // matrix multiplication

// Transpose
val mT = m1.t  // or m1.transpose

// Element access
val elem = m1(1, 2)       // row 1, col 2 = 6.0
val row = m1(1, ::)       // second row
val col = m1(::, 2)       // third column

// Slicing
val submatrix = m1(0 to 1, 0 to 1)  // 2x2 top-left submatrix

println(s"Matrix:\n$m1")
println(s"Transpose:\n$mT")
println(s"Row 1: $row")
println(s"Col 2: $col")
```

### Linear Algebra Operations

```scala
import breeze.linalg.*
import breeze.linalg.svd.SVD

// Determinant
val det = det(m1)   // ≈ 0 (singular matrix)

// Inverse
val m3 = DenseMatrix(
  (2.0, 1.0, 0.0),
  (1.0, 3.0, 1.0),
  (0.0, 1.0, 2.0)
)
val inv = inv(m3)

// Verify: m3 * inv(m3) ≈ identity
println(s"m3 * inv(m3) ≈\n${m3 * inv}")

// Solve linear system: Ax = b
val A = DenseMatrix(
  (2.0, 1.0),
  (1.0, 3.0)
)
val b = DenseVector(5.0, 10.0)
val x = A \ b   // solve for x
println(s"Solution x: $x")
println(s"Verify Ax = b: ${A * x}")

// SVD (Singular Value Decomposition)
val SVD(u, s, vt) = svd(m3)
println(s"U:\n$u")
println(s"Singular values: $s")
println(s"V^T:\n$vt")

// PCA ด้วย SVD
def pca(data: DenseMatrix[Double], nComponents: Int): DenseMatrix[Double] =
  // Center data
  val means = mean(data(::, *))
  val centered = data(*, ::) - means.t
  
  // Compute covariance matrix
  val n = data.rows.toDouble
  val cov = (centered.t * centered) / (n - 1)
  
  // SVD
  val SVD(_, _, vt) = svd(cov)
  
  // Take top components
  val components = vt(0 until nComponents, ::).t
  
  // Project data
  centered * components
```

### Gradient Descent Implementation

```scala
import breeze.linalg.*
import breeze.numerics.*

// Linear Regression ด้วย Gradient Descent
class LinearRegression(learningRate: Double = 0.01, maxIter: Int = 1000, tol: Double = 1e-6):
  private var weights: DenseVector[Double] = _
  private var bias: Double = 0.0
  private val lossHistory = scala.collection.mutable.ArrayBuffer[Double]()
  
  def fit(X: DenseMatrix[Double], y: DenseVector[Double]): Unit =
    val nSamples = X.rows
    val nFeatures = X.cols
    weights = DenseVector.zeros[Double](nFeatures)
    
    for iter <- 0 until maxIter do
      // Forward pass
      val predictions = X * weights + bias
      val errors = predictions - y
      
      // Compute loss (MSE)
      val loss = sum(errors *:* errors) / (2.0 * nSamples)
      lossHistory += loss
      
      // Backward pass (gradients)
      val dWeights = (X.t * errors) / nSamples.toDouble
      val dBias = sum(errors) / nSamples.toDouble
      
      // Update parameters
      val prevWeights = weights.copy
      weights -= dWeights * learningRate
      bias -= dBias * learningRate
      
      // Check convergence
      val weightChange = norm(weights - prevWeights)
      if weightChange < tol then
        println(s"Converged at iteration $iter")
        return
  
  def predict(X: DenseMatrix[Double]): DenseVector[Double] =
    X * weights + bias
  
  def getLossHistory: Seq[Double] = lossHistory.toSeq
  
  def getWeights: DenseVector[Double] = weights
  def getBias: Double = bias

// ทดสอบ Linear Regression
@main def testLinearRegression(): Unit =
  import breeze.stats.distributions.*
  
  val rand = new scala.util.Random(42)
  val n = 1000
  
  // Generate synthetic data: y = 2x1 + 3x2 + noise
  val X = DenseMatrix.tabulate(n, 2) { (i, j) =>
    if j == 0 then rand.nextGaussian() else rand.nextGaussian() * 2
  }
  
  val trueWeights = DenseVector(2.0, 3.0)
  val noise = DenseVector.tabulate(n)(i => rand.nextGaussian() * 0.1)
  val y = X * trueWeights + noise
  
  val model = new LinearRegression(learningRate = 0.01, maxIter = 2000)
  model.fit(X, y)
  
  println(s"True weights: $trueWeights")
  println(s"Learned weights: ${model.getWeights}")
  println(s"Bias: ${model.getBias}")
  println(s"Final loss: ${model.getLossHistory.last}")
```

---

## Apache Spark MLlib

### Setup

```scala
// build.sbt
libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-mllib" % "3.5.0",
  "org.apache.spark" %% "spark-sql" % "3.5.0"
)
```

### Basic Classification Pipeline

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.ml.*
import org.apache.spark.ml.classification.*
import org.apache.spark.ml.evaluation.*
import org.apache.spark.ml.feature.*
import org.apache.spark.ml.tuning.*
import org.apache.spark.sql.functions.*

val spark = SparkSession.builder()
  .appName("ML Pipeline")
  .master("local[*]")
  .config("spark.driver.memory", "4g")
  .getOrCreate()

import spark.implicits.*

// Load data
val rawData = spark.read
  .option("header", "true")
  .option("inferSchema", "true")
  .csv("data/customer_churn.csv")

rawData.printSchema()
rawData.show(5)

// Feature engineering
val categoricalCols = Array("gender", "contract", "paymentMethod")
val numericCols = Array("tenure", "monthlyCharges", "totalCharges")
val labelCol = "churn"

// Preprocessing pipeline
val stages = scala.collection.mutable.ArrayBuffer[PipelineStage]()

// String indexing for categorical features
val stringIndexers = categoricalCols.map { col =>
  new StringIndexer()
    .setInputCol(col)
    .setOutputCol(s"${col}Index")
    .setHandleInvalid("keep")
}
stages ++= stringIndexers

// One-hot encoding
val oneHotEncoders = categoricalCols.map { col =>
  new OneHotEncoder()
    .setInputCol(s"${col}Index")
    .setOutputCol(s"${col}Vec")
}
stages ++= oneHotEncoders

// Handle missing values
val imputer = new Imputer()
  .setInputCols(numericCols)
  .setOutputCols(numericCols.map(c => s"${c}Imputed"))
  .setStrategy("mean")
stages += imputer

// Scale numeric features
val scaler = new StandardScaler()
  .setInputCol("numericFeatures")
  .setOutputCol("scaledFeatures")
  .setWithMean(true)
  .setWithStd(true)

// Assemble numeric features
val numericAssembler = new VectorAssembler()
  .setInputCols(numericCols.map(c => s"${c}Imputed"))
  .setOutputCol("numericFeatures")
stages += numericAssembler
stages += scaler

// Assemble all features
val allFeatureCols = categoricalCols.map(c => s"${c}Vec") :+ "scaledFeatures"
val assembler = new VectorAssembler()
  .setInputCols(allFeatureCols)
  .setOutputCol("features")
  .setHandleInvalid("skip")
stages += assembler

// Label indexer
val labelIndexer = new StringIndexer()
  .setInputCol(labelCol)
  .setOutputCol("label")
stages += labelIndexer

// Classifier: Random Forest
val classifier = new RandomForestClassifier()
  .setFeaturesCol("features")
  .setLabelCol("label")
  .setNumTrees(100)
  .setMaxDepth(10)
  .setSeed(42)
stages += classifier

// Convert prediction back to label
val labelConverter = new IndexToString()
  .setInputCol("prediction")
  .setOutputCol("predictedLabel")
  .setLabels(new StringIndexer()
    .setInputCol(labelCol)
    .setOutputCol("label")
    .fit(rawData)
    .labels)
stages += labelConverter

// Build pipeline
val pipeline = new Pipeline().setStages(stages.toArray)

// Split data
val Array(trainData, testData) = rawData.randomSplit(Array(0.8, 0.2), seed = 42)

// Train
println("Training pipeline...")
val pipelineModel = pipeline.fit(trainData)

// Evaluate
val predictions = pipelineModel.transform(testData)

val evaluator = new BinaryClassificationEvaluator()
  .setLabelCol("label")
  .setRawPredictionCol("rawPrediction")
  .setMetricName("areaUnderROC")

val auc = evaluator.evaluate(predictions)
println(s"AUC: $auc")

// Multi-metric evaluation
val multiEval = new MulticlassClassificationEvaluator()
  .setLabelCol("label")
  .setPredictionCol("prediction")

val accuracy = multiEval.setMetricName("accuracy").evaluate(predictions)
val f1 = multiEval.setMetricName("f1").evaluate(predictions)
val precision = multiEval.setMetricName("weightedPrecision").evaluate(predictions)
val recall = multiEval.setMetricName("weightedRecall").evaluate(predictions)

println(s"Accuracy: $accuracy")
println(s"F1 Score: $f1")
println(s"Precision: $precision")
println(s"Recall: $recall")
```

---

## Feature Engineering

### Advanced Feature Engineering

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.*
import org.apache.spark.ml.feature.*

// Date features
def addDateFeatures(df: org.apache.spark.sql.DataFrame, dateCol: String) =
  df
    .withColumn(s"${dateCol}_year", year(col(dateCol)))
    .withColumn(s"${dateCol}_month", month(col(dateCol)))
    .withColumn(s"${dateCol}_day", dayofmonth(col(dateCol)))
    .withColumn(s"${dateCol}_dayofweek", dayofweek(col(dateCol)))
    .withColumn(s"${dateCol}_quarter", quarter(col(dateCol)))
    .withColumn(s"${dateCol}_is_weekend", 
      (dayofweek(col(dateCol)) isin (1, 7)).cast("int"))

// Text features
def addTextFeatures(df: org.apache.spark.sql.DataFrame, textCol: String) =
  df
    .withColumn(s"${textCol}_length", length(col(textCol)))
    .withColumn(s"${textCol}_word_count", 
      size(split(col(textCol), "\\s+")))
    .withColumn(s"${textCol}_char_count", 
      length(regexp_replace(col(textCol), "\\s", "")))

// TF-IDF สำหรับ text
def createTfIdfFeatures(
  df: org.apache.spark.sql.DataFrame,
  textCol: String,
  vocabSize: Int = 10000
): (org.apache.spark.sql.DataFrame, IDF) =
  val tokenizer = new Tokenizer()
    .setInputCol(textCol)
    .setOutputCol(s"${textCol}_words")
  
  val remover = new StopWordsRemover()
    .setInputCol(s"${textCol}_words")
    .setOutputCol(s"${textCol}_filtered")
  
  val hashTF = new HashingTF()
    .setInputCol(s"${textCol}_filtered")
    .setOutputCol(s"${textCol}_tf")
    .setNumFeatures(vocabSize)
  
  val idf = new IDF()
    .setInputCol(s"${textCol}_tf")
    .setOutputCol(s"${textCol}_tfidf")
  
  val processed = tokenizer.transform(df)
  val filtered = remover.transform(processed)
  val withTF = hashTF.transform(filtered)
  val idfModel = idf.fit(withTF)
  val result = idfModel.transform(withTF)
  
  (result, idfModel)

// Polynomial features
def addPolynomialFeatures(
  df: org.apache.spark.sql.DataFrame, 
  numericCols: Array[String],
  degree: Int = 2
): org.apache.spark.sql.DataFrame =
  val assembler = new VectorAssembler()
    .setInputCols(numericCols)
    .setOutputCol("polyInput")
  
  val poly = new PolynomialExpansion()
    .setInputCol("polyInput")
    .setOutputCol("polyFeatures")
    .setDegree(degree)
  
  val assembled = assembler.transform(df)
  poly.transform(assembled)
    .drop("polyInput")
```

---

## Model Evaluation

### Comprehensive Model Evaluation

```scala
import org.apache.spark.ml.evaluation.*
import org.apache.spark.sql.{DataFrame, functions as F}

// Custom evaluator สำหรับ business metrics
object ModelEvaluator:
  // Classification metrics
  def evaluateClassification(predictions: DataFrame, labelCol: String = "label"): Map[String, Double] =
    val multiEval = new MulticlassClassificationEvaluator()
      .setLabelCol(labelCol)
      .setPredictionCol("prediction")
    
    val binaryEval = new BinaryClassificationEvaluator()
      .setLabelCol(labelCol)
      .setRawPredictionCol("rawPrediction")
    
    Map(
      "accuracy" -> multiEval.setMetricName("accuracy").evaluate(predictions),
      "f1" -> multiEval.setMetricName("f1").evaluate(predictions),
      "precision" -> multiEval.setMetricName("weightedPrecision").evaluate(predictions),
      "recall" -> multiEval.setMetricName("weightedRecall").evaluate(predictions),
      "auc_roc" -> binaryEval.setMetricName("areaUnderROC").evaluate(predictions),
      "auc_pr" -> binaryEval.setMetricName("areaUnderPR").evaluate(predictions)
    )
  
  // Confusion matrix
  def confusionMatrix(predictions: DataFrame): DataFrame =
    predictions
      .groupBy("label", "prediction")
      .count()
      .orderBy("label", "prediction")
  
  // Regression metrics
  def evaluateRegression(predictions: DataFrame, labelCol: String = "label"): Map[String, Double] =
    val regEval = new RegressionEvaluator()
      .setLabelCol(labelCol)
      .setPredictionCol("prediction")
    
    Map(
      "rmse" -> regEval.setMetricName("rmse").evaluate(predictions),
      "mse" -> regEval.setMetricName("mse").evaluate(predictions),
      "mae" -> regEval.setMetricName("mae").evaluate(predictions),
      "r2" -> regEval.setMetricName("r2").evaluate(predictions)
    )
  
  // Print metrics report
  def printReport(metrics: Map[String, Double], title: String = "Model Metrics"): Unit =
    println(s"\n=== $title ===")
    metrics.foreach { (name, value) =>
      println(f"$name%-20s: $value%.4f")
    }

// Cross-validation
def crossValidate(
  pipeline: Pipeline,
  trainData: DataFrame,
  paramGrid: Array[org.apache.spark.ml.param.ParamMap],
  numFolds: Int = 5,
  metricName: String = "areaUnderROC"
): CrossValidatorModel =
  val evaluator = new BinaryClassificationEvaluator()
    .setLabelCol("label")
    .setMetricName(metricName)
  
  val cv = new CrossValidator()
    .setEstimator(pipeline)
    .setEvaluator(evaluator)
    .setEstimatorParamMaps(paramGrid)
    .setNumFolds(numFolds)
    .setParallelism(4) // parallel fold computation
  
  println(s"Running ${numFolds}-fold cross-validation with ${paramGrid.length} parameter combinations...")
  cv.fit(trainData)
```

---

## Serving ML Models ผ่าน HTTP

### Model Serving ด้วย http4s

```scala
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import io.circe.*
import io.circe.generic.auto.*
import cats.effect.*
import org.apache.spark.ml.PipelineModel
import org.apache.spark.sql.SparkSession

// Request/Response types
case class PredictionRequest(
  features: Map[String, String],
  returnProbability: Boolean = false
)

case class PredictionResponse(
  prediction: String,
  probability: Option[Double] = None,
  modelVersion: String,
  latencyMs: Long
)

case class BatchPredictionRequest(
  instances: List[Map[String, String]]
)

case class BatchPredictionResponse(
  predictions: List[PredictionResponse],
  totalLatencyMs: Long
)

// Model serving service
class ModelServingService(
  model: PipelineModel,
  spark: SparkSession,
  modelVersion: String = "1.0.0"
):
  import spark.implicits.*
  
  def predict(request: PredictionRequest): IO[PredictionResponse] =
    IO {
      val start = System.currentTimeMillis()
      
      // Create DataFrame from request
      val featureList = request.features.toList
      val df = spark.createDataFrame(
        List(request.features)
      ).toDF()
      
      // Run prediction
      val predictions = model.transform(df)
      val row = predictions.first()
      
      val prediction = row.getAs[String]("predictedLabel")
      val probability = if request.returnProbability then
        val probVec = row.getAs[org.apache.spark.ml.linalg.Vector]("probability")
        Some(probVec.toArray.max)
      else None
      
      val latency = System.currentTimeMillis() - start
      
      PredictionResponse(prediction, probability, modelVersion, latency)
    }
  
  def predictBatch(request: BatchPredictionRequest): IO[BatchPredictionResponse] =
    IO {
      val start = System.currentTimeMillis()
      
      // Create DataFrame from batch
      val df = spark.createDataFrame(request.instances)
      val predictions = model.transform(df)
      
      val results = predictions.collect().map { row =>
        val prediction = row.getAs[String]("predictedLabel")
        PredictionResponse(prediction, None, modelVersion, 0)
      }.toList
      
      BatchPredictionResponse(results, System.currentTimeMillis() - start)
    }

// HTTP Routes
class ModelRoutes(service: ModelServingService):
  implicit val predReqDecoder: EntityDecoder[IO, PredictionRequest] = jsonOf
  implicit val batchReqDecoder: EntityDecoder[IO, BatchPredictionRequest] = jsonOf
  
  val routes: HttpRoutes[IO] = HttpRoutes.of[IO] {
    case req @ POST -> Root / "predict" =>
      req.as[PredictionRequest].flatMap { request =>
        service.predict(request).flatMap(Ok(_))
      }
    
    case req @ POST -> Root / "predict" / "batch" =>
      req.as[BatchPredictionRequest].flatMap { request =>
        service.predictBatch(request).flatMap(Ok(_))
      }
    
    case GET -> Root / "health" =>
      Ok(Map("status" -> "healthy", "model_version" -> "1.0.0").asJson)
    
    case GET -> Root / "model" / "info" =>
      Ok(Map(
        "version" -> "1.0.0",
        "algorithm" -> "RandomForest",
        "features" -> 20,
        "classes" -> 2
      ).asJson)
  }

// Main server
object ModelServer extends IOApp:
  def run(args: List[String]): IO[ExitCode] =
    val spark = SparkSession.builder()
      .appName("ModelServer")
      .master("local[4]")
      .getOrCreate()
    spark.sparkContext.setLogLevel("ERROR")
    
    val model = PipelineModel.load("models/churn-model-v1")
    val service = new ModelServingService(model, spark)
    val routes = new ModelRoutes(service)
    
    import org.http4s.ember.server.*
    import com.comcast.ip4s.*
    
    EmberServerBuilder
      .default[IO]
      .withHost(host"0.0.0.0")
      .withPort(port"8080")
      .withHttpApp(routes.routes.orNotFound)
      .build
      .useForever
      .as(ExitCode.Success)
```

---

## Complete ML Pipeline

### End-to-End Churn Prediction Pipeline

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.ml.*
import org.apache.spark.ml.classification.*
import org.apache.spark.ml.evaluation.*
import org.apache.spark.ml.feature.*
import org.apache.spark.ml.tuning.*
import org.apache.spark.sql.functions.*
import org.apache.spark.ml.param.ParamMap

object ChurnPredictionPipeline:
  def main(args: Array[String]): Unit =
    val spark = SparkSession.builder()
      .appName("Churn Prediction")
      .master("local[*]")
      .config("spark.sql.shuffle.partitions", "8")
      .getOrCreate()
    
    import spark.implicits.*
    spark.sparkContext.setLogLevel("WARN")
    
    // === 1. Load and Explore Data ===
    println("=== Step 1: Loading Data ===")
    val data = spark.read
      .option("header", "true")
      .option("inferSchema", "true")
      .csv("data/telco_churn.csv")
    
    println(s"Total records: ${data.count()}")
    println(s"Schema:")
    data.printSchema()
    
    // Check class balance
    println("\nChurn distribution:")
    data.groupBy("Churn").count().show()
    
    // === 2. Data Cleaning ===
    println("\n=== Step 2: Data Cleaning ===")
    val cleanData = data
      .withColumn("TotalCharges", 
        when(col("TotalCharges") === " ", lit("0"))
          .otherwise(col("TotalCharges"))
          .cast("double"))
      .withColumn("ChurnLabel",
        when(col("Churn") === "Yes", 1.0).otherwise(0.0))
      .na.fill(Map(
        "TotalCharges" -> 0.0,
        "tenure" -> 0,
        "MonthlyCharges" -> 0.0
      ))
    
    println(s"Records after cleaning: ${cleanData.count()}")
    
    // === 3. Feature Engineering ===
    println("\n=== Step 3: Feature Engineering ===")
    val engineeredData = cleanData
      .withColumn("avg_monthly_spend",
        when(col("tenure") > 0, col("TotalCharges") / col("tenure"))
          .otherwise(col("MonthlyCharges")))
      .withColumn("is_high_value",
        when(col("MonthlyCharges") > 70, 1.0).otherwise(0.0))
      .withColumn("is_long_tenure",
        when(col("tenure") > 24, 1.0).otherwise(0.0))
      .withColumn("contract_type_encoded",
        when(col("Contract") === "Month-to-month", 0)
          .when(col("Contract") === "One year", 1)
          .otherwise(2))
    
    // === 4. Build Pipeline ===
    println("\n=== Step 4: Building Pipeline ===")
    
    val categoricalCols = Array(
      "gender", "Partner", "Dependents", "PhoneService",
      "MultipleLines", "InternetService", "OnlineSecurity",
      "OnlineBackup", "DeviceProtection", "TechSupport",
      "StreamingTV", "StreamingMovies", "PaperlessBilling",
      "PaymentMethod"
    )
    
    val numericCols = Array(
      "tenure", "MonthlyCharges", "TotalCharges",
      "avg_monthly_spend", "is_high_value", "is_long_tenure",
      "contract_type_encoded"
    )
    
    val stages = scala.collection.mutable.ArrayBuffer[PipelineStage]()
    
    // String indexing
    val indexers = categoricalCols.map { col =>
      new StringIndexer()
        .setInputCol(col)
        .setOutputCol(s"${col}_idx")
        .setHandleInvalid("keep")
    }
    stages ++= indexers
    
    // One-hot encoding
    val encoders = categoricalCols.map { col =>
      new OneHotEncoder()
        .setInputCol(s"${col}_idx")
        .setOutputCol(s"${col}_vec")
    }
    stages ++= encoders
    
    // Feature assembly
    val allCols = categoricalCols.map(c => s"${c}_vec") ++ numericCols
    val assembler = new VectorAssembler()
      .setInputCols(allCols)
      .setOutputCol("rawFeatures")
      .setHandleInvalid("skip")
    stages += assembler
    
    // Normalization
    val scaler = new StandardScaler()
      .setInputCol("rawFeatures")
      .setOutputCol("features")
      .setWithMean(true)
      .setWithStd(true)
    stages += scaler
    
    // Classifier
    val classifier = new GBTClassifier()
      .setFeaturesCol("features")
      .setLabelCol("ChurnLabel")
      .setMaxIter(100)
      .setMaxDepth(5)
      .setStepSize(0.1)
      .setSeed(42)
    stages += classifier
    
    val pipeline = new Pipeline().setStages(stages.toArray)
    
    // === 5. Hyperparameter Tuning ===
    println("\n=== Step 5: Hyperparameter Tuning ===")
    val paramGrid = new ParamGridBuilder()
      .addGrid(classifier.maxDepth, Array(3, 5, 7))
      .addGrid(classifier.maxIter, Array(50, 100))
      .addGrid(classifier.stepSize, Array(0.05, 0.1))
      .build()
    
    val evaluator = new BinaryClassificationEvaluator()
      .setLabelCol("ChurnLabel")
      .setMetricName("areaUnderROC")
    
    // Split data
    val Array(trainData, testData) = engineeredData.randomSplit(Array(0.8, 0.2), seed = 42)
    println(s"Train size: ${trainData.count()}, Test size: ${testData.count()}")
    
    // Cross-validation
    val cv = new CrossValidator()
      .setEstimator(pipeline)
      .setEvaluator(evaluator)
      .setEstimatorParamMaps(paramGrid)
      .setNumFolds(3)
      .setParallelism(2)
    
    println("Running cross-validation (this may take a while)...")
    val cvModel = cv.fit(trainData)
    
    println(s"Best AUC (CV): ${cvModel.avgMetrics.max}")
    
    // === 6. Final Evaluation ===
    println("\n=== Step 6: Final Evaluation ===")
    val testPredictions = cvModel.bestModel.transform(testData)
    
    val testAuc = evaluator.evaluate(testPredictions)
    println(s"Test AUC: $testAuc")
    
    val multiEval = new MulticlassClassificationEvaluator()
      .setLabelCol("ChurnLabel")
      .setPredictionCol("prediction")
    
    println(s"Accuracy: ${multiEval.setMetricName("accuracy").evaluate(testPredictions)}")
    println(s"F1 Score: ${multiEval.setMetricName("f1").evaluate(testPredictions)}")
    
    // Confusion matrix
    println("\nConfusion Matrix:")
    testPredictions
      .groupBy("ChurnLabel", "prediction")
      .count()
      .orderBy("ChurnLabel", "prediction")
      .show()
    
    // Feature importance (GBT)
    val gbtModel = cvModel.bestModel
      .asInstanceOf[PipelineModel]
      .stages.last
      .asInstanceOf[GBTClassificationModel]
    
    val featureImportances = gbtModel.featureImportances
    val importanceMap = allCols.zip(featureImportances.toArray)
      .sortBy(-_._2)
      .take(10)
    
    println("\nTop 10 Important Features:")
    importanceMap.foreach { (feature, importance) =>
      println(f"  $feature%-35s: $importance%.4f")
    }
    
    // === 7. Save Model ===
    println("\n=== Step 7: Saving Model ===")
    cvModel.bestModel.save("models/churn-model-v1")
    println("Model saved to models/churn-model-v1")
    
    spark.stop()
```

---

## สรุป

Machine Learning ด้วย Scala มีหลายทางเลือก:

| Library | Use Case | Ecosystem |
|---------|---------|---------|
| Breeze | Numerical computing, custom algorithms | Scala native |
| Spark MLlib | Large-scale ML, distributed training | Big data |
| Flink ML | Streaming ML, online learning | Real-time |
| DL4J (Deeplearning4j) | Deep learning | JVM DL |

### Best Practices

```
1. Feature engineering มักสำคัญกว่าการเลือก algorithm
2. Cross-validation เสมอ อย่า overfit กับ test set
3. Monitor model drift ใน production
4. Document data lineage และ model decisions
5. Version control models เหมือน code
```

---

*[← ส่วนที่ 82: Distributed Streaming Systems](part-82-streaming-systems.md) | [ส่วนที่ 84: Protocol Buffers and Binary Formats →](part-84-protocol-buffers.md)*
