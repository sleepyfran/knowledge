+++
title = "Formatters in SwiftUI"
date = "2022-06-21T20:30:00.047Z"
author = "sleepyfran"
authorTwitter = "sleepyfran"
tags = [ "programming", "swift", "swiftui" ]
showFullContent = true
+++

One of the most common issues I had with SwiftUI were always related to handling
different data types inside of the `Text` or `TextField` controls, specially
when handling numbers like `Decimal` where you'd need to manually specify a
`NumberFormatter` with the correct settings and yada yada yada.

Luckily since iOS 15, iPadOS 15 and macOS 12 it's possible to simply pass a format to the `Text` and `TextField` controls and it takes care of displaying it
correctly by itself. This means that we can, for example, create a text field
that formats itself to display the amount in US Dollars with this little amount
of code:

```swift
import SwiftUI

struct ContentView: View {
    @State var accountBalance: Decimal = 1500.24
    
    var body: some View {
        VStack(alignment: .center) {
            TextField(
                "Enter your balance",
                value: $accountBalance,
                format: .currency(code: "USD")
            )
            .textFieldStyle(.roundedBorder)
        }
    }
}
```

Which displays this:
![iOS Simulator displaying a TextField control formatting the content in US Dollars](/blog/img/swiftui-formatting/swiftui-formatting_2022-06-21-22-26-30.png)

And of course it supports all the currency codes that you could originally put
on a `NumberFormatter`, as easy as changing `USD` for `GBP` and we got ourselves
a nice looking Text Field that displays its content in Pounds:

![iOS Simulator displaying a TextField control formatting the content in Pounds](/blog/img/swiftui-formatting/swiftui-formatting_2022-06-21-22-26-56.png)

This also works by using the `format` parameter in a `Text` control:

```swift
import SwiftUI

struct ContentView: View {
    @State var accountBalance: Decimal = 1500.24
    
    var body: some View {
        VStack(alignment: .center) {
            Text(accountBalance,format: .currency(code: "USD"))
        }
    }
}
```

![iOS Simulator displaying a Text control formatting the content in US Dollars](/blog/img/swiftui-formatting/swiftui-formatting_2022-06-21-22-25-53.png)

And these shortcuts are just for the base case, if you require any more configuration
on top of that you can use the `FormatStyle` directly and configure it to your
liking. For example, this next snippet shows the amount formatted for US Dollars
always displaying the sign and scaling the amount 4 times:

```swift
import SwiftUI

struct ContentView: View {
    @State var accountBalance: Decimal = 1500.24
    
    var body: some View {
        VStack(alignment: .center) {
            Text(
                accountBalance,
                format: Decimal.FormatStyle.Currency(code: "USD")
                    .sign(strategy: .always())
                    .scale(4)
            )
        }
    }
}
```

![iOS Simulator showing a Text control formatting as explained above](/blog/img/swiftui-formatting/swiftui-formatting_2022-06-21-22-25-12.png)

There's a lot of other formatters available, so simply add `format` to your control
and try all of them out!
