# SwiftUI - Data Flow

---
## ObservableObject
Concrete types that wish to emit an event when its instance changes can conform to the `ObservableObject` protocol.

```Swift
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

```Swift
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

```Swift
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
