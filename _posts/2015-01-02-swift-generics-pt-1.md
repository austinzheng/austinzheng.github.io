---
layout: post
title: "Generics in Swift, Part 1"
date:   2015-01-02 02:00:00
tags: swift
---

*Part 5 of a series on Swift enums, pattern matching, and generics. [Previous post.]({% post_url 2014-12-24-protocols-in-swift %})*

*Parts of this blog post are adapted from a [talk](http://realm.io/news/swift-enums-pattern-matching-generics/) I gave at the Swift Language Users Group.*

Swift supports a concept known as [generic programming][link-gp], often abbreviated as **generics**. Succinctly, generics can be understood as a tool allowing us to write code that isn't specific to a single type, but rather can be used across a wide variety of different types.

## Motivation ##

Say that one day, we decide to write a function that takes in two `Int`s and swaps them in-place. (For those who are interested, this is an expanded version of the [example][link-generics] from the official Swift book.)

{% highlight swift %}
func swapInts(inout this: Int, inout that: Int) {
  let temp = this
  this = that
  that = temp
}
{% endhighlight %}

This function is quite straightforward. The `inout` modifier on `this` (and `that`) simply means that reassigning `this` in the function will reassign whatever variable was passed into the function when it was called. For example:

{% highlight swift %}
var a = 10
var b = 20

println("a = \(a), b = \(b)")    // prints out "a = 10, b = 20"
swapInts(&a, &b)
println("a = \(a), b = \(b)")    // prints out "a = 20, b = 10"
{% endhighlight %}

This is great, until we decide that we want a function to swap two `String`s in-place. But we can't use `swapInts`, since that function doesn't accept `String`-typed arguments. Instead, we write a second function:

{% highlight swift %}
func swapStrings(inout this: String, inout that: String) {
  let temp = this
  this = that
  that = temp
}
{% endhighlight %}

This works, but it's not a good solution for two main reasons:

* What if we decide tomorrow that we need to be able to swap `Bool`s as well? For every new type we want to be able to swap a new function must be written.
* Our two functions, `swapInts` and `swapStrings`, contain the same code. If we find a bug in one implementation we must fix it in both.

### Solving the problem with `Any` ###

How can we make our code better? Inspired by dynamically typed languages, we replace our swap functions with one that takes two arguments of `Any` type. (`Any` is a special type alias that can describe any Swift type, vaguely similar to `id` from Objective-C.)

{% highlight swift %}
func swap(inout this: Any, inout that: Any) {
  let temp = this
  this = that
  that = temp
}
{% endhighlight %}

Excellent! We've replaced both `swapInts` and `swapStrings` with a single function that can be used with any type, not just strings and integers! However, our elegant solution comes with one glaring problem: it is no longer type safe at compile-time. For example:

{% highlight swift %}
var firstInt = 10
var secondInt = 20
var firstString = "hello"
var secondString = "world"

// These work fine
swap(&firstInt, &secondInt)
swap(&firstString, &secondString)

// This compiles, but will crash at runtime
swap(&firstInt, &secondString)
{% endhighlight %}

Note that `swapInts` and `swapStrings` *are* type safe: if you try to pass arguments with invalid types into those functions the compiler will produce an error at compile-time.

### Solving the problem with generics ###

Is there a way to combine both the elegance of our `Any`-based solution with compile-time type safety? Yes, if we use generics.

If we think about our `swap` function a bit more, we can conceptualize it as a function that takes two variables and swaps their values. We don't care *what* types the two variables are, as long as *they both have the same type*. Generic programming allows us to express this constraint and write the following function:

{% highlight swift %}
func swap<T>(inout this: T, inout that: T) {
  let temp = this
  this = that
  that = temp
}
{% endhighlight %}

The `<T>` following the function name indicates that we are declaring a *type parameter* named `T`. `T` by itself isn't a concrete type, but it can be thought of as a wildcard representing another type, such as `String` or `Int`, when used in the function signature.

We also note that both our parameters are of type `T`, which means that they both have to be the same type. So if we pass a `String` for `this`, we must pass a `String` for `that` as well.

It turns out that there is actually a `swap` function included in Swift's standard library! As we might expect, its function signature is very similar to the function signature from our example:

{% highlight swift %}
func swap<T>(inout a: T, inout b: T)
{% endhighlight %}

This sort of programming technique is known more formally as [*parametric polymorphism*][link-ppoly]. In parametric polymorphism, a function is written so that it can apply the same basic operations to its input values no matter what types the inputs are.

There are other forms of polymorphism as well; these include *ad-hoc polymorphism*, where a function or operator is overloaded so, for example, `==` compares integers differently than it compares strings, and *inclusion polymorphism*, where you can use a subclass of `Foo` anywhere you'd ordinarily use a `Foo`.

### So, their purpose is? ###

Generics increase the expressiveness of the code we write while retaining type safety. Using generics, those of us programming in a language with a static typing discipline can recapture some measure of the freedom afforded to those using dynamically-typed languages like Python or Ruby. 

Because we used generics, we don't have to duplicate code. Instead of writing out dozens of `swap` functions for all the types we care about, we only need one function that handles any type we need.

In a similar vein, generics make typed containers practical. We don't need to implement dozens of collection types such as `ArrayOfInts` or `ArrayOfStrings`. Instead, `Array<T>` suffices, where `T` can be any type we want our array to contain.

Generics do come with potential downsides. They make the type system more complex (and more powerful), so it takes more time to reason about and model our problems using generics. Generics also can't describe every possible set of constraints we might come up with.

## Generic functions, types, and protocols ##

In Swift, *functions* and *types* can be made generic. There also exist tools for defining *protocols* that can be used alongside generic types.

### Functions ###

The `swap` function in the previous section is an example of a generic function. Let's examine it in more detail:

{% highlight swift %}
func swap<T>(inout this: T, inout that: T) {
  let temp = this
  this = that
  that = temp
}
{% endhighlight %}

This function's name is `swap`. Immediately following the name, but preceding the list of formal parameters enclosed by the parentheses `(` and `)`, is the generic type signature. This is denoted by the angle brackets `<` and `>`.

Within the generic type signature a single *type parameter*, `T`, is declared. If needed, multiple type parameters can be declared, separated by commas: `<T, U>`. Conventionally, type parameters are given either single capital letter names, or `CamelCase` names where the first letter of the name is capitalized.

A type parameter is a 'variable' for a type. At compile-time, we don't know exactly what type will be used when calling the function, so we refer to the type using the placeholder `T`. The actual values of the type parameters are determined when the function is called:

{% highlight swift %}
var a : Int = 10
var b : Int = 20

// This invocation of swap() has T = Int, since a and b are both Ints
swap(&a, &b)

var x : String = "foo"
var y : String = "bar"

// This invocation of swap() has T = String, since x and y are both Strings
swap(&x, &y)
{% endhighlight %}

Type parameters can be used when declaring the types of the function's *arguments*, as well as the function's *return type*. If two arguments (or an argument and a return value) are declared with the same type parameter, then they must have the same types when the function is used. For example, in our `swap` function, both `this` and `that` are declared to be of type `T`, so they must have the same type.

Two arguments that are declared with different type parameters don't need to be of the same type when the function is used, but they don't have to be of different types either. For example:

{% highlight swift %}
func foo<T, U>(arg1: T, arg2: U) {
  /* ... */
}

// These are all fine
foo(100, "Swift")
foo(100, 200)
{% endhighlight %}

Note that methods, initializers, operator implementations, and subscripts can also be generic in the same way as functions.

### Types ###

Types (structs, enums, and classes) can also be generic. Types are generic often because we wish to express them in terms of other types. For example, we talk about an `Array` of integers, or an `Array` of strings. We talk about a `Dictionary` mapping strings to floats. We talk about an `Optional` of type `Foo`. Many container types are implemented as generic types.

For generic types, the generic type signature comes immediately after the type name, and before the colon and any superclass or protocols. Type parameters from the generic type signature can be used in properties, methods, initializers, or subscripts, as shown below:

{% highlight swift %}
// A container representing an N x N square matrix
struct SquareMatrix<T> {
  var backingArray : [T] = []
  let size : Int
  
  func itemAt(row: Int, column: Int) -> T {
    // ...
  }

  init(size: Int, initial: T) {
    self.size = size
    backingArray = Array(count: size*size, repeatedValue: initial)
  }
}
{% endhighlight %}

The full type of a generic type is fixed when an instance of that type is created. For example:

{% highlight swift %}
let a = SquareMatrix(size: 10, initial: 50)
// a is a SquareMatrix<Int>

let b : SquareMatrix<String?> = SquareMatrix(size: 5, initial: nil)
// b is a SquareMatrix<String?>
{% endhighlight %}

In the first case, the compiler is smart enough to infer that, since the initializer is being called with an `Int` for the argument `initial`, the full type of `a` must be `SquareMatrix<Int>`. In the second case, the full type of `b` is explicitly specified (and is actually required, since it's not clear exactly what type the compiler should be inferring from the initial value).

### Protocols ###

Protocols can't be directly made generic. Instead, protocols can have what are called *associated types*, which are declared using `typealias`:

{% highlight swift %}
protocol FooProtocol {
  typealias SomeType
  func fooFunc() -> SomeType
}
{% endhighlight %}

The associated type in the previous example is `SomeType`. It serves some of the same purposes as the type parameters we've seen previously: it's not a specific type like `Int` or `String`, but rather a wildcard that is used in the function signature for `fooFunc()`.

Associated types are given a concrete type value *when a type conforms to a protocol*. For example:

{% highlight swift %}
struct Bar : FooProtocol {
  func fooFunc() -> String {
    return "metasyntactic variables are awesome"
  }
}
{% endhighlight %}

How does this work? In order to conform to `FooProtocol`, `Bar` had to implement `fooFunc()`. However, `fooFunc`'s return type is an associated type, which means that `Bar` can choose whatever type it wants as the return type!

Since `Bar` chose to implement `fooFunc()` such that it returns a `String`, the associated type `Bar.SomeType` thus becomes `String`. Note that our `Bar` struct never explicitly states "my `SomeType` should be set to `String`". Rather, it creates that association implicitly when it chooses to implement `fooFunc()` in a certain way.

If another type (e.g. `Baz`) conforms to `FooProtocol`, its associated type (e.g. `Baz.SomeType`) can be something completely different. Associated types are declared by protocols, but *associated* with the types that conform to the protocols.

So, with all of this said and done, what type information is attached to a given struct, enum, or class?

* The base type (for example, `Array` or `Dictionary`)
* The types that any type parameters are set to (for example, `<Int>` or `<String, Bool>`)
* Any associated types defined by protocols that the struct, enum, or class conforms to

Finally, protocols can also use the special type `Self` when defining their methods or properties (or subscripts, initializers, etc). `Self` must be equal to the conforming type. This is better shown in an example:

{% highlight swift %}
protocol ZeroValueable {
  class func zeroEquivalentValue() -> Self
}

struct Coord : ZeroValueable {
  let x : Int
  let y : Int
  static func zeroEquivalentValue() -> Coord {
    return Coord(x: 0, y: 0)
  }
}
{% endhighlight %}

Since the protocol declares `zeroEquivalentValue` to return a value of type `Self`, `Coord` has no choice but to implement `zeroEquivalentValue` as a function whose return type is `Coord`.

## So far... ##

Up to this point we've tried to answer two questions:

* What problem do generics solve?
* How do we add support for generics to functions, types, and protocols?

There is a third, critically important question that the next blog post will cover:

* How do we define constraints on our generic type variables?

Without constraints, the usefulness of generics is limited. The next post will also contain a extended example which will tie together all the concepts we discussed earlier about generics, as well as several smaller examples to try and make explicit the concepts underlying generics.

If you are still here, thanks for reading! Generics are not a trivial topic, especially for those who haven't seen them in other languages before.

[link-gp]:          http://en.wikipedia.org/wiki/Generic_programming
[link-generics]:    https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Generics.html
[link-ppoly]:       http://en.wikipedia.org/wiki/Parametric_polymorphism