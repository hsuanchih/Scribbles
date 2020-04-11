# Combine Publishers - Operator, Subject & \@Published

Here we're building on [Combine - Publisher, Subscriber & Subscription](iOS-OSX/Combine-Publisher-Subscriber-Subscription.md) to solidify our understanding of Combine by looking at some practical examples of publishers. Specifically, we'll explore implementations of an operator, a subject, and the \@Published property wrapper.

---
## Operator

Operators are special kinds of publishers in that they manipulate data received from the upstream to produce new data to the downstream. There are a lot of operators at our disposal. However our focus is here more on understanding how they work and not how they're used, so [here's a detailed list of all operators and when to use them](https://developer.apple.com/documentation/combine/publishers). What we'll do here instead is pick one from the list and look at how it does what it does.

Let's pick `Publishers.Map` as our example, and start with an example of how we use this operator.

```Swift
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

```Swift
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

```Swift
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

Now let's see the implementation for `PassthroughSubject`.

```Swift
public final class PassthroughSubject<Output, Failure: Error>: Subject  {

    private let _lock = UnfairRecursiveLock.allocate()

    private var _completion: Subscribers.Completion<Failure>?

    // TODO: Combine uses bag data structure
    private var _subscriptions: [Conduit] = []

    internal var upstreamSubscriptions: [Subscription] = []

    internal var hasAnyDownstreamDemand = false

    public init() {}

    deinit {
        for subscription in _subscriptions {
            subscription._downstream = nil
        }
        _lock.deallocate()
    }

    public func send(subscription: Subscription) {
        _lock.do {
            upstreamSubscriptions.append(subscription)
            if hasAnyDownstreamDemand {
                subscription.request(.unlimited)
            }
        }
    }

    public func receive<Downstream: Subscriber>(subscriber: Downstream)
        where Output == Downstream.Input, Failure == Downstream.Failure
    {
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

    public func send(completion: Subscribers.Completion<Failure>) {
        _lock.do {
            _completion = completion
            for subscriber in _subscriptions {
                subscriber._receive(completion: completion)
            }
        }
    }

    private func _acknowledgeDownstreamDemand() {
        _lock.do {
            guard !hasAnyDownstreamDemand else { return }
            hasAnyDownstreamDemand = true
            for subscription in upstreamSubscriptions {
                subscription.request(.unlimited)
            }
        }
    }
}

extension PassthroughSubject {

    fileprivate final class Conduit: Subscription {

        fileprivate var _parent: PassthroughSubject?

        fileprivate var _downstream: AnySubscriber<Output, Failure>?

        fileprivate var _demand: Subscribers.Demand = .none

        fileprivate var _isCompleted: Bool {
            return _parent == nil
        }

        fileprivate init(parent: PassthroughSubject,
                         downstream: AnySubscriber<Output, Failure>) {
            _parent = parent
            _downstream = downstream
        }

        fileprivate func _receive(completion: Subscribers.Completion<Failure>) {
            if !_isCompleted {
                _parent = nil
                _downstream?.receive(completion: completion)
            }
        }

        fileprivate func request(_ demand: Subscribers.Demand) {
            demand.assertNonZero()
            _parent?._lock.do {
                _demand += demand
            }
            _parent?._acknowledgeDownstreamDemand()
        }

        fileprivate func cancel() {
            _parent = nil
        }
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/PassthroughSubject.swift))

---
## \@Published

```Swift
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
