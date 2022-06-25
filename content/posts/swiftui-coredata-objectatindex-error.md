+++
title = "SwiftUI with CoreData: objectAtIndex unrecognized error"
date = "2022-06-21T22:42:09+02:00"
author = "sleepyfran"
authorTwitter = "sleepyfran"
cover = ""
tags = [ "swift", "swiftui", "coredata", "gotcha" ]
showFullContent = true
keywords = [
  "coredata",
  "swiftui",
  "swift",
  "objectAtIndex",
  "unrecognized selector sent to instance"
]
+++

Ah, if it isn't those awesome errors that make it fun to work with SwiftUI! So
in retrospective this one makes sense, but if you're like me who's just starting
out with CoreData and SwiftUI this error makes zero sense.

In essence I was getting this error when trying to do something like this:

```
2022-06-21 22:41:30.447763+0200 App[56115:1179524] -[__NSCFSet objectAtIndex:]: unrecognized selector sent to instance 0x600003df9c20
```

```swift
import CoreData
import SwiftUI

struct BudgetView: View {
    @FetchRequest(entity: CategoryGroup.entity(), sortDescriptors: [])
    var categoryGroups: FetchedResults<CategoryGroup>
    
    var body: some View {
        VStack {
            List {
                ForEach(categoryGroups, id: \.id) { group in
                    Text(group.name)
                    
                    ForEach(group.categories, id: \.id) { category in
                        Text(category.name)
                    }
                }
            }
        }
    }
}
```

So basically I have this `CategoryGroup` entity that contains a `name` field
which is a simple string (so far so good, if I just put it up until the
`Text(group.name)` it works), but it also contains the `categories` field which
is a list of the `Category` entity. Accessing the `categories` field gives us
the error above.

By default all To Many relationships in CoreData use the `NSSet` class, which
unless you check the `Ordered` option in the model it is not enumerable and therefore
you get the error above. Don't you love it when something that could have been a
compile error is instead a _right in your face_ error?

## The "easy" solution

In order to solve this, one solution is to enter in your CoreData model and go to the entity
that contains the To Many relationship with the other entity, in my case that's the
`CategoryGroup` entity that contains a To Many relationship to `Category`. Select
the relationship itself and on the right sidebar check the "Ordered" option:

![Xcode showing the Ordered option in the right sidebar](/blog/img/swiftui-coredata-objectAtIndex-error/swiftui-coredata-objectAtIndex-error_2022-06-21-22-54-49.png)

This will allow you to enumerate the content of the relationship thus making it
usable in the `ForEach`.

## A better way

In any case I found that interacting directly with the `NSSet` class can get a bit
complicated and we expose too much details to our UI that we shouldn't really be
dealing with. A better way is to add a computed variable to our model that automatically
does the translation for us into a more usable data structure like a `Set` or an
`Array` while also letting us apply a custom order. In my case I chose an array,
ordering the elements by their ID:

```swift
public var categoriesArray: [Category] {
    let set = categories as? Set<Category> ?? []
    return set.sorted {
        $0.id < $1.id
    }
}
```

So that means that now instead of doing the `ForEach` directly on the `categories`
property we have to use this computed one:

```swift
import CoreData
import SwiftUI

struct BudgetView: View {
    @FetchRequest(entity: CategoryGroup.entity(), sortDescriptors: [])
    var categoryGroups: FetchedResults<CategoryGroup>
    
    var body: some View {
        VStack {
            List {
                ForEach(categoryGroups, id: \.id) { group in
                    Text(group.name)
                    
                    /* The change is here */
                    ForEach(group.categoriesArray, id: \.id) { category in
                        Text(category.name)
                    }
                }
            }
        }
    }
}
```

And with that we should get no more complaints and our app will run happily.
