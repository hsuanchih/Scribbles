# Combine Subscribers - Assign & Sink

We learned the basics of subscribers in [Combine - Publisher, Subscriber & Subscription](iOS-OSX/Combine-Publisher-Subscriber-Subscription.md) and implemented our own. Here we're extending our discussion on subscribers by looking at two concrete subscriber types that come with Combine:
* `Subscribers.Assign`
* `Subscribers.Sink`

---
## Assign

`Subscribers.Assign` is subscriber capable of assigning a publisher’s output to a property of an object. Let's see an example of how it's used before we go into how it works.

```Swift
// A ValuePrinter prints its value to the console
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


// As expected, the console outputs:
// 1
// 2
// 3
// 4
// 5
```

Now let's look at the implementation of `Subscribers.Assign`.

```Swift
enum SubscriptionStatus {
    case awaitingSubscription
    case subscribed(Subscription)
    case terminal
}

extension Subscribers {

    public final class Assign<Root, Input>: Subscriber, Cancellable {
        
        public typealias Failure = Never

        public private(set) var object: Root?

        public let keyPath: ReferenceWritableKeyPath<Root, Input>

        private var status = SubscriptionStatus.awaitingSubscription

        public init(object: Root, keyPath: ReferenceWritableKeyPath<Root, Input>) {
            self.object = object
            self.keyPath = keyPath
        }

        public func receive(subscription: Subscription) {
            switch status {
            case .subscribed, .terminal:
                subscription.cancel()
            case .awaitingSubscription:
                status = .subscribed(subscription)
                subscription.request(.unlimited)
            }
        }

        public func receive(_ value: Input) -> Subscribers.Demand {
            switch status {
            case .subscribed:
                object?[keyPath: keyPath] = value
            case .awaitingSubscription, .terminal:
                break
            }
            return .none
        }

        public func receive(completion: Subscribers.Completion<Never>) {
            cancel()
        }

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

    public func assign<Root>(to keyPath: ReferenceWritableKeyPath<Root, Output>,
                             on object: Root) -> AnyCancellable {
        let subscriber = Subscribers.Assign(object: object, keyPath: keyPath)
        subscribe(subscriber)
        return AnyCancellable(subscriber)
    }
}
```
(Snippet from [OpenCombine](https://github.com/broadwaylamb/OpenCombine/blob/master/Sources/OpenCombine/Subscribers/Subscribers.Assign.swift))

---
## Sink

```Swift
enum SubscriptionStatus {
    case awaitingSubscription
    case subscribed(Subscription)
    case terminal
}

extension Subscribers {

    // A simple subscriber that requests an unlimited number of values upon subscription.
    public final class Sink<Input, Failure: Error> : Subscriber, Cancellable {
        
        public let receiveValue: (Input) -> Void

        public let receiveCompletion: (Subscribers.Completion<Failure>) -> Void

        private var status = SubscriptionStatus.awaitingSubscription

        public init(
            receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void,
            receiveValue: @escaping ((Input) -> Void)
        ) {
            self.receiveCompletion = receiveCompletion
            self.receiveValue = receiveValue
        }

        public func receive(subscription: Subscription) {
            switch status {
            case .subscribed, .terminal:
                subscription.cancel()
            case .awaitingSubscription:
                status = .subscribed(subscription)
                subscription.request(.unlimited)
            }
        }

        public func receive(_ value: Input) -> Subscribers.Demand {
            receiveValue(value)
            return .none
        }

        public func receive(completion: Subscribers.Completion<Failure>) {
            receiveCompletion(completion)
            status = .terminal
        }

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

    public func sink(
        receiveCompletion: @escaping (Subscribers.Completion<Failure>) -> Void,
        receiveValue: @escaping ((Output) -> Void)
    ) -> AnyCancellable {

        let subscriber = Subscribers.Sink<Output, Failure>(
            receiveCompletion: receiveCompletion,
            receiveValue: receiveValue
        )
        subscribe(subscriber)
        return AnyCancellable(subscriber)
    }
}

extension Publisher where Failure == Never {

    public func sink(
        receiveValue: @escaping (Output) -> Void
    ) -> AnyCancellable {

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
