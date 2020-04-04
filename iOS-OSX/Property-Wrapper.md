# Property Wrapper
---
Propert wrapper is a new Swift feature that helps power a part of SwiftUI & Combine. 
We're going to see how it works, and also look at some of its use cases. 

---
## What is a Property Wrapper?

A property wrapper is literally as the name reads - a wrapper for a property. But why would we ever want to have a wrapper 
around a property you might ask? It's because a lot of the times we either want to process the data or run validation on it 
before writing it back to the property. Other times we might want to abstract away data access from how that data is actually 
stored. Property wrapper allows us to do that.

---
## Property Wrapper Prerequisites

A property wrapper is represented with a type declaration (can be a `class`, `struct`, or `enum`) prefixed with `@propertyWrapper`. Its contract includes a mandatory `wrappedValue` implementation, and an optional `projectedValue` imeplementation. 

`wrappedValue`, as the name suggests, represents the underlying value wrapped by the property wrapper, and can be implemented either as a computed property (this is typically the case) or a stored property (and we'll see an example of why this can be the case). 

`projectedValue` was originally named `wrapperValue`, suggesting its historical intention to project the wrapper itself rather than the value it wraps, and can be accessed using the `$` prefix.

Memberwise initializer `init(wrappedValue:)` comes out-of-the-box with property wrappers declared as `struct`s, and is otherwise left to the developer.

---
## Implementing a Property Wrapper

Let's start with a simple property wrapper to see how everything comes together. We declare a property wrapper called `StringWrapper`. Currently it doesn't do much apart from being able to store a `String` to `wrappedValue`.

```Swift
// StringWrapper declaration
@propertyWrapper struct StringWrapper {
    var wrappedValue : String
}
```

With `StringWrapper`, we can now declare a property like so:

```Swift
@StringWrapper var string : String = "some string"
```

When we make the above declaration, the compiler synthesizes our declaration into the follwing:
* A private stored property of type `StringWrapper` named `_string`, initialized with value `"some string"`
* A public computed property `string`, with accessors referencing `_string`'s `wrappedValue`

```Swift 
private var _string: StringWrapper = StringWrapper(wrappedValue: "some string")

public var string: String {
    get { _string.wrappedValue }
    set { _string.wrappedValue = newValue }
}
```

We now know what a property wrapper is under the hood. For sake of completeness, let's also fulfill the optional contract and add `projectedValue` implementation to `StringWrapper` and see what happens.

```Swift
@propertyWrapper struct StringWrapper {
    var wrappedValue : String
    public var projectedValue: Self {
        get { self }
        set { self = newValue }
    }
}
```

Our `StringWrapper` declaration now translates to:

```Swift
private var _string: StringWrapper = StringWrapper(wrappedValue: "some string")

public var string: String {
    get { _string.wrappedValue }
    set { _string.wrappedValue = newValue }
}

public var $string: StringWrapper {
    get { _string.projectedValue }
    set { _string.projectedValue = newValue }
}
```

There. We've not only implemented our first property wrapper, but also gained a bit of understanding as to how property wrappers work. The property wrapper we've put together here isn't very useful though, as it simply serves as an indirect storage for the `string` property. This is certainly not what property wrappers are meant for. So in the next section we'll be looking at a more practical use case for property wrappers.
