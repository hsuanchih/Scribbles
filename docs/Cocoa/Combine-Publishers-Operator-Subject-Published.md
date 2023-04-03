---
layout: default
title: Combine Publishers - Operator, Subject & @Published
parent: Cocoa
---

# Combine Publishers - Operator, Subject & @Published

Here we're building on [Combine - Publisher, Subscriber & Subscription](Combine-Publisher-Subscriber-Subscription.md) to solidify our understanding of Combine by looking at some practical examples of publishers. Specifically, we'll explore implementations of an operator, a subject, and the \@Published property wrapper.

---
## Operator

Operators are special kinds of publishers in that they manipulate data received from the upstream to produce new data to the downstream. There are a lot of operators at our disposal. However our focus is here more on understanding how they work and not how they're used, so [here's a detailed list of all operators and when to use them](https://developer.apple.com/documentation/combine/publishers). What we'll do here instead is pick one from the list and look at how it does what it does.

Let's pick `Publishers.Map` as our example, and start with an example of how we use this operator.

```swift
// Here we define a publisher that publishes values from 1 to 5,
// and pipes its downstream with a Publishers.Map operator.
// The operator transforms the output of the publisher by squaring the values it produces.

// Finally we attach a subscriber of type Sink which outputs:
// "Completed with:" with a Subscribers.Completion instance when a completion event is received, or
// "Received value:" with the value received when a value is received
Array(1...5).publisher.map { $0*$0 }.sink(receiveCompletion: { print("Completed with: \($0)") })
    { print("Received value: \($0)") }
```
Now let's see how the `Publishers.Map` operator does what it does:

```swift
extension Publishers {
    
    public struct Map<Upstream: Publisher, Output>: Publisher {
        
        // This publisher's failure type must match its upstream
        public typealias Failure = Upstream.Failure

        // The publisher from which this publisher receives elements.
        public let upstream: Upstream

        // The closure that transforms elements from the upstream publisher.
        public let transform: (Upstream.Output) -> Output

        // This publisher's initializer requires:
        // 1. Another publisher that is its upstream
        // 2. A closure that defines how values from the upstream should be transformed
        public init(upstream: Upstream,
                    transform: @escaping (Upstream.Output) -> Output) {
            self.upstream = upstream
            self.transform = transform
        }
        
        // Publisher protocol conformance:
        // When the subscriber calls subscribe on this publisher passing itself as the subscriber,
        // this publisher wraps the subscriber (alongside the transformation closure) inside another 
        // subscriber type Inner, and calls subscribe on its upstream, this time passing the Inner
        // instance as parameter.
        public func receive<Downstream: Subscriber>(subscriber: Downstream)
            where Output == Downstream.Input, Downstream.Failure == Upstream.Failure {
            upstream.subscribe(Inner(downstream: subscriber, map: transform))
        }
        
        // This is convenience method for piping this operator with another map operator:
        // A new Publisher.Map is created using its initializer
        public func map<Result>(_ transform: @escaping (Output) -> Result) -> Publishers.Map<Upstream, Result> {
            return .init(upstream: upstream) { transform(self.transform($0)) }
        }
    }
}

extension Publishers.Map {

    // The Inner type is a subscriber that wraps its downstream
    private struct Inner<Downstream: Subscriber> : Subscriber, CustomCombineIdentifierConvertible
        where Downstream.Input == Output, Downstream.Failure == Upstream.Failure {
        
        // The subscriber's input & failure types must match its upstream
        typealias Input = Upstream.Output
        typealias Failure = Upstream.Failure
        
        // CustomCombineIdentifierConvertible protocol conformance
        let combineIdentifier = CombineIdentifier()

        // This subscriber manages:
        // 1. The downstream subscriber
        // 2. The value transformation
        private let downstream: Downstream
        private let map: (Input) -> Output
        fileprivate init(downstream: Downstream, map: @escaping (Input) -> Output) {
            self.downstream = downstream
            self.map = map
        }

        // Subscriber protocol conformance:
        func receive(subscription: Subscription) {
        
            // Forward subscription to its downstream when it receives
            // one from its upstream
            downstream.receive(subscription: subscription)
        }
        func receive(_ input: Input) -> Subscribers.Demand {
            
            // Apply transformation on the value received from upstream, 
            // and forward it downstream
            return downstream.receive(map(input))
        }
        func receive(completion: Subscribers.Completion<Failure>) {
            
            // Forward the completion event downstream
            downstream.receive(completion: completion)
        }
    }
}

extension Publisher {
    
    // We can call map on the upstream publisher to attach a Publishers.Map publisher to it
    // (like we did in our example)
    public func map<Result>(_ transform: @escaping (Output) -> Result) -> Publishers.Map<Self, Result> {
        
        // Create a new Publisher.Map publisher, setting the called publisher as its upstream and passing
        // in the transformation closure
        return Publishers.Map(upstream: self, transform: transform)
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Publishers/Publishers.Map.swift))

---
## Subject

A subject is a publisher that offers methods to outside callers, allowing them to publish values & events. The way a subject achieves this is by conforming to the `Subject` protocol. Concrete subjects that come out-of-the-box with Combine are `CurrentValueSubject` and `PassthroughSubject`.

We are going to look at the `Subject` protocol as a warm up to understanding the `PassthroughSubject` implementation.

```swift
protocol Subject: AnyObject, Publisher {

    // Sends a value to the subscriber.
    func send(_ value: Output)

    // Sends a completion event to the subscriber.
    func send(completion: Subscribers.Completion<Failure>)

    // Provides this Subject an opportunity to establish demand for any new upstream
    // subscriptions
    func send(subscription: Subscription)
}

extension Subject where Output == Void {

    // If the output of the publisher is void, 
    // the send(_ value: Output) method should sends a void
    public func send() {
        send(())
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Subject.swift))

A `PassthroughSubject` is a publisher typically used to as an adaptor to bridge existing imperative code to the publish/subscribe model. Let's see the implementation for `PassthroughSubject`.

```swift
public final class PassthroughSubject<Output, Failure: Error>: Subject  {

    // Locking mechanism to enforce atomic operations
    private let _lock = UnfairRecursiveLock.allocate()

    // This property tracks whether the subject has sent a completion event 
    private var _completion: Subscribers.Completion<Failure>?

    // This list keeps subscriptions (upstream subscriptions) sent by the subject
    internal var upstreamSubscriptions: [Subscription] = []
    
    // This list keeps subscriptions (downstream subscriptions) associated with subscribers
    private var _subscriptions: [Conduit] = []

    // This property tracks whether the subject has received a demand from any downstream 
    // subscriber
    internal var hasAnyDownstreamDemand = false
    
    // Default initializer with no parameters
    public init() {}

    // Publisher protocol conformance
    // 
    public func receive<Downstream: Subscriber>(subscriber: Downstream)
        where Output == Downstream.Input, Failure == Downstream.Failure {
        _lock.do {
            if let completion = _completion {
                subscriber.receive(subscription: Subscriptions.empty)
                subscriber.receive(completion: completion)
                return
            } else {
                let subscription = Conduit(parent: self,
                                           downstream: AnySubscriber(subscriber))

                _subscriptions.append(subscription)
                subscriber.receive(subscription: subscription)
            }
        }
    }

    // Subject protocol conformance
    // Call this method on the subject to send subscription to the subscriber
    // Subscriptions injected are added to upstream subscriptions
    // Injected subscriptions are requested unlimited demands regardless of demand
    // requested by the downstream
    public func send(subscription: Subscription) {
        _lock.do {
            upstreamSubscriptions.append(subscription)
            if hasAnyDownstreamDemand {
                subscription.request(.unlimited)
            }
        }
    }
    // Call this method on the subject to send values to each subscriber
    // to send the value to all subscribers, and update each downstream subscription
    // demand with ensuing demands returned from the subscriber  
    public func send(_ input: Output) {
        _lock.do {
            for subscription in _subscriptions
                where !subscription._isCompleted && subscription._demand > 0
            {
                let newDemand = subscription._downstream?.receive(input) ?? .none
                subscription._demand += newDemand
                subscription._demand -= 1
            }
        }
    }
    // Call this method on the subject to send completion event to each subscriber
    // The subscription is no longer effective once the completion event is sent
    public func send(completion: Subscribers.Completion<Failure>) {
        _lock.do {
            _completion = completion
            for subscriber in _subscriptions {
                subscriber._receive(completion: completion)
            }
        }
    }

    // Downstream subscription call this method when subscriber request demand
    // through it
    private func _acknowledgeDownstreamDemand() {
        _lock.do {
            guard !hasAnyDownstreamDemand else { return }
            hasAnyDownstreamDemand = true
            for subscription in upstreamSubscriptions {
                subscription.request(.unlimited)
            }
        }
    }
    
    // Release all downstream subscribers on deallocation
    deinit {
        for subscription in _subscriptions {
            subscription._downstream = nil
        }
        _lock.deallocate()
    }
}

extension PassthroughSubject {

    // Conduit is a concrete subscription type
    fileprivate final class Conduit: Subscription {

        // This property holds the demand requested by the subscriber
        fileprivate var _demand: Subscribers.Demand = .none
        
        // This computed property returns whether the subscription is complete:
        // the subscription is considered complete when the completion event
        // is received by the subscriber
        fileprivate var _isCompleted: Bool {
            return _parent == nil
        }
        
        // The owner of this subscription is a PassthroughSubject, and the PassthroughSubject's
        // downstream is a subscriber
        fileprivate var _parent: PassthroughSubject?
        fileprivate var _downstream: AnySubscriber<Output, Failure>?
        fileprivate init(parent: PassthroughSubject, downstream: AnySubscriber<Output, Failure>) {
            _parent = parent
            _downstream = downstream
        }

        // The subscription forwards the completion event downstream when the PassthroughSubject
        // sends the completion event
        fileprivate func _receive(completion: Subscribers.Completion<Failure>) {
            if !_isCompleted {
                _parent = nil
                _downstream?.receive(completion: completion)
            }
        }

        // Subscription protocol conformance
        // The subscriber calls this method on the subscription to specify
        // the number of values it wishes to receive before completion
        // Upstream subscriptions injected through the subject always receives
        // an unmilimted demand regardless of the actual demand requested by the subscriber
        fileprivate func request(_ demand: Subscribers.Demand) {
            demand.assertNonZero()
            _parent?._lock.do {
                _demand += demand
            }
            _parent?._acknowledgeDownstreamDemand()
        }
        
        // Cancellable protocol conformance
        // The subscription/subject relationship is de-coupled when the subscriber
        // cancels the subscription
        fileprivate func cancel() {
            _parent = nil
        }
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/PassthroughSubject.swift))

---
## @Published

Now we have a better idea of what a `PassthroughSubject` is and how it works, we can now look at how it is used to formulate the `@Published` property wrapper. Let's first review how the `@Published` is used with this example from [Apple](https://developer.apple.com/documentation/combine/published):

```swift
class Weather {
    @Published var temperature: Double
    init(temperature: Double) {
        self.temperature = temperature
    }
}

let weather = Weather(temperature: 20)
cancellable = weather.$temperature
    .sink() {
        print ("Temperature now: \($0)")
}
weather.temperature = 25

// Prints:
// Temperature now: 20.0
// Temperature now: 25.0
```

Now let's implement `@Published`:

```swift
@propertyWrapper
struct Published<Value> {
    private let subject = PassthroughSubject<Value, Never>()
    var wrappedValue : Value {
        didSet { subject.send(wrappedValue) }
    }
    public var projectedValue: PassthroughSubject<Value, Never> {
        subject
    }
}
```
