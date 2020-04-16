# SwiftUI - Presentation
---
## View Basics
### View
A `View` is the fundamental building block for everything visible on screen in SwiftUI. All concrete view types must conform to the `View` protocol (labels, images, controls, stacks, containers, etc) as the minimum requirement.

```Swift
protocol View {
    
    // The body of a view must be a View
    associatedtype Body : View
    var body: Self.Body { get }
}
```

### ViewBuilder
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

### View Modifier
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
---
## Navigation
### TabView
A SwiftUI `TabView` is UIKit's `UITabBarController` counterpart. In UIKit, a `UITabBarController` can hold a number of `UIViewController`s users can switch amongst; the `TabView` is no different.

```Swift
struct TabView<SelectionValue, Content> where SelectionValue : Hashable, Content : View {
    
    // This is typically the initializer we want to call on the TabView, passing in:
    // 1. A binding to the selected tab in the TabView
    // 2. A number of SwiftUI views equivalent to each view controller a UITabBarController
    //    in UIKit
    init(selection: Binding<SelectionValue>?, @ViewBuilder content: () -> Content) {}
}
```

Here's a template for how to layout a `TabView` dynamically.

```Swift
struct ContentView: View {
    
    // Categories shown in AirBnB's Tab Bar
    enum Category : String, CaseIterable, CustomStringConvertible {
        case explore, saved, trips, inbox, profile
    
        var description: String {
            rawValue.uppercased()
        }
        var imageName: String {
            "image-\(rawValue)"
        }
    }
    
    // Selected tab state whose binding is passed into TabView's initializer
    @State private var selectedTab = Category.explore
  
    var body: some View {
        TabView(selection: $selectedTab) {
        
            // Use ForEach to dynamically create the tabs in TabView
            ForEach(Category.allCases, id: \.self) { category in
                
                // DetailView is our custom-defined view corresponding
                // to each user-selected tab
                DetailView(categoryName: category.description)
                    
                    // View modifier constructs what each tab bar should look like
                    .tabItem {
                        Image(systemName: category.imageName).resizable()
                        Text(category.description)
                    }
            }
        }
    }
}
```
### NavigationView
A SwiftUI `NavigationView` can be seen as UIKit's `UINavigationController` equivalent. Recall that every visible component in SwiftUI is a view, and a `NavigationView` is no different - it is a view for managing a stack of views as a visible path in a navigation hierarchy. Let's start with its declaration:

```Swift
struct NavigationView<Content> where Content : View {
    private var content : Content
    init(@ViewBuilder content: () -> Content) {
        self.content = content()
        .
    }
}
```
From the snippet above we see that we need to give the `NavigationView` some content in its initializer - what should the content be? Similar to how we'd give a `UINavigationController` a `UIViewController` instance as its `rootViewController` in UIKit, the content we should give to the `NavigationView` is the same content we would put in the view of our `UINavigationController`'s `rootViewController`. 

Here's a template.
```Swift
// We declare HomeView with a Button titled "Hit Me"
// that performs no action when clicked.
struct HomeView : View {
    var body : some View {
        VStack {
            Button(action: {}) {
                Text("Hit Me")
                    .foregroundColor(.white)
                    .padding()
                    .background(Color.green)
                    .cornerRadius(10)
            }
        }
    }
}

// Here we embed HomeView inside a NavigationView.
// Note the navigation bar title view modifier should apply
// to each view on the navigation stack & not the NavigationView
struct ContentView: View {
    var body: some View {
        NavigationView {
            HomeView()
                .navigationBarTitle("Home")
        }
    }
}
```

### NavigationLink
`NavigationView` took care of the mechanics of the navigation stack - more specfically providing the navigation bar and the back buttons to pop views off the stack. The question that follows is how can we push views onto the navigation stack. In other words, how do we implement UIKit's equivalent of `UINavigationController`'s `pushViewController(_:animated:)` in SwiftUI? The `NavigationLink` serves this exact purpose.

```Swift
struct NavigationLink<Label, Destination> where Label : View, Destination : View {

    // NavigationLink offers various initializer's, but here are the 2 that are
    // most relevant to our discussion:
    // The most basic is providing NavigationLink the following:
    // 1. A destination - the view we'd want to display when the link is tapped
    // 2. A label - the view we'd want to display as the link itself
    init(destination: Destination, @ViewBuilder label: () -> Label) {}
    init(destination: Destination, isActive: Binding<Bool>, @ViewBuilder label: () -> Label) {}
}
```

Building on our template from the __NavigationView__ section, we now want to push a new view onto the navigation stack when the "Hit Me" is tapped.

```Swift
// Here we replaced the Button with a NavigationLink
// while preserving the same UI
// The NavigationLink pushes another HomeView onto the
// navigation stack
struct HomeView : View {
    let id : Int
    var body : some View {
        VStack {
            NavigationLink(destination: HomeView(id: id+1)
                .navigationBarTitle("Level \(id+1)")) {
                    Text("Hit Me")
                        .foregroundColor(.white)
                        .padding()
                        .background(Color.green)
                        .cornerRadius(10)
            }
        }
    }
}

// ContentView remains pretty much the same, except
// HomeView now takes a parameter "id"
struct ContentView: View {
    var body: some View {
        NavigationView {
            HomeView(id: 0)
                .navigationBarTitle("Home")
        }
    }
}
```
