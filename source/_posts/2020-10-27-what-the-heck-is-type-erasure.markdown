---
layout: post
title: "What the heck Is Type Erasure and Why Should We Care?"
date: 2020-10-27 17:14:59 -0500
comments: true
categories: [swift,protocols,generics]
keywords: type erasure,generic protocols
description: A look into how can type erasure can be used to overcome the limitations of swift's generic protocols.
---

We are all unique individuals whose minds process and organize information in different ways, and whose communication styles reflect that internal organization. When I first encountered the words **Type Erasure** I thought: *"Oh, some cool swifty thing. I'd better find out about it"*. So, I started googling and reading. But, time and again, I came up short. Even though I (mostly) understood what I had read, my understanding felt incomplete. There seemed to be something missing, as though the author had not made some essential concept clear. However, I persevered and now I get it. So for this, my first ever blog post, I'm going to take a shot at explaining type erasure. Maybe it will help someone else who thinks differently.

<!-- more -->

## Generic Protocols Have Some Limitations

Swift's implementation of generic protocols comes with some limitations. We cannot use generic protocols in all of the ways that we are accustomed to using regular protocols. Let's have a look. Here are some ways in which we normally use a protocol:

{% gist 0912a5667b684b0b6894ab021f59e6d5 RegularProtocol.swift %}
[{% img /images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/RegularProtocol.png %}](/images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/RegularProtocol.png)

As you can see in the REPL screen shot, the compiler is happy with the code. However, when we use a generic protocol the results are different:

{% gist 0912a5667b684b0b6894ab021f59e6d5 GenericProtocol.swift %}
[{% img center /images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/GenericProtocol.png %}](/images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/GenericProtocol.png)

Now we have the infamous *"error: protocol 'MyProtocol' can only be used as a generic constraint because it has Self or associated type requirements"* compiler error on lines 4, 5, & 6. We are not allowed to use a generic protocol as a type specfier. Why?

Well, from the compiler's perspective the issue is type safety. The compiler does not know the type of the associated type and therefore cannot enforce type safety. From the swift development team's perspective: I do not know what challenges they faced while creating generic protocols but I think they decided that the benefits of generic protocols as they currently stand are sufficiently compelling (hmmmm ... a good topic for another post) to justify their incorporation into the language in spite of the limitations.

## An Example Wherein We Would Like To Use Generic Protocols To Specify Types

{% gist 0912a5667b684b0b6894ab021f59e6d5 SoldiersAndWeapons.swift %}

We can create an army of infantry men, or grenadiers, or etc. But we cannot create an army of soldiers.

{% gist 0912a5667b684b0b6894ab021f59e6d5 ProblematicArmy.swift %}
[{% img center /images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/ProblematicArmy.png %}](/images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/ProblematicArmy.png)

## Type Erasure To The Rescue

Type Erasure is a coding technique whereby we can work around the type specification limitation of generic protocols. For any given generic protocol, let's call it MyGenericProtocol, we will define an *erasing* type, let's call it MyTypeEraser. MyTypeEraser will adopt MyGenericProtocol and will be used as a substitute for MyGenericProtocol when specifying types: wherever we would have liked to use MyGenericProtocol to specify a type, we will instead use MyTypeEraser. MyTypeEraser will give us all of the benefits of MyGenericProtocol without the limitation. Let's see how this works.

In the case of our Army example, the generic protocol that we are dealing with is Soldier. We will name our type eraser AnySoldier. \[Note that the prefix `Any` followed by the name of the generic protocol is a naming convention that Apple follows in the swift standard library when type erasing is employed: [AnySequence](https://developer.apple.com/reference/swift/anysequence), [AnyCollection](https://developer.apple.com/reference/swift/anycollection), etc.\] An AnySoldier will be initialized using an instance of some other Soldier adopting type (Sniper, Infantryman, etc.). The AnySoldier will *capture* the actual soldier's implementation of the Soldier protocol (you'll soon see what we mean by capture). The AnySoldier will then be used as a standin for the actual soldier: wherever we need a Soldier adopter such as an Infantryman or a Grenadier, we will instead create and use an AnySoldier. Whenever the application accesses an AnySoldier via the Soldier protocol, the AnySoldier will forward that access to the original soldier instance. The concrete AnySoldier type will effectively behave like a Soldier protocol because nothing other than the protocol's interface will be exposed. AnySoldier can be used without the compiler having any knowledge of the original instance's type; that original type **will have been erased**. This will all become more clear as we proceed.

## AnySoldier: First Pass

Let's start developing AnySoldier. In order for AnySoldier to adopt the Soldier protocol it must specify the type of Weapon. How shall we handle this? Well, we could make AnySoldier generic with regard to the type of Weapon.

{% gist 0912a5667b684b0b6894ab021f59e6d5 AnySoldierFirstPass.swift %}
[{% img center /images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/AnySoldierFirstPass.png %}](/images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/AnySoldierFirstPass.png)

Well, that's not what we want. We weren't able to add a Grenadier to the army. We can create an army of any type of soldier as long as they all use the same type of Weapon (in this case Rifle). We've moved our issue from the Soldier to the Weapon. What can we do?

## AnySoldier: Second Pass

How about if we type erase Weapon as well? Let's create an AnyWeapon.

{% gist 0912a5667b684b0b6894ab021f59e6d5 AnySoldierSecondPass.swift %}
[{% img center /images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/AnySoldierSecondPass.png %}](/images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/AnySoldierSecondPass.png)

There we go; the compiler is happy. We can create an army and populate it with any and all types of Soldiers. Great! But wait a minute: the weapons aren't firing (i.e. the print statements are not coming to the terminal). What is going on?

## AnySoldier: Final Solution

Well, remember that we said: "AnySoldier will *capture* the actual soldier's implementation of the Soldier protocol". We haven't yet implemented the captures. Time for the magic sauce.

{% gist 0912a5667b684b0b6894ab021f59e6d5 AnySoldierFinalSolution.swift %}
[{% img center /images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/AnySoldierFinalSolution.png %}](/images/blog/2017/01/24/what-the-heck-is-type-erasure-and-why-should-we-care/AnySoldierFinalSolution.png)

Voila! All weapons are firing. Let there be war.

## Wrapping It Up

An AnySoldier is able to *assume the Soldier identity* of any adopter of the Soldier protocol (Sniper, Infantryman, Grenadier, or Artillaryman). The AnySoldier accomplishes this without any reference to the original type. That original type has dissappeared from view; **it has been erased**. With AnySoldier at our disposal we can write code that is equivalent to what we would have written were we able to use Soldier to specify types. AFAIK there is no functional difference between specifying types with AnySoldier vs Soldier (given that we were allowed to). I would hazard to say that if a future swift allowed us to use generic protocols as type specifiers then a find/replace AnySoldier => Soldier would do the trick. This is why we care about type erasure.

There you have it. Type Erasure a la Verticon. I hope that it spoke to you. As I said, this is my first blog post. I expect to learn and to get better at it. Any feedback will be welcome. For those of you whose different way of thinking is different from mine, here are some other posts on the same topic:
* [Rob Napier](http://robnapier.net/erasure)
* [Hector Matos](https://krakendev.io/blog/generic-protocols-and-their-shortcomings)
