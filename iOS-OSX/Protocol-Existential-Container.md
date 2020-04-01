# Protocol Semantics & Existential Container
---
We've gained a better understanding of how reference & value types are managed by the application in Reference & Value Semantics in Swift. 
Here we're going to wrap up the discussion with a look into protocol types.

---
## Protocol Types
It makes sense to kick-off a discussion on procotol types with a protocol declaration. So let's start with a protocol `Drawable`:
```Swift
// Declare a Drawable protocol: 
// Any concrete type that can draw things can conform to this protocol
// and implement the draw() method
protocol Drawable {
    func draw()
}
```
We now have our `Drawable` protocol, but a protocol isn't of much use on its own, so let's have a few value types conform to this protocol:
```Swift
// A Point has coordinate (x,y), and can draw
struct Point : Drawable {
    let x, y : Double
    func draw() {
        // Draw a point
    }
}
// A Line is connected by 2 points with coordinates (x1,y1) & (x2,y2), and can also draw
struct Line : Drawable {
    let x1, y1, x2, y2 : Double
    func draw() {
        // Draw a line
    }
}
```
Suppose we now initialize an array `drawables` with elements of type `Drawable`, add a point & line to the array, and then call `draw()` on each element:
```Swift
var drawables : [Drawable] = []
drawables.append(Point(x: 0,y: 0))
drawables.append(Line(x1: 0, y1: 0, x2: 3, y2: 5))
drawables.forEach { $0.draw() }
```
The code above brings about 2 interesting questions:
1. The `drawables` array can store elements of different sizes (ie, instances `Point` & `Line`) - very atypical design for collections that support random access. 
How does it manage constant-time memory access with variable memory offsets?
2. `Point` & `Line` each implements its own `draw()` method, so how does the correct `draw()` method get called?

We're going to dissolve these mysteries.

---
## Existential Container
