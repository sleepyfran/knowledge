---
weight: 1
title: "Formatters"
---

# Formatters

The easiest way to get a formatted element is to use the `formatted` method that is available on a myriad of data types without the need to instantiate our own formatter. And the best thing is that all these things are also available directly on SwiftUI controls like `Text` and `TextField` by using the `format` parameter.

Below are the most common/useful ones I've found so far.

## Dates
In order to get a date formatted, we can simply call `formatted` on a Date type, which gives us the full, long formatted date:

```swift
Date.now.formatted() // "7/16/2022, 1:15 PM"
```

Of course this by itself is not very useful, but we can customize how we want the date to be shown by adding parameters to the function call. If we add `.timeDate` as a parameter we'll get the same result, but now we can chain calls in a fluent fashion to customize the output, for example:

```swift
Date.now.formatted(
    .dateTime.hour().minute()
) // "1:15 PM"

Date.now.formatted(
    .dateTime.year().month()
) // "Jul 2022"
```

You can also pass parameters to each individual component:

```swift
Date.now.formatted(
    .dateTime.year(.twoDigits).month()
) // "Jul 22"
```

### Relative dates
Using the `relative` parameter, we can get how long it's been in a relative fashion since a date:

```swift
Date.now.addingTimeInterval(-15000).formatted(
    .relative(presentation: .numeric)
) // "4 hours ago"
```

### Date ranges
If you use `formatted` on a range of dates, you get the duration between the two dates:

```swift
let now = Date.now
let in16Minutes = Date.now.addingTimeInterval(1000)
(now..<in16Minutes).formatted(
    .components(style: .narrow)
) // "16min 40sec"
```

## Currencies
```swift
16.40.formatted(.currency(code: "EUR")) // "€16.40"
25.34.formatted(.currency(code: "USD")) // "$25.34"
```

Sadly for this one it doesn't seem to respect the currency's usual symbol position and instead uses the one specified in the user's location. In my case I have my computer language set to English US which by default positions the symbol in the left side while for Euros the symbol is usually on the right.

## Numeric
```swift
12.formatted(.number.sign(strategy: .always())) // "+12"
12.formatted(.percent) // 12%
```
