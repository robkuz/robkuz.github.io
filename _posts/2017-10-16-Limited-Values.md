---
published: true
layout: post
title: Creating Generic Wrappers for Validated Values
tags:
- F#
- Generic Wrappers
---

On how I started to refactor some duplicate code and ended with generic wrapper types

## How it all began
Being our projects janitor (amongst other jobs) I do some cleanup of our code base regularily.
Last week I came across the following module 

```fsharp
module WrappedString =
    type IWrappedString =
        abstract Value : string

    let create canonicalize isValid ctor (s:string) =
        if s = null
        then None
        else
            let s' = canonicalize s
            if isValid s'
            then Some (ctor s')
            else None

    let apply f (s:IWrappedString) = s.Value |> f

    let value s = apply id s

    let equals left right =
        (value left) = (value right)

    let compareTo left right =
        (value left).CompareTo (value right)

    let singleLineTrimmed s =
        System.Text.RegularExpressions.Regex.Replace(s,"\s"," ").Trim()

    let lengthValidator len (s:string) =
        s.Length <= len

    let numericValidator (s : string) =
        s.ToCharArray ()
        |> Array.exists (fun c -> c < '0' || c > '9')
        |> not

    type String100 = private String100 of string with
        interface IWrappedString with
            member this.Value = let (String100 s) = this in s

    let string100 = create singleLineTrimmed (lengthValidator 100) String100

    let convertTo100 s = apply string100 s

    type String50 = private String50 of string with
        interface IWrappedString with
            member this.Value = let (String50 s) = this in s

    let string50 = create singleLineTrimmed (lengthValidator 50)  String50

    let convertTo50 s = apply string50 s

    type NumericString8 = private NumericString8 of string with
        interface IWrappedString with
            member this.Value = let (NumericString8 s) = this in s

    let numericString8 = create singleLineTrimmed (fun n -> numericValidator n && lengthValidator 8 n) NumericString8

    let convertToNumeric8 s = apply numericString8 s
```

And there where more types like `String5`, `NumericString2` etc. and an additional module called `WrappedInteger` 
that did something very similar with integers.

I was shocked! Somehow my managerial communication skills must have taken a deep dive 
as obviously my teams was under the impression that they are being paid per lines of code.

## What's the Problem?
Now you might think "What's the problem? We have a lot of similar code in our code base!"

There are 4 problems I see here
- Code duplication (and lots of it and all problems that go along with it)
- Much of the implemetation that happens on the value level even thou the problem can and should be expressed on the type level
- Non generic types and values that lead to even more non generic code at the call site (and a polution of namespaces)
- And finally plain out wrong generic functions to somehow easen the pain of that mountain of code

Let's start a rewrite where we can create a generic solution that lives mostly on the type level as opposed on the value level.

## Goal
First we start with the definition of our goal:  
We want to create a generic wrapper forall types of 't that has an associated validator for 't that checks the validity of that value 't during creation time and the validator should configurable to validate different values of 't.

## Some Approaches
Let's try that

```fsharp
    type LimitedValue<'t> = LimitedValue of 't 
```

Now that was easy but sort of fruitless as the most important aspect was not met. __Validation during creation__.
Our next try might look like this:

```fsharp
type LimitedValue<'t>
    LimitedValue of 't 
    with
        static member Create(x:'t) : Option<LimitedValue<'t>> =
            x 
            |> runSomeValidation
            |> Option.Map LimitedValue
```

Now this code has some issues (apart from the simple fact that it doesn't compile).
Where does `runSomeValidation` come from?  
The usual approach within F# is to push that as a param of the `Create` function. But this would be really
bad as we would loose the type information right after a successful validation.

```fsharp
let strOfLen2 = LimitedValue.Create checkLenOf2 "yo"
let strOfLen4 = LimitedValue.Create checkLenOf4 "cool"
```

In the above code both values have the type `LimitedValue<string>`. That is not what we want as later code might
assume some properties for `'t` which are not true. The only way is to expose the validation requirements on the
`LimitedValue` too. In Order to do that lets first define an interface for validation.

```fsharp
    type Validator<'t>  =
        abstract member Validate: 't -> Option<'t>
```

So we have a validator that will return us `Some zValue` when valid or `None` if not. We could obviously also simple return a bool, however returning an `Option` will make composition much easier later on.
Still there is a problem. One of our requirements stated that the validator should be configurable. So lets change this.

```fsharp
    type Validator<'t, 'config>  =
        abstract member Validate: 'config -> 't -> Option<'t>
```

However this still doesn't solve our problem which we will see as we have another look into our `Create` function. For the moment we will assume that we somehow are able to create a validator within that function out of thin air.

```fsharp
static member Create(x:'t) : Option<LimitedValue<'t>> =
    x 
    |> someValidator.Validate someConfigValue
    |> Option.Map LimitedValue
```

Again this will not compile as `someConfigValue` is not defined (remember for this excercise we assummed the validator somehow does exist).
The only way to solve this would be to push the the configuration data as a parameter to the `Create` function like this `static member Create (config: 'config) (x:'t) : Option<LimitedValue<'t>>`.  
However doing this would end up with pretty much with the same problem as before. We would be able to create and validate the `LimitedValue` and its wrapped content however loose the information about what and how we validated immediately after the creation process.

## Refinement
Lets return back to our stated goal and refine it:

We want to create a generic wrapper forall 't  
that has an associated validator for 't  
that checks the validity of that value 't during creation time and  
that validator  __should configurable during compile time__ to validate different values of 't 
while maintaining the information on the validation requirements even after creation time.

## Another Approach
Now this leaves us with only 2 options in regards to the `Validator` interface

```fsharp
type Validator<'t, 'config>  =
    abstract member Validate: 't -> Option<'t>
    abstract member Config: 'config
```

Either we define an additional abstract property that we then implement for each concrete validator or

```fsharp
type Validator<'t, 'config> (config: 'config) =
    abstract member Validate: 't -> Option<'t>
```

We define the config data as a constructor parameter - which begs the question how we define the necessary value at compile time (but more on that later)?

We will opt for the second option as it much simpler to  configure something via a simple constructor parameter then
to have to implement an abstract method. And while we are at it - couldn't the `Validate` method (in the sense of a
.net _method_ but also in the meaning of an _approach_) be made easier?

Yes it can!

## The "Library Code"

```fsharp
type Validator<'config, 't> (config: 'config, vfn: 'config -> 't -> Option<'t>) =
    member this.Validate(x:'t) : Option<'t> = vfn config x
```

So we create an abstract `Validator` that later will be initialises with a configuration value (and its type)
a function that receives that configuration data and the actual value to be validated and returns an `Option` of that given value.

Now we can finally finish the `LimitedValue` type. First by merging our validator into the type by declaring it as one additional generic type parameters and then by implementing the `Create` method where a `Validator` instance is created out of thin air. ;-)

```fsharp
type LimitedValue<'validator, 't> = 
    LimitedValue of 't 
    with
        static member Create(x:'t) : Option<LimitedValue<'validator, 't>> =
            x 
            |> (new 'validator()).Validate 
            |> Option.Map LimitedValue
```

Trying to do this will yield a lot of swearing by the compiler. The generic type parameter `'validator` needs to be narrowed down via the infamous statically resolved type parameters (SRTP) and consequently when using SRTPs we also need to inline our `Create` method so we get

```fsharp
type LimitedValue<'v, 'c, 't    when    'v :> Validator<'c, 't>
                                and     'v : (new: unit -> 'v)> = 
    LimitedValue of 't 
    with
        static member inline Create(x:'t) : Option<LimitedValue<'v, 'c, 't>> =
            x 
            |> (new 'v()).Validate 
            |>> LimitedValue
```

All of this (a total of 10 lines of code) makes up our library code. 

## Building the Real Types

Now we can start to build our real types and their validators. We will start with a string of a max size of 5. However first one more trivial helper function

```fsharp
let private validate normalize fn v = if fn (normalize v) then Some (normalize v) else None
```

and then the first real validation function

```fsharp
let validateLen len s = validate trim (fun (s:string) -> s.Length <= len) s
```

and then the first `Validator` can be created by defining the config data type and the type of the wrapped value and the validation function to be used.

```fsharp
type LenValidator(config) = inherit Validator<int, string>(config, validateLen)
```

### Epiphany 
The empiphany I had while tinkering around with this smallish library is that we can __elevate concepts from the value level into the type level when we inherit from a type and supply default values to constructors__.

The `LenValidator` however is still only a "prototype" for different variations of the same. Our first concrete `Validator` will be one for validating strings of a max size of 5.

```fsharp
type Size5 () = inherit LenValidator(5) 
```

and finally the real `LimitedValue`

```fsharp
type String5 = LimitedValue<Size5, int, string>
let okString = String5.Create "short" // Some
let failString = String5.Create "much too long" //None
```

And if the need for another limited length string arises (say len of 20) we simply create a new concrete `Validator` and a type

```fsharp
type Size20 () = inherit LenValidator(20) 
type String20 = LimitedValue<Size10, int, string>
let okString = String5.Create "short"            //OK
let failString = String5.Create "much too long"  //OK
```

## What Else Can We Do?

But what if you want to create a totally different kind of `LimitedValue`?

Easy!

- create a configurable validation function
- create an _abstract_ `Validator` that is initialized with this validation function and therefore adjusted to your config data type and the input type
- create the concrete `Validator`s by providing the configuration data
- create concrete `LimitedValue`s by defining type aliases using the previously defined `Validator`s as generic parameters
- repeat steps 3 & 4 for each new variation

Lets try that with some integer range checking

```fsharp
    let validateRange (min,max) v = validate id (fun v -> v >= min && v <= max) v
    //use id as for ints we do not normalize anything

    type NumRangeValidator(config) = inherit Validator<int * int, int>(config, validateRange)

    type MaxPos100 () = inherit NumRangeValidator(0, 100)
    type RangeMinus100To100 () = inherit NumRangeValidator(-100, 100)

    type PositiveInt100 = LimitedValue<MaxPos100, int * int, int>
    type Minus100To100 = LimitedValue<RangeMinus100To100, int * int, int>
```

Or what about other types like an email or validating for the inclusion within a list?

```fsharp
type EmailValidator () = inherit RegexValidator("(^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$)")
type FsharpVersionValidator() = inherit ListValidator<float>([1.0;2.0;3.0;3.1;4.0;4.1])
```

## Final Touches
Before we finish we lets implement a few helper functions to make our life even easier and to take advantage of F#s type inference.

```fsharp
let inline mkLimitedValue (x: ^S) : Option< ^T> = (^T: (static member Create: ^S -> Option< ^T>) x)
let inline extract (x:^S) = (^S: (static member Extract: ^S -> ^T) x)
let inline convertTo (x: ^S) : Option< ^T> = (^T: (static member ConvertTo: ^S -> Option< ^T>) x)

//and in LimitedValue
    static member inline Extract (x : LimitedValue<'x, 'y, 'z> ) = let (LimitedValue s) = x in s
    static member inline ConvertTo(x : LimitedValue<'x, 'y, 'q> ) : Option<LimitedValue<'a, 'b, 'q>> = 
        let (LimitedValue v) = x
        mkLimitedValue v
```

and then we could do this

```fsharp
let a: Option<PositiveInt100> = mkLimitedValue 100
let b = a.Value |> extract
let c = a.Value
let c: Option<PositiveInt20000> = convertTo c

type Foo = { foo: PositiveInt100; bar: String5 }

let foo = { foo = (mkLimitedValue 50).Value ; bar = (mkLimitedValue "foo").Value}
```

I hope you enjoyed the ride and I hope you come up with lets of `LimitedValue`s.
I will create a github project to collect many of those. Send me a pull request if you have any that you like to share.

If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com) or leave a 
[comment at the issue tracker of this repository](https://github.com/robkuz/robkuz.github.io/issues/1)

PS: One thing I really like to improve is to get rid of that intermediate step of defining (naming) a Validator it would be extremely cool if one could do this instead

```fsharp
type Minus100To100 = LimitedValue<inherit NumRangeValidator(0, 100), int * int, int>
```

It's easy to envision a point where the naming of the validator (and also the final type) becomes pretty akward as one needs to express multiple validation requirements at once which at the end would be just spelling out literal values. Assume you would like to wrap the following structure and its Validator

```fsharp
type Vat = {country: string; started: int; value: int}
let validateVat config v = validate id (fun _ -> true) v
type VatValidator(config) = inherit Validator<Vat, int>(config, validateVat)
type DE_2007_19() = inherit VatValidator({country = "DE"; started = 2007; value = 19})
```
It would be so much nicer if one could express literal values directly on the type level. [Please follow this discussion on it](https://github.com/fsharp/fslang-suggestions/issues/608)

