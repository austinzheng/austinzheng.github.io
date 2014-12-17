---
layout: post
title: "Enums in Swift"
date:   2014-12-15 09:00:00
tags: swift
---

*Part 1 of a series on Swift enums, pattern matching, and generics.*

*Parts of this blog post are adapted from a [talk I gave](http://realm.io/news/swift-enums-pattern-matching-generics/) at the Swift Language Users Group.*

Like many programming languages both ancient and modern, Swift supports [**enums**][link-enums]. However, Swift's enums are considerably more powerful than those found in some of its predecessors.

Enums are [*value types*][link-valuetypes], like Swift's structs, and can be given computed properties and methods. Enums cannot be given stored properties.

Enums can actually be categorized into three distinct subcategories, informally referred to here as *basic enums*, *raw value enums*, and *associated value enums*. Read on to find out about each subcategory.


## Basic Enum ##

The **basic enum** just enumerates a number of fixed cases:

{% highlight swift %}
enum Direction {
    case North
    case South
    case East
    case West
    // For conciseness, could also declare as:
    // case North, South, East, West
}

// To create:
let myDirection = Direction.North

// To test:
if myDirection == Direction.North {
  println("you're facing north")
}
{% endhighlight %}

In this form, enums serve the same purpose they do in a language like C or Objective-C: providing a type-safe way to specify exactly one of a finite number of possible values.

This is preferable to, for example, using numbers to implicitly represent cases, in which you have two main problems:

* What each valid number means is either stored in your head or in a comment somewhere, rather than being baked into the code itself.
* If someone tries to use an invalid numerical value, the compiler won't warn you and your code will fail at runtime.

It's important to note that, unlike C or Objective-C enums, Swift enums declared in the above fashion are *not* equivalent to integers: `North` is not an alias for `0`, and so forth.


## Raw Value Enum ##

The second form of enum is the **raw value enum**. Each case of a raw value enum corresponds to a fixed 'raw' value:

{% highlight swift %}
enum Title : String {
    case CEO = "Chief Executive Officer"
    case CTO = "Chief Technical Officer"
    case CFO = "Chief Financial Officer"
}
{% endhighlight %}

Notice the two main differences between the `Title` enum and the basic `Direction` enum described previously:

* The type of the raw value, `String`, is specified after the name of the enum.
* Each case must specify an associated raw value.

Raw value enum cases are *not* aliases of their raw values, and the two cannot be used interchangeably:

{% highlight swift %}
// BAD! Will not compile!
let myCEOString : String = Title.CEO
// BAD! Will not compile!
let myCEOValue : Title = "Chief Executive Officer"
{% endhighlight %}

However, raw value enums do support conversion between each enum value and its corresponding raw value, via the `rawValue` property and a special initializer that takes a `rawValue` argument:

{% highlight swift %}
let myString : String = Title.CEO.rawValue
// myString is now "Chief Executive Officer"

let myTitle : Title = Title(rawValue: "Chief Executive Officer")!
// myTitle is now Title.CEO

let badTitle : Title? = Title(rawValue: "not a valid title")
// badTitle is now nil
{% endhighlight %}

Note that the initializer that turns a raw value (in this case, a string) into an enum instance returns an [*optional*][link-optional]. This is because it's possible to invoke the initializer with an invalid raw value, in which case it would return `nil` instead of an enum value.

Raw value enums whose raw values are of the `Int` type are treated slightly differently:

{% highlight swift %}
enum Planet : Int {
    case Mercury = 1
    case Venus, Earth, Mars // 2, 3, 4
    case Jupiter = 100
    case Saturn, Uranus, Neptune // 101, 102, 13
}
{% endhighlight %}

You do not need to explicitly define raw values for every case within an `Int`-valued raw value enum. Cases without explicit raw values will be assigned values 'counting up' from the last explicit raw value, or from `0` if no such explicit raw value exists. In the above example, since `Mercury` was explicitly assigned `1`, `Venus` is implicitly assigned `2`, and `Earth` is implicitly assigned `3`. However, since `Jupiter` was explicitly assigned `100`, `Saturn` implicitly gets `101`, and so forth.


## Associated Value Enum ##

This is where things begin to get interesting. Enums that aren't raw value enums can be declared in such a way that each case can store *associated data*. What does that mean? Take the following example, 'borrowed' from the official Swift book:

{% highlight swift %}
enum Barcode {
  case UPCA(sys: Int, data: Int, check: Int)
  case QRCode(data: String)
}

let myNormalBarcode = Barcode.UPCA(sys: 0, data: 2791701919, check: 3)
let myQRCode = Barcode.QRCode(data: "http://example.com")
{% endhighlight %}

The `Barcode` enum has two cases, `UPCA` and `QRCode`. However, each instance of a `Barcode`, which is one of those two cases, also has additional data associated with it! Exactly what data depends on the case. In the `UPCA` case, three integers are associated with a [UPC-A barcode][link-upc]. In the `QRCode` case, one string is associated with a [QR code][link-qr].

The associated values need not be labeled. The above example can be more concisely expressed as follows:

{% highlight swift %}
enum Barcode {
  case UPCA(Int, Int, Int)
  case QRCode(String)
}

let myNormalBarcode = Barcode.UPCA(0, 2791701919, 3)
let myQRCode = Barcode.QRCode("http://example.com")
{% endhighlight %}

Not all cases in an associated value enum need to contain associated data. For example, Swift's optionals are implemented as an enum!

{% highlight swift %}
enum Optional<T> {
    case None       // i.e. nil
    case Some(T)    // not nil, some value of type T
}
{% endhighlight %}

Associated value enums implement what are known as [sum types][link-adt]. One way to think of them is to consider them a type which represents *one* of several possible types (or type combinations):

{% highlight swift %}
enum EitherIntOrString {
    case AsInt(Int)
    case AsString(String)
}
{% endhighlight %}

`EitherIntOrString` can be thought of as a type that represents either an `Int` value or a `String` value (but not both at the same time). What about a generic version that can be used with any two types?

{% highlight swift %}
// Does not compile!
enum Either<T, U> {
    case First(T)
    case Second(U)
}
{% endhighlight %}

Unfortunately, this example code does not compile due to compiler limitations, although the code itself is perfectly valid. There are a number of [workarounds](https://github.com/robrix/Box) to achieve the same effect.

Sum types are great at representing data structures like trees. For example, the following example describes an enum which could be used to construct a tree representing a [JSON][link-json] data structure:

{% highlight swift %}
enum JSONNode {
    case NullNode
    case StringNode(String)
    case NumberNode(Float)
    case BoolNode(Bool)
    case ArrayNode([JSONNode])
    case ObjectNode([String:JSONNode])
}

// JSON: [10.0, "hello", false]
let myJSONTree : JSONNode = .ArrayNode([.NumberNode(10.0), .StringNode("hello"), .BoolNode(false)])
{% endhighlight %}

Now that we've created associated value enums, how do we work with them? Unlike the other two subcategories of enums, associated value enums can't be tested within `if` or `while` statements. This is true even for those cases that don't contain associated values, like `Optional`'s `None` case. Instead, we must use pattern-matching via `switch`:

{% highlight swift %}
let myNode : JSONNode = // ...
switch myNode {
    case .NullNode: println("null")
    case let .StringNode(s): 
        // Switch statements allow us to extract the associated value 
        // for an enum case, via let-binding
        println(s)
    case let .NumberNode(n): println("\(n)")
    case let .BoolNode(b): println("\(b)")
    case .ArrayNode: println("Array")
    case .ObjectNode: println("Object")
}
{% endhighlight %}

The next blog post in this series will discuss pattern-matching in more detail.

[link-enums]:                   http://en.wikipedia.org/wiki/Enumerated_type
[link-valuetypes]:              https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/ClassesAndStructures.html#//apple_ref/doc/uid/TP40014097-CH13-XID_144
[link-optional]:                http://en.wikipedia.org/wiki/Option_type
[link-upc]:                     http://en.wikipedia.org/wiki/Universal_Product_Code
[link-qr]:                      http://en.wikipedia.org/wiki/QR_code
[link-adt]:                     http://en.wikipedia.org/wiki/Algebraic_data_type
[link-json]:                    http://en.wikipedia.org/wiki/JSON