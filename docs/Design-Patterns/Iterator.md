# Iterator
---

__Iterator__ lets you traverse elements in a collection without exposing its underlying representation. 
The [IteratorProtocol](https://developer.apple.com/documentation/swift/iteratorprotocol) is typically used for building this design pattern.

---
## Example

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
