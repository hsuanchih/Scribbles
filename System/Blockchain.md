# Blockchain
---
## Motivation
A blockchain offers a means to maintain a decentralized, distributed digital ledger through validation, verification, consensus, and immutability. 
The fundamental building blocks of a blockchain is what its name is in the literal sense - a chain of blocks that contain data. 
This entry is an attempt to make a concrete case of the blockchain by breaking down & implement its logical components. 
To wrap things up, we'll piece the components together and have them work in tandem.

---
## Cryptographic Hash

The immutability nature of a blockchain is sustained through cryptographic hashes. Cryptographic hashes are on-way mathematical functions used to ensure data integrity (unless collisions occur). Blockchain uses hashing to ensure not only a single block in the chain, but the entire chain. The way a blockchain is able to do so is by including the hash value of the previous block in the hash computation of each block. Modifying the content of a block invalidates all the blocks that are chained after it. There are many variants of cryptographic hashes, here we'll stick with SHA-256.

```Swift
import CryptoKit

struct Crypto {

    // To make life easier, we constrain the input of the hash function 
    // to conform to Encodable. We don't have to make assumptions about
    // what the concrete data type we're hashing, as long as it can be encoded.
    static func sha256<Input: Encodable>(_ input: Input) -> String {
        SHA256.hash(data: try! JSONEncoder().encode(input))
            .makeIterator()
            .map { String(format: "%02x", $0) }
            .joined()
    }
}
```
---
## Block

A block in a blockchain is used to track some content. The content itself can be anything - it can be a transaction of digital currency, or just a entry of your personal diary. The content stored by the block and the hash value of its previous block are combined to compute its hash value.

```Swift
struct Block {
    
    // Body of a block includes:
    // * Hash value of the previous block
    // * Time at which the block is created
    // * The actual data stored by the block
    // * Hash value of the block
    public var previous : String = ""
    public let createdAt : TimeInterval = Date().timeIntervalSince1970
    public let data : Data
    public var hash : String = ""
    
    // Create a block to store some data, the data
    // to store can be of any type as long as it conforms to Encodable
    public init?<T: Encodable>(data: T) {
        do { self.data = try JSONEncoder().encode(data) }
        catch { return nil }
    }
    
    // Content used to compute the hash value of the current block
    public var content : String {
        "\(previous) \(createdAt) \(String(data: data, encoding: .utf16)!)"
    }
}
```
---
## Blockchain

Now let's create a blockchain to store some blocks. 

```Swift
struct Blockchain {
    
    // Chains all the blocks in the blockchain
    private var blocks : [Block] = []
    
    // Adds a new block to the blockchain
    // If the block to add to the chain is not the genesis block,
    // we want to link this block to the last block on the chain.
    // In any case, we'll compute the hash value of the block based on 
    // its content, and append the block to the chain
    public mutating func add(_ block: Block) {
        var block = block
        if let last = blocks.last {
            block.previous = last.hash
        }
        block.hash = Crypto.sha256(block.content)
        blocks.append(block)
    }
    
    // Validates whether all blocks in the chain are valid
    // There are 2 conditions that must be met:
    // * The current block's stored hash value should equal the
    //   computed hash value from its content
    // * The current block's hash value must equal its next block's
    //   previous value
    public func validate() -> Bool {
        for index in 0..<blocks.count-1 {
            let current = blocks[index], next = blocks[index+1]
            if current.hash != Crypto.sha256(current.content) || current.hash != next.previous {
                return false
            }
        }
        return true
    }
}
```
This blockchain implementation allows us to add new blocks to the chain & validate existing blocks. Things seem to fall nicely into place now, but there's a slight problem:

Recall from earlier that modifying the content of a block in the chain will invalidate all the block that are after it. A simple way to restore the validity of the chain after modifying a block is to re-compute the hash values of all the blocks follow the modified block. We want to prevent this from happening at all costs, but our current blockchain makes it too easy to add a new block to the chain. We want to increase the difficulty for adding a new block to the chain, and we can do so with something called a proof of work.

---
## Proof of Work
The intention of a proof of work is to slow down the process of adding a new block to the chain by introducing some difficulty. This is done by setting requirements that are computation heavy but rather easy to validate. One example of a proof of work is to require the computed hash value of a block to have a certain number of leading zeros for the block to be considered valid for chaining.

Previously we generate the hash value of our block based on the content of the block. If the content doesn't change, neither will its hash value. Needless to say, we'll need to introduce a new variable into our hash computation - a random value referred to as a __nonce__. Here's what our blockchain implementation looks like with proof of work.

```Swift
// Block (including nonce)
struct Block {
    
    // Body of a block includes:
    // * Hash value of the previous block
    // * Time at which the block is created
    // * The actual data stored by the block
    // * Hash value of the block
    public var previous : String = ""
    public let createdAt : TimeInterval = Date().timeIntervalSince1970
    public let data : Data
    public var hash : String = ""
    
    // We now add a nonce for computing Proof of Work
    public var nonce : Int = 0
    
    // Create a block to store some data, the data
    // to store can be of any type as long as it conforms to Encodable
    public init?<T: Encodable>(data: T) {
        do { self.data = try JSONEncoder().encode(data) }
        catch { return nil }
    }
    
    // Content used to compute the hash value of the current block,
    // now including the nonce for Proof of Work
    public var content : String {
        "\(previous) \(createdAt) \(String(data: data, encoding: .utf16)!) \(nonce)"
    }
}


// Blockchain (including Proof of Work)
struct Blockchain {
    
    // Chains all the blocks in the blockchain
    private var blocks : [Block] = []
    
    // We now add a difficulty level as part of Proof of Work
    public var difficulty : Int
    public init(difficulty: Int) {
        self.difficulty = difficulty
    }
    
    // Adds a new block to the blockchain
    // If the block to add to the chain is not the genesis block,
    // we want to link this block to the last block on the chain.
    // In any case, we'll compute the hash value of the block based on 
    // its content, and append the block to the chain
    public mutating func add(_ block: Block) {
        var block = block
        if let last = blocks.last {
            block.previous = last.hash
        }
        block.hash = proofOfWork(&block)
        blocks.append(block)
    }
    
    // Validates whether all blocks in the chain are valid
    // There are 2 conditions that must be met:
    // * The current block's stored hash value should equal the
    //   computed hash value from its content
    // * The current block's hash value must equal its next block's
    //   previous value
    public func validate() -> Bool {
        for index in 0..<blocks.count-1 {
            let current = blocks[index], next = blocks[index+1]
            if current.hash != Crypto.sha256(current.content) || current.hash != next.previous {
                return false
            }
        }
        return true
    }
    
    // Proof of Work computation
    // The difficulty level defines how many leading zeros must be in the
    // hash output of block's content.
    // We randomly pick a value as the nonce each time, and compute the
    // hash value until we arrive at one that meets the difficulty level
    private func proofOfWork(_ block: inout Block) -> String {
        let prefix = String(Array(repeating: "0", count: difficulty))
        var hash = Crypto.sha256(block.content)
        repeat {
            block.nonce = Int.random(in: 0...Int.max)
            hash = Crypto.sha256(block.content)
        } while !hash.hasPrefix(prefix)
        return hash
    }
}
```

---
## Transaction
Now let's put everything together using concrete transactions. To do so let's assume that our blockchain is used to track transactions of digital currency. Each transaction will consist of a sender and a receiver - both identified by their public key. A transaction will also include an amount that is to be transacted:

```Swift
typealias PublicKey = UUID
struct Transaction : Codable {
    let sender: PublicKey, receiver: PublicKey, amount: Double
}
```

---
## Reward System
Also, a peer who wins out on the proof of work competition will receive a certain amount of reward - this reward from the system to the peer will take form in a new transaction. Let's add that to our blockchain implementation:
```Swift
// Blockchain (including Pending Transactions, Proof of Work, and Rewards)
struct BlockChain {
    
    // Our blockchain now has a reward system:
    // The reward amount is 0.23 for every Proof of Work completed
    // assigned by the system
    // We also add a helper method to compute the balance of a user
    public static let publicKey = PublicKey()
    private var reward : Double = 0.23
    private mutating func reward(to peer: PublicKey) {
        accept(Transaction(sender: Self.publicKey, receiver: peer, amount: reward))
    }
    public func balance(_ peer: PublicKey) -> Double {
        blocks.reduce(0) {
            do {
                let transaction = try JSONDecoder().decode(Transaction.self, from: $1.data)
                if transaction.receiver == peer {
                    return $0+transaction.amount
                }
            } catch {}
            return $0
        }
    }
    
    // We also track pending transactions not yet added to the blockchain
    // Peers take pending transactions to construct a block, 
    // and complete the proof of work to add transactions to the blockchain
    private var transactions : [Transaction] = []
    public mutating func accept(_ transaction: Transaction) {
        transactions.append(transaction)
    }
    public var nextTransaction : Transaction? {
        mutating get {
            guard !transactions.isEmpty else { return nil }
            return transactions.removeFirst()
        }
    }
    
    // Chains all the blocks in the blockchain
    private var blocks : [Block] = []
    
    // We now add a difficulty level as part of Proof of Work
    public var difficulty : Int
    public init(difficulty: Int) {
        self.difficulty = difficulty
    }
    
    // Adds a new block to the blockchain
    // If the block to add to the chain is not the genesis block,
    // we want to link this block to the last block on the chain.
    // In any case, we'll compute the hash value of the block based on 
    // its content, and append the block to the chain
    public mutating func add(_ block: Block, by peer: PublicKey) {
        print(
            """
            
            ==========================================================
            \(peer) adds a new block to the chain
            """
        )
        var block = block
        if let last = blocks.last {
            block.previous = last.hash
        }
        block.hash = proofOfWork(&block)
        reward(to: peer)
        blocks.append(block)
    }
    
    // Proof of Work computation
    // The difficulty level defines how many leading zeros must be in the
    // hash output of block's content.
    // We randomly pick a value as the nonce each time, and compute the
    // hash value until we arrive at one that meets the difficulty level
    private func proofOfWork(_ block: inout Block) -> String {
        let prefix = String(Array(repeating: "0", count: difficulty)),
        startTime = Date().timeIntervalSince1970
        
        print("Computing Proof of Work...")
        
        var hash = Crypto.sha256(block.content)
        repeat {
            block.nonce = Int.random(in: 0...Int.max)
            hash = Crypto.sha256(block.content)
        } while !hash.hasPrefix(prefix)
        
        print(
            """
            Computation complete: \(hash)
            Time used: \((Date().timeIntervalSince1970-startTime)) seconds
            
            """
        )
        
        return hash
    }
    
    // Validates whether all blocks in the chain are valid
    // There are 2 conditions that must be met:
    // * The current block's stored hash value should equal the
    //   computed hash value from its content
    // * The current block's hash value must equal its next block's
    //   previous value
    public func validate() -> Bool {
        for index in 0..<blocks.count-1 {
            let current = blocks[index], next = blocks[index+1]
            if current.hash != Crypto.sha256(current.content) || current.hash != next.previous {
                return false
            }
        }
        return true
    }
}
```

---
## Simulation
Finally here's a simulation of the blockchain we built:
```Swift
// We start the simulation with a blockchain of difficulty 3
var blockchain = BlockChain(difficulty: 3)

// And seed the blockchain with a transaction from alice to bob
let alice = PublicKey(), bob = PublicKey()
blockchain.accept(
    Transaction(
        sender: alice,
        receiver: bob,
        amount: Double.random(in: 0...20)
    )
)

// Then I take over and do the heavy lifting, adding each pending transaction
// to the blockchain
let me = PublicKey()
while let transaction = blockchain.nextTransaction {
    guard let block = Block(data: transaction) else { continue }
    blockchain.add(block, by: me)
    print("Balance for \(me) is: \(blockchain.balance(me))")
}
```
With some sample console output:
```
==========================================================
994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 adds a new block to the chain
Computing Proof of Work...
Computation complete: 000d668dd9e38b893c366a3c1938be8a96526bbe04bff26f96a2e4aa5f49848b
Time used: 4.692615985870361 seconds

Balance for 994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 is: 0.0

==========================================================
994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 adds a new block to the chain
Computing Proof of Work...
Computation complete: 0005d07b09fe88aed40456a7656c982be8022a015e48667f0f1def6ea8c75b77
Time used: 18.09759211540222 seconds

Balance for 994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 is: 0.23

==========================================================
994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 adds a new block to the chain
Computing Proof of Work...
Computation complete: 000be6742b87c17871532ea4c8ecfd8bea0c1560888dbdce774c24d0f13f94a5
Time used: 0.4398808479309082 seconds

Balance for 994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 is: 0.46

==========================================================
994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 adds a new block to the chain
Computing Proof of Work...
Computation complete: 0001948fc549b450ed6580e6d63097220bc7f1466450b1a56a55694555fca35f
Time used: 27.193089962005615 seconds

Balance for 994E0CFB-D8CD-4ABD-ADF7-C1EFE60302E2 is: 0.6900000000000001
```

---
## Wrapping Up
So we've built an over-simplified flavor of a blockchain with the intent to understand its components. This implementation is still far from the real thing, as we've limited the implementation to a simplified client-side view. Remember the blockchain is a distributed ledger backed by a peer-to-peer network, and involves many facets like digital signature, message transport, synchronization, and consensus - none of which we've addressed here. Nevertheless, we're a step further in demystifying how blockchains work. 
