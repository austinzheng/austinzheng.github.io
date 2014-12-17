---
layout: post
title: "Swift's pattern-matching switch statement"
date:   2014-12-16 22:00:00
tags: swift
---

*Part 2 of a series on Swift enums, pattern matching, and generics. [Previous post.]({% post_url 2014-12-16-swift-enums %})*

*Parts of this blog post are adapted from a [talk I gave](http://realm.io/news/swift-enums-pattern-matching-generics/) at the Swift Language Users Group.*

Swift includes support for [**pattern matching**][link-pm] through the `switch` control-flow construct. Like in C or Objective-C, the switch construct attempts to match an input expression to one of several cases, executing code corresponding to the first matching case.

## Switch statement basics ##

A simple switch statement follows:

{% highlight swift %}
let value = 10
switch value {
case 10: println("ten")         // this is printed
case 20: println("twenty")
case 30: println("thirty")
default: println("some other number")
}
{% endhighlight %}

The switch statement above looks very similar to a switch statement that one might find in C. However, there are several key differences:

* The test expression, in this case `value`, does not need to be surrounded by parentheses.
* There is a `default` catch-all clause. In Swift, switch statements **must be exhaustive**.
* There is no use of the keyword `break`, although it exists and can be used. In Swift, only one branch is executed by default; Swift does **not** carry over C's fallthrough-by-default behavior. Fallthrough behavior must be manually specified using `fallthrough`.

Switch branches cannot be empty; use `break` to indicate that nothing should be done for a given branch.

## Switching on equatable values ##

Swift's switch statement can match not only numbers, but any equatable type (such as strings):

{% highlight swift %}
let stateCode = "CA"
switch stateCode {
case "CA": println("California")    // this is printed
case "DE": println("Delaware")
case "PA": println("Pennsylvania")
case "TX": println("Texas")
default: println("Another state...")
}
{% endhighlight %}

## Switching on ranges ##

Swift can also match numeric expressions to ranges (constructed using `...` or `..<`):

{% highlight swift %}
let v = 15
switch abs(v) {
case 0...9: println("single digit")
case 10...99: println("double digits")
case 100...999: println("triple digits")
default: println("four or more digits")
}
{% endhighlight %}

## Switching on tuples ##

When the predicate expression is a tuple type, Swift matches each part of the tuple separately:

{% highlight swift %}
let person = ("Helen", 25)
switch person {
case ("Helen", let age):
  println("You are Helen, and you are \(age) years old")
case (_, 13...19):
  println("You are a teenager, not named Helen")
case ("Bob", _):
  println("You are not a teenager, but your name is Bob")
case (_, _):
  println("No comment")
}
{% endhighlight %}

A few things to note:

* The `_` can be used as a 'don't care' marker. For example, the second case will ignore the first tuple element (the person's name).
* The final expression, `(_, _)`, functions as a catch-all and is exactly equivalent to `default` in this example.

## Binding values in cases ##

As seen in the previous section, the `let` keyword can be used in patterns to create a temporary binding for a value. For example:

{% highlight swift %}
let myTuple = ("abcd", 1234)
switch myTuple {
case let (x, y):
  println("the string 'x' is \(x), the integer 'y' is \(y)")
}
{% endhighlight %}

Note that a `let`-binding pattern always succeeds. This is why our single case above is sufficient to make the switch statement exhaustive. As well, if we only care about binding some of the elements in the tuple, we can use `_` to ignore the ones we don't care about.

The following example is exactly equivalent, since `let` distributes over the patterns in the tuple:

{% highlight swift %}
let myTuple = ("abcd", 1234)
switch myTuple {
case (let x, let y):
  println("the string 'x' is \(x), the integer 'y' is \(y)")
}
{% endhighlight %}

Finally, it is also possible to use `var` in lieu of `let`, allowing you to change the value of the symbol. (This is generally a bad idea, though.)

## Switching on enums ##

Switch statements are an easy way to work with basic and raw-valued enums, and the only real way to work with enums with associated values. Switching on enums is quite straightforward:

{% highlight swift %}
enum Direction {
  case North, South, West, East
}
let myDir = Direction.West
switch myDir {
case .North: println("you're going north") 
case .South: println("you're going south")
case .West: println("you're going west")
case .East: println("you're going east")
}
{% endhighlight %}

Notice that we didn't need to fully qualify each case (e.g. `case Direction.North`). Swift's type inference knows that the predicate expression is of type `Direction`, so we can use the abbreviated syntax for specifying each case.

If we have enums with associated values, we can use `let`-binding to get at the values:

{% highlight swift %}
enum ParseResult {
  case Coordinates(Int, Int)
  case Error(String)
}
let result = ParseResult.Coordinates(12, 15)
switch result {
case let .Coordinates(x, y):
  println("Successful parse. Coords are (\(x), \(y)).")
case .Error(let err):
  println("Failed parse. Error is \(err)")
}
{% endhighlight %}

Note again the two ways the `let`-binding can be set up.

## Switching on subtypes ##

If the switch predicate's type is a class, it's possible to use `is` and `as` to switch on subtypes. For example:

{% highlight swift %}
let view : UIView = getView() // get a view...
switch view {
case is UIImageView:
  println("It's an image view")
case let label as UILabel:
  println("It's a label")
case let tblv as UITableView:
  let sectionCount = tblv.numberOfSections()
  println("It's a table view with \(sectionCount) sections")
default:
  println("It's some other UIView or subclass")
}
{% endhighlight %}

`is` just checks the type, while `as` is used along with `let` to downcast the predicate. For example, even though `view` is a `UIView`, `tblv` is a `UITableView`, so we can call methods and properties specific to that subclass on `tblv`.

Note that the cases must describe subclasses of the predicate's type. For example, the compiler is smart enough to reject a case like `case is UIView`, since that case will always match.

## where guards ##

Any case can be further qualified with a guard clause. The guard clause follows the case's pattern, and is comprised of the keyword `where` followed by an expression that evaluates to true or false. Here is an example of a switch statement where all matching logic is carried out through guard clauses:

{% highlight swift %}
let view : UIView = getView() // get a view...
switch view {
case _ where view.frame.size.height < 50:
  println("This view is shorter than 50 units")
case _ where view.frame.size.width > 20:
  println("This view is at least 50 units tall, and more than 20 units wide")
case _ where view.backgroundColor == UIColor.greenColor():
  println("This view is at least 50 units tall, at most 20 units wide, and green")
default:
  println("This view can't be described by this example")
}
{% endhighlight %}

Unfortunately, guard clauses tend to force you to include a `default` case, since the exhaustiveness checker cannot really reason about the logic within the guard clauses.

## Switch expressions ##

Unlike similar switch constructs in languages such as Rust or Scala, Swift's switch statement can't be used as an expression.

This is inconvenient, especially when you wish to use a switch statement to initialize a constant to one of several values. Normally, you would have to declare a `var` instead, give it a default value, and then set it to the correct value using the switch statement. This is ugly and can obscure your intent.

However, there exists a workaround. You can wrap a switch statement in a closure and call that closure, simulating a proper switch expression. An example follows:

{% highlight swift %}
let name = "blue"
let color : UIColor = {
  switch name {
  case "red": return UIColor.redColor()
  case "green": return UIColor.greenColor()
  case "blue": return UIColor.blueColor()
  case "yellow": return UIColor.yellowColor()
  default: return UIColor.clearColor()
  }
}()
// color is now UIColor.blueColor()
{% endhighlight %}

We declare the constant `color` of type `UIColor`. We then initialize it to the result of running an anonymous closure (` = { ... }()`). Finally, we write code in the closure to return one of several colors depending on the value of `name`. This closure's type is inferred to be `() -> UIColor`, so running it produces a `UIColor` value.

Thank you for reading this rather long post about Swift's pattern-matching switch statement! The next blog post in this series will discuss customizing the pattern-matching system.

[link-pm]:      https://developer.apple.com/library/ios/documentation/Swift/Conceptual/Swift_Programming_Language/Patterns.html
