# Property Wrapper
---
Propert wrapper is a new Swift feature that helps power a part of SwiftUI & Combine. 
We're going to see how it works, and also look at some of its use cases. 

---
## What is a Property Wrapper?
A property wrapper is literally as the name reads - a wrapper for a property. But why would we ever want to have a wrapper 
around a property you might ask? It's because a lot of the times we either want to process the data or run validation on it 
before writing it back to the property. Other times we might want to abstract away data access from how that data is actually 
stored. Property wrapper allows us to do that.
