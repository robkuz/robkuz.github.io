---
published: true
layout: post
title: Higher Kindended Types in F# Part IV - Signature Annotations
tags:
- F#
- Higher Kinded Types
---
A short introduction on how to express higher kinded type also on function signatures and write even more generic compile time checked functions.

### Series Table of Contents

I hope you liked the previous parts of this series 

- [Part I - What is the Problem Anyways?](https://robkuz.github.io/Higher-kinded-types-in-fsharp-Intro-Part-I/)
- [Part II - A Short Visit to Haskell Land](https://robkuz.github.io/HKTS-in-fsharp-Part-II-A-Short-Visit-To-Haskell-Land/)
- [Part III - Concept Emulator](https://robkuz.github.io/HKTS-in-fsharp-part-III-Concept-Emulation)
- Part IV - Signature Annotations
- [Part V - Roundup and FAQ]() soon to be published

## The Case of the Missing Type Classes

The examples sofar have shown how to define a higher kinded type using decomposition approaches.
However this is not the end of the story. What you usually want (and can do) with languages supporting
HKTs is to write generic functions that operate on those types and are even able to express relationships
between the constituent parts of such a a type.

Let's return to our previous examples (and this might be a good time to (re)read at least [part III](https://robkuz.github.io/HKTS-in-fsharp-part-III-Concept-Emulation) of it - or even all of it)

``` fsharp
type Brand<'a, 'b> (value:obj) =
    member self.Apply() : obj = value   

type Decision<'a> = Decision of 'a

type InProgress () = class end
type Finished () =  class end

type InProgress<'a> = {value: 'a; decision: Decision<'a>}
type Finished<'a> = {value: 'a; initial: 'a; timestamp: int}
```

Before we continue let's define some terms:

- _Brand_ is the container that we need to actually define a HKT
- the parameterless classes that somehow mirror every type that we want to defunctionalize we will call _Tag_
- and finally the the type that we want to elavate we will call _Kind_ (actually that should be _potential Kind_ but ...)

So when we have defined our Brand (which we should import from a library) our Tag and our Kind we can easily define concrete
higher kinded types like this
``` fsharp
type FinishedInt = Brand<Finished, int>
```

and we can define our `inj` and `prj` functions to transform between the _Brand(ed)_ representation and the native _Kind_.

``` fsharp
let inline inj (value: Finished<'A>) : Brand<Finished, 'A> = new Brand<_,_>(value)
let inline prj (value: Brand<Finished, 'A>) : Finished<'A> = value.Apply() :?> _
```

But there is a problem. We have to write a separate `inj` and `prj` for every `Tag` and `Kind` pair even thou their
implementation is exactly the same and is screaming **generic implementation**. But we need those exact parameter signatures.
On top of all of this we have to implement those functions in their own module as functions can not be overloaded.

``` fsharp
module INP =
    let inline inj (value: InProgress<'A>) : Brand<InProgress, 'A> = new Brand<_,_>(value)
    let inline prj (value: Brand<InProgress, 'A>) : InProgress<'A> = value.Apply() :?> _

module FIN =
    let inline inj (value: Finished<'A>) : Brand<Finished, 'A> = new Brand<_,_>(value)
    let inline prj (value: Brand<Finished, 'A>) : Finished<'A> = value.Apply() :?> _
```

And if we add a new type we need to do all of this once again

``` fsharp
type Cancelled () =  class end
type Cancelled<'a> = {value: 'a; timestamp: int}

module FIN =
    let inline inj (value: Cancelled<'A>) : Brand<Cancelled, 'A> = new Brand<_,_>(value)
    let inline prj (value: Brand<Cancelled, 'A>) : Cancelled<'A> = value.Apply() :?> _
```

Wow! This is like AC/DCs concert boxes on full blast yelling "Type Classes" (or "Thunder")! Btw. if you want to make yourself heard [here is the type class proposal](https://github.com/fsharp/fslang-suggestions/issues/243)

## Build your Own Type Classes
Alas I promised you that there is a way to turn that boilerplate mess into a generic and understandable code.
Let's first try a very naive version of it and simply leave the types out.

``` fsharp
let inline inj value = new Brand<_,_>(value)
let inline prj(value: Brand<'Tag, 'ValueT1>) = value.Apply() :?> _
```

And now we create a Brand by using `inj`

``` fsharp
let x1 : Brand<Finished, int> = {value = 1; initial = 2; timestamp = 3} |> inj
```

That works. Great! How about we let the type inferrer do a bit more of the work

``` fsharp
let err10 = {value = 1; initial = 2; timestamp = 3} |> inj
```

Oops! That didn't work nearly as I hoped for. The compiler tells us

``` fsharp
Value restriction. The value 'err10' has been inferred to have generic type
    val err10 : Test.Brand<'_a,'_b>    
Either define 'err10' as a simple data term, make it a function with explicit arguments or, 
if you do not intend for it to be generic, add a type annotation.
```

So the compiler wants us to have a bit more type annotations. OK you might think that isn't so bad.
I got it running and I got a compiler error.  
Well not exactly - Apart from not being able to infer types anymore we can do this now

``` fsharp
let err20 : Brand<InProgress, int> = {value = 1; initial = 2; timestamp = 3} |> inj
```

without any compiler error or even worse

``` fsharp
let err20 : Brand<InProgress, string> = {value = 1; initial = 2; timestamp = 3} |> inj
```

This code will result in a casting error during runtime. :-(  
Unsurprisingly the naive version didn't work out.  
Before we continue lets analyse what we really want and that would be the following signature

``` fsharp
let inline inj (value: Kind<'A, 'Tag>) : Brand<'Tag, 'A> = new Brand<_,_>(value)
let inline prj (value: Brand<'Tag, 'A>) : Kind<'A, 'Tag> = value.Apply() :?> _
```

We want a generic _Kind_ that has its usual generic parameter and its assicated _Tag_ and that generic parameter 
and that _Tag_ should be mirrored in the _Brand_ signature ...  
Only F# doesn't support generic types that have other generic type params. That would be a higher kinded type.  
Again how can this be solved? Somehow a type is needed that binds the ordinary geneneric param together with the _Tag_ and the _Kind_ ...  
Or maybe just a function signature? Let's try to extend our _Kinds_ with a `Kind` member function

``` fsharp
type InProgress<'a> = {value: 'a; decision: Decision<'a>}
    with
    member x.Kind(): ('a * InProgress) = undef()

type Finished<'a> = {value: 'a; initial: 'a; timestamp: int}
    with
    member x.Kind(): ('a * Finished) = undef()
```

Define some simple helper so we do not need to really implement those `Kind` members

``` fsharp
let undef () : ('a * 'b) = (Unchecked.defaultof<'a>, Unchecked.defaultof<'b>)
```

And now we rewrite our `inj` and `prj` functions using this information

``` fsharp
let inline inj<'Kind, 'Tag, 'A when 'Kind : (member Kind : unit -> 'A * 'Tag)> (value: 'Kind) : Brand<'Tag, 'A> = new Brand<_,_>(value)
let inline prj<'Kind, 'Tag, 'A when 'Kind : (member Kind : unit -> 'A * 'Tag)> (value: Brand<'Tag, 'A>) : 'Kind = value.Apply() :?> _
```

### Testing
Now lets test our code once again

``` fsharp
let success01 = {value = 1; initial = 2; timestamp = 3} |> inj
```

and this will yield `success01 :: Brand<Finished, int>`. Nice! What about the reverse one?

``` fsharp
let success02 = success01 |> prj
```

This will yield `success02 :: Finished<int>`. So type inferrence works as expected (and you can also annotate if you want to).  
So what about those nasty runtime/casting errors. Lets try those

``` fsharp
let err20 : Brand<InProgress, int> = {value = 1; initial = 2; timestamp = 3} |> inj
//                ^^^^^^^^^^ <- wrong
```

Now we are getting the following **compile time** error

``` fsharp
Type mismatch. Expecting a
    'Finished<int> -> Brand<InProgress,int>'    
but given a
    'Finished<int> -> Brand<Finished,int>'    
The type 'InProgress' does not match the type 'Finished'
```

That is exactly what we wanted. Let's test the other param

``` fsharp
let err21 : Brand<Finished, string> = {value = 1; initial = 2; timestamp = 3} |> inj
//                          ^^^^^^ <- wrong
```

and will get thow following correct compile time error

``` fsharp
Type mismatch. Expecting a
    'Finished<int> -> Brand<Finished,string>'    
but given a
    'Finished<int> -> Brand<Finished,int>'    
The type 'string' does not match the type 'int'
```

### Condense
Very nice. Again what we wanted and expected.  
The whole exercise came at a cost thou: the function signatures for `inj` and `prj` have become a bit wordy it seems.
So we will condense that using some type aliasing

``` fsharp
type Kind<'Kind, 'Tag, 'A when 'Kind : (member Kind : unit -> 'A * 'Tag)> = 'Kind
```

and this new function signatures

``` fsharp
let inline inj(value: Kind<'Kind, 'Tag, 'A>) : Brand<'Tag, 'A> = new Brand<_,_>(value)
let inline prj(value: Brand<'Tag, 'A>) : Kind<'Kind, 'Tag, 'A> = value.Apply() :?> _
```

The only thing to take care of is that your IDE will show you now the type alias for the _Kind_. Something like `Kind<obj,Finished,int>` or `Kind<Finished<int>,Finished,int>`
instead of just `Finished<int>`

So are we done?  
Almost ...  
Well ...  
hmm ...
Nope!  

## What About Type Aliases?

This all will only work if (and only if) you control those _Kinds_ or you can apply extension methods. If however your _Kind_ is an type alias itself like this  

``` fsharp
type Cancelled<'a> = List<'a>
```

Then it cant work as you can not extend type aliases

``` fsharp
type Cancelled<'a> = List<'a>
    with
    member x.Kind(): ('a * Cancelled) = undef()
```

So we get this error

``` fsharp
Type abbreviations cannot have augmentations
```

### Sigh!  

What can we do?  
One solution would be to wrap the type into a single case discriminate union or a 1 field record

``` fsharp
type Cancelled<'a> = Cancelled of List<'a>
    with
    member x.Kind(): ('a * Cancelled) = undef()
```

This will work. However now you will have more ceremony on the call site as you now will have to wrap and unwrap your type all of the time

``` fsharp
let extract (Cancelled xs) : List<'a> = xs

let x = [1 2 3] |> Cancelled |> inj
let success02 = x |> prj |> extract
```

It might not be so bad but still we are already using _Brands_ as wrappers in order to encapsulate the concept of HKTs and now we also need an additional wrapper for the _Kind_ itself. Not good!  

## Build Your Own Type Checker

Luckily there is another way by building our own TypeChecker

First we start reordering our types, and then we get rid of the `Kind` members. Instead we define a projection function within each _Tag_

``` fsharp
type InProgress<'a> = {value: 'a; decision: Decision<'a>}
type InProgress () = 
    static member Prj(value: Brand<InProgress, 'A>) : InProgress<'A> = value.Apply() :?> _

type Finished<'a> = {value: 'a; initial: 'a; timestamp: int}
type Finished () = 
    static member Prj(value: Brand<Finished, 'A>) : Finished<'A> = value.Apply() :?> _

type Cancelled<'a> = 'a list
type Cancelled () =
    static member Prj(value: Brand<Cancelled, 'A>) : Cancelled<'A> = value.Apply() :?> _
```

Then we define a type checker type with lots of overloaded operator functions. For each _Kind_ we want to handle one.

``` fsharp
type TypeChecker = TypeChecker
    with
    static member ($) (TypeChecker, _ : Cancelled<'a>) : (Cancelled * 'a) = undef()
    static member ($) (TypeChecker, _ : Finished<'a>) : (Finished * 'a) = undef()
    static member ($) (TypeChecker, _ : InProgress<'a>) : (InProgress * 'a) = undef()
```

These overloaded operator functions will now be used in the same way that the `Kind` members where used previously.
And finally we define an inline wrapper to call those overloaded operators.

``` fsharp
let inline typeCheck x = TypeChecker $ x 
```

Let's implement our rather short injection function now

``` fsharp
let inline inj (value: 'Kind) : Brand<'Tag, 'A> = 
    let _ : ('Tag * 'A) = typeCheck value
    new Brand<_,_>(value)
```

**WTF?!** What kind if witch craft is this? What does `let _ : ('Tag * 'A) = typeCheck value` do?  

Effectively NOTHING!  
It is only there to trigger the checking of the compiler. Remember in our first example we bolted the type check into the `Kind` member function and
therefore we were able to check that as part of the function signature of our `(value: 'Kind)` parameter. When using type aliases as _Kinds_ that doesn't work anymore. Now we need to extract that type check into its own type and trigger it via polymorphic invoction.

The last part is the `prj` function

``` fsharp
let inline prj(value: Brand< ^Tag, 'A>) : 'Kind = 
    (^Tag : (static member Prj: Brand< ^Tag, 'A> -> 'Kind) (value))
```

This is relatively easy (if you disregard the invocation using generic type constraints) we simply relay to our `Prj` member functions define in the _Tag_ types.  
Let's run a few tests

``` fsharp
let x0 = {value = 1; initial = 2; timestamp = 3} |> inj
// val x0 :: Brand<Finished, int>
let y1 = [1; 2; 3] |> inj
// val y1 :: Brand<Cancelled, int>
```

This all works perfectly - also the type inferrence. Lets project a bit

``` fsharp
let y2 = y1 |> prj
// val y2 :: Cancelled<int>
```

Great - that works as well. The only thing is that somehow we ended up implementing `inj` and `prj` for every type. It is an improvement thou.
We dont need to create a seperate module for each type and we need not care which function can be applied to which type.

Are we done?  
Almost. Really!  

Just a word of warning: the last iteration I showed you was triggered by the fact that you can't attach extension methods to type aliases and in reality also the initial approach doesn't work in general if you are not in control of the type. The question now is what happens if you introduce another type alias into the system so that you

``` fsharp
type Cancelled<'a> = 'a list
type Cancelled () =
    static member Prj(value: Brand<Cancelled, 'A>) : Cancelled<'A> = value.Apply() :?> _

type Done () = 
    static member Prj(value: Brand<Done, 'A>) : Done<'A> = value.Apply() :?> _
and Done<'a> = 'a list
```

This is will of course confuse the type inferrer. So you need to add some type annotations

``` fsharp
let y1 : Brand<Cancelled, int> = [1; 2; 3] |> inj
```

The projection function will work as expected thou 

``` fsharp
let y2 = y1 |> prjX
```

This is it!

Follow me to the (soon to be published) [5th part of this series for a roundup and FAQ]()

If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com) or leave a 
[comment at the issue tracker of this repository](https://github.com/robkuz/robkuz.github.io/issues/1)
