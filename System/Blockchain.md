# Blockchain
---
## Motivation
A blockchain offers a means to maintain a decentralized, distributed digital ledger through validation, verification, consensus, and immutability. 
The fundamental building blocks of a blockchain is what its name is in the literal sense - a chain of blocks that contain data. 
This entry is an attempt to make a concrete case of the blockchain by breaking down & implement its logical components. 
To wrap things up, we'll piece the components together and have them work in tandem.

---
## Block
```Swift
import Foundation
import CryptoKit

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
