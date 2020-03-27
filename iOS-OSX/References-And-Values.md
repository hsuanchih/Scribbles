# References & Values
---
The intent of this entry is to review the differences between reference & value types, and leverage our understanding to optmize the performance of our applications. I reckon it's probably most appropriate, as a warm up to this discussion, to first revisit the fundamentals from [WWDC 2016, Session 416 - Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/).

## Value Types
* Value types are copied rather than shared
* Memory for value types are allocated on the call stack
* Value types come with compiler-generated memberwise initializer
* Value types do not support inheritance

<img src="images/value-type-memorylayout.png" height="260"/>

Image from [Presentation Slides, WWDC 2016, Session 416](https://devstreaming-cdn.apple.com/videos/wwdc/2016/416k7f0xkmz28rvlvwb/416/416_understanding_swift_performance.pdf?dl=1)
```Swift
// Define value type Point
struct Point {
    var x, y : Double
}

// Allocate memory for point1 on the stack & initialize (x,y) = (0,0)
let point1 = Point(x: 0, y: 0)

// Allocate memory for point2 on the stack & initialize (x,y) = (point1.x, point1.y)
var point2 = point1

// point1.x is 0, point2.x is 5
point2.x = 5
```


## Reference Types
* Reference types are shared rather than copied
* Memory for reference types are allocated on the heap
* Reference types do not come with compiler-generated memberwise initializer
* Reference types can support inheritance

<img src="images/reference-type-memorylayout.png" height="260"/>

Image from [Presentation Slides, WWDC 2016, Session 416](https://devstreaming-cdn.apple.com/videos/wwdc/2016/416k7f0xkmz28rvlvwb/416/416_understanding_swift_performance.pdf?dl=1)
```Swift
// Define reference type Point
class Point {
    var x, y : Double

    // Reference types don't come with memberwise initializer
    // We need to explicitly declare one
    init(x: Double, y: Double) {
        (self.x, self.y) = (x,y)
    }
}

// 1. Allocate memory for instance of Point on the heap with (x,y) = (0,0)
// 2. Allocate memory for point1 on the stack storing address to the Point object on the heap
let point1 = Point(x: 0, y: 0)

// Allocate memory for point2 on the stack storing address to the Point object on the heap
var point2 = point1

// point2.x is 5, and point1.x is now also 5
point2.x = 5
```
Looking at the image above, it appears that the memory block allocated on the heap is more memory than is necessary to accommodate variables x & y. This is because an excess of memory is required to manage reference types.</br> 
To name a subset here: 
* __`isa` pointer__ - Reference types support inheritance, thus we need a way to walk the inheritance hierarchy in dynamic binding
* __Reference counter__ - Heap memory needs to be recycled when it's no longer in use, thus tracking active references to the memory is important

Below is a snippet simulating (simplified) compiler generated logic from the code above to illustrate what's really going on behind the scenes:
```Swift
// Retain/release imeplementations
func retain<T: ReferenceCounting>(_ object: T) {
    object.refCount+=1
}
func release<T: ReferenceCounting>(_ object: T) {
    object.refCount-=1
}

// All reference types implicitly support reference counting
protocol ReferenceCounting : class {
    var refCount : Int { get set }
}
extension Point : ReferenceCounting {}

class Point {

    // Reference types have memory allocated for reference counting
    var refCount : Int
    
    var x, y : Double
    init(x: Double, y: Double) {
        refCount = 1
        (self.x, self.y) = (x,y)
    }
}
let point1 = Point(x: 0, y: 0)
var point2 = point1

// refCount of the Point object is incremented when point2 makes reference to it
retain(point2)
point2.x = 5

// refCount of the Point object is decremented when point1 is popped off the call stack, 
// and decremented again when point2 is popped off the stack.
// Now refCount of the Point object is now 0, and can be safely recycled.
release(point1)
release(point2)
```

## Performance Metrics
### Is memory allocated on stack or heap?
* Stack - memory allocation is fast
    * Decrement & increment the stak pointer to allocate & deallocate memory.
* Heap - memory allocation is slow
    * Need to search for contiguous blocks of memory of sufficient size, potentially de-fragmenting memory blocks to satisfy the allocation
    * House-keeping post deallocation
    * Thread-safety overhead (locking mechanism) with heap access

### How much reference counting overhead is incurred at run-time?

### What portion of method calls on instances are dynamically dispatched?
