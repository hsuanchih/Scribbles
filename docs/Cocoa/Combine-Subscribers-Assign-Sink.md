---
layout: default
title: Combine Subscribers - Assign & Sink
parent: Cocoa
---

# Combine Subscribers - Assign & Sink

We learned the basics of subscribers in [Combine - Publisher, Subscriber & Subscription](Combine-Publisher-Subscriber-Subscription.md) and implemented our own. Here we're extending our discussion on subscribers by looking at two concrete subscriber types that come with Combine:
* `Subscribers.Assign`
* `Subscribers.Sink`

---
## Subscribers.Assign

`Subscribers.Assign` is subscriber intended for assigning a publisher’s output to a property of an object. Let's see an example of how it's used before we break down how it works.

```swift
// A ValuePrinter prints its value to the console whenever its value changes
class ValuePrinter<Value> {
    var value: Value {
        didSet {
            print(value)
        }
    }
    init(_ value: Value) {
        self.value = value
    }
}

// Here we define a publisher that publishes values from 1 to 5
// and register to it a subscriber of type Assign, who then writes
// the values received to the ValuePrinter's value property
Array(1...5).publisher.assign(to: \.value, on: ValuePrinter(0))


// And the console outputs:
// 1
// 2
// 3
// 4
// 5
```

Now let's look at the implementation of `Subscribers.Assign`.

```swift
// A Subscriber can be in one of 3 states:
// - Awaiting a subscription
// - Subscribed with a subscription
// - Terminated subscription
enum SubscriptionStatus {
    case awaitingSubscription
    case subscribed(Subscription)
    case terminal
}

extension Subscribers {
    
    // Subscribers.Assign is a cancellable subscriber that never fails
    public final class Assign<Root, Input>: Subscriber, Cancellable {
        public typealias Failure = Never

        // This subscriber holds the destination object and keypath of its destination property
        // to which the subscriber should write to whenever it receives a value
        public private(set) var object: Root?
        public let keyPath: ReferenceWritableKeyPath<Root, Input>

        // The subscriber's subscription status is initially in the awaiting state
        // Note that SubscriptionStatus holds the subscription for the subscriber
        private var status = SubscriptionStatus.awaitingSubscription
        
        // The subscriber's initializer requires the object & property keypath of the destination
        // object be designated
        public init(object: Root, keyPath: ReferenceWritableKeyPath<Root, Input>) {
            self.object = object
            self.keyPath = keyPath
        }

        // The subscriber receives a subscription
        // 1. If the subscriber is currently subscribed or the previous subscription has been cancelled:
        //    the received subscription will be cancelled
        // 2. If the subscriber is awaiting a subscription:
        //    the received subscription is accepted, and the subscriber makes a request for 
        //    unlimited demand to the subscription
        public func receive(subscription: Subscription) {
            switch status {
            case .subscribed, .terminal:
                subscription.cancel()
            case .awaitingSubscription:
                status = .subscribed(subscription)
                subscription.request(.unlimited)
            }
        }

        // The subscriber receives a value
        // Write to the destination object's property through its keypath only if the
        // the subscriber is currently subscribed
        // Ignore the value otherwise
        public func receive(_ value: Input) -> Subscribers.Demand {
            switch status {
            case .subscribed:
                object?[keyPath: keyPath] = value
            case .awaitingSubscription, .terminal:
                break
            }
            return .none
        }
        
        // The subscriber receives a completion event
        // Subscribers.Assign never fails & ignores the completion event altogether
        // The subscription is cancelled when the completion event is received nonetheless
        public func receive(completion: Subscribers.Completion<Never>) {
            cancel()
        }

        // Call cancel to cancel the subscription
        // If the subscriber is currently susbcribed, cancel the subscription and move
        // SubscriptionStatus to terminated
        public func cancel() {
            guard case let .subscribed(subscription) = status else {
                return
            }
            subscription.cancel()
            status = .terminal
            object = nil
        }
    }
}

extension Publisher where Failure == Never {

    // We can call assign on the publisher to attach a Subscribers.Assign subscriber to it
    // (like we did in our example)
    public func assign<Root>(to keyPath: ReferenceWritableKeyPath<Root, Output>,
                             on object: Root) -> AnyCancellable {
        
        // 1. Create a Subscribers.Assign subscriber using its default initializer
        // 2. Subscribe to the puslisher passing in the subscriber object
        // 3. Return the subscriber object wrapped as AnyCancellable
        let subscriber = Subscribers.Assign(object: object, keyPath: keyPath)
        subscribe(subscriber)
        return AnyCancellable(subscriber)
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Subscribers/Subscribers.Assign.swift))

---
## Subscribers.Sink

`Subscribers.Sink` is subscriber whose behavior is defined by closure-wrapped logic assigned to it. Again, let's see an example of how it's used before we break down how it works.

```swift
// Here we define a publisher that publishes values from 1 to 5
// and register to it a subscriber of type Sink which outputs:
// "Completed with:" with a Subscribers.Completion instance when a completion event is received, or
// "Received value:" with the value received when a value is received
Array(1...5).publisher.sink(receiveCompletion: { print("Completed with: \($0)") }) 
    { print("Received value: \($0)") }


// And the console outputs:
// Received value: 1
// Received value: 2
// Received value: 3
// Received value: 4
// Received value: 5
// Completed with: finished
```

Now let's see the implementation of `Subscribers.Sink`.

```swift
// A Subscriber can be in one of 3 states:
// - Awaiting a subscription
// - Subscribed with a subscription
// - Terminated subscription
enum SubscriptionStatus {
    case awaitingSubscription
    case subscribed(Subscription)
    case terminal
}

extension Subscribers {

    // Subscribers.Sink is a cancellable subscriber that can fails
    // A simple subscriber that requests an unlimited number of values upon subscription.
    public final class Sink<Input, Failure: Error> : Subscriber, Cancellable {
        
        // This subscriber's behavior is user-defined, respectively via:
        // 1. Closure to handle received values
        // 2. Closure to handle completion event
        public let receiveValue: (Input) -> Void
        public let receiveCompletion: (Subscribers.Completion<Failure>) -> Void

        // The subscriber's subscription status is initially in the awaiting state
        // Note that SubscriptionStatus holds the subscription for the subscriber
        private var status = SubscriptionStatus.awaitingSubscription
        
        // The subscriber's initializer requires:
        // 1. A closure that defines how received values should be handled
        // 2. A closure that defines how the completion event should be handled
        public init(
            receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void,
            receiveValue: @escaping ((Input) -> Void)
        ) {
            self.receiveCompletion = receiveCompletion
            self.receiveValue = receiveValue
        }

        // The subscriber receives a subscription
        // 1. If the subscriber is currently subscribed or the previous subscription has been cancelled:
        //    the received subscription will be cancelled
        // 2. If the subscriber is awaiting a subscription:
        //    the received subscription is accepted, and the subscriber makes a request for 
        //    unlimited demand to the subscription
        public func receive(subscription: Subscription) {
            switch status {
            case .subscribed, .terminal:
                subscription.cancel()
            case .awaitingSubscription:
                status = .subscribed(subscription)
                subscription.request(.unlimited)
            }
        }
        
        // The subscriber receives a value
        // Defer to user-defined logic to handle the value
        public func receive(_ value: Input) -> Subscribers.Demand {
            receiveValue(value)
            return .none
        }
        
        // The subscriber receives a completion event
        // Defer to user-defined logic to handle the completion event
        public func receive(completion: Subscribers.Completion<Failure>) {
            receiveCompletion(completion)
            status = .terminal
        }

        // Call cancel to cancel the subscription
        // If the subscriber is currently susbcribed, cancel the subscription and move
        // SubscriptionStatus to terminated
        public func cancel() {
            guard case let .subscribed(subscription) = status else {
                return
            }
            subscription.cancel()
            status = .terminal
        }
    }
}

extension Publisher {

    // We can call sink on the publisher to attach a Subscribers.Sink subscriber to it
    // (like we did in our example)
    public func sink(
        receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void,
        receiveValue: @escaping ((Output) -> Void)
    ) -> AnyCancellable {

        // 1. Create a Subscribers.Sink subscriber using its default initializer
        // 2. Subscribe to the puslisher passing in the subscriber object
        // 3. Return the subscriber object wrapped in a AnyCancellable type
        let subscriber = Subscribers.Sink<Output, Failure>(
            receiveCompletion: receiveCompletion,
            receiveValue: receiveValue
        )
        subscribe(subscriber)
        return AnyCancellable(subscriber)
    }
}

extension Publisher where Failure == Never {
    
    // This sink call attaches a Subscribers.Sink subscriber to a publisher that never fails,
    // and is the one to use if the subscriber is not interested in the completion event
    public func sink(
        receiveValue: @escaping (Output) -> Void
    ) -> AnyCancellable {
        
        // 1. Create a Subscribers.Sink subscriber using its default initializer, passing
        //    a no-action completion event handler as default value
        // 2. Subscribe to the puslisher passing in the subscriber object
        // 3. Return the subscriber object wrapped in a AnyCancellable type
        let subscriber = Subscribers.Sink<Output, Failure>(
            receiveCompletion: { _ in },
            receiveValue: receiveValue
        )
        subscribe(subscriber)
        return AnyCancellable(subscriber)
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Subscribers/Subscribers.Sink.swift))
