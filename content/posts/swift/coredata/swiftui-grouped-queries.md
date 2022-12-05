+++
title = "Grouped queries in SwiftUI"
date = "2022-07-13T15:43:39+02:00"
author = "sleepyfran"
authorTwitter = "sleepyfran"
cover = ""
tags = [
  "coredata",
  "swiftui",
  "swift",
]
showFullContent = false
+++

## Querying

While developing [Maby](https://github.com/sleepyfran/maby/) I needed to retrieve all events that happened, grouped by dates. Now of course I could just slap a `FetchRequest` and then manually group the results into a dictionary, specially given how easy it is with the [grouping init](<https://developer.apple.com/documentation/swift/dictionary/init(grouping:by:)>), but it turns out that there's a better way.

Since iOS/iPadOS/tvOS 15, macOS 12 and watchOS 8 there's a new fetch request in town called [`SectionedFetchRequest`](https://developer.apple.com/documentation/swiftui/sectionedfetchrequest/) that allows us to specify a type to group by and an entity to group, so for example in my case I could just do the following:

```swift
@SectionedFetchRequest<Date, Event>(
	sectionIdentifier: \.start,
	sortDescriptors: [
		SortDescriptor(\.start, order: .reverse)
	]
) private var events: SectionedFetchResults<Date, Event>
```

`start` being the date property that I want to group by. With this, we can go ahead and use the results just like we would with a normal fetch request, but having to iterate two times: one for the groups, another for the items inside the group:

```swift
ForEach(events) { section in
	Section(header: Text(section.id, format: .dateTime)) {
		ForEach(section) { event in
			Text(event.name)
		}
	}
}
```

In the code above, `section.id` is the ID that we gave the `SectionFetchRequest` to group, so in my case it's a date and that's why I have to specify the `.dateTime` format for the text to show correctly. So the `section` is a struct that acts as a collection, that's why we can both grab the ID from it but also iterate over it with `ForEach`.

This would work if you chose something else than a date to group by, for example strings, numbers and the like work great for grouping, but with dates depending on how you store them and how you want to group them you'd find an issue just like I did here: I want to group by day, not by the whole date. An event that happened the 15th of August at 14:50 is the same as it happened at 14:30 for me. In order to fix this I simply introduced this computed property in my model:

```swift
@objc public var groupStart: Date {
	let components = Calendar.current.dateComponents(
		[.year, .month, .day],
		from: self.start
	)
	return Calendar.current.date(from: components)!
}
```

This basically retrieves just the day, month and year from the date and returns a Date from it, effectively removing the time from it. With this, changing the grouping property to this computed one makes it work great:

```swift
@SectionedFetchRequest<Date, Event>(
	sectionIdentifier: \.groupStart,
	sortDescriptors: [
		SortDescriptor(\.start, order: .reverse)
	]
) private var events: SectionedFetchResults<Date, Event>
```
_(Note that I kept the original date as the sort descriptor so that the events still show ordered by date with the time taken into account)_

## Implementing `onDelete`

With sectioned requests there's just a tiny more work involved to support the deletions on the items:

```swift
ForEach(events) { section in
	Section(header: Text(section.id, format: .dateTime)) {
		ForEach(section) { event in
			Text(event.name)
		}
		.onDelete { indexSet in
			indexSet.forEach {
				viewContext.delete(section[$0])
			}
		}
	}
}
```
_(Note that I'm calling the view context's delete method directly to keep the example small, feel free to change it for your preferred way of deleting)_

Basically we need to rely on the `section` that we're iterating to retrieve the items, since the `indexSet` that the `onDelete` function gives us is relative to the **current** group. However, since `section` is iterable we can just grab each element by its index, easy!
