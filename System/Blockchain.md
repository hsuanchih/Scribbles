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
struct BlockChain {
    
    // Chains all the blocks in the blockchain
    private var blocks : [Block] = []
    
    // Adds a new block to the blockchain
    public mutating func add(_ block: Block) {
        var block = block
        
        // If the block to add to the chain is not the genesis block
        // we want to link this block to the last block on the chain
        if let last = blocks.last {
            block.previous = last.hash
        }
        
        // Then compute the hash value of the block based on its content
        block.hash = Crypto.sha256(block.content)
        
        // And append the block to the chain
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

```
