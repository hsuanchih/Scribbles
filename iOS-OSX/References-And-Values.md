# References & Values
---
The intent of this entry is to review the differences between reference & value types, and leverage our understanding to optmize the performance of our applications. I reckon it's probably most appropriate, as a warm up to this discussion, to first revisit the fundamentals from [WWDC 2016, Session 416 - Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/).

## Value Types
* Value types are copied rather than shared
* Memory for value types are allocated on the call stack
* Value types come with compiler-generated memberwise initializer
* Value types do not support inheritance

## Reference Types
* Reference types are shared rather than copied
* Memory for reference types are allocated on the heap
* Reference types do not come with compiler-generated memberwise initializer
* Reference types can support inheritance