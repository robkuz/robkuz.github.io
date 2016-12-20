---
published: true
---
#Creating Expressive Error Messages with Business Rule Functions

## A Journey of Syntax, Code Quotations, Refactorings and Monads

F# is pretty relaxed when it comes to naming identifiers. Identfiers can be anything as long as you put those identifiers in double backticks. For example the below is syntaticticly valid code

    let ``Creating Expressive Error Messages for Business Rule Functionsa`` : Blogpost = createBlogpost <| Some "crude ideas"

(Sadly its not valid semanticaly - I am still working on that lib to create blog posts for me)

Now wouldn't it be cool if we could create useful error messages for functions that check our business rules? Something that can be expressed by this type:

    type BusinessRule<'R> = 'R -> bool

Basically a function that takes any value checks it and then returns if the check was successful or not.   

##How could that work?

Also let's create a a type and some BusinessRules for it.

### Types

    type Invoice = {
        net: float
        vat: float
        gross: float
        county: string
    }

We will pretend that the above is some really neat design ... 

### Business Rules
And the rules might look similar to this

    let ``gross = net * (1 + vat)`` (inv: Invoice) : bool = inv.gross = inv.net * (1 + inv.vat)
    let ``vat value and country must match`` (inv: Invoice) : bool  = // ... not really interesting
    let ``country must exist`` (inv: Invoice) : bool  = // ... lookup inv.country etc.

There are some nice things to be Aware of about those biz rules

- The name of the function can contain pretty much any char, operator or keyword we like without getting an syntax error.
- depending on the business rule the rule name and the rule implementation could be pretty similar

### Using the Business Rules

A very naive (but straight forward) way to use These Business rules would be like in the following Code

    let update lens value invoice : invoice =
        let updated_inv = set lens value invoice // just ignore this and assume that we get an updated invoice here
        if not (``gross = net * (1 + vat)`` updated_inv then failwith "'gross = net * (1 + vat)' failed"
        if not (``vat value and country must match`` updated_inv then failwith "'vat value and country must match' failed"
        if not (``country must exist`` updated_inv then failwith "'country must exist' failed"
        updated_inv

## Critique

Wow this code has quite some issues

1. It is very procedural not a bit of functional
2. its hugely verbose instead of concise
3. it throws exceptions and requires callers to use nasty try-catche-blocks
4. it can only notify about one error at a time but not about multiple ones
5. the error message and the name of the function could diviate over time when the name of the function gets updated

## Refactor

### Synchronizing function name and error message

Let's refactor the code above so we get to a code base that doesn't have the issue listed above

First we write a wrapper for executing our business rule function. The easiest approach would be

    let check (f: 'a -> bool) v : bool = 
        if not <| f v then failwith (sprintf "%A failed" (f.GetName()))
                                                        //^^^^^^^^^^^^ That's not gonna work
        true

Only that doesn't work! Functions (Function-Types) in F# dont return their names but some kind of mangled garbage. Therefore the only way is to fall back on quotation expressions gymnastics. So our next approach is

    let check (f: Expr<'a -> bool>) v : bool = 
        let getFn (e:Expr) : MethodInfo =
            let rec get' e =
                match e with
                    | Call (_, mi, _) -> mi
                    | Lambda (_, body) -> get' body
                    | _ -> failwith <| sprintf "not a function %A" e
            get' e

        let name = (getFn f).Name
        if not <| f v then failwith (sprintf "%A failed" name)
        true

But that doesnt work either: `f` now is an expression not a function anymore and can't be invoked!  
We could of course use the `MethodInfo` object and invoke this. However invoking on `MethodInfo` is pretty messy on .NET particular in the presence of generic type parameters.  
So is there any onther way? Luckily there is: the quotation Compiler (https://www.nuget.org/packages/FSharp.Quotations.Compiler).  
This can compile `Expr<'a -> bool>` into `'a -> bool`. So we will refactor `check` again moving the quotation code outside and using the `QuotationEvaluator`

    let getFn (e: Expr<'a -> bool>) : MethodInfo =
        let rec get' e =
            match e with
                | Call (_, mi, _) -> mi
                | Lambda (_, body) -> get' body
                | _ -> failwith <| sprintf "not a function %A" e
        get' e

    let check (e: Expr<'a -> bool>) v : bool = 
        let (fn, name)  = (QuotationEvaluator.Evaluate e, (getFn e).Name)
        if not <| fn v then failwith (sprintf "%A failed" name)
        true

And our `update` function can now be easily rewritten 

    let update lens value invoice : invoice =
        let updated_inv = set lens value invoice 
        check <@ ``gross = net * (1 + vat)`` @> updated_inv
        check <@ ``vat value and country must match`` @> updated_inv
        check <@ ``country must exist`` @> updated_inv
        updated_inv

So we already slashed quite a bit of verbosity - yet its still not very functional.  

### Functionalize

We can do this by simply reordering the parameters of `check` and the rewriting `update` using pipes and maps

    //reordered params v <-> e
    let check v (e: Expr<'a -> bool>) : bool = 
        let (fn, name)  = (QuotationEvaluator.Evaluate e, (getFn e).Name)
        if not <| fn v then failwith (sprintf "%A failed" name)
        true

    let update lens value invoice : invoice =
        let updated_inv = set lens value invoice 
        [
            <@ ``gross = net * (1 + vat)`` @>
            <@ ``vat value and country must match`` @>
            <@ ``country must exist`` @>
        ]
        |> List.map (check updated_inv)
        updated_inv

Now this is getting nice. We are now able to extract the names of the functions and use them in our error messages, are code is pretty dense and we moved to a functional design.

### Wrapping the error

Now lets see how we can get rid of errors by throwing exceptions. For this to happen we need to use something like Haskell's Either monad. In F# 4.1 such a type will be included.
For now and as an excercise we will build our own

    type Result<'TSuccess, 'TError> = 
        | Success of 'TSuccess 
        | Error of 'TError
        static member Extract (x : Result<'TSuccess, 'TError> ) = 
            match this with 
            | Success s -> (Some s, None)
            | Error e   -> (None, Some e)

And a few helper functions to make it easiert to work with pipes

    let extract (r:Result<_,_>) = Result.Extract(r)
    let isError (r:Result<_,_>) = r |> extract |> fst = None

Now we rewrite `check` so it returns a `Result` instead of throwing an exception 

    let check v (e: Expr<'a -> bool>) : Result<'a, string> = 
        let (fn, name)  = (QuotationEvaluator.Evaluate e, (getFn e).Name)
        if fn v then Success v else Error (sprintf "%A failed" name)


and another helper function to execute the business tules join the errors (if any) or return the value

    let checkRules bizrules v  =
        let hasErrors = isError |> List.filter >> List.isEmpty >> not
        let joinErrors (xs:Result<'a, 'b> list) = xs |> List.map (extract >> snd) |> List.fold (+) ""

        let res  = bizrules |> List.map (check v)
        if hasErrors res then
            joinErrors res |> Error
        else
            v |> Success


Finally  we rewrite the `update` function as

    let update lens value invoice : invoice =
        set lens value invoice 
        |> checkRules 
                [ 
                    <@ ``gross = net * (1 + vat)`` @>
                    <@ ``vat value and country must match`` @>
                    <@ ``country must exist`` @>
                ]

That looks neat and tidy!

Is that it?

## And nor for something completely different ... The Larch

Sometimes you might want to use all of that plumbing above but still Exit from the process as soon as you get  first error.  
How can this be done? First we need to extend the `Result` type

    type Result<'TSuccess, 'TError> with 
        static member Bind (x:Result<'T,_>, f:'T->Result<'U, _>) : Result<'U,_> = 
            match x with 
                | Success s ->  f s
                | Error e -> Error e

We create operator for calling that method

    let (>>=) x f = Result.Bind(x, f)

And once more we reorder parameters. This time using `flip`

    let check' :  'T -> Result<'T, string> = flip check

And then we rewrite `update` one last time (I promise)

    let update lens value invoice : invoice =
        set lens value invoice 
        |>     check' ``gross = net * (1 + vat)``
        >>= check' ``vat value and country must match``
        >>= check' ``country must exist``

That's it - really! I hope you enjoyed the show.
