---
layout: default
title: Structural
parent: Design Patterns
---

# Design Patterns - Structural
---

<h3><ul>
    <li><a href="#Adapter">Adapter</a></li>
    <li><a href="#Proxy">Proxy</a></li>
</ul></h3>

---
## Adapter

__Adapter__ provides a structure that allows entities with incompatible interfaces to collaborate. 
The Model-View-ViewModel is one example that fits this design pattern.

__Example__
```swift
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
---
## Proxy

A __Proxy__ acts as a placeholder for the intended party, allowing additional logic to be applied before a request reaches the intended party.

__Example__
```swift
// The Server accepts a request from the client and returns with a response
protocol Server {
    associatedtype Response
    associatedtype Error : Swift.Error
    func request(url: URL) -> Result<Response, Error>
}

// A Proxy is a type of Server that validates a url and re-routes the request to other servers
protocol Proxy : Server {
    var servers : [AnyServer] { get }
    func isBlackedListed(url: URL) -> Bool
    func redirectRequest(url: URL, to server: AnyServer) -> Result<Response, Error>
}
extension Proxy where Response == AnyServer.Response, Error == AnyServer.Error {
    func redirectRequest(url: URL, to server: AnyServer) -> Result<AnyServer.Response, AnyServer.Error> {
        print("Server \(server.id) requesting ...")
        return server.request(url: url)
    }
}

// AnyServer is a type erased server capable of handling a client request
struct AnyServer : Server {
    typealias Response = Data
    typealias Error = Swift.Error
    let id : Int
    func request(url: URL) -> Result<Response, Error> {
        do {
            return .success(try Data(contentsOf: url))
        } catch {
            return .failure(error)
        }
    }
}

// ProxyServer is a concrete Proxy that manages multiple servers, 
// validates request urls, and redirects requests to servers it manages
struct ProxyServer : Proxy, Server {
    enum ProxyError : Error {
        case blacklisted
    }
    typealias Response = AnyServer.Response
    typealias Error = AnyServer.Error
    let servers : [AnyServer]
    
    func isBlackedListed(url: URL) -> Bool {
        url.absoluteString.hasSuffix(".org")
    }
    func request(url: URL) -> Result<Response, Error> {
        guard !isBlackedListed(url: url) else { return .failure(ProxyError.blacklisted) }
        return redirectRequest(url: url, to: servers[Int.random(in: 0..<servers.count)])
    }
}


// Use Case:
// Create a ProxyServer with multiple Servers
let proxyServer = ProxyServer(servers: (1...30).map(AnyServer.init))
var urlString = "http://www.google.com"
let domains = ["com", "org", "arpa"]

// Make requests through the ProxyServer
for i in 0...10 {
    print(proxyServer.request(url: URL(string: urlString)!))
    urlString = urlString.replacingOccurrences(of: domains[i%domains.count], with: domains[(i+1)%domains.count])
    sleep(2)
}

// Console Output:
// Server 7 requesting ...
// success(14358 bytes)
// failure(__lldb_expr_134.ProxyServer.ProxyError.blacklisted)
// Server 22 requesting ...
// failure(Error Domain=NSCocoaErrorDomain Code=256 "The file couldn’t be opened." UserInfo={NSURL=http://www.google.arpa})
// Server 15 requesting ...
// success(14361 bytes)
// failure(__lldb_expr_134.ProxyServer.ProxyError.blacklisted)
// Server 3 requesting ...
// failure(Error Domain=NSCocoaErrorDomain Code=256 "The file couldn’t be opened." UserInfo={NSURL=http://www.google.arpa})
// Server 23 requesting ...
// success(14361 bytes)
// failure(__lldb_expr_134.ProxyServer.ProxyError.blacklisted)
// Server 10 requesting ...
// failure(Error Domain=NSCocoaErrorDomain Code=256 "The file couldn’t be opened." UserInfo={NSURL=http://www.google.arpa})
// Server 10 requesting ...
// success(14358 bytes)
// failure(__lldb_expr_134.ProxyServer.ProxyError.blacklisted)
```
