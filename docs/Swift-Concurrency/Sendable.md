---
layout: default
title: Sendable
parent: Swift Concurrency
---

# Sendable
---

## Data Race

A data race occurs when two or more threads access the same location in memory, and at least one of the access is a write.
Data race will result in non-deterministic behavior, and may result in crashes in the event of bad memory access.

## Sendable

A type is sendable if an instance of this type can be passed through concurrency boundaries without causing data races. Generally speaking, a type is sendable if it is:
* A value type
* An immutable reference type
* A mutable reference type that internally manages access to its state

## The `Sendable` Protocol

The `Sendable` protocol is a marker protocol and does not have requirements of any kind. Its purpose is to indicate to the compiler that instance of types conforming to this protocol is safe to pass across concurrency boundaries.

```Swift
@_marker
protocol Sendable {}
```

## Making a Mutable Reference Type Sendable

Value types & immutable reference types are sendable, but how do we make a mutable reference type sendable so that it can be safely passed across concurrency boundaries?

__Using Locks:__
```Swift
class Notebook {
    private let lock: NSLock = NSLock()
    private var _content: String = ""
    var content: String {
        get {
            defer { lock.unlock() }
            lock.lock()
            return _content
        }
        set {
            defer { lock.unlock() }
            lock.lock()
            _content = newValue
        }
    }
}
```

__Using Semaphores:__
```Swift
class Notebook {
    private let semaphore: DispatchSemaphore = DispatchSemaphore(value: 1)
    private var _content: String = ""
    var content: String {
        get {
            defer { semaphore.signal() }
            semaphore.wait()
            return _content
        }
        set {
            defer { semaphore.signal() }
            semaphore.wait()
            _content = newValue
        }
    }
}
```

__Using Dispatch Queues (Synchronous Dispatch):__
```Swift
class Notebook {
    private let queue: DispatchQueue = DispatchQueue(label: "")
    private var _content: String = ""
    var content: String {
        get {
            queue.sync { _content }
        }
        set {
            queue.sync { _content = newValue }
        }
    }
}
```

__Using Dispatch Queues (Asynchronous Dispatch):__
```Swift
class Notebook {
    private let queue: DispatchQueue = DispatchQueue(label: "")
    private var content: String = ""

    func readContent(completion: @escaping (String) -> Void) {
        queue.async {
            completion(self.content)
        }
    }

    func writeContent(_ content: String, completion: @escaping () -> Void) {
        queue.async {
            self.content = content
            completion()
        }
    }
}
```

## Sendable Functions/Closures

Functions & closures can also be sent across concurrency boundaries without causing data races if objects it captures are sendable - we refer to them as sendable functions/closures.

We can indicate to the compiler that a data type is sendable by having the type conform to the `Sendable` protocol. Functions & closures can also be sendable, but they are not explicit types & cannot conform to the `Sendable` protocol - they rely on the `@Sendable` attribute to communicate sendability.

__Communicating Sendability with Function:__
```Swift
// Anotate a function using the @Sendable attribute to communicate sendability to the compiler
@Sendable func sendableFunction() {
    // Function body
}

// Function with sendable function as parameter
func functionWithSendableFunctionAsParameter(sendableFunction: @Sendable () -> Void) {
    // Function body
}
```

__Communicating Sendability with Closure:__
```Swift
// Sendable closure declaration with explicit typing
let explicitlyTypeClosure: @Sendable () -> Void = {
    // Closure body
}

// Sendable closure declaration with implicit typing
let implicitlyTypeClosure = { @Sendable in
    // Closure body
}
```