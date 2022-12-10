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

> This article is part of the 2022 F# Advent Calendar. Go check out the other awesome posts that are part of it [here](https://sergeytihon.com/2022/10/28/f-advent-calendar-in-english-2022/)!

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

So are you wondering how we can do something like this? Well, wonder no more, let's get our hands dirty!

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
    member inline _.Yield(()) = ()

    [<CustomOperation("header")>]
    member inline _.Header((), title: string) : Slide =
        [ TextBlock.create [ TextBlock.text title ] :> IView ]

let slide = SlideBuilder()
```

We need to also implement yield since it's a requirement for having custom operations working, but we don't actually want it to do anything so we'll return a unit from it. For the actual header we'll also take a unit as the initial argument and the title assigned to the header. The reason why we're taking a unit as the first parameter is that this parameter usually represents the _previous state_ of the expression. Taking a unit means that we don't want any previous state and therefore we force the `header` operation to be the first one in the expression and also allow to only have one header per slide.

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

We will start by defining a `DeckProperty` union type which will hold all the different content that a deck can accept. So far we only accept a title, so a simple branch of a string is all we need for now. We then define a yield that takes a unit and returns a unit, just like in the previous expression. The `title` custom operation simply produces a value of `Title` with the given string and then all that's left is defining a `Run` method that takes a prop and produces our domain Deck type.

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

By extending our union of properties and adding a `Yield` method that accepts a slide and extending the `Run` method to support slides, we can now run the code defined above by adding `yield` before the `slide` expression, and it works as well, but that's not what we want! We want to be able to add expressions without having to yield them.

In order to do so, we need to implement another method call `Delay`, whose signature is `(unit -> M<'T>) -> M<'T>`, which in our case would mean that given any function that takes a unit and returns a known wrapped type (like a slide!) we can turn it into our `DeckProperty` type. We will also define `Combine`, whose signature is `M<'T> * M<'T> -> M<'T>`, which is basically merging two properties together in our case: 

```fsharp
type DeckBuilder() =
    (* ... *)

    member inline _.Delay(f: unit -> DeckProperty) = f ()
    member inline _.Combine(props1: DeckProperty, props2: DeckProperty) = props2

    (* ... *)
```

Now you might be looking at that combine and thinking "aren't we basically discarding what was previously there?" and you're absolutely right, if we try to add two `slides` inside of the deck only the latest one will get to stay and the other one will be discarded. Let's fix this by producing a list of properties instead of just one single property:

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
- Changed the `Combine` method to take a list as the second parameter, so that we can accept all the previous properties that were added in the deck. That way we can append the new one to all the previous props.
- Created a new `Run` method that takes a list instead of a single property, and uses the previously defined one to fold the list of properties into one single deck. This is the method that will get called when the expression finishes, so that we don't have to do the folding later on.

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

# A brief introduction to Avalonia.FuncUI

For the UI part we'll use Avalonia, specifically [FuncUI](https://github.com/fsprojects/Avalonia.FuncUI) which is a more functional-friendly way of creating UIs using Avalonia and F#. Since we already have a project set-up let's just quickly get an app running, starting with the dependencies, we need the following ones:

```xml
<PackageReference Include="Avalonia" Version="11.0.0-preview3" />
<PackageReference Include="Avalonia.Desktop" Version="11.0.0-preview3" />
<PackageReference Include="Avalonia.Themes.Fluent" Version="11.0.0-preview3" />
<PackageReference Include="JaggerJo.Avalonia.FuncUI" Version="0.6.0-preview3" />
```

You can either add them manually under a `<ItemGroup>` in the fsproj file or add them through a NuGet UI if your IDE supports that. Once we have the dependencies ready, it's time to start defining how we want our API to look like.

# Abstracting the deck presentation

Once a user creates a deck with our DSL, I'd like to have a simple method call `showPresentation` that takes a deck and takes care of all the heavy lifting needed to get the Avalonia app ready and rendered. This is all great for the consumer, but we need to actually go ahead and define all this:

```fsharp
(* --- Here we will actually render the whole presentation later on --- *)
let root deck =
    Component(fun ctx -> TextBlock.create [ TextBlock.text deck.Title ])

(* --- Entrypoint --- *)
type MainWindow(deck: Deck) as this =
    inherit HostWindow()

    do
        base.Title <- "SharpPoint"
        base.MinWidth <- 1280.0
        base.MinHeight <- 720.0
        this.Padding <- Thickness(10, 30, 10, 0)
        this.Content <- root deck
        this.ExtendClientAreaToDecorationsHint <- true

type App(deck: Deck) =
    inherit Application()

    override this.Initialize() =
        this.Styles.Add(FluentTheme(baseUri = null, Mode = FluentThemeMode.Dark))

    override this.OnFrameworkInitializationCompleted() =
        match this.ApplicationLifetime with
        | :? IClassicDesktopStyleApplicationLifetime as desktopLifetime ->
            let mainWindow = MainWindow(deck)
            desktopLifetime.MainWindow <- mainWindow
        | _ -> ()

let showPresentation deck =
    AppBuilder
        .Configure(fun _ -> App(deck))
        .UsePlatformDetect()
        .UseSkia()
        .StartWithClassicDesktopLifetime([||])
    |> ignore
```

In our `showPresentation` function we take a deck and we pass it all the way down while we configure the Avalonia app for starting by loading the base styles (I chose the default fluent theme) and setting our root view as the main content of a 1280x720 window. That `ExtendClientAreaToDecorationsHint` hides the chrome of the window so that we have everything covered by the content, so that's why we also need a little padding on the content to not show them over the window controls. If we now use the `showPresentation` function with a deck, it'll show our title!

```fsharp
deck {
    title "A test"

    slide { header "Hello world!" }

    slide { header "Wow!" }

    slide { header "Much wow!" }
}
|> showPresentation
```

![A screenshot of the app running with just a title saying "A test"](/blog/img/ce-in-fsharp/initial-app-running.png)
