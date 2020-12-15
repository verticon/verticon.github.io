---
layout: post
title: "Swift Capture Lists Rock"
date: 2020-11-17 14:55:19 -0400
comments: true
categories: [swift]
keywords: closures,capture lists
description: A look into how a capture list can be used to modify the manner in which a closure captures its environment.
---

Closures capture (enclose) their environment. This is to say that a closure is able to access any variable that is visible from the context wherein the closure is defined. This remains true even if the closure is passed into another environment where from those variables are not visible (ex. a function defined in a different file). Capture lists provide a syntax for modifying the manner whereby a closure captures its environment.

<!-- more -->

## First a Look at Closures Without Capture Lists

In order to understand capture lists we must first understand how a closure functions when no capture list has been declared. In the absence of a capture list, a closure accesses its environment's variables through the use of pointers.[Caveat: I am using the word "pointer" because it works very well when discussing closure behavior. However, I have some uncertainty about whether or not it is the 100% technically correct term]. This is best illustrated with an example:

{% gist 4399100df8182eda310903fab4a6be43 ExampleOne.swift %}

Note that the variable *name* has the value "verticon" at the time that the closure is defined. But if the value is changed and then the closure is again executed the new value is printed. The closure has a pointer to the variable and will always print its current value. Let's do the same thing with a struct:

{% gist 4399100df8182eda310903fab4a6be43 ExampleTwo.swift %}

Again the closure has a pointer to the variable and will always print its current value.

The first two examples used value types. Let's see what happens with a reference type:

{% gist 4399100df8182eda310903fab4a6be43 ExampleThree.swift %}

This example takes a bit more thought. The variable *name* refers to a Name type. Its value is a reference to an actual instance of the Name type. The closure has a pointer to *name*. When the closure dereferences that pointer it obtains *name's* current value, a references to a Name instance. Just as in the previous two examples the closure always prints the current value of the variable.

## On To Capture Lists

The syntax of a capture list is [variable1, variable2, ..., variableN] The comma separated list of items inside of the square brackets are the names of variables from the environment - those variables will have their capture semantics altered. Here are some examples of closure declarations that specify a capture list:

    let value = 1
    let closure1 = { [value] in return value }
    let closure2 = { [value] (i: Int) in return i * value }
    let closure3 = { [value] (i: Int) -> Bool in return i > value }

 A capture list must occur at the start of a closure declaration and the closure declaration must, at a minimum, include the `in` keyword. Strangely, the following declaration compiles but the capture semantics of the variable *value* are not altered:

    let closure1 = { [value]
        return value }

The following, however, works as expected:

    let closure1 = { [value] in
        return value }

It feels like a compiler bug. I am using Swift 3.1

So what do we mean when we say that the variables specified in the capture list have their capture semantics altered? We mean this: the closure will no longer capture a pointer to the variable; instead it will capture the value that the variable has at the time that the closure is created. If the variable's value is subsequently changed then the closure will continue to execute with the old value. To see this in action let's revisit our previous examples, modified by specifying a capture list.

{% gist 4399100df8182eda310903fab4a6be43 ExampleOneRevisited.swift %}

{% gist 4399100df8182eda310903fab4a6be43 ExampleTwoRevisited.swift %}

{% gist 4399100df8182eda310903fab4a6be43 ExampleThreeRevisited.swift %}

In each case the value printed by the closure does not change when the value of the variable is changed. This is because the closure has captured the variable's value instead of a pointer to it. Pretty straight forward, yes?

## But Wait, There's More

Something to realize about closures is that they are reference type objects. This means that they, just like class objects, remain in existence until their reference count goes to zero. And, *this is important*, any objects referenced by the closure cannot be deallocated as long as the closure remains in existence. Unless, that is, we take steps to prevent that from occurring - more about that as we go along.

Let's introduce the code that we will use to explore the object retention behavior of closures. [Note: This code is ready to run in a playground]

{% gist 4399100df8182eda310903fab4a6be43 ReferenceRetention.swift %}

The Creator class creates a closure and hands it to an Executor for execution. We'll be using a Creator instance to look into the retention consequences of the closure accessing the Creator's members. The Creator contains several versions of the closure which we will be commenting/uncommenting in order to observe the differences

The Executor class stores a closure, thus keeping it "alive". It uses a timer to periodically execute the closure, continuing to do so until "something" (such as our Controller) sets the closure to nil.

The Controller class sets things into motion by instantiating a Creator. It also presents a UI with two buttons: one is used to set the Creator variable to nil; the other is used to set the Executor's closure variable to nil.

#### Closure version 1

    let closure = { log("\n\(name)'s closure executed") } // Version 1

Here the value of the initializer's `nameArg` argument is captured. This has no retention consequences and hence the Creator and its Name are immediately deallocated when the deallocate creator button is pressed. The closure "lives on" and continues to execute until the deallocate closure button is pressed.

    Verticon's closure executed 2017-10-04 20:17:04 +0000

    Verticon's closure executed 2017-10-04 20:17:06 +0000

    Deallocate creator button pressed 2017-10-04 20:17:07 +0000
    Creator deallocated 2017-10-04 20:17:07 +0000
    Name deallocated 2017-10-04 20:17:07 +0000

    Verticon's closure executed 2017-10-04 20:17:08 +0000

    Verticon's closure executed 2017-10-04 20:17:10 +0000

    Deallocate closure button pressed 2017-10-04 20:17:11 +0000
    Closure deallocated 2017-10-04 20:17:11 +0000

#### Closure version 2

      let closure = { [name = self.storedName.name] in log("\n\(name)'s closure executed") } // Version 2

Here we've introduced a new capture list syntax. It turns out that a capture list allows us to declare new variables for use by the closure. Here we've created the `name` variable so that the closure can print the name without having to reference the Creator or Name class instances. The results are exactly the same as for version 1

#### Closure version 3

let closure = { [name = self.storedName] in log("\n\(name)'s closure executed") } // Version 3

This time the closure is referencing the storeName instance [we are again declaring a local variable so that the closure will not need to reference `self`]. This results in the deallocation of the Name instance being delayed until the closure is deallocated.

    Verticon's closure executed 2017-10-04 20:30:28 +0000

    Verticon's closure executed 2017-10-04 20:30:30 +0000

    Deallocate creator button pressed 2017-10-04 20:30:31 +0000
    Creator deallocated 2017-10-04 20:30:31 +0000

    Verticon's closure executed 2017-10-04 20:30:32 +0000

    Verticon's closure executed 2017-10-04 20:30:34 +0000

    Deallocate closure button pressed 2017-10-04 20:30:35 +0000
    Closure deallocated 2017-10-04 20:30:35 +0000
    Name deallocated 2017-10-04 20:30:35 +0000

#### Closure version 4

    let closure = { log("\n\(self.storedName)'s closure executed") } // Version 4

This time the closure is referencing both the Name and Creator instances resulting in both deallocations being delayed until the the closure is deallocated.

[An ancillary inquiry: In version 4, what has the closure captured? A pointer to self? If yes then what is it about capturing a pointer to a reference variable that causes the reference's count to be incremented? These thoughts led to my initial caveat regarding the use of the word "pointer".]

    Verticon's closure executed 2017-10-04 20:35:39 +0000

    Verticon's closure executed 2017-10-04 20:35:41 +0000

    Deallocate creator button pressed 2017-10-04 20:35:42 +0000

    Verticon's closure executed 2017-10-04 20:35:43 +0000

    Verticon's closure executed 2017-10-04 20:35:45 +0000

    Deallocate closure button pressed 2017-10-04 20:35:46 +0000
    Closure deallocated 2017-10-04 20:35:46 +0000
    Creator deallocated 2017-10-04 20:35:46 +0000
    Name deallocated 2017-10-04 20:35:46 +0000


## Fine But So What?

Okay, we've seen that a closure can cause an object's reference count to increase but why do we care? Well, in the case of the Creator/Executor/Controller example the Controller can access the variable that holds the closure reference and nil it when needed, but what if this is not the case? What if we release the closure *out into the wild* by invoking a function such as this:

    func doSomething(with: @escaping () -> Void)

The @escaping attribute tells us that the function is going to store the closure reference. Now the closure is *living out there somewhere* and there is nothing that we can do about. There is, however, something that we can do so that this scenario does not prevent us from deallocating the objects to which the closure refers: we can use a capture list to declare that the closure's references are `weak`.

#### Closure version 5

    let closure = { [weak self] in log("\n\(self?.storedName.name ?? "NoName")'s closure executed") } // Version 5

Now when the Deallocate Creator button is pressed the Creator and the Name are immediately deallocated even though the closure is still alive.

    Verticon's closure executed 2017-10-05 16:22:30 +0000

    Verticon's closure executed 2017-10-05 16:22:32 +0000

    Deallocate creator button pressed 2017-10-05 16:22:33 +0000
    Creator deallocated 2017-10-05 16:22:33 +0000
    Name deallocated 2017-10-05 16:22:33 +0000

    NoName's closure executed 2017-10-05 16:22:34 +0000

    NoName's closure executed 2017-10-05 16:22:36 +0000

    Deallocate closure button pressed 2017-10-05 16:22:38 +0000
    Closure deallocated 2017-10-05 16:22:38 +0000

There is one other option that we have for preventing a closure from holding a strong reference: we can use the `unowned` attribute instead of the `weak` attribute. The `unowned` attribute has the advantage of not being checked for nil (rather like an implicitly unwrapped optional) and therefor results in faster executing code (which might matter). However, in order to use `unowned` we must be certain that our code meets a certain criteria: we must guarantee that the closure will never again be executed after the object being referenced has been deallocated. Like `weak`, `unowned` will allow the object to be deallocated but like an implicitly unwrapped optional it will cause the program to trap if it is used after being set to nil (the system sets weak and unowned references to nil when the object being referenced is deallocated).

#### Closure version 6

    let closure = { [unowned self] in log("\n\(self.storedName)'s closure executed") } // Version 6

When the Deallocate Creator button is pressed the program crashes. [In a playground this is evidenced by the Controller disappearing from the Timeline Assistant Editor and a home screen appearing] However, if we first press the Deallocate Closure button then all is well. Here is a less contrived example wherein `unowned` is used successfully:

{% gist 4399100df8182eda310903fab4a6be43 UnownedSelf.swift %}

## Conclusion

Alright, there we have it. When we write closures we can capture variables with confidence.
