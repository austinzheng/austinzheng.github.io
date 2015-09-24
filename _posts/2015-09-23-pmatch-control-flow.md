---
layout: post
title: "Swift 2: Control Flow Pattern Matching Examples"
date:   2015-09-23 23:00:00
tags: swift
---

This article aims to provide examples of the expanded pattern matching support which ships with Swift 2.

## Pattern matching using `if` and `guard`

New in Swift 2 is support for pattern matching within `if` statements (and `guard` statements). Let's first define a simple enum:

{% highlight swift %}
// A generic 'number' type that can represent either an integer, a 
// floating-point value, or a boolean.
enum Number {
  case IntegerValue(Int)
  case DoubleValue(Double)
  case BooleanValue(Bool)
}
{% endhighlight %}


### Checking for a certain case

Use case: we want to check whether a value corresponds to a certain case. This works regardless of whether the case has associated values or not, but does not unbox the value.

{% highlight swift %}
let myNumber : Number = // ...

if case .IntegerValue = myNumber {
  print("myNumber is an integer")
}
{% endhighlight %}

The *pattern*, `case .IntegerValue`, comes first, and the value to be matched, the variable `myNumber`, comes after the equals sign.

The equivalent Swift 1.2 version, using `switch`, follows:

{% highlight swift %}
switch myNumber {
  case .IntegerValue: print ("myNumber is an integer")
  default: break
}
{% endhighlight %}


### Unboxing an associated value

Use case: we want to check whether a value corresponds to a certain case, and if so we want to extract the associated value(s).

{% highlight swift %}
if case let .IntegerValue(theInt) = myNumber {
  print("myNumber is the integer \(theInt)")
}
{% endhighlight %}

The pattern now becomes `case let .IntegerValue(theInt)`. The value to be matched remains the same as in the previous example.

Here is an example of the same concept, but applied to `guard`. The semantics of the predicates for `guard` and `if` are identical, and so pattern matching works exactly the same way.

{% highlight swift %}
func getObjectInArray<T>(array: [T], atIndex index: Number) -> T? {
  guard case let .IntegerValue(idx) = index else {
    print("This method only accepts integer arguments!")
    return nil
  }
  return array[idx]
}
{% endhighlight %}


### Qualifying with `where` clauses

As with any `case` in a `switch` statement, a `where` clause can optionally follow a pattern to provide additional constraints. Let's modify the `getObjectInArray:atIndex:` function from the previous example:

{% highlight swift %}
func getObjectInArray<T>(array: [T], atIndex index: Number) -> T? {
  guard case let .IntegerValue(idx) = index where idx >= 0 && idx < array.count else {
    print("This method only accepts integer arguments that are in bounds!")
    return nil
  }
  return array[idx]
}
{% endhighlight %}


### Complex `if` predicates

Swift's `if` statement is surprisingly capable. An `if` statement can have multiple predicates, separated by commas. Predicates fall into one of three categories:

* **Simple logical test** (e.g. `foo == 10 || bar > baz`). There can only be one such predicate, and it must be placed first.
* **Optional unwrapping** (e.g. `let foo = maybeFoo where foo > 10`). If an optional unwrapping predicate immediately follows another optional unwrapping predicate, the `let` can be omitted. Can be fitted with a `where` qualifier.
* **Pattern matching** (e.g. `case let .Bar(String) = theValue`), as discussed above. Can be fitted with a `where` qualifier.

Predicates are evaluated in the order they are defined, and no predicates after a failing predicate will be evaluated. Here is a (contrived) example of a complicated `if` statement.

{% highlight swift %}
let a : Int = // ...
let b : Int = // ...
let firstValue : Number? = // ...
let secondValue : Number? = // ...

if a != b,
  let firstValue = firstValue,
  secondValue = secondValue,
  case let .IntegerValue(first) = firstValue,
  case let .IntegerValue(second) = secondValue where second > first {
    print("a and b are different, and secondValue is greater than firstValue")
    print("a + b + firstValue + secondValue = \(a + b + first + second)")
}
{% endhighlight %}

As always, prioritize intelligibility over cleverness when building such constructs.


## Pattern matching using `for`-`in`

Pattern matching can be used in conjunction with the `for`-`in` loop. In this case the intent is to iterate over the items within a sequence, but only those items which match the given pattern. An example follows:

{% highlight swift %}
enum EngineeringField : String {
  case Civil, Mechanical, Electrical, Chemical, Nuclear
}

enum Entity {
  case EngineeringStudent(name: String, year: Int, dept: EngineeringField)
  case HumanitiesStudent(name: String, year: Int, dept: String)
  case EngineeringProf(name: String, dept: EngineeringField)
  case HumanitiesProf(name: String, dept: String)
}

// A list of entities currently present in the classroom.
var currentlyPresent = [
  Entity.EngineeringProf(name: "Alice", dept: .Mechanical),
  Entity.EngineeringStudent(name: "Belinda", year: 2016, dept: .Mechanical),
  Entity.EngineeringStudent(name: "Charlie", year: 2017, dept: .Chemical),
  Entity.HumanitiesStudent(name: "David", year: 2017, dept: "English Literature"),
  Entity.HumanitiesStudent(name: "Evelyn", year: 2018, dept: "Philosophy"),
  Entity.EngineeringStudent(name: "Farhad", year: 2017, dept: .Mechanical)
]

// Only iterate over the engineering students
for case let .EngineeringStudent(name, _, dept) in currentlyPresent {
  print("\(name), who studies \(dept) Engineering, is present.")
}
/* Output:
Belinda, who studies Mechanical Engineering, is present.
Charlie, who studies Chemical Engineering, is present.
Farhad, who studies Mechanical Engineering, is present.
*/
{% endhighlight %}

In the `for`-`in` predicate, the pattern each element is compared against precedes the `in` keyword, and the sequence to iterate over follows.

Note that, as with patterns in `switch` statements, we can unwrap multiple associated values, and ignore those we don't care about using `_`. We could, if necessary, also add a `where` clause with additional conditions to further constrain which elements we iterate over.


## Pattern matching using `while`

Pattern matching can also be used in conjunction with the `while` loop. In this case the intent is to continue iterating so long as some value in the predicate successfully matches a pattern. To wit:

{% highlight swift %}
enum Status {
  case Continue(Int)
  case Finished
}

func doSomething(input: Int) -> Status {
  if input > 5 {
    return .Finished
  } else {
    // Do work
    print("Doing work with input \(input)")
    return .Continue(input + 1)
  }
}

var latestStatus = doSomething(1)
while case let .Continue(nextInput) = latestStatus {
  latestStatus = doSomething(nextInput)
}
/* Output:
Doing work with input 1
Doing work with input 2
Doing work with input 3
Doing work with input 4
Doing work with input 5
*/
{% endhighlight %}

Note that the compound predicates described above in *Complex if predicates* are also supported by the `while` loop, including use of `where` clauses.


## Advanced patterns

A brief overview of other patterns supported by the pattern matching mechanism follows.

### Optional pattern

The new optional pattern, `myValue?`, is equivalent to `.Some(myValue)` for optionals. (Note that you can use these patterns within a `switch` statement's `case` as well, like with all other patterns.)

{% highlight swift %}
// Refer to the 'Number' enum from before.
let myValue : Number? = // ...

if case .IntegerValue? = myValue {
  print("myValue was not nil, it is some unspecified integer")
}

if case let .IntegerValue(theInt)? = myValue {
  print("myValue was not nil, and it contained the integer \(theInt)")
}
{% endhighlight %}


### Nested enumerations

As before, patterns can be nested, including enumeration patterns. The above examples could also be written as:

{% highlight swift %}
if case .Some(.IntegerValue) = myValue {
  print("myValue was not nil, it was some unspecified integer")
}

if case let .Some(.IntegerValue(theInt)) = myValue {
  print("myValue was not nil, and it contained the integer \(theInt)")
}
{% endhighlight %}


### Type casting patterns

Finally, `is` and `as` can be used to check if an item of some protocol type is actually a specific, conforming concrete type, or if an item of some class type is actually an instance of a subclass.

{% highlight swift %}
class Animal {
  var age : Int
  init(age: Int) { self.age = age }
}

class Cat : Animal {
  var hasFur : Bool, name : String
  init(age: Int, name: String, hasFur: Bool) {
    self.hasFur = hasFur
    self.name = name
    super.init(age: age)
  }
}

class Dog : Animal { /* ... */ }

enum Thing {
  case SomeAnimal(Animal)
  case SomeVegetable
  case SomeMineral
}

let someThing : Thing = // ...

if case .SomeAnimal(is Cat) = someThing {
  print("someThing is an Animal, specifically a Cat")
}

if case let .SomeAnimal(theCat as Cat) = someThing {
  print("someThing is a \(theCat.age)-year-old Cat named \(theCat.name)")
}
{% endhighlight %}
