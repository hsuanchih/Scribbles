# References & Values
---
The intent of this entry is to review the differences between reference & value types, and leverage our understanding to optmize the performance of our applications. I reckon it's probably most appropriate, as a warm up to this discussion, to first revisit the fundamentals from [WWDC 2016, Session 416 - Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/).

## Value Types
* Value types are copied rather than shared
* Memory for value types are allocated on the call stack
* Value types come with compiler-generated memberwise initializer
* Value types do not support inheritance

<img src="images/value-type-memorylayout.png" height="270"/>

Image from [Presentation Slides, WWDC 2016, Session 416](https://devstreaming-cdn.apple.com/videos/wwdc/2016/416k7f0xkmz28rvlvwb/416/416_understanding_swift_performance.pdf?dl=1)
```Swift
// Define value type Point
struct Point {
    var x, y : Double
}

let point1 = Point(x: 0, y: 0)
var point2 = point1
point2.x = 5
```


## Reference Types
* Reference types are shared rather than copied
* Memory for reference types are allocated on the heap
* Reference types do not come with compiler-generated memberwise initializer
* Reference types can support inheritance

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
