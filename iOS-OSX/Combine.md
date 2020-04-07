# Combine

## Fundamental Building Blocks

* Publishers
* Subscribers
* Operators

## Logical Overview

<img src="images/combine-publisher-subscriber.png" height="420"/>

---
## Publishers

```Swift
protocol Publisher {
    associatedType Output
    associatedType Failure : Error
    
    func subscribe<S>(_ subscriber: S) where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
}
```

```Swift
extension NotificationCenter {
    struct Publisher : Combine.Publisher {
        typealias Output = Notification
        typealias Failure = Never
        init(center: NotificationCenter, name: Notification.Name, object: Any? = nil)
    }
}
```
---
## Subscribers

```Swift
protocol Subscriber {
    associatedtype Input
    associatedtype Failure : Error
    
    func receive(subscription: Subscription)
    func receive(_ input: Self.Input) -> Subscribers.Demand
    func receive(completion: Subscribers.Completion<Self.Failure>)
}
```

```Swift
extension Subscribers {
    class Assign<Root, Input>: Subscriber, Cancellable {
        typealias Failure = Never
        init(object: Root, keyPath: ReferenceWritableKeyPath<Root, Input>)
    }
}
```
---
## Operators
