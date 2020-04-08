# Combine

A lot of the times we're interested in receiving updates about value and state changes over time. In the past we've relied on delegation, callbacks/closures, and notifications (NotificationCenter/Key-Value-Observation) to get it done. Today we're going to look at the publisher/subscriber approach through Combine.

## Fundamental Building Blocks

Let's first establish some terminology forming the fundamental building blocks of combine:
* __Publisher__ - Party responsible for defining how values/errors are produced, but not necessarily the content producer itself
* __Subscriber__ - Party interested in the value changes, and expresses its interest by subscribing to a publisher
* __Operator__

## Logical Overview

On the one hand we have a publisher that can publish events over time, and on the other, a subscriber that is interested in knowing about these events - how do we connect them together? This connection is established through a third party called a subscription. Here's a step-by-step on how it happens:

<img src="images/combine-publisher-subscriber.png" height="420"/>

1. The subscriber expresses its interest in events from the publisher by subscribing itself.
2. In turn the publisher creates a subscription sends it to the subscriber.
3. The subscriber requests a number of values from the subscription before it begins to receive events.
4. Publisher continues to publish events to the subscriber until either the terms of the subscription is fulfilled or an error ocurrs.

Keep this overview in mind as it will form the foundation of our exploration in this chapter. 

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
The associated types are straightforward, but what should the body of `func subscribe<S>(_ subscriber: S)`, `func receive<S>(subscriber: S)`, and `func subscribe<S>(_ subject: S) -> AnyCancellable` look like? And what is a subject anyway? We'll touch on all of these one at a time, but let's start with some default implementations in the `Publisher` protocol extension:

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
That's a bit of clarity - the protocol extension provides default implementations for 2 of the 3 methods, leaving us with `func receive<S>(subscriber: S)`. There are a few concrete publisher types that come with Combine out-of-the-box. We'll have a peek under the hood to gain better intuition for how to implement our own.

### Just:

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
