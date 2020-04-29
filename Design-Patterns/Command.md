# Design Patterns - Behavioral

<h3><ul>
    <li><a href="#Command">Command</a></li>
    <li><a href="#Chain-of-Responsibility">Chain of Responsibility</a></li>
    <li><a href="#Iterator">Iterator</a></li>
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
