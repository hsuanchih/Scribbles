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
1. The `drawables` array can store elements of different sizes (ie, instances `Point` & `Line`) - very atypical for collections that support random access. How does it manage constant-time memory access with variable memory offsets?
2. `Point` & `Line` each implements its own `draw()` method, so how does the correct `draw()` method get called?

We're going to dissolve these mysteries.

---
## Existential Container

As it turns out, our `drawable` array from the previous section doesn't actually store elements of different sizes. Instead, a protocol type is stored inside a fixed-sized memory container called the existential container. A existential container is an allocation of __5 Machine Words__, the first 3 of which are used as a __value buffer__ to store the contents of the concrete type:

```
  Existential Container                 Point : Drawable
 _______________________             _______________________
||                     ||           || x: 0                ||
||_____________________||           ||_____________________||
||     valueBuffer     ||           || y: 0                ||
||_____________________||           ||_____________________||
||                     ||           ||                     ||
||_____________________||           ||_____________________||
|                       |           |                       |
|_______________________|           |_______________________|
|                       |           |                       |
|_______________________|           |_______________________|

```
Things seem to work out nicely for instances of `Point` - its coordinate (x,y) fits nicely into the value buffer, but what about instances of `Line`? For sure we're going to be needing more than 3 words to accommodate its coordinates. For instances unable to fit the value buffer, we resort to heap allocation and instead use the value buffer to manage its reference:

```
     Line : Drawable                          Heap
 _______________________             _______________________
||                     ||---------->| x1: 0                 |
||_____________________||           |_______________________|
||     valueBuffer     ||           | y1: 0                 |
||_____________________||           |_______________________|
||                     ||           | x2: 3                 |
||_____________________||           |_______________________|
|                       |           | y2: 5                 |
|_______________________|           |_______________________|
|                       |
|_______________________|

```
Now that's quite a bit of added run-time overhead just to store a line (heap allocation & reference counting). Recall that we've allocated 5 words to the existential container - why don't we just make use of the other 2 words and avoid the extra work altogether? These 2 words are dedicated to their own purposes, and we'll look at them next.

---
## Value Witness Table

Regardless of whether the `Drawable` object exists on the stack or the heap, we're going to need a way to manage the life-time of the object. The is done through the value witness table, which takes care of the allocation, memory-copy, tear-down, and destruction of the object. There's one copy of the value witness table per type, and can be referenced through the existential container via its fourth word.

```
     Line : Drawable                   Line : Drawable VWT
 _______________________             _______________________
||                     ||     |---->| allocate:             |
||_____________________||     |     |_______________________|
||     valueBuffer     ||     |     | copy:                 |
||_____________________||     |     |_______________________|
||                     ||     |     | destruct:             |
||_____________________||     |     |_______________________|
|  value witness table  |-----|     | deallocate:           |
|_______________________|           |_______________________|
|                       |
|_______________________|

```

---
## Protocol Witness Table

Finally, to answer how the correct implementation of `draw()` gets called on `Point` & `Line`, we've arrived at protocol witness table - the last word of the existential container. The protocol witness table stores the addresses to the implementation of the contract as declared in a protocol, and is also one copy per type.

```
     Line : Drawable                    Line : Drawable PWT
 ________________________             _______________________
||                      ||     |---->| draw:                 |
||______________________||     |     |_______________________|
||     valueBuffer      ||     |
||______________________||     |
||                      ||     |
||______________________||     | 
|  value witness table  ||     | 
|________________________|     |     
| protocol witness table |-----|
|________________________|

```
That's straightforward enough. But what if `Line` conforms to more than one protocol? Let's add another protocol declaration `GeometricAttribute` with property `length`, and have `Line` conform to the protocol.

```Swift
protocol GeometricAttribute {
    var length : Double { get }
}

protocol Drawable {
    func draw()
}

struct Line {
    let x1, y1, x2, y2 : Double
}

extension Line : Drawable {
    func draw() {
        // Draw a line
    }
}

extension Line : GeometricAttribute {
    var length: Double {
        return sqrt(pow(x2-x1, 2)+pow(y2-y1, 2))
    }
}

let line : GeometricAttribute = Line(x1: 0, y1: 0, x2: 3, y2: 5)
```

What would the protocol witness table look like now?

```
Line : 
Drawable, GeometricAttribute                    
 ________________________             _______________________
||                      ||     |---->| draw:                 |  Line : Drawable PWT
||______________________||     |     |_______________________|
||     valueBuffer      ||     |     | length:               |  Line : GeometricAttribute
||______________________||     |     |_______________________|
||                      ||     |
||______________________||     | 
|  value witness table  ||     | 
|________________________|     |     
| protocol witness table |-----|
|________________________|

```
We would have 2 protocol witness tables in the lookup. Here `length` is a computed property, and equivalently represented as a method with no parameters.
