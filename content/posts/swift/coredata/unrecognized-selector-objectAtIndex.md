+++
title = "Unrecognized selector `objectAtIndex`"
date = "2022-06-21T11:43:39+02:00"
author = "sleepyfran"
authorTwitter = "sleepyfran"
cover = ""
tags = [
  "coredata",
  "swiftui",
  "swift",
  "gotchas"
]
showFullContent = false
+++

# Unrecognized selector `objectAtIndex`
```
2022-06-21 22:41:30.447763+0200 App[56115:1179524] -[__NSCFSet objectAtIndex:]: unrecognized selector sent to instance 0x600003df9c20
```

This happened with the following code, basically when trying to do a `ForEach` on a property coming from a CoreData model:

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

This property is actually declared manually as an array:

```swift
public final class CategoryGroup: NSManagedObject, Identifiable {
    @NSManaged public var categories: [Category]
}
```

But even though _I_ declared it as an array apparently the underlying type that CoreData actually puts there is an `NSSet`, which by default does not allow to retrieve stuff by its index (does that unrecognized selector `objectAtIndex` makes a bit more sense now?)

The easy solution is to mark the property as "Ordered" in the CoreData model. The **proper** solution, however, is to not interact with `NSSet` directly unless we can avoid it, so we can simply introduce a computed variable that transforms our property into an array (or whatever other data type we need!):

```swift
public var categoriesArray: [Category] {
    let set = categories as? Set<Category> ?? []
    return set.sorted {
        $0.id < $1.id
    }
}
```

This casts the `NSSet` into an array sorting it by their ID. Now instead of doing the `ForEach` directly on the `categories` property we have to use this computed one:

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
