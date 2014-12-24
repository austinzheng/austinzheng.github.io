---
layout: post
title: "Protocols in Swift"
date:   2014-12-24 03:00:00
tags: swift
---

*Part 4 of a series on Swift enums, pattern matching, and generics. [Previous post.]({% post_url 2014-12-17-custom-pattern-matching %})*

*Parts of this blog post are adapted from a [talk I gave](http://realm.io/news/swift-enums-pattern-matching-generics/) at the Swift Language Users Group.*

Having covered Swift's pattern-matching switch statement to some level of detail, we now turn our attention to another one of Swift's major features: generics. In order to understand how generics work in Swift, though, it is first necessary to understand Swift's [**protocols**][link-protocols].

## Protocol basics ##

Those who have worked with Objective-C or other object-oriented languages might already be familiar with [protocols][link-wikiprotocols]. For example, protocols in Java are known as [interfaces][link-javainterface]. For those who aren't as familiar with the concept, the following brief overview will explain what protocols are and how they work.

A protocol can be thought of as a *promise* or *contract* that a type (specifically, a class, struct, or enum) makes with the compiler. Protocols may be empty, or they may have one or more method or property signatures. The following example declares a simple protocol:

{% highlight swift %}
protocol MyProtocol {
  var someProperty : Int { get set }
  func someMethod(x: Int, y: Int) -> String
}
{% endhighlight %}

Let's break this protocol declaration down:

* The protocol is declared with the `protocol` keyword, and its name is `MyProtocol`
* The first item the protocol defines is an `Int`-typed property named `someProperty`; this property must be readable and writable
* The second item the protocol defines is a method that takes two `Int`s as arguments and returns a `String`
* The protocol declares interfaces for its property and method, but it doesn't define how they are implemented

Now that we have a protocol, what can we do with it? It turns out that protocols exist so that *other types can conform to them*. A type can declare that it conforms to one or more protocols. When it does so, it is promising the compiler that it will provide implementations for all the methods and properties declared in those protocols.

Let us take the following class:

{% highlight swift %}
class MyClass {
  // nothing here...
}
{% endhighlight %}

...and make it conform to `MyProtocol`. To declare that a type conforms to one or more protocols, a colon is inserted after the type name, and the protocols are listed after the colon. (If the type is a class that has a superclass, the protocols are listed after the superclass.)

{% highlight swift %}
class MyClass : MyProtocol {
  // still nothing here...
}
{% endhighlight %}

Are we done yet? We are not: the previous snippet won't compile. This is because we've made `MyClass` promise that it will provide implementations for the methods and properties in `MyProtocol`. But `MyClass` doesn't actually have any code to implement either the method or property. The compiler sees that `MyClass` isn't fulfilling the promise it made, and so it will complain.

How can we rectify this? We can add code to implement the property and method in `MyProtocol`:

{% highlight swift %}
class MyClass : MyProtocol {
  var myProperty : Int = 0
  func someMethod(x: Int, y: Int) -> String {
    return "your numbers are \(x) and \(y)"
  }
}
{% endhighlight %}

Now the compiler will consider the promise satisfied and compile our code. It's important to note that, although the protocol defined what properties and methods that conforming types had to implement, it was up to `MyClass` to decide how it was going to implement those properties and methods for itself.


## Using protocols ##

How can we leverage the fact that a given type conforms to some given protocol?

Properties, variables, and constants in Swift are typed. We are accustomed to giving them concrete types:

{% highlight swift %}
let townName : String = "West Meoley"
var population : Int = 3000

func printMyNumber(myNumber: Int) { 
  println("my number is \(myNumber)")
}
{% endhighlight %}

However, we can also declare a property, variable, or constant with a *protocol* as its type:

{% highlight swift %}
// 'Printable' is a protocol that defines a 'description' property, which is a
//  description of the object
var itemToPrint : Printable
itemToPrint = 10.0
println(itemToPrint.description)
itemToPrint = false
println(itemToPrint.description)

func printMyItem(myItem : Printable) {
  println("My item is " + myItem.description)
}
{% endhighlight %}

If a property, variable, or constant is declared with a protocol as its type, we have access to all the methods and properties that protocol defines - and nothing else.

We can also declare a property, variable, or constant whose type must satisfy *multiple protocols*, using `protocol<ProtocolA, ProtocolB, ProtocolC, ...>`:

{% highlight swift %}
// 'DebugPrintable' is a protocol that defines a 'debugDescription' property
func printMyItem(myItem : protocol<Printable, DebugPrintable>) {
  let desc = myItem.description
  let debugDesc = myItem.debugDescription
  println("My item is described as \"\(desc)\", and its debug description is \"\(debugDesc)\"")
}

struct Point : Printable, DebugPrintable {
  let x : Int
  let y : Int
  var description : String { return "(\(x), \(y))" }
  var debugDescription : String { return "x value: \(x), y value: \(y)" }
}
printMyItem(Point(x: 10, y: 20))
// Prints to console:
// My item is described as "(10, 20)", and its debug description is "x value: 10, y value: 20"
{% endhighlight %}

If you try playing around with Swift's built-in protocols, you might run into an error that reads something like `Protocol 'Foo' can only be used as a generic constraint because it has Self or associated type requirements`. The next article in this series will discuss generics in detail, but suffice it to say that you cannot use the techniques described above to work with certain protocols. Instead, you must constrain your types using the generics system. A simple example follows:

{% highlight swift %}
// 'Equatable' is a protocol that allows types that implement it to be compared using ==
// This does NOT work
func equateThreeValues(x: Equatable, y: Equatable, z: Equatable) -> Bool {
  return (x == y) && (y == z)
}

// Instead, you must do this
func equateThreeValues<T : Equatable>(x: T, y: T, z: T) -> Bool {
  return (x == y) && (y == z)
}
{% endhighlight %}

A future article will discuss why this restriction actually makes sense (and why the first, invalid example is actually semantically incorrect) in more detail.


## Motivation ##

Hopefully, it is now clear that:

* Protocols may *define* methods and properties (or be empty)
* Types can choose to *conform* to protocols
* A type that conforms to a protocol must provide implementations for that protocol's methods and properties
* If given a variable (constant, property) of protocol type, that protocol's methods and properties can be invoked upon the variable

But the question remains: why are protocols useful?

Just as protocols forced you to fulfill a promise by implementing some methods or properties, a protocol also serves as a promise to you, the developer, that some object or value is capable of doing something. For example, if you have an object that conforms to the `Printable` protocol, you don't need to care whether it's an `Int`, a `Double`, a `Range`, an `Array`, or something else; you can be assured that it will have a `description` property you can call to get a string describing the object.

### Parent-child relationships ###

One practical use of protocols is to define flexible parent-child relationships.

For example, the `UITableView` class in Cocoa implements a list-style UI widget (a [*table view*][link-tableview]) consisting of some number of rows, called *cells*. The table view needs some way to figure out how many rows it should display, exactly what cell it should display for a given row, what to do when a given row is tapped, and so forth. For now, let's just worry about how the table view knows how many rows to display.

If we were inventing `UITableView`, one way we might solve this problem is by defining a `UITableViewDataSource` class with the method `numberOfRows() -> Int`. The table view would then have a `dataSource` property of type `UITableViewDataSource`:

{% highlight swift %}
class UITableViewDataSource {
  func numberOfRows() -> Int {
    fatalError("You must subclass this class and override this method!")
  }
}

class UITableView {
  var dataSource : UITableViewDataSource?

  func renderTableView() {
    let rowCount = dataSource.numberOfRows()
    for rowIdx in 0..<rowCount {
      // render the cell at index 'rowIdx'...
    }
  }
}

// MyDataSource is a *subclass* of UITableViewDataSource
class MyDataSource : UITableViewDataSource {
  override func numberOfRows() -> Int {
    return 10
  }
}

let myTableView = UITableView()
myTableView.dataSource = MyDataSource()

{% endhighlight %}

Developers would subclass the `UITableViewDataSource` class and override the `numberOfRows()` method to return however many rows they wanted, and then assign to their table view's `dataSource` property an instance of this custom subclass. Later, the table view would  query the subclass by calling the `numberOfRows()` method and getting the number of rows it should display.

This solution works fine, but it forces our developers to make their data source objects subclasses of our `UITableViewDataSource` class. This might be too restrictive, especially since the table view only really cares that it has a `numberOfRows()` method to call on its `dataSource`.

A better option would be to ditch our class and instead create a `UITableViewDataSource` *protocol* instead. This protocol would declare the `numberOfRows()` method. Then, our developers would be able to create any type of data source object, be it a class or struct, and declare that their data source conforms to our `UITableViewDataSource` protocol. Finally, they'd implement the `numberOfRows()` method in their data source object. Everything else would work the same way:

{% highlight swift %}
protocol UITableViewDataSource {
  func numberOfRows() -> Int
}

class UITableView {
  var dataSource : UITableViewDataSource?

  func renderTableView() {
    let rowCount = dataSource.numberOfRows()
    for rowIdx in 0..<rowCount {
      // render the cell at index 'rowIdx'...
    }
  }
}

// MyDataSource is a struct that *implements* UITableViewDataSource
struct MyDataSource : UITableViewDataSource {
  func numberOfRows() -> Int {
    return 10
  }
}

let myTableView = UITableView()
myTableView.dataSource = MyDataSource()

{% endhighlight %}

In fact, the actual `UITableView` class works very much like the hypothetical table view we've just described.

### Aggregating functionality ###

Protocols can also be used to [aggregate functionality][link-composition] for a given type.

For example, let's say we are building a computer game, and we're writing a struct to represent the player character. Player characters are *named* objects in the game, because the player has a name, and they can be *healed* (like non-player characters and monsters, but not items). Player characters can also be *moved*, since the player is a mighty hero/heroine of prophecy (and not a tree). A sketch of how we might implement such a model follows:

{% highlight swift %}
protocol Nameable {
  var name : String { get }
}

protocol Healable {
  func heal(hitPointsToHeal: Int)
}

protocol Moveable {
  func move(direction: Direction, distance: Double)
}

// A PlayerCharacter has a name, therefore it conforms to Nameable
// A PlayerCharacter can be healed, therefore it conforms to Healable
// A PlayerCharacter can be moved, therefore it conforms to Moveable
class PlayerCharacter : Nameable, Healable, Moveable {
  var name : String = // ...
  func heal(hitPointsToHeal: Int) { /* ... */ }
  func move(direction: Direction, distance: Double) { /* ... */ }
}
{% endhighlight %}

Many of the types included in the Swift standard library are defined in such a way. For example, the `CFunctionPointer` struct in Swift, which represents a [C function pointer][link-fnptr], has the following signature:

{% highlight swift %}
struct CFunctionPointer<T> : Equatable, Hashable, NilLiteralConvertible {
  // ...
}
{% endhighlight %}

This type conforms to three different protocols:

* `Equatable`, which means that you can compare two `CFunctionPointer`s using the `==` operator
* `Hashable`, which means that you can get a `CFunctionPointer`'s hash value by checking its `hashValue` property (this also means you can use it as a key in a dictionary)
* `NilLiteralConvertible`, which means that the type has provided an initializer that allows Swift to convert a `nil` into a `CFunctionPointer` representing a null pointer in C

A historical note: when used in an object-oriented context, Swift's system of single inheritance and protocols is sometimes termed 'single inheritance of implementation, multiple inheritance of interface'. What this means is that a class in Swift can inherit an *interface* for a method or property from multiple places (protocols, its superclass), but it can only inherit at most one *implementation* (from its superclass).

### Heterogeneous collections ###

A third possible use for protocols is *giving multiple types something in common*, so they can be mixed and matched within the same array or dictionary. (More generally, this can be viewed as allowing various types to all be described by the same generic type parameter, a topic which will be covered in more detail in the forthcoming article on generics.)

Those who went through the previous posts may remember an example in which JSON data was represented using enums and associated types. What if we want a less cumbersome way to represent JSON, closer in spirit to Objective-C's untyped collections?

{% highlight swift %}
// An empty protocol; it exists to mark different types as JSON types
protocol JSONType { }
// We can use extensions to make existing classes conform to additional protocols
extension String : JSONType { }
extension Double : JSONType { }
extension Bool : JSONType { }
extension Array : JSONType { }
extension Dictionary : JSONType { }

let b : [JSONType] = [10.1, "hello", "goodbye"]
let a : [JSONType] = [10.2, "foo", true, false, b]
{% endhighlight %}

This looks beautiful! We can create an array mixing and matching JSON types as easily as we could in Objective-C. However, there is a serious problem: *all* arrays and dictionaries are now considered valid JSON types, not just those containing JSON-typed objects! Unfortunately, Swift's type system is not powerful enough to represent what we really want, which is something like:

{% highlight swift %}
// This is NOT valid Swift code
// Only Arrays which contain objects of JSONType are also JSONType conforming
extension Array<T where T : JSONType> : JSONType { }
{% endhighlight %}

Note that even though this JSON example isn't practical, the technique itself is still valuable (as long as we remain cognizant of its limitations). For example, let's say we wanted to collect the results of various computations in an array, and then print them all out once all the computations had completed:

{% highlight swift %}
// A custom type, representing a two-dimensional point
struct Point : Printable {
  let x : Int
  let y : Int
  var description : String { return "(\(x), \(y))" }
}

// For some odd reason String isn't publicly declared as Printable 
extension String : Printable {
  public var description : String { return self }
}

let buffer : [Printable] = [1.0, "hello", true, Point(x: 10, y: 20)]
// Note that running this in a Playground won't print out Point's description correctly
println(buffer.description)
// Or...
for item in buffer {
  println(item.description)
}
{% endhighlight %}

In this case, we are able to store our original results in `buffer`, and then leverage the fact that they are all `Printable` to print out their descriptions for the benefit of the end user.


## More about protocols ##

Now that we have established both the fundamental operation and motivation behind protocols, a few notes on several important details about protocols follow.

### Properties in protocols ###

Protocols can declare properties, as noted earlier, but only using `var`. Properties can be declared as either `get` (read-only) or `get set` (read-write). They cannot be declared as write-only. A conforming type can choose to implement a property either as a stored property or a computed property.

Even if a protocol declares a property as `get`, a conforming type can choose to implement that property as a read-write property (for example, by implementing a setter for a computed property, or by using a stored property).


### Protocol inheritance ###

Protocols can inherit from other protocols. Unlike with classes, a protocol can inherit from multiple parent protocols. In the following example, `P3` inherits from both `P1` and `P2`:

{% highlight swift %}
protocol P1 { 
  func foo()
}

protocol P2 {
  func bar()
}

// Any type that implements P3 must implement not only baz(), but also foo()
//  and bar()
protocol P3 : P1, P2 { 
  func baz() 
}
{% endhighlight %}

If a type implements a protocol, it automatically conforms to that protocol's parent protocols as well:

{% highlight swift %}
struct MyStruct : P3 {
  func foo() { println("foo") }
  func bar() { println("bar") }
  func baz() { println("baz") }
}

func doSomething(x: P1) {
  x.foo()
}
// MyStruct conforms to P3, but can be used as a P1 or P2
doSomething(MyStruct())
{% endhighlight %}

### Extensions and protocols ###

Swift's [**extensions**][link-extensions] are a way to add functionality to an existing type. Extensions can be used to add protocol conformance to an existing type, as demonstrated by this (admittedly contrived) example:

{% highlight swift %}
protocol NotEmptyProtocol {
  var notEmpty : Bool { get }
}

extension Optional : NotEmptyProtocol {
  var notEmpty : Bool {
    switch self {
    case .None: return false
    case .Some: return true
    }
  }
}

extension Array : NotEmptyProtocol {
  var notEmpty : Bool {
    return !self.isEmpty
  }
}
{% endhighlight %}

Another common use of extensions is to break the definition of a single type into several chunks, each logically organized around one or more protocols.

### Initializers, subscripts, and operators ###

In addition to methods and properties, protocols can also define [initializers][link-initializers], [subscripts][link-subscripts], and [operators][link-operators].

A protocol declares an **initializer** just like it would a method:

{% highlight swift %}
protocol InitWithIntegerProtocol {
  init(integerValue: Int)
}
{% endhighlight %}

Protocols definining initializers are most useful in the context of generics, and will be further discussed in a forthcoming article.

A protocol declares a **subscript** using a combination of function and property notation, similar to how subscripts are declared in concrete types. Like with properties, subscripts can either be declared as `get` or `get set`.

{% highlight swift %}
protocol IntStringSubscriptable {
  subscript(idx: Int) -> String { get set }
}
{% endhighlight %}

A protocol declares an **operator** just like it would a method. However, a conforming class does not implement the overloaded operator as a method; instead, it must be implemented as a global function with the proper types. For example, Swift's `Equatable` protocol is defined as follows:

{% highlight swift %}
protocol Equatable {
  func ==(lhs: Self, rhs: Self) -> Bool
}
{% endhighlight %}

(`Self` is a generic parameter that must be substituted with the conforming type's own type.)

If a type wished to implement the `Equatable` protocol, it would do so as follows:

{% highlight swift %}
struct Point : Equatable {
  let x : Int
  let y : Int
}

func ==(lhs: Point, rhs: Point) -> Bool {
  return lhs.x == rhs.x && lhs.y == rhs.y
}
{% endhighlight %}

Notice how the `==` operator overload is defined outside the `Point` struct, and how both `lhs` and `rhs` are of type `Point` (in accordance with the protocol definition).

### Class-only protocols ###

A protocol can be limited to only classes by listing `class` before any parent protocols:

{% highlight swift %}
protocol ClassOnlyProtocol : class, ParentProtocol1, ParentProtocol2 {
  //...
}
{% endhighlight %}

Trying to define a struct or enum that conforms to a class-only protocol will cause a compile-time error.

### `@objc` protocols ###

A protocol can be declared with the modifier `@objc`:

{% highlight swift %}
@objc protocol LegacyProtocol {
  //...
}
{% endhighlight %}

`@objc` protocols have certain characteristics:

* They are exposed to Objective-C code as part of the interop system
* Only classes can conform to them; enums and structs cannot conform to a `@objc` protocol
* `is`, `as`, and `as?` can be used to check or downcast a class to an `@objc` protocol type
* Methods and properties can be marked `optional` (which removes the requirement for conforming classes to implement them), and optional chaining can be used to try invoking an optional method or property

The Apple [interop guide][link-interop] has more information on using `@objc` protocols.

Thanks for reading this rather long post about Swift's protocols! The next blog post in this series will discuss generics in Swift.


[link-wikiprotocols]:     http://en.wikipedia.org/wiki/Protocol_(object-oriented_programming)
[link-protocols]:         https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html
[link-javainterface]:     http://docs.oracle.com/javase/tutorial/java/concepts/interface.html
[link-tableview]:         https://developer.apple.com/library/ios/documentation/UserExperience/Conceptual/TableView_iPhone/AboutTableViewsiPhone/AboutTableViewsiPhone.html
[link-composition]:       http://en.wikipedia.org/wiki/Composition_over_inheritance
[link-fnptr]:             http://en.wikipedia.org/wiki/Function_pointer
[link-extensions]:        https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Extensions.html
[link-initializers]:      https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Initialization.html
[link-subscripts]:        https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Subscripts.html
[link-operators]:         https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AdvancedOperators.html#//apple_ref/doc/uid/TP40014097-CH27-XID_84
[link-interop]:           https://developer.apple.com/library/ios/documentation/Swift/Conceptual/BuildingCocoaApps/MixandMatch.html#//apple_ref/doc/uid/TP40014216-CH10-XID_86
