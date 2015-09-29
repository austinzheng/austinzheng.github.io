---
layout: post
title: "Generics in Swift, Part 2"
date:   2015-09-29 03:30:00
tags: swift
---

*Part 6 of a series on Swift enums, pattern matching, and generics.*

In this article, I aim to provide a comprehensive description of how the generic programming facilities in Swift work. For a gentler introduction, please see the [previous post]({% post_url 2015-01-02-swift-generics-pt-1 %}) in this series.

[Generic programming][link-generic] is a form of programming in which things such as types (classes, structs, enums) and functions can be defined with the help of **type parameters**. Type parameters are placeholders for actual types which are 'filled in' whenever the generic type or function is actually used.

The most visible example of generic programming in Swift is the humble `Array` type. In Objective-C, an `NSArray` was an `NSArray`, and it could contain objects of any type. Swift doesn't have 'just' `Array`s, though. Arrays are always parameterized by the type of the items they carry. So in Swift we have `Array<Int>`, `Array<UIView>`, and so forth. `Array` is a type, and `Int` is a type, and generics allow these two types to work together in a way that makes sense and conveys extra information.

(*n.b.* The idiomatic way to represent an `Array` containing `Foo` instances in Swift is `[Foo]`. This article uses `Array<Foo>` in order to aid understanding and emphasize the fact that `Array` is an ordinary generic type; the longer form is exactly equivalent to `[Foo]`.)

## Why generics?

Here are a few reasons why generic programming is useful for a statically typed language:

* **Type safety**. Container types (e.g. arrays) can be given information about the types of the items they hold, which allows the compiler to ensure that only the right kinds of objects can be inserted into or returned from the collection. This applies to any type which can be parameterized in terms of one or more other types, and it also applies to relationships between types.

* **Less code duplication**. In some cases you need to carry out essentially the same operation on items of varying types. Instead of writing multiple copies of the same function which differ only in terms of their argument and return value types, you can write a single generic function. This is less error-prone and can make your intent clearer.

* **Library flexibility**. Libraries that expose APIs can avoid forcing their consumers to supply arguments and accept return values of a fixed type. Instead, they can abstract over these types using generics. For example, generics might allow an API function to accept not only `Array` arguments, but an instance of any type that can be treated as a collection.

If you don't understand yet how generics apply to these topics, read through the rest of the article and revisit this section. Hopefully, seeing how generics work will make things clearer.


## Generic entities

Generic programming in Swift shows up in two different contexts: when defining types, and when defining functions. The presence of a **generic type signature**, the part of the type or function declaration denoted by the angle brackets `<` and `>`, indicates that a type or function is generic.

### Generic types

All three of Swift's primary user-defined types can be made generic. Here's an example of a `Result` enumeration type that can store either a `Success` value or a `Failure` value.

{% highlight swift %}
enum Result<T, U> {
  case Success(T)
  case Failure(U)
}
{% endhighlight %}

Right after the name of the type, `Result`, we define the *type parameters*, `T`, and `U`. These type parameters are placeholders that will be replaced with actual types when creating an instance of that type. For example:

{% highlight swift %}
let aSuccess : Result<Int, String> = .Success(123)
let aFailure : Result<Int, String> = .Failure("temperature too high")
{% endhighlight %}

A generic type's type parameters can be used in the following ways:

* As the type of a property
* As the type of an associated value in an enum case
* As a return type or an argument type for a method or subscript
* As an argument type for an initializer
* As a type parameter for another generic type used within the type definition (for example, an `Array<T>` property)

Type parameters like `T` only show up in the definition of a type. Whenever that type is used, those type parameters must be replaced by actual types.

For example, `Array` is a generic type whose type parameter is named `Element`. This type parameter describes the type of the things the `Array` stores. When you write Swift code, you never deal with plain `Array`s or `Array<Element>`s. You always deal with `Array`s of concrete types: for example, `Array<String>` or `Array<UIWindow>`. If you ever work with `Array<T>`s, it is *always* in the context of type parameters that were defined by other types or functions.


### Generic functions

Functions and methods can be generic as well, either by themselves or (in the case of methods) in the context of a generic type.

{% highlight swift %}
// Given an item, return an array with that item repeated n times
func populate<T>(item: T, numberOfTimes times: Int) -> [T] {
  var buffer : [T] = []
  for _ in 0 ..< times {
    buffer.append(item)
  }
  return buffer
}
{% endhighlight %}

Again, the type parameters are defined within the angle brackets `<` and `>` immediately following the function name. They can then be used as part of the argument or return types.

Types can have generic methods, whether or not the type itself is generic. 

{% highlight swift %}
extension Result {
  // Transform the value of the 'Result' using one of two mapping functions
  func transform<V>(left: T -> V, right: U -> V) -> V {
    switch self {
    case .Success(let value): return left(value)
    case .Failure(let value): return right(value)
    }
  }
}
{% endhighlight %}

Our `transform` method is generic, and resides within the generic type `Result`. In addition to the type parameters `T` and `U` defined by `Result`, we also have access to the type parameter `V` defined by the generic method itself.

Be aware that 'shadowing' type parameters (for example, if the method had defined `U` instead of `V`) can lead to cryptic error messages.


## Associated types

Protocols in Swift cannot be defined generically using type parameters. Instead, protocols can define what are known as **associated types** using the `typealias` keyword.

{% highlight swift %}
// A protocol for things that can accept food.
protocol FoodEatingType {
  typealias Food

  var isSatiated : Bool { get }
  func feed(food: Food)
}
{% endhighlight %}

In this example, `Food` is an associated type defined on the protocol `FoodEatingType`. Protocols can support as many associated types as necessary.

Associated types, like type parameters, are placeholders. A type that wants to conform to the protocol gets to decide what concrete type `Food` should be (e.g. `Hay` or `Rabbit`). The way it does this is by implementing the protocol's properties and methods, and deciding what types replace the associated types in the actual implementations. Here is an example:

{% highlight swift %}
class Koala : FoodEatingType {
  var foodLevel = 0
  var isSatiated : Bool { return foodLevel < 10 }

  // Koalas are notoriously picky eaters
  func feed(food: Eucalyptus) {
    // ...
    if !isSatiated {
      foodLevel += 1
    }
  }
}
{% endhighlight %}

The *associated type* `Food` is defined to be `Eucalyptus` for `Koala`s. In other words, `Koala.Food` is defined to be `Eucalyptus`. A type that conforms to multiple protocols might end up pulling in multiple associated types.

If a conforming type is generic, you can also use the type parameters to help define the associated types:

{% highlight swift %}
// Gourmand Wolf is a picky eater and will only eat their favorite food.
// Individual wolves may prefer different foods, though.
class GourmandWolf<FoodType> : FoodEatingType {
  var isSatiated : Bool { return false }

  func feed(food: FoodType) {
    // ...
  }
}

let meela = GourmandWolf<Rabbit>()
let rabbit = Rabbit()
meela.feed(rabbit)
{% endhighlight %}

In this case, `GourmandWolf<Goose>.Food` is `Goose`, while `GourmandWolf<Sheep>.Food` is `Sheep`.

As an aside, associated types can be required to conform to protocols by appending `:` and one or more comma-separated protocol names. For example, the keys of a [heap][link-heap] must be comparable, and we can express that constraint as follows:

{% highlight swift %}
// Types that conform represent simple max-heaps which use their elements as keys
protocol MaxHeapType {
  // Elements must support comparison ops; e.g. 'a is greater than b'.
  typealias Element : Comparable

  func insert(x: Element)
  func findMax() -> Element?
  func deleteMax() -> Bool
}
{% endhighlight %}

Finally, the special 'associated type' `Self` in a protocol definition always refers to the conforming type's type. For example, a type `Foo` that implements a protocol `BarType` must replace `Self` with `Foo` when implementing `BarType`'s methods or properties.


## Type constraints

Up until now, we've only seen 'free' type parameters. `T` and `U` in our previous examples could be satisfied by any type. The standard library's `Array` is an example of a type that places no constraints on its type parameter: if you can define a type `Foo`, you can create an `Array<Foo>`.

### Motivating example

Sometimes, this isn't enough. Let's work through an illustrative example. We will try to write a function that takes in an array of objects and returns the 'largest' one:

{% highlight swift %}
// Doesn't compile.
func findLargestInArray<T>(array: Array<T>) -> T? {
  if array.isEmpty { return nil }
  var soFar : T = array[0]
  for i in 1..<array.count {
    soFar = array[i] > soFar ? array[i] : soFar
  }
  return soFar
}
{% endhighlight %}

Swift's compiler complains. It turns out that our code involves a comparison: `array[i] > soFar`. We know that `array[i]` is of type `T`, and `soFar` is also of type `T`. But Swift asks us (not unreasonably): "how can you know that `T`s are comparable?". What if we created an empty struct `Foo`, and called this function on an array of `Foo`s? In this case, there isn't any version of `>` that can compare two `Foo`s.

In Swift, [protocols are how types declare they support functionality]({% post_url 2014-12-24-protocols-in-swift %}). It turns out there is a standard library protocol which guarantees that conforming types can be compared using `>`, and it's called (surprise) `Comparable`. So if we could somehow restrict our arguments to our function to be "only arrays whose elements are `Comparable`", we could get our function to compile properly.

This is indeed possible, as the revised example below shows. Try calling the function with arrays containing instances of other types: `String` (compiles) and `NSView` (won't compile).

{% highlight swift %}
// Note that <T> became <T : Comparable>, meaning that whatever type fills in
// 'T' must conform to Comparable.
func findLargestInArray<T : Comparable>(array: Array<T>) -> T? {
  if array.isEmpty { return nil }
  var soFar : T = array[0]
  for i in 1..<array.count {
    soFar = array[i] > soFar ? array[i] : soFar
  }
  return soFar
}

// Example usage:
let arrayToTest = [1, 2, 3, 100, 12]
// We're calling 'findLargestInArray()' with T = Int.
if let result = findLargestInArray(arrayToTest) {
  print("the largest element in the array is \(result)")
} else {
  print("the array was empty...")
}
// prints: "the largest element in the array is 100"
{% endhighlight %}

The paradox of generics is as such: the more you narrow down your generic type parameters by adding constraints, the more you're allowed to do with them. Type parameters that are completely unconstrained can't be used for much more than swapping or insertion/removal from collections. Type parameters that are tightly constrained gain access to many more methods and properties than they would otherwise, and can be used as arguments to many more functions.


### Simple constraints

If the only constraint you need is for each type parameter to conform to at most one protocol, you can append `:` and the name of the protocol after the type parameter. In the following example, `T` can be any type, but `U` must be `Equatable` and `V` must be `Hashable`:

{% highlight swift %}
func example<T, U : Equatable, V : Hashable>(foo: T, bar: U, baz: V) {
  // ...
}
{% endhighlight %}

### What can you constrain?

Perhaps you need finer-grained control. Fair enough. Let's first tally up exactly what you have at your disposal:

* Type parameters like `U`, `V`, etc. 

These might have been defined by your type or method's generic type signature, or by an enclosing type or method. (For example, a generic method belonging to a generic type `Foo` has access to `Foo`'s type parameters, as well as its own. Refer to the `transform()` example from earlier.)

* Type parameters' associated types (if any)

If you require any type parameters to conform to a protocol, you get access to the type parameter's associated types as well. For example, if `U` conforms to `FoodEatingType` from before, you automatically gain access to the associated type `U.Food`. If `V` conforms to both `MaxHeapType` and the standard library's [`RawRepresentable` protocol][link-rawrep], you now have both `V.Element` and `V.RawValue` to work with.

### How can you constrain it?

Swift supports three types of constraints:

* `T : SomeProtocol`: the type parameter `T` must be satisfied by a type which conforms to the protocol `SomeProtocol`. Note that protocol composition using `protocol<Foo, Bar>` works here.
* `T == U`: the type parameter or associated type `T` must be equal to the type parameter or associated type `U`.
* `T : SomeClass`: `T` must be a class; more specifically, `T` must be satisfied by an instance of `SomeClass` or one of its subclasses. This one is rare.


### Putting it all together

Put together your generic type signature as follows:

* Declare your type parameters. If you want, each type parameter declared here can conform to a single protocol as described above.
* `where` keyword
* Declare your constraints, separated by commas.

Here is a (highly contrived) example intended to demonstrate the syntax:

{% highlight swift %}
protocol Foo {
  typealias Key
  typealias Element
}

protocol Bar {
  typealias RawGeneratorType
}

func example<T : Foo, U, V where V : Foo, V : Bar, T.Key == V.RawGeneratorType, U == V.Element>
  (arg1: T, arg2: U, arg3: V) -> U {
  // ...
}
{% endhighlight %}

Don't be intimidated! We're going to break the generic type signature down step by step. Before the `where` keyword, we declare three type parameters: `T`, `U`, and `V`. `T` must conform to the `Foo` protocol (`T : Foo`).

After the `where` keyword, we declare four constraints:

* `V` must conform to the `Foo` protocol (`V : Foo`).
* `V` must also conform to the `Bar` protocol (`V : Bar`).
* Since `T` conforms to `Foo`, `T` has an associated type `T.Key`. `V` also has an associated type `V.RawGeneratorType`. Those two types must be the same (`T.Key == V.RawGeneratorType`).
* Since `V` conforms to `Foo`, `V` has an associated type `V.Element`. That associated type must be the same as `U` (`U == V.Element`).

Now, wherever you use the `example()` function, you need to choose your types `T`, `U`, and `V` such that they fulfill all these criteria. Only then will your code compile.


## Constrained extensions

New in Swift 2 are constrained extensions, a powerful language feature which leverages generics.

In Swift, [extensions][link-extension] allow you to add methods to any type, even types you didn't define. They also allow you to add default implementations to methods defined in a protocol. Constrained extensions let you take this even further:

* For a **generic type** (like `Array`), you can add methods which are only available if the generic type's type parameters (or those parameters' associated types) match a certain constraint.
* For a **protocol with associated types** (like `CollectionType`), you can add default implementations for protocol methods which are only available if the associated types match a certain constraint.

Methods on constrained extensions are a great replacement for generic functions that have elaborate, difficult-to-read generic type signatures.

(*n.b.* In addition to methods, extensions also support computed properties, initializers, and subscripts, and all of this applies to those as well.)

### Syntax and limitations

The syntax for a constrained extension is as follows:

{% highlight swift %}
// Methods in this extension are only available to Arrays whose elements are
// both hashable and comparable.
extension Array where Element : Hashable, Element : Comparable {
  // ...
}
{% endhighlight %}

The `where` keyword follows the type name, and is itself followed by a list of one or more constraint clauses separated by commas.

You can't (yet) constrain generic types so tightly that they would become non-generic (using `==`):

{% highlight swift %}
// An extension on Array<Int>.
// Error: "Same-type requirement makes generic parameter 'Element' non-generic"
extension Array where Element == Int {
  // ...
}
{% endhighlight %}

Nor can your constrained extensions add conformance to additional protocols:

{% highlight swift %}
protocol MyProtocol { /* ... */ }

// Only Arrays with comparable elements conform to MyProtocol.
// Error: "Extension of type 'Array' with constraints cannot have an inheritance clause"
extension Array : MyProtocol where Element : Comparable {
  // ...
}
{% endhighlight %}

Hopefully both these limitations will disappear in the future.


### Extension on `CollectionType`

Let's rewrite the `findLargestInArray()` free function we defined earlier as a method on a constrained extension. In fact, we can allow this method to operate on any collection type (defined by the [`CollectionType` protocol][link-collectiontype]), not just arrays!

Types that conform to `CollectionType` represent collections of elements, like arrays and sets. It turns out that, for various reasons, `CollectionType` has an associated type that conforms to `GeneratorType`, and `GeneratorType` in turn has an associated type named `Element`. If you want to learn more about why this is, check out my [article on sequences]({% post_url 2015-01-24-swift-seq %}). Otherwise, just note that the type of a `CollectionType`'s elements can be represented by `Self.Generator.Element`.

Now, if we want to write a method for `CollectionType` instances that finds the largest element in the collection, it makes sense to restrict this method to only those `CollectionType` instances containing things that can be compared. This lends itself naturally to the following:

{% highlight swift %}
extension CollectionType where Self.Generator.Element : Comparable {
  func largestValue() -> Generator.Element? {
    // 'guard var' is, indeed, a real thing .__.
    guard var largestSoFar = first else {
      return nil
    }
    for item in self {
      // 'item' and 'largestSoFar' are both type Self.Generator.Element.
      // Because of the 'where' clause, we know that Self.Generator.Element is
      // Comparable, and so we can invoke the '>' operator on its instances.
      if item > largestSoFar {
        largestSoFar = item
      }
    }
    return largestSoFar
  }
}
{% endhighlight %}

(Note that we could have left off the '`Self.`' in the `where` clause constraint, for brevity.)

Now we can call our new method on any `CollectionType` whose elements are comparable. Behold:

{% highlight swift %}
// Works on Arrays...
[1, 2, 3, 4, 5].largestValue()

// ...the CharacterViews of Strings...
"qwertyuiop".characters.largestValue()

// ...and even Ranges
(0..<9002).largestValue()
{% endhighlight %}


### Extension on `Array`

Extensions on generic types work similarly. The primary difference is that you can constrain on the generic type parameters as well (note that `Array` is defined as `Array<Element>`).

The following extension method matches the behavior of our free `findLargestInArray()` function more closely, since it can only be called on `Array` instances. Otherwise it is nearly the same as the one we defined for `CollectionType`:

{% highlight swift %}
extension Array where Element : Comparable {
  func largestValue() -> Generator.Element? {
    guard var largestSoFar = first else {
      return nil
    }
    for item in self {
      if item > largestSoFar {
        largestSoFar = item
      }
    }
    return largestSoFar
  }
}
{% endhighlight %}


## Details

Now that you (hopefully) understand the semantics of generic programming in Swift, a discussion of several related topics follows.

### Implementation

There are a couple of ways in which generics might be implemented by a language.

Because Java needed to maintain backwards compatibility when generics were introduced, generics in Java are implemented using [*type erasure*][link-te]. When Java code is compiled, the compiler performs type checking and then discards the type information, inserting downcasts as necessary. Because type information is no longer available at runtime, Java's generics are [limited in quite a few ways][link-reifiedjava].

Another approach, taken by C++, is to perform [specialization][link-specialization]. In this approach, different implementations of the same generic entity (in C++, *template*) may be generated by the compiler, each implementation corresponding to use of that entity by specific types. Each implementation might be optimized differently based on the concrete types that it takes.

Swift's generics support is closer to that of C++. The Swift compiler can emit different variants of the same generic function if doing so will allow for increased optimization. Type information is retained at runtime, allowing you to write code like the following:

{% highlight swift %}
protocol SimpleInitable { init() }

// Create an array filled with 'count' empty instances of T
func emptyInstances<T : SimpleInitable>(count: Int) -> [T] {
  return (0 ..< count).map { _ in T() }
}
{% endhighlight %}

Without specialization, we might invoke this method with differing return types, but it would be impossible at run-time to know exactly what concrete type `T` should be used to fill the array.


### Limitations

Generics are a compile-time construct. Generic code is specialized at compile time, and all the information that generics require to function must be available at compile time. This means that generics aren't a replacement for runtime type checking and casting using `is` and `as?`, or for overriding methods in subclasses.

If a generics-based solution to a given problem seems excessively troublesome (in particular, if you run into issues getting the compiler to accept your code), ask yourself:

* Could I replace my use of generics with multiple hand-coded non-generic copies of my generic functions and types? (In other words, could I manually specialize my code?)

* Is my program relying on information known only at run-time in order to determine what concrete types should be used when invoking my generic functions or types? For example, a function returns instances of class or protocol `A`, and you are trying to use generics to implement polymorphic behavior depending on what subclass or conforming type is actually dispensed.

If your answer to the first question is 'no' and/or your answer to the second question is 'yes', generics are almost certainly not the right tool for whatever you are trying to accomplish.


### Conclusion

Static typing can be a useful tool, but a type system that is insufficiently expressive only gets in the way of the software engineer. Swift's type system, while still limited in some important ways, derives a great amount of power from the language's support for generics. Much of the Swift standard library utilizes generic programming, and many open-source libraries do so as well. This ultimately grants you, the developer, additional flexibility in choosing how to implement solutions to your problems.

Suggestions and criticism are welcome. In particular, if any of the subject matter remains unclear please drop me a note; anything that helps me improve this article is greatly appreciated.


[link-generic]:           https://en.wikipedia.org/wiki/Generic_programming
[link-te]:                https://en.wikipedia.org/wiki/Type_erasure
[link-specialization]:    https://en.wikipedia.org/wiki/Generic_programming#Template_specialization
[link-reifiedjava]:       http://gafter.blogspot.ro/2006/11/reified-generics-for-java.html 
[link-heap]:              https://en.wikipedia.org/wiki/Heap_(data_structure)
[link-rawrep]:            https://developer.apple.com/library/prerelease/ios/documentation/Swift/Reference/Swift_RawRepresentable_Protocol/index.html
[link-extension]:         https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Extensions.html
[link-collectiontype]:    https://developer.apple.com/library/prerelease/ios/documentation/Swift/Reference/Swift_CollectionType_Protocol/index.html
