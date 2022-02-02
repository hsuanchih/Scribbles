# Proxy
---

A __Proxy__ acts as a placeholder for the intended party, allowing additional logic to be applied before a request reaches the intended party.

---
## Example

```Swift
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
