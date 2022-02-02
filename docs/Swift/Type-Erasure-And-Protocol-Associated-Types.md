---
layout: default
title: Type Erasure & Protocol with Associated Types
parent: Swift
---

# Type Erasure & Protocol with Associated Types
---
Generics have always played an important part in languages that support polymorphism. 
They provide a means to code reusability without compromising for type safety. 
We will look at how generic types are converted to specific types at compile time via type erasure, and the limitations for protocols that support generic types in Swift.

---
## Generics
Generics in Swift approximates to generics in other languages syntactically and semantically for the most part. Here are how generics can be used with `Enum`, `Class` & `Struct`.

```swift
enum Optional<Wrapped> {
    case none, some(Wrapped)
}

class TreeNode<Value> {
    var value : Value
    var left, right : TreeNode?
    init(_ value: Value) {
        self.value = value
    }
}

struct Stack<Element> {
    var items = [Element]()
    mutating func push(_ item: Element) {
        items.append(item)
    }
    mutating func pop() -> Element {
        return items.removeLast()
    }
}
```
What about `Protocol`s? `Protocol`s are a bit different - they support generics through associated types:

```swift
protocol Container {
    associatedtype Item
    mutating func append(_ item: Item)
    var count: Int { get }
    subscript(i: Int) -> Item { get }
}
```
---
## Type Erasure
Type erasure, as the name suggests, can be perceived as the process of erasing the generic type with a specific type at compile time. We've gotten the basics of generics out of the way, so let's erase some types using declarations.

```swift
_ = Optional.some(3)
// This declaration erases Optional's generic Wrapped type with Int:
enum Optional {
    case none, some(Int)
}

_ = TreeNode("M")
// This declaration erases TreeNode's generic Value type with String:
class TreeNode {
    var value : String
    var left, right : TreeNode?
    init(_ value: String) {
        self.value = value
    }
}

_ = Stack<TreeNode<Int>>()
// This declaration erases Stack's generic Element type with TreeNode<Int>:
struct Stack {
    var items = [TreeNode<Int>]()
    mutating func push(_ item: TreeNode<Int>) {
        items.append(item)
    }
    mutating func pop() -> TreeNode<Int> {
        return items.removeLast()
    }
}
```
Pretty straightforward so far. Let's also try to type erase a `Protocol`'s generic type with a declaration:

```swift
// This declaration doesn't erase anything, so it must not be correct.
// And the compiler tells us what the problem is with error: 
// Protocol 'Container' can only be used as a generic constraint 
// because it has Self or associated type requirements
let container : Container?

// This one attempts to erase something, but the syntax smells funky
// and it must not be correct either.
// And the compiler tells us what the problem is with error:
// Cannot specialize non-generic type 'Container'
let container : Container<Int>?
```
Soon enough we find out that there's no simple way to do it. This is because protocol associated type substitutions only happen at conformance, not declaration. Let's start down that route - we'll build a concrete type that conforms to the `Container` protocol and erase its associated type:

```swift
// We implement a concrete type StringContainer conforming to the Container protocol
struct StringContainer : Container {

    // Erase Container's associated type with String
    typealias Item = String
    
    // Use an array internally as a container
    private var container : [Item] = []
    
    // These are the Container protocol contract
    mutating func append(_ item: Item) {
        container.append(item)
    }
    var count: Int {
        container.count
    }
    subscript(i: Int) -> Item {
        container[i]
    }
}
```
We've built a `StringContainer` and used it to erase the generic `Item` type. This is one step forward, but now there's nothing generic about the `StringContainer` type - the container can only store `String`s. And if we wanted a container that stores `Int`s, we'll have to implement a `IntContainer` and re-implement the same logic. This defeats the purpose of having generics in the first place. Can we do better?

---
## Type-Erasing Concrete Types
In the end what we want is something that not only supports generics, but can also have its generic types erased at declaration. Let's borrow from an idea we sumbled upon earlier:

```swift
// This one attempts to erase something, but the syntax smells funky
// and it must not be correct either.
// And the compiler tells us what the problem is with error:
// Cannot specialize non-generic type 'Container'
let container : Container<Int>?
```
This syntax is obviously incorrect, but it would be perfect if we can somehow end up with something that does exactly this snippet intends to do. So let's give it a try.

```swift
struct AnyContainer<Item> : Container {
    
    // Erase Container's associated type with AnyContainer's Item type
    typealias Item = Item
    private var container : [Item] = []
    
    // These are the Container protocol contract
    mutating func append(_ item: Item) {
        container.append(item)
    }
    var count: Int {
        container.count
    }
    subscript(i: Int) -> Item {
        container[i]
    }
}
```
So here it is. A concrete type `AnyContainer` that erases `Container` protocol's associated type with its generic type, and in turn allows us to type erase its generic type with a specific type. What we see below is now a valid declaration.

```swift
_ = AnyContainer<String>()
```
These kinds of type-erasing types are quite common in Swift. Many types prefixed with `Any` serve this specific purpose - [`AnySequence`](https://developer.apple.com/documentation/swift/anysequence), [`AnyCollection`](https://developer.apple.com/documentation/swift/anycollection),[`AnyIterator`](https://developer.apple.com/documentation/swift/anyiterator) and others.
