# Command
---

__Command__ encapsulates a request/operation into an object with all resources necessary to perform the request/operation. 
This allows for external mechanics to queue a requests for execution, and can optionally support undo operations. 
The __Command__ design pattern is evident in Cocoa implementations such as [GCD & Operations](https://github.com/hsuanchih/Scribbles/blob/master/Cocoa/Concurrency-GCD-Operations.md) 
and [`NSInvocation`](https://developer.apple.com/documentation/foundation/nsinvocation).

---
## Example
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
```

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
