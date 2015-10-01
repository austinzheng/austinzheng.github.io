---
layout: post
title: "Swift 2: Control Flow and Error Handling"
date:   2015-06-08 18:00:00
tags: swift
---

**[Swift 2.0][link-swiftblog]** was announced today at WWDC.

By far the most important news regarding Swift's future was the announcement that the language will be open-sourced later this year under an OSI-compliant license, with Apple taking it upon themselves to port the compiler and standard library to Linux. The possibilities are limitless - imagine Swift running on embedded systems, powering back-end services, or even serving as a next-generation application language for certain other smartphone platforms.

The next iteration of Apple's fledgling programming language also includes a number of new features, including protocol extensions, novel forms of control flow, and exception-style error handling. This article will discuss the latter two topics. A [future article]({% post_url 2015-09-29-swift-generics-pt-2 %}) will discuss protocol extensions and complete my (long-stalled) series on generics.

## do and repeat-while

The classic [do-while loop][link-dowhile] has been renamed `repeat`-`while`, in order to make way for the new `do` statement. By itself, `do` introduces a [nested scope][link-scope], much like bare braces in C. For example:

{% highlight swift %}
// When run, produces the following output:
// 'protagonist' in inner scope is "Jyaku"
// 'protagonist' in outer scope is "Meela"
func nestedScope() {
  let protagonist = "Meela"
  do {
    let protagonist = "Jyaku"
    print("'protagonist' in inner scope is \"\(protagonist)\"")
  }
  print("'protagonist' in outer scope is \"\(protagonist)\"")
}
{% endhighlight %}

Note the definition of the constant `protagonist` within the nested scope, which shadows the previously-declared constant `protagonist` within the outer scope of the function.


## guard ##

Guards are another new feature. Their purpose is to *validate* some condition, and *force a stop to execution* if that condition is not met. Here is a simple example that illustrates how `guard` works:

{% highlight swift %}
func firstExample(input: Int) {
  // start of guard
  guard inputIsAValidNumber(input) else {
    return
  }
  // end of guard
  performSomeTaskWithInput(input)
}
{% endhighlight %}

The `guard` statement begins with the keyword `guard`, followed by a predicate (an expression that returns a boolean or equivalent), followed by the keyword `else`, and then a block of code comprising the guard's else clause.

When control reaches a guard statement, the predicate is first checked. If the predicate evaluates to false, the else clause is run. The else clause is a required part of the guard statement, and must either exit the scope (for example, by `return`ing from a function) or call a `@noreturn` function.

The overall effect of a guard, thus, is to prevent code from continuing to execute along some default path unless a condition is met. In our example above, `performSomeTaskWithInput()` can only ever be called if `inputIsAValidNumber(input)` returns true.

A more interesting example of guard usage is developed below. The following function takes in a string containing a four-part IPv4 address in [dot-decimal notation][link-dotdecimal], and attempts to convert it into the equivalent numeric value. It returns an `IPResult`, an enum which encapsulates either the value or a failure message.

{% highlight swift %}
enum IPResult {
  case Success(Int)
  case Failure(String)
}

func numericiseIPAddress(ipAddr: String) -> IPResult {
  // Split dot-decimal string into its (presumably) four components
  let components = split(ipAddr.characters) { $0 == "."}
  
  // Ensure split happened correctly
  guard components.count == 4 else {
    return .Failure(components.count < 4 ? "too few components" : "too many components")
  }

  // Get first octet
  guard let firstOctet = Int(String(components[0]))
    where firstOctet >= 0 && firstOctet < 256 else {
      return .Failure("first octet was bad")
  }
  // Get second octet
  guard let secondOctet = Int(String(components[1]))
    where secondOctet >= 0 && secondOctet < 256 else {
      return .Failure("second octet was bad")
  }
  // Get third octet
  guard let thirdOctet = Int(String(components[2]))
    where thirdOctet >= 0 && thirdOctet < 256 else {
      return .Failure("third octet was bad")
  }
  // Get fourth octet
  guard let fourthOctet = Int(String(components[3]))
    where fourthOctet >= 0 && fourthOctet < 256 else {
      return .Failure("fourth octet was bad")
  }
  // Calculate numerical value of IP address, given the values of each octet
  let value = fourthOctet
    + thirdOctet * 256
    + secondOctet * (256*256)
    + firstOctet * (256*256*256)
  return .Success(value)
}

// Example usage:
let rawAddress = "73.170.78.5"
let result = numericiseIPAddress(rawAddress)
switch result {
case let .Success(value): 
  print("IP address \"\(rawAddress)\" is equivalent to the numeric value \(value)")
case let .Failure(f): 
  print("Failed to convert \"\(rawAddress)\" into a numeric value; error is \(f)")
}
{% endhighlight %}

In the above example, the second through fifth guard clauses each unwrap an [optional][link-option] using `let`, syntax shared with the `if`-`let` construct.

However, once the optional is unwrapped successfully, the unwrapped value can be used by subsequent code. For example, `firstOctet`, `secondOctet`, `thirdOctet`, and `fourthOctet` are all unwrapped from a call to a failable initializer, but once unwrapped they can all be used to calculate `value`.

Also note the presence of the keyword `where`, which defines an additional expression which must evaluate to true in order for the predicate to be satisfied.

Guards provide an alternative to deeply nested `if`-`let` constructs or Swift 1.2's compound `if`-`let`. They are especially useful if there is work specific to a failure mode that must be done before the enclosing function can return or the enclosing scope can be exited. The IP address example above uses the else clauses to specify the error that occurred when returning the failure result.

Compare:

{% highlight swift %}
func numericiseIPAddress(ipAddr: String) -> IPResult {
  let components = split(ipAddr.characters) { $0 == "."}
  if components.count == 4 {
    if let firstOctet = Int(String(components[0]))
      where firstOctet >= 0 && firstOctet < 256 {
        if let secondOctet = Int(String(components[1]))
          where secondOctet >= 0 && secondOctet < 256 {
            if let thirdOctet = Int(String(components[2]))
              where thirdOctet >= 0 && thirdOctet < 256 {
                if let fourthOctet = Int(String(components[3]))
                  where fourthOctet >= 0 && fourthOctet < 256 {
                    let value = fourthOctet
                      + thirdOctet * 256
                      + secondOctet * (256*256)
                      + firstOctet * (256*256*256)
                    return .Success(value)
                } else {
                  return .Failure("fourth octet was bad")
                }
            } else {
              return .Failure("third octet was bad")
            }
        } else {
          return .Failure("second octet was bad")
        }
    } else {
      return .Failure("first octet was bad")
    }
  } else {
    return .Failure(components.count < 4 ? "too few components" : "too many components")
  }
}
{% endhighlight %}


## defer ##

`defer` is yet another Swift 1.2 control flow construct. It defines a block of code that is not executed until execution is just about to leave the current scope:

{% highlight swift %}
// When run, produces the following output:
// operation started
// operation complete
// This will be run second-last
// This will be run last
func deferExample() {
  defer {
    print("This will be run last")
  }
  defer {
    print("This will be run second-last")
  }
  print("operation started")
  // ...
  print("operation complete")
}
{% endhighlight %}

Multiple `defer` statements can be defined within a scope. The first `defer` statement that appears in the scope will be the very last one to be executed.

Defer is useful for performing unconditional clean-up, whether or not some operation succeeds or fails. For example, `defer` can be used to specify code that gracefully closes a file handle or network connection, or deallocates a manually allocated buffer (e.g. for C interop).


## Error handling ##

*Check out the [official documentation][link-sberrors] for more information on Swift 2.0's error handling.*

`do`, `guard`, and `defer` all lead into Swift 2.0's new exception-like error handling system (which [is not without its caveats][link-exceptiontweet]). To demonstrate how it works, let's convert the IP address example above to use Swift 2.0 style error handling.

First of all, errors are defined by declaring types that conform to the new `ErrorType` protocol. A type that conforms is a type that can be thrown as an error. Instead of using our `IPResult` type to store either a successful result or an error, let's define a new `IPError` enum whose instances we can throw as errors:

{% highlight swift %}
enum IPError : ErrorType {
  case TooFewComponents
  case TooManyComponents
  case OctetWasBad(Int)   // int is the octet #; 1, 2, 3, or 4
}
{% endhighlight %}

Next, functions, methods, and closures which can throw errors are declared with the `throws` keyword following the parameter list (and preceding the return-type `->`, if present). Only functions declared with this keyword can contain code which throws unhandled errors. Our `numericiseIPAdress()` function can be converted to throw errors as such:

{% highlight swift %}
func numericiseIPAddress(ipAddr: String) throws -> Int {
  let components = split(ipAddr.characters) { $0 == "."}

  // Ensure split happened correctly
  guard components.count == 4 else {
    if components.count < 4 {
      throw IPError.TooFewComponents
    } else {
      throw IPError.TooManyComponents
    }
  }

  // Get first octet
  guard let firstOctet = Int(String(components[0]))
    where firstOctet >= 0 && firstOctet < 256 else {
      throw IPError.OctetWasBad(1)
  }
  // Get second octet
  guard let secondOctet = Int(String(components[1]))
    where secondOctet >= 0 && secondOctet < 256 else {
      throw IPError.OctetWasBad(2)
  }
  // Get third octet
  guard let thirdOctet = Int(String(components[2]))
    where thirdOctet >= 0 && thirdOctet < 256 else {
      throw IPError.OctetWasBad(3)
  }
  // Get fourth octet
  guard let fourthOctet = Int(String(components[3]))
    where fourthOctet >= 0 && fourthOctet < 256 else {
      throw IPError.OctetWasBad(4)
  }
  return fourthOctet
    + thirdOctet * 256
    + secondOctet * (256*256)
    + firstOctet * (256*256*256)
}
{% endhighlight %}

Our function now returns a bare `Int` (not an `Int?` or a result enum), but can also throw an error (in this case, an `IPError`). The `throw` keyword, followed by an instance of an `ErrorType`, throws an error, causing execution to return to the enclosing scope immediately so that the error can be handled.

In order to *catch* errors that are thrown, we use the `do`-`catch` construct. Let's demonstrate this by writing a function which will read in user input from the command line (using Swift 2.0's nifty new `readLine` function), and try to parse it as an IP address:

{% highlight swift %}
func tryNumericisingIP() {
  // Get the value from the user
  print("Enter four-part IP here: > ", terminator: "")
  let maybeInput = readLine(stripNewline: true)

  // Only continue if we actually received a value
  guard let input = maybeInput else {
    return
  }

  // Try numericising the user-provided IP. Handle any errors.
  do {
    let numericValue = try numericiseIPAddress(input)
    print("Your IP address, \(input) is equal to \(numericValue)")
  } catch IPError.TooFewComponents {
    print("IP address had fewer than 4 components")
  } catch IPError.TooManyComponents {
    print("IP address had more than 4 components")
  } catch let IPError.OctetWasBad(octet) {
    print("Octet number \(octet) was invalid - either non-numeric or out of bounds")
  } catch {
    // This shouldn't be necessary, but the compiler complains about 
    //  exhaustiveness. Maybe an early beta seed bug.
    print("Encountered an unknown error \(error)")
  }
}
{% endhighlight %}

The code that is enclosed within the `do` clause is executed. Functions or methods that might throw an error (because they are marked by `throws`) must be invoked with `try`. Errors that are thrown as a result of invoking such a function are then handled by the `catch` clauses. If a `catch` clause doesn't specify a pattern, the error that was thrown is automatically bound to the `error` local variable, as seen above.

There is also a 'force-try' construct, `try!`, which assumes that the function being invoked won't return an exception and requires no catch clauses. If one is returned, the application will terminate with a run-time error. In this way, `try!` is analogous to 'force-downcasting' (`as!`) and 'force-unwrapping' (`myOptionalValue!`).

Note that the catch clauses work very similarly to clauses in the pattern-matching `switch` statement. The compiler attempts to perform exhaustiveness analysis. If your `catch` clauses don't catch every possible error, then the compiler assumes that an uncaught error should bubble up to the next higher scope or function, to be handled there. If our `do`-`catch` construct above wasn't exhaustive, we'd need to declare `tryNumericisingIP()` as `throws`, and have its caller handle any unhandled errors.

`defer` wasn't part of this example, but it's not too hard to see how it could be used in cases where a file needs to be closed, or a network or database connection needs to be torn down, even if an error condition manifests. In this sense it plays much the same role as the [`finally` block][link-javafinally] in a Java-style `try`-`catch`-`finally` construct.


## Grab Bag

To close out this article, here is a discussion of a number of minor control flow improvements that ship with Swift 2.0.


### for-in where clauses

The `for`-`in` loop can now be defined with a `where` clause; the loop will only be run for items that fulfill the where clause's predicate:

{% highlight swift %}
let someNumbers = [10, 20, 30, 40, 50]
for number in someNumbers where number > 20 {
  print("Here is a number greater than 20: \(number)")
}
// Output:
// Here is a number greater than 20: 30
// Here is a number greater than 20: 40
// Here is a number greater than 20: 50
{% endhighlight %}


### case clauses in conditionals

It's a little obscure, but you can use `case` clauses with certain patterns alongside `if`, `while`, `guard`, and `for-in`. The supported patterns are the enumeration patterns, the new optional pattern (`myVar?`, equivalent to `.Some(myVar)`), expression patterns, and the `is`/`as` type-casting patterns.

This means, for example, `switch` is no longer the only way to work with enums with associated values:

{% highlight swift %}
// Associated value enum, representing the preferred way of referring to some entity
enum Identifier {
  case Name(String)
  case Number(Int)
  case None
}

var robotIdentifier : Identifier = .Name("R2-D2")

if case .None = robotIdentifier {
  print("Robot is nameless")
} else if case let .Name(name) = robotIdentifier where name == "R2-D2" {
  print("Luke Skywalker's faithful astromech")
} else if case let .Name(robotName) = robotIdentifier {
  print("Robot is named \(robotName)")
} else if case let .Number(robotID) = robotIdentifier {
  print("Robot's identification number is \(robotID)")
}
{% endhighlight %}

In the following example, a very simple civilization simulator starts its simulation off during a golden age. Every time the `while` loop runs the age is advanced; there is a random chance the golden age continues or decays into a `NormalAge`. The `while` loop allows the simulated civilization to `celebrate()` each iteration of the loop the civilization remains within its golden age.

{% highlight swift %}
protocol AgeType { func nextAge() -> AgeType }

class GoldenAge : AgeType {
  var length : Int = 1

  func nextAge() -> AgeType {
    length += 1
    return arc4random_uniform(9) > 2 ? self : NormalAge()
  }

  func celebrate() {
    print("hooray! fireworks over our capital city (\(length*100) years).")
  }
}

class NormalAge : AgeType {
  func nextAge() -> AgeType { return self }
}

var currentAge : AgeType? = GoldenAge()
while case let thisAge = currentAge as? GoldenAge, case let theAge? = thisAge {
  theAge.celebrate()
  currentAge = theAge.nextAge()
}

// Sample output:
// hooray! fireworks over our capital city (100 years).
// hooray! fireworks over our capital city (200 years).
// hooray! fireworks over our capital city (300 years).
// hooray! fireworks over our capital city (400 years).
{% endhighlight %}

Try out Swift 2.0's control flow features, and see if they make your code more intelligible!


[link-swiftblog]:           https://developer.apple.com/swift/blog/?id=29
[link-dowhile]:             http://en.wikipedia.org/wiki/Do_while_loop
[link-dotdecimal]:          http://en.wikipedia.org/wiki/Dot-decimal_notation
[link-option]:              http://en.wikipedia.org/wiki/Option_type#Swift
[link-scope]:               http://en.wikipedia.org/wiki/Scope_(computer_science)#Block_scope
[link-sberrors]:            https://developer.apple.com/library/prerelease/ios/documentation/Swift/Conceptual/Swift_Programming_Language/ErrorHandling.html
[link-exceptiontweet]:      https://twitter.com/jspahrsummers/status/608066730250924032
[link-javafinally]:         https://docs.oracle.com/javase/tutorial/essential/exceptions/finally.html
