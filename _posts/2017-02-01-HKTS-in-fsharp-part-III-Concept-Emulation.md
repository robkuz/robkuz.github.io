---
published: true
layout: post
title: Higher Kindended Types in F# Part III - Concept Emulator
tags:
- F#
- Higher Kinded Types
---

I hope you liked the previous parts of this series 

- [Part I - What is the Problem Anyways?](https://robkuz.github.io/Higher-kinded-types-in-fsharp-Intro-Part-I/)
- [Part II - A Short Visit to Haskell Land](https://robkuz.github.io/2017-01-31-HKTS-in-fsharp-Part-II-A-Short-Visit-To-Haskell-Land.md)
- Part III - Concept Emulator 
- [Part IV - Roundup and FAQ]() soon to be published

Are you ready to see how you can do those damned Higher Kinded Types in F#?  
Before we start please be aware that as HKTs are not a native concept to F# our solution is clumsy and far from Haskell's elegance. 
It requires quite a bit of plumbing and it is leaky as it will show on call sites.

You can read about the general idea [in the paper "Lightweight higher-kinded polymorphism"](https://ocamllabs.github.io/higher/lightweight-higher-kinded-polymorphism.pdf). 

In F# we will have to introduce one or multiple explicit types for so we can defunctionalize. 
We will call this kind of type `Brand`(s) as in the paper.

``` fsharp
type Brand<'a, 'b> (value:obj) =
    member self.Apply() : obj = value   
```
That's it. No really! On a very basic level thats all that is needed. 
Lets move on with our business domain.

``` fsharp
type Decision<'a> = Decision of 'a
type InProgress<'a> = {value: 'a; decision: Decision<'a>}
type Finished<'a> = {value: 'a; initial: 'a; timestamp: DateTime}
```

Nothing new here sofar but now we have to create some marker classes that mirror our states in our business domain.

``` fsharp
type InProgress private  () = class end
type Finished private () = class end
```

And 2 helper functions to make live a bit easier on the call site. This helpers will be repsonsible to project and inject our real values from the `Brand`. 
Accourdingly they are named `prj` and `inj`.

``` fsharp
module INP =
    let inline inj (value: InProgress<'A>) : Brand<InProgress, 'A> = new Brand<_,_>(value)
    let inline prj (value: Brand<InProgress, 'A>) : InProgress<'A> = value.Apply() :?> _

module FIN =
    let inline inj (value: Finished<'A>) : Brand<Finished, 'A> = new Brand<_,_>(value)
    let inline prj (value: Brand<Finished, 'A>) : Finished<'A> = value.Apply() :?> _
```

So for every type that we want to push into the HKT domain we create on the one hand a marker class and a set of `inj` and `prj` functions 
to make working with the brands easier on the call site. Armed with these tools we can already harvest much of what HKTs has to offer. 
Let' see the code in action:

``` fsharp
let x: Brand<InProgress, int> = {value = 1, Decision 2} |> INP.inj
let y: InProgress<int> = x |> INP.prj
```

Now this is already cool. The call site ceremony isn't exactly nice looking but using F#s pipe operator acceptable. 

How about we write transition function from `Brand<InProgress, 'a>` to `Brand<Finished, 'a>`?

``` fsharp
let applyDecision (Decision d) v = d

let toFinishedStructure ({value = v; decisions = d}: InProgress<'a>) : Finished<'a> =
    {value = applyDecision d v; initial = v; timestamp = DateTime.Now}

let toFinishedBrand (x: Brand<InProgress, 'a>) : Brand<Finished, 'a> = 
    x 
    |> INP.prj
    |> toFinishedPure
    |> FIN.inj

let a = {InProgress.value = 1; decision = Decision 2} |> INP.inj
let b = toFinishedBrand a
```

Wow! That ain't so bad. As I said not as elegant in Haskell but still... 
If you want HKT and you work with homogeneous sets of those what you have seen so far will carry you quite a long way.

But what if we have a heterogeneous set of HKTs like in our LineItem example? 

``` fsharp
type LineItem<'a> = {
    articleID: Brand<'a, string>
    units:     Brand<'a, int>
    amount:    Brand<'a, float>
}
let someLineItemProgress: LineItem<InProgress> = someCreateFn()
```

The easiest way would be to simply define a `map` function and call it with some transition function

``` fsharp
let mapLineItem f li = {
        articleID   = li.articleID |> f
        units       = li.units |> f
        amount      = li.amount |> f
    }

mapLineItem toFinishedBrand someLineItemProgress
```

Only that doesnt work and gives us an error

``` fsharp
let mapLineItem f li = {
    articleID   = li.articleID |> f
    units       = li.units |> f
                                ^^^^^^^^^^
                                Type mismatch. Expecting a
                                'Brand<'a,int> -> 'b'    
                                but given a
                                'Brand<'a,int> -> 'd'    
                                The type 'string' does not match the type 'int'
    amount      = li.amount |> f
}
```

This is because the type inferrence algorithm looks first at the application of `articleID   = li.articleID |> f` and infers the type of
`f` to be `(Brand<'a,string> -> Brand<'c,string>)`. Only that is not true. The type of `f` should be `(Brand<'a, 'x> -> Brand<'c,'x>)`. 
We would need a way to inline first class function parameters: `let mapLineItem (inline f) li = `. 
Sadly that is not possible in F#. In Haskell can do this only by using the `{-# LANGUAGE RankNTypes #-}` compiler directive. 
Which means: "allow polymorphic functions as first class paramters". 
Haskell expresses this via function signatures similar to this one `map :: forall f g. (forall a. f a -> g a) -> R f -> R g`.

Can we get that into F# nevertheless? Yes - but now it's getting ugly. 
Let's start with some helper function and a member method on our to type to be mapped over.

``` fsharp
let createSafe (parent:obj) (prop:Lazy<'t>): 't = if not <| isNull parent then prop.Force() else Unchecked.defaultof<'t>

type LineItem<'a> = 
with        
    member inline self.Imap(invoker, s:LineItem<'a>): LineItem<'b> =
        {
            articleID = invoker $ (createSafe s (lazy(s.articleID)))
            units =     invoker $ (createSafe s (lazy(s.units))) 
            amount =    invoker $ (createSafe s (lazy(s.amount))) 
        }
```

First we create a helper function for lazily creating properties on any object if it is null. This is particular helpful if you will create new objects of deeply nested HKTs.
And an inlined (member) function `imap` that is used to map over our type. Why `imap` you ask? Because its not really a map function. At least not from a signature point of view.
Only now we are ready to implement the element mapping. And wow this is getting really, really heavy! Oh F#! <sigh>

``` fsharp
type ToFinish = ToFinish with 
    static member inline ($) (ToFinish, b) = toFinishedBrand b

let inline imap (invoker: ^I) (s:^S) : ^R = (^S: (member Imap: ^I -> ^S -> ^R) (s, invoker, s))

let inline toFinished li = imap ToFinish li
```

So we need a type with one single static member that is also inlined where we implement our mapping. puuuh!
The `imap`function is only a helper (and can be reused) to create the final `toFinish` wrapper function so this whole mess can be easily used on the call site.

And finally let's use that all

``` fsharp
let ili : LineItem<InProgress> = 
    {
        articleID = {InProgress.value = "0815"; decision = Decision "0815"} |> INP.inj
        units =     {InProgress.value = 4; decision = Decision 5} |> INP.inj
        amount =    {InProgress.value = 9.99; decision = Decision 9.9} |> INP.inj
    }

let fli : LineItem<Finished> = toFinished ili
```

I'd say that looks good. At least on the call site! 
I hope you enjoyed the show.

Follow me to the (soon to be published) [4th part of this series for a roundup and FAQ]()

If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com)