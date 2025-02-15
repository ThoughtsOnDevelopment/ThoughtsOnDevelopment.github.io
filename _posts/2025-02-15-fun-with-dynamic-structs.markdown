---
layout: post
title:  "Fun with Dynamic Structs in Swift"
date:   2025-02-15 01:54:00 +0300
tags: swift dynamic fun
---

Some really crazy stuff can be done with Swift.
Imagine that you have a task, which is basically a chain of calculations, and every calculation adds one new property. You want it to be readable, and since you're still playing around with it, you don't want to write a lot of boilerplate code. Also, everything should be type-safe.

For example, consider a real-world scenario where you are processing an order in an e-commerce application. The chain of calculations could look like this:

```
OrderID
    -> OrderDetails
    -> PaymentInfo
    -> ShippingAddress
    -> DeliveryStatus
```

Each step in the chain adds new information to the order, and Swift's type-safe features ensure that each piece of data is handled correctly.

How can we implement this in Swift?

## Tuples

Well, we could use tuples. But those are not really readable, and, if you need to add a new propery, you'll need to create another tuple. For the best readability, you'll propably need to create a typealias, and have named parameters. Like this:

```swift
typealias OrderStep1 = (id: Int)
typealias OrderStep2 = (id: Int, details: String)
typealias OrderStep3 = (id: Int, details: String, payment: Double)

func addPayment(_ prev: OrderStep2, payment: Double) -> OrderStep3 {
    return (prev.id, prev.details, payment)
}
```

Well, this will work. Not really readable, but, c'mon, it's working.

## Optionall the Things!

We could simly use a struct which will have all the properties optional. This will require having a lot of additional checks, whether the property is set or not, and can lead to some unexpected behavior, but sometimes it is a viable solution.

```swift
struct Order {
    var id: Int?
    var details: String?
    var payment: Double?
    var address: String?
    var status: String?
}

```

## Structs

We can have a struct for each step, and have a function that will create a new struct, for each step.
A bit more boilerplate, but this is, probably, the most readable and cleanest solution.

```swift
struct OrderStep1 {
    var id: Int
    func addDetails(_ details: String) -> OrderStep2 {
        .init(id: id, details: details)
    }
}

struct OrderStep2 {
    var id: Int
    var details: String
}
```

## Dynamic Struct

Now, let's try a more dynamic approach. The idea is to have a struct, which will have access to own property and will be able to create a new struct, adding a new property.

```swift
struct DynamicStruct<Base, Extension> {
    private let _extension: Extension
    private let _base: Base
    
    init(combine extensionData: Extension, with base: Base) {
        self._extension = extensionData
        self._base = base
    }
}

struct OrderID {
    var id: Int
}

let order = DynamicStruct(combine: OrderID(id: 1), with: ())
```

### Accessing underlying properties

Now, let's add an feature that will allow us to have access to the structure we're wrapping.
We will use [@dynamicMemberLookup](https://www.hackingwithswift.com/articles/55/how-to-use-dynamic-member-lookup-in-swift) for this.

```swift
@dynamicMemberLookup
struct DynamicStruct<Base, Extension> {
    ...
    /// Access to the base struct
    subscript<U>(dynamicMember keyPath: KeyPath<Base, U>) -> U {
        _base[keyPath: keyPath]
    }

    /// Access to the extesion
    subscript<U>(dynamicMember keyPath: KeyPath<Extension, U>) -> U {
        _extension[keyPath: keyPath]
    }
}

let order = DynamicStruct(combine: OrderID(id: 1), with: ())

// Now we can access the id property, and it will even be suggested by Xcode
print(order.id) // 1

```

### Adding a new property

Finally, let's add an ability to add more properties.

```swift
extension DynamicStruct {
    func extending<NewExtension>(with newExtension: NewExtension) -> DynamicStruct<Self, NewExtension> {
        .init(combine: newExtension, with: self)
    }
}

struct OrderAddress {
    let address: String
    let city: String
}

let oder = DynamicStruct(combine: OrderID(id: 1), with: ())
let withAddress = order.extending(with: OrderAddress(address: "Some address", city: "Some city"))

// Now we can access both the id and the address
print(withAddress.id) 
print(withAddress.address) 
print(withAddress.city)

```

We should probably want to have a tyealias for this, to make the code a bit more readable.

```swift
typealias Step1 = DynamicStruct<OrderID, ()>
typealias Step2 = DynamicStruct<Step1, OrderAddress>

extension Step1 {
    /// This extension is not really neccessary, but it will make the code a bit more readable
    func addAddress(_ address: OrderAddress) -> Step2 {
        order.extending(with: address)
    }
}
```

Finally!
We have a bit hacky, but working solution for creating a dynamic struct in Swift.
We now can add more properties, combine them, remove and play around with them in any order we want.

Now, if we change or add a new property, we would be able to do it on-the-fly, withouth touching the existing code too much.


## Conclusion

This post is a bit more about what can we do with Swift, rather than what we should do.

Dynamic structs can be used in some rare cases, but this is more about playing around with the language, and trying new approaches.

If you're about to use it in the production code, make sure you understand the consequences and the possible issues you might face. You'll probably need to check how fast this approach, and if it will affect the performance. Long generics chains in mix with dynamic properties can cause compile time issues, and a bigger metadata size. This approach even cannot handle the case, when there are two properties with the same name.


Hope you enjoyed this post, and may be will try to implement something similar in your project.
