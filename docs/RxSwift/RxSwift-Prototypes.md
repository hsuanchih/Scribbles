---
layout: default
title: RxSwift Prototypes
parent: RxSwift
---

# RxSwift Prototypes
---
## Obervables

### ObservableConvertibleType
Types can conform to this protocol to indicate that it be converted into an `Observable` type.
```swift
/// Type that can be converted to observable sequence (`Observable<Element>`).
public protocol ObservableConvertibleType {
    /// Type of elements in sequence.
    associatedtype Element

    /// Converts `self` to `Observable` sequence.
    ///
    /// - returns: Observable sequence that represents `self`.
    func asObservable() -> Observable<Element>
}
```

### ObservableType
Types can conform to this protocol to indicate that it is an `Observable`.
```swift
/// Represents a push style sequence.
public protocol ObservableType: ObservableConvertibleType {
    /**
    Subscribes `observer` to receive events for this sequence.
    
    ### Grammar
    
    **Next\* (Error | Completed)?**
    
    * sequences can produce zero or more elements so zero or more `Next` events can be sent to `observer`
    * once an `Error` or `Completed` event is sent, the sequence terminates and can't produce any other elements
    
    It is possible that events are sent from different threads, but no two events can be sent concurrently to
    `observer`.
    
    ### Resource Management
    
    When sequence sends `Complete` or `Error` event all internal resources that compute sequence elements
    will be freed.
    
    To cancel production of sequence elements and free resources immediately, call `dispose` on returned
    subscription.
    
    - returns: Subscription for `observer` that can be used to cancel production of sequence elements and free resources.
    */
    func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element
}

extension ObservableType {
    
    /// Default implementation of converting `ObservableType` to `Observable`.
    public func asObservable() -> Observable<Element> {
        // temporary workaround
        //return Observable.create(subscribe: self.subscribe)
        Observable.create { o in self.subscribe(o) }
    }
}
```

### Observable\<Element\>
A concrete (however, __abstract__) `Observable` type that type-erases the generic type declared in the `ObservableType` protocol.
```swift
/// A type-erased `ObservableType`. 
///
/// It represents a push style sequence.
public class Observable<Element> : ObservableType {
    init() {
#if TRACE_RESOURCES
        _ = Resources.incrementTotal()
#endif
    }
    
    public func subscribe<Observer: ObserverType>(_ observer: Observer) -> Disposable where Observer.Element == Element {
        rxAbstractMethod()
    }
    
    public func asObservable() -> Observable<Element> { self }
    
    deinit {
#if TRACE_RESOURCES
        _ = Resources.decrementTotal()
#endif
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
            // The returned disposable needs to release all references once it was disposed.
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

private final class SinkDisposer: Cancelable {
    private enum DisposeState: Int32 {
        case disposed = 1
        case sinkAndSubscriptionSet = 2
    }

    private let state = AtomicInt(0)
    private var sink: Disposable?
    private var subscription: Disposable?

    var isDisposed: Bool {
        isFlagSet(self.state, DisposeState.disposed.rawValue)
    }

    func setSinkAndSubscription(sink: Disposable, subscription: Disposable) {
        self.sink = sink
        self.subscription = subscription

        let previousState = fetchOr(self.state, DisposeState.sinkAndSubscriptionSet.rawValue)
        if (previousState & DisposeState.sinkAndSubscriptionSet.rawValue) != 0 {
            rxFatalError("Sink and subscription were already set")
        }

        if (previousState & DisposeState.disposed.rawValue) != 0 {
            sink.dispose()
            subscription.dispose()
            self.sink = nil
            self.subscription = nil
        }
    }

    func dispose() {
        let previousState = fetchOr(self.state, DisposeState.disposed.rawValue)

        if (previousState & DisposeState.disposed.rawValue) != 0 {
            return
        }

        if (previousState & DisposeState.sinkAndSubscriptionSet.rawValue) != 0 {
            guard let sink = self.sink else {
                rxFatalError("Sink not set")
            }
            guard let subscription = self.subscription else {
                rxFatalError("Subscription not set")
            }

            sink.dispose()
            subscription.dispose()

            self.sink = nil
            self.subscription = nil
        }
    }
}
```
### Sink\<Observer\>
In the logical sense, a `Sink` serves as a bridge between the `Observable` and the `Observer`.

In practice, a `Sink` is used by `RxSwift` to store the logic used to designate the `Producer`, and maintains references to:
* Its observer, and
* A `SinkDisposer`
```swift
class Sink<Observer: ObserverType>: Disposable {
    fileprivate let observer: Observer
    fileprivate let cancel: Cancelable
    private let disposed = AtomicInt(0)

    #if DEBUG
        private let synchronizationTracker = SynchronizationTracker()
    #endif

    init(observer: Observer, cancel: Cancelable) {
#if TRACE_RESOURCES
        _ = Resources.incrementTotal()
#endif
        self.observer = observer
        self.cancel = cancel
    }

    final func forwardOn(_ event: Event<Observer.Element>) {
        #if DEBUG
            self.synchronizationTracker.register(synchronizationErrorMessage: .default)
            defer { self.synchronizationTracker.unregister() }
        #endif
        if isFlagSet(self.disposed, 1) {
            return
        }
        self.observer.on(event)
    }

    final func forwarder() -> SinkForward<Observer> {
        SinkForward(forward: self)
    }

    final var isDisposed: Bool {
        isFlagSet(self.disposed, 1)
    }

    func dispose() {
        fetchOr(self.disposed, 1)
        self.cancel.dispose()
    }

    deinit {
#if TRACE_RESOURCES
       _ =  Resources.decrementTotal()
#endif
    }
}

final class SinkForward<Observer: ObserverType>: ObserverType {
    typealias Element = Observer.Element 

    private let forward: Sink<Observer>

    init(forward: Sink<Observer>) {
        self.forward = forward
    }

    final func on(_ event: Event<Element>) {
        switch event {
        case .next:
            self.forward.observer.on(event)
        case .error, .completed:
            self.forward.observer.on(event)
            self.forward.cancel.dispose()
        }
    }
}
```
### Type Diagram of Observables
An observable in `RxSwift` is either:
* A concrete type that conforms to the `ObservableType` protocol, or
* A subclass of `Observable<Element>`, or
* A subclass of `Producer<Element>`

Operators are `Observable`s types that provide their own distinct `Sink` implementations.

![type-diagram-rxswift](https://user-images.githubusercontent.com/5953587/230755362-b76b940b-6040-499e-a5d1-ba09d5471e2f.jpg)
Image from [8 Mistakes to Avoid while Using RxSwift — Part 1](https://medium.com/@polidea/8-mistakes-to-avoid-while-using-rxswift-part-1-e45b21b47649)

---
## Observers
### ObserverType
Types can conform to this protocol to indicate that it is an `Observer`.
```swift
/// Supports push-style iteration over an observable sequence.
public protocol ObserverType {
    /// The type of elements in sequence that observer can observe.
    associatedtype Element

    /// Notify observer about sequence event.
    ///
    /// - parameter event: Event that occurred.
    func on(_ event: Event<Element>)
}

/// Convenience API extensions to provide alternate next, error, completed events
extension ObserverType {
    
    /// Convenience method equivalent to `on(.next(element: Element))`
    ///
    /// - parameter element: Next element to send to observer(s)
    public func onNext(_ element: Element) {
        self.on(.next(element))
    }
    
    /// Convenience method equivalent to `on(.completed)`
    public func onCompleted() {
        self.on(.completed)
    }
    
    /// Convenience method equivalent to `on(.error(Swift.Error))`
    /// - parameter error: Swift.Error to send to observer(s)
    public func onError(_ error: Swift.Error) {
        self.on(.error(error))
    }
}
```
### AnyObserver\<Element\>
A concrete `Observer` type that type-erases the generic type declared in the `ObserverType` protocol.
```swift
/// A type-erased `ObserverType`.
///
/// Forwards operations to an arbitrary underlying observer with the same `Element` type, hiding the specifics of the underlying observer type.
public struct AnyObserver<Element> : ObserverType {
    /// Anonymous event handler type.
    public typealias EventHandler = (Event<Element>) -> Void

    private let observer: EventHandler

    /// Construct an instance whose `on(event)` calls `eventHandler(event)`
    ///
    /// - parameter eventHandler: Event handler that observes sequences events.
    public init(eventHandler: @escaping EventHandler) {
        self.observer = eventHandler
    }
    
    /// Construct an instance whose `on(event)` calls `observer.on(event)`
    ///
    /// - parameter observer: Observer that receives sequence events.
    public init<Observer: ObserverType>(_ observer: Observer) where Observer.Element == Element {
        self.observer = observer.on
    }
    
    /// Send `event` to this observer.
    ///
    /// - parameter event: Event instance.
    public func on(_ event: Event<Element>) {
        self.observer(event)
    }

    /// Erases type of observer and returns canonical observer.
    ///
    /// - returns: type erased observer.
    public func asObserver() -> AnyObserver<Element> {
        self
    }
}

extension AnyObserver {
    /// Collection of `AnyObserver`s
    typealias s = Bag<(Event<Element>) -> Void>
}

extension ObserverType {
    /// Erases type of observer and returns canonical observer.
    ///
    /// - returns: type erased observer.
    public func asObserver() -> AnyObserver<Element> {
        AnyObserver(self)
    }

    /// Transforms observer of type R to type E using custom transform method.
    /// Each event sent to result observer is transformed and sent to `self`.
    ///
    /// - returns: observer that transforms events.
    public func mapObserver<Result>(_ transform: @escaping (Result) throws -> Element) -> AnyObserver<Result> {
        AnyObserver { e in
            self.on(e.map(transform))
        }
    }
}
```

---
## Disposables
### Disposable
Types can conform to this protocol to indicate that it can be disposed of.
```swift
/// Represents a disposable resource.
public protocol Disposable {
    /// Dispose resource.
    func dispose()
}
```
### Cancelable
Types that can be disposed of & want to stay in-the-know can conform to the `Cancelable` protocol.
```swift
/// Represents disposable resource with state tracking.
public protocol Cancelable : Disposable {
    /// Was resource disposed.
    var isDisposed: Bool { get }
}
```
### DispoeBag
A `DisposeBag` exists for memory management, and manages the life-time of `Disposable` added to the bag.
```swift
extension Disposable {
    /// Adds `self` to `bag`
    ///
    /// - parameter bag: `DisposeBag` to add `self` to.
    public func disposed(by bag: DisposeBag) {
        bag.insert(self)
    }
}

/**
Thread safe bag that disposes added disposables on `deinit`.

This returns ARC (RAII) like resource management to `RxSwift`.

In case contained disposables need to be disposed, just put a different dispose bag
or create a new one in its place.

    self.existingDisposeBag = DisposeBag()

In case explicit disposal is necessary, there is also `CompositeDisposable`.
*/
public final class DisposeBag: DisposeBase {
    
    private var lock = SpinLock()
    
    // state
    private var disposables = [Disposable]()
    private var isDisposed = false
    
    /// Constructs new empty dispose bag.
    public override init() {
        super.init()
    }

    /// Adds `disposable` to be disposed when dispose bag is being deinited.
    ///
    /// - parameter disposable: Disposable to add.
    public func insert(_ disposable: Disposable) {
        self._insert(disposable)?.dispose()
    }
    
    private func _insert(_ disposable: Disposable) -> Disposable? {
        self.lock.performLocked {
            if self.isDisposed {
                return disposable
            }

            self.disposables.append(disposable)

            return nil
        }
    }

    /// This is internal on purpose, take a look at `CompositeDisposable` instead.
    private func dispose() {
        let oldDisposables = self._dispose()

        for disposable in oldDisposables {
            disposable.dispose()
        }
    }

    private func _dispose() -> [Disposable] {
        self.lock.performLocked {
            let disposables = self.disposables
            
            self.disposables.removeAll(keepingCapacity: false)
            self.isDisposed = true
            
            return disposables
        }
    }
    
    deinit {
        self.dispose()
    }
}

extension DisposeBag {
    /// Convenience init allows a list of disposables to be gathered for disposal.
    public convenience init(disposing disposables: Disposable...) {
        self.init()
        self.disposables += disposables
    }

    /// Convenience init which utilizes a function builder to let you pass in a list of
    /// disposables to make a DisposeBag of.
    public convenience init(@DisposableBuilder builder: () -> [Disposable]) {
      self.init(disposing: builder())
    }

    /// Convenience init allows an array of disposables to be gathered for disposal.
    public convenience init(disposing disposables: [Disposable]) {
        self.init()
        self.disposables += disposables
    }

    /// Convenience function allows a list of disposables to be gathered for disposal.
    public func insert(_ disposables: Disposable...) {
        self.insert(disposables)
    }

    /// Convenience function allows a list of disposables to be gathered for disposal.
    public func insert(@DisposableBuilder builder: () -> [Disposable]) {
        self.insert(builder())
    }

    /// Convenience function allows an array of disposables to be gathered for disposal.
    public func insert(_ disposables: [Disposable]) {
        self.lock.performLocked {
            if self.isDisposed {
                disposables.forEach { $0.dispose() }
            } else {
                self.disposables += disposables
            }
        }
    }

    /// A function builder accepting a list of Disposables and returning them as an array.
    #if swift(>=5.4)
    @resultBuilder
    public struct DisposableBuilder {
      public static func buildBlock(_ disposables: Disposable...) -> [Disposable] {
        return disposables
      }
    }
    #else
    @_functionBuilder
    public struct DisposableBuilder {
      public static func buildBlock(_ disposables: Disposable...) -> [Disposable] {
        return disposables
      }
    }
    #endif
    
}
```
