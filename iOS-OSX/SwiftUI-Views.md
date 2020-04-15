# SwiftUI - Views
---
## View
A `View` is the fundamental building block for everything visible on screen in SwiftUI. All concrete view types must conform to the `View` protocol (labels, images, controls, containers, etc) as the minimum requirement.

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
