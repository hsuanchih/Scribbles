# Combine

A lot of the times we're interested in receiving updates about value and state changes over time. In the past we've relied on delegation, callbacks/closures, and notifications (NotificationCenter/Key-Value-Observation) to get it done. Today we're going to look at the publisher/subscriber approach through Combine.

## Fundamental Building Blocks

Let's first establish some terminology forming the fundamental building blocks of combine:
* __Subscription__ - Entity representing the connection of s subscriber to a publisher
* __Publisher__ - Entity responsible for defining how values/errors are produced, but not necessarily the content producer itself
* __Subscriber__ - Entity interested in the value changes, and expresses its interest by subscribing to a publisher
* __Operator__

## Logical Overview

On the one hand we have a publisher that can publish events over time, and on the other, a subscriber that is interested in knowing about these events - how do we connect them together? This connection is established through a third entity called a subscription. Here's a step-by-step on how it happens:

<img src="images/combine-publisher-subscriber.png" height="420"/>

1. The subscriber expresses its interest in events from the publisher by subscribing itself.
2. In turn the publisher creates a subscription sends it to the subscriber.
3. The subscriber requests a number of values from the subscription before it begins to receive events.
4. Publisher continues to publish events to the subscriber until either the terms of the subscription is fulfilled or an error ocurrs.

Keep this overview in mind as it will form the foundation of our exploration in this chapter. 

---
## Subscription

It makes the most sense to kick-off our exploration with a subscription as it serves as the binding between a publisher & a subscriber. The `Subscription` protocol is one that all concrete subscription types must conform to. Let's look at the contract:

```Swift
protocol Subscription: Cancellable, CustomCombineIdentifierConvertible {

    // A subscriber calls this method on the subscription with a demand to let the publisher 
    // know the number of values it may send to the subscriber
    func request(_ demand: Subscribers.Demand)
}
```
Succint, but we're introduced to a few new types here. Let's go over them one by one:
```Swift
// Cancellable protocol is an indicator that an activity/action supports cancellation
protocol Cancellable {
    // Call cancel on a concrete Cancellable type to cancel an action/activity.
    func cancel()
}

// CustomCombineIdentifierConvertible protocol provides unique identifier to publisher streams
// A default implementation comes with its protocol extension
protocol CustomCombineIdentifierConvertible {
    var combineIdentifier: CombineIdentifier { get }
}
extension CustomCombineIdentifierConvertible where Self: AnyObject {
    public var combineIdentifier: CombineIdentifier {
        return CombineIdentifier(self)
    }
}

// A requested number of items, sent to a publisher from a subscriber through the subscription
extension Subscribers {
    struct Demand, Codable, CustomStringConvertible {
    
        // A subscriber can demand a limited number of elements to be sent
        static func max(_ value: Int) -> Subscribers.Demand
        
        // A subscriber can demand an incessant number of elements to be sent
        static let unlimited: Subscribers.Demand
        
        // A subscriber can demand no elements to be sent
        static let none: Subscribers.Demand
    }
}
```

---
## Publisher

Before we get into concrete publisher types, let's first have a look at the contract a publisher will need to fulfill:
```Swift
protocol Publisher {

    // A concrete publisher must specify its:
    // 1. Output type
    // 2. Failure type - if the publisher never fails, its failure type can be declared Never
    associatedType Output
    associatedType Failure : Error
    
    // A subscriber calls subscribe on the publisher to register its interest for the publisher
    // The constraints here specifies that: 
    // 1. The subscriber's input type must match the publisher's output type
    // 2. The subscriber's & publisher's error types must also match
    func subscribe<S>(_ subscriber: S) 
        where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
    
    // Attaches the specified subscriber to this publisher.
    func receive<S>(subscriber: S) 
        where S : Subscriber, Self.Failure == S.Failure, Self.Output == S.Input
    
    // Attaches the specified subject to this publisher.
    func subscribe<S>(_ subject: S) -> AnyCancellable 
        where S : Subject, Self.Failure == S.Failure, Self.Output == S.Output
}
```
The associated types are straightforward, but what should the body these look like? 
* `func subscribe<S>(_ subscriber: S)`, 
* `func receive<S>(subscriber: S)`, and 
* `func subscribe<S>(_ subject: S) -> AnyCancellable` 

And what is a subject anyway? We'll touch on all of these one at a time, but let's start with some default implementations in the `Publisher` protocol extension:

```Swift
extension Publisher {
    public func subscribe<Subscriber: Combine.Subscriber>(_ subscriber: Subscriber) 
        where Failure == Subscriber.Failure, Output == Subscriber.Input {
        
        // Call receive to attach the specified subscriber to this publisher
        receive(subscriber: subscriber)
    }

    public func subscribe<Subject: Combine.Subject>(_ subject: Subject) -> AnyCancellable
        where Failure == Subject.Failure, Output == Subject.Output {
        
        // Convert a subject into a subscriber
        let subscriber = SubjectSubscriber(subject)
        
        // Call subscribe to attach the subscriber to the publisher
        self.subscribe(subscriber)
        
        // Return a Cancellable object from the subscriber
        return AnyCancellable(subscriber)
    }
}
```
That's a bit of clarity - the protocol extension provides default implementations for 2 of the 3 methods, leaving us with `func receive<S>(subscriber: S)`. If you remember the logical overview from earlier this is where the publisher sends the subscription over to the subscriber.

---
## Subscriber

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
## Operator
