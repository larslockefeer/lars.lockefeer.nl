---
layout: post
title:  "Map, filter and reduce in Swift"
date:   2015-06-17 19:42:18 +0100
categories: posts
---

Having worked with Swift full-time for over half a year, I feel that now is a good time to start sharing bits and pieces on how to write Swift code instead of Objective-C code that happens to be syntactically correct Swift. We'll start off easy today with an overview of `map`, `filter` and `reduce`, three nifty little methods that greatly reduce the amount of `for`-loops we write from day to day in Swift.

Having mentioned `for`-loops, you probably already guessed that `map`, `filter` and `reduce` can be used to perform operations on collections. Let's have a look at how these functions behave on a very simple collection: an array with elements of type `T`.

To start off our example, let's define an array of integers:

{% highlight swift %}
let numbers = [1, 2, 3, 4, 5]
{% endhighlight %}

## Map

`map` can be used to transform a collection. It takes a function `t: T -> U` and returns a collection containing the results of calling `t` on each element of the collection. We could use this method to multiply all elements in our array by 2:

{% highlight swift %}
let multipliedNumbers = numbers.map { elem in
  return elem * 2
}
// multipliedNumbers = [2, 4, 6, 8, 10]
{% endhighlight %}

Using Swift's syntactical sugar, we can make a few optimizations here. First of all, the parameter that is passed to the function `t` can be referred to with the shorthand syntax `$0`. Second, since `t` only consists of a single line, we can omit the `return` statement, yielding the following one-liner:

{% highlight swift %}
let multipliedNumbers2 = numbers.map { $0 * 2 }
{% endhighlight %}

The full signature of the `map` function is: `map(t: T -> U) -> [U]`. Note that the type of the resulting collection is not equal to the type of the collection it received as input. So rather than transforming integers into integers as we did in our example, we could transform them into any other type.

### Filter

`filter` can be used to filter elements out of a collection depending on a condition. It takes a function `i: T -> Bool` and returns a collection containing those results in the original collection for which `i` returns `true`. Hence, the full signature of the `filter` function is: `filter(i: T -> Bool) -> [T]`.

Now, let's use that to filter our collection by keeping only the even numbers:

{% highlight swift %}
let evenNumbers = numbers.filter { $0 % 2 == 0 }
// evenNumbers = [2, 4]
{% endhighlight %}

### Reduce

With the `reduce` method, all values in a collection can be combined into a single value. The function takes two parameters: the initial value `i: U` and a function `c: (U, T) -> U` that takes the accumulated value and an element of the collection as a parameter. The full signature of the `reduce` method is: `reduce(i: U, c: (U, T) -> U) -> U`.

Let's use this method to calculate the sum of all elements in our array of numbers:

{% highlight swift %}
let sum = numbers.reduce(0) { $0 + $1 }
// sum = 15
{% endhighlight %}

It may not be immediately clear what is happening here, so let's look at the first two iterations. At the first pass, `c` is called with the initial value 0 and the first element of the array (1) as parameters. By definition, it calculates the sum of these 2: `0 + 1 = 1` which is then passed on to the next iteration. In this next iteration, `c` will again be executed, this time with the accumulated value (1) and the second element (2) as parameters, returning the sum: `1 + 1 = 2`. This procedure continues until the last element of the array is reached, at which point the accumulated value will simply be returned.

### Chaining the methods together

The full power of `map`, `filter` and `reduce` becomes especially apparent when these methods are chained together. Have a look at the following example:

{% highlight swift %}
let someNumbers = [1, 2, 3, 4, 5]
let result = someNumbers.filter {
  $0 % 2 == 0
}.map {
  $0 + 10
}.reduce(0) {
  $0 + $1
}
{% endhighlight %}

Here we remove the odd numbers from the array with `filter`. Then, we offset each of them  by 10 and calculate the sum of all elements. Verify for yourself that, after executing these chain of methods, `result` should be set to 26.

That's it for today! This post is also available as a [playground](/uploads/2015/map-filter-reduce.playground.zip). Stay tuned for more Swifty bits and pieces.
