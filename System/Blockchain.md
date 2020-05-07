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

```Swift
@dynamicMemberLookup
struct Block {
    struct Content : Encodable {
        let previous : String
        let timestamp : TimeInterval
        let data : Data
    }
    public let content : Content
    public let hash : String
    
    private init(
        previous: String,
        timestamp : TimeInterval,
        data: Data,
        hashfunction: (Content)->String) {
        
        content = Content(previous: previous, timestamp: timestamp, data: data)
        hash = hashfunction(content)
    }
    
    public init?<T: Encodable>(
        previous: String = "",
        timestamp : TimeInterval = Date().timeIntervalSince1970,
        data: T,
        hashfunction: (Content)->String = Crypto.sha256) {
        
        do {
            self.init(
                previous: previous,
                timestamp: timestamp,
                data: try JSONEncoder().encode(data),
                hashfunction: hashfunction
            )
        } catch {
            return nil
        }
    }
    
    public subscript<T>(dynamicMember keyPath: KeyPath<Content, T>) -> T {
        return content[keyPath: keyPath]
    }
}
```
