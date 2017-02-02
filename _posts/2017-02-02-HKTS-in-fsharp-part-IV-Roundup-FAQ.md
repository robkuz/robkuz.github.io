---
published: false
layout: post
title: Higher Kindended Types in F# Part IV - Roundup and FAQ
tags:
- F#
- Higher Kinded Types
---

 And how does that compare to Haskell or how the story about your and your PM would have played out.
 
- New State 1: adding a new state to F# or Haskell means adding a new type first of all. 
- New State 2: However in F# you will have to add 2 specific functions for injecting and projecting as well
- State Transitions 1: Write the given function `fromA2B`. Same for Haskell and F#. 
- State Transitions 2:It is a good idea in F# to write a wrapper function with the following implementation `fromBrandAToBrandB (b:Brand<A, 'x>) : Brand<B, 'x> = x |> prj |> fromA2B |> inj`
- Mapping over type that has HKT elements 1: you write a map. Kinda same for Haskell and F#. The F# version needing a member function and bein generally more wordy.
- Mapping over type that has HKT elements 2: For F# you now also have to transform the simple first class function into a full-blown type with a static member 

I hope I could convince you of the benefits of having higher kinded types in your code and the remaining question is: can we do something similar in F#?
Follow me to the [third installment of this series]() to find out about it.
