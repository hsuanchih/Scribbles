---
layout: default
title: Memory Management in RxSwift
parent: RxSwift
---

# Memory Management in RxSwift
---
## The Question
What happens when an `Observer` subscribes to an `Observable` in RxSwift?
```swift
func someFunction() {
    // Setup a local observable
    let observable: Observable<Int> = Observable.create { observer in
        // Send value 5 to the observer
        observer.onNext(5)
        return Disposables.create()
    }
    
    // Setup a local observer
    let observer: AnyObserver<Int> = AnyObserver { event in
        switch event {
        case .next(let value):
            // We expect the value to be 5 here
            print(value)
        case .completed, .error:
            break
        }
    }
    
    // The observer subscribes to the observable   
    observable.subscribe(observer)
    
    // Question:
    // Do the observable/observer remain in memory after the function returns?
}
```
---
## Code-Level Decomposition

### AnonymousObservable\<Element\>
A concrete observable to store the subscribe handler passed into `Observable.create` call.
```swift
extension ObservableType {
    public static func create(_ subscribe: @escaping (AnyObserver<Element>) -> Disposable) -> Observable<Element> {
        AnonymousObservable(subscribe)
    }
}

final private class AnonymousObservable<Element>: Producer<Element> {
    typealias SubscribeHandler = (AnyObserver<Element>) -> Disposable

    let subscribeHandler: SubscribeHandler

    init(_ subscribeHandler: @escaping SubscribeHandler) {
        self.subscribeHandler = subscribeHandler
    }

    override func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        // An AnonymousObservableSink is created with references to:
        // * The Observer
        // * The SinkDisposer
        let sink = AnonymousObservableSink(observer: observer, cancel: cancel)
        
        let subscription = sink.run(self)
        
        // Return:
        // * AnonymousObservableSink as the sink
        // * The disposable created & returned from the Observable.create closure as the subscription 
        return (sink: sink, subscription: subscription)
    }
}
```

### Producer\<Element\>
A concrete `Observable` type that inherits from the __abstract__ type `Observable<Element>`.
```swift
class Producer<Element>: Observable<Element> {
    override init() {
        super.init()
    }

    override func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        if !CurrentThreadScheduler.isScheduleRequired {
            // The following happens when we call subscribe on an Observable:
            // 1. A SinkDisposer is created
            // 2. The Observable calls its implementation of the run function, and returns a Sink & Subscription
            // 3. Assign the Sink & Subscription to the SinkDisposer
            // 4. Return the SinkDisposer
            let disposer = SinkDisposer()
            let sinkAndSubscription = self.run(observer, cancel: disposer)
            disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)

            return disposer
        }
        else {
            return CurrentThreadScheduler.instance.schedule(()) { _ in
                let disposer = SinkDisposer()
                let sinkAndSubscription = self.run(observer, cancel: disposer)
                disposer.setSinkAndSubscription(sink: sinkAndSubscription.sink, subscription: sinkAndSubscription.subscription)

                return disposer
            }
        }
    }

    func run<Observer: ObserverType>(_ observer: Observer, cancel: Cancelable) -> (sink: Disposable, subscription: Disposable) where Observer.Element == Element {
        rxAbstractMethod()
    }
}
```

### Sink\<Observer\>
In the logical sense, a `Sink` serves as a bridge between the `Observable` and the `Observer`.</br>
In practice, a `Sink` is used by RxSwift to store the logic used to designate the `Producer`, and maintains references to:
* Its observer, and
* A `SinkDisposer`

```swift
class Sink<Observer: ObserverType>: Disposable {
    // This property keeps a reference to the observer
    fileprivate let observer: Observer
    
    // This property keeps a reference to the SinkDisposer
    fileprivate let cancel: Cancelable
    
    // This flag tracks whether the sink is disposed
    private let disposed = AtomicInt(0)

    init(observer: Observer, cancel: Cancelable) {
        self.observer = observer
        self.cancel = cancel
    }

    final func forwardOn(_ event: Event<Observer.Element>) {
        // Send an event to the observer only if the Sink has not yet been disposed
        if isFlagSet(self.disposed, 1) {
            return
        }
        self.observer.on(event)
    }

    final var isDisposed: Bool {
        // Check if the sink has been disposed
        isFlagSet(self.disposed, 1)
    }

    func dispose() {
        // Update the disposed flag on the Sink, and call dispose on the SinkDisposer
        fetchOr(self.disposed, 1)
        self.cancel.dispose()
    }
}
```

### AnonymousObservableSink\<Observer\>
A sink to the `AnonymousObservable` type.
```swift
final private class AnonymousObservableSink<Observer: ObserverType>: Sink<Observer>, ObserverType {
    typealias Element = Observer.Element 
    typealias Parent = AnonymousObservable<Element>

    // state
    private let isStopped = AtomicInt(0)

    override init(observer: Observer, cancel: Cancelable) {
        super.init(observer: observer, cancel: cancel)
    }

    func on(_ event: Event<Element>) {
        switch event {
        case .next:
            if load(self.isStopped) == 1 {
                return
            }
            self.forwardOn(event)
        case .error, .completed:
            if fetchOr(self.isStopped, 1) == 0 {
                self.forwardOn(event)
                self.dispose()
            }
        }
    }

    func run(_ parent: Parent) -> Disposable {
        parent.subscribeHandler(AnyObserver(self))
    }
}
```

### SinkDisposer
A type responsible for disposing of both the sink & the subscription.
```swift
private final class SinkDisposer: Cancelable {
    private enum DisposeState: Int32 {
        case disposed = 1
        case sinkAndSubscriptionSet = 2
    }

    // The SinkDisposer can be in one of the following states:
    // * Initial - Sink & Subscription have not been set
    // * SinkAndSubscriptionSet - Sink & Subscription have not been set
    // * Disposed - SinkDisposer has been disposed
    private let state = AtomicInt(0)
    
    // This property keeps a reference to the Sink
    private var sink: Disposable?
    
    // This property keeps a reference to the SinkDisposer associated with
    // the previous subscription
    private var subscription: Disposable?

    var isDisposed: Bool {
        // Check if the sink has been disposed
        isFlagSet(self.state, DisposeState.disposed.rawValue)
    }

    func setSinkAndSubscription(sink: Disposable, subscription: Disposable) {
        // Assign the Sink & Subscription to the SinkDisposer
        self.sink = sink
        self.subscription = subscription

        // Update the current state of the SinkDisposer to SinkAndSubscriptionSet
        let previousState = fetchOr(self.state, DisposeState.sinkAndSubscriptionSet.rawValue)
        
        // The Sink & Subscription are expected to be assigned only once.
        // Thus if the Sink & Subscription have been previously assigned, throw an exception
        if (previousState & DisposeState.sinkAndSubscriptionSet.rawValue) != 0 {
            rxFatalError("Sink and subscription were already set")
        }

        // If the SinkDisposer is already disposed of, also dispose of its Sink & Subscription
        if (previousState & DisposeState.disposed.rawValue) != 0 {
            sink.dispose()
            subscription.dispose()
            self.sink = nil
            self.subscription = nil
        }
    }

    func dispose() {
        // If the SinkDisposer is already disposed of, repeatedly calling
        // this function does nothing
        let previousState = fetchOr(self.state, DisposeState.disposed.rawValue)

        if (previousState & DisposeState.disposed.rawValue) != 0 {
            return
        }

        if (previousState & DisposeState.sinkAndSubscriptionSet.rawValue) != 0 {
            // If either of the following:
            // * Sink
            // * Subscription  
            // is not set when this function is called, an exception is thrown
            guard let sink = self.sink else {
                rxFatalError("Sink not set")
            }
            guard let subscription = self.subscription else {
                rxFatalError("Subscription not set")
            }

            // Otherwise dispose of the Sink & Subscription
            sink.dispose()
            subscription.dispose()

            self.sink = nil
            self.subscription = nil
        }
    }
}
```

## Visual Depiction
### Bird's Eye View

![Object-Graph-Producer-Based-Subscription](https://user-images.githubusercontent.com/5953587/233791659-c5bf6640-9101-4a5c-bfef-db9414b82a4a.jpg)
Image from [8 Mistakes to Avoid while Using RxSwift — Part 1](https://medium.com/@polidea/8-mistakes-to-avoid-while-using-rxswift-part-1-e45b21b47649)

### Satellite View

![Mental-Model-Subscription-Memory-Management](https://user-images.githubusercontent.com/5953587/233791690-343d9727-39d1-42c1-9a23-3f7669ab7af0.jpg)
Image from [8 Mistakes to Avoid while Using RxSwift — Part 1](https://medium.com/@polidea/8-mistakes-to-avoid-while-using-rxswift-part-1-e45b21b47649)
