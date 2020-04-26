# Prototype
---
__Prototype__ allows a copy of a complex object to be created in its existing state without introducing additional dependencies.
The `NSCopying` protocol exists to serve this exact purpose.

---
## Example

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
