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
