# Part 38: GraphQL กับ Caliban

## สารบัญ
1. [GraphQL Overview](#graphql-overview)
2. [Caliban Setup](#caliban-setup)
3. [Schema Definition](#schema-definition)
4. [Resolvers](#resolvers)
5. [Subscriptions](#subscriptions)
6. [Complete Example](#complete-example)

---

## GraphQL Overview

### แนวคิด GraphQL

```
GraphQL vs REST:
- Client requests exactly what it needs
- Single endpoint for all operations
- Strongly typed schema
- Real-time with subscriptions

Operations:
- Query:        read data
- Mutation:     modify data  
- Subscription: real-time updates

Type System:
- Scalar: Int, Float, String, Boolean, ID
- Object: { field: Type }
- List: [Type]
- Non-null: Type!
- Enum, Interface, Union, Input
```

### Dependencies

```scala
libraryDependencies ++= Seq(
  "com.github.ghostdogpr" %% "caliban"            % "2.5.1",
  "com.github.ghostdogpr" %% "caliban-http4s"     % "2.5.1",
  "com.github.ghostdogpr" %% "caliban-cats"       % "2.5.1",
  "com.github.ghostdogpr" %% "caliban-federation" % "2.5.1"  // for federated GraphQL
)
```

---

## Caliban Setup

### Basic Setup

```scala
import caliban.*
import caliban.schema.*
import caliban.schema.Schema.auto.*
import cats.effect.IO
import zio.*

// Caliban uses ZIO by default, but can be used with Cats Effect
// using caliban-cats

// Data models (auto-derived schema)
case class Episode(id: Int, title: String, airDate: String)
case class Character(
  id: Int,
  name: String,
  homePlanet: Option[String],
  episodes: List[Episode]
)

// Arguments (input types)
case class CharacterArgs(name: String)
case class EpisodeArgs(title: Option[String])

// Query type
case class Queries(
  characters: ZIO[Any, Nothing, List[Character]],
  character: CharacterArgs => ZIO[Any, Nothing, Option[Character]]
)
```

---

## Schema Definition

### Complete Schema

```scala
import caliban.*
import caliban.schema.*
import caliban.schema.Annotations.*
import zio.*

// GraphQL schema annotations
@GQLDescription("Represents a Star Wars character")
case class Character(
  @GQLDescription("Unique identifier")
  id: Int,
  @GQLDescription("Full name of the character")
  name: String,
  @GQLDescription("Home planet")
  homePlanet: Option[String],
  @GQLDeprecated("Use episodeIds instead")
  episodes: List[String],
  episodeIds: List[Int]
)

@GQLDescription("A Star Wars episode")
case class Episode(
  id: Int,
  title: String,
  releaseYear: Int
)

enum Affiliation:
  case LightSide, DarkSide, Neutral

case class SearchArgs(
  name: Option[String],
  affiliation: Option[Affiliation],
  limit: Int = 10
)

case class CreateCharacterInput(
  name: String,
  homePlanet: Option[String]
)

// Queries
case class Queries(
  character: Int => ZIO[Any, Throwable, Option[Character]],
  characters: SearchArgs => ZIO[Any, Throwable, List[Character]],
  episode: Int => ZIO[Any, Throwable, Option[Episode]],
  episodes: ZIO[Any, Throwable, List[Episode]]
)

// Mutations
case class Mutations(
  createCharacter: CreateCharacterInput => ZIO[Any, Throwable, Character],
  deleteCharacter: Int => ZIO[Any, Throwable, Boolean]
)
```

---

## Resolvers

### Implementing Resolvers

```scala
import caliban.*
import caliban.schema.*
import zio.*
import zio.stm.*

// In-memory data store
object CharacterStore:
  private val characters = Ref.make(List(
    Character(1, "Luke Skywalker", Some("Tatooine"), List("A New Hope"), List(4)),
    Character(2, "Darth Vader", Some("Tatooine"), List("A New Hope"), List(4)),
    Character(3, "Yoda", None, List("The Empire Strikes Back"), List(5))
  ))

  def getAll: ZIO[Any, Nothing, List[Character]] =
    characters.get

  def getById(id: Int): ZIO[Any, Nothing, Option[Character]] =
    characters.get.map(_.find(_.id == id))

  def search(name: Option[String], limit: Int): ZIO[Any, Nothing, List[Character]] =
    characters.get.map { chars =>
      val filtered = name.fold(chars)(n => chars.filter(_.name.toLowerCase.contains(n.toLowerCase)))
      filtered.take(limit)
    }

  def create(name: String, homePlanet: Option[String]): ZIO[Any, Nothing, Character] =
    characters.modify { chars =>
      val id = chars.map(_.id).maxOption.getOrElse(0) + 1
      val c = Character(id, name, homePlanet, Nil, Nil)
      (c, chars :+ c)
    }

  def delete(id: Int): ZIO[Any, Nothing, Boolean] =
    characters.modify { chars =>
      val exists = chars.exists(_.id == id)
      (exists, if exists then chars.filterNot(_.id == id) else chars)
    }

// Build schema
val api = graphQL(
  RootResolver(
    Queries(
      character = id => CharacterStore.getById(id),
      characters = args => CharacterStore.search(args.name, args.limit),
      episode = id => ZIO.succeed(None),  // TODO
      episodes = ZIO.succeed(Nil)         // TODO
    ),
    Mutations(
      createCharacter = input => CharacterStore.create(input.name, input.homePlanet),
      deleteCharacter = id => CharacterStore.delete(id)
    )
  )
)
```

---

## Subscriptions

### Real-Time Updates

```scala
import caliban.*
import caliban.schema.*
import zio.*
import zio.stream.*

// Event type
case class CharacterEvent(
  eventType: String,  // "created", "deleted", "updated"
  character: Character
)

// Hub for broadcasting events
val eventHub = ZIO.serviceWithZIO[Hub[CharacterEvent]](hub => ZIO.succeed(hub))

case class Subscriptions(
  characterEvents: ZStream[Hub[CharacterEvent], Nothing, CharacterEvent],
  characterCreated: ZStream[Hub[CharacterEvent], Nothing, Character]
)

// Subscription resolvers
val subscriptions = Subscriptions(
  characterEvents = ZStream.fromHub(hub),
  characterCreated = ZStream.fromHub(hub)
    .filter(_.eventType == "created")
    .map(_.character)
)

// Full API with subscriptions
val fullApi = graphQL(
  RootResolver(
    queries = Queries(/* ... */),
    mutations = Mutations(/* ... */),
    subscriptions = subscriptions
  )
)
```

---

## Complete Example

### http4s Integration

```scala
import caliban.*
import caliban.interop.cats.CatsInterop
import caliban.http4s.Http4sAdapter
import cats.effect.{IO, IOApp}
import org.http4s.ember.server.EmberServerBuilder
import org.http4s.server.Router
import com.comcast.ip4s.*
import zio.*
import zio.interop.catz.*

object GraphQLApp extends IOApp.Simple:

  // Sample data
  case class Book(id: Int, title: String, author: String, year: Int)
  case class BookArgs(id: Int)
  case class SearchArgs(query: Option[String], limit: Int = 10)
  case class AddBookInput(title: String, author: String, year: Int)

  var books = List(
    Book(1, "Programming in Scala", "Odersky", 2021),
    Book(2, "Functional Programming in Scala", "Chiusano", 2014),
    Book(3, "Scala with Cats", "Gurnell", 2020)
  )
  var nextId = 4

  case class Queries(
    book: BookArgs => Task[Option[Book]],
    books: SearchArgs => Task[List[Book]],
    bookCount: Task[Int]
  )

  case class Mutations(
    addBook: AddBookInput => Task[Book],
    removeBook: BookArgs => Task[Boolean]
  )

  val queries = Queries(
    book = args => ZIO.succeed(books.find(_.id == args.id)),
    books = args =>
      ZIO.succeed {
        args.query match
          case Some(q) => books.filter(b =>
            b.title.toLowerCase.contains(q.toLowerCase) ||
            b.author.toLowerCase.contains(q.toLowerCase)
          ).take(args.limit)
          case None => books.take(args.limit)
      },
    bookCount = ZIO.succeed(books.size)
  )

  val mutations = Mutations(
    addBook = input =>
      ZIO.succeed {
        val book = Book(nextId, input.title, input.author, input.year)
        books = books :+ book
        nextId += 1
        book
      },
    removeBook = args =>
      ZIO.succeed {
        val exists = books.exists(_.id == args.id)
        books = books.filterNot(_.id == args.id)
        exists
      }
  )

  val api = graphQL(RootResolver(queries, mutations))

  def run: IO[Unit] =
    val runtime = Runtime.default
    CatsInterop.toEffect(api.interpreter)(using runtime).flatMap { interpreter =>
      val graphqlRoutes = Http4sAdapter.makeHttpRoutes(interpreter)
      EmberServerBuilder.default[IO]
        .withHost(ipv4"0.0.0.0")
        .withPort(port"8080")
        .withHttpApp(Router(
          "/api/graphql"          -> graphqlRoutes,
          "/api/graphql/graphiql" -> Http4sAdapter.makeGraphiQLRoutes
        ).orNotFound)
        .build
        .useForever
    }
```

### GraphQL Client Queries

```graphql
# Get a book
query GetBook {
  book(id: 1) {
    id
    title
    author
    year
  }
}

# Search books
query SearchBooks {
  books(query: "Scala", limit: 5) {
    id
    title
    author
  }
}

# Add a book
mutation AddBook {
  addBook(input: {
    title: "ZIO in Action"
    author: "Adam Fraser"
    year: 2023
  }) {
    id
    title
  }
}

# Book count
query Count {
  bookCount
}
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ GraphQL concepts: queries, mutations, subscriptions
- ✅ Caliban schema definition ด้วย case classes
- ✅ Schema annotations: @GQLDescription, @GQLDeprecated
- ✅ Implementing resolvers กับ ZIO
- ✅ Real-time subscriptions กับ ZStream/Hub
- ✅ http4s integration กับ GraphiQL

---

*[← Part 37: Kafka](part-37-kafka.md) | [Part 39: gRPC →](part-39-grpc.md)*
