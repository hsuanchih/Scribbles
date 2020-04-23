# Chain of Responsibility
---

__Chain of Responsibility__ delegates a single responsibility to a collection of entities that can handle a specific request. Upon receiving a request, an entity can either handle the request if it is within the scope of its responsibility, or pass it along to the next entity in a chain of handlers. The Cocoa [__Responder Chain__](../Cocoa/Event-Handling-And-Responder-Chain.md) fits this design pattern.

---
## Example

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