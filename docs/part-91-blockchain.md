# ส่วนที่ 91: Blockchain Concepts กับ Scala

## สารบัญ

1. [Blockchain Data Structure](#blockchain-data-structure)
2. [Hash Functions และ Merkle Trees](#hash-functions-และ-merkle-trees)
3. [Consensus Algorithms](#consensus-algorithms)
4. [Smart Contract Basics](#smart-contract-basics)
5. [Simple Blockchain Implementation](#simple-blockchain-implementation)
6. [Cryptocurrency Concepts](#cryptocurrency-concepts)
7. [Complete Blockchain Example](#complete-blockchain-example)
8. [สรุป](#สรุป)

---

## Blockchain Data Structure

Blockchain คือ linked list ของ blocks ที่เชื่อมโยงกันด้วย cryptographic hashes แต่ละ block มี hash ของ block ก่อนหน้า ทำให้ไม่สามารถแก้ไขข้อมูลย้อนหลังได้

### Block Structure

```scala
import java.time.Instant
import java.security.MessageDigest

case class BlockHeader(
  version: Int,
  previousHash: String,
  merkleRoot: String,
  timestamp: Instant,
  difficulty: Int,
  nonce: Long
):
  def hash: String =
    val input = s"$version$previousHash$merkleRoot${timestamp.toEpochMilli}$difficulty$nonce"
    SHA256.hash(input)

case class Transaction(
  id: String,
  sender: String,
  recipient: String,
  amount: BigDecimal,
  timestamp: Instant,
  signature: Option[String] = None
):
  def hash: String =
    SHA256.hash(s"$id$sender$recipient$amount${timestamp.toEpochMilli}")

case class Block(
  header: BlockHeader,
  transactions: List[Transaction],
  height: Long
):
  def hash: String = header.hash

  def isValid: Boolean =
    val merkle = MerkleTree.buildRoot(transactions.map(_.hash))
    merkle == header.merkleRoot

object Block:
  val genesisBlock: Block =
    val genesisTx = Transaction(
      id = "genesis",
      sender = "0" * 64,
      recipient = "genesis_address",
      amount = 50,
      timestamp = Instant.parse("2009-01-03T18:15:05Z")
    )
    val header = BlockHeader(
      version = 1,
      previousHash = "0" * 64,
      merkleRoot = MerkleTree.buildRoot(List(genesisTx.hash)),
      timestamp = Instant.parse("2009-01-03T18:15:05Z"),
      difficulty = 4,
      nonce = 0L
    )
    Block(header, List(genesisTx), 0L)
```

### Blockchain Chain

```scala
import scala.annotation.tailrec

class Blockchain(private val chain: Vector[Block] = Vector(Block.genesisBlock)):

  def latestBlock: Block = chain.last

  def addBlock(transactions: List[Transaction], difficulty: Int): Blockchain =
    val previousHash = latestBlock.hash
    val merkleRoot   = MerkleTree.buildRoot(transactions.map(_.hash))
    val mined        = mine(
      BlockHeader(
        version      = 1,
        previousHash = previousHash,
        merkleRoot   = merkleRoot,
        timestamp    = Instant.now(),
        difficulty   = difficulty,
        nonce        = 0L
      ),
      difficulty
    )
    val newBlock = Block(mined, transactions, chain.length.toLong)
    new Blockchain(chain :+ newBlock)

  @tailrec
  private def mine(header: BlockHeader, difficulty: Int): BlockHeader =
    val prefix = "0" * difficulty
    if header.hash.startsWith(prefix) then header
    else mine(header.copy(nonce = header.nonce + 1), difficulty)

  def isValid: Boolean =
    chain.sliding(2).forall:
      case Seq(prev, curr) =>
        curr.header.previousHash == prev.hash &&
        curr.header.hash.startsWith("0" * curr.header.difficulty) &&
        curr.isValid
      case _ => true

  def getBalance(address: String): BigDecimal =
    chain.flatMap(_.transactions).foldLeft(BigDecimal(0)):
      case (bal, tx) if tx.recipient == address => bal + tx.amount
      case (bal, tx) if tx.sender   == address  => bal - tx.amount
      case (bal, _)                              => bal

  def length: Int = chain.length
  def blocks: Vector[Block] = chain
```

---

## Hash Functions และ Merkle Trees

### SHA256 Implementation

```scala
import java.security.MessageDigest
import java.nio.charset.StandardCharsets

object SHA256:
  def hash(input: String): String =
    val digest = MessageDigest.getInstance("SHA-256")
    val bytes  = digest.digest(input.getBytes(StandardCharsets.UTF_8))
    bytes.map("%02x".format(_)).mkString

  def hashBytes(bytes: Array[Byte]): Array[Byte] =
    val digest = MessageDigest.getInstance("SHA-256")
    digest.digest(bytes)

  def doubleHash(input: String): String =
    hash(hash(input))

// Test
@main def testHash(): Unit =
  println(SHA256.hash("Hello, Blockchain!"))
  // e.g. 3b4c...
  println(SHA256.doubleHash("Bitcoin"))
```

### Merkle Tree

Merkle Tree เป็น binary tree ที่ใช้คำนวณ root hash ของชุด transactions ช่วยให้ verify ข้อมูลได้อย่างมีประสิทธิภาพ

```scala
sealed trait MerkleNode:
  def hash: String

case class MerkleLeaf(data: String) extends MerkleNode:
  val hash: String = SHA256.hash(data)

case class MerkleInternal(left: MerkleNode, right: MerkleNode) extends MerkleNode:
  val hash: String = SHA256.hash(left.hash + right.hash)

object MerkleTree:
  def build(data: List[String]): Option[MerkleNode] =
    if data.isEmpty then None
    else Some(buildLevel(data.map(MerkleLeaf.apply)))

  @tailrec
  private def buildLevel(nodes: List[MerkleNode]): MerkleNode =
    if nodes.length == 1 then nodes.head
    else
      val pairs = nodes
        .grouped(2)
        .map:
          case List(l, r) => MerkleInternal(l, r)
          case List(l)    => MerkleInternal(l, l) // duplicate last if odd
          case _          => throw new RuntimeException("Unexpected case")
        .toList
      buildLevel(pairs)

  def buildRoot(data: List[String]): String =
    build(data).map(_.hash).getOrElse("0" * 64)

  // Merkle Proof - verify a transaction without downloading the whole block
  def generateProof(data: List[String], target: String): List[(String, Boolean)] =
    def go(nodes: List[MerkleNode], targetHash: String): List[(String, Boolean)] =
      if nodes.length == 1 then List.empty
      else
        val pairs = nodes.grouped(2).toList
        pairs.zipWithIndex.flatMap:
          case (List(l, r), _) if l.hash == targetHash =>
            (r.hash, true) :: go(nodes.grouped(2).map:
              case List(a, b) => MerkleInternal(a, b)
              case List(a)    => MerkleInternal(a, a)
              case _          => throw new RuntimeException("Unexpected")
            .toList, MerkleInternal(l, r).hash)
          case (List(l, r), _) if r.hash == targetHash =>
            (l.hash, false) :: go(nodes.grouped(2).map:
              case List(a, b) => MerkleInternal(a, b)
              case List(a)    => MerkleInternal(a, a)
              case _          => throw new RuntimeException("Unexpected")
            .toList, MerkleInternal(l, r).hash)
          case _ => List.empty

    val leaves     = data.map(MerkleLeaf.apply)
    val targetHash = SHA256.hash(target)
    go(leaves, targetHash)

  def verifyProof(txHash: String, proof: List[(String, Boolean)], root: String): Boolean =
    val computedRoot = proof.foldLeft(txHash):
      case (current, (hash, isRight)) =>
        if isRight then SHA256.hash(current + hash)
        else SHA256.hash(hash + current)
    computedRoot == root
```

---

## Consensus Algorithms

### Proof of Work

```scala
case class ProofOfWork(difficulty: Int):
  private val prefix = "0" * difficulty

  def mine(data: String): (Long, String) =
    @tailrec
    def loop(nonce: Long): (Long, String) =
      val hash = SHA256.hash(data + nonce.toString)
      if hash.startsWith(prefix) then (nonce, hash)
      else loop(nonce + 1)
    loop(0L)

  def verify(data: String, nonce: Long): Boolean =
    SHA256.hash(data + nonce.toString).startsWith(prefix)

@main def powDemo(): Unit =
  val pow    = ProofOfWork(difficulty = 4)
  val data   = "block data: Alice -> Bob: 10 BTC"
  val start  = System.currentTimeMillis()
  val (nonce, hash) = pow.mine(data)
  val elapsed = System.currentTimeMillis() - start
  println(s"Nonce: $nonce")
  println(s"Hash:  $hash")
  println(s"Time:  ${elapsed}ms")
  println(s"Valid: ${pow.verify(data, nonce)}")
```

### Proof of Stake (simplified)

```scala
import scala.util.Random

case class Validator(address: String, stake: BigDecimal)

object ProofOfStake:
  def selectValidator(validators: List[Validator], seed: Long): Option[Validator] =
    if validators.isEmpty then None
    else
      val totalStake = validators.map(_.stake).sum
      val rng        = new Random(seed)
      val target     = BigDecimal(rng.nextDouble()) * totalStake
      @tailrec
      def pick(remaining: List[Validator], accumulated: BigDecimal): Option[Validator] =
        remaining match
          case Nil    => None
          case v :: t =>
            val next = accumulated + v.stake
            if next >= target then Some(v)
            else pick(t, next)
      pick(validators, BigDecimal(0))

  def slashValidator(validator: Validator, penaltyPct: BigDecimal): Validator =
    validator.copy(stake = validator.stake * (1 - penaltyPct / 100))
```

### Byzantine Fault Tolerance (PBFT simplified)

```scala
enum PBFTPhase:
  case PrePrepare, Prepare, Commit

case class PBFTMessage(
  phase: PBFTPhase,
  viewNumber: Int,
  sequenceNumber: Int,
  digest: String,
  nodeId: Int
)

class PBFTNode(val id: Int, val totalNodes: Int):
  private val faultTolerance = (totalNodes - 1) / 3 // f nodes can fail
  private val quorum         = 2 * faultTolerance + 1

  def canReach2fPlus1Quorum(votes: Int): Boolean =
    votes >= quorum

  def processPrePrepare(msg: PBFTMessage): Boolean =
    msg.phase == PBFTPhase.PrePrepare && msg.viewNumber >= 0

  def processPrepare(messages: List[PBFTMessage]): Boolean =
    val prepares = messages.count(m => m.phase == PBFTPhase.Prepare && m.nodeId != id)
    canReach2fPlus1Quorum(prepares)

  def processCommit(messages: List[PBFTMessage]): Boolean =
    val commits = messages.count(_.phase == PBFTPhase.Commit)
    canReach2fPlus1Quorum(commits)
```

---

## Smart Contract Basics

### Simple Smart Contract DSL

```scala
// Smart Contract state และ operations
case class ContractState(
  owner: String,
  balances: Map[String, BigDecimal],
  totalSupply: BigDecimal
)

// Contract operations
sealed trait ContractOp
case class Transfer(from: String, to: String, amount: BigDecimal) extends ContractOp
case class Mint(to: String, amount: BigDecimal) extends ContractOp
case class Burn(from: String, amount: BigDecimal) extends ContractOp

// Contract execution
object ERC20Contract:
  def execute(state: ContractState, op: ContractOp, caller: String): Either[String, ContractState] =
    op match
      case Transfer(from, to, amount) =>
        if from != caller then Left("Unauthorized: caller is not sender")
        else
          val fromBal = state.balances.getOrElse(from, BigDecimal(0))
          if fromBal < amount then Left(s"Insufficient balance: have $fromBal, need $amount")
          else
            Right(state.copy(
              balances = state.balances
                .updated(from, fromBal - amount)
                .updated(to, state.balances.getOrElse(to, BigDecimal(0)) + amount)
            ))

      case Mint(to, amount) =>
        if caller != state.owner then Left("Only owner can mint")
        else
          Right(state.copy(
            balances    = state.balances.updated(to, state.balances.getOrElse(to, BigDecimal(0)) + amount),
            totalSupply = state.totalSupply + amount
          ))

      case Burn(from, amount) =>
        val bal = state.balances.getOrElse(from, BigDecimal(0))
        if from != caller then Left("Unauthorized")
        else if bal < amount then Left("Insufficient balance to burn")
        else
          Right(state.copy(
            balances    = state.balances.updated(from, bal - amount),
            totalSupply = state.totalSupply - amount
          ))

// Multi-signature contract
case class MultiSigWallet(
  owners: Set[String],
  requiredSignatures: Int,
  pendingTxs: Map[Int, PendingTx]
)

case class PendingTx(to: String, amount: BigDecimal, approvals: Set[String], executed: Boolean)

object MultiSigContract:
  def submitTransaction(
    wallet: MultiSigWallet,
    caller: String,
    to: String,
    amount: BigDecimal,
    txId: Int
  ): Either[String, MultiSigWallet] =
    if !wallet.owners.contains(caller) then Left("Not an owner")
    else
      val tx = PendingTx(to, amount, Set(caller), executed = false)
      Right(wallet.copy(pendingTxs = wallet.pendingTxs + (txId -> tx)))

  def approveTransaction(
    wallet: MultiSigWallet,
    caller: String,
    txId: Int
  ): Either[String, (MultiSigWallet, Boolean)] =
    if !wallet.owners.contains(caller) then Left("Not an owner")
    else
      wallet.pendingTxs.get(txId) match
        case None     => Left("Transaction not found")
        case Some(tx) =>
          if tx.executed then Left("Already executed")
          else
            val updated  = tx.copy(approvals = tx.approvals + caller)
            val executed = updated.approvals.size >= wallet.requiredSignatures
            val newWallet = wallet.copy(
              pendingTxs = wallet.pendingTxs + (txId -> updated.copy(executed = executed))
            )
            Right((newWallet, executed))
```

---

## Simple Blockchain Implementation

### Complete Mini Blockchain

```scala
import java.time.Instant
import scala.collection.mutable

// Wallet
case class Wallet(privateKey: String, publicKey: String, address: String)

object WalletGenerator:
  def generate(): Wallet =
    val privateKey = SHA256.hash(java.util.UUID.randomUUID().toString)
    val publicKey  = SHA256.hash(privateKey)
    val address    = "0x" + SHA256.hash(publicKey).take(40)
    Wallet(privateKey, publicKey, address)

// Transaction Pool (mempool)
class TransactionPool:
  private val pending = mutable.ListBuffer.empty[Transaction]

  def add(tx: Transaction): Unit =
    if tx.amount > 0 then pending += tx

  def getAndClear(maxCount: Int): List[Transaction] =
    val txs = pending.take(maxCount).toList
    pending.dropInPlace(maxCount)
    txs

  def size: Int = pending.size

// Full Node
class FullNode(difficulty: Int = 4, blockReward: BigDecimal = 50):
  private var blockchain  = new Blockchain()
  private val pool        = new TransactionPool()
  private var peers       = List.empty[FullNode]

  def connectPeer(node: FullNode): Unit =
    peers = node :: peers

  def broadcastTransaction(tx: Transaction): Unit =
    pool.add(tx)
    peers.foreach(_.receiveTransaction(tx))

  def receiveTransaction(tx: Transaction): Unit =
    pool.add(tx)

  def mineBlock(minerAddress: String): Block =
    val reward = Transaction(
      id        = SHA256.hash(s"reward-${Instant.now()}"),
      sender    = "network",
      recipient = minerAddress,
      amount    = blockReward,
      timestamp = Instant.now()
    )
    val txs   = reward :: pool.getAndClear(9) // max 10 tx per block
    val newBC = blockchain.addBlock(txs, difficulty)
    blockchain = newBC
    val newBlock = blockchain.latestBlock
    peers.foreach(_.receiveBlock(newBlock))
    newBlock

  def receiveBlock(block: Block): Unit =
    // Simple validation: accept if it extends our chain
    if block.header.previousHash == blockchain.latestBlock.hash then
      val newBC = new Blockchain(blockchain.blocks :+ block)
      blockchain = newBC

  def getBalance(address: String): BigDecimal =
    blockchain.getBalance(address)

  def chainLength: Int = blockchain.length

  def isChainValid: Boolean = blockchain.isValid
```

### Demo

```scala
@main def blockchainDemo(): Unit =
  val node1 = FullNode(difficulty = 3)
  val node2 = FullNode(difficulty = 3)
  node1.connectPeer(node2)

  val alice  = WalletGenerator.generate()
  val bob    = WalletGenerator.generate()
  val miner  = WalletGenerator.generate()

  // Mine genesis reward for alice
  node1.mineBlock(alice.address)
  println(s"Alice balance after mining: ${node1.getBalance(alice.address)}")

  // Create transactions
  val tx1 = Transaction(
    id        = SHA256.hash("tx1"),
    sender    = alice.address,
    recipient = bob.address,
    amount    = BigDecimal(10),
    timestamp = Instant.now()
  )
  node1.broadcastTransaction(tx1)

  // Mine another block
  node1.mineBlock(miner.address)
  println(s"Alice balance: ${node1.getBalance(alice.address)}")
  println(s"Bob balance:   ${node1.getBalance(bob.address)}")
  println(s"Miner balance: ${node1.getBalance(miner.address)}")
  println(s"Chain valid:   ${node1.isChainValid}")
  println(s"Chain length:  ${node1.chainLength}")
```

---

## Cryptocurrency Concepts

### UTXO Model (Unspent Transaction Output)

```scala
case class UTXO(txId: String, outputIndex: Int, amount: BigDecimal, owner: String)

case class TxInput(utxoTxId: String, utxoIndex: Int, signature: String)
case class TxOutput(amount: BigDecimal, owner: String)

case class BitcoinLikeTx(
  id: String,
  inputs: List[TxInput],
  outputs: List[TxOutput],
  timestamp: Instant
)

class UTXOSet(private var utxos: Map[(String, Int), UTXO] = Map.empty):

  def addUTXO(utxo: UTXO): Unit =
    utxos = utxos + ((utxo.txId, utxo.outputIndex) -> utxo)

  def spendUTXO(txId: String, outputIndex: Int): Option[UTXO] =
    val key = (txId, outputIndex)
    val utxo = utxos.get(key)
    utxos = utxos - key
    utxo

  def getUTXOsFor(address: String): List[UTXO] =
    utxos.values.filter(_.owner == address).toList

  def getBalance(address: String): BigDecimal =
    getUTXOsFor(address).map(_.amount).sum

  def processTransaction(tx: BitcoinLikeTx): Either[String, Unit] =
    // Verify all inputs exist
    val inputUtxos = tx.inputs.traverse: input =>
      utxos.get((input.utxoTxId, input.utxoIndex))
        .toRight(s"UTXO ${input.utxoTxId}:${input.utxoIndex} not found")

    inputUtxos.map: ins =>
      val inputTotal  = ins.map(_.amount).sum
      val outputTotal = tx.outputs.map(_.amount).sum
      if inputTotal < outputTotal then
        Left(s"Insufficient inputs: $inputTotal < $outputTotal")
      else
        // Remove spent UTXOs
        tx.inputs.foreach(i => spendUTXO(i.utxoTxId, i.utxoIndex))
        // Add new UTXOs
        tx.outputs.zipWithIndex.foreach: (out, idx) =>
          addUTXO(UTXO(tx.id, idx, out.amount, out.owner))
        Right(())
    .joinRight

// Scala trick for flattening Either[String, Either[String, Unit]]
extension [A, B](e: Either[A, Either[A, B]])
  def joinRight: Either[A, B] = e.flatMap(identity)
```

### Token Standard

```scala
// ERC-20 like Token
trait Token[F[_]]:
  def name: String
  def symbol: String
  def decimals: Int
  def totalSupply: F[BigDecimal]
  def balanceOf(address: String): F[BigDecimal]
  def transfer(from: String, to: String, amount: BigDecimal): F[Either[String, Unit]]
  def approve(owner: String, spender: String, amount: BigDecimal): F[Unit]
  def transferFrom(spender: String, from: String, to: String, amount: BigDecimal): F[Either[String, Unit]]
  def allowance(owner: String, spender: String): F[BigDecimal]

// ERC-721 like NFT
case class NFT(
  tokenId: Long,
  owner: String,
  metadata: NFTMetadata
)

case class NFTMetadata(
  name: String,
  description: String,
  imageUrl: String,
  attributes: Map[String, String]
)

trait NFTContract[F[_]]:
  def mint(to: String, metadata: NFTMetadata): F[Long]
  def transfer(from: String, to: String, tokenId: Long): F[Either[String, Unit]]
  def ownerOf(tokenId: Long): F[Option[String]]
  def tokensOf(address: String): F[List[NFT]]
  def burn(tokenId: Long): F[Either[String, Unit]]
```

---

## Complete Blockchain Example

### Full Application

```scala
import cats.effect.*
import cats.effect.std.{Console, Queue}
import fs2.Stream

// Event-driven Blockchain Node
sealed trait NodeEvent
case class NewTransaction(tx: Transaction) extends NodeEvent
case class NewBlock(block: Block) extends NodeEvent
case class MineRequest(minerAddress: String) extends NodeEvent

class BlockchainNode(
  difficulty: Int,
  blockReward: BigDecimal,
  eventQueue: Queue[IO, NodeEvent]
) extends IOApp:

  private val blockchainRef   = IO.ref(new Blockchain()).unsafeRunSync()
  private val txPoolRef       = IO.ref(List.empty[Transaction]).unsafeRunSync()

  def submitTransaction(tx: Transaction): IO[Unit] =
    eventQueue.offer(NewTransaction(tx))

  def requestMine(minerAddress: String): IO[Unit] =
    eventQueue.offer(MineRequest(minerAddress))

  def getChainInfo: IO[String] =
    for
      bc  <- blockchainRef.get
      txs <- txPoolRef.get
    yield
      s"""
         |Chain length:  ${bc.length}
         |Chain valid:   ${bc.isValid}
         |Pending txs:   ${txs.length}
         |Latest hash:   ${bc.latestBlock.hash.take(16)}...
       """.stripMargin

  def eventLoop: IO[Unit] =
    Stream
      .fromQueueUnterminated(eventQueue)
      .evalMap:
        case NewTransaction(tx) =>
          txPoolRef.update(tx :: _) *>
          Console[IO].println(s"[TX] Added ${tx.id.take(8)}... to mempool")

        case NewBlock(block) =>
          blockchainRef.update: bc =>
            if block.header.previousHash == bc.latestBlock.hash then
              new Blockchain(bc.blocks :+ block)
            else bc
          *> Console[IO].println(s"[BLOCK] Received block at height ${block.height}")

        case MineRequest(minerAddress) =>
          for
            bc  <- blockchainRef.get
            txs <- txPoolRef.get
            reward = Transaction(
              id        = SHA256.hash(s"reward-${Instant.now()}-$minerAddress"),
              sender    = "network",
              recipient = minerAddress,
              amount    = blockReward,
              timestamp = Instant.now()
            )
            allTxs = reward :: txs.take(9)
            newBC  = bc.addBlock(allTxs, difficulty)
            _   <- blockchainRef.set(newBC)
            _   <- txPoolRef.update(_.drop(9))
            _   <- Console[IO].println(s"[MINE] Mined block at height ${newBC.latestBlock.height}")
          yield ()
      .compile
      .drain

  def run(args: List[String]): IO[ExitCode] =
    for
      _    <- Console[IO].println("Starting blockchain node...")
      node <- Queue.unbounded[IO, NodeEvent].map: q =>
                new BlockchainNode(3, 50, q)
      alice = WalletGenerator.generate()
      bob   = WalletGenerator.generate()
      _    <- node.requestMine(alice.address)
      _    <- node.submitTransaction(
                Transaction(SHA256.hash("tx1"), alice.address, bob.address, 10, Instant.now())
              )
      _    <- node.requestMine(alice.address)
      info <- node.getChainInfo
      _    <- Console[IO].println(info)
    yield ExitCode.Success
```

### Blockchain Explorer API

```scala
import org.http4s.*
import org.http4s.dsl.io.*
import org.http4s.circe.*
import io.circe.generic.auto.*

object BlockchainExplorer:
  def routes(blockchain: Blockchain): HttpRoutes[IO] =
    HttpRoutes.of[IO]:
      case GET -> Root / "blocks" =>
        Ok(blockchain.blocks.map: block =>
          Map(
            "height"    -> block.height.toString,
            "hash"      -> block.hash,
            "prevHash"  -> block.header.previousHash,
            "txCount"   -> block.transactions.length.toString,
            "timestamp" -> block.header.timestamp.toString
          )
        )

      case GET -> Root / "blocks" / LongVar(height) =>
        blockchain.blocks.find(_.height == height) match
          case Some(block) => Ok(block)
          case None        => NotFound()

      case GET -> Root / "transactions" / txId =>
        val found = blockchain.blocks.flatMap(_.transactions).find(_.id == txId)
        found match
          case Some(tx) => Ok(tx)
          case None     => NotFound()

      case GET -> Root / "address" / address / "balance" =>
        Ok(Map("address" -> address, "balance" -> blockchain.getBalance(address).toString))

      case GET -> Root / "address" / address / "transactions" =>
        val txs = blockchain.blocks.flatMap(_.transactions)
          .filter(tx => tx.sender == address || tx.recipient == address)
        Ok(txs)

      case GET -> Root / "stats" =>
        Ok(Map(
          "height"           -> blockchain.length.toString,
          "totalTransactions" -> blockchain.blocks.flatMap(_.transactions).length.toString,
          "isValid"          -> blockchain.isValid.toString
        ))
```

---

## สรุป

Blockchain ใน Scala มีแนวคิดสำคัญ:

- **Data Structure**: Linked blocks ที่เชื่อมด้วย cryptographic hashes
- **Merkle Tree**: โครงสร้างข้อมูลที่ช่วย verify transactions อย่างมีประสิทธิภาพ
- **Consensus**: Proof of Work, Proof of Stake สำหรับตกลงกันระหว่าง nodes
- **Smart Contracts**: โปรแกรมที่รันบน blockchain ด้วย deterministic execution
- **UTXO**: Bitcoin model สำหรับ track ยอดเงิน
- **Functional Style**: ใช้ immutable data และ pure functions ทำให้ verify correctness ง่ายขึ้น

---

*[← ส่วนที่ 90: Functional Application Architecture](part-90-fp-architecture.md) | [ส่วนที่ 92: Advanced Database Operations →](part-92-datastore.md)*
