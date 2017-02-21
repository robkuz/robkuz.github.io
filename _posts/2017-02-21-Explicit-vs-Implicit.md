---
published: true
layout: post
title: Explicit Is Better Than Implicit? I Disagree!
tags:
- F#
- explict parameter passing vs implicit parameter passing
---

On how to smooth your API for call site usage and make your API-Users happier.

## Typescript Sweetness

Typescript has a really nice feature where you can define a discriminate union like type with very little syntactic overhead

```Typescript
function foo(x: String|Boolean){}
```

(Wasn't that supposed to be a blog about F#?  Hold on young padawan. We're getting there!)

The usage of `foo` is pretty easy

```Typescript
foo("bar");
// or
foo(true);
//but the line below will not compile
foo(1);
```

To do something usefull with the param `x` within `foo` there needs to be code within `foo` that descriminates between the
types. A sort of pattern matching. Usually this looks similar to this

```Typescript
function foo(x: String|Bool){
    if (typeof(x) == "String") {
        someStringFun(x)
    }
    else {
        someNumberBool(x)
    }
}
foo("bar");
```

## The F# Way

In F# there is this notion that explicit is better than implicit. So in order to have something remotely similar in F# you have to do quite a bit of heavy lifting 
AND your call site will be wordy beyond belief.  
We start with a explicit discriminate union type as function parameters in F# need to be of exactly one type.

```fsharp
type FooParam =
    | BoolParam of bool
    | StrParam of string
```

Then we write the function that we want to have a pattern match like the Typescript example
```fsharp
let foo (x: FooParam) = 
    match x with
    | BoolParam y -> printfn "Bool: %A" y
    | StrParam z -> printfn "String: %A" z```

On the call site we then create an instance of the newly wrapper DU and feed it into the function.

```fsharp
foo (BoolParam true) |> ignore
```

Essentially this works. However we have forced our function user to KNOW about that wrapper and he or she needs to create instances for it.  
So far so good (or better ugly). Aren't there better ways? Well in Haskell ...

## Haskell Anybody?

("Oh no, oh no - not Haskell again" screams the young padawan in agony. To which I say sit still and learn!)

... in Haskell you have type classes for this.

```Haskell
class Foo a where
   foo :: a -> a
```

A Haskell class is nothing like an OO class rather something like an interface in Java only even more abstract. You *implement* that interface by creating an instance of it by supplying concrete types.

```Haskell
instance Foo Bool where
   foo x = x  --useless impl but you get the idea

instance Foo String where
   foo x = x
```

The cool thing about that is that every implementation for a specific type is completely detached from any other implementation for another type. And both implementations could be in totally different modules. 
Kinda like .NET extension methods. And then you just call the function with the parameter value of the needed type. No Wrapping no knowing about wrappers.

```Haskell
x = foo True
y = foo "Bar"
```

## Back To F# And Operator Overloading

So is there a way we can easen the pain of doing this in F#? Yes indeed [vote for this language suggestion on type classes](https://github.com/fsharp/fslang-suggestions/issues/243) 
and while you do I show you how to get something similar in F#.
First let me introduce you to .NETs implicit conversion operator. "Implicit conversion operator" you ask?  So it is possible after all?  
Let's see. The signature of that rare creature is `static member op_Implicit(x: 'a) : 'b`. Let's see what we can do with that.

```fsharp
type FooParam =
| BoolParam of bool
| StrParam of string
with 
    static member op_Implicit(x: bool) = BoolParam x
    static member op_Implicit(x: string) = StrParam x
```

After we have defined those overloaded methods we call our `foo` function without the wrapper.

```fsharp
let bar = foo true
```

Sadly no implicit conversion here - just an compiler error that `foo expects a FooParam not a bool`. So we still have to somehow force that conversion (aka make it explicit). 
For that to do we first create an inline function that calls that conversion function

```fsharp
let inline (!>) (x:^a) : ^b = ((^a or ^b) : (static member op_Implicit : ^a -> ^b) x)
```

and use that newly created operator explicitly.

```fsharp
let bar = foo (!> true)
```

That does work indeed now. A little bit too many parens for a non-lisp source code file. But OK.  
Wait! If we can use the operator `!>` explicitly why not move the operator into our function itself? Let's try that and also mark the function as inline.

```fsharp
let inline foo x = 
    let z = !> x
    match z with
    | BoolParam y -> printfn "Bool: %A" y
    | StrParam z -> printfn "String: %A" z
```

Now that looks promising: the inferred type is `val foo : x:'a -> unit (requires member op_Implicit)`.  
Calling `foo` yields the effects we expect without any errors.

```fsharp
let bar1 = foo "bar"
let bar2 = foo true
```
## Finishing touches

While we were massaging our type system we lost one ability however: `foo` can't handle `FooParams` anymore.
```fsharp
let bar1 = foo (StrParam "bar") //compile error now
```

This is easily solveable:  we simply need to add one more `op_Implicit` member to our type

```fsharp
type FooParam =
| BoolParam of bool
| StrParam of string
with 
    static member op_Implicit(x: bool) = BoolParam x
    static member op_Implicit(x: string) = StrParam x
    static member op_Implicit(x: FooParam) = x  // <- trivial 
```

Now calling `foo` with a `FooParam` works.

```fsharp
let bar1 = foo (StrParam "bar") //compile error now
```

## So everything is OK then?

Unfortunately not! All of the above will work when we have full control over the definition of our synthetic parameter type. But what if we are faced with a type that we want to convert to that
isn't under our control? Let's assume there is a type `ForeignModule.Result` which is defined as

```fsharp
module ForeignModule =
    type Result<'a, 'b> =
    | Success of 'a
    | Error of 'b
```

We can try to extend that type within our own module via extension methods

```fsharp
module OurModule =
    open ForeignModule
    type Result<'a, 'b> with
        static member op_Implicit(x: string) = Success x
```

However that wont work as operators can not be added via extension methods. sigh. For this to work we need to use a completely different approach.
We hace to create a type purely for transformation purposes and some static methods in it.

```fsharp
type ToResult = ToResult
with
    static member ($) (_: ToResult, l: string) : Result<string, string> = Success l
    //and again a method that just returns the input so we can handle strings and Results
    static member ($) (_: ToResult, l: Result<string, string>) : Result<string, string> = l
```

And than a helper method to trigger those static methods

```fsharp
let inline toResult x : Result<string, string> = ToResult $ x
```

Please dont ask me why we have to use the ($) here - It doesn't work any other way.  
Finally we write the function that uses our implicit conversion

```fsharp
let inline bar x = 
    let z = toResult x
    match z with
    | Success s -> sprintf "Success: %A" s
    | Error e -> sprintf "Error: %A" e
```

Young padawan, our lecture is coming to an end. A few parting words still ...

## Limits

There is some limitation when overloading and using generic type constraints. For example overloading on the same generic type with different generic params like this

```fsharp
type ToResult = ToResult
with
    static member ($) (_: ToResult, l: Result<string, string>) : Result<string, string> = ...
    static member ($) (_: ToResult, l: Result<int, string>) : Result<string, string> = ...
```

wont work and give you a compiler error telling that no fitting overloaded method could be found. This will happen at the call site and might be confusing if you don't know what is happening.

## Explict vs Implicit

I think when talking about explicit vs implicit type conversions we should in rather talk about conversions that we have control over (in principle) as opposed to conversions we can't control. 
Javascript's infamous `"" + true` coming to mind.
So yes explicit is necessary however that should most likely not happen at the call site but thru other means.

## Cast your votes

As you have seen - almost everything is possible already right now. However it takes quite a bit of heavy lifting and greets you with some strange errors sometimes. So please 
[vote for type classes](https://github.com/fsharp/fslang-suggestions/issues/243) or if you can't at least 
[vote for erased union types](hhttps://github.com/fsharp/fslang-suggestions/issues/538)

If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com)
