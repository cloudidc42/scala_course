# ส่วนที่ 79: Scala.js

## สารบัญ

1. [Scala.js Setup และ Build](#scalajs-setup-และ-build)
2. [Interop กับ JavaScript](#interop-กับ-javascript)
3. [DOM Manipulation](#dom-manipulation)
4. [sjs-dom Library](#sjs-dom-library)
5. [AJAX และ Fetch API](#ajax-และ-fetch-api)
6. [React Bindings กับ Slinky](#react-bindings-กับ-slinky)
7. [Shared Code: Scala + Scala.js](#shared-code-scala--scalajs)
8. [Complete Web App Example](#complete-web-app-example)
9. [สรุป](#สรุป)

---

## Scala.js Setup และ Build

Scala.js ช่วยให้เขียน frontend ด้วย Scala ที่ compile เป็น JavaScript

### Project Structure

```
my-fullstack-app/
├── build.sbt
├── project/
│   ├── plugins.sbt
│   └── build.properties
├── shared/
│   └── src/main/scala/
│       └── models/      # shared between JVM and JS
├── jvm/
│   └── src/main/scala/  # server-side code
└── js/
    └── src/main/scala/  # client-side code
```

### Build Configuration

```scala
// project/plugins.sbt
addSbtPlugin("org.scala-js" % "sbt-scalajs" % "1.14.0")
addSbtPlugin("org.portable-scala" % "sbt-scalajs-crossproject" % "1.3.2")
addSbtPlugin("org.scala-js" % "sbt-jsdependencies" % "1.0.2")
```

```scala
// build.sbt
import org.scalajs.sbtplugin.ScalaJSPlugin.autoImport._

lazy val root = project.in(file("."))
  .aggregate(shared.js, shared.jvm, js, jvm)

lazy val shared = crossProject(JSPlatform, JVMPlatform)
  .crossType(CrossType.Pure)
  .in(file("shared"))
  .settings(
    scalaVersion := "3.3.1",
    libraryDependencies ++= Seq(
      "io.circe" %%% "circe-core" % "0.14.6",
      "io.circe" %%% "circe-generic" % "0.14.6",
      "io.circe" %%% "circe-parser" % "0.14.6"
    )
  )

lazy val js = project.in(file("js"))
  .enablePlugins(ScalaJSPlugin)
  .dependsOn(shared.js)
  .settings(
    scalaVersion := "3.3.1",
    scalaJSUseMainModuleInitializer := true,
    
    // Linking mode
    scalaJSLinkerConfig ~= {
      _.withModuleKind(ModuleKind.ESModule)
       .withSourceMap(true)
    },
    
    libraryDependencies ++= Seq(
      "org.scala-js" %%% "scalajs-dom" % "2.8.0",
      "me.shadaj" %%% "slinky-core" % "0.7.4",
      "me.shadaj" %%% "slinky-web" % "0.7.4",
      "com.lihaoyi" %%% "upickle" % "3.1.3"
    ),
    
    // NPM dependencies
    npmDependencies in Compile ++= Seq(
      "react" -> "18.2.0",
      "react-dom" -> "18.2.0"
    )
  )

lazy val jvm = project.in(file("jvm"))
  .dependsOn(shared.jvm)
  .settings(
    scalaVersion := "3.3.1",
    libraryDependencies ++= Seq(
      "org.http4s" %% "http4s-ember-server" % "0.23.24",
      "org.http4s" %% "http4s-circe" % "0.23.24"
    )
  )
```

### Main Entry Point

```scala
// js/src/main/scala/Main.scala
import org.scalajs.dom
import scala.scalajs.js
import scala.scalajs.js.annotation.*

@main def main(): Unit =
  dom.document.addEventListener("DOMContentLoaded", (_: dom.Event) => {
    val container = dom.document.getElementById("root")
    if container != null then
      println("Scala.js application starting...")
      App.render(container)
    else
      dom.console.error("Root element not found!")
  })
```

---

## Interop กับ JavaScript

### JavaScript Type Facades

```scala
// js/src/main/scala/facades/JsFacades.scala
package facades

import scala.scalajs.js
import scala.scalajs.js.annotation.*

// Facade สำหรับ JavaScript library
@js.native
@JSImport("moment", JSImport.Default)
class Moment extends js.Object:
  def format(pattern: String): String = js.native
  def fromNow(): String = js.native
  def add(amount: Int, unit: String): Moment = js.native
  def subtract(amount: Int, unit: String): Moment = js.native
  def isBefore(other: Moment): Boolean = js.native
  def isAfter(other: Moment): Boolean = js.native

@js.native
@JSImport("moment", JSImport.Namespace)
object moment extends js.Object:
  def apply(date: String): Moment = js.native
  def apply(timestamp: Long): Moment = js.native
  def apply(): Moment = js.native

// Facade สำหรับ browser APIs
@js.native
@JSGlobal("localStorage")
object LocalStorage extends js.Object:
  def getItem(key: String): String | Null = js.native
  def setItem(key: String, value: String): Unit = js.native
  def removeItem(key: String): Unit = js.native
  def clear(): Unit = js.native
  def length: Int = js.native

// Facade สำหรับ Chart.js
@js.native
@JSImport("chart.js", "Chart")
class Chart(
  canvas: org.scalajs.dom.HTMLCanvasElement,
  config: js.Object
) extends js.Object:
  def update(): Unit = js.native
  def destroy(): Unit = js.native
  var data: js.Dynamic = js.native

// Global JavaScript functions
@js.native
@JSGlobal
object console extends js.Object:
  def log(msg: js.Any): Unit = js.native
  def error(msg: js.Any): Unit = js.native
  def warn(msg: js.Any): Unit = js.native

// Using js.Dynamic for unknown APIs
object DynamicInterop:
  def callUnsafeApi(obj: js.Dynamic): String =
    obj.someMethod("arg1", 42).toString
  
  def createJsObject(): js.Dynamic =
    js.Dynamic.literal(
      name = "John",
      age = 30,
      hobbies = js.Array("reading", "coding")
    )
  
  // Type-safe conversion
  def fromJsValue[A](jsValue: js.Dynamic)(using js.JSON.value.Decoder[A]): Option[A] =
    try
      Some(jsValue.asInstanceOf[A])
    catch
      case _: Exception => None

// js.Promise interop
import scala.concurrent.{Future, Promise}
import scala.scalajs.concurrent.JSExecutionContext.Implicits.queue

def fetchData(url: String): Future[String] =
  val promise = Promise[String]()
  
  val xhr = new org.scalajs.dom.XMLHttpRequest()
  xhr.open("GET", url)
  xhr.onload = (_: org.scalajs.dom.Event) => {
    if xhr.status == 200 then
      promise.success(xhr.responseText)
    else
      promise.failure(new Exception(s"HTTP error: ${xhr.status}"))
  }
  xhr.onerror = (_: org.scalajs.dom.Event) =>
    promise.failure(new Exception("Network error"))
  xhr.send()
  
  promise.future
```

---

## DOM Manipulation

### DOM Operations

```scala
// js/src/main/scala/dom/DomOperations.scala
package dom

import org.scalajs.dom
import org.scalajs.dom.{document, window, Element, HTMLElement, HTMLInputElement}
import scala.scalajs.js

object DomOps:
  
  // Query selectors
  def querySelector(selector: String): Option[Element] =
    Option(document.querySelector(selector))
  
  def querySelectorAll(selector: String): List[Element] =
    val nodeList = document.querySelectorAll(selector)
    (0 until nodeList.length).map(nodeList(_)).toList
  
  // Element creation
  def createElement(tag: String, attributes: Map[String, String] = Map.empty): HTMLElement =
    val el = document.createElement(tag).asInstanceOf[HTMLElement]
    attributes.foreach { (attr, value) => el.setAttribute(attr, value) }
    el
  
  // Event handling
  def addEventListener[E <: dom.Event](
    element: Element,
    eventType: String,
    handler: E => Unit,
    useCapture: Boolean = false
  ): () => Unit =
    val jsHandler: js.Function1[E, Unit] = (e: E) => handler(e)
    element.addEventListener(eventType, jsHandler, useCapture)
    () => element.removeEventListener(eventType, jsHandler)  // cleanup function
  
  // CSS class manipulation
  extension (el: Element)
    def addClass(className: String): Element =
      el.classList.add(className)
      el
    
    def removeClass(className: String): Element =
      el.classList.remove(className)
      el
    
    def toggleClass(className: String): Element =
      el.classList.toggle(className)
      el
    
    def hasClass(className: String): Boolean =
      el.classList.contains(className)

// Virtual DOM operations
class VirtualDomBuilder:
  
  def createTodoList(todos: List[Todo]): HTMLElement =
    val container = document.createElement("div").asInstanceOf[HTMLElement]
    container.className = "todo-container"
    
    val header = document.createElement("h1").asInstanceOf[HTMLElement]
    header.textContent = "Todo List"
    container.appendChild(header)
    
    val list = document.createElement("ul").asInstanceOf[HTMLElement]
    list.className = "todo-list"
    
    todos.foreach { todo =>
      val item = createTodoItem(todo)
      list.appendChild(item)
    }
    
    container.appendChild(list)
    container
  
  private def createTodoItem(todo: Todo): HTMLElement =
    val li = document.createElement("li").asInstanceOf[HTMLElement]
    li.className = if todo.completed then "todo-item completed" else "todo-item"
    li.dataset("id") = todo.id.toString
    
    val checkbox = document.createElement("input").asInstanceOf[HTMLInputElement]
    checkbox.type = "checkbox"
    checkbox.checked = todo.completed
    
    val text = document.createElement("span").asInstanceOf[HTMLElement]
    text.textContent = todo.title
    
    val deleteBtn = document.createElement("button").asInstanceOf[HTMLElement]
    deleteBtn.textContent = "ลบ"
    deleteBtn.className = "delete-btn"
    
    li.appendChild(checkbox)
    li.appendChild(text)
    li.appendChild(deleteBtn)
    li

case class Todo(id: Int, title: String, completed: Boolean)
```

---

## sjs-dom Library

### DOM Event Handling

```scala
// js/src/main/scala/dom/SjsDomExamples.scala
package dom

import org.scalajs.dom
import org.scalajs.dom.*
import scala.scalajs.js

// Form handling
class FormHandler(formId: String):
  
  def setup(): Unit =
    val form = document.getElementById(formId).asInstanceOf[HTMLFormElement]
    
    form.addEventListener("submit", (e: dom.Event) => {
      e.preventDefault()
      handleSubmit(form)
    })
    
    // Real-time validation
    form.querySelectorAll("input[required]").foreach { input =>
      input.addEventListener("blur", (_: dom.Event) => {
        validateField(input.asInstanceOf[HTMLInputElement])
      })
    }
  
  private def handleSubmit(form: HTMLFormElement): Unit =
    val formData = new FormData(form)
    val data = Map(
      "name" -> formData.get("name").asInstanceOf[String],
      "email" -> formData.get("email").asInstanceOf[String]
    )
    println(s"Form submitted: $data")
  
  private def validateField(input: HTMLInputElement): Boolean =
    val isValid = input.validity.valid
    
    if isValid then
      input.classList.remove("error")
      input.classList.add("valid")
    else
      input.classList.remove("valid")
      input.classList.add("error")
      showValidationMessage(input)
    
    isValid
  
  private def showValidationMessage(input: HTMLInputElement): Unit =
    val message = input.validationMessage
    var errorDiv = input.nextElementSibling
    
    if errorDiv == null || !errorDiv.classList.contains("error-message") then
      errorDiv = document.createElement("div")
      errorDiv.className = "error-message"
      input.parentNode.insertBefore(errorDiv, input.nextSibling)
    
    errorDiv.textContent = message

// Intersection Observer API
class LazyLoader:
  
  def setup(): Unit =
    val options = js.Dynamic.literal(
      root = null,
      rootMargin = "0px",
      threshold = 0.1
    )
    
    val observer = new IntersectionObserver(
      (entries: js.Array[IntersectionObserverEntry], _: IntersectionObserver) => {
        entries.foreach { entry =>
          if entry.isIntersecting then
            loadElement(entry.target.asInstanceOf[HTMLElement])
        }
      },
      options
    )
    
    document.querySelectorAll("[data-lazy]").foreach { el =>
      observer.observe(el)
    }
  
  private def loadElement(el: HTMLElement): Unit =
    el.dataset.get("lazy").foreach { src =>
      el match
        case img: HTMLImageElement =>
          img.src = src
          img.classList.remove("lazy")
        case _ =>
          el.style.backgroundImage = s"url($src)"
    }

// Web Workers
class WorkerManager:
  
  def createWorker(scriptUrl: String): Worker =
    new Worker(scriptUrl)
  
  def runInWorker(worker: Worker, data: js.Any): scala.concurrent.Future[js.Any] =
    val p = scala.concurrent.Promise[js.Any]()
    
    worker.onmessage = (e: MessageEvent) => {
      p.success(e.data)
    }
    
    worker.onerror = (e: ErrorEvent) => {
      p.failure(new Exception(e.message))
    }
    
    worker.postMessage(data)
    p.future

// Canvas API
class CanvasDrawer(canvasId: String):
  private lazy val canvas = document.getElementById(canvasId).asInstanceOf[HTMLCanvasElement]
  private lazy val ctx = canvas.getContext("2d").asInstanceOf[CanvasRenderingContext2D]
  
  def drawBarChart(data: List[(String, Double)], title: String): Unit =
    val width = canvas.width.toDouble
    val height = canvas.height.toDouble
    val padding = 50.0
    val barWidth = (width - padding * 2) / data.length - 10
    val maxValue = data.map(_._2).maxOption.getOrElse(1.0)
    
    // Clear canvas
    ctx.clearRect(0, 0, width, height)
    
    // Draw title
    ctx.font = "bold 16px Arial"
    ctx.textAlign = "center"
    ctx.fillText(title, width / 2, 30)
    
    // Draw bars
    data.zipWithIndex.foreach { ((label, value), i) =>
      val barHeight = (value / maxValue) * (height - padding * 2)
      val x = padding + i * (barWidth + 10)
      val y = height - padding - barHeight
      
      // Bar color
      ctx.fillStyle = s"hsl(${i * 360 / data.length}, 70%, 50%)"
      ctx.fillRect(x, y, barWidth, barHeight)
      
      // Label
      ctx.fillStyle = "#333"
      ctx.font = "12px Arial"
      ctx.textAlign = "center"
      ctx.fillText(label, x + barWidth / 2, height - padding + 15)
      
      // Value
      ctx.fillText(value.toString, x + barWidth / 2, y - 5)
    }
    
    // Axes
    ctx.beginPath()
    ctx.moveTo(padding, padding)
    ctx.lineTo(padding, height - padding)
    ctx.lineTo(width - padding, height - padding)
    ctx.strokeStyle = "#333"
    ctx.lineWidth = 2
    ctx.stroke()
```

---

## AJAX และ Fetch API

### HTTP Client

```scala
// js/src/main/scala/http/FetchClient.scala
package http

import org.scalajs.dom
import org.scalajs.dom.{Fetch, Headers, RequestInit, Response}
import scala.scalajs.js
import scala.scalajs.js.Thenable.Implicits._
import scala.concurrent.Future
import scala.scalajs.concurrent.JSExecutionContext.Implicits.queue
import io.circe.*
import io.circe.parser.*
import io.circe.syntax.*

// HTTP client abstraction
case class HttpError(status: Int, message: String) extends Exception(message)

case class ApiResponse[A](
  data: A,
  status: Int,
  headers: Map[String, String]
)

class FetchHttpClient(baseUrl: String, defaultHeaders: Map[String, String] = Map.empty):
  
  def get[A: Decoder](path: String): Future[Either[String, A]] =
    request(path, "GET", None)
  
  def post[B: Encoder, A: Decoder](path: String, body: B): Future[Either[String, A]] =
    request(path, "POST", Some(body.asJson.noSpaces))
  
  def put[B: Encoder, A: Decoder](path: String, body: B): Future[Either[String, A]] =
    request(path, "PUT", Some(body.asJson.noSpaces))
  
  def delete[A: Decoder](path: String): Future[Either[String, A]] =
    request(path, "DELETE", None)
  
  private def request[A: Decoder](
    path: String,
    method: String,
    body: Option[String]
  ): Future[Either[String, A]] =
    val headers = new Headers()
    (defaultHeaders ++ Map("Content-Type" -> "application/json"))
      .foreach { (k, v) => headers.append(k, v) }
    
    val init = js.Dynamic.literal(
      method = method,
      headers = headers,
      body = body.map(js.Any.fromString).getOrElse(js.undefined)
    ).asInstanceOf[RequestInit]
    
    Fetch.fetch(s"$baseUrl$path", init)
      .toFuture
      .flatMap { response =>
        response.text().toFuture.map { text =>
          if response.ok then
            decode[A](text).left.map(_.message)
          else
            Left(s"HTTP ${response.status}: $text")
        }
      }
      .recover { case e => Left(e.getMessage) }

// Using the client
case class Product(id: String, name: String, price: Double)
case class CreateProductRequest(name: String, price: Double)

class ProductApiClient(client: FetchHttpClient):
  
  def listProducts(): Future[Either[String, List[Product]]] =
    client.get[List[Product]]("/api/v1/products")
  
  def createProduct(request: CreateProductRequest): Future[Either[String, Product]] =
    client.post[CreateProductRequest, Product]("/api/v1/products", request)
  
  def getProduct(id: String): Future[Either[String, Product]] =
    client.get[Product](s"/api/v1/products/$id")

// Retry logic
def withRetry[A](
  operation: () => Future[Either[String, A]],
  maxRetries: Int = 3,
  delayMs: Int = 1000
): Future[Either[String, A]] =
  import scala.concurrent.Promise
  
  def attempt(retriesLeft: Int): Future[Either[String, A]] =
    operation().flatMap {
      case Left(error) if retriesLeft > 0 =>
        println(s"Retrying... ($retriesLeft attempts left)")
        val p = Promise[Either[String, A]]()
        dom.window.setTimeout(
          () => attempt(retriesLeft - 1).foreach(p.success),
          delayMs
        )
        p.future
      case result => Future.successful(result)
    }
  
  attempt(maxRetries)
```

---

## React Bindings กับ Slinky

### Slinky Components

```scala
// js/src/main/scala/components/Components.scala
package components

import slinky.core.*
import slinky.core.annotations.react
import slinky.core.facade.ReactElement
import slinky.web.html.*

// Stateless functional component
@react object Button:
  case class Props(
    label: String,
    onClick: () => Unit,
    disabled: Boolean = false,
    variant: String = "primary"
  )
  
  def render(props: Props): ReactElement =
    button(
      className := s"btn btn-${props.variant}",
      onClick := (_ => props.onClick()),
      disabled := props.disabled
    )(props.label)

// Stateful component
@react class Counter extends Component:
  case class Props(initialValue: Int = 0, step: Int = 1)
  
  case class State(count: Int)
  
  def initialState = State(props.initialValue)
  
  def render(): ReactElement =
    div(className := "counter")(
      button(onClick := (_ => setState(s => s.copy(count = s.count - props.step))))("-"),
      span(className := "count")(state.count.toString),
      button(onClick := (_ => setState(s => s.copy(count = s.count + props.step))))("+ ")
    )

// Hooks with Slinky
import slinky.core.facade.Hooks.*

@react object TodoApp:
  case class Props(apiUrl: String)
  
  case class TodoItem(id: Int, text: String, completed: Boolean)
  
  def render(props: Props): ReactElement =
    val (todos, setTodos) = useState(List.empty[TodoItem])
    val (inputText, setInputText) = useState("")
    val (loading, setLoading) = useState(false)
    val (error, setError) = useState(Option.empty[String])
    
    useEffect(
      () => {
        setLoading(true)
        import scala.scalajs.concurrent.JSExecutionContext.Implicits.queue
        
        val client = FetchHttpClient(props.apiUrl)
        client.get[List[TodoItem]]("/todos").foreach {
          case Right(items) =>
            setTodos(items)
            setLoading(false)
          case Left(err) =>
            setError(Some(err))
            setLoading(false)
        }
        () => ()
      },
      Seq.empty
    )
    
    div(className := "todo-app")(
      h1()("Todo List"),
      
      if loading then div(className := "loading")("กำลังโหลด...")
      else if error.isDefined then div(className := "error")(s"ข้อผิดพลาด: ${error.get}")
      else div()(
        // Input form
        div(className := "todo-input")(
          input(
            type_ := "text",
            value := inputText,
            onChange := (e => setInputText(e.target.asInstanceOf[org.scalajs.dom.HTMLInputElement].value)),
            placeholder := "เพิ่ม Todo..."
          ),
          button(
            onClick := (_ => {
              if inputText.nonEmpty then
                val newTodo = TodoItem(todos.length + 1, inputText, false)
                setTodos(todos :+ newTodo)
                setInputText("")
            })
          )("เพิ่ม")
        ),
        
        // Todo list
        ul(className := "todo-list")(
          todos.map { todo =>
            li(
              key := todo.id.toString,
              className := (if todo.completed then "completed" else "")
            )(
              input(
                type_ := "checkbox",
                checked := todo.completed,
                onChange := (_ => setTodos(
                  todos.map(t => if t.id == todo.id then t.copy(completed = !t.completed) else t)
                ))
              ),
              span()(todo.text),
              button(
                className := "delete",
                onClick := (_ => setTodos(todos.filterNot(_.id == todo.id)))
              )("ลบ")
            )
          }*
        ),
        
        // Statistics
        div(className := "stats")(
          span()(s"ทั้งหมด: ${todos.length}"),
          span()(s"เสร็จแล้ว: ${todos.count(_.completed)}"),
          span()(s"ยังไม่เสร็จ: ${todos.count(!_.completed)}")
        )
      )
    )
```

### Context API

```scala
// js/src/main/scala/context/AppContext.scala
package context

import slinky.core.*
import slinky.core.facade.{ReactContext, ReactElement}
import slinky.core.annotations.react

// Theme context
case class Theme(primaryColor: String, darkMode: Boolean)

object ThemeContext:
  val context: ReactContext[Theme] = React.createContext(
    Theme(primaryColor = "#007bff", darkMode = false)
  )

@react object ThemeProvider:
  case class Props(theme: Theme, children: ReactElement)
  
  def render(props: Props): ReactElement =
    ThemeContext.context.Provider(value = props.theme)(props.children)

// Using context in component
@react object ThemedButton:
  case class Props(label: String, onClick: () => Unit)
  
  def render(props: Props): ReactElement =
    val theme = useContext(ThemeContext.context)
    
    button(
      style := js.Dynamic.literal(
        backgroundColor = theme.primaryColor,
        color = if theme.darkMode then "white" else "black"
      ),
      onClick := (_ => props.onClick())
    )(props.label)
```

---

## Shared Code: Scala + Scala.js

### Cross-Platform Models

```scala
// shared/src/main/scala/models/SharedModels.scala
package models

import io.circe.*
import io.circe.generic.auto.*

// Models ที่ใช้ได้ทั้ง JVM และ JS
case class User(
  id: String,
  email: String,
  name: String,
  role: UserRole
)

enum UserRole derives Encoder, Decoder:
  case Admin, User, Guest

case class Product(
  id: String,
  name: String,
  description: String,
  price: BigDecimal,
  category: String,
  stock: Int
)

case class Order(
  id: String,
  userId: String,
  items: List[OrderItem],
  status: OrderStatus,
  totalAmount: BigDecimal,
  createdAt: String
)

case class OrderItem(
  productId: String,
  quantity: Int,
  price: BigDecimal
)

enum OrderStatus derives Encoder, Decoder:
  case Pending, Confirmed, Shipped, Delivered, Cancelled

// Validation logic - เหมือนกันทั้ง server และ client
object Validation:
  
  case class ValidationError(field: String, message: String)
  
  def validateEmail(email: String): Either[ValidationError, String] =
    val emailRegex = "^[A-Za-z0-9+_.-]+@(.+)$".r
    if emailRegex.matches(email) then Right(email)
    else Left(ValidationError("email", "รูปแบบอีเมลไม่ถูกต้อง"))
  
  def validatePassword(password: String): Either[ValidationError, String] =
    if password.length < 8 then
      Left(ValidationError("password", "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"))
    else if !password.exists(_.isDigit) then
      Left(ValidationError("password", "รหัสผ่านต้องมีตัวเลขอย่างน้อย 1 ตัว"))
    else Right(password)
  
  def validateProduct(name: String, price: BigDecimal): List[ValidationError] =
    List(
      if name.trim.isEmpty then Some(ValidationError("name", "ชื่อสินค้าต้องไม่ว่างเปล่า")) else None,
      if name.length > 100 then Some(ValidationError("name", "ชื่อสินค้าต้องไม่เกิน 100 ตัวอักษร")) else None,
      if price <= 0 then Some(ValidationError("price", "ราคาต้องมากกว่า 0")) else None
    ).flatten

// Shared business logic
object CartCalculations:
  
  def calculateSubtotal(items: List[OrderItem]): BigDecimal =
    items.map(item => item.price * item.quantity).sum
  
  def calculateDiscount(subtotal: BigDecimal, discountPercent: Int): BigDecimal =
    subtotal * discountPercent / 100
  
  def calculateTax(amount: BigDecimal, taxRate: BigDecimal = 0.07): BigDecimal =
    amount * taxRate
  
  def calculateTotal(
    items: List[OrderItem],
    discountPercent: Int = 0,
    taxRate: BigDecimal = 0.07
  ): OrderTotal =
    val subtotal = calculateSubtotal(items)
    val discount = calculateDiscount(subtotal, discountPercent)
    val afterDiscount = subtotal - discount
    val tax = calculateTax(afterDiscount, taxRate)
    val total = afterDiscount + tax
    
    OrderTotal(
      subtotal = subtotal,
      discount = discount,
      tax = tax,
      total = total
    )

case class OrderTotal(
  subtotal: BigDecimal,
  discount: BigDecimal,
  tax: BigDecimal,
  total: BigDecimal
)
```

---

## Complete Web App Example

### Shopping Cart Application

```scala
// js/src/main/scala/app/ShoppingCart.scala
package app

import slinky.core.*
import slinky.core.annotations.react
import slinky.core.facade.{ReactElement, Hooks}
import Hooks.*
import slinky.web.html.*
import models.*
import http.*
import scala.scalajs.concurrent.JSExecutionContext.Implicits.queue
import scala.concurrent.Future

// State management
case class AppState(
  products: List[Product] = List.empty,
  cart: List[CartItem] = List.empty,
  currentUser: Option[User] = None,
  loading: Boolean = false,
  error: Option[String] = None
)

case class CartItem(
  product: Product,
  quantity: Int
):
  def subtotal: BigDecimal = product.price * quantity

// Main App
@react object App:
  type Props = Unit
  
  def render(props: Props): ReactElement =
    val (state, setState) = useState(AppState(loading = true))
    val apiClient = ProductApiClient(FetchHttpClient("http://localhost:8080"))
    
    useEffect(() => {
      apiClient.listProducts().foreach {
        case Right(products) =>
          setState(s => s.copy(products = products.map(p =>
            Product(p.id, p.name, "", BigDecimal(p.price), "", 100)
          ), loading = false))
        case Left(err) =>
          setState(s => s.copy(error = Some(err), loading = false))
      }
      () => ()
    }, Seq.empty)
    
    val addToCart: Product => Unit = product =>
      setState { s =>
        s.cart.find(_.product.id == product.id) match
          case Some(item) =>
            s.copy(cart = s.cart.map(i =>
              if i.product.id == product.id then i.copy(quantity = i.quantity + 1)
              else i
            ))
          case None =>
            s.copy(cart = s.cart :+ CartItem(product, 1))
      }
    
    val removeFromCart: String => Unit = productId =>
      setState(s => s.copy(cart = s.cart.filterNot(_.product.id == productId)))
    
    val updateQuantity: (String, Int) => Unit = (productId, qty) =>
      if qty <= 0 then removeFromCart(productId)
      else setState(s => s.copy(cart = s.cart.map(item =>
        if item.product.id == productId then item.copy(quantity = qty) else item
      )))
    
    div(className := "app")(
      Header(state.currentUser, state.cart.length),
      
      div(className := "main-content")(
        div(className := "product-section")(
          h2()("สินค้าทั้งหมด"),
          
          if state.loading then LoadingSpinner()
          else if state.error.isDefined then ErrorMessage(state.error.get)
          else ProductGrid(state.products, addToCart)
        ),
        
        ShoppingCartComponent(
          items = state.cart,
          onRemove = removeFromCart,
          onUpdateQuantity = updateQuantity,
          onCheckout = () => println("Checkout!")
        )
      )
    )

// Header component
@react object Header:
  case class Props(user: Option[User], cartCount: Int)
  
  def render(props: Props): ReactElement =
    header(className := "app-header")(
      h1()("Scala Shop"),
      nav()(
        a(href := "/")("หน้าแรก"),
        a(href := "/products")("สินค้า"),
        props.user.fold(
          a(href := "/login")("เข้าสู่ระบบ")
        )(u => span()(s"สวัสดี ${u.name}"))
      ),
      div(className := "cart-icon")(
        span(className := "cart-count")(props.cartCount.toString)
      )
    )

// Product Grid
@react object ProductGrid:
  case class Props(products: List[Product], onAddToCart: Product => Unit)
  
  def render(props: Props): ReactElement =
    div(className := "product-grid")(
      props.products.map { product =>
        ProductCard(product, () => props.onAddToCart(product))
      }*
    )

// Product Card
@react object ProductCard:
  case class Props(product: Product, onAddToCart: () => Unit)
  
  def render(props: Props): ReactElement =
    div(className := "product-card", key := props.product.id)(
      div(className := "product-image")(
        img(src := "/placeholder.jpg", alt := props.product.name)
      ),
      div(className := "product-info")(
        h3()(props.product.name),
        p(className := "description")(props.product.description),
        div(className := "price-section")(
          span(className := "price")(s"฿${props.product.price}"),
          span(className := "stock")(
            if props.product.stock > 0 then s"เหลือ ${props.product.stock} ชิ้น"
            else "สินค้าหมด"
          )
        ),
        button(
          className := "add-to-cart-btn",
          disabled := props.product.stock <= 0,
          onClick := (_ => props.onAddToCart())
        )("เพิ่มลงตะกร้า")
      )
    )

// Shopping Cart Component
@react object ShoppingCartComponent:
  case class Props(
    items: List[CartItem],
    onRemove: String => Unit,
    onUpdateQuantity: (String, Int) => Unit,
    onCheckout: () => Unit
  )
  
  def render(props: Props): ReactElement =
    val total = CartCalculations.calculateTotal(
      props.items.map(i => OrderItem(i.product.id, i.quantity, i.product.price))
    )
    
    div(className := "shopping-cart")(
      h2()("ตะกร้าสินค้า"),
      
      if props.items.isEmpty then
        p(className := "empty-cart")("ตะกร้าว่างเปล่า")
      else
        div()(
          div(className := "cart-items")(
            props.items.map { item =>
              CartItemRow(
                item,
                () => props.onRemove(item.product.id),
                (qty: Int) => props.onUpdateQuantity(item.product.id, qty)
              )
            }*
          ),
          
          div(className := "cart-summary")(
            div(className := "summary-row")(
              span()("ราคารวม:"),
              span()(s"฿${total.subtotal}")
            ),
            div(className := "summary-row")(
              span()("ภาษี (7%):"),
              span()(s"฿${total.tax.setScale(2, BigDecimal.RoundingMode.HALF_UP)}")
            ),
            div(className := "summary-row total")(
              span()("รวมทั้งหมด:"),
              span(className := "total-price")(s"฿${total.total.setScale(2, BigDecimal.RoundingMode.HALF_UP)}")
            ),
            
            button(
              className := "checkout-btn",
              onClick := (_ => props.onCheckout())
            )("ดำเนินการชำระเงิน")
          )
        )
    )

@react object CartItemRow:
  case class Props(
    item: CartItem,
    onRemove: () => Unit,
    onUpdateQuantity: Int => Unit
  )
  
  def render(props: Props): ReactElement =
    div(className := "cart-item")(
      span(className := "item-name")(props.item.product.name),
      div(className := "quantity-control")(
        button(onClick := (_ => props.onUpdateQuantity(props.item.quantity - 1)))("-"),
        span()(props.item.quantity.toString),
        button(onClick := (_ => props.onUpdateQuantity(props.item.quantity + 1)))("+")
      ),
      span(className := "item-price")(s"฿${props.item.subtotal}"),
      button(className := "remove-btn", onClick := (_ => props.onRemove()))("×")
    )

// Loading and Error components
@react object LoadingSpinner:
  type Props = Unit
  def render(props: Props): ReactElement =
    div(className := "loading-spinner")("กำลังโหลด...")

@react object ErrorMessage:
  case class Props(message: String)
  def render(props: Props): ReactElement =
    div(className := "error-message")(s"ข้อผิดพลาด: ${props.message}")
```

### Application Entry Point

```scala
// js/src/main/scala/Main.scala
import org.scalajs.dom
import slinky.web.ReactDOM
import app.App

@main def main(): Unit =
  dom.document.addEventListener("DOMContentLoaded", (_: dom.Event) => {
    val container = dom.document.getElementById("root")
    if container != null then
      ReactDOM.render(App(()), container)
    else
      dom.console.error("ไม่พบ element 'root'")
  })
```

### HTML Template

```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Scala Shop</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>
  <div id="root">กำลังโหลด...</div>
  <script type="module" src="main.js"></script>
</body>
</html>
```

### CSS Styles

```css
/* styles.css */
* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: 'Sarabun', sans-serif;
  background: #f5f5f5;
  color: #333;
}

.app-header {
  background: #1a73e8;
  color: white;
  padding: 1rem 2rem;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1.5rem;
  padding: 1rem;
}

.product-card {
  background: white;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  transition: transform 0.2s;
}

.product-card:hover { transform: translateY(-4px); }

.add-to-cart-btn {
  background: #1a73e8;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.add-to-cart-btn:disabled {
  background: #ccc;
  cursor: not-allowed;
}
```

---

## สรุป

Scala.js ทำให้เราสามารถเขียน frontend ด้วย Scala ได้อย่างสมบูรณ์:

1. **Setup**: ใช้ sbt กับ scala-js plugin และ cross-project
2. **JavaScript Interop**: facades และ `@JSImport` สำหรับ JS libraries
3. **DOM Manipulation**: ใช้ scalajs-dom สำหรับ browser APIs
4. **Fetch API**: HTTP client แบบ type-safe
5. **React (Slinky)**: Component-based UI ด้วย React bindings
6. **Shared Code**: Models และ business logic เดียวกันสำหรับ JVM และ JS
7. **State Management**: Hooks และ Context API
8. **Cross-compilation**: ตรวจสอบ type safety ทั้ง server และ client
9. **Performance**: Scala.js generates optimized JavaScript
10. **Full-stack**: สร้าง web app แบบ full-stack ด้วย Scala

Scala.js เหมาะสำหรับ teams ที่ต้องการใช้ Scala ทั้ง backend และ frontend พร้อมกับ share code และ type-safety ตลอดทั้ง stack

---

*[← ส่วนที่ 78: Event-Driven Architecture](part-78-event-driven.md)*
