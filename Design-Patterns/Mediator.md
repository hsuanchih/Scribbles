# Mediator
---

Imagine a scenario whereby objects `A`, `B`, `C`, `D` each needs to communicate with each other. 
Invariably this means that object `A` would need to have reference to objects `B`, `C` & `D`. 
This also means that object `B` would also need to have references to objects `A`, `C` & `D`.
Similar dependencies would apply to objects `C`, `D`. Imagine the chaos introduced by these object inter-dependencies.

The objective is to decouple these dependencies through a third party acting as an intermediary, and restrict communication amongst these objects through a __Mediator__.

---
## Example

```Swift
protocol Identifiable {
    var id : String { get }
}

// Participant protocol defines the contract for each participant in the mediation
protocol Participant : class, Identifiable {
    var mediator : Mediator? { get }
    func send(message: String)
    func receive(message: String, from participant: Participant)
}
extension Participant {
    func send(message: String) {
        mediator?.receive(message: message, from: self)
    }
}

// Mediator protcol defines the contract for a mediator involved in the mediation
protocol Mediator : class {
    var participants : [Participant] { get set }
    func add(_ participant: Participant)
    func remove(_ participant: Participant)
    func receive(message: String, from participant: Participant)
}
extension Mediator {
    func add(_ participant: Participant) {
        guard !(participants.contains { $0 === participant }) else { return }
        participants.append(participant)
    }
    func remove(_ participant: Participant) {
        if let index = (participants.lastIndex { $0 === participant }) {
            participants.swapAt(index, participants.count-1)
            participants.removeLast()
        }
    }
    func receive(message: String, from participant: Participant) {
        participants.forEach {
            if $0 !== participant {
                $0.receive(message: message, from: participant)
            }
        }
    }
}

// Concrete Mediator type
class SomeMediator : Mediator {
    var participants: [Participant] = []
}

// Concrete Participant type
class SomeParticipant : Participant {
    let id: String
    weak var mediator: Mediator?
    func receive(message: String, from participant: Participant) {
        print("\(id) received message \(message) from \(participant.id)")
    }
    init(id: String, mediator: Mediator? = nil) {
        self.id = id
        self.mediator = mediator
        mediator?.add(self)
    }
    deinit {
        mediator?.remove(self)
    }
}

// Use Case:
// Create a mediator
let mediator = SomeMediator()

// Create participants A, B, C, & D
let participants = ["A", "B", "C", "D"].map { SomeParticipant(id: $0, mediator: mediator) }

// Each participant send a message to every othere participant via the mediator
participants.forEach { $0.send(message: "'I am \($0.id)'") }

// Console Output:
// B received message 'I am A' from A
// C received message 'I am A' from A
// D received message 'I am A' from A
// A received message 'I am B' from B
// C received message 'I am B' from B
// D received message 'I am B' from B
// A received message 'I am C' from C
// B received message 'I am C' from C
// D received message 'I am C' from C
// A received message 'I am D' from D
// B received message 'I am D' from D
// C received message 'I am D' from D
```
