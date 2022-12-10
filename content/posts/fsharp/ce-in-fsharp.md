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
type Slide = { Header: string }

type Deck = { Title: string; Slides: Slide list }
```

We'll start by defining simple types that we can expand upon later. First, our slides will consist of a list of _content_, which can be anything that can be displayed inside of a slide. We'll start by just supporting a header, but we can always expand it later given that we're defining a union type. For the deck, we'll have a title field which will be the initial view displayed once we load the deck into the program.

# Implementing our DSL

With this we can start defining our first computation expression: a slide. For now let's just add a `header` operation that will represent a header inside of a slide later on:

```fsharp
type SlideBuilder() =
    member inline _.Yield(()) = ()

    [<CustomOperation("header")>]
    member inline _.Header((), header: string) : Slide = { Header = header }

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
        newProp :: previousProps

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

You might be thinking now "does he not know that appending means putting the value at the end of the list and :: definitely does not do that?". Yes, I do! However keep in mind that expressions are evaluated from the bottom up, which means that we'll be combining the last value _first_ and only afterwards the rest, so that's why pre-pending values actually appends them to the end of the list in the end.

With this we can add multiple `slide` expressions inside of our deck without having to yield anything, cool! However there's one big gap: we can't add our previously defined `title` and then add slides, we get the following error:

> This control construct may only be used if the computation expression builder defines a 'For' method

So let's do that now!

```fsharp
type DeckBuilder() =
    (* ... *)
    member inline _.For(prop: DeckProperty, f: unit -> DeckProperty list) =
      prop :: f()
```

In the docs they specify that the signature for `For` is `seq<'T> * ('T -> M<'U>) -> seq<M<'U>>`, however that does not force us to specify the first argument as a sequence, it could be any "wrapped type" of T and since in our case we just have one property and we want to combine it with any function that returns a list of decks, we can just use a single property. As with `Combine`, keep in mind that expressions are evaluated from the bottom up, which is why we can prepend the new value to the previous ones.

And would you look at that, it works!

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

For the UI part we'll use Avalonia, specifically [FuncUI](https://github.com/fsprojects/Avalonia.FuncUI) which is a more functional-friendly way of creating UIs using Avalonia and F#. I won't get into details of the hows and whys of FuncUI, so if you want to learn more feel free to check out the repo linked above!

Since we already have a project set-up let's just quickly get an app running, starting with the dependencies, we need the following ones:

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

Let's move this first slide to its own component and start the process of handling more slides. I'll declare a state which holds the current slide and use it on the root component. I'll also declare a separate component for the initial slide where I just display a big text with the title of the slide centered in the screen. We'll declare the current slide as a separate type to have a clear separation between what's the initial slide and an actual slide that the user created instead of having to do tricks with the indexes of the slides:

```fsharp
type private CurrentSlide =
    | Initial
    | Slide of index: int

type private GlobalState = { CurrentSlide: IWritable<CurrentSlide> }
let state = { CurrentSlide = new State<CurrentSlide>(Initial) }

let initialSlide deck =
    Component.create (
        "initial-slide",
        fun _ ->
            StackPanel.create [
                StackPanel.horizontalAlignment HorizontalAlignment.Center
                StackPanel.verticalAlignment VerticalAlignment.Center
                StackPanel.children [
                    TextBlock.create [
                        TextBlock.fontSize 72
                        TextBlock.fontWeight FontWeight.Bold
                        TextBlock.text deck.Title
                    ]
                ]
            ]
    )

let private root deck =
    Component(fun ctx ->
        let currentSlide = ctx.usePassedRead state.CurrentSlide
        
        match currentSlide.Current with
        | Initial -> initialSlide deck
        | _ -> failwith "Coming soon!"
    )
```

The reason for declaring a global state instead of one scoped to the root view is that Avalonia by default listens to key events in the whole application window and if we want to listen to events inside of one component, we have to manually focus it. So let's just declare our state globally and modify it from the window.

We obviously have a couple of holes to fill in our implementation, so let's do that now!

## Keyboard navigation

Let's start by just displaying a mocked slide that will display its index so that we can get navigation working and then we can worry about actually displaying what the user chose. We'll assign the component the _slide-x_ ID so that FuncUI can [properly detect changes between the slides](https://funcui.avaloniaui.net/components/component-lifetime#component-identity-key):

```fsharp
let private slide (index: int) slide =
    Component.create(
        $"slide-{index}",
        fun _ ->
            TextBlock.create [
                TextBlock.text $"Slide number {index}"
            ]
    ) :> IView

(* ... *)

let private root deck =
    Component(fun ctx ->
        (* ... *)
        
        match currentSlide.Current with
        | Initial -> initialSlide deck
        | Slide idx ->
            deck.Slides
            |> List.item idx
            |> slide idx
    )
```

If we change the initial state to one of the slides, we'll see a simple slide displaying its index. Cool! Now that we have that in place we can implement navigation between slides with the keyboard. For that let's modify the MainWindow declaration to add a key down handler:

```fsharp
let hasSlideAvailableIn idx slides =
    slides
    |> List.tryItem idx
    |> Option.isSome

type MainWindow(deck: Deck) as this =
    (* ... *)

    override this.OnKeyDown event =
        let current = state.CurrentSlide.Current
            
        match current, event.Key with
        | Slide 0, Key.Left ->
            (* Go back to the initial slide *)
            Initial
        | Initial, Key.Right when List.isEmpty deck.Slides |> not ->
            (* Go to the first slide (if any) *)
            Slide 0
        | Slide index, Key.Left when deck.Slides |> hasSlideAvailableIn (index - 1) ->
            (* Go to the previous slide (if any) *)
            index - 1 |> Slide
        | Slide index, Key.Right when deck.Slides |> hasSlideAvailableIn (index + 1) ->
            (* Go to the next slide (if any) *)
            index + 1 |> Slide
        | _ -> current
        |> state.CurrentSlide.Set
```

I love how succinct pattern matching can be when combined with tuples! There's a bit of logic required to not go out of bounds from the list and also to handle the initial slide gracefully, so we basically pattern match through all the possibilities: 

- Pressing the right key while on the initial slide should go to the first slide of the presentation if there's any.
- Pressing right when there's more slides in the presentation should advance the current slide.
- Pressing left when there's still slides before the current one should go back to the previous.
- Pressing left when we're in the first user-declared slide should go back to the initial.
- Otherwise, keep it as it is.

I've abstracted the "if any" slide in a separate function to make the match easier to read. So now we can navigate between all our slides, awesome! Let's go and display the slides now.

## Displaying the actual slides

The only thing left is displaying the actual content of the slides instead of the mocked ones we created before. Since we only support the header we have it easy:

```fsharp
let private slide (index: int) slide =
    Component.create(
        $"slide-{index}",
        fun _ ->
            StackPanel.create [
               StackPanel.children [
                   if System.String.IsNullOrEmpty slide.Header |> not then
                       TextBlock.create [
                           TextBlock.fontSize 48
                           TextBlock.fontWeight FontWeight.Bold
                           TextBlock.text slide.Header
                       ]
               ]
            ]
    ) :> IView
```

And there we go, it works!

![A video recording of the app showing three slides that read "SharpPoint: Presentations made sharper" as the title, "This is the first slide", "...Wow, this is the second" and "NO WAY, a third?!"](/blog/img/ce-in-fsharp/slides-app.gif)
