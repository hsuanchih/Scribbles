# Combine Publishers - Operator, Subject & \@Published

Here we're building on [Combine - Publisher, Subscriber & Subscription](iOS-OSX/Combine-Publisher-Subscriber-Subscription.md) to solidify our understanding of Combine by looking at some practical examples of publishers. Specifically, we'll explore implementations of an operator, a subject, and the \@Published property wrapper.

---
## Operator

Operators are special kinds of publishers in that they manipulate data received from the upstream to produce new data to the downstream. There are a lot of operators at our disposal. However our focus is here more on understanding how they work and not how they're used, so [here's a detailed list of all operators and when to use them](https://developer.apple.com/documentation/combine/publishers). What we'll do here instead is pick one from the list and look at how it does what it does.

---
## Subject

```Swift
/// A publisher that exposes a method for outside callers to publish elements.
///
/// A subject is a publisher that you can use to ”inject” values into a stream, by calling
/// its `send()` method. This can be useful for adapting existing imperative code to the
/// Combine model.
protocol Subject: AnyObject, Publisher {

    /// Sends a value to the subscriber.
    ///
    /// - Parameter value: The value to send.
    func send(_ value: Output)

    /// Sends a completion signal to the subscriber.
    ///
    /// - Parameter completion: A `Completion` instance which indicates whether publishing
    ///   has finished normally or failed with an error.
    func send(completion: Subscribers.Completion<Failure>)

    /// Provides this Subject an opportunity to establish demand for any new upstream
    /// subscriptions (say, via `Publisher.subscribe<S: Subject>(_: Subject)`)
    func send(subscription: Subscription)
}

extension Subject where Output == Void {

    /// Signals subscribers.
    public func send() {
        send(())
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Subject.swift))


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
