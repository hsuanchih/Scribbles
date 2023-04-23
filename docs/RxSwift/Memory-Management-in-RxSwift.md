---
layout: default
title: Memory Management in RxSwift
parent: RxSwift
---

# Memory Management in RxSwift
---

### AnonymousObservable\<Element\>
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


## Visual Depiction
### Bird's Eye View

![Object-Graph-Producer-Based-Subscription](https://user-images.githubusercontent.com/5953587/233791659-c5bf6640-9101-4a5c-bfef-db9414b82a4a.jpg)
Image from [8 Mistakes to Avoid while Using RxSwift — Part 1](https://medium.com/@polidea/8-mistakes-to-avoid-while-using-rxswift-part-1-e45b21b47649)

### Satellite View

![Mental-Model-Subscription-Memory-Management](https://user-images.githubusercontent.com/5953587/233791690-343d9727-39d1-42c1-9a23-3f7669ab7af0.jpg)
Image from [8 Mistakes to Avoid while Using RxSwift — Part 1](https://medium.com/@polidea/8-mistakes-to-avoid-while-using-rxswift-part-1-e45b21b47649)
