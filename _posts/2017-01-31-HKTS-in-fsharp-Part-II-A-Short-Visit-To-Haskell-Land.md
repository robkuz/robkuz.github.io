---
published: true
layout: post
title: Higher Kindended Types in F# Part II - A Short Visit To Haskell Land
tags:
- F#
- Higher Kinded Types
---

As you reached this shore I assume you liked [Part I - What is the Problem Anyways?](https://robkuz.github.io/Higher-kinded-types-in-fsharp-Intro-Part-I/) of this series.  
Now we are going to look how the Haskell community will solve our problem. If you have no experience in Haskell, don't worry. 
The syntax and concepts you will encounter are really easy to grasp.

First we start with some language directives. That is the Haskell community's way to quickly evolve the language and maintain compatibility.

``` Haskell
{-# LANGUAGE RankNTypes #-}
{-# LANGUAGE MultiParamTypeClasses #-}
```

I will explain what these 2 directives do in a moment. First define some simple types. I aliased Date to Int to make the examples easier 
and also Decision would be a much more complex type in reality

``` Haskell
type Date = Int
data Decision a = Decision a 
```

Now lets define our state types as we did in our F# programm.

``` Haskell
data InProgress a = InProgress {progress :: a, decision :: Decision a } 
data Finished a = Finished {finished :: a, initial :: a, ftimestamp :: Date } 
```

And our LineItem types

``` Haskell
data LineItem a = LineItem {article :: a String, amount :: a Float, units :: a Int } 
```

Well only that its not types but type (singular). The reason here is that because of higher kinded types we dont need to declare a specific `LineItemInProgress` type or
a `LineItemFinished` type. The magic is in these declaration `... :: a SomeType`. What does that mean? Actually we are telling the compiler 
that we like to substitute `a` with a type constructor of kind `* -> *` which will get `SomeType` as a parameter. 
WTF is a `type constructor` and what is `kind * -> *`?
Lets dive a bit deeper onto that using on of the type above

``` Haskell
    data InProgress a = InProgress ...
         ^^^^^^^^^^ ^   ^^^^^^^^^^
             |      |         |
             |      |         -------- Value constructor
             |      ------------------ Kind parameter(s)
             ------------------------- Type constructor
```

A type constructor can be regarded as some (partially applied) function definition - only that this special function does not expect an value as a parameter
but another type (the kind) in order to create another (fully specified) type. Another quick example here

``` Haskell
data Foo a b = Foo a b
```

the type constructor above is of kind `* -> * -> *`

``` Haskell
type FooInt = Foo Int
```

Then we partially apply a type and yield another type constructor of kind `* -> *`

``` Haskell
type FooIntStr = FooInt String
```

And then final specific type.
So again if we say `... :: a String` we advise the compiler to substitute a type constructor for `a` of kind `* -> *` and apply the type `String`to it

Let's return to our code. The next are some trivial functions to make the example look more realistic ...

``` Haskell
now = 1
applyDecision (Decision d) v = d
```

And then we reach type classes. So what are they for?

``` Haskell
class Transition a b where  
    transit :: a -> b  
```

First we define a class (which has nothing to do with OO at all but more of an interface) that defines a function named transform from type `a` to `b`. 

``` Haskell
instance Transition (InProgress a) (Finished a) where
    transit (InProgress {progress = p, decision = d}) = Finished {finished = applyDecision d p, initial = p, ftimestamp = now }

instance Transform (Finished a) (InProgress a) where
    transit (Finished {finished = _, initial = i, ftimestamp = _ } ) = InProgress {progress = i, decision = Decision i}
```

And the we write 2 instances of Transition where we transition from `InProgress` to `Finished` and vice versa.
Finally we write a map function of type `map :: LineItem f -> LineItem g`

``` Haskell
mapLineItem (LineItem {article = art, amount = amt, units = u }) = LineItem {article = transit art, amount = transit amt, units = transit u}
```

And that is it. Not further longwinded transformation functions for specific types for LineItem like `LineItemFinished`.

Now lets use it by first creating a `LineItem InProgress` value

``` Haskell
x :: LineItem InProgress
x = LineItem {
    article = InProgress {progress = "art01", decision = Decision "art01"}, 
    amount = InProgress {progress = 10.0, decision = Decision 9.0}, 
    units = InProgress {progress = 5, decision = Decision 5} }
```

and then mapping that newly created value to other types

``` Haskell
y :: LineItem Finished
y = mapLineItem x
```

and back

``` Haskell
z :: LineItem InProgress
z = mapLineItem y
```

and finally try to transition a `LineItem Progress` to `LineItem Progress` 

``` Haskell
fails :: LineItem InProgress
fails = mapLineItem x
```

which must fail with an compiler error because

``` Haskell
No instance for (Transform (InProgress Float) (InProgress Float))
arising from a use of mapLineItem
```

Cool! Now lets observe how resillient this code is against changes.

First what happens if introduce a new state. There is no difference between F# and Haskell here. You have to define a type for that

``` Haskell
data Cancelled a = Cancelled {cancelled :: a, ctimestamp :: Date }   
```

Just definining the type won't cut here so we have to define transition function(s) as well

``` Haskell
instance Transition (InProgress a) (Cancelled a) where
    transit (InProgress {progress = p, decision = d}) = ...

instance Transition (Finished a) (Cancelled a) where ...
```

Here the difference is already visible. In Haskell you will have to write the transition function between the state types 
only whereas in F# you additionally have to create new transitions functions between the specified `LineItem` types.

And finally what about a new property on the LineItem type itself? In Haskell you simply add that additional field to the type (like in F#)

``` Haskell
data LineItem a = LineItem {article :: a String, amount :: a Float, units :: a Int, vat :: a Float } 
```

You then need to adjust value construction calls (like in F#)

``` Haskell
x :: LineItem InProgress
x = LineItem {
    article = InProgress {progress = "art01", decision = Decision "art01"}, 
    amount = InProgress {progress = 10.0, decision = Decision 9.0}, 
    units = InProgress {progress = 5, decision = Decision 5}.
    vat = InProgress {progress = 19.0, decision = Decision 7.0}
    }
```

and that is it - nothing more. your done. Hasta la vista, baby! This is sweet and concise!

In F# however this seemingly trivial change will ripple thru your code like a tsunami. 
You will have to touch every type and therefore also every transition function.  
Its OK if you silently weep now.

I hope I could convince you of the benefits of having higher kinded types in your code and the remaining question is: can we do something similar in F#?
Follow me to the [third installment of this series - Concept Emulation](https://robkuz.github.io/2017-02-01-HKTS-in-fsharp-part-III-Concept-Emulation) to find out about it.

If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com)
