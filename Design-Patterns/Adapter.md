# Adapter
---

__Adapter__ provides a structure that allows entities with incompatible interfaces to collaborate. 
The Model-View-ViewModel is one example that fits this design pattern.

---
## Example

```Swift
// We want to use an adapter to transform the PersonInfo structure input into a 
// prettified JSON string output
// Here's what the PersonInfo data structure looks like
struct PersonInfo : Encodable {
    struct Name : Codable {
        let firstname : String
        let lastname : String
    }
    let name : Name
    let birthday : Date
}

// The downstream DataProvider will output a string regardless of the format
// of its input
// The upstream DataProvider will take in a PersonInfo object, and encode it
// into Data format
protocol DataProvider {
    associatedtype Input
    associatedtype Output
    var input : Input { get }
    var output : Output { get }
}
extension DataProvider where Input : Encodable, Output == Data {
    var output : Output {
        try! JSONEncoder().encode(input)
    }
}

// The Adapter takes the output format from the upstream, and
// converts into the output format accepted by the downstream's
// client
protocol Adapter {
    associatedtype UpStream
    associatedtype DownStream
    func convert(_ upstream: UpStream) -> DownStream
}
extension Adapter where UpStream == Data, DownStream == String {
    func convert(_ upstream: UpStream) -> DownStream {
        String(data: upstream, encoding: .utf8)!
    }
}

// PersonInfoProvider is a concrete upstream DataProvider that accepts
// PersonInfo data structure as input, and outputs JSON Data
struct PersonInfoProvider : DataProvider {
    typealias Input = PersonInfo
    typealias Output = Data
    let input : Input
}

// JSONStringProvider takes PersonInfoProvider as its adaptee
// and converts the output from PersonInfoProvider into prettified JSON String
struct JSONStringProvider : DataProvider, Adapter {
    
    typealias Input = PersonInfoProvider.Output
    typealias Output = String
    typealias UpStream = Input
    typealias DownStream = Output
    
    let adaptee : PersonInfoProvider
    var input : Input { adaptee.output }
    var output: Output { convert(input) }
}


// Use Case:
// Create a PersonInfo object
let personInfo = PersonInfo(name: PersonInfo.Name(firstname: "John", lastname: "Doe"), birthday: Date())

// Setup an adaptee to take in the PersonInfo object
let adaptee = PersonInfoProvider(input: personInfo)

// Setup the target and bind it with the adaptee
let target = JSONStringProvider(adaptee: adaptee)

// Print the target's output
print(target.output)

// Console Output:
// {"name":{"firstname":"John","lastname":"Doe"},"birthday":609445568.17656398}
```
