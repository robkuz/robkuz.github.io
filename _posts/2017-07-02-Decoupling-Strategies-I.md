---
published: true
layout: post
title: Decoupling Strategies for F# Code Using Extension Methods and Object Expressions - Part I
tags:
- F#
- Extension Methods, Object Expressions
---

On how to implement orthogonal feature sets on generic types

## Simple Generic types

Assume the follwing generic type

```fsharp
type ItemState<'a> =
    | New
    | InChange of 'a 
    | InClearing of 'a
    | Done of 'a
```

You will certainly agree that this type communicates nicely whatever it needs pretty concisely.
Now life isn't as easy and some constraints within F# force us to implement some more members into the type directly 

```fsharp
type ItemState<'a> =
    | New
    | InChange of 'a 
    | InClearing of 'a
    | Done of 'a
    with
    static member Extract(s: ItemState<'b>) =
        match s with
        | New           -> None 
        | InChange d    -> Some d
        | InClearing d  -> Some d
        | Done d        -> Some d
    interface ICanHazValue with
        member this.HasValue() = 
            match this with
            | New -> false
            | _   -> true
```

Now that isn't as sweet as before but there are a few (good?) reasons that require the definition of these methods
directly on the type.
The `Extract` member method is used by the excellent [FSharpPlus](https://github.com/gusty/FSharpPlus) library.  
FsharpPlus uses statically resolved type parameters to call member methods and as #Fsharp doesn't pick up such member methods when they are defined as extension methods it needs to be defined directly on the initial type definition.  
Likewise F# (The CLR?) does not allow to implement interfaces as _extensions_. So we have to implement `ICanHazValue` again 
directly within our type.

## Let's have some extension
As our type is already overburdened with other, non-significant code we want to make sure that any new orthogonal features will be implemented outside that initial type definition. The goto way as of F# 4.1 is to use extension methods.  
Let's make our type understand JSON-serialization. However not only should our type understand JSON but also the referred generic type should know about JSON. It is sensible to assume that `ItemState<'a>` is part of a larger tree like structure that can be jsonfied in one go.  
Our first try might look like this

```fsharp
module JSON =
    type ItemState<'a> with
        member this.ToJson() =
            match this with
            | New           -> ["new",          Bool true] 
            | InChange d    -> ["in_change",    d.toJson()] 
            | InClearing d  -> ["in_clearing",  d.toJson()] 
            | Done d        -> ["done",         d.toJson()] 
            |> asObject
```

and will soon be greeted with the following error message on on `d.ToJson()` calls within the match statement.

```fsharp
[FS0072] Lookup on object of indeterminate type based on   
information prior to this program point. A type annotation may be  
needed prior to this program point to constrain the type of the  
object. This may allow the lookup to be resolved.
```

OK this makes sense - F# indeed can not determine the type of `d` in this context.  

## Statically Resolved Type Parameters (SRTPs)
We need to give the compiler some hints on what the generic type `'a` is. So our next iteration will use statically resolved type parameters and might look like this

```fsharp
type ItemState<'a when 'a: (member ToJson: unit -> Json)> with
    member this.ToJson() =
        match this with
        | New           -> ["new",          Bool true] 
        | InChange d    -> ["in_change",    d.toJson()] 
        | InClearing d  -> ["in_clearing",  d.toJson()] 
        | Done d        -> ["done",         d.toJson()] 
        |> asObject
```

However this made things only worse and instead of those 3 errors concerning `d` we now have 2 additional errors

## More Errors
```fsharp
type ItemState<'a when 'a: (member ToJson: unit -> Json)> with
   //^^^^^^^^^
   //[FS0957] The declared type parameters for this type extension 
   //do not match the declared type parameters on the original type //'ItemState<_>'
    member this.ToJson() =
         //^^^^^^^^^^^
         //[FS0670] This code is not 
         //sufficiently generic. The type variable  ^a when  ^a : 
         //(member ToJson :  ^a -> Json) could not be generalized 
         //because it would escape its scope.```
```
Let's start with the infamous __could not be generalized because it would escape its scope__ error. Infamous because I really don't understand what that means and because it can be usally remedied by simply `inline`ing the offending method. So we will do exactly that.

The other error is a bit easier to understand. We added that static constraint onto our type for only the definition of the extenison method. However the initial type definition does not define any such constraint. So the unconstrained initial type definition and the later constrained one are in opposition to each other (althou this is exactly what we want at the end of the day)

## Extending The inital type
So we add that SRTPs also to the initial type definition like this
```fsharp
type ItemState<'a when 'a: (member ToJson: unit -> Json)> =
    | New
    | InChange of 'a 
    | InClearing of 'a
    | Done of 'a
    with
    static member Extract(s: ItemState<'b>) =
        match s with
        | New           -> None 
        | InChange d    -> Some d
        | InClearing d  -> Some d
        | Done d        -> Some d
    interface ICanHazValue with
        member this.HasValue() = 
            match this with
            | New -> false
            | _   -> true
```
And tada! Our old friend __could not be generalized because it would escape its scope__ error greets us happily at  
```fsharp
static member Extract(s: ItemState<'b>)
```
And as always we resolve this error by inlining said method so that it reads  
```fsharp
static member inline Extract(s: ItemState<'b>)
```
Only to see that our old friend has wandered to the next method definition   
```fsharp
member this.HasValue()
```

Now let's inline this method definition as well.  
```fsharp
member inline this.HasValue()
```
Only - this does not work this time and we get an error telling us:  
```fsharp
[FS3151] This member, function or value declaration may not be declared 'inline'
```
Bummer!

## A Dead End
So our attempts to adorn everything with SRTPs and inlining has come to an a sudden halt and we need some other aproach.  
Before we dive into the concrete solution lets review what I have written before: F# does not allow the implementation of an interface within an extension context. Really?  

## Object Expressions
Well, yes sure - only we don't need to. We can easily implement an interface completely outside of a type by using object expressions. Let's try this
```fsharp
type ItemState<'a when 'a: (member ToJson: unit -> Json)> =
    | New
    | InChange of 'a 
    | InClearing of 'a
    | Done of 'a
    with
    static member inline Extract(s: XtemState<'b>) =
        match s with
        | New           -> None 
        | InChange d    -> Some d
        | InClearing d  -> Some d
        | Done d        -> Some d
    
let inline toICanHazValue (v:ItemState<_>) = {
    new ICanHazValue with
        member this.HasValue() = 
            match v with
            | New -> false
            | _   -> true
}
```

Well, that looks neat! Be aware thou that `toIcanHazValue` needs to be inlined as well. But apart from that everything looks nice.

So how does our extension method that triggered that whole rewrite looks like now?It should be without error, shouldn't it?

## Errors ? WTF!
Interestingly enough our initial error on `d.ToJson()` calls is still there. 

```fsharp
[FS0072] Lookup on object of indeterminate type based on   
information prior to this program point. A type annotation may be  
needed prior to this program point to constrain the type of the  
object. This may allow the lookup to be resolved.
```
Even thou we constrained the generic parameters all the way to the initial type definition. What a let down!  
We can fix that by throwing even more SRTPs-Foo onto it.

```fsharp
//uh,uh - Full-Blown SRTPs-Foo
let inline toJson (v:^T):Json = (^T: (member ToJson: unit -> Json) v)

type ItemState<'a when 'a: (member ToJson: unit -> Json)> with
    member inline this.ToJson() =
        
        match this with
        | New           -> ["new",          Bool true] 
        | InChange d    -> ["in_change",    toJson d] 
        | InClearing d  -> ["in_clearing",  toJson d] 
        | Done d        -> ["done",         toJson d] 
        |> asObject
```
And indeed now all of it compiles. puuh! All compiles? Really?

## Of New Types and Aliases
Yes it does if our program would be as simplistic as that.  
But let's assume that before we started to implement that orthogonal feature of JSON serialization into our code base we had used our type on other types. Something similar to this
```fsharp
module SomeOtherModule =
    open Types
    type Foo<'a> = Foo of  ItemState<'a>
```
Some new error again this time telling us
```fsharp
  [FS0001] The declared type parameter 'a' cannot be used here since the type parameter cannot be resolved at compile time
```
This one is luckily easily resolved by also adding the explict constrained on those types as well
```fsharp
module SomeOtherModule =
    open Types
    type Foo<'a when 'a: (member ToJson: unit -> Json)> = Foo of  ItemState<'a>
```
And if you happen to have functions with explicit generic parameters like  
```fsharp
`let baring<'a>(v: 'a, x:ItemState<'a>) = ...`  
```
you need to add the type constraints to those too. So it becomes  
```fsharp
let baring<'a when 'a: (member ToJson: unit -> Json)>(v: 'a, x:ItemState<'a>) = ...
```

And when we have done that for any traces that we find in our code base - then and only then everything will compile fine. Yeah! Finally!

## Review
Let's step back for a moment and review what we have done sofar

- We have defined a type with some additional member methods and an interface implementation
- then we wanted to implement some orthogonal feature set using extension methods so our initial type definiton could stay as concise as possible
- this led to the definiton of a type constraint on the extension method
- which led to the definition of a type constraint on our initial type definition
- which required us to adorn other types and functions that use our generic type to also have that type constraint
- we needed to inline a truckload of methods and functions
- and we needed to remove interface implementations and have them as object expressions (which is good thing by the way)

Wow!  

And that was just ONE orthogonal feature. Just imagine what would happen if you had multiple of those like `Transactionable`, `Memorize`, etc. I am sure that you will find half a dozen such features in any halfway significant program.

We will continue our journey soon to see which other approaches F# offers to us in the [2nd part of this series ]()

If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com) or leave a 
[comment at the issue tracker of this repository](https://github.com/robkuz/robkuz.github.io/issues/1)

PS: A student of this material might be tempted to ask  
Padawan: "Master, if this all undesireable, what did you show it to me and why didn't you show me the solution"  
Master: "Now to learn from my mistakes, you can"