---
layout: default
title: Global Actor
parent: Swift Concurrency
---

# Global Actor
---

## What is a Global Actor?

A global actor provides a way to setup a singleton instance of an actor that can be shared within the program.

## The `GlobalActor` Protocol

```Swift
/// A type that represents a globally-unique actor that can be used to isolate
/// various declarations anywhere in the program.
///
/// A type that conforms to the `GlobalActor` protocol and is marked with the
/// the `@globalActor` attribute can be used as a custom attribute. Such types
/// are called global actor types, and can be applied to any declaration to
/// specify that such types are isolated to that global actor type. When using
/// such a declaration from another actor (or from nonisolated code),
/// synchronization is performed through the \c shared actor instance to ensure
/// mutually-exclusive access to the declaration.
public protocol GlobalActor {
    /// The type of the shared actor instance that will be used to provide
    /// mutually-exclusive access to declarations annotated with the given global
    /// actor type.
    associatedtype ActorType: Actor

    /// The shared actor instance that will be used to provide mutually-exclusive
    /// access to declarations annotated with the given global actor type.
    ///
    /// The value of this property must always evaluate to the same actor
    /// instance.
    static var shared: ActorType { get }
}
```

## Implementing a Global Actor

```Swift
// Having a type conform to the GlobalActor protocol is one way
// to implement a global actor
public struct SomeGlobalActor: GlobalActor {
    public actor SomeActor {}
    public static let shared = SomeActor()
}

// Using the @globalActor keyword is another way to implement a global actor
@globalActor
public struct SomeGlobalActor {
    public actor SomeActor {}
    public static let shared = SomeActor()
}
```

## What is the Main Actor

The main actor is a global actor that provides actor isolation through a serial executor which designates the main thread.

## Global Actors at Type-Level

When a type itself is annotated with a global actor, all its methods, properties, and subscripts will be implicitly isolated to that global actor.

```Swift
@MainActor
@dynamicMemberLookup
class SomeType {
    // Properties are implicitly @MainActor
    var property: String?

    // Functions are implicitly @MainActor
    func function() {}

    // Subscripts are implicitly @MainActor
    subscript<Value>(dynamicMember keyPath: KeyPath<SomeType, Value>) -> Value {
        self[keyPath: keyPath]
    }
}
```

## Global Actors at Property-Level

```Swift
@dynamicMemberLookup
class SomeType {
    // Only this property is isolated to the main actor
    @MainActor var property: String?

    func function() {}

    subscript<Value>(dynamicMember keyPath: KeyPath<SomeType, Value>) -> Value {
        self[keyPath: keyPath]
    }
}
```

## Global Actors at Function-Level

```Swift
@dynamicMemberLookup
class SomeType {
    var property: String?

    // Only this function is isolated to the main actor
    @MainActor func function() {}

    subscript<Value>(dynamicMember keyPath: KeyPath<SomeType, Value>) -> Value {
        self[keyPath: keyPath]
    }
}
```

## Global Actors with Closures

```Swift
// Using global actor in closure declaration with explicit typing
let explicitlyTypeClosure: @MainActor () -> Void = {
    // Closure body
}

// Using global actor in closure declaration with implicit typing
let implicitlyTypeClosure = { @MainActor in
    // Closure body
}
```