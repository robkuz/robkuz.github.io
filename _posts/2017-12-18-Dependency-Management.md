---
published: false
layout: post
title: Dependency Management & Injection (3 + 1 Solutions revisited)
tags:
- F#
- Generic Wrappers
- Advent
---

# Dependency Management & Injection (3 + 1 Solutions revisited)

According to common lore the 3 hardest things in computer science are
- Naming things and
- cache invalidation

I must admit I disagree. For me the hardest thing is depencendy management - by far. Specifically if my app grows beyond a few KLOC.
Now one could assume that F# does a pretty good job in managing those dependencies. And in a way it does as long as we stay on the type level.
![dependency graph](/images/dependency-graph.jpg)
The dependency graph above shows how the type level information flows from No-Dependency on the library layer to Many-Dependencies on the programm level.
However as soon as we leave the type level and move into the value level the picture reverts. Usually the programm layer being an the borders of the system
creates values (often via some impure approach, think ports & adapters) and sends those values into the lower level functions. And so our first approach
of dependency management is to simply 

## 1) Inject Dependent Values via Function Parameter

```fsharp
let someBizFn depValue1 depValue realVal = ...
```

Now this works pretty fine ... as long as the function in the programm layer and the function from the business layer are relatively close to each other.
But imagine a situation where a function from the programm layer is pretty close to the top of the the dependency graph and (implictly) calls a function from business layer
that is rather at the very bottom of that layer and imagine a lot of functions in between. Then our code will easily look like this

```fsharp
module LowLevelBiz =
    let someBizFn depValue1 depValue realVal = ...

module HighLevelBiz =
    let someHighLevelBizFn depValue1 depValue2 andSome depA andSomeOther depB realVal = 
        let x = someOtherLowLevelBizFn andSome andSomeOther depB realVal
        let y = someBizFn  depValue1 depValue realVal
        // ...

module ProgrammLayer  =
    let someProgrammFn depValue1 depValue2 andSome depA andSomeOther depB andAlso thisDep andThat otherDep realVal = 
        let x = someHighLevelBizFn depValue1 depValue andSome depA andSomeOther depB realVal
        let y = someBizFn  depValue1 depValue realVal
        // ...
```

Having all these dependencies as singular params will lead to functions long param lists and become pretty unwieldly fast. So usually people start to 

## 2) Pack all Dependencies into a Type or Types

```fsharp
module HighLevelBiz =
    type SomeDependencyType = {
        depValue1 : DepValue1Type 
        depValue2 : DepValue2Type
        andSome   : AndSomeType
        depA      : DepAType
        depB      : DepBType
    }
```

and then there will be a redifined function and a call like this
```fsharp
module HighLevelBiz =
    let someHighLevelBizFn (deps:SomeDependencyType) realVal = 
        let y = someBizFn deps.depValue1 deps.depValue realVal
        //...

module ProgrammLayer =
    let deps = {defaultDeps with depValue1 = someDepValue1; depA = someDepAValue}
    someHighLevelBizFn deps andTheRealValue
```

However (depending on your domain and your application size) there is a high probability that there is another high level biz function with almost the same set of parameters.

```fsharp
module HighLevelBiz =
    let someOtherHighLevelBizFn depValue1 depValue2 andSome depA andSomeOther depC realVal = 
                                                                        //    ^^^^ Not depB but depC
        // ...
```

Naturally there will be the tendency to rewrite the type that contains the dependencies as

```fsharp
module HighLevelBiz =
    type SomeDependencyType = {
        depValue1 : DepValue1Type 
        depValue2 : DepValue2Type
        andSome   : AndSomeType
        depA      : DepAType
        depB      : Option<DepBType>  // <- now an option
        depC      : Option<DepCType>  // <- additional field
    }
```

Ok this works. However only if the types used within `SomeDependencyType` are from the library layer. As soon you it starts to contain types from the
Business Layers the compiler will start to complain. So that won't work but the temptation will be there. The solution will be then to define seperate *dependency* - types for each function so that

```fsharp
module HighLevelBiz =
    type DependencyTypeForSomeHighLevelBizFn = {
        depValue1 : DepValue1Type 
        depValue2 : DepValue2Type
        andSome   : AndSomeType
        depA      : DepAType
        depB      : DepBType
    }

    type DependencyTypeForSomeOtherHighLevelBizFn = {
        depValue1 : DepValue1Type 
        depValue2 : DepValue2Type
        andSome   : AndSomeType
        depA      : DepAType
        depC      : DepCBType
    }

    let someHighLevelBizFn (deps:DependencyTypeForSomeHighLevelBizFn) realVal = ...
    let someOtherHighLevelBizFn (deps:DependencyTypeForSomeOtherHighLevelBizFn) realVal = ...
```

So yeah we moved the long list of params into a type which makes the function definition and call site easier but then we have to define lots and lots of types. This is usally the point
where the temptation becomes very strong to 

## 3) Move all Dependency into an Untyped Map

```fsharp
module HighLevelBiz =
    let someHighLevelBizFn (deps:Map<string, obj>) realVal = 
        let y = someBizFn (deps.find("depValue1") :?> DepValue1Type) (deps.find("depA") :?> DepAType) realVal
        //...

module ProgrammLayer =
    let deps = Map.ofList [("depValue1", someDepValue1); ("depA", someDepAValue)]
    someHighLevelBizFn deps andTheRealValue
```

So we have a pragmatic solution and traded type safety and totallity for it. This seems to be a pretty steep price to pay.
So is there a solution where we don't loose all of the benefits? Yes there is:

## 4) Using Extension Methods

We can restore type safety and with a bit of discipline we can fake totality. Let's start by creating a `Map` like structure. We will call that `Context`.
No, no! That is not good - yeah naming is really hard. We will call it `ExecutionContext` :-P


```fsharp
type ExecutionContext () = 
    let mutable values: Map<string, obj> = Map.empty
    member this.Values
        with get() = values
        and set (vs: Map<string, obj>) = values <- vs
```

And then we define on the business layer some *typed* extension methods and use them in the functions

```fsharp
module HighLevelBizA =
    let keyDepValue1 = "HighLevelBizA.keyDepValue1"
    type ExecutionContext with
        member this.DepValue1
            with get(): DepValue1Type  = this.Values.find keyDepValue1 :?> _
            and set (value: DepValue1Type) = this.Values <- this.Values.Add (keyDepValue1, (box value)) 

    let keyDepA = "HighLevelBizA.keyDepA"
    type ExecutionContext with
        member this.DepA
            with get(): DepAType  = this.Values.find keyDepA :?> _
            and set (value: DepAType) = this.Values <- this.Values.Add (keyDepA, (box value)) 


    let someHighLevelBizFn (deps:ExecutionContext) realVal = 
        let x = deps.DebValue1
        ...

    let someOtherHighLevelBizFn (deps:ExecutionContext) realVal = 
        let y = deps.DepA
```

And finally on the programm layer we create and bind our dependencies like this

```fsharp
module ProgrammLayer  =
    open HighLevelBizA

    type SomeContext (a: DepValue1Type, b: DepAType) as this = 
        inherit ExecutionContext() 
        do
            this.DepValue1 <- a
            this.DepA <- b

    let someProgrammFn (deps: ExecutionContext) realVal = 
        let x = someHighLevelBizFn deps realVal
        let y = someBizFn deps realVal
        // ...
```

So now we gained type safety again and with a bit of discipline using a `do` block we can assume (aka fake)
totallity. But somehow this solutions seems to be pretty heavy weight in it's own. 5 lines of code for each dependency, really? 

Let me give you some heuristics on when to use this solution:
- there are functions with many dependencies (parameters)
- there are many functions that share the same dependencies but not all of them
- the functions are distributed across multiple modules

and finaly you have structured your code using (OO) interfaces with generic type parameters like this

```fsharp
type IDocPart<'delta, 'pures> = 
    abstract member Validate: Context -> 'pures -> Result<'pures, string>
```

And now imagine a document type that is a tree of `IDocPart`s where each implementor of that interface requires a (slightly) different set of dependencies. Solution 1) and 2) will simply not work then. Solution 3) and 4) will be the only games in town.

## 5) The Real Solution

Actually all of the approaches above bet on the fact that F# is not a pure language and that it is sort of acceptable to thread *impure* dependencies deep into your system. Now you can question that and honestly you should do question that and if you do question it you might end up writing some free Monads which Mark Seeman has written nicely about
[in Pure Times](http://blog.ploeh.dk/2017/06/27/pure-times/)

I hope you enjoyed the ride.  
If you have any comments drop me a note on twitter or via email. You'll find the contact info on my [homepage](http://www.robkuz.com) or leave a 
[comment at the issue tracker of this repository](https://github.com/robkuz/robkuz.github.io/issues/1)


PS: There are people out there that claim that "off by one errors" are also really hard. 
Using FP languages and mapping over structures I have found that this is not to be true.  
PPS: the code can be made a bit safer using reflection. Probably something for a second part of this blog.