---
layout: post
title: "Learning Haskell, Part 1"
date:   2015-01-06 01:45:00
tags: haskell
---

And now for something completely different! One thing I've been wanting to do for a while is to begin learning [**Haskell**][link-haskell], and I finally got the chance to sit down and play around with it today.

This (series) of blog posts *isn't* intended to be a full-fledged introduction to the language (as if I were even remotely qualified to write such a thing). However, it is meant as a step-by-step exposition as to how I am learning/learned the language. The hope is that reading through these notes might offer a helpful perspective for those looking to learn Haskell themselves.


## What is Haskell? ##

Haskell is a [functional programming][link-funcprog] language. It emphasizes [static typing][link-statictyping], [lazy evaluation][link-laziness], [referential transparency][link-reftrans], and [first-class functions][link-fcfun], among other concepts. What does this mean?

* *Static typing* means that bindings, function parameters, and function return values all have fixed [*types*][link-type] that are known when the program is compiled.
* *Lazy evaluation* means that expressions don't have to be evaluated immediately or fully, but only when their values are actually needed.
* *Referential transparency* means that functions always produce the same output for a given input. Special constructs are used to handle side effects, like I/O.
* Support for *first-class functions* implies that functions can be passed as arguments to other functions or returned by other functions.


## Initial setup ##

Since I'm working on an OS X box, I used [Homebrew][link-homebrew] to install [GHC][link-ghc], which is perhaps the most commonly used Haskell compiler, as well as Cabal, which is a package manager: `brew install ghc cabal-install`. Otherwise, setup directions and installers for Linux, OS X, and Windows can be found [here][link-haskelldl].

Once I installed the tools, I created a working directory. I `cd`ed into that directory from the terminal and started Haskell's REPL by typing `ghci`.

Haskell source files are postfixed with `.hs`. Once in the REPL, you can quit by typing `:quit`. You can load a source file in the current directory using `:load <FILENAME>`. If you change the source file in your text editor, you can reload it using `:reload`.


## The basics ##

In Haskell, line comments are preceded by `--`.

Simple math works the way you'd expect it to:

{% highlight haskell %}
2 + 2
-- = 4

10 * 8
-- = 80
{% endhighlight %}

You have to wrap negative number literals in parentheses to prevent a parsing error:

{% highlight haskell %}
2 + (-3)
-- = -1
{% endhighlight %}

The `\` operator returns a floating-point result:

{% highlight haskell %}
10 / 2
-- = 5.0
{% endhighlight %}

There are functions to perform integer division or the modulo operation. A function that takes two arguments can be wrapped in backticks and used in infix position:

{% highlight haskell %}
10 `div` 3
-- = 3
div 10 3
-- also = 3

10 `mod` 3
-- = 1
mod 10 3
-- also = 1
{% endhighlight %}

There are also the comparison operators `>`, `<`, `>=`, `<=`, `==`, and `/=` (not equal to), as well as the function `not` and the logical operators `&&` (and) and `||` (or). The boolean literals are `True` and `False` (with the initial capital letter).

{% highlight haskell %}
5 > 2
-- = True

True && False
-- = False

not False
-- = True
{% endhighlight %}


## Our first function ##

Create a new Haskell source file. We'll define a function called `addOne` that takes in an integer, and returns that integer plus one:

{% highlight haskell %}
addOne :: Integer -> Integer
addOne n = n + 1
{% endhighlight %}

The first line indicates that we are declaring a function named `addOne`. This function takes in a single `Integer` argument and returns an `Integer`.

The second line indicates that for any input to our function, which we'll call `n`, we will return `n + 1`.

Note that the `=` is being used in its mathematical sense: we are saying that anywhere in our program we have `addOne n`, we can replace it with `n + 1` and get the exact same result. This is an example of referential transparency in action, since our function always returns the same value for a given input.

After saving the source file, go back to the REPL. There, type in `:reload` (or `:load <FILENAME>` if you haven't already). Then:

{% highlight haskell %}
addOne 10
-- = 11
{% endhighlight %}

We invoke a function with its name followed by any arguments, all separated by spaces. No commas or parentheses necessary. (The `mod` example above demonstrates how a two parameter function would be called.)

Note that Haskell supports both `Int` and `Integer` types. An `Integer` can be arbitrarily large, constrained only by your computer's memory. Other important types include `Bool` (booleans) and `String` (a Unicode text string). You can also define types for 2-tuples (and by extension, 3-tuples, 4-tuples...). For example: `(Integer, String)` is a tuple type whose first member is an integer and whose second member is a string.


## Functions and pattern matching ##

Let's write another function. This one will return a last name given a first name:

{% highlight haskell %}
lastName :: String -> String
lastName "anthony" = "gillis"
lastName "michelle" = "jocasta"
lastName "gregory" = "tragos"
{% endhighlight %}

Haskell function definitions rely upon pattern matching. If you invoke the `lastName` function with the argument `"anthony"`, the function will return the string `"gillis"`. If you invoke it with `"michelle"`, it will instead return `"jocasta"`. After calling `:reload` in the REPL:

{% highlight haskell %}
lastName "anthony"
-- = "gillis"
{% endhighlight %}

Excellent. But what if we invoke `lastName` with a first name that isn't one of the three we specified? 

{% highlight haskell %}
lastName "bob"
{% endhighlight %}

We get an exception: `Non-exhaustive patterns in function lastName`. The problem is that our function isn't *total* - it doesn't have a defined output for every possible input. In general, functions should be total whenever possible. Let's redefine our function:

{% highlight haskell %}
lastName :: String -> String
lastName "anthony" = "gillis"
lastName "michelle" = "jocasta"
lastName "gregory" = "tragos"
lastName n = "<unknown>"
{% endhighlight %}

Now our function is total. The last case is a catch-all case - if none of the first three cases match, the last case always binds the parameter to `n` (which is unused). We can also use `_` in lieu of `n` to indicate that we don't care about the value (and this would indeed be preferable, since using it makes our intent more obvious).


## Multiple arguments ##

Let's write a function that takes more than one argument. We're going to write a function, `areAscending`, that takes in three integers and returns True iff they are strictly increasing in value:

{% highlight haskell %}
areAscending :: Integer -> Integer -> Integer -> Bool
areAscending a b c = a < b && b < c
{% endhighlight %}

Our type signature looks strange, doesn't it? Why isn't there a distinction between the parameter types and the return value type? It turns out multi-parameter functions in Haskell are [curried][link-curry]. Currying a function takes a function with multiple parameters and turns it into a series of functions that take a single parameter and return another function. For example, in pseudo-Swift:

`myFunc(a: A, b: B, c: C) -> Z`

becomes

`func1(a: A)`, which when called returns a function `func2(b: B)`, which returns a function `func3(c: C)`, which returns the final result of type `Z`.

Note also in our pattern match that we assign the first parameter to `a`, the second to `b`, and the third to `c`, so we can use them after the `=` to perform our computation.

Calling our function in the REPL:

{% highlight haskell %}
areAscending 1 2 3
-- = True

areAscending 3 4 2
-- = False
{% endhighlight %}

If we wanted to call this function with an expression instead of a literal for one of its arguments, we'd need to wrap it in parentheses, because of the relative precedence of function invocation:

{% highlight haskell %}
areAscending 1 (1 + 1) 3
-- = True
{% endhighlight %}

`areAscending 1 1 + 1 3` is interpreted as `(areAscending 1 1) + (1 3)`, which is meaningless.


## Guards on pattern matching ##

Sometimes pattern matching alone is insufficient to describe a function. A pattern can be given *guards*. If the pattern matches, each guard's test expression is checked in order, and if the test expression matches that guard's value is used.

Let's code up a helper function for the otherwise execrable [FizzBuzz][link-fizzbuzz] interview question. This function will take in an integer and return one of four strings: "fizzbuzz" if the number is divisible by both 3 and 5, "fizz" if it is divisible by 3, "buzz" if it is divisible by 5, or "" otherwise.

{% highlight haskell %}
fizzBuzzHelper :: Integer -> String
fizzBuzzHelper n
  | n `mod` 3 == 0 && n `mod` 5 == 0 = "fizzbuzz"
  | n `mod` 3 == 0 = "fizz"
  | n `mod` 5 == 0 = "buzz"
  | otherwise = ""
{% endhighlight %}

Guards are associated with a single case. They start with `|`, followed by a test expression that returns True or False, followed by `=`, followed by the value to return. The catch-all `otherwise` handles any case the previous guards don't catch.

Back in the REPL...

{% highlight haskell %}
fizzBuzzHelper 33
-- = "fizz"

fizzBuzzHelper 15
-- = "fizzbuzz"
{% endhighlight %}


## Zero-argument functions ##

At this point, we know how to define functions of one or more arguments, and how to use pattern matching and guards to return a value of our choosing. What about zero-argument functions?

Given what we know about functions, we might guess that a zero-argument function is a 'function' that always returns a single value. This is the principle of referential transparency again: since there's only one way to call a function with no arguments, that function can only ever return a single value:

{% highlight haskell %}
someValue :: String
someValue = "hello world"
{% endhighlight %}

Here's a zero-argument function. Zero-argument functions are exactly the same as constants. Remember that referential transparency means that anywhere we see `someValue`, we can replace it with `"hello world"`, which is exactly what we'd expect from a constant.


## Lists ##

At this point along our journey into the wonderful world of Haskell, we can begin looking at a data type of fundamental importance: the `List`. A list represents a sequence of items.

Haskell's lists are typed. The type of a list of `Integer` is `[List]`. The type of a list of `String` is `[String]`. List literals are declared as such: `[1,2,3,4,5]`. The commas are mandatory, and the last element can't be followed by a comma. The empty list is just `[]`.

What can we do with lists? A non-exhaustive list of possibilities follows:

### Cons ###

We can add an element to the head of a list:

{% highlight haskell %}
1 : [2,3]
-- = [1,2,3]
{% endhighlight %}

In fact, `[1,2,3]` is syntactic sugar for `1:2:3:[]`. If you are familiar with Lisp, you may recognize this form as how lists are put together in Lisp.

### Concat ###

We can concatenate two lists together:

{% highlight haskell %}
[1,2,3] ++ [4,5,6]
-- = [1,2,3,4,5,6]
{% endhighlight %}


### Length ###

Use `length` to get the length of a list:

{% highlight haskell %}
length [1,2,3,4]
-- = 4
{% endhighlight %}


### Deconstruct ###

Perhaps most importantly, we can deconstruct a list using pattern matching. Let's write a function to 'dismember' a list of strings - given a list of strings, it'll return a tuple consisting of the first `String` element as well as the rest of the list:

{% highlight haskell %}
dismember :: [String] -> (String, [String])
dismember [] = ("", [])
dismember (x:xs) = (x, xs)
{% endhighlight %}

Note our pattern match! We are defining our function such that it binds `x` to the first element in the list, and `xs` to the list starting with the second element. Also note our base case with `[]`: without it, our function isn't total, since we can't destructure an empty array.

{% highlight haskell %}
dismember ["foo","bar","baz","qux"]
-- = ("foo",["bar","baz","qux"])
{% endhighlight %}

We can also nest this sort of pattern. Let's write an even more violent function, 'eviscerate', that takes a list of strings and returns a 3-tuple containing the first two elements and the rest of the list:

{% highlight haskell %}
eviscerate :: [String] -> (String, String, [String])
eviscerate [] = ("", "", [])
eviscerate (x1:(x2:xs)) = (x1, x2, xs)
eviscerate (x1:xs) = (x1, "", xs)
{% endhighlight %}

Note our third case, which handles the case in which the list has only one element. If we switch the second and third cases, the third case will actually never be invoked (because any non-empty list will match the `(x1:xs)` case), and in fact the compiler will warn us.

{% highlight haskell %}
eviscerate ["foo","bar","baz","qux"]
("foo","bar",["baz","qux"])
{% endhighlight %}


## No `nil`? ##

If you've been following along closely, you may be ready to eviscerate me for using `""`, `"<unknown>"`, and other such values as placeholders for nothing, instead of something like `nil`.

After all, what if we called `eviscerate` on a list containing empty strings, rather than an empty list? Indeed, using these placeholder values is generally bad programming practice, and only done here to demonstrate the other concepts more easily. 

In fact, Haskell doesn't have `nil`. Instead, it has a special `Maybe` type. `Maybe` shows up in Swift as the `Optional` type, one of a surprisingly large number of ideas Swift 'borrowed' from Haskell. (Fun fact: early versions of the Swift standard library headers had a comment that referred to Haskell by name.)


## Recursion ##

Up to this point, we have not declared a single variable whose value we mutated, nor have we written any imperative-style `for` loops. The functional style comes with its own tools for repetition, such as the map and fold higher-order functions and recursion. For now, we'll look at recursion.

[Recursion][link-recur] is a technique in which a problem is solved in terms of smaller versions of the same problem. More concretely, we use recursion when a function calls itself with different arguments to continue solving a problem. At some point our function decides that it's reached the 'bottom', and returns an actual value instead of calling itself again. This 'bottom' of the problem is often referred to as the *base case*.

Here is a very simple example of recursion, the oft-seen naive method of generating [Fibonacci numbers][link-fib]:

{% highlight haskell %}
fib :: Integer -> Integer
fib 0 = 1
fib 1 = 1
fib n = fib (n - 1) + fib (n - 2)
{% endhighlight %}

Our base cases are `fib 0` and `fib 1`. Otherwise, if our single parameter is larger than 1, we calculate `fib n` in terms of `fib` itself, only invoked with smaller values. Even if it's not immediately obvious what the sequence of function calls is, hopefully it makes sense that eventually the recursion will stop because the arguments become small enough to reach the base cases.

Another example: what if we wanted a `countdown` function to sum up all the numbers counting down from some starting value? For example, `countdown 5` would return 5 + 4 + 3 + 2 + 1 = 15.

If we were implementing this in C we'd probably use a for loop and an accumulator variable:

{% highlight c %}
// Precondition: x > 0
int countdown(int x) {
  int acc = x;
  for (int i=x-1; i>0; i--) {
    acc += i;
  }
  return acc;
}
{% endhighlight %}

In Haskell, though, we would eschew such a loop in favor of recursion:

{% highlight haskell %}
countdown :: Integer -> Integer
countdown n = helper n 0

helper :: Integer -> Integer -> Integer
helper 0 acc = acc
helper n acc = helper (n - 1) (acc + n)
{% endhighlight %}

In this example, `helper` does all the work. `helper` takes a value to count down from, as well as an accumulated total, and does one of two things: either it returns `acc` (base case), or it calls `helper` recursively with a smaller countdown value and an updated accumulator.

The `countdown` function itself simply calls `helper` with the proper initial conditions (i.e. an accumulator of 0).

Note how we replaced our mutable variable with an argument to our recursive function. Very broadly, we can consider each call of a recursive function equivalent to a loop iteration in an iterative solution. Working values that mutate in an iterative solution are, in the recursive solution, instead transformed and passed as arguments to the recursive call.

Even though recursion and iteration are equivalent in power, it turns out that recursion isn't often the best abstraction. There are higher-level tools, such as the previously-mentioned map and fold. But recursion is there if you need it.

Recursion can be used to work with lists, especially when used in conjunction with list destructuring. For example, what if we wanted a function that took in a list of integers and returned a new list with any values greater than 10 replaced by their squares? (Also, we can't use `map`.)

{% highlight haskell %}
squareOverTens :: [Integer] -> [Integer]
squareOverTens [] = []
squareOverTens (n:xn)
  | n > 10 = (n * n) : squareOverTens xn
  | otherwise = n : squareOverTens xn 
{% endhighlight %}

Here it is. We use a combination of pattern matching on lists `(n:xn)` and recursion to build up the result list one element at a time. Our first pattern is our base case, while our second pattern contains two guards that return different values based on whether `n` is greater than 10 or not.

In the REPL:

{% highlight haskell %}
squareOverTens [1,2,11,3,4,12]
-- = [1,2,121,3,4,144]
{% endhighlight %}


## Personal notes ##

I've only so far learned a very tiny bit of what Haskell has to offer, but I already like what I’ve seen so far. I’ve tried some of these techniques before in Scala and Swift, but Haskell feels refreshingly clear and concise, purpose-built for this style of programming. The fact that functions are referentially transparent makes reasoning about what they do easier, and faciliates building programs up in a principled fashion from smaller pieces. There is no need to worry that calling a function is somehow going to change hidden state whose effects don't show up until much later.

By the way, if you've read this far and you've noticed an error, something that is unclear, or something I am not doing in an idiomatic fashion, please do drop me a note! Being given the opportunity to fix a mistake would definitely make my day.


[link-haskell]:          https://www.haskell.org/
[link-statictyping]:     http://en.wikipedia.org/wiki/Type_system#Static_type-checking
[link-type]:             http://en.wikipedia.org/wiki/Data_type
[link-laziness]:         http://en.wikipedia.org/wiki/Lazy_evaluation
[link-reftrans]:         http://en.wikipedia.org/wiki/Referential_transparency_(computer_science)
[link-fcfun]:            http://en.wikipedia.org/wiki/First-class_function
[link-funcprog]:         http://en.wikipedia.org/wiki/Functional_programming
[link-homebrew]:         http://brew.sh/
[link-ghc]:              http://en.wikipedia.org/wiki/Glasgow_Haskell_Compiler
[link-haskelldl]:        https://www.haskell.org/platform/
[link-curry]:            http://en.wikipedia.org/wiki/Currying
[link-fizzbuzz]:         http://c2.com/cgi/wiki?FizzBuzzTest
[link-recur]:            http://en.wikipedia.org/wiki/Recursion_(computer_science)
[link-fib]:              http://en.wikipedia.org/wiki/Fibonacci_number
