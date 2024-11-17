---
layout: default
title: SwiftUI - Data Flow
parent: SwiftUI
---

# SwiftUI - Data Flow
---
## @State

A state is a stored value of primitive type representing the single source of truth in SwiftUI. A value wrapped in `@State` is stored data managed by SwiftUI that has the same life time as the view in which it is declared. Unlike imperative programming paradigm with UIKit, UI updates in SwiftUI are driven by state changes. The wrapped value returns direct access to the underlying stored value. The projected value returns a binding to the wrapped value.

```swift
@propertyWrapper 
struct State<Value> {

    // Creates the state with an initial value
    init(initialValue value: Value)

    // Accesses the underlying value wrapped by this wrapper
    // Do not access wrappedValue directly, use the state property instead
    var wrappedValue: Value { get nonmutating set }
    
    // Returns a binding to the underlying value wrapped by this wrapper
    // Use the $ syntax to access projectedValue
    var projectedValue: Binding<Value> { get }
}
```
---
## @Binding

A binding does not store any values itself. Rather, it provides a gateway to read & write value owned by the source of truth (`@State`). Note that the objective of a binding is to provide accessors to the stored value that is declared elsewhere, thus a binding property does not need an initial value.

```swift
@propertyWrapper @dynamicMemberLookup 
struct Binding<Value> {

    // Creates a binding with closures that read and write the binding value
    init(get: @escaping () -> Value, set: @escaping (Value) -> Void)
    
    // Accesses the underlying value referenced by the binding variable
    // Do not access wrappedValue directly, use the binding property instead
    var wrappedValue: Value { get nonmutating set }
    
    // Returns the same binding that can be passed down the view hierarchy
    // Use the $ syntax to access projectedValue
    var projectedValue: Binding<Value> { get }
}
```
---
## @ObservedObject

A observed object is a dependecy on an external data source which conforms to `ObservableObject` protocol. `@ObservedObject` creates a binding to the observable object similar to the binding created between `@Binding` and `@State`.

```swift
@propertyWrapper @frozen 
struct ObservedObject<ObjectType> where ObjectType : ObservableObject {
    
    // Initialize with value of type that conforms to ObservableObject
    init(initialValue: ObjectType)
    
    // Accesses the underlying value referenced by the observed object.
    // Do not access wrappedValue directly, use the observed object property instead
    var wrappedValue: ObjectType
    
    // Returns the same observed object binding that can be passed down the view hierarchy
    // Use the $ syntax to access projectedValue
    var projectedValue: ObservedObject<ObjectType>.Wrapper { get }
}
```
---
## @EnvironmentObject

A environment object provides yet another way to propagate dependecy on an external data source which conforms to `ObservableObject` protocol. Instead of propagating the binding through parameter injection, the binding is passed down the view hierarchy by calling [`environmentObject(_:)`](https://developer.apple.com/documentation/swiftui/text/3365512-environmentobject) on the ancester view.

```swift
// Pass the observable object down the view hierarchy
contentView.environmentObject(someObservableObject)

// All subviews of contentView can now access someObservableObject with @EnvironmentObject declaration
struct SomeViewDownTheHeirarchy: View {
    @EnvironmentObject var someObservableObject: SomeObservableObject
}
```
---
## ObservableObject
Concrete types that wish to emit an event when its instance changes can conform to the `ObservableObject` protocol.

```swift
protocol ObservableObject : AnyObject {

    // The default publisher for a ObservableObject is a ObservableObjectPublisher
    // and doesn't fail
    associatedtype ObjectWillChangePublisher : Publisher = ObservableObjectPublisher 
        where Self.ObjectWillChangePublisher.Failure == Never
    
    // Returns a concrete publisher for sending value changes
    var objectWillChange: Self.ObjectWillChangePublisher { get }
}
```

An concrete `ObservableObject` type comes with a publisher `objectWillChange` that synthesizes emission of value changes for properties marked `@Published`.

```swift
class Contact: ObservableObject {
    @Published var name: String
    @Published var age: Int

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

// Create a contact "Hsuan-Chih Chuang" and have a sink subscribe to the
// contact's pusblisher
let me = Contact(name: "Hsuan-Chih Chuang", age: 34)
_ = me.objectWillChange
      .sink { _ in print("\(me.age) will change") }

// Update the contact's age to 35
me.age = 35

// Console Output:
// "34 will change"
```

Alternatively, we can customize our own publisher logic if we want to.

```swift
class Contact: ObservableObject {
    @Published var name: String
    var age: Int {
        willSet {
            // Send a value change event only if age is 36
            if newValue == 36 {
                objectWillChange.send()
            }
        }
    }
    
    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }
}

// Create a contact "Hsuan-Chih Chuang" and have a sink subscribe to the
// contact's pusblisher
let me = Contact(name: "Hsuan-Chih Chuang", age: 34)
let cancellable = me.objectWillChange
    .sink { _ in print("\(me.age) will change") }

// Update the contact's age to 35
me.age = 35

// Update the contact's age to 36
me.age = 36

// Console Output:
// "35 will change"
```
