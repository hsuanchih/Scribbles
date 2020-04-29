# Design Patterns - Creational

<ul>
<li><h2><a href="Builder.md/##Builder">Builder</a></h2></li>
<li><h2><a href="Builder.md/##Prototype">Prototype</a></h2></li>
</ul>

---
## Builder

__Builder__ is a creational mechanism that allows for construction of complex objects step by step. You can create different representations of an object via the same construction process.

__Example__
```Swift
// Vehicle defines the fundamental attributes of a vehicle
protocol Vehicle : class {
    var name : String { get set }
    var color : Color { get set }
    var numberOfWheels : Int { get set }
    var numberOfDoors : Int { get set }
    var numberOfWings : Int { get set }
}

// VehicleBuilder allows you to build different kinds of vehicles 
protocol VehicleBuilder : Vehicle {
    func name(_ name: String) -> Self
    func color(_ color: Color) -> Self
    func numberOfWheels(_ numberOfWheels: Int) -> Self
    func numberOfDoors(_ numberOfDoors: Int) -> Self
    func numberOfWings(_ numberOfWinfs: Int) -> Self
}
extension VehicleBuilder {
    func name(_ name: String) -> Self {
        self.name = name
        return self
    }
    func color(_ color: Color) -> Self {
        self.color = color
        return self
    }
    func numberOfWheels(_ numberOfWheels: Int) -> Self {
        self.numberOfWheels = numberOfWheels
        return self
    }
    func numberOfDoors(_ numberOfDoors: Int) -> Self {
        self.numberOfDoors = numberOfDoors
        return self
    }
    func numberOfWings(_ numberOfWings: Int) -> Self {
        self.numberOfWings = numberOfWings
        return self
    }
}

// AnyVehicle is a concrete VehicleBuilder
class AnyVehicle : VehicleBuilder {
    var name : String = ""
    var color : Color = .clear
    var numberOfWheels : Int = 0
    var numberOfDoors : Int = 0
    var numberOfWings : Int = 0
}

// Build a green bicycle
let greenBicycle = AnyVehicle()
    .name("Bicycle")
    .color(.green)
    .numberOfWheels(2)
    .numberOfDoors(0)
    .numberOfWings(0)

// Build a red car
let redCar = AnyVehicle()
    .name("Car")
    .color(.red)
    .numberOfWheels(4)
    .numberOfDoors(4)
    .numberOfWings(0)

// Build a blue plane
let bluePlane = AnyVehicle()
    .name("Plane")
    .color(.blue)
    .numberOfWheels(20)
    .numberOfDoors(8)
    .numberOfWings(2)
```
---
## Prototype

__Prototype__ allows a copy of a complex object to be created in its existing state without introducing additional dependencies.
The [`NSCopying`](https://developer.apple.com/documentation/foundation/nscopying) protocol exists to serve this exact purpose.

#### Example
```Swift
// Prototype extends the intention of NSCopying by casting the re-created copy
// from Any to a specific type
protocol Prototype : NSCopying {
    func clone() -> Self
}
extension Prototype {
    func clone() -> Self {
        copy() as! Self
    }
}

// TicTacToe simulates a 3x3 game of tic-tac-toe with 2 players
class TicTacToe {
    enum Move : Int, CaseIterable, CustomStringConvertible {
        case x, o
        var next : Self {
            Self(rawValue: ((rawValue+1)%Self.allCases.count))!
        }
        var description: String {
            switch self {
            case .x: return "x"
            case .o: return "o"
            }
        }
    }
    enum Value : CustomStringConvertible {
        case none, some(Move)
        var description: String {
            switch self {
            case .none: return "."
            case let .some(move): return move.description
            }
        }
    }
    var board : [[Value]] = Array(repeating: Array(repeating: .none, count: 3), count: 3)
    var currentMove : Move = .x
    func play(_ row: Int, _ col: Int) -> Bool {
        switch (row, col) {
        case (0..<board.count, 0..<board.first!.count):
            if case .none = board[row][col] {
                board[row][col] = .some(currentMove)
                currentMove = currentMove.next
                return true
            }
            fallthrough
        default:
            return false
        }
    }
}

// Have TicTacToe conform to Prototype & define a convenience
// initializer to replicate the current game
extension TicTacToe : Prototype {
    convenience init(_ board: [[Value]], _ currentMove: Move) {
        self.init()
        self.board = board
        self.currentMove = currentMove
    }
    func copy(with zone: NSZone? = nil) -> Any {
        return TicTacToe(board, currentMove)
    }
}


// Use Case:
// Create a game of tic-tac-toe
let game = TicTacToe()

// Make a few moves
_ = game.play(0, 2)
_ = game.play(1, 1)
_ = game.play(2, 2)
print("Original Game:\n\(game.board)")

// Clone the original game
let clonedGame = game.clone()

// And continue play...
clonedGame.play(0, 1)
print("Cloned Game:\n\(clonedGame.board)")

// Console Output:
Original Game:
[[., ., x], [., o, .], [., ., x]]
Cloned Game:
[[., o, x], [., o, .], [., ., x]]
```
