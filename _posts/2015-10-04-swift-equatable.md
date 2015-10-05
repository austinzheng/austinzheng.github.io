---
layout: post
title: "Notions of Equality in Swift"
date:   2015-10-04 23:00:00
tags: swift
---

In this article I aim to discuss notions of equality, Swift's `Equatable` protocol, how equality in Swift differs from equality in more conventional object-oriented languages, and how the two can be reconciled.

## Notions of equality

The concept of equality is deceptively complex. In mathematics, we state that `2 + 2` is equal to `4`. In other words, when simplifying an equation or expression, any time we see `2 + 2` we can excise it and replace it with `4`. The two expressions are exactly equivalent.

But in the context of a windowing system, if two windows happen to be the same size, at the same position, and contain the same contents, are they interchangeable? Even if both windows' representations in memory are identical, randomly taking some of the references to one window and making them point to the other is unlikely to result in anything but strange behavior. Only if two references point to the same window instance can they be said to be equivalent.

Swift supports both of these forms of equality. Let's examine each in turn.


### Value equality

*Value equality* is checked using the `==` operator, which returns whether or not two instances are equivalent to each other.

{% highlight swift %}
let a = 10
let b = 10
let c = 11
a == b        // true; 10 == 10
a == c        // false; 10 != 11

// Swift arrays are value types, and hence the following holds
let d = [1, 2, 3]
let e = [4, 5, 6]
let f = [1, 2, 3]
d == e        // false
d == f        // true
{% endhighlight %}

Exactly what 'equivalent' means may vary based on the type of the instances being compared. However, value equality generally adheres to the following requirements:

* It should be *reflexive*; any object of a type supporting equality is equal to itself: `a == a`.
* It should be *symmetric*; `b == a` iff `a == b`.
* It should be *transitive*: if `a == b` and `b == c`, then `a == c`.

Implementations of `==` that violate these properties invariably result in subtle bugs and confused programmers, and should be avoided if at all possible.


### Reference equality

*Reference equality* is checked using the `===` operator, which returns whether or not two [*references*][link-wikiref] refer to the same object. Reference equality is only meaningful when reference types are being compared, whereas value equality pertains to both reference and value types.

(If you do not understand the distinction between reference and value types, I recommend reading Mike Ash's [blog post][link-valref] on the subject.)

Two distinct instances of an object can be equal when compared using `==`, but two references, each pointing to one of those instances, will compare as unequal using `===`. See the following example:

{% highlight swift %}
// Here is a simple class. Assume we've defined two MyObjects as equal in value
// iff both instance variables are equal across both objects.
class MyObject : Equatable {
  let a : Int, b : String
  init(a: Int, b: String) { self.a = a; self.b = b }
}
// ...

let a = MyObject(a: 10, b: "foo")
let b = a
let c = MyObject(a: 10, b: "foo")

a == b    // true; 'a' and 'b' are equal in value
a === b   // true; 'a' and 'b' point to the same instance

a == c    // true; 'a' and 'b' are equal in value
a === c   // false; 'a' and 'c' are different instances
{% endhighlight %}

Note that, according to the reflexive property, if two objects are reference-equal, they are also value-equal.


## The `Equatable` protocol

We will focus most of our attention on *value equality*, as reference equality's semantics are quite well defined and the default implementation of `===` suffices for almost all use cases.

In Swift, types accrete attributes by conforming to protocols. A type can declare to other code that it supports value equality by conforming to the `Equatable` protocol:

{% highlight swift %}
protocol Equatable {
  func ==(lhs: Self, rhs: Self) -> Bool
}
{% endhighlight %}

The protocol definition gives us enough information to figure out what conforming to `Equatable` means: instance of a conforming type can be compared to another instance **of the same type** to check whether or not they are equal in value.

`Self` in the protocol definition is a placeholder referring to the conforming type. Both arguments to `==` are of type `Self`, which means that a type which wishes to become `Equatable` must provide an implementation of the `==` operator which compares two instances of that same type.

As such, value equality in Swift is **homogeneous**: trying to compare two instances of different types causes a type error, rather than returning a guaranteed `false` result. When the built-in type `Int` conforms to `Equatable`, the only guarantee we get is that we are allowed to compare an `Int` to another `Int` using `==`. When the built-in type `String` conforms to `Equatable`, the only guarantee we get is that we are allowed to compare a `String` to another `String`, and so forth.


## Using `Equatable`

We can build a simple type that conforms to `Equatable` as follows:

{% highlight swift %}
struct Coordinate : Equatable {
  let x : Double, y : Double
}

func ==(lhs: Coordinate, rhs: Coordinate) -> Bool {
  return lhs.x == lhs.y && rhs.x == rhs.y
}
{% endhighlight %}

Then we can use it:

{% highlight swift %}
let a = Coordinate(x: 10, y: 20)
let b = Coordinate(x: 20, y: 10)
a == b      // false
{% endhighlight %}

Now, for something different. It would be great if we could write a function which took in three equatable objects and returned whether or not they were all equal in value:

{% highlight swift %}
// Note: does NOT compile!
func threeWayEquals(a: Equatable, b: Equatable, c: Equatable) -> Bool {
  return a == b && b == c
}
{% endhighlight %}

This doesn't compile! But what if it did? Remember that `Int` and `String` are both `Equatable` types as well. We could call the function with the following arguments:

{% highlight swift %}
let a : Int = 10
let b : String = "foobar"
let c : Coordinate = Coordinate(x: 10, y: 11)
threeWayEquals(a, b, c)
{% endhighlight %}

The types check out. `Int`, `String`, and `Coordinate` all conform to `Equatable`. But we don't have any guarantee that an implementation of `==` exists that can compare an `Int` to a `String`, or a `String` to a `Coordinate`, and if those implementations don't actually exist, now the compiler is in trouble.

The actual Swift compiler complains that our protocol can only be used as a constraint on a generic parameter because its definition contains `Self`. We can fix our function to allow it to compile:

{% highlight swift %}
func threeWayEquals<T : Equatable>(a: T, b: T, c: T) -> Bool {
  return a == b && b == c
}
{% endhighlight %}

This version of the function requires all its arguments to be `Equatable` as well, but also requires them to all be of the same type (a type that, therefore, conforms to `Equatable`). In this case the compiler is guaranteed the implementation of `==` that it needs exists.


## Object-oriented equality

Why is Swift equality so 'different' from equality in, say, Objective-C? Swift complains if we compare a `String` and an `Int`; Objective-C is fine if we send the message `isEqual:` to an instance of `NSNumber` with a `NSString` as the argument. We can answer this question by looking at how object-oriented languages treat the concept of equality.


### Basics of inheritance

First of all, many commonly used object-oriented languages ship with a designated base class from which all other classes should inherit. For example, Java and C# have `Object`, while Objective-C has `NSObject` (and `NSProxy`). Swift is atypical in that it does not provide such a base class; any Swift class can serve as a base class.

In a program written in an object-oriented language, if `B` is a subclass of `A`, ideally instances of `B` can be used anywhere instances of `A` can be used without breaking the program. (This is the [Liskov substitution principle][link-liskov].) We often say `B` ['is-a'][link-isa] `A`. If `Cow` inherits from `Herbivore` inherits from `Animal`, a `Cow` is a `Cow`, but also a more specific type of `Herbivore`, which is in turn a more specific type of `Animal`. In the Cocoa world, `NSString` is a more specific type of `NSObject`; `UILabel` is a more specific type of `UIView`.


### Creating a base class

In object-oriented languages with a designated base class, the base class usually specifies some overarching functionality, such as comparing an object with another object.

In languages like Java and Objective-C, the `==` operator applied to instances of objects checks reference equality like Swift's `===` operator. In order to check value equality, an `equals` method is defined on the base class. This method invariably takes another instance of the base class and returns a boolean.

To illustrate how this works, let's define a 'sublanguage' based off Swift. We'll create our own class hierarchy, with our own designated base class (`AZObject`), and decree that all user-specified types must inherit from `AZObject` or one of its subclasses.

{% highlight swift %}
// Our base class
class AZObject : Equatable {
  init() { }

  // Return whether or not an object is 'value-equal' to another object.
  func equals(another: AZObject) -> Bool {
    return self === another
  }
}

func ==(lhs: AZObject, rhs: AZObject) -> Bool {
  return lhs.equals(rhs)
}
{% endhighlight %}

Our `AZObject` base class provides an `equals()` method, since it's advantageous to allow any object to be compared with any other object. We'll decide that any one of our featureless `AZObject` instances can only be equal to itself, which means that value equality and reference equality for `AZObject`s are identical.


### Some `AZObject` subclasses

If we create a subclass of our base class, for example `AZData`, that subclass can then override `equals()` however it wants. For example, two `AZData`s might only be equal if all their constituent bytes are equivalent to each other:

{% highlight swift %}
/// An object representing an immutable blob of binary data.
class AZData : AZObject {
  private let ptr : UnsafeMutablePointer<UInt8>
  private let backingStore : UnsafeMutableBufferPointer<UInt8>

  override func equals(another: AZObject) -> Bool {
    if let anotherData = another as? AZData {
      // Perform a byte-by-byte comparison of the data contained within the two buffers.
      let thatBackingStore = anotherData.backingStore
      guard backingStore.count == thatBackingStore.count else { return false }
      for (index, item) in backingStore.enumerate() {
        if thatBackingStore[index] != item { return false }
      }
      return true
    }
    // If the other object isn't a data blob, they're obviously not equal.
    return false
  }

  // ...

  init(bytes: [UInt8]) {
    ptr = UnsafeMutablePointer.alloc(bytes.count)
    backingStore = UnsafeMutableBufferPointer(start: ptr, count: bytes.count)
    for (index, b) in bytes.enumerate() { backingStore[index] = b }
  }

  deinit { ptr.destroy() }
}
{% endhighlight %}

Two `AZNumber`s might compare some normalized representation of their numeric values:

{% highlight swift %}
/// An object representing a number.
class AZNumber : AZObject {
  private enum NumberType {
    case Integer(Int), FlPt(Double), Boolean(Bool)
  }
  private let backingStore : NumberType

  override func equals(another: AZObject) -> Bool {
    if let anotherNumber = another as? AZNumber {
      // For didactic purposes, use the floating point value as the normalized value
      // This allows "AZNumber(100)" to equal "AZNumber(100.0)"
      return flPtValue == anotherNumber.flPtValue
    }
    return false
  }

  var flPtValue : Double {
    switch backingStore {
    case let .Integer(v): return Double(v)
    case let .FlPt(v): return v
    // Please don't actually define your Booleans like this.
    case let .Boolean(v): return v ? 1 : 0
    }
  }

  // ...
}
{% endhighlight %}

Different types may have very different notions of what value equality means, even though they all descend from the same base class.


### All objects are equatable

In the system we've devised, comparing two objects that aren't the same type is a valid operation; it just always returns `false`. As such, value equality for our `AZObject`-based sublanguage is **heterogeneous**. In effect, any object can be compared with any other object.

More specifically, this is possible because:

* All subclasses of `AZObject` are also `AZObject`s ('is-a'). An `AZData` is also an `AZObject`; an `AZNumber` is also an `AZObject`.
* Our contract on `AZObject` specifies that value equality is a meaningful operation defined on two `AZObject`s.
* Therefore, equating any two instances of `AZObject` subclasses is a meaningful operation, since it is equivalent to equating two `AZObject` instances.

In practice, this is how both Java and Objective-C implement value equality. All your custom types descend from `Object` and `NSObject`, so you get some notion of value equality 'for free' from the default implementation.


### Limitations

What happens if we define a third class, `AZJSONNode`, but neglect to override `equals()` within our implementation? In that case, we fall back to `AZObject`'s definition of `equals()`, the one that checks for reference equality. In many cases this is incorrect behavior.

Heterogeneous comparison is fraught with danger. Neither compile-time nor run-time checking will warn us if we forget to override `equals()`. Our default implementation of `AZObject`, so convenient and inviting at first, ends up breaking the Liskov substitution principle in cases where the default behavior is wrong.

At the very least, programmers who implement their own custom classes under such a system need to be aware of the differences between value equality and reference equality. They need to know that their types' definitions of value equality default to reference equality unless they explicitly implement their own `equals()` methods. If they aren't aware of all this (and unless they read the documentation closely they're unlikely to be), they run the risk of writing seemingly-working code containing subtle bugs.


### Solutions

If [abstract methods][link-absmeth] existed in Swift, we might choose to make `equals()` abstract, forcing each type to provide its own implementation, but this would come at the cost of making it impossible to instantiate `AZObject`s. (Depending on our use case, this might not be a bad tradeoff.)

A better, more general solution is described in the [Protocol-Oriented Programming][link-wwdcpop] video from WWDC 2015 (around the 40:30 mark). We can define a protocol `AnyEquatable` and a protocol extension that only applies when the type the protocol is applied to is also `Equatable`. `AnyEquatable` itself contains no associated types, and unlike `Equatable` can be used outside the context of generic constraints:

{% highlight swift %}
protocol AnyEquatable { }

extension AnyEquatable where Self : Equatable {
  // otherObject could also be 'Any'
  func equals(otherObject: AnyEquatable) -> Bool {
    if let otherAsSelf = otherObject as? Self {
      return otherAsSelf == self
    }
    return false
  }
}
{% endhighlight %}

Note that the implementation of `equals()` is, structurally, a generalization of the `equals()` methods implemented for `AZData` and `AZNumber`. A downcast is used to ensure both arguments are of the same type, and if so value equality is checked using `==`. Otherwise, the comparison returns `false`.

Any value that is both `AnyEquatable` and `Equatable` can be compared with any other `AnyEquatable` value 'for free', and with the correct semantics. We need write no additional code except for a few empty extensions declaring conformance:

{% highlight swift %}
extension Int : AnyEquatable { }
extension String : AnyEquatable { }

class Foo : AnyEquatable { }

// This compiles, and returns 'true', since Int is Equatable.
15.equals(15)

// These return 'false'.
15.equals(16)
15.equals("hello")

// This also returns 'false', even though Foo isn't Equatable.
15.equals(Foo())
{% endhighlight %}

Our `AnyEquatable` protocol isn't a perfect replacement for `Equatable`. If we wanted to implement our `threeWayEquals()` using `AnyEquatable`, we'd still need to constrain at least one of the arguments' types to be `Equatable` so we have access to our `equals()` method:

{% highlight swift %}
func threeWayEquals<T where T : AnyEquatable, T : Equatable>(a: T,
  b: AnyEquatable,
  c: AnyEquatable) -> Bool {
    return a.equals(b) && a.equals(c)
}
{% endhighlight %}

However, if we already know at least one of the concrete types we want to compare, we no longer need to make our function generic at all.

{% highlight swift %}
func threeWayEquals(a: Int, b: AnyEquatable, c: AnyEquatable) -> Bool {
  return a.equals(b) && a.equals(c)
}
{% endhighlight %}


## Conclusion

Despite marketing slogans, Swift isn't "Objective-C without the C". It approaches many problems in a way very different from common object-oriented languages, often for sound underlying reasons. The differences between value equality and reference equality are subtle yet, to some extent, fundamental. Instead of papering over them, Swift makes the delineation between the two concepts explicit.

People often like to characterize programming languages as more or less 'strict'. I would argue that 'strict' might not be the best analogy. Some languages provide a high level of convenience in exchange for semantics that are hidden away or made implicit. Other languages make the semantics more explicit, requiring a bit of additional work on the programmer's end.

Even so, at every point along this spectrum your programs are being executed in a specific way, whether or not you are aware of what those specifics are. And I would argue that it's important to be aware, whether you're using an object-oriented or functional language, a statically-typed or dynamically-typed one.

Some of the work Swift makes you do is due to limitations of the type system. Some of it might be due to unfamiliarity. But some of it involves clarifying intent, resolving ambiguity, and making you think about exactly what you want your program to do. And in the long run, that's a good thing.


[link-wikiref]:     https://en.wikipedia.org/wiki/Reference_(computer_science)
[link-valref]:      https://mikeash.com/pyblog/friday-qa-2015-07-17-when-to-use-swift-structs-and-classes.html
[link-isa]:			https://en.wikipedia.org/wiki/Is-a
[link-liskov]:      https://en.wikipedia.org/wiki/Liskov_substitution_principle
[link-absmeth]:		https://en.wikipedia.org/wiki/Method_(computer_programming)#Abstract_methods
[link-wwdcpop]:		https://developer.apple.com/videos/play/wwdc2015-408/
