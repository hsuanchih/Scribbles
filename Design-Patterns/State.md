# State
---

__State__ behaves much like a state machine in that an object's behavior changes depending on its internal state.

---
## Example
```Swift
// Runnable protocol representing an object's behavior
protocol Runnable {
    func run()
}

// Routine performs a routine depending on the time of day
enum Routine : Int, CaseIterable {
    case morning, noon, afternoon, evening, midnight, reset
    
    mutating func next() {
        guard self != .reset else { return }
        self = Self(rawValue: (rawValue+1))!
    }
}

// Runnable protocol conformance
extension Routine : Runnable {
    func run() {
        switch self {
        case .morning:
            print("Wake up")
        case .noon:
            print("Eat lunch")
        case .afternoon:
            print("Work")
        case .evening:
            print("Exercise")
        case .midnight:
            print("Go to bed")
        case .reset:
            break
        }
    }
}

// Start a routine at midnight, and observe
// its behavior throughout the day
var routine = Routine.morning
repeat {
    routine.run()
    routine.next()
} while routine != .reset
```
