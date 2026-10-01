# Part 49: Real-Time Chat Application

## สารบัญ
1. [Architecture](#architecture)
2. [WebSocket Server](#websocket-server)
3. [Message Routing](#message-routing)
4. [Presence and Status](#presence)
5. [Persistence](#persistence)
6. [Complete Example](#complete-example)

---

## Architecture

### Chat System Design

```
Real-Time Chat Architecture:

Client (Browser/Mobile)
       │
       │ WebSocket
       ▼
┌─────────────────┐
│   http4s Server │ ← WebSocket handler
│  (per instance) │
└────────┬────────┘
         │
    Pub/Sub via Redis
         │
┌────────▼────────┐     ┌──────────────────┐
│ Message Router  │────▶│ Message History  │
│ (Redis PubSub)  │     │  (PostgreSQL)    │
└────────────────┘     └──────────────────┘

Message Flow:
1. Client connects via WebSocket
2. Server subscribes to user's channels
3. Client sends message
4. Server validates and stores
5. Server publishes to Redis channel
6. All instances receive and forward to connected clients
```

---

## WebSocket Server

### http4s WebSocket Handler

```scala
import cats.effect.{IO, Ref}
import org.http4s.server.websocket.WebSocketBuilder2
import org.http4s.websocket.WebSocketFrame
import fs2.{Stream, Pipe}
import io.circe.generic.auto.*
import io.circe.syntax.*
import io.circe.parser.*

// Messages
sealed trait ClientMessage
case class SendMessage(roomId: String, content: String) extends ClientMessage
case class JoinRoom(roomId: String)                     extends ClientMessage
case class LeaveRoom(roomId: String)                    extends ClientMessage

sealed trait ServerMessage
case class NewMessage(roomId: String, from: String, content: String, timestamp: Long) extends ServerMessage
case class UserJoined(roomId: String, userId: String)  extends ServerMessage
case class UserLeft(roomId: String, userId: String)    extends ServerMessage
case class Error(message: String)                      extends ServerMessage

// Connected client state
case class Client(
  userId: String,
  username: String,
  rooms: Set[String]
)

// WebSocket handler
class ChatWebSocketHandler(
  chatService: ChatService,
  ws: WebSocketBuilder2[IO]
):
  def handle(userId: String, username: String): IO[org.http4s.Response[IO]] =
    for
      clientRef <- Ref.of[IO, Client](Client(userId, username, Set.empty))
      (outQueue, outStream) <- fs2.concurrent.Queue.unbounded[IO, ServerMessage].map { q =>
        (q, Stream.fromQueueUnterminated(q))
      }
      _ <- chatService.registerClient(userId, outQueue)
      send: Pipe[IO, WebSocketFrame, Unit] = stream =>
        stream
          .evalMap {
            case WebSocketFrame.Text(text, _) =>
              decode[ClientMessage](text) match
                case Right(msg) => handleClientMessage(userId, clientRef, outQueue, msg)
                case Left(err)  => outQueue.offer(Error(s"Invalid message: $err"))
            case WebSocketFrame.Close(_) =>
              cleanup(userId, clientRef, chatService)
            case _ => IO.unit
          }
      receive: Stream[IO, WebSocketFrame] = outStream.map { msg =>
        WebSocketFrame.Text(msg.asJson.noSpaces)
      }
      response <- ws.build(receive, send)
    yield response

  private def handleClientMessage(
    userId: String,
    clientRef: Ref[IO, Client],
    queue: fs2.concurrent.Queue[IO, ServerMessage],
    msg: ClientMessage
  ): IO[Unit] = msg match
    case JoinRoom(roomId) =>
      for
        _ <- chatService.joinRoom(userId, roomId)
        _ <- clientRef.update(c => c.copy(rooms = c.rooms + roomId))
        _ <- chatService.broadcastToRoom(roomId, UserJoined(roomId, userId))
      yield ()

    case LeaveRoom(roomId) =>
      for
        _ <- chatService.leaveRoom(userId, roomId)
        _ <- clientRef.update(c => c.copy(rooms = c.rooms - roomId))
        _ <- chatService.broadcastToRoom(roomId, UserLeft(roomId, userId))
      yield ()

    case SendMessage(roomId, content) =>
      for
        client <- clientRef.get
        _ <- if client.rooms.contains(roomId) then
          chatService.sendMessage(userId, roomId, content)
        else
          queue.offer(Error(s"Not in room: $roomId"))
      yield ()

  private def cleanup(
    userId: String,
    clientRef: Ref[IO, Client],
    svc: ChatService
  ): IO[Unit] =
    for
      client <- clientRef.get
      _      <- client.rooms.toList.traverse_(roomId =>
        svc.broadcastToRoom(roomId, UserLeft(roomId, userId))
      )
      _      <- svc.unregisterClient(userId)
    yield ()
```

---

## Message Routing

### Chat Service

```scala
import cats.effect.{IO, Ref}
import fs2.concurrent.Queue

class ChatService:
  // Connected clients: userId -> outgoing queue
  private val clients = Ref.unsafe[IO, Map[String, Queue[IO, ServerMessage]]](Map.empty)

  // Room membership: roomId -> Set[userId]
  private val rooms = Ref.unsafe[IO, Map[String, Set[String]]](Map.empty)

  def registerClient(userId: String, queue: Queue[IO, ServerMessage]): IO[Unit] =
    clients.update(_ + (userId -> queue))

  def unregisterClient(userId: String): IO[Unit] =
    for
      _     <- clients.update(_ - userId)
      // Remove from all rooms
      rms   <- rooms.get
      toLeave = rms.collect { case (rid, users) if users.contains(userId) => rid }
      _     <- toLeave.toList.traverse_(rid => rooms.update(m =>
        m.get(rid).fold(m)(u => m + (rid -> (u - userId)))
      ))
    yield ()

  def joinRoom(userId: String, roomId: String): IO[Unit] =
    rooms.update { m =>
      val members = m.getOrElse(roomId, Set.empty)
      m + (roomId -> (members + userId))
    }

  def leaveRoom(userId: String, roomId: String): IO[Unit] =
    rooms.update { m =>
      m.get(roomId).fold(m) { members =>
        m + (roomId -> (members - userId))
      }
    }

  def sendMessage(fromUserId: String, roomId: String, content: String): IO[Unit] =
    val msg = NewMessage(roomId, fromUserId, content, System.currentTimeMillis())
    broadcastToRoom(roomId, msg)

  def broadcastToRoom(roomId: String, msg: ServerMessage): IO[Unit] =
    for
      rms <- rooms.get
      cls <- clients.get
      members = rms.getOrElse(roomId, Set.empty)
      _ <- members.toList.traverse_ { userId =>
        cls.get(userId).fold(IO.unit)(_.offer(msg))
      }
    yield ()

  def sendToUser(userId: String, msg: ServerMessage): IO[Unit] =
    clients.get.flatMap { cls =>
      cls.get(userId).fold(IO.unit)(_.offer(msg))
    }

  def getRoomMembers(roomId: String): IO[Set[String]] =
    rooms.get.map(_.getOrElse(roomId, Set.empty))

  def getConnectedUsers: IO[List[String]] =
    clients.get.map(_.keys.toList)
```

---

## Presence and Status

### Online/Offline Tracking

```scala
import dev.profunktor.redis4cats.RedisCommands
import cats.effect.IO
import scala.concurrent.duration.*

enum UserStatus:
  case Online, Away, Busy, Offline

case class UserPresence(userId: String, status: UserStatus, lastSeen: Long)

class PresenceService(redis: RedisCommands[IO, String, String]):
  private def presenceKey(userId: String) = s"presence:$userId"
  private val HeartbeatTTL = 30.seconds

  def setOnline(userId: String): IO[Unit] =
    redis.setEx(presenceKey(userId), "online", HeartbeatTTL).void

  def setStatus(userId: String, status: UserStatus): IO[Unit] =
    redis.setEx(presenceKey(userId), status.toString.toLowerCase, HeartbeatTTL).void

  def heartbeat(userId: String): IO[Unit] =
    redis.expire(presenceKey(userId), HeartbeatTTL).void

  def setOffline(userId: String): IO[Unit] =
    redis.del(presenceKey(userId)).void

  def getStatus(userId: String): IO[UserStatus] =
    redis.get(presenceKey(userId)).map {
      case Some("online") => UserStatus.Online
      case Some("away")   => UserStatus.Away
      case Some("busy")   => UserStatus.Busy
      case _              => UserStatus.Offline
    }

  def getOnlineUsers(userIds: List[String]): IO[List[String]] =
    userIds.filterM { userId =>
      getStatus(userId).map(_ != UserStatus.Offline)
    }
```

---

## Persistence

### Message History

```scala
import doobie.*
import doobie.implicits.*
import cats.effect.IO
import java.time.Instant

case class Message(
  id: Long,
  roomId: String,
  fromUserId: String,
  content: String,
  sentAt: Instant
)

class MessageRepository(xa: Transactor[IO]):
  def save(roomId: String, fromUserId: String, content: String): IO[Message] =
    sql"""
      INSERT INTO messages (room_id, from_user_id, content, sent_at)
      VALUES ($roomId, $fromUserId, $content, NOW())
      RETURNING id, room_id, from_user_id, content, sent_at
    """.query[Message].unique.transact(xa)

  def getHistory(roomId: String, limit: Int = 50, before: Option[Long] = None): IO[List[Message]] =
    val base = fr"SELECT id, room_id, from_user_id, content, sent_at FROM messages WHERE room_id = $roomId"
    val cursor = before.map(id => fr"AND id < $id").getOrElse(fr"")
    val q = base ++ cursor ++ fr"ORDER BY id DESC LIMIT $limit"
    q.query[Message].to[List].transact(xa).map(_.reverse)

  def searchMessages(roomId: String, query: String): IO[List[Message]] =
    sql"""
      SELECT id, room_id, from_user_id, content, sent_at
      FROM messages
      WHERE room_id = $roomId
        AND to_tsvector('english', content) @@ plainto_tsquery($query)
      ORDER BY sent_at DESC
      LIMIT 100
    """.query[Message].to[List].transact(xa)
```

---

## Complete Example

### Full Chat Server

```scala
import cats.effect.{IO, IOApp}
import org.http4s.ember.server.EmberServerBuilder
import org.http4s.server.Router
import org.http4s.HttpRoutes
import com.comcast.ip4s.*

object ChatServer extends IOApp.Simple:
  def run: IO[Unit] =
    val chatService = new ChatService()
    val presenceService: IO[PresenceService] = IO.stub  // from Redis
    val msgRepo: IO[MessageRepository] = IO.stub        // from DB

    EmberServerBuilder.default[IO]
      .withHost(ipv4"0.0.0.0")
      .withPort(port"8080")
      .withHttpWebSocketApp { wsb =>
        val handler = ChatWebSocketHandler(chatService, wsb)

        Router(
          // WebSocket endpoint
          "/ws/chat" -> HttpRoutes.of[IO] {
            case req @ GET -> Root =>
              // Extract userId from JWT token
              req.headers.get(ci"Authorization").flatMap { _ =>
                // Normally: verify JWT and get userId
                Some(("user-123", "Alice"))
              } match
                case Some((userId, username)) =>
                  handler.handle(userId, username)
                case None =>
                  IO.pure(org.http4s.Response[IO](org.http4s.Status.Unauthorized))
          },

          // REST endpoints
          "/api/chat" -> HttpRoutes.of[IO] {
            case GET -> Root / "rooms" / roomId / "history" =>
              IO.stub  // return message history

            case GET -> Root / "users" / userId / "status" =>
              IO.stub  // return presence status
          }
        ).orNotFound
      }
      .build
      .useForever

  // Heartbeat to keep presence alive
  def startHeartbeatJob(presenceService: PresenceService, chatService: ChatService): IO[Unit] =
    fs2.Stream.awakeEvery[IO](15.seconds)
      .evalMap { _ =>
        chatService.getConnectedUsers
          .flatMap(users => users.traverse_(presenceService.heartbeat))
      }
      .compile.drain
```

---

## สรุป

ในส่วนนี้คุณได้เรียนรู้:
- ✅ Real-time chat architecture
- ✅ WebSocket handler กับ http4s
- ✅ Message routing: broadcast to rooms
- ✅ User presence: online/offline tracking ด้วย Redis
- ✅ Message persistence: history and search
- ✅ Complete chat server กับ heartbeat

---

*[← Part 48: E-Commerce](part-48-ecommerce.md) | [Part 50: Data Pipeline →](part-50-data-pipeline.md)*
