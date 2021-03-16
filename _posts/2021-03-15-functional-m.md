---
layout: post
date: 2021-03-15 20:17
title: Functional M
description: Functional techniques make Power Query/M code more readable
comments: true
category: 
- power query
tags:
- m
- power query
- functional programming
---
<figure>
    <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/b/b0/F_of_x.svg/1280px-F_of_x.svg.png" style="max-width: 841px;"/>
</figure>

Power Query (or rather M) is a functional language. This means simply that functions can be passed around just as any other variable would be, a function can for example take another function as input or return a new function.

I want to delve a bit deeper into this at some point but this post will give a very tangible example how applying functional techniques can make your code both easier to write and easier to read.

<!--more-->

## Untangling the nest

A colleague recently reminded me of [this](https://community.powerbi.com/t5/Desktop/Privacy-Hashing-of-keys/m-p/534546/highlight/true#M250677) post I had made on the Power BI forums years ago. It offers a, rather roundabout, way of generating hashes in Power Query. This comes in handy primarily when there's a need of anonymising data.

The solution looks like this (see post for details):

{% highlight powerquery %}
CalculateHash = (x as text) as text => Binary.ToText(
    Binary.FromList(
        List.FirstN(
            List.LastN(
                Binary.ToList(
                    Binary.Compress(Text.ToBinary(x, BinaryEncoding.Base64), Compression.GZip)
                ),
            8), 
        4)
     ),
BinaryEncoding.Hex)
{% endhighlight %}

Nested code like this is just awful to read, with the first operation on the function input hiding somewhere in the middle of `f8(f7(f6(f5(f4(f3(f2(f1(x))))))))`.

We can write this in a more idiomatic Power Query fashion, binding each step to a self-documenting identifier:  

{% highlight powerquery %}
CalculateHash = (x as text) as text => let
    #"Convert Text to Binary" = Text.ToBinary(x, BinaryEncoding.Base64),
    #"Compress binary data" = Binary.Compress(#"Convert Text to Binary", Compression.GZip),
    #"Convert binary to list" = Binary.ToList(#"Compress binary data"),
    #"Extract footer" = List.LastN(#"Convert binary to list", 8),
    #"Extract checksum" = List.FirstN(#"Extract footer", 4),
    #"Convert checksum to binary" = Binary.FromList(#"Extract checksum"),
    #"Convert checksum to hexadecimal" = Binary.ToText(#"Convert checksum to binary", BinaryEncoding.Hex)
    in #"Convert checksum to hexadecimal"
{% endhighlight %}

This is a good approach if you want to see each applied step in the Power Query editor (or if you want others to) but it's very hard to read the code itself. Both these examples make me question the typographical choices I made for this blog.

## Pipes

Working with R (or other more or less strictly functional languages such as F#) there is often the notion of a pipe operator. 

Say we want to take a list of numbers, filter out the odd ones, double the remaining even numbers and then reverse the list. Working with R and the Margittr package it looks something like:

{% highlight R %}
library(magrittr)
x <- seq(1,9) %>%
  Filter(function(x) x %% 2 == 0, .) %>%
  (function(x) x * 2) %>%
  rev
{% endhighlight %}

F# is much cleaner looking but the principle remains the same.

{% highlight fsharp %}
let x = 
    seq {1..9} 
    |> Seq.filter(fun x -> x % 2 = 0)
    |> Seq.map(fun x -> x * 2)
    |> Seq.rev
{% endhighlight %}

The code is written in the order in which the operations take place, the F# code has the almost magical quality of making sense even if you've never come across the language itself before.

Can we make Power Query behave in the same way? Yes we can (kind of)!

## Pipes in Power Query

We don't have the ability to implement new unary operators in Power Query/M (that is operators that sit in between its inputs, such as `+` or `-`) but as the language is functional we can come up with a pretty slick pretend solution.

The corresponding M syntax will look like this:

{% highlight powerquery %}
let x = Pipe({1..9})(
    each List.Select(_, Number.IsEven),
    each List.Transform(_, each _ * 2),
    List.Reverse
)
{% endhighlight %}

And, we can write the hash generator in the original example like this:

{% highlight powerquery %}
let Hash = Pipe(1234)(
      Text.From,
      each Text.ToBinary(_, BinaryEncoding.Base64),
      each Binary.Compress(_, Compression.GZip),
      Binary.ToList,
      each List.LastN(_, 8),
      each List.FirstN(_, 4),
      Binary.FromList,
      each Binary.ToText(_, BinaryEncoding.Hex)
  )
{% endhighlight %}

The `each` operator you see dotted around here is syntactic sugar (a shortcut plain and simple) for a one argument lambda (anonymous function) such as `(x) => ...`. 

This means that `each List.Select(_, Number.IsEven)` is just a slightly shorter version of `(x) => List.Select(x, Number.IsEven)`. The syntax should look familiar as you will see these anonymous functions being generated automatically by the Power Query user interface all the time.

So, how does it work? Here's how I went about it...

## Building a simple M pipe syntax

Let's start with a value `x`, to that value let's apply a function `f1`, and to the result of that function let's apply `f2`, then `f3` and so on. It then seems a good idea that we have an arbitrary length `list` of `function` to work with. 

We want to apply these functions in turn and then return a single value, which is a pefect task for the built-in `List.Accumulate` [function](https://docs.microsoft.com/en-us/powerquery-m/list-accumulate). 

The function syntax looks like this:

`List.Accumulate(list as list, seed as any, accumulator as function) as any`

As input it takes a list (this would be our list of functions), a seed (this will be our initial value `x`) and an accumulator function which we'll get to in a second.

A simple use case for`List.Accumulate` would be this, an alternative implementation of `List.Sum`:

{% highlight powerquery %}
let List.Sum2 = (list as list) => 
    List.Accumulate(
        list, 
        0, 
        (state, current) => state + current
    )
{% endhighlight %}

The accumulator function is a two argument lambda, the first argument captures the state of the operation while the second argument captures the current value in the list. With the `seed` argument set to `0` the very first calculation when running this over a list of the numbers 1-9 will then be `(0, 1) => 0 + 1` then  `(1, 2) => 1 + 2` and so on.

If the current value instead is a function we want to apply to the state (remember functions are just values) we simply define the accumulator as `(x, f) => f(x)`. 

And the heart of our final function is then:

{% highlight powerquery %}
List.Accumulate(
    functions, 
    object, 
    (x, f) => f(x)
)
{% endhighlight %}

We need to bind the `object` variable (i.e. the initial state) so we wrap it all up in the following function definition:

{% highlight powerquery %}
Pipe = (object as any) as function => 
    let RunPipe = (functions as list) as any => 
        List.Accumulate(
            functions, 
            object, 
            (x, f) => f(x)
        )
    in Function.From(type function (f as function) as any, RunPipe)
{% endhighlight %}

The `Pipe` function actually _return_ a function, hence the `Pipe(...)(...)` syntax when we invoke it. We could implement this as a two argument function that takes an object and a list but I personally find the `Pipe(..., {...})` option a bit less easy on the eyes.

This technique is called a closure, which means we can also apply a transformation in stages. For instance, let's define `let p = Pipe({1..9})` and then base multiple calculations based off this, e.g. `sumIsEven = p(List.Sum, Number.IsEven)` and `hasEvenNumberOfElements = p(List.Length, Number.IsEven)`.

The very last part of our pipe functions `Function.From(type function (f as function) as any, RunPipe)` is what allows the returned function to take an arbitrary number of arguments. `Function.From` returns a function which converts its arguments into a list and applies it to the function specified in the second argument.

And that's all there is to it, a few lines of code to save us writing many, many more.