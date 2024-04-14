---
layout: default
title: Actor
parent: Swift Concurrency
---

# Actor
---

## What is an Actor?

An Actor provides mechanism for reference types to eliminate data race by enforcing synchronized access to its states. An Actor does so by:
* Isolating its state from the rest of the program
* Ensuring mutually-exclusive access to its state

## The `Actor` Protocol

The `Actor` protocol requires that the conforming type provides a serial executor to facilitate serialized access to its mutable states:

```Swift
public protocol Actor: AnyActor {
    /// Retrieve the executor for this actor as an optimized, unowned
    /// reference.
    ///
    /// This property must always evaluate to the same executor for a
    /// given actor instance, and holding on to the actor must keep the
    /// executor alive.
    ///
    /// This property will be implicitly accessed when work needs to be
    /// scheduled onto this actor.  These accesses may be merged,
    /// eliminated, and rearranged with other work, and they may even
    /// be introduced when not strictly required.  Visible side effects
    /// are therefore strongly discouraged within this property.
    nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}

public protocol AnyActor: AnyObject, Sendable
```

## Making a Mutable Reference Type Sendable using Actors

```Swift
// Declaring a type as actor automatically enforces access to its mutable state
// through its serial executor
actor Notebook {
    var content: String = ""
}

// This route will likely be available in the future
final class Notebook: Actor {
    // When custom serial executor designation is made available,
    // access to this mutable state can be enforced to go through this serial executor
    nonisolated var unownedExecutor: UnownedSerialExecutor
    private var content: String = ""
}
```

## Actor Isolation

* All immutable states (`let` properties) on an actor are not isolated to the actor.
* All mutable states, computed properties (`var` properties) and functions are regarded as actor-isolated by default - regardless of whether these properties & functions access a mutable state.

```Swift
actor Notebook {
    // This is an immutable state, so it is not isolated to the actor
    let id: String = UUID().uuidString

    // This is a mutable state, therefore it is actor-isolated
    var content: String = ""

    // This function is regarded as actor-isolated by default,
    // regardless of whether it accesses a mutable state or not
    func doSomething() {
        // Do something here
    }
}
```

Variables & functions that do not have to be actor-isolated can communicate non-isolation explicitly using the `non-isolated` keyword.

__Conforming to `Hashable`:__

```Swift
extension Notebook: Hashable {
    static func == (lhs: Notebook, rhs: Notebook) -> Bool {
        lhs.id == rhs.id
    }

    // This function can be declared non-isolated because it does not
    // reference mutable state.
    nonisolated func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

__Conforming to `CustomStringConvertible`:__

```Swift
extension Notebook: CustomStringConvertible {
    // This function can be declared non-isolated because it does not
    // reference mutable state.
    nonisolated var description: String {
        "Notebook with ID: \(id)"
    }
}
```

## Actor Reentrancy & Race Condition

__Actor Reentrancy__ - Actor isolated functions are reentrant. When an actor-isolated function suspends, reentrancy allows other work to execute on the actor before the original actor-isolated function resumes.

__Race Condition__ - A race condition occurs when the timing or order of events affect the correctness of a piece of code.

__An Example of Actor Reentrancy Causing Race Condition:__

```Swift
// Snippet taken from:
// https://swiftsenpai.com/swift/actor-reentrancy-problem/
actor BankAccount {
    
    private var balance = 1000
    
    func withdraw(_ amount: Int) async {
        
        print("🤓 Check balance for withdrawal: \(amount)")
        
        guard canWithdraw(amount) else {
            print("🚫 Not enough balance to withdraw: \(amount)")
            return
        }
        
        // This is a suspension point
        guard await authorizeTransaction() else {
            return
        }
        
        print("✅ Transaction authorized: \(amount)")
        
        balance -= amount
        
        print("💰 Account balance: \(balance)")
    }
    
    private func canWithdraw(_ amount: Int) -> Bool {
        return amount <= balance
    }
    
    private func authorizeTransaction() async -> Bool {
        
        // Wait for 1 second
        try? await Task.sleep(nanoseconds: 1 * 1000000000)
        
        return true
    }
}

let account: BankAccount = BankAccount(balance: 1)

Task.detached {
    await account.withdraw(1)
}

Task.detached {
    await account.withdraw(1)
}

// Output:
// 🤓 Check balance for withdrawal: 1
// 🤓 Check balance for withdrawal: 1
// ✅ Transaction authorized: 1
// 💰 Account balance: 0
// ✅ Transaction authorized: 1
// 💰 Account balance: -1
```