# Builder
---

__Builder__ is a creational mechanism that allows for construction of complex objects step by step. You can create different representations of an object via the same construction process.

---
## Example

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
