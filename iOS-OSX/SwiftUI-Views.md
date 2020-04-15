# SwiftUI - Views
---
## View
A `View` is the fundamental building block for everything visible on screen in SwiftUI. All concrete view types must conform to the `View` protocol (labels, images, controls, stacks, containers, etc) as the minimum requirement.

```Swift
protocol View {
    
    // The body of a view must conform to View
    associatedtype Body : View
    var body: Self.Body { get }
}
```

---
## ViewBuilder
A `ViewBuilder` is capable of constructing the contents of a composite view based on individual views declared in a closure.
Concrete `View` types that can be formed using a composition of one or more subviews use a `ViewBuilder` to build its content.
Take `HStack` for example, we declare views we want to layout in the `HStack` in the closure parameter we pass into its initializer - 
and the `HStack` uses a `ViewBuilder` to construct its content. 

```Swift
// ContentView 
struct ContentView: View {
    var body: some View {
        // Layout a image, followed by some text in a HStack
        HStack {
            Image(systemName: "fruit-image")
            Text("Fruits")
        }
    }
}

// HStack
struct HStack<Content> where Content : View {
    
    private var content: Content
    
    // This is the initializer we call on the HStack to populate ContentView's body
    init(alignment: VerticalAlignment = .center, 
          spacing: CGFloat? = nil, 
          @ViewBuilder content: () -> Content) {
        
        // HStack uses a ViewBuilder here to construct its content
        self.content = content()
        .
        .
    }
    
    // Return output of the ViewBuilder as the view's body
    var body : some View {
        content
    } 
}
```
---
## View Modifier
View modifiers are used to apply configurations a view and its subviews. Modifiers available to a view vary depending on attributes pertinent to that specific type of view. Here's an example of view modifiers in action.

```Swift
struct ContentView: View {
    @Binding var value: Double
    var textColor: Color
    var body: some View {
    
        // Create a slider initialized with a value binding
        Slider(value: $value)
            // Apply a view modifier on the slider to set the background color green
            .background(Color.green)
            // Apply another view modifer on the slider to set the corner radius to 10 points
            .cornerRadius(10)
    }
}
```

Here's are the methods we call on the `Slider` above to add view modifiers:

```Swift
// Modifier to assign a background to the view
@inlinable public func background<Background>(
    _ background: Background, 
    alignment: Alignment = .center) -> some View where Background : View {
}

// Modifier to assign corner radius
@inlinable public func cornerRadius(
    _ radius: CGFloat, 
    antialiased: Bool = true) -> some View {
}
```

It's important to note that views in SwiftUI are value types, and even though view modifiers seem to resemble builder patterns we've all, at some point in time, declared on our types, there is a nuance to be pointed out here - modifiers do not modify the view directly, but creates a new view wrapping the original view inside it. This also means that the order of applying modifiers is not commutative. Using the `Slider` example:

```Swift
// This slider will have green background & rounded corners
Slider(value: $value)
    .background(Color.green)
    .cornerRadius(10)

// This slider will have green background, but will not have rounded corners
Slider(value: $value)
    .cornerRadius(10)
    .background(Color.green)
```
