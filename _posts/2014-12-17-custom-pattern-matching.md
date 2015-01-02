---
layout: post
title: "Custom pattern matching in Swift"
date:   2014-12-17 19:00:00
tags: swift
---

*Part 3 of a series on Swift enums, pattern matching, and generics. [Previous post.]({% post_url 2014-12-16-swift-pattern-matching-switch %})*

*Parts of this blog post are adapted from a [talk](http://realm.io/news/swift-enums-pattern-matching-generics/) I gave at the Swift Language Users Group.*

By now, if you've been following along with the other posts in this series, you've seen how the switch statement's pattern matching works in a variety of different scenarios. It turns out, though, that you can define novel pattern matching behavior as well.

## Pattern-match operator ##

The key to doing so is the `~=` operator, which takes in two expressions: a *pattern expression* and a *predicate expression*, and returns true or false depending on whether the two expressions match. (Exactly what 'match' means is up to the code implementing the operator.)

`~=` is rather strange in that (as far as I can tell) it is not intended to be directly used as an operator. It is only briefly mentioned in the [Swift book][link-exptr] and in the Swift standard library headers. However, it serves an important role as the mechanism underlying the pattern-matching capabilities of Swift's switch statement.

A function prototype for implementing `~=` follows, where `PatternType` and `PredicateType` can be replaced with whatever types you want:

{% highlight swift %}
func ~=(pattern: PatternType, predicate: PredicateType) -> Bool
{% endhighlight %}

To make clearer the purpose of `~=`, consider the following switch statement:

{% highlight swift %}
switch predicate {
case pattern1: 
  doSomething()
case pattern2: 
  doSomethingElse()
// ...
default: 
  doDefault()
}
{% endhighlight %}

You could conceivably rewrite that switch statement as the following, functionally equivalent if-else ladder:

{% highlight swift %}
let r = predicate
if pattern1 ~= r {
  doSomething()
}
else if pattern2 ~= r { 
  doSomethingElse()
}
// ...
else {
  doDefault() 
}
{% endhighlight %}

(Note, however, that the Swift grammar prevents you from using certain pattern expressions in an if-else statement, so the two forms are not fully interchangeable.)


## Motivation ##

Imagine the following pseudo-Swift switch statement:

{% highlight swift %}
let myArray = [1, 2, 4, 3]
switch myArray {
case [..., 0, 0, 0]:
  // Arbitrary-length array whose last three items are all 0
  doSomething()
case [4, ...]:
  // Arbitrary-length array whose first item is 4
  doSomething2()
case [_, 2, _, 3]:
  // Four-item array whose second item is 2, and whose fourth item is 3
  doSomething3()
case [_, _, 8]:
  // Three-item array whose third item is 8
  doSomething4()
default:
  doDefault()
}
{% endhighlight %}

This sort of matching seems impossible given Swift's built-in capabilities, but it turns out that we can actually implement something very similar!


## Creating a pattern ##

From the definition of `~=` provided earlier, we know that we need to write a function that takes a predicate expression (in this case, an array of integers), as well as some sort of pattern expression, and returns whether or not the predicate matches the pattern.

This leads to the question: what sort of pattern expression do we want to use? We know that we want four sorts of tokens or markers in our pattern expression:

* A way to specify that an integer in the predicate array at a given position should match a certain value
* A way to specify that an integer in the predicate array should be accepted, no matter what (the `_` in the pseudocode)
* A way to specify that we should match all things from the beginning of the array (the `...`)
* A way to specify that we should match all things to the end of the array (also `...`)

To fulfill this requirement, we can create the following enum:

{% highlight swift %}
enum Token {
  case Wildcard         // the '_' in the pseudocode
  case FromBeginning    // the '...' in the pseudocode
  case ToEnd            // the '...' in the psudocode
  case Literal(Int)
}
{% endhighlight %}

An array of `Token`s can therefore be used as our pattern.


## Review of operator overloading ##

Now that we've figured out all the types we need, we can write the actual matcher. To do this, we will take advantage of Swift's support for custom and overloaded operators, since `~=` is an operator.

But first, a brief overview of how custom and overloaded operators work in Swift. (For a more comprehensive reference, refer to the [*Advanced Operators* section][link-advop] in the Swift book.)

In languages like C, Java, or Objective-C, [*operators*][link-op] are special constructs that act somewhat like functions in that they take in one or more arguments and perform some action and maybe return some value; they may also be invoked in special ways. However, operators in these languages are baked into the language specification and cannot be added to or extended.

At the other extreme, languages like Scala have almost no dedicated operators. Instead, constructs that would be operators in other languages are implemented as methods. Concomitant to this flexibility is a large list of syntax rules that define special ways that methods can be called, for example allowing for expressions like `a + 2` rather than `a.+(2)`.

Swift takes a middle ground, allowing custom operators to be defined while subjecting them to some restrictions. Custom operators must start with one of several predefined characters and be declared as prefix, postfix, or infix. Once a custom operator is *declared*, one can then write a global function to *implement* the custom operator for some permutation of argument and return types. Certain built-in operators can't be overriden, but all other operators are 'custom' operators defined in the standard library.

It's possible to write multiple implementations for a single operator, as long as the implementations have different type signatures. This is what we mean by *operator overloading* - an operator behaves differently depending on the types of the values passed to it. For example, you could write a class representing [matrices](http://en.wikipedia.org/wiki/Matrix_(mathematics)) and overload `+` to allow you to add two matrix objects just as naturally as you'd add two `Int`s. Swift's type system allows it to choose the appropriate implementation based on how the operator is used.

When we implement our custom matcher, we are actually overloading the `~=` operator. Once we're done, Swift will know that, whenever the predicate is an array of `Int`s and the pattern is an array of `Token`s, it should use our custom matcher. Otherwise, it'll try to use one of the other matchers that ship as part of the standard library.


## Writing the matcher ##

The custom matcher, once implemented, is as follows:

{% highlight swift %}
func ~=(pattern: [Token], predicate: [Int]) -> Bool {
  var ctr = 0
  for currentPattern in pattern {
    if ctr >= predicate.count || ctr < 0 {
      return false
    }
    switch currentPattern {
    case .Wildcard:
      ctr++
    case .FromBeginning where ctr == 0:
      ctr = (predicate.count - pattern.count + 1)
    case .FromBeginning:
      return false
    case .ToEnd:
      return true
    case .Literal(let v) where v != predicate[ctr]:
      return false
    case .Literal:
      ctr++
    }
  }
}
{% endhighlight %}

The matcher itself is not very complicated. It walks through both the pattern and predicate arrays, taking different actions based on what tokens it finds in the pattern array:

* If the matcher sees a wildcard, it moves on to the next token unconditionally.
* The only valid place a `FromBeginning` token can be is at the start; otherwise, the counter is moved to near the end of the array.
* If a `ToEnd` token is found, everything else is assumed to match.
* If a literal value token is found, the values are compared.
* A counter check forces the matcher to return false if improper use of the `FromBeginning` or `ToEnd` tokens moves the counter out of bounds.


## Custom matching in action ##

Putting everything together, we get a properly working matcher that extends the behavior of Swift's switch statement:

{% highlight swift %}
let myArray = [1, 2, 4, 3]
switch myArray {
case [.FromBeginning, .Literal(0), .Literal(0), .Literal(0)]:
  println("last three elements are 0")
case [.Literal(4), .ToEnd]:
  println("first element is 4, everything else doesn't matter")
case [.Wildcard, .Literal(2), .Wildcard, .Literal(3)]:
  println("4 elements; second element is 2, fourth element is 3")
case [.Wildcard, .Wildcard, .Literal(8)]:
  println("third element (out of 3) is 8")
default:
  println("catchall")
}
{% endhighlight %}

There it is! However, the aesthetics leave something to be desired. It's possible, strictly for fun, to use the `NilLiteralConvertible` protocol and additional custom operators to implement something that actually looks like extensions to the language syntax. I'll leave it up to the imagination as to how this could be done (or you can check out [this gist][link-gist]).

Although this example was fun to build, it isn't terribly practical to use (and I would advise you to think long and hard before trying something like this in a production codebase). Its primary purpose is demonstrating what sorts of strange things you can do with the pattern-matching expression operator.

The [next blog post]({% post_url 2014-12-24-protocols-in-swift %}) in this series will discuss protocols, laying the foundation for generics.


[link-exptr]:       https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Patterns.html#//apple_ref/doc/uid/TP40014097-CH36-XID_1020
[link-advop]:       https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/AdvancedOperators.html#//apple_ref/doc/uid/TP40014097-CH27-XID_84
[link-op]:          http://en.wikipedia.org/wiki/Operator_(computer_programming)
[link-gist]:        https://gist.github.com/austinzheng/0af8aba547b1965d76c1