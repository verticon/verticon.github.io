---
layout: post
title: "What Does FlatMap Flatten and What Does Flatten Mean?"
date: 2020-11-02 19:37:48 -0500
comments: true
categories: [swift standard library, sequences, higher order functions]
keywords: flatMap,map
description: What does flatMap do and how is it different from map?
---
FlatMap, and its close cousin map, are two of the higher order functions (i.e. functions that accept and/or return other functions) that are declared by the Sequence protocol. Today I'm going to share what I have learned about flatMap. I'll address the questions: What does flatMap flatten? What does flatten mean?

<!-- more -->

## Let's Begin With a Look at Map

If you are like me then you will find that map is a bit easier to understand than flatMap. Let's start by having a look at map. Here is its declaration:

   `func map<T>(_ transform: (Self.Iterator.Element) throws -> T) rethrows -> [T]`

Let's break this signature down:
* Map is a generic method with one type parameter T  
* The map method declares one parameter; a function with single parameter whose type is the type of the elements of the Sequence upon which map is being invoked and whose return type is T.  
* Map returns a value of type `Array<T>`

How does the compiler resolve the type parameter? T is resolved to whatever type is returned by the transform function. The examples below will demonstrate this. Here are the steps that map performs:

1. Iterate through the sequence's elements.
2. Pass each element in turn to the transform function.
3. Append the return value of the transform function to an array.
4. Return the array.

An example is worth a thousand words, right? Let's have a few.  

##### Map Example 1

    let sequence1 = [1, 2, 3, 4]
    func transform1(arg: Int) -> Int { print("Arg = \(arg)"); return 2 * arg }
    let result1 = sequence1.map(transform1)
    print("Result is \(type(of: result1)) = \(result1)\n")
    // Prints:
    //    Arg = 1
    //    Arg = 2
    //    Arg = 3
    //    Arg = 4
    //    Result is Array<Int> = [2, 4, 6, 8]

Each `Int` in sequence1 is passed to transform1; hence transform1 must be declared to take an `Int`. transform1 returns it argument multiplied by two. The type of the result is an array of the type returned by transform 1, `Array<Int>`.

##### Map Example 2

Let's see what happens if the transform function instead returns a String.

    func transform2(arg: Int) -> String { print("Arg = \(arg)"); return String(arg) }
    let result2 = sequence1.map(transform2)
    print("Result is \(type(of: result2)) = \(result2)\n")
    // Prints:
    //    Arg = 1
    //    Arg = 2
    //    Arg = 3
    //    Arg = 4
    //    Result is Array<String> = ["1", "2", "3", "4"]

Now the type of the result is `Array<String>`. Easy, yes? Let's do one more.

##### Map Example 3

    let sequence3 = [[1, 2], [3, 4]]
    func transform3(arg: [Int]) -> Double { print("Arg = \(arg)"); return Double(arg.reduce(0){ return $0 + $1 }) }
    let result3 = sequence3.map(transform3)
    print("Result is \(type(of: result3)) = \(result3)\n")
    // Prints:
    //    Arg = [1, 2]
    //    Arg = [3, 4]
    //    Result is Array<Double> = [3.0, 7.0]

Here the elements of the sequence are `Array<Int>` so that is what the transform function receives. Transform3 returns the sum of the array elements as a double (I know; I used reduce without explaining it but WTH). The type of the result is thus `Array<Double>`

Okay, enough of map. Lets bring on the real protagonist of our story, flatMap.

## On to FlatMap

How does flatMap differ from map? During our visit with map I listed the four steps that map performs. FlatMap does something different on step 3:

1. (same)
2. (same)
3. FlatMap performs some additional processing (flattening) of the value returned by the transform function and appends the result of that processing to an array.
4. (same)

So, we can now answer the question "What does flatMap flatten?" - it flattens the value returned by the transform function. We'll soon see what flatten means. [The efficacy  of the prefix "flat" (and my use of flatten) in conveying what flatMap does is, I think, subjective. As we shall see, it is a reasonable choice]

Unlike map, flatMap is overloaded. Here are the declarations:

  **Overload 1**

  `func flatMap<SegmentOfResult>(_ transform: (Self.Iterator.Element) throws -> SegmentOfResult) rethrows -> [SegmentOfResult.Iterator.Element] where SegmentOfResult : Sequence`

  **Overload 2**

  `func flatMap<ElementOfResult>(_ transform: (Self.Iterator.Element) throws -> ElementOfResult?) rethrows -> [ElementOfResult]`


Let's start by looking at Overload 1. I'm going to change the name of the type parameter from SegmentOfResult to T so that we can more readily compare it to map's declaration:

`func map<T>(_ transform: (Self.Iterator.Element) throws -> T) rethrows -> [T]`

`func flatMap<T>(_ transform: (Self.Iterator.Element) throws -> T) rethrows -> [T.Iterator.Element] where T : Sequence`

The two declarations are similar. What Overload 1 does is to distinguish the case wherein the value returned by the transform function is itself a `Sequence`. In this case, what is appended to the result array are each of the individual elements of that sequence rather than the sequence itself - the sequence will be **flattened**

Now for Overload 2:

`func map<T>(_ transform: (Self.Iterator.Element) throws -> T) rethrows -> [T]`

`func flatMap<T>(_ transform: (Self.Iterator.Element) throws -> T?) rethrows -> [T]`

The two declarations are very similar. What Overload 2 does is to distinguish the case wherein the value returned by the transform function is an `Optional`. In this case, what is appended to the result array is the value that is wrapped by that optional - the optional will be **flattened**. The result array contains Ts, not T?s - if the transform function returns a nil then flatMap will discard it.

Okay, that all sounds good. But what if the value returned by the transform function is neither a `Sequence` nor an `Optional`? The answer is revealed by the following code:

    func f1() -> Int { return 1 }
    print("\(type(of: f1)) returns \(f1())")
    // Prints:
    // () -> Int returns 1

    func f2(_ f: @escaping () -> Int?) {
        print("\(type(of: f)) returns \(f())")
    }

    f2(f1)
    // Prints:
    // (()) -> Optional<Int> returns Optional(1)

As we can see, the compiler allows us to pass f1 as the argument for f2's parameter. Or, more generally, a () -> T as the argument for a () -> T? parameter. The print outs reveal that the compiler has "magically transformed" (surely there is a non-magical explanation?) f1's type when it is passed to f2. So, if the transform function returns a T, instead of a T?, the compiler is still able to match it to the signature of Overload 2.

Summarizing, if the transform function returns a `Sequence` then Overload 1 is used, else Overload 2 is used.

Now that we know what flatMap flattens and what it means to flatten, its time for some examples. Let's get on with it! In each of these examples the transform function will simply print its argument (so that we can see what it is) and then return it (so that we can see how flatMap flattens it).

##### FlatMap Example 1

    let sequence1 = [[1, 2], [3, 4], [5, 6]]
    let transform1 = { (arg: [Int]) -> [Int] in print("Arg = \(arg)"); return arg }
    let result1 = sequence1.flatMap(transform1)
    print("Result is \(type(of: result1)) = \(result1)\n")
    // Prints:
    //    Arg = [1, 2]
    //    Arg = [3, 4]
    //    Arg = [5, 6]
    //    Result is Array<Int> = [1, 2, 3, 4, 5, 6]

Overload 1 is used to flat map `Array<Array<Int>>` to `Array<Int>`.

##### FlatMap Example 2

Let's be a bit more terse.

    let result2 = [1, 2, 3].flatMap() { (arg: Int) -> Int in print("Arg = \(arg)"); return arg }
    print("Result is \(type(of: result2)) = \(result2)\n")
    // Prints:
    //    Arg = 1
    //    Arg = 2
    //    Arg = 3
    //    Result is Array<Int> = [1, 2, 3]

Overload 2 is used to flat map `Array<Int>` to `Array<Int>`.

##### FlatMap Example 3

    let result3 = [[[1, 2], [3,4]], [[5, 6], [7, 8]]].flatMap() { (arg: [[Int]]) -> [[Int]] in print("Arg = \(arg)"); return arg }
    print("Result is \(type(of: result3)) = \(result3)\n")
    // Prints:
    //    Arg = [[1, 2], [3, 4]]
    //    Arg = [[5, 6], [7, 8]]
    //    Result is Array<Array<Int>> = [[1, 2], [3, 4], [5, 6], [7, 8]]

Overload 1 is used to flat map `Array<Array<Array<Int>>>` to `Array<Array<Int>>`.

##### FlatMap Example 4

How about a sequence other than `Array<T>`, such as `CountableClosedRange<Int>`?

    let result4 = (1 ... 3).flatMap() { (arg: Int) -> Int in print("Arg = \(arg)"); return arg }
    print("Result is \(type(of: result4)) = \(result4)\n")
    // Prints:
    //    Arg = 1
    //    Arg = 2
    //    Arg = 3
    //    Result is Array<Int> = [1, 2, 3]

Overload 2 is used to flat map `CountableClosedRange<Int>` to `Array<Int>`.

##### FlatMap Example 5

For a final example, let's see flatMap's nil filtering in action.

    let result5 = [1, 2, nil, 4].flatMap() { (arg: Int?) -> Int? in print("Arg = \(arg)"); return arg }
    print("Result is \(type(of: result5)) = \(result5)\n")
    // Prints:
    //    Arg = Optional(1)
    //    Arg = Optional(2)
    //    Arg = nil
    //    Arg = Optional(4)
    Result is Array<Int> = [1, 2, 4]

Overload 2 is used to flat map `Array<Int?>` to `Array<Int>`.

## Conclusion

Let's conclude by restating the answers to our original questions:

1. What does flatMap flatten?  
    It flattens the return value of the transform function.
2. What does flatten mean?  
    a. In the case of a sequence it means to extract the elements of the sequence.  
    b. In the case of an optional it means to unwrap the optional or to discard it if nil. As was demonstrated, this also works for non optionals.

Well, there you have it; flatMap explained. I hope that you found it to be clear and understandable. It'd be great to receive your feedback.
