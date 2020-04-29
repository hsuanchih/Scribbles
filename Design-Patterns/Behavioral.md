# Design Patterns - Behavioral
---
<h3><ul>
    <li><a href="#Command">Command</a></li>
    <li><a href="#Chain-of-Responsibility">Chain of Responsibility</a></li>
    <li><a href="#Iterator">Iterator</a></li>
    <li><a href="#Mediator">Mediator</a></li>
    <li><a href="#Memento">Memento</a></li>
    <li><a href="#State">State</a></li>
</ul></h3>

---
## Command

__Command__ encapsulates a request/operation into an object with all resources necessary to perform the request/operation. 
This allows for external mechanics to queue a requests for execution, and can optionally support undo operations. 
The __Command__ design pattern is evident in Cocoa implementations such as [GCD & Operations](https://github.com/hsuanchih/Scribbles/blob/master/Cocoa/Concurrency-GCD-Operations.md) 
and [`NSInvocation`](https://developer.apple.com/documentation/foundation/nsinvocation).

__Example__
```Swift
// The Invoker is responsible for managing & executing Commands
class Invoker {
    private var commands : [Command]
    init(_ commands: Command...) {
        self.commands = commands
    }
    func add(_ command: Command) {
        commands.append(command)
        run()
    }
    
    func run() {
        while !commands.isEmpty {
            commands.removeFirst().execute()
        }
    }
}

// Each Command object provides a execute method to
// let the Invoker execute each command wihtout knowing
// its implementation details
protocol Command {
    func execute()
}

// A Receiver Type implements the execution logic to be wrapped
// inside a Command. This Receiver Type offers flexibility in that
// it can either offer primitive logic, or more complex logic that
// requires capturing the receiver object.
typealias Receiver = ()->Void

// AnyCommand implements a concrete Command
struct AnyCommand : Command {
    private let receiver : Receiver
    init(_ receiver: @escaping Receiver) {
        self.receiver = receiver
    }
    func execute() {
        receiver()
    }
}


// Use Case:
// Create an Invoker
let invoker = Invoker()
// Add a sequence of logic each wrapped inside a Command object
for i in 0..<10 {
    invoker.add(AnyCommand { print("Running command \(i)") })
}
// Kickoff the invoker & optionally add additional Commands 
invoker.run()
invoker.add(AnyCommand { print("Final command") })


// Console Output:
// Running command 0
// Running command 1
// Running command 2
// Running command 3
// Running command 4
// Running command 5
// Running command 6
// Running command 7
// Running command 8
// Running command 9
// Final command
```
---
## Chain of Responsibility

__Chain of Responsibility__ delegates a single responsibility to a collection of entities that can handle a specific request. Upon receiving a request, an entity can either handle the request if it is within the scope of its responsibility, or pass it along to the next entity in a chain of handlers. The Cocoa [Responder Chain](../Cocoa/Event-Handling-And-Responder-Chain.md) fits this design pattern.

__Example__
```Swift
// The approval chain to approve a business expenditure filed by an employee
// If the approver is authorized to approve the expenditure, approve it
// Otherwise pass the request up the approval chain
protocol ApprovalChain {
    var nextInChain : ApprovalChain? { get set }
    func approveExpenditure(_ expenditure : Double) -> Result<ApprovalStatus, ApprovalError>
}

// The request is rejected if it exceeds a certain amount
enum ApprovalError : Error {
    case amountExceedLimit
}
// If the authorized approver is absent, the approval status enters pending state
// Otherwise the request is either approved or rejected
enum ApprovalStatus {
    case pendingApproval, approved
}

// A line manager can approve expenditures up to $1000.00
// If expenditure exceeds such amount, approval from the director is required
class LineManager : ApprovalChain {
    var nextInChain : ApprovalChain?
    
    func approveExpenditure(_ expenditure: Double) -> Result<ApprovalStatus, ApprovalError> {
        guard expenditure <= 1000 else {
            return nextInChain?.approveExpenditure(expenditure) ?? .success(.pendingApproval)
        }
        return .success(.approved)
    }
}

// A director will approve expenditures up to $12000.00
// If expenditure exceeds such amount, request will be rejected
class Director : ApprovalChain {
    var nextInChain : ApprovalChain?
    
    func approveExpenditure(_ expenditure: Double) -> Result<ApprovalStatus, ApprovalError> {
        guard expenditure <= 12000 else {
            return .failure(.amountExceedLimit)
        }
        return .success(.approved)
    }
}

// Use Case:
// A line manager is the first handler of submitted expenditure request
let lineManager = LineManager()

lineManager.approveExpenditure(100)
// success(ApprovalStatus.approved)

lineManager.approveExpenditure(1001)
// success(ApprovalStatus.pendingApproval)

// Assign a director as the next handler in the responsibility chain
lineManager.nextInChain = Director()

print(lineManager.approveExpenditure(1001))
// success(ApprovalStatus.approved)

print(lineManager.approveExpenditure(12001))
// failure(ApprovalError.amountExceedLimit)
```
---
## Iterator

__Iterator__ lets you traverse elements in a collection without exposing its underlying representation. 
The [IteratorProtocol](https://developer.apple.com/documentation/swift/iteratorprotocol) is typically used for building this design pattern.

__Example__
```Swift
// TreeNode declaration
class TreeNode<Value> {
    var value : Value
    var left, right : TreeNode?
    init(_ value: Value) {
        self.value = value
    }
}

// A BSTIterator traverses a binary search tree in-order
class BSTIterator<Value> {
    private var stack : [Element] = []
    
    init(_ root: Element?) {
        inOrder(root)
    }
    func inOrder(_ node: Element?) {
        var node = node
        while let curr = node {
            stack.append(curr)
            node = curr.left
        }
    }
}

// Conformance to IteratorProtocol
extension BSTIterator : IteratorProtocol {
    typealias Element = TreeNode<Value>
    
    func next() -> Element? {
        guard let next = stack.last else { return nil }
        defer { inOrder(next.right) }
        return stack.removeLast()
    }
}

// Build this binary search tree
//      10
//    /    \
//   5     13
//    \    /
//     8  11
let root = TreeNode(10)
root.left = TreeNode(5)
root.left?.right = TreeNode(8)
root.right = TreeNode(13)
root.right?.left = TreeNode(11)

// Pass the root of the tree into the BST iterator
let iterator = BSTIterator(root)

// Traverse the tree until the end is reached
// Console Output:
// 5
// 8
// 10
// 11
// 13
while let next = iterator.next() {
    print(next.value)
}
```
---
## Mediator

Imagine a scenario whereby objects `A`, `B`, `C`, `D` each needs to communicate with each other. 
Invariably this means that object `A` would need to have reference to objects `B`, `C` & `D`. 
This also means that object `B` would also need to have references to objects `A`, `C` & `D`.
Similar dependencies would apply to objects `C`, `D`. Imagine the chaos introduced by these object inter-dependencies.

The objective is to decouple these dependencies through a third party acting as an intermediary, and restrict communication amongst these objects through a __Mediator__.

__Example__

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
---
## Memento

__Memento__ provides a mechanism for saving snapshots of an object’s state at various points in time in order to be restored at another time it in future.

__Example__
```Swift
// The Originator is responsible for generating snapshots of an object's state
// Each stored snapshot is referred to as a Memento
protocol Originator {
    associatedtype MementoType
    var memento : MementoType { get set }
}

// The CareTaker is responsible for saving snapshots generated by the Originator
// It holds the orignator and implements the saving & restoration of Mementos
protocol CareTaker {
    associatedtype OriginatorType : Originator
    var originator : OriginatorType { get set }
    var mementos : [OriginatorType.MementoType] { get set }
    mutating func save()
    mutating func restore()
}
extension CareTaker {
    mutating func save() {
        mementos.append(originator.memento)
    }
    mutating func restore() {
        guard let _ = mementos.last else { return }
        originator.memento = mementos.removeLast()
    }
}

// Value is the object we want to snapshot
// It implements logic for value updates, as well as snapshot generation
struct Value : Codable {
    public private(set) var lastUpdated : Date
    public private(set) var value : Int {
        didSet { lastUpdated = Date() }
    }
    public mutating func add(_ value: Int) {
        self.value += value
    }
    public mutating func subtract(_ value: Int) {
        self.value -= value
    }
}
extension Value : Originator {
    typealias MementoType = Data
    var memento : MementoType {
        get { try! JSONEncoder().encode(self) }
        set { self = try! JSONDecoder().decode(type(of: self), from: newValue) }
    }
}
extension Value : CustomStringConvertible {
    var description: String {
        "\(value) \(ISO8601DateFormatter().string(from: lastUpdated))"
    }
}

// Adder is a simple adder utility that also acts as the CareTaker
struct Adder : CareTaker {

    // CareTaker protocol conformance
    typealias OriginatorType = Value
    var originator: Value = Value(lastUpdated: Date(), value: 0)
    var mementos : [Data] = []
    
    // Adder functionality
    public var value : Value { originator }
    public mutating func add(_ value: Int) {
        originator.add(value)
    }
    public mutating func subtract(_ value: Int) {
        originator.subtract(value)
    }
}

// Use Case:
// Create an adder, perform a number of addition & subtraction
// save the state of each arithmetic operation at each stage
var adder = Adder()
adder.save()
for i in 1...5 {
    sleep(1)
    adder.add(i)
    adder.save()
    sleep(1)
    adder.subtract(i)
    adder.save()
}

// Rewind each intermediate arithmetic operation performed above
for _ in 1...10 {
    adder.restore()
    print(adder.value)
}

// Console Output:
// 0 2020-04-24T03:23:43Z
// 5 2020-04-24T03:23:42Z
// 0 2020-04-24T03:23:41Z
// 4 2020-04-24T03:23:40Z
// 0 2020-04-24T03:23:39Z
// 3 2020-04-24T03:23:38Z
// 0 2020-04-24T03:23:37Z
// 2 2020-04-24T03:23:36Z
// 0 2020-04-24T03:23:35Z
// 1 2020-04-24T03:23:34Z
```
---
## State

__State__ behaves much like a state machine in that an object's behavior changes depending on its internal state.

__Example__
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
