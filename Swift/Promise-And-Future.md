# Promise & Future
---
The motivation for this chapter is an attempt to clean up code smells in asynchronous call chaining. Imagine the scenario whereby we want to send multiple network requests, but not one after another - we want to send a request only after a response is received from the previous request. How would we chain all these requests together? 

The intuitive solution is to start a request in the callback of the previous request, and soon enough we see that pattern of closures within closures within closures - a callback hell. There's certainly more than one way to cleanup the mess, but here we'll be looking at using promise & future to solve this problem.

So what's the difference between a __Promise__ and a __Future__? Behavior wise they are the same. Conceptually, however, a __Promise__ is a token of commitment to honor an agreement in the __Future__, despite not being able to do so at the time of making the promise. And that is how promises & futures relate to each other. 

We've probably seen promises/futures in action from the past, but in this chapter, we're going to see how they work under the hood. Let's start with a simple promise and build on it.

```Swift
// The most primitive form of a Promise
// The agreement of this promise is to deliver a Value when the promise can be fulfilled
class Promise<Value> {

    // A callback stored within a promise
    // This callback is called some time in the future when the promise can be fulfilled
    private var callback : ((Value) -> Void)?
    
    // The resolve method is called by the promise in the future to fulfill the promise
    func resolve(_ value: Value) -> Void {
        callback?(value)
    }
    
    // The then method can be called on the promise at anytime to designate the behavior
    // that should happen at the time when the the promise is fulfilled
    func then(onResolved: @escaping (Value) -> Void) {
        callback = onResolved
    }
}
```

Now that we have a simple promise that claims to deliver in the future, let's see how we can use it in a practical situation:

```Swift
// This decorator extends the legacy dataTask imeplemenation and returns a promise.
// The promise commits to deliver a Result<Data, Error> when the network responds to our request
// some time in the future. And it does so by calling the resolve method when the network responds. 
extension URLSession {
    func resumeDataTask(with urlRequest: URLRequest) -> Promise<Result<Data, Error>> {
        var urlRequest = urlRequest
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        let promise = Promise<Result<Data, Error>>()
        dataTask(with: urlRequest) { (data, response, error) in
            guard let data = data else {
                if let error = error {
                    promise.resolve(.failure(error))
                }
                return
            }
            promise.resolve(.success(data))
        }.resume()
        return promise
    }
}

// In our example, we'll be making requests to numbersapi.com, NumberModel below defines
// its response data model 
struct NumberModel: Decodable {
    let text : String
    let number : Double
    let type : String
}

// Send our request, and use JSONDecoder to decode the response
// Recall that the decorator resumeDataTask returns a Promise from which we can
// ask to perform certain actions when the promise is fulfilled.
// This is excatly what we've done here.
URLSession.shared.resumeDataTask(with: URLRequest(url: URL(string: "http://numbersapi.com/7")!))
    .then { result in
        switch result {
        case .success(let data):
            do { print(try JSONDecoder().decode(NumberModel.self, from: data))
            } catch { print(error) }
        case .failure(let error):
            print(error)
        }
    }

// Console Output:
// NumberModel(text: "7 is the figurative number of seas.", number: 7.0, type: "trivia")
```

So far our promise implementation allows us to store a callback upfront to be invoked some time in the future. But there's an immediate issue we need to address - an issue made obvious if the promise resolves before we get an opportunity to call `then` on the promise. Take this example where a promise resolves synchronously:

```Swift
let promise = Promise<Int>()

// The promise resolves before a callback is assigned
promise.resolve(3)

// The value 3 never gets printed
promise.then { value in print(value) }
```

A valid argument might be that promises are not meant to be used this way. After all, promises are supposed to be fulfilled some time in the future - we have other programming paradigms to handle synchronous propagation of values much more naive than promises. Nevertheless, we should not be making assumptions about temporal relationships between promise resolution and callback assignment. The way to fix it is to give a promise state:

```Swift
// The most primitive form of a Promise
// The agreement of this promise is to deliver a Value when the promise can be fulfilled
class Promise<Value> {

    // A callback stored within a promise
    // This callback is called some time in the future when the promise can be fulfilled
    private var callback : ((Value) -> Void)?
    
    // This promise now has state:
    // A promise is initially pending, but will be resolved as soon as the promise
    // is fulfilled
    private var state : State<Value> = .pending
    enum State<Value> {
        case pending
        case resolved(Value)
    }
    
    // The resolve method is called by the promise in the future to fulfill the promise
    func resolve(_ value: Value) -> Void {
        // Only resolve & fulfill the promise if it is still pending 
        guard case .pending = state else { return }
        state = .resolved(value)
        callback?(value)
    }
    
    // The then method can be called on the promise at anytime to designate the behavior
    // that should happen at the time when the the promise is fulfilled
    func then(onResolved: @escaping (Value) -> Void) {
        callback = onResolved
        // Should callback if the promise resolves before this method is invoked on the promise
        if case .resolved(let value) = state {
            callback?(value)
        }
    }
}
```
