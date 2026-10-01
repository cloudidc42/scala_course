# ตอนที่ 88: Web Scraping with Scala

## สารบัญ

1. [http4s Client สำหรับ HTTP Requests](#http4s-client-สำหรับ-http-requests)
2. [HTML Parsing ด้วย jsoup](#html-parsing-ด้วย-jsoup)
3. [CSS Selectors และ XPath](#css-selectors-และ-xpath)
4. [Pagination และ Crawling](#pagination-และ-crawling)
5. [Rate Limiting และ Polite Crawling](#rate-limiting-และ-polite-crawling)
6. [Data Extraction และ Transformation](#data-extraction-และ-transformation)
7. [Complete Scraper Example](#complete-scraper-example)
8. [สรุป](#สรุป)

---

## http4s Client สำหรับ HTTP Requests

### การตั้งค่า build.sbt

```scala
// build.sbt
ThisBuild / scalaVersion := "3.3.1"

lazy val root = (project in file("."))
  .settings(
    name := "scala-web-scraper",
    libraryDependencies ++= Seq(
      // HTTP Client
      "org.http4s"    %% "http4s-ember-client" % "0.23.23",
      "org.http4s"    %% "http4s-circe"        % "0.23.23",
      "org.http4s"    %% "http4s-dsl"          % "0.23.23",
      
      // Cats Effect
      "org.typelevel" %% "cats-effect"         % "3.5.2",
      
      // HTML Parsing
      "org.jsoup"      % "jsoup"               % "1.17.1",
      
      // JSON
      "io.circe"      %% "circe-core"          % "0.14.6",
      "io.circe"      %% "circe-generic"       % "0.14.6",
      
      // CSV
      "com.github.tototoshi" %% "scala-csv"    % "1.3.10",
      
      // Logging
      "org.typelevel" %% "log4cats-slf4j"      % "2.6.0",
      "org.slf4j"      % "slf4j-simple"        % "2.0.9"
    )
  )
```

### HTTP Client พื้นฐาน

```scala
package com.example.scraper

import cats.effect.*
import org.http4s.*
import org.http4s.ember.client.*
import org.http4s.client.Client
import org.http4s.headers.*
import org.typelevel.ci.*
import scala.concurrent.duration.*

// HTTP Client configuration
object HttpClientConfig:
  
  def create(): Resource[IO, Client[IO]] =
    EmberClientBuilder
      .default[IO]
      .withTimeout(30.seconds)
      .withIdleTimeInPool(60.seconds)
      .withMaxTotal(20)        // Max connections
      .withMaxPerKey(_ => 5)   // Max connections per host
      .build
  
  // Client พร้อม headers ที่เหมาะสม
  def politeClient(client: Client[IO]): Client[IO] =
    Client: req =>
      val politeReq = req
        .putHeaders(
          `User-Agent`(ProductId("ScalaBot", Some("1.0"))),
          Header.Raw(ci"Accept", "text/html,application/xhtml+xml"),
          Header.Raw(ci"Accept-Language", "th-TH,th;q=0.9,en;q=0.8"),
          Header.Raw(ci"Accept-Encoding", "gzip, deflate")
        )
      client.run(politeReq)
```

### การดึง HTML

```scala
package com.example.scraper

import cats.effect.*
import org.http4s.*
import org.http4s.client.Client
import org.http4s.Status

sealed trait FetchError
case class HttpError(status: Status, url: String) extends FetchError
case class NetworkError(message: String, url: String) extends FetchError
case class ParseError(message: String, url: String) extends FetchError

class HttpFetcher(client: Client[IO]):
  
  def fetchHtml(url: String): IO[Either[FetchError, String]] =
    Uri.fromString(url) match
      case Left(err) => IO.pure(Left(NetworkError(err.message, url)))
      case Right(uri) =>
        client.run(Request[IO](uri = uri)).use: response =>
          if response.status.isSuccess then
            response.bodyText.compile.string
              .map(html => Right(html))
              .handleError(e => Left(NetworkError(e.getMessage, url)))
          else
            IO.pure(Left(HttpError(response.status, url)))
        .handleError: e =>
          Left(NetworkError(e.getMessage, url))
  
  def fetchHtmlWithRetry(url: String, maxRetries: Int = 3): IO[Either[FetchError, String]] =
    def attempt(retriesLeft: Int): IO[Either[FetchError, String]] =
      fetchHtml(url).flatMap:
        case Right(html) => IO.pure(Right(html))
        case Left(err) if retriesLeft > 0 =>
          IO.println(s"Retry $url (${maxRetries - retriesLeft + 1}/$maxRetries)") >>
          IO.sleep(scala.concurrent.duration.Duration.fromNanos(2000000000L * (maxRetries - retriesLeft + 1))) >>
          attempt(retriesLeft - 1)
        case Left(err) => IO.pure(Left(err))
    
    attempt(maxRetries)
  
  def fetchMultiple(urls: List[String], parallelism: Int = 5): IO[List[(String, Either[FetchError, String])]] =
    import cats.syntax.all.*
    
    urls
      .grouped(parallelism)
      .toList
      .flatTraverse: batch =>
        batch.parTraverse: url =>
          fetchHtml(url).map(url -> _)
```

---

## HTML Parsing ด้วย jsoup

### Jsoup Wrapper สำหรับ Scala

```scala
package com.example.scraper

import org.jsoup.Jsoup
import org.jsoup.nodes.{Document, Element}
import org.jsoup.select.Elements
import scala.jdk.CollectionConverters.*
import cats.effect.IO

class HtmlParser:
  
  def parse(html: String, baseUrl: String = ""): Either[ParseError, Document] =
    try
      Right(Jsoup.parse(html, baseUrl))
    catch
      case e: Exception =>
        Left(ParseError(e.getMessage, baseUrl))
  
  def parseUrl(url: String): IO[Either[ParseError, Document]] =
    IO:
      try Right(Jsoup.connect(url).get())
      catch
        case e: Exception => Left(ParseError(e.getMessage, url))

// Extension methods สำหรับ Document และ Element
extension (doc: Document)
  
  def selectFirst(cssSelector: String): Option[Element] =
    Option(doc.selectFirst(cssSelector))
  
  def selectAll(cssSelector: String): List[Element] =
    doc.select(cssSelector).asScala.toList
  
  def title: String = doc.title()
  
  def meta(name: String): Option[String] =
    Option(doc.selectFirst(s"meta[name=$name]"))
      .flatMap(e => Option(e.attr("content")))
  
  def metaProperty(property: String): Option[String] =
    Option(doc.selectFirst(s"meta[property=$property]"))
      .flatMap(e => Option(e.attr("content")))

extension (element: Element)
  
  def text: String = element.text()
  def html: String = element.html()
  def outerHtml: String = element.outerHtml()
  
  def attr(name: String): Option[String] =
    val value = element.attr(name)
    if value.isEmpty then None else Some(value)
  
  def absUrl(name: String): Option[String] =
    val url = element.absUrl(name)
    if url.isEmpty then None else Some(url)
  
  def selectFirst(cssSelector: String): Option[Element] =
    Option(element.selectFirst(cssSelector))
  
  def selectAll(cssSelector: String): List[Element] =
    element.select(cssSelector).asScala.toList
  
  def children: List[Element] =
    element.children().asScala.toList
  
  def siblings: List[Element] =
    element.siblingElements().asScala.toList
  
  def parent: Option[Element] =
    Option(element.parent())
```

### การ Extract ข้อมูลจาก HTML

```scala
package com.example.scraper

import org.jsoup.nodes.Document

// Data models
case class Article(
  title: String,
  url: String,
  author: Option[String],
  publishDate: Option[String],
  category: Option[String],
  tags: List[String],
  summary: Option[String],
  imageUrl: Option[String]
)

case class Product(
  name: String,
  price: Option[Double],
  currency: String,
  sku: Option[String],
  imageUrl: Option[String],
  rating: Option[Double],
  reviewCount: Option[Int],
  availability: String,
  description: Option[String]
)

// Article extractor
class ArticleExtractor:
  
  def extract(doc: Document, url: String): Option[Article] =
    val title = extractTitle(doc)
    if title.isEmpty then None
    else Some(Article(
      title       = title,
      url         = url,
      author      = extractAuthor(doc),
      publishDate = extractPublishDate(doc),
      category    = extractCategory(doc),
      tags        = extractTags(doc),
      summary     = extractSummary(doc),
      imageUrl    = extractMainImage(doc)
    ))
  
  private def extractTitle(doc: Document): String =
    // ลอง selectors หลายแบบ
    val selectors = List(
      "h1.article-title",
      "h1.post-title",
      "h1[itemprop=headline]",
      "meta[property=og:title]",
      "h1"
    )
    
    selectors
      .flatMap: sel =>
        if sel.startsWith("meta") then
          doc.selectFirst(sel).flatMap(_.attr("content"))
        else
          doc.selectFirst(sel).map(_.text)
      .headOption
      .getOrElse(doc.title())
  
  private def extractAuthor(doc: Document): Option[String] =
    val selectors = List(
      "a[rel=author]",
      "span[itemprop=author]",
      "[class*=author]",
      "meta[name=author]"
    )
    
    selectors.flatMap: sel =>
      if sel.startsWith("meta") then
        doc.selectFirst(sel).flatMap(_.attr("content"))
      else
        doc.selectFirst(sel).map(_.text).filter(_.nonEmpty)
    .headOption
  
  private def extractPublishDate(doc: Document): Option[String] =
    val selectors = List(
      "time[datetime]",
      "time[pubdate]",
      "[itemprop=datePublished]",
      "meta[name=publish_date]",
      "meta[property=article:published_time]"
    )
    
    selectors.flatMap: sel =>
      val element = doc.selectFirst(sel)
      element.flatMap: e =>
        e.attr("datetime")
          .orElse(e.attr("content"))
          .orElse(Some(e.text).filter(_.nonEmpty))
    .headOption
  
  private def extractCategory(doc: Document): Option[String] =
    doc.selectFirst("[class*=category], [class*=section], nav.breadcrumb li:last-child")
      .map(_.text)
  
  private def extractTags(doc: Document): List[String] =
    val tagSelectors = List(
      "a[rel=tag]",
      "[class*=tag] a",
      ".tags a",
      ".labels a"
    )
    
    tagSelectors.flatMap: sel =>
      doc.selectAll(sel).map(_.text).filter(_.nonEmpty)
    .distinct
  
  private def extractSummary(doc: Document): Option[String] =
    val selectors = List(
      "meta[name=description]",
      "meta[property=og:description]",
      "[itemprop=description]",
      ".article-summary",
      ".excerpt"
    )
    
    selectors.flatMap: sel =>
      if sel.startsWith("meta") then
        doc.selectFirst(sel).flatMap(_.attr("content"))
      else
        doc.selectFirst(sel).map(_.text).filter(_.nonEmpty)
    .headOption
  
  private def extractMainImage(doc: Document): Option[String] =
    val selectors = List(
      "meta[property=og:image]",
      "[itemprop=image]",
      ".article-image img",
      "article img",
      ".post-thumbnail img"
    )
    
    selectors.flatMap: sel =>
      if sel.startsWith("meta") then
        doc.selectFirst(sel).flatMap(_.attr("content"))
      else
        doc.selectFirst(sel)
          .flatMap(e => e.absUrl("src").orElse(e.attr("src")))
    .headOption

// Product extractor
class ProductExtractor:
  
  def extract(doc: Document, url: String): Option[Product] =
    extractName(doc).map: name =>
      Product(
        name         = name,
        price        = extractPrice(doc),
        currency     = extractCurrency(doc),
        sku          = extractSku(doc),
        imageUrl     = extractImage(doc),
        rating       = extractRating(doc),
        reviewCount  = extractReviewCount(doc),
        availability = extractAvailability(doc),
        description  = extractDescription(doc)
      )
  
  private def extractName(doc: Document): Option[String] =
    List(
      "h1[itemprop=name]",
      ".product-title h1",
      ".product-name",
      "h1"
    ).flatMap: sel =>
      doc.selectFirst(sel).map(_.text).filter(_.nonEmpty)
    .headOption
  
  private def extractPrice(doc: Document): Option[Double] =
    val priceText = List(
      "[itemprop=price]",
      ".price",
      ".product-price",
      "[class*=price]"
    ).flatMap: sel =>
      doc.selectFirst(sel)
        .flatMap(e => e.attr("content").orElse(Some(e.text)))
        .filter(_.nonEmpty)
    .headOption
    
    priceText.flatMap: text =>
      val cleaned = text.replaceAll("[^0-9.]", "")
      cleaned.toDoubleOption
  
  private def extractCurrency(doc: Document): String =
    doc.selectFirst("[itemprop=priceCurrency]")
      .flatMap(_.attr("content"))
      .getOrElse("THB")
  
  private def extractSku(doc: Document): Option[String] =
    List("[itemprop=sku]", "[class*=sku]", "[class*=product-id]")
      .flatMap(sel => doc.selectFirst(sel).map(_.text).filter(_.nonEmpty))
      .headOption
  
  private def extractImage(doc: Document): Option[String] =
    List(
      "meta[property=og:image]",
      "[itemprop=image]",
      ".product-image img",
      "#product-image img"
    ).flatMap: sel =>
      if sel.startsWith("meta") then
        doc.selectFirst(sel).flatMap(_.attr("content"))
      else
        doc.selectFirst(sel).flatMap(e => e.absUrl("src").orElse(e.attr("src")))
    .headOption
  
  private def extractRating(doc: Document): Option[Double] =
    List("[itemprop=ratingValue]", "[class*=rating]")
      .flatMap: sel =>
        doc.selectFirst(sel)
          .flatMap(e => e.attr("content").orElse(Some(e.text)))
          .flatMap(_.toDoubleOption)
      .headOption
  
  private def extractReviewCount(doc: Document): Option[Int] =
    List("[itemprop=reviewCount]", "[class*=review-count]")
      .flatMap: sel =>
        doc.selectFirst(sel)
          .flatMap(e => e.attr("content").orElse(Some(e.text)))
          .flatMap(s => s.replaceAll("[^0-9]", "").toIntOption)
      .headOption
  
  private def extractAvailability(doc: Document): String =
    doc.selectFirst("[itemprop=availability]")
      .flatMap(e => e.attr("content").orElse(Some(e.text)))
      .map: avail =>
        if avail.contains("InStock") || avail.contains("มีสินค้า") then "in_stock"
        else if avail.contains("OutOfStock") || avail.contains("สินค้าหมด") then "out_of_stock"
        else "unknown"
      .getOrElse("unknown")
  
  private def extractDescription(doc: Document): Option[String] =
    List("[itemprop=description]", ".product-description", "#description")
      .flatMap: sel =>
        doc.selectFirst(sel).map(_.text).filter(_.nonEmpty)
      .headOption
```

---

## CSS Selectors และ XPath

### CSS Selector Reference

```scala
// CSS Selector Examples ที่ใช้บ่อย

object CssSelectors:
  
  // Basic selectors
  val byTagName    = "div"              // ทุก <div>
  val byId         = "#main-content"    // element ที่มี id="main-content"
  val byClass      = ".article"         // elements ที่มี class="article"
  val byAttribute  = "[href]"           // elements ที่มี href attribute
  val byAttrValue  = "[type=submit]"    // elements ที่มี type="submit"
  
  // Combinators
  val descendant   = "div p"            // <p> ที่อยู่ใน <div>
  val child        = "ul > li"          // <li> ที่เป็น direct child ของ <ul>
  val adjacent     = "h2 + p"           // <p> ที่อยู่ถัดจาก <h2>
  val sibling      = "h2 ~ p"           // <p> ทุกตัวที่อยู่หลัง <h2>
  
  // Pseudo-classes
  val firstChild   = "li:first-child"   // <li> ตัวแรก
  val lastChild    = "li:last-child"    // <li> ตัวสุดท้าย
  val nthChild     = "tr:nth-child(2)"  // <tr> ตัวที่ 2
  val nthChildEven = "tr:nth-child(even)" // <tr> ตัวคู่
  val notSelector  = "p:not(.skip)"     // <p> ที่ไม่มี class="skip"
  
  // Attribute selectors
  val attrContains = "[class*=product]"  // class ที่มีคำว่า "product"
  val attrStarts   = "[href^=https]"     // href ที่ขึ้นต้นด้วย "https"
  val attrEnds     = "[src$=.png]"       // src ที่ลงท้ายด้วย ".png"
  
  // Jsoup-specific
  val containsText = ":contains(price)"  // elements ที่มีข้อความ "price"
  val hasClass     = ".parent:has(.child)" // elements ที่มี child ตาม selector

// Usage examples
def extractNavLinks(doc: Document): List[String] =
  doc.selectAll("nav a[href]")
    .flatMap(_.absUrl("href"))
    .filter(_.nonEmpty)
    .distinct

def extractTableData(doc: Document): List[List[String]] =
  doc.selectAll("table tr").map: row =>
    row.selectAll("td, th").map(_.text)

def extractStructuredData(doc: Document): Map[String, String] =
  doc.selectAll("dl dt").flatMap: dt =>
    dt.selectFirst("+ dd").map: dd =>
      dt.text -> dd.text
  .toMap
```

---

## Pagination และ Crawling

### Pagination Strategy

```scala
package com.example.scraper

import cats.effect.*
import cats.syntax.all.*

// Pagination types
sealed trait PaginationType
case class NumberedPages(baseUrl: String => String) extends PaginationType
case class NextLink(selector: String) extends PaginationType
case class InfiniteScroll(apiUrl: String => String) extends PaginationType
case class CursorBased(getNextCursor: String => Option[String]) extends PaginationType

class Paginator(fetcher: HttpFetcher, parser: HtmlParser):
  
  // ดึงทุก pages ด้วย numbered pagination
  def scrapeAllPages[A](
    firstPageUrl: String,
    maxPages: Int,
    extractItems: Document => List[A],
    getNextUrl: Document => Option[String]
  ): IO[List[A]] =
    
    def scrapeFrom(url: String, pageNum: Int, accumulated: List[A]): IO[List[A]] =
      if pageNum > maxPages then IO.pure(accumulated)
      else
        fetcher.fetchHtml(url).flatMap:
          case Left(err) =>
            IO.println(s"Error on page $pageNum: $err") >>
            IO.pure(accumulated)
          
          case Right(html) =>
            parser.parse(html, url) match
              case Left(err) =>
                IO.println(s"Parse error: $err") >>
                IO.pure(accumulated)
              
              case Right(doc) =>
                val items = extractItems(doc)
                val nextUrl = getNextUrl(doc)
                
                IO.println(s"Page $pageNum: found ${items.length} items") >>
                (nextUrl match
                  case Some(next) if items.nonEmpty =>
                    scrapeFrom(next, pageNum + 1, accumulated ++ items)
                  case _ =>
                    IO.pure(accumulated ++ items)
                )
    
    scrapeFrom(firstPageUrl, 1, List.empty)
  
  // ดึงด้วย URL pattern
  def scrapeByPattern[A](
    makeUrl: Int => String,
    maxPages: Int,
    extractItems: Document => List[A],
    stopCondition: List[A] => Boolean = _.isEmpty
  ): IO[List[A]] =
    
    def loop(pageNum: Int, accumulated: List[A]): IO[List[A]] =
      if pageNum > maxPages then IO.pure(accumulated)
      else
        val url = makeUrl(pageNum)
        fetcher.fetchHtml(url).flatMap:
          case Left(_) => IO.pure(accumulated)
          case Right(html) =>
            parser.parse(html, url) match
              case Left(_) => IO.pure(accumulated)
              case Right(doc) =>
                val items = extractItems(doc)
                if stopCondition(items) then IO.pure(accumulated)
                else loop(pageNum + 1, accumulated ++ items)
    
    loop(1, List.empty)

// Web Crawler
class WebCrawler(fetcher: HttpFetcher, parser: HtmlParser):
  
  def crawl(
    seedUrls: List[String],
    maxDepth: Int,
    maxPages: Int,
    urlFilter: String => Boolean = _ => true
  ): IO[Map[String, Document]] =
    
    def crawlBFS(
      queue: List[(String, Int)],
      visited: Set[String],
      results: Map[String, Document]
    ): IO[Map[String, Document]] =
      
      if queue.isEmpty || results.size >= maxPages then
        IO.pure(results)
      else
        val (url, depth) :: rest = queue
        
        if visited.contains(url) || !urlFilter(url) then
          crawlBFS(rest, visited, results)
        else
          fetcher.fetchHtml(url).flatMap:
            case Left(err) =>
              IO.println(s"Crawl error for $url: $err") >>
              crawlBFS(rest, visited + url, results)
            
            case Right(html) =>
              parser.parse(html, url) match
                case Left(_) =>
                  crawlBFS(rest, visited + url, results)
                
                case Right(doc) =>
                  val newLinks = if depth < maxDepth then
                    extractLinks(doc, url).filterNot(visited.contains)
                      .take(10) // Limit links per page
                      .map(_ -> (depth + 1))
                  else List.empty
                  
                  IO.println(s"Crawled: $url (depth=$depth, links=${newLinks.length})") >>
                  crawlBFS(
                    rest ++ newLinks,
                    visited + url,
                    results + (url -> doc)
                  )
    
    crawlBFS(seedUrls.map(_ -> 0), Set.empty, Map.empty)
  
  private def extractLinks(doc: Document, baseUrl: String): List[String] =
    doc.selectAll("a[href]")
      .flatMap(_.absUrl("href"))
      .filter: url =>
        url.nonEmpty &&
        (url.startsWith("http://") || url.startsWith("https://")) &&
        !url.contains("#")
      .distinct
```

---

## Rate Limiting และ Polite Crawling

### Rate Limiter

```scala
package com.example.scraper

import cats.effect.*
import cats.effect.std.{Semaphore, Queue}
import scala.concurrent.duration.*

class RateLimiter(
  requestsPerSecond: Double,
  semaphore: Semaphore[IO]
):
  
  private val delayBetweenRequests: Duration =
    (1000.0 / requestsPerSecond).milliseconds
  
  def throttle[A](action: IO[A]): IO[A] =
    semaphore.permit.surround:
      action <* IO.sleep(delayBetweenRequests)

object RateLimiter:
  
  def create(requestsPerSecond: Double, maxConcurrent: Int = 1): Resource[IO, RateLimiter] =
    Resource.eval:
      Semaphore[IO](maxConcurrent).map: sem =>
        new RateLimiter(requestsPerSecond, sem)

// Domain-specific rate limiter
class DomainRateLimiter:
  private val limiters: Ref[IO, Map[String, RateLimiter]] =
    Ref.unsafe(Map.empty)
  
  def getLimiter(domain: String, rps: Double = 1.0): IO[RateLimiter] =
    limiters.get.flatMap: current =>
      current.get(domain) match
        case Some(limiter) => IO.pure(limiter)
        case None =>
          Semaphore[IO](1).map: sem =>
            val limiter = new RateLimiter(rps, sem)
            limiter
          .flatTap: limiter =>
            limiters.update(_ + (domain -> limiter))
  
  def throttleRequest[A](url: String)(action: IO[A]): IO[A] =
    val domain = extractDomain(url)
    getLimiter(domain).flatMap(_.throttle(action))
  
  private def extractDomain(url: String): String =
    url.split("/")(2) // Simplified - ใช้ URI parser จริงๆ

// Polite scraper
class PoliteScraper(
  fetcher: HttpFetcher,
  rateLimiter: DomainRateLimiter,
  respectRobotsTxt: Boolean = true
):
  
  private val robotsCache: Ref[IO, Map[String, RobotsRules]] =
    Ref.unsafe(Map.empty)
  
  def fetch(url: String): IO[Either[FetchError, String]] =
    checkRobotsTxt(url).flatMap:
      case false =>
        IO.pure(Left(NetworkError("Disallowed by robots.txt", url)))
      case true =>
        rateLimiter.throttleRequest(url):
          fetcher.fetchHtml(url)
  
  private def checkRobotsTxt(url: String): IO[Boolean] =
    if !respectRobotsTxt then IO.pure(true)
    else
      val domain = extractDomain(url)
      val path = extractPath(url)
      
      getRobotsRules(domain).map: rules =>
        rules.isAllowed("ScalaBot", path)
  
  private def getRobotsRules(domain: String): IO[RobotsRules] =
    robotsCache.get.flatMap: cache =>
      cache.get(domain) match
        case Some(rules) => IO.pure(rules)
        case None =>
          fetchRobotsTxt(domain).flatMap: rules =>
            robotsCache.update(_ + (domain -> rules)) >> IO.pure(rules)
  
  private def fetchRobotsTxt(domain: String): IO[RobotsRules] =
    fetcher.fetchHtml(s"https://$domain/robots.txt")
      .map:
        case Right(content) => RobotsRules.parse(content)
        case Left(_)        => RobotsRules.allowAll
  
  private def extractDomain(url: String): String = url.split("/")(2)
  private def extractPath(url: String): String =
    url.split("/").drop(3).mkString("/", "/", "")

// Robots.txt parser (simplified)
case class RobotsRules(
  disallowedPaths: List[String],
  crawlDelay: Option[Int]
):
  def isAllowed(userAgent: String, path: String): Boolean =
    !disallowedPaths.exists(path.startsWith)

object RobotsRules:
  val allowAll = RobotsRules(List.empty, None)
  
  def parse(content: String): RobotsRules =
    val lines = content.split("\n").map(_.trim).filterNot(_.startsWith("#"))
    var disallowed = List.empty[String]
    var crawlDelay: Option[Int] = None
    var currentAgent = "*"
    
    lines.foreach: line =>
      if line.startsWith("User-agent:") then
        currentAgent = line.stripPrefix("User-agent:").trim
      else if line.startsWith("Disallow:") && (currentAgent == "*" || currentAgent.toLowerCase.contains("scala")) then
        val path = line.stripPrefix("Disallow:").trim
        if path.nonEmpty then disallowed = disallowed :+ path
      else if line.startsWith("Crawl-delay:") then
        line.stripPrefix("Crawl-delay:").trim.toIntOption.foreach: d =>
          crawlDelay = Some(d)
    
    RobotsRules(disallowed, crawlDelay)
```

---

## Data Extraction และ Transformation

### Data Transformation Pipeline

```scala
package com.example.scraper

import cats.effect.*
import cats.syntax.all.*
import io.circe.*
import io.circe.generic.auto.*
import io.circe.syntax.*
import java.io.{FileWriter, BufferedWriter}

// Scraped data models
case class ScrapedArticle(
  url: String,
  title: String,
  author: Option[String],
  publishDate: Option[String],
  content: String,
  tags: List[String],
  scrapedAt: Long = System.currentTimeMillis()
)

case class ScrapedProduct(
  url: String,
  name: String,
  price: Option[Double],
  currency: String,
  availability: String,
  description: Option[String],
  imageUrl: Option[String],
  scrapedAt: Long = System.currentTimeMillis()
)

// Data pipeline
class DataPipeline[A]:
  
  def filter(pred: A => Boolean): DataPipeline[A] = this
  def map[B](f: A => B): DataPipeline[B] = new DataPipeline[B]
  def flatMap[B](f: A => List[B]): DataPipeline[B] = new DataPipeline[B]

// Data exporter
class DataExporter:
  
  def toJson[A: Encoder](items: List[A], outputPath: String): IO[Unit] =
    IO:
      val json = items.asJson.spaces2
      val writer = new BufferedWriter(new FileWriter(outputPath))
      try
        writer.write(json)
      finally
        writer.close()
  
  def toCsv(items: List[Map[String, String]], outputPath: String): IO[Unit] =
    IO:
      if items.isEmpty then ()
      else
        val headers = items.head.keys.toList
        val writer = new BufferedWriter(new FileWriter(outputPath))
        try
          // Header row
          writer.write(headers.map(escapeCsv).mkString(","))
          writer.newLine()
          
          // Data rows
          items.foreach: item =>
            val row = headers.map(h => escapeCsv(item.getOrElse(h, "")))
            writer.write(row.mkString(","))
            writer.newLine()
        finally
          writer.close()
  
  private def escapeCsv(value: String): String =
    if value.contains(",") || value.contains("\"") || value.contains("\n") then
      "\"" + value.replace("\"", "\"\"") + "\""
    else value
  
  def toNdjson[A: Encoder](items: List[A], outputPath: String): IO[Unit] =
    IO:
      val writer = new BufferedWriter(new FileWriter(outputPath))
      try
        items.foreach: item =>
          writer.write(item.asJson.noSpaces)
          writer.newLine()
      finally
        writer.close()
```

---

## Complete Scraper Example

### News Site Scraper

```scala
package com.example.scraper

import cats.effect.*
import cats.syntax.all.*
import org.http4s.ember.client.*
import org.jsoup.nodes.Document
import io.circe.generic.auto.*
import scala.concurrent.duration.*

// =============================================================
// Complete News Scraper
// =============================================================

object NewsScraper extends IOApp.Simple:
  
  def run: IO[Unit] =
    HttpClientConfig.create().use: httpClient =>
      val polite   = HttpClientConfig.politeClient(httpClient)
      val fetcher  = new HttpFetcher(polite)
      val parser   = new HtmlParser()
      val extractor = new ArticleExtractor()
      val exporter = new DataExporter()
      
      val scraper = new ArticleScraper(fetcher, parser, extractor)
      
      for
        articles <- scraper.scrapeNewsSite(
          baseUrl   = "https://example-news.com",
          maxPages  = 5,
          delayMs   = 2000
        )
        
        _ <- IO.println(s"Scraped ${articles.length} articles")
        
        // Export data
        _ <- exporter.toJson(articles, "output/articles.json")
        _ <- exporter.toCsv(
          articles.map(articleToMap),
          "output/articles.csv"
        )
        
        _ <- IO.println("Scraping complete!")
      yield ()
  
  private def articleToMap(a: ScrapedArticle): Map[String, String] =
    Map(
      "url"         -> a.url,
      "title"       -> a.title,
      "author"      -> a.author.getOrElse(""),
      "publishDate" -> a.publishDate.getOrElse(""),
      "tags"        -> a.tags.mkString("|"),
      "scrapedAt"   -> a.scrapedAt.toString
    )

class ArticleScraper(
  fetcher: HttpFetcher,
  parser: HtmlParser,
  extractor: ArticleExtractor
):
  
  def scrapeNewsSite(
    baseUrl: String,
    maxPages: Int,
    delayMs: Long
  ): IO[List[ScrapedArticle]] =
    
    def scrapeListPage(pageNum: Int): IO[List[String]] =
      val url = s"$baseUrl/news?page=$pageNum"
      fetcher.fetchHtml(url).map:
        case Left(err) =>
          println(s"Error fetching list page $pageNum: $err")
          List.empty
        case Right(html) =>
          parser.parse(html, url) match
            case Left(_) => List.empty
            case Right(doc) =>
              doc.selectAll("article a.article-link, .news-list a[href]")
                .flatMap(_.absUrl("href"))
                .filter(_.nonEmpty)
                .distinct
    
    def scrapeArticle(url: String): IO[Option[ScrapedArticle]] =
      IO.sleep(delayMs.milliseconds) >>
      fetcher.fetchHtml(url).map:
        case Left(err) =>
          println(s"Error fetching article $url: $err")
          None
        case Right(html) =>
          parser.parse(html, url) match
            case Left(_) => None
            case Right(doc) =>
              extractor.extract(doc, url).map: article =>
                ScrapedArticle(
                  url         = url,
                  title       = article.title,
                  author      = article.author,
                  publishDate = article.publishDate,
                  content     = extractContent(doc),
                  tags        = article.tags
                )
    
    for
      // ดึง URLs จากหลาย pages
      allUrls <- (1 to maxPages).toList.flatTraverse(scrapeListPage)
      _ <- IO.println(s"Found ${allUrls.length} article URLs")
      
      // ดึง articles
      articles <- allUrls.traverse(scrapeArticle).map(_.flatten)
    yield articles
  
  private def extractContent(doc: Document): String =
    val contentSelectors = List(
      "article .content",
      ".article-body",
      ".post-content",
      "article p"
    )
    
    contentSelectors.flatMap: sel =>
      doc.selectAll(sel).map(_.text).filter(_.nonEmpty)
    .mkString("\n\n")

// =============================================================
// E-commerce Product Scraper
// =============================================================

class ProductScraper(
  fetcher: HttpFetcher,
  parser: HtmlParser,
  extractor: ProductExtractor
):
  
  def scrapeCategory(
    categoryUrl: String,
    maxPages: Int = 10
  ): IO[List[ScrapedProduct]] =
    
    def getProductUrls(doc: Document): List[String] =
      doc.selectAll(".product-card a, .product-item a[href], .product-list a")
        .flatMap(_.absUrl("href"))
        .filter(_.contains("/product/"))
        .distinct
    
    def getNextPageUrl(doc: Document): Option[String] =
      doc.selectFirst("a.next, a[rel=next], .pagination .next a")
        .flatMap(_.absUrl("href"))
    
    val paginator = new Paginator(fetcher, parser)
    
    paginator.scrapeAllPages(
      firstPageUrl = categoryUrl,
      maxPages     = maxPages,
      extractItems = getProductUrls,
      getNextUrl   = getNextPageUrl
    ).flatMap: allUrls =>
      allUrls.traverse: url =>
        IO.sleep(1.second) >>
        fetcher.fetchHtml(url).map:
          case Left(_) => None
          case Right(html) =>
            parser.parse(html, url).toOption.flatMap: doc =>
              extractor.extract(doc, url).map: p =>
                ScrapedProduct(
                  url          = url,
                  name         = p.name,
                  price        = p.price,
                  currency     = p.currency,
                  availability = p.availability,
                  description  = p.description,
                  imageUrl     = p.imageUrl
                )
      .map(_.flatten)

// =============================================================
// Main Application
// =============================================================

object ScraperApp extends IOApp:
  
  def run(args: List[String]): IO[ExitCode] =
    val target = args.headOption.getOrElse("articles")
    
    HttpClientConfig.create().use: client =>
      val fetcher   = new HttpFetcher(HttpClientConfig.politeClient(client))
      val parser    = new HtmlParser()
      val exporter  = new DataExporter()
      
      target match
        case "articles" =>
          val scraper = new ArticleScraper(fetcher, parser, new ArticleExtractor())
          for
            articles <- scraper.scrapeNewsSite(
              baseUrl  = sys.env.getOrElse("TARGET_URL", "https://example.com"),
              maxPages = 3,
              delayMs  = 2000
            )
            _ <- IO.println(s"Total articles: ${articles.length}")
            _ <- exporter.toJson(articles, "output/articles.json")
          yield ExitCode.Success
        
        case "products" =>
          val scraper = new ProductScraper(fetcher, parser, new ProductExtractor())
          for
            products <- scraper.scrapeCategory(
              categoryUrl = sys.env.getOrElse("CATEGORY_URL", "https://example.com/category"),
              maxPages    = 5
            )
            _ <- IO.println(s"Total products: ${products.length}")
            _ <- exporter.toJson(products, "output/products.json")
          yield ExitCode.Success
        
        case _ =>
          IO.println(s"Unknown target: $target. Use 'articles' or 'products'") >>
          IO.pure(ExitCode.Error)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **http4s Client**: การตั้งค่าและใช้งาน HTTP client สำหรับดึง web pages
2. **Jsoup**: HTML parsing ด้วย Java library ที่ทรงพลัง
3. **CSS Selectors**: การเลือก elements ด้วย selectors แบบต่างๆ
4. **Pagination**: จัดการ pagination แบบ numbered, next-link
5. **Web Crawling**: BFS crawler ที่ควบคุม depth และ pages
6. **Rate Limiting**: ป้องกันการ overload servers ด้วย rate limiting
7. **Polite Crawling**: เคารพ robots.txt และ crawl-delay
8. **Data Export**: บันทึกข้อมูลเป็น JSON, CSV, NDJSON
9. **Complete Scrapers**: ตัวอย่าง News และ E-commerce scrapers

### ข้อควรระวัง

- **Terms of Service**: ตรวจสอบ ToS ก่อนทำ scraping
- **robots.txt**: เคารพข้อจำกัดที่ระบุไว้
- **Rate Limiting**: ไม่ส่ง requests เร็วเกินไป
- **Legal Considerations**: บางประเทศมีกฎหมายเกี่ยวกับ web scraping

---

*[← ตอนที่ 87: Testing with Cats Effect](part-87-cats-effect-testing.md) | [ตอนที่ 89: Building CLI Tools →](part-89-cli-tools.md)*
