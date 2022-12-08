+++
title = "Creating DSLs using F#'s Computation Expressions"
date = "2022-12-06T20:15:21.512Z"
author = "sleepyfran"
authorTwitter = "sleepyfran"
cover = ""
tags = [ "fsharp", "computation expressions" ]
showFullContent = false
draft = true
+++

Computation expressions (CEs from now on) are those things that sound so scary when you hear the name and even more scary when you see the definition on the official documentation:

> Computation expressions in F# provide a convenient syntax for writing computations that can be sequenced and combined using control flow constructs and bindings. Depending on the kind of computation expression, they can be thought of as a way to express monads, monoids, monad transformers, and applicative functors.

But once you get the hang of it, it could be a wonderful way to explore the many different ways by which you can create DSL or Domain Specific Languages in F#. In this post I'll go over some of the nicest DSL I've found so far in F# and we'll create our own that, inspired by [DeckUI](https://github.com/joshdholtz/DeckUI), will kick an [Avalonia](https://avaloniaui.net/) application that shows a presentation.

# First, some prior art

There's already quite a few really nice examples of DSL that use CEs, the one that immediately comes to mind is [FSHttp](https://github.com/fsprojects/FSHttp), which uses CEs to declare HTTP requests:

```fsharp
http {
    POST "https://reqres.in/api/users"
    CacheControl "no-cache"
    body
    jsonSerialize
        {|
            name = "morpheus"
            job = "leader"
        |}
}
```

Or projects like [Validus](https://github.com/pimbrouwers/Validus) that use the applicative nature of CEs to create powerful validations with a few single lines of code:

```fsharp:
let nameValidator = Check.String.betweenLen 3 64

let firstNameValidator =
    ValidatorGroup(nameValidator)
        .Then(Check.String.notEquals dto.LastName)
        .Build()

validate {
  let! first = firstNameValidator "First name" dto.FirstName
  and! last = nameValidator "Last name" dto.LastName
  and! age = Check.optional (Check.Int.between 1 120) "Age" dto.Age

  return {
      Name = { First = first; Last = last }
      Age = age }
}
```

# Defining our domain

Let's start by defining how we want our DSL to look like. We need to be able to declare two simple things: slides and decks. Slides will represent one individual slide inside of our presentation, while our deck will be the presentation itself and should hold all the slides. Let's get our domain defined:

```fsharp
open Avalonia.FuncUI.Types

type SlideContent = IView
type Slide = SlideContent list
type Deck = { Title: string; Slides: Slide list }
```

We'll be using the types defined on Avalonia directly because slides don't really need any other abstraction: we'll simply define our DSL as custom operations in the computation expression and transform that in each method to an actual view that we can then pass into Avalonia.

For the deck, though, we'll just have a title (which we'll use to populate the title of the window) and a list of slides that define the presentation.

# Implementing our DSL

With this we can start defining our first computation expression: a slide. For now let's just add a `header` operation that will represent a header inside of a slide later on:

```fsharp
type SlideBuilder() =
    member inline _.Yield(_) = []

    [<CustomOperation("header")>]
    member inline _.Header(state: Slide, title: string) : Slide =
        state
        @ [ TextBlock.create [ TextBlock.text title ] :> IView ]

let slide = SlideBuilder()
```

We need to also implement yield since it's a requirement for having custom operations working, but we don't actually want it to do anything so we'll return an empty array from it. For the actual header, though, we take a state, which is the previous list of views defined (for now empty because we don't support any other operation) and we combine with a TextBlock that contains the given title that the user passed.

Looking good! We can now use our slide builder like this:

```fsharp
slide {
    header "Hello world!"
}
```

And if we actually consume the result of the expression we will see that it's a list with one element, the TextBlock we used before. Cool, so far so good! We will be adding more stuff to the slides later, but let's now try to define a DSL for the deck itself. As we defined previously, our deck should have two parts: a title for the deck and a list of slides that we will show. Let's start with the title since it's the easy part.

```fsharp
[<RequireQualifiedAccess>]
type DeckProperty =
    | Title of string

type DeckBuilder() =
    member inline _.Yield(()) = ()
    member inline _.Run(DeckProperty.Title title) = { Title = title; Slides = [] }
    
    [<CustomOperation("title")>]
    member inline _.Title((), title: string) = DeckProperty.Title title

let deck = DeckBuilder()
```

We will start by defining a `DeckProperty` union type which will hold all the different content that a deck can accept. So far we only accept a title, so a simple branch of a string is all we need for now. We then define a yield that takes a unit and returns a unit, this is simply to indicate that we're in the beginning of the deck and there's nothing to produce yet. The `title` custom operation simply produces a value of `Title` with the given string and then all that's left is defining a `Run` method that takes a prop and produces our domain Deck type.

Cool! Trying out our expression so far produces a Deck type with no slides and "Test Deck" as the title:

```fsharp
deck {
    title "Test Deck"
}
```

Now, what about if we want to support a `slide` inside of our `deck`? Can we do that? Let's try out, let's add the previously defined slide inside of our deck and see what happens:

```fsharp
deck {
  (*
    ERROR: This control construct may only be used if the
           computation expression builder defines a 'Zero' method
  *)
  slide {
    header "Hello world!"
  }
}
```

Okay, let's take a step back and try to support _yielding_ slides inside of our deck:

```fsharp
[<RequireQualifiedAccess>]
type DeckProperty =
    | Title of string
    | Slide of Slide

type DeckBuilder() =
    member inline _.Yield(()) = ()
    member inline _.Yield(slide: Slide) = DeckProperty.Slide slide

    member inline _.Run(prop) =
        match prop with
        | DeckProperty.Title title -> { Title = title; Slides = [] }
        | DeckProperty.Slide slide -> { Title = ""; Slides = [ slide ] }

    [<CustomOperation("title")>]
    member inline _.Title((), title: string) = DeckProperty.Title title
```

Simply extending our union of properties and adding a `Yield` method that accepts a slide and extending the `Run` method to support slides allows us to execute the previous code by adding `yield` before the `slide` expression, and it works as well, but that's not what we want! We want to be able to add expressions without having to yield them.

In order to do so, we need to implement another method call `Delay`, whose signature is `(unit -> M<'T>) -> M<'T>`, which in our case would mean that given any function that takes a unit and returns a known wrapped type (like a slide!) we can turn it into our `DeckProperty` type, and `Combine` which is defined as either `M<'T> * M<'T> -> M<'T>`, which is basically merging two properties together in our case: 

```fsharp
type DeckBuilder() =
    (* ... *)

    member inline _.Delay(f: unit -> DeckProperty) = f ()
    member inline _.Combine(props1: DeckProperty, props2: DeckProperty) = x2

    (* ... *)
```

Now you might be looking at that combine and thinking "aren't we basically discarding what was previously there?" and you're absolutely right, if we try to add two `slides` inside of the deck only the latest one will stay. Let's fix this by producing a list of properties instead of just one single property:

```fsharp
let firstDefined str1 str2 =
    if System.String.IsNullOrEmpty str1 then
        str2
    else
        str1

type DeckBuilder() =
    (* ... *)

    member inline _.Delay(f: unit -> DeckProperty list) = f()
    member inline _.Delay(f: unit -> DeckProperty) = [f ()]
    
    member inline _.Combine(newProp: DeckProperty, previousProps: DeckProperty list) =
      previousProps @ [ newProp ]

    (* ... *)
        
    member inline x.Run(props: DeckProperty list) =
        props
        |> List.fold
            (fun acc prop ->
                let itemDeck = x.Run(prop)

                { Title = firstDefined acc.Title itemDeck.Title
                  Slides = acc.Slides @ itemDeck.Slides })
            { Title = ""; Slides = [] }
```

It might look like a bunch of changes but it's easier than it looks. We've just:
- Changed the `Delay` method to return a list given one single property and also to support functions that return lists, in which case we simply return the result.
- Changed the `Combine` method to take a list as the second parameter, so that we can accept all the previous properties that were added in the deck so that we can append the new one.
- Created a new `Run` method that uses the previously defined one and folds the list of properties into one single deck. This is the method that will get called when the expression finishes, so that the consumers just take a deck instead of a list of properties.

With this we can add multiple `slide` expressions inside of our deck without having to yield anything, cool! However there's one big gap: we can't add our previously defined `title` and then add slides, we get the following error:

> This control construct may only be used if the computation expression builder defines a 'For' method

So let's do that now!

```fsharp
type DeckBuilder() =
    (* ... *)
    member inline _.For(prop: DeckProperty, f: unit -> DeckProperty list) =
      [ prop ] @ f ()
```

In the docs they specify that the signature for `For` is `seq<'T> * ('T -> M<'U>) -> seq<M<'U>>`, however that does not force us to specify the first argument as a sequence since in our case we just have one property and we want to combine it with any function that returns a list of decks. And would you look at that, it works!

```fsharp
deck {
    title "Testing Deck with title"
    
    slide {
        header "This works"
    }
    
    slide {
        header "...and also this!"
    }
    
    slide {
        header "Much wow!"
    }
}
```

So that concludes our DSL, now let's go over the second part which is actually doing something useful with it.

# Interpreting our domain
