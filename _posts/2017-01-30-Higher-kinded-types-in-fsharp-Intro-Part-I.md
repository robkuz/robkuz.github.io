---
published: true
layout: post
title: Higher Kinded Types in F# - The Introduction
tags:
- F#
- Higher Kinded Types
---

# A Story About the Need for Higher Kinded Types
You probably will have seen people tweeting complaints (agreed that is mostly me) about the lack of higher kinded types (HKT) in F# 
and you might have thought to yourself I don't need no stinkin' HKT 
or you might have thought what the heck are these HKT anyways if you are more of the open minded type (OMT).

This series of blogposts will show case what HKT are using one concrete example, how HKTs are employed natively in Haskell 
and how you can use and emulate them in F#

- Part I - What is the Problem Anyways? - That's where you are now
- [Part II - A Short Visit to Haskell Land](https://robkuz.github.io/HKTS-in-fsharp-Part-II-A-Short-Visit-To-Haskell-Land)
- [Part III - How to emulate HKTs in F#](https://robkuz.github.io/HKTS-in-fsharp-part-III-Concept-Emulation)
- [Part IV - Signature Annotations](https://robkuz.github.io/HKTS-in-fsharp-Part-IV-Signature-Annotations)
- [Part V - Roundup and FAQ]() soon to be published

## A real life story
So you have been transferred to this new project - some invoicing stuff! Who needs that? But hey they are using F# (yay!) 
if only that greasy product manager wouldn't be on the team. He was a drag on the last project.

## Day 1 - A simple and clean design
You start with your PM the first design/coding session - It's about line items. Very soon you 2 come up with the following design.

``` fsharp
type LineItem = {
    articleID: string
    units:     int
    amount:    float
}
```

That was a productive day and somehow the PM was nice and understanding. Maybe he isn't that bad guy you always pictured him?

### Day 2 - La complexidad!
The next day the PM start your conversition by stating he read thru the analysts docs and that there are changes to be made to your design.  
You wonder why he read the anaysis docs only now. He explains to you that the `LineItem` needs to be processed in stages thru out the program.  
There are 2 stages: `InProgress` and `Finished`.  
While `InProgress` any property can be changed by the system. For example we might reduce the units because there aren't enough on stock or we reduce the amount because our client is getting some rebate etc.
So you start creating 2 types

``` fsharp
type Decision<'a> = Decision of 'a
type InProgress<'a> = {value: 'a; decisions: Decision<'a>}

type LineItemInProgress = {
    articleID: InProgress<string>
    units:     InProgress<int>
    amount:    InProgress<float>
}
```

The `Finished` `LineItem` simply protocols the decisions and the initial value with a timestamp.

``` fsharp
type Finished<'a> = {value: 'a; initial: 'a; timestamp: DateTime}

type LineItemFinished = {
    articleID: Finished<string>
    units:     Finished<int>
    amount:    Finished<float>
}
```

Those 2 types a pretty much the same. But hey what can you do? It's F# <sigh>!  
The next step is to build a state transforming function from `InProgress` to `Finished`

``` fsharp
let inline toFinished (x: InProgress<'a>) : Finished<'a> = //does something with the decisions 
let toFinishedLineItem x : LineItemFinished = 
    {
        articleID = toFinished x.articleID
        units     = toFinished x.units
        amount    = toFinished x.amount
    }
```

"That is not so bad! One type with a some duplication" you think to yourself. And up to here you are right.  
Yet while you were happily coding those types your product manager had a brilliant idea!  
PM : We need to move `Finished` line items to `InProgress` back. Our system has no realtime interface to the outside world 
so that sometimes the decision that have been made are wrong."

So you open your editor to reuse your already existing code - only there is no reuse (except when you have put "copy & paste" into your mental category of "reuse").  
As `InProgressLineItem` and `FinishedLineItem` are 2 distinctly types and your function signature is 
`toFinishedLineItem :: LineItemInProgress -> LineItemFinished` there is no way to invert that signature.  
So we copy, paste and adjust that code. But hey! It's F#!

``` fsharp
let inline toInProgress (x: Finished<'a>) : InProgress<'a> = //does something with the decisions 
let toInProgressLineItem x : LineItemInProgress = 
    {
        articleID = toInProgress x.articleID
        units     = toInProgress x.units
        amount    = toInProgress x.amount
    }
```

And while you compile and test your newly pasted code your product manager strolls into your office to tell you about his latest stroke of genius!  
PM: "We need a `Cancelled` state and a `New` state because the `New` state is when ...` this is the moment your mind starts to wander 
and ideas of waterboarding your PM flash before you eyes.

PM: "Also we need state transitions from:  
- New -> InProgress
- InProgress -> New
- InProgress -> Cancelled
- Cancelled -> InProgress
- Cancelled -> Finished
- Finished -> Cancelled"

The PM cuts the meeting short as he sees the murderous look building up in your eyes without telling you about those `Archived` and `Prepared` states.
You sigh deeply and start to create all those similar types and transition functions

``` fsharp
type Cancelled<'a> = {value: 'a; decisions: List<Decision>; timestamp: DateTime; reason: string}

type CancelledLineItem = {
    articleID: Cancelled<string>
    units:     Cancelled<int>
    amount:    Cancelled<float>
}

let inline toCancelled (x: InProgress<'a>) : Cancelled<'a> = //does something with the decisions 
let toCancelledLineItem x : CancelledLineItem = 
    {
        articleID = toCancelled x.articleID
        units     = toCancelled x.units
        amount    = toCancelled x.amount
    }

type New<'a> = {value: 'a; ... } //I guess you have the idea by now
```

### Late at night - Almost day 3
While the clock crawls towards midnight your mail client beeps. A message from your product manager boasting on how he smart and quality consious he is 
and that he detected a serious error that needs fixing before tomorrows release: "A `LineItem` must include a `VAT` property".  
You reach out to Amazon to order a Bazooka over night - you gonna finish that slimer once and for all tomorrow morning ...

## Changes, Changes, Changes and bit of Compiler help
Latest here you should understand that that last requirement will make you touch every single type you created 
and every single transition function and every single test you wrote.

Agreed: F#'s compiler and type inferrence makes most of these changes pretty mechanical but in a way that makes it even more painful. 
Why not let the compiler do those things it can easily do instead us having to write and maintain tons of copy & pasted code?

Before I explain how to emulate HKT in F# in Part III of this series lets look briefly into Haskell and how they solve this in [Part II - A Short Visit to Haskell Land](https://robkuz.github.io/HKTS-in-fsharp-Part-II-A-Short-Visit-To-Haskell-Land).  
If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com)
