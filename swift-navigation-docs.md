# pointfreeco/swift-navigation Documentation

Auto-generated from https://github.com/pointfreeco/swift-navigation
Generated on: Sun Jan  4 11:08:51 UTC 2026

## Documentation from Sources/AppKitNavigation/Documentation.docc

### AppKitNavigation

# ``AppKitNavigation``

Tools for making AppKit navigation simpler, more ergonomic and more precise.

## Additional Resources

- [GitHub Repo](https://github.com/pointfreeco/swift-navigation)
- [Discussions](https://github.com/pointfreeco/swift-navigation/discussions)
- [Point-Free Videos](https://www.pointfree.co/collections/ukit)

## Topics

### Animations

- ``withAppKitAnimation(_:_:completion:)``
- ``AppKitAnimation``
- ``SwiftNavigation/UIBinding``
- ``SwiftNavigation/UITransaction``

### Xcode previews

- ``NSViewRepresenting``
- ``NSViewControllerRepresenting``

---

### AppKitAnimation

# ``AppKitNavigation/AppKitAnimation``

## Topics

### Getting the default animation

- ``default``

### Getting linear animations

- ``linear``
- ``linear(duration:)``

### Getting eased animations

- ``easeIn``
- ``easeIn(duration:)``
- ``easeOut``
- ``easeOut(duration:)``
- ``easeInOut``
- ``easeInOut(duration:)``

### Creating custom animations

- ``animate(duration:timingFunction:)``
- ``init(_:)``

---

## Documentation from Sources/SwiftNavigation/Documentation.docc

### CrossPlatform

# Cross-platform

Learn the basics of how this library's tools can be used to build tools for cross-platform 
development.

## Overview

The tools provided by this library can also form the foundation of building navigation tools for
non-Apple platforms, such as Windows, Linux, Wasm and more. We do not currently provide any such
tools at this moment, but it is possible for them to be built externally.

For example, in Wasm it is possible to use the ``observe(isolation:_:)-9xf99`` function to observe
changes to a model and update the DOM:

```swift
import JavaScriptKit

var countLabel = document.createElement("span")
_ = document.body.appendChild(countLabel)

let token = observe {
  countLabel.innerText = .string("Count: \(model.count)")
}
```

And it's possible to drive navigation from state, such as an alert:

```swift
alert(isPresented: $model.isShowingErrorAlert) {
  "Something went wrong"
}
```

And you can build more advanced tools for presenting and dismissing `<dialog>`'s in the browser.

---

### WhatIsNavigation

# What is navigation?

Learn how one can think of navigation as a domain modeling problem, and how that leads to the
creation of concise and testable APIs for navigation.

## Overview

We will define navigation as a "mode" change in an application. The most prototypical example of
this is the drill-down. A user taps a button, and a right-to-left animation transitions you from the
current screen to the next screen.

> Important: Everything that follows is mostly focused on SwiftUI and UIKit navigation, but the 
ideas apply to other platforms too, such as Windows, Linux, Wasm, and more.

But there are many more examples of navigation beyond stacks and links. Modals can be thought of as
navigation, too. A sheet can slide from bottom-to-top and transition you from the current screen to
a new screen. A full-screen cover can further take over the entire screen. Or a popover can
partially take over the screen.

Alerts and confirmation dialogs can also be thought of navigation as they are also modals that take
full control over the interface and force you to make a selection.

It's even possible for you to define your own notions of navigation, such as menus, toast
notifications, and more.

## State-driven navigation

All of these seemingly disparate examples of navigation can be unified under a single API.
Presentation and dismissal can be described with an optional piece of state:

  * When the state changes from `nil` to non-`nil`, a screen can be presented, whether that be
    _via_ a drill-down, modal, _etc._
  * And when the state changes from non-`nil` to `nil`, a screen can be dismissed.

Driving navigation from state like this can be incredibly powerful:

  * It guarantees that your model will always be in sync with the visual representation of the UI.
    It shouldn't be possible for a piece of state to be non-`nil` and not have the corresponding
    view present.
  * It easily enables deep linking capabilities. If all forms of navigation in your application are
    driven off of state, then you can instantly open your application into any state imaginable by
    simply constructing a piece of data, handing it to SwiftUI, and letting it do its thing.
  * It also allows you to write unit tests for navigation logic without resorting to UI tests, which
    can be slow, flakey, and introduce instability into your test suite. If you write a unit test
    showing that when a user performs an action that a piece of state went from `nil` to non-`nil`,
    then you can be assured that the user would be navigated to the next screen.
  * And finally, it is a platform agnostic way to think about application design. Nothing discussed
    above is SwiftUI-specific. These ideas apply to all view paradigms (SwiftUI, UIKit, AppKit, 
    etc.), and all platforms (Windows, Linux, Wasm, etc.).

This is why state-driven navigation is so great, and SwiftUI does a pretty great job at providing
these tools. However, there are ways to improve SwiftUI's tools, _and_ its possible to bring
state-driven tools to other Apple frameworks such as UIKit and AppKit, and even to other non-Apple
_platforms_, such as Windows, Linux, Wasm, and more.

---

### AlertState

# ``SwiftNavigation/AlertState``

## Topics

### Creating alerts

- ``init(title:actions:message:)``

### Reading alert data

- ``id``
- ``title``
- ``message``
- ``buttons``

### Transforming alerts

- ``map(_:)``

---

### ButtonState

# ``SwiftNavigation/ButtonState``

## Topics

### Creating buttons

- ``init(role:action:label:)-65t48``
- ``init(role:action:label:)-8tmop``
- ``ButtonStateRole``
- ``ButtonStateAction``

### Composing buttons

- ``ButtonStateBuilder``

### Reading button data

- ``id``
- ``role-swift.property``
- ``action``
- ``label``

### Performing actions

- ``withAction(_:)-64bph``
- ``withAction(_:)-t2w``

### Transforming buttons

- ``SwiftUI/Button``
- ``SwiftUI/ButtonRole``
- ``map(_:)``

---

### ConfirmationDialogState

# ``SwiftNavigation/ConfirmationDialogState``

## Topics

### Creating dialogs

- ``init(title:actions:message:)``
- ``init(titleVisibility:title:actions:message:)``
- ``ConfirmationDialogStateTitleVisibility``

### Reading dialog data

- ``id``
- ``title``
- ``titleVisibility``
- ``message``
- ``buttons``

### Transforming dialogs

- ``map(_:)``
- ``SwiftUI/Visibility``

---

### NSObjectObserve

# ``ObjectiveC/NSObject/observe(_:)-94oxy``

## Topics

### Attaching data to observation

- ``ObjectiveC/NSObject/observe(_:)-294aq``

---

### Observe

# ``SwiftNavigation/observe(isolation:_:)-9xf99``

## Topics

### Attaching data to observation

- ``observe(isolation:_:)-34d7t``

---

### TextState

# ``SwiftNavigation/TextState``

## Topics

### Creating text state

- ``init(_:)``
- ``init(_:tableName:bundle:comment:)``
- ``init(verbatim:)``

### Text state transformations

- ``SwiftUI/Text``
- ``Swift/String``

---

### UIBindable

# ``SwiftNavigation/UIBindable``

## Topics

### Creating a bindable value

- ``init(_:fileID:filePath:line:column:)-6gjp6``
- ``init(_:fileID:filePath:line:column:)-49wmd``
- ``init(wrappedValue:fileID:filePath:line:column:)-3t841``
- ``init(wrappedValue:fileID:filePath:line:column:)-20n7u``
- ``init(projectedValue:)``

### Getting the value

- ``wrappedValue``
- ``projectedValue``
- ``subscript(dynamicMember:)``

---

### UIBinding

# ``SwiftNavigation/UIBinding``

## Topics

### Creating a binding

- ``init(wrappedValue:)``
- ``init(_:)-1p53m``
- ``init(_:)-3ww0m``
- ``init(_:)-9wed9``
- ``init(_:fileID:filePath:line:column:)``
- ``init(projectedValue:)``
- ``constant(_:)``

### Getting the value

- ``wrappedValue``
- ``projectedValue``
- ``subscript(dynamicMember:)-61aos``
- ``subscript(dynamicMember:)-95b8x``
- ``subscript(dynamicMember:)-xef5``
- ``subscript(dynamicMember:)-lmkz``
- ``subscript(dynamicMember:)-63hti``

### Managing changes

- ``id``
- ``transaction(_:)``
- ``transaction``
- ``UIBindingIdentifier``

---

### SwiftNavigation

# ``SwiftNavigation``

Bringing simple and powerful navigation tools to all Swift platforms, inspired by SwiftUI.

## Additional Resources

- [GitHub Repo](https://github.com/pointfreeco/swift-navigation)
- [Discussions](https://github.com/pointfreeco/swift-navigation/discussions)
- [Point-Free Videos](https://www.pointfree.co/)

## Overview

This library contains a suite of tools that form the foundation for building powerful state
management and navigation APIs for Apple platforms, such as SwiftUI, UIKit, and AppKit, as well as
for non-Apple platforms, such as Windows, Linux, Wasm, and more.

The SwiftNavigation library forms the foundation that more advanced tools can be built upon, such
as the UIKitNavigation and SwiftUINavigation libraries. There are two primary tools provided:

* ``observe(isolation:_:)-9xf99``: Minimally observe changes in a model.
* ``UIBinding``: Two-way binding for connecting navigation and UI components to an observable model.

In addition to these tools there are some supplementary concepts that allow you to build more 
powerful tools, such as ``UITransaction``, which associates animations and other data with state
changes, and ``UINavigationPath``, which is a type-erased stack of data that helps in describing
stack-based navigation.

All of these tools form the foundation for how one can build more powerful and robust tools for
SwiftUI, UIKit, AppKit, and even non-Apple platforms.

## Topics

### Essentials

- <doc:WhatIsNavigation>

### Observing changes to state

- ``observe(isolation:_:)-9xf99``
- ``ObjectiveC/NSObject/observe(_:)-94oxy``
- ``ObserveToken``

### Creating and sharing state

- ``UIBindable``
- ``UIBinding``

### Attaching data to mutations

- ``withUITransaction(_:_:)``
- ``withUITransaction(_:_:_:)``
- ``UITransaction``
- ``UITransactionKey``

### Stack-based navigation

- ``UINavigationPath``
- ``HashableObject``

### UI state

- ``TextState``
- ``AlertState``
- ``ConfirmationDialogState``
- ``ButtonState``

---

## Documentation from Sources/SwiftUINavigation/Documentation.docc

### AlertsDialogs

# Alerts and dialogs

Learn how to present alerts and confirmation dialogs in a concise and testable manner.

## Overview

The library comes with new tools for driving alerts and confirmation dialogs from optional and enum
state, and makes them more testable.

### Alerts

Suppose you have a feature for deleting something in your application and you want to show an alert
for the user to confirm the deletion. You can do this by holding onto an optional `AlertState` in
your model, as well as an enum that describes every action that can happen in the alert:


```swift
@Observable
class FeatureModel {
  var alert: AlertState<AlertAction>?
  enum AlertAction {
    case confirmDelete
  }

  // ...
}
```

Then, when you need to show an alert you can update the alert state with a title, message and
buttons:

```swift
func deleteButtonTapped() {
  self.alert = AlertState {
    TextState("Are you sure?")
  } actions: {
    ButtonState(role: .destructive, action: .confirmDelete) {
      TextState("Delete")
    }
    ButtonState(role: .cancel) {
      TextState("Nevermind")
    }
  } message: {
    TextState("Deleting this item cannot be undone.")
  }
}
```

The type `TextState` is closely related to `Text` from SwiftUI, but plays more nicely with
equatability. This makes it possible to write tests against these values.

> Tip: The `actions` closure is a result builder, which allows you to insert small bits of logic:
> ```swift
> } actions: {
>   if item.isLocked {
>     ButtonState(role: .destructive, action: .confirmDelete) {
>       TextState("Unlock and delete")
>     }
>   } else {
>     ButtonState(role: .destructive, action: .confirmDelete) {
>       TextState("Delete")
>     }
>   }
>   ButtonState(role: .cancel) {
>     TextState("Nevermind")
>   }
> }
> ```

Next you can provide an endpoint that will be called when the alert is interacted with:

```swift
func alertButtonTapped(_ action: AlertAction?) {
  switch action {
  case .confirmDelete:
    // NB: Perform deletion logic here
  case nil:
    // NB: Perform cancel button logic here
  }
}
```

Finally, you can use a new, overloaded `.alert` view modifier for showing the alert when this state
becomes non-`nil`:

```swift
struct ContentView: View {
  @ObservedObject var model: FeatureModel

  var body: some View {
    List {
      // ...
    }
    .alert($model.alert) { action in
      model.alertButtonTapped(action)
    }
  }
}
```

By having all of the alert's state in your feature's model, you instantly unlock the ability to test
it:

```swift
func testDelete() {
  let model = FeatureModel(/* ... */)

  model.deleteButtonTapped()
  XCTAssertEqual(model.alert?.title, TextState("Are you sure?"))

  model.alertButtonTapped(.confirmDelete)
  // NB: Assert that deletion actually occurred.
}
```

This works because all of the types for describing an alert are `Equatable`, including `AlertState`,
`TextState`, and even the buttons.

Sometimes it is not optimal to model the alert as an optional. In particular, if a feature can
navigate to multiple, mutually exclusive screens, then a "case-pathable" enum is more appropriate.

In such a case:

```swift
@Observable
class FeatureModel {
  var destination: Destination?

  @CasePathable
  enum Destination {
    case alert(AlertState<AlertAction>)
    // NB: Other destinations
  }

  enum AlertAction {
    case confirmDelete
  }

  // ...
}
```

With this kind of set up you can use an alternative `alert` view modifier that takes an additional
argument for specifying which case of the enum drives the presentation of the alert:

```swift
.alert($model.destination.alert) { action in
  model.alertButtonTapped(action)
}
```

Note that the `case` argument is specified via a concept known as "case paths", which are like
key paths except tuned specifically for enums and cases rather than structs and properties.

### Confirmation dialogs

The APIs for driving confirmation dialogs from optional and enum state look nearly identical to that
of alerts.

For example, the model for a delete confirmation could look like this:

```swift
@Observable
class FeatureModel {
  var dialog: ConfirmationDialogState<DialogAction>?
  enum DialogAction {
    case confirmDelete
  }

  func deleteButtonTapped() {
    dialog = ConfirmationDialogState(titleVisibility: .visible) {
      TextState("Are you sure?")
    } actions: {
      ButtonState(role: .destructive, action: .confirmDelete) {
        TextState("Delete")
      }
      ButtonState(role: .cancel) {
        TextState("Nevermind")
      }
    } message: {
      TextState("Deleting this item cannot be undone.")
    }
  }

  func dialogButtonTapped(_ action: DialogAction?) {
    switch action {
    case .confirmDelete:
      // NB: Perform deletion logic here
    case nil:
      // NB: Perform cancel button logic here
    }
  }
}
```

And then the view would look like this:

```swift
struct ContentView: View {
  @ObservedObject var model: FeatureModel

  var body: some View {
    List {
      // ...
    }
    .confirmationDialog($model.dialog) { action in
      dialogButtonTapped(action)
    }
  }
}
```

## Topics

### Alerts and dialogs

- ``SwiftUI/View/alert(item:title:actions:message:)``
- ``SwiftUI/View/alert(item:title:actions:)``
- ``SwiftUI/View/confirmationDialog(item:titleVisibility:title:actions:message:)``
- ``SwiftUI/View/confirmationDialog(item:titleVisibility:title:actions:)``

### Alert state and dialog state

- ``SwiftUI/View/alert(_:action:)-sgyk``
- ``SwiftUI/View/alert(_:action:)-1gtsa``
- ``SwiftUI/View/confirmationDialog(_:action:)-9alh7``
- ``SwiftUI/View/confirmationDialog(_:action:)-7mxx7``

---

### Bindings

# Bindings

Learn how to manage certain view state, such as `@FocusState` directly in your observable classes.

## Overview

SwiftUI comes with many property wrappers that can be used in views to drive view state, such as

`@FocusState`. Unfortunately, these property wrappers _must_ be used in views. It's not possible to
extract this logic to an `@Observable` class and integrate it with the rest of the model's business
logic, and be in a better position to test this state.

We can work around these limitations by introducing a published field to your observable object and
synchronizing it to view state with the `bind` view modifier that ships with this library.

For example, suppose you have a sign in flow where if the API request to sign in fails, you want
to refocus the email field. The model can be implemented like so:

```swift
@Observable
class SignInModel {
  var email: String
  var password: String
  var focus: Field?
  enum Field { case email, password }

  func signInButtonTapped() async throws {
    do {
      try await self.apiClient.signIn(self.email, self.password)
    } catch {
      self.focus = .email
    }
  }
}
```

Notice that we store the focus as a regular `var` property in the model rather than `@FocusState`.
This is because `@FocusState` only works when installed directly in a view. It cannot be used in
an observable class.

You can implement the view as you would normally, except you must also use `@FocusState` for the
focus _and_ use the `bind` helper to make sure that changes to the model's focus are replayed to
the view, and vice versa.

```swift
struct SignInView: View {
  @FocusState var focus: SignInModel.Field?
  @ObservedObject var model: SignInModel

  var body: some View {
    Form {
      TextField("Email", text: self.$model.email)
        .focused(self.$focus, equals: .email)
      TextField("Password", text: self.$model.password)
        .focused(self.$focus, equals: .password)
      Button("Sign in") {
        Task {
          await self.model.signInButtonTapped()
        }
      }
    }
    // ⬇️ Replays changes of `model.focus` to `focus` and vice-versa.
    .bind(self.$model.focus, to: self.$focus)
  }
}
```

## Topics

### Dynamic case lookup

- ``SwiftUI/Binding/subscript(dynamicMember:)-9abgy``
- ``SwiftUI/Binding/subscript(dynamicMember:)-4i40p``
- ``SwiftUI/Binding/subscript(dynamicMember:)-8vc80``

### Unwrapping bindings

- ``SwiftUI/Binding/init(unwrapping:)``

### Binding transformations

- ``SwiftUI/Binding/init(_:)``
- ``SwiftUI/Binding/removeDuplicates()``
- ``SwiftUI/Binding/removeDuplicates(by:)``

### Supporting views

- ``SwiftUI/View/bind(_:to:)``

---

### Navigation

# Navigation links and destinations

Learn how to drive navigation in `NavigationView` and `NavigationStack` in a concise and testable
manner.

## Overview

The library comes with new tools for driving drill-down navigation with optional and enum values.
This includes new initializers on `NavigationLink` and new overloads of the `navigationDestination`
view modifier.

Suppose your view or model holds a piece of optional state that represents whether or not a
drill-down should occur:

```swift
struct ContentView: View {
  @State var destination: Int?

  // ...
}
```

Further suppose that the screen being navigated to wants a binding to the integer when it is
non-`nil`. You can construct a `NavigationLink` that will activate when that state becomes
non-`nil`, and will deactivate when the state becomes `nil`:

```swift
NavigationLink(item: $destination) { isActive in
  destination = isActive ? 42 : nil
} destination: { $number in
  CounterView(number: $number)
} label: {
  Text("Go to counter")
}
```

The first trailing closure is the "action" of the navigation link. It is invoked with `true` when
the user taps on the link, and it is invoked with `false` when the user taps the back button or
swipes on the left edge of the screen. It is your job to hydrate the state in the action closure.

The second trailing closure, labeled `destination`, takes an argument that is the binding of the
unwrapped state. This binding can be handed to the child view, and any changes made by the parent
will be reflected in the child, and vice-versa.

For iOS 16+ you can use the `navigationDestination` overload:

```swift
Button {
  destination = 42
} label: {
  Text("Go to counter")
}
.navigationDestination(item: $model.destination) { $item in
  CounterView(number: $number)
}
```

Sometimes it is not optimal to model navigation destinations as optionals. In particular, if a
feature can navigate to multiple, mutually exclusive screens, then an enum is more appropriate.

Suppose that in addition to be able to drill down to a counter view that one can also open a
sheet with some text. We can model those destinations as an enum:

```swift
@CasePathable
enum Destination {
  case counter(Int)
  case text(String)
}
```

> Note: We have applied the `@CasePathable` macro from
> [CasePaths](https://github.com/pointfreeco.swift-case-paths), which allows the navigation binding
> to use "dynamic case lookup" to a particular enum case.

And we can hold an optional destination in state to represent whether or not we are navigated to
one of these destinations:

```swift
@State var destination: Destination?
```

With this set up you can make use of the
``SwiftUI/NavigationLink/init(item:onNavigate:destination:label:)`` initializer on
`NavigationLink` in order to specify a binding to the optional destination, and further specify
which case of the enum you want driving navigation:

```swift
NavigationLink(item: $destination.counter) { isActive in
  destination = isActive ? .counter(42) : nil
} destination: { $number in
  CounterView(number: $number)
} label: {
  Text("Go to counter")
}
```

And similarly for ``SwiftUI/View/navigationDestination(item:destination:)``:

```swift
Button {
  destination = .counter(42)
} label: {
  Text("Go to counter")
}
.navigationDestination(item: $model.destination.counter) { $number in
  CounterView(number: $number)
}
```

## Topics

### Navigation views and modifiers

- ``SwiftUI/View/navigationDestination(item:destination:)``
- ``SwiftUI/View/navigationDestination(item:destination:)``
- ``SwiftUI/NavigationLink/init(item:onNavigate:destination:label:)``

---

### SheetsPopoversCovers

# Sheets, popovers, and covers

Learn how to present sheets, popovers and covers in a concise and testable manner.

## Overview

The library comes with new tools for driving sheets, popovers and covers from optional and enum
state.

* [Sheets](#Sheets)
* [Popovers](#Popovers)
* [Covers](#Covers)

### Sheets

Suppose your view or model holds a piece of optional state that represents whether or not a modal
sheet is presented:

```swift
struct ContentView: View {
  @State var destination: Int?

  // ...
}
```

Further suppose that the screen being presented wants a binding to the integer when it is non-`nil`.
You can use the `sheet(item:)` overload that comes with the library:

```swift
var body: some View {
  List {
    // ...
  }
  .sheet(item: $destination) { $number in
    CounterView(number: $number)
  }
}
```

Notice that the trailing closure is handed a binding to the unwrapped state. This binding can be
handed to the child view, and any changes made by the parent will be reflected in the child, and
vice-versa.

However, this does not compile just yet because `sheet(item:)` requires that the item being 
presented conform to `Identifable`, and `Int` does not conform. This library comes with an overload
of `sheet`, called ``SwiftUI/View/sheet(item:id:onDismiss:content:)-1hi9l``, that allows you to 
specify the ID of the item being presented:

```swift
var body: some View {
  List {
    // ...
  }
  .sheet(item: $destination, id: \.self) { $number in
    CounterView(number: $number)
  }
}
```

Sometimes it is not optimal to model presentation destinations as optionals. In particular, if a
feature can navigate to multiple, mutually exclusive screens, then an enum is more appropriate.

There is an additional overload of the `sheet` for this situation. If you model your destinations
as a "case-pathable" enum:

```swift
@State var destination: Destination?

@CasePathable
enum Destination {
  case counter(Int)
  // More destinations
}
```

Then you can show a sheet from the `counter` case with the following:

```swift
var body: some View {
  List {
    // ...
  }
  .sheet(item: $destination.counter, id: \.self) { $number in
    CounterView(number: $number)
  }
}
```

### Popovers

Popovers work similarly to sheets. If the popover's state is represented as an optional you can do
the following:

```swift
struct ContentView: View {
  @State var destination: Int?

  var body: some View {
    List {
      // ...
    }
    .popover(item: $destination, id: \.self) { $number in
      CounterView(number: $number)
    }
  }
}
```

And if the popover state is represented as a "case-pathable" enum, then you can do the following:

```swift
struct ContentView: View {
  @State var destination: Destination?

  @CasePathable
  enum Destination {
    case counter(Int)
    // More destinations
  }

  var body: some View {
    List {
      // ...
    }
    .popover(item: $destination.counter, id: \.self) { $number in
      CounterView(number: $number)
    }
  }
}
```

### Covers

Full screen covers work similarly to sheets and popovers. If the cover's state is represented as an
optional you can do the following:

```swift
struct ContentView: View {
  @State var destination: Int?

  var body: some View {
    List {
      // ...
    }
    .fullscreenCover(item: $destination, id: \.self) { $number in
      CounterView(number: $number)
    }
  }
}
```

And if the covers' state is represented as a "case-pathable" enum, then you can do the following:

```swift
struct ContentView: View {
  @State var destination: Destination?

  @CasePathable
  enum Destination {
    case counter(Int)
    // More destinations
  }

  var body: some View {
    List {
      // ...
    }
    .fullscreenCover(item: $destination.counter, id: \.self) { $number in
      CounterView(number: $number)
    }
  }
}
```

## Topics

### Presentation modifiers

- ``SwiftUI/View/fullScreenCover(item:id:onDismiss:content:)-9csbq``
- ``SwiftUI/View/fullScreenCover(item:onDismiss:content:)``
- ``SwiftUI/View/fullScreenCover(item:id:onDismiss:content:)-14to1``
- ``SwiftUI/View/popover(item:id:attachmentAnchor:arrowEdge:content:)-3un96``
- ``SwiftUI/View/popover(item:attachmentAnchor:arrowEdge:content:)``
- ``SwiftUI/View/popover(item:id:attachmentAnchor:arrowEdge:content:)-57svy``
- ``SwiftUI/View/sheet(item:id:onDismiss:content:)-1hi9l``
- ``SwiftUI/View/sheet(item:onDismiss:content:)``
- ``SwiftUI/View/sheet(item:id:onDismiss:content:)-6tgux``

---

### SwiftUINavigationTools

# SwiftUI navigation

Learn more about SwiftUI's tools for navigations, and how this library improves upon them.

## SwiftUI's navigation tools

---

### SwiftUINavigation

# ``SwiftUINavigation``

Tools for making SwiftUI navigation simpler, more ergonomic and more precise.

## Additional Resources

- [GitHub Repo](https://github.com/pointfreeco/swift-navigation)
- [Discussions](https://github.com/pointfreeco/swift-navigation/discussions)
- [Point-Free Videos](https://www.pointfree.co/collections/swiftui/navigation)

## Overview

SwiftUI comes with many forms of navigation (tabs, alerts, dialogs, modal sheets, popovers,
navigation links, and more), and each comes with a few ways to construct them. These ways roughly
fall in two categories:

  * "Fire-and-forget": These are initializers and methods that do not take binding arguments, which
    means SwiftUI fully manages navigation state internally. This makes it is easy to get something
    on the screen quickly, but you also have no programmatic control over the navigation. Examples
    of this are the initializers on [`TabView`][TabView.init] and
    [`NavigationLink`][NavigationLink.init] that do not take a binding.

  * "State-driven": Most other initializers and methods do take a binding, which means you can
    mutate state in your domain to tell SwiftUI when it should activate or deactivate navigation.
    Using these APIs is more complicated than the "fire-and-forget" style, but doing so instantly
    gives you the ability to deep-link into any state of your application by just constructing a
    piece of data, handing it to a SwiftUI view, and letting SwiftUI handle the rest.

Navigation that is "state-driven" is the more powerful form of navigation, albeit slightly more
complicated. To wield it correctly you must be able to model your domain as concisely as possible,
and this usually means using enums.

Unfortunately, SwiftUI does not ship with all of the tools necessary to model our domains with
enums and make use of navigation APIs. This library bridges that gap by providing APIs that allow
you to model your navigation destinations as an enum, and then drive navigation by a binding
to that enum.

For example, suppose you have a feature that can present a sheet for creating an item, drill-down to
a view for editing an item, and can present an alert for confirming to delete an item. One can
technically model this with 3 separate optionals:

```swift
@Observable
class FeatureModel {
  var addItem: AddItemModel?
  var deleteItemAlertIsPresented: Bool
  var editItem: EditItemModel?
  // ...
}
```

And then in the view one can use the `sheet`, `navigationDestination` and `alert` view modifiers
to describe the type of navigation:

```swift
.sheet(item: $model.addItem) { addItemModel in
  AddItemView(model: addItemModel)
}
.alert("Delete?", isPresented: $model.deleteItemAlertIsPresented) {
  Button("Yes", role: .destructive) { /* ... */ }
  Button("No", role: .cancel) {}
}
.navigationDestination(item: $model.editItem) { editItemModel in
  EditItemModel(model: editItemModel)
}
```

This works great at first, but also introduces a lot of unnecessary complexity into your feature.
These 3 optionals means that there are technically 8 different states: All can be `nil`, one can
be non-`nil`, two could be non-`nil`, or all three could be non-`nil`. But only 4 of those states
are valid: either all are `nil` or exactly one is non-`nil`.

By allowing these 4 other invalid states we can accidentally tell SwiftUI to both present a sheet
and alert at the same time, but that is not a valid thing to do in SwiftUI. The framework will even
print a message to the console letting you know that in the future it may actually crash your app.

Luckily Swift comes with the perfect tool for dealing with this kind of situation: enums! They
allow you to concisely define a type that can be one of many cases. So, we can refactor our 3
optionals as an enum with 3 cases, and then hold onto a single piece of optional state:

```swift
@Observable
class FeatureModel {
  var destination: Destination?

  enum Destination {
    case addItem(AddItemModel)
    case deleteItemAlert
    case editItem(EditItemModel)
  }
}
```

This is more concise, and we get compile-time verification that at most one destination can be
active at a time. However, SwiftUI does not come with the tools to drive navigation from this model.
This is where the SwiftUINavigation tools becomes useful.

We start by annotating the `Destination` enum with the `@CasePathable` macro, which allows one to
refer to the cases of an enum with dot-syntax just like one does with structs and properties:

```diff
+@CasePathable
 enum Destination {
   // ...
 }
```

And now one can use simple dot-chaining syntax to derive a binding from a particular case of
the `destination` property:

```swift
.sheet(item: $model.destination.addItem) { addItemModel in
  AddItemView(model: addItemModel)
}
.alert("Delete?", isPresented: Binding($model.destination.deleteItemAlert)) {
  Button("Yes", role: .destructive) { /* ... */ }
  Button("No", role: .cancel) {}
}
.navigationDestination(item: $model.destination.editItem) { editItemModel in
  EditItemView(model: editItemModel)
}
```

> Note: For the alert we are using the special `Binding` initializer that turns a `Binding<Void?>`
> into a `Binding<Bool>`.

We now have a concise way of describing all of the destinations a feature can navigate to, and
we can still use SwiftUI's navigation APIs.

## Topics

### Tools

- <doc:SwiftUINavigationTools>
- <doc:Navigation>
- <doc:SheetsPopoversCovers>
- <doc:AlertsDialogs>
- <doc:Bindings>

## See Also

The collection of videos from [Point-Free](https://www.pointfree.co) that dive deep into the
development of the library.

* [Point-Free Videos](https://www.pointfree.co/collections/swiftui/navigation)

[NavigationLink.init]: https://developer.apple.com/documentation/swiftui/navigationlink/init(destination:label:)-27n7s
[TabView.init]: https://developer.apple.com/documentation/swiftui/tabview/init(content:)

---

## Documentation from Sources/UIKitNavigation/Documentation.docc

### UIColorWell

# ``UIKit/UIColorWell``

## Topics

### Binding to observable state

- ``UIKit/UIColorWell/init(frame:selectedColor:)``
- ``UIKit/UIColorWell/bind(selectedColor:)``

---

### UIControlProtocol

# ``UIKitNavigation/UIControlProtocol``

## Topics

### Binding to observable state

- ``bind(_:to:for:)``
- ``bind(_:to:for:set:)``

---

### UIDatePicker

# ``UIKit/UIDatePicker``

## Topics

### Binding to observable state

- ``UIKit/UIDatePicker/init(frame:date:)``
- ``UIKit/UIDatePicker/bind(date:)``

---

### UIKitAnimation

# ``UIKitNavigation/UIKitAnimation``

## Topics

### Getting the default animation

- ``default``

### Getting linear animations

- ``linear``
- ``linear(duration:)``

### Getting eased animations

- ``easeIn``
- ``easeIn(duration:)``
- ``easeOut``
- ``easeOut(duration:)``
- ``easeInOut``
- ``easeInOut(duration:)``

### Creating custom animations

- ``animate(withDuration:delay:options:)``
- ``animate(withDuration:delay:usingSpringWithDamping:initialSpringVelocity:options:)``
- ``animate(springDuration:bounce:initialSpringVelocity:delay:options:)``
- ``init(_:)``

### Configuring an animation

- ``delay(_:)``
- ``repeatCount(_:autoreverses:)``
- ``repeatForever(autoreverses:)``
- ``speed(_:)``

---

### UIPageControl

# ``UIKit/UIPageControl``

## Topics

### Binding to observable state

- ``UIKit/UIPageControl/init(frame:currentPage:)``
- ``UIKit/UIPageControl/bind(currentPage:)``

---

### UISlider

# ``UIKit/UISlider``

## Topics

### Binding to observable state

- ``UIKit/UISlider/init(frame:value:)``
- ``UIKit/UISlider/bind(value:)``

---

### UIStepper

# ``UIKit/UIStepper``

## Topics

### Binding to observable state

- ``UIKit/UIStepper/init(frame:value:)``
- ``UIKit/UIStepper/bind(value:)``

---

### UISwitch

# ``UIKit/UISwitch``

## Topics

### Binding to observable state

- ``UIKit/UISwitch/init(frame:isOn:)``
- ``UIKit/UISwitch/bind(isOn:)``

---

### UITextField

# ``UIKit/UITextField``

## Topics

### Binding to observable text

- ``UIKit/UITextField/init(frame:text:)``
- ``UIKit/UITextField/bind(text:)``

### Binding to attributed text

- ``UIKit/UITextField/init(frame:attributedText:)``
- ``UIKit/UITextField/bind(attributedText:)``

### Binding to focus state

- ``UIKit/UITextField/bind(focus:)``
- ``UIKit/UITextField/bind(focus:equals:)``

### Binding to text selection

- ``UIKit/UITextField/bind(selection:)``

---

### UIViewController

# ``UIKit/UIViewController``

## Topics

### Tree-based navigation

- ``UIKit/UIViewController/present(isPresented:onDismiss:content:)``
- ``UIKit/UIViewController/present(item:onDismiss:content:)-4m7m3``
- ``UIKit/UIViewController/present(item:onDismiss:content:)-4x5io``
- ``UIKit/UIViewController/present(item:id:onDismiss:content:)-4xafn``
- ``UIKit/UIViewController/present(item:id:onDismiss:content:)-9fu88``
- ``UIKit/UIViewController/navigationDestination(isPresented:content:)``
- ``UIKit/UIViewController/navigationDestination(item:content:)-1gks3``
- ``UIKit/UIViewController/navigationDestination(item:content:)-5auro``

### Custom tree-based navigation

- ``UIKit/UIViewController/destination(isPresented:content:present:dismiss:)``
- ``UIKit/UIViewController/destination(item:content:present:dismiss:)``
- ``UIKit/UIViewController/destination(item:id:content:present:dismiss:)``

### Stack-based navigation

- ``NavigationStackController``
- ``UIKit/UIViewController/navigationDestination(for:destination:)``
- ``UIKit/UIViewController/push(value:)``

---

### UIKitNavigation

# ``UIKitNavigation``

Tools for making SwiftUI navigation simpler, more ergonomic and more precise.

## Additional Resources

- [GitHub Repo](https://github.com/pointfreeco/swift-navigation)
- [Discussions](https://github.com/pointfreeco/swift-navigation/discussions)
- [Point-Free Videos](https://www.pointfree.co/collections/ukit)

## Overview

UIKit provides a few simple tools for navigation, but none of them are state-driven. Its navigation
tools are what is known as "fire-and-forget", which means you simply invoke a method to trigger
a navigation, but there is no representation of that event in your feature's state.


For example, to present a sheet from a button press one can simply do:

```swift
let button = UIButton(type: .system, primaryAction: UIAction { [weak self] _ in
  present(SettingsViewController(), animated: true)
})
```

This makes it easy to get started with navigation, but there are a few problems with this:

* It is difficult to determine from your feature's logic what child features are currently 
presented. You can check the `presentedViewController` property on `UIViewController` directly, 
but then that logic must exist in the view (and so hard to test), and you have to do extra work
to inspect the type-erased controller to truly see what is being presented.
* It is difficult to perform deep-linking to any feature of your app. You must duplicate the 
logic that invokes `present` or `pushViewController` in another part of your app in order to
deep-link into child features.

SwiftUI has taught us, it is incredibly powerful to be able to drive navigation from state. It 
allows you to encapsulate more of your feature's logic in an isolated and testable domain, and it 
unlocks deep linking for free since one just needs to construct a piece of state that represents 
where you want to navigate to, hand it to SwiftUI, and let SwiftUI do the rest.

The UIKitNavigation library brings a powerful suite of navigation tools to UIKit that are heavily
inspired by SwiftUI. For example, if you have a feature that can navigate to 3 different screens,
you can model that as an enum with 3 cases and some optional state: 

```swift
@Observable
class FeatureModel {
  var destination: Destination?

  enum Destination {
    case addItem(AddItemModel)
    case deleteItemAlert
    case editItem(EditItemModel)
  }
}
```

This allows us to prove that at most one of these destinations can be active at a time. And it
would be great if we could present and dismiss these child features based on the value of
`destination`. 

This is possible, but first we have to make one small change to the `Destination` enum by annotating
it with the `@CasePathable` macro:

```diff
+@CasePathable
 enum Destination {
   // ...
 }
```

This allows us to derive key paths and properties for each case of an enum, which is not currently
possible in native Swift.

With that done one can drive navigation in a _view controller_ using tools in the library: 

```swift
class FeatureViewController: UIViewController {
  @UIBindable var model: FeatureModel

  func viewDidLoad() {
    super.viewDidLoad()

    // Set up view hierarchy

    present(item: $model.destination.addItem) { addItemModel in
      AddItemViewController(model: addItemModel)
    }
    present(isPresented: Binding($model.destination.deleteItemAlert)) {
      let alert = UIAlertController(title: "Delete?", message: message, preferredStyle: .alert)
      alert.addAction(UIAlertAction(title: "Yes", style: .destructive))
      alert.addAction(UIAlertAction(title: "No", style: .cancel))
      return alert
    }
    navigationDestination(item: $model.destination.editItem) { editItemModel in
      EditItemViewController(model: editItemModel)
    }
  }
}
```

By using the libraries navigation tools we can be guaranteed that the model will be kept in sync
with the view. When the state becomes non-`nil` the corresponding form of navigation will be 
triggered, and when the presented view is dismissed, the state will be `nil`'d out.

Another powerful aspect of SwiftUI is its ability to update its UI whenever state in an observable
model changes. And thanks to Swift's observation tools this can be done done implicitly and 
minimally: whichever fields are accessed in the `body` of the view are automatically tracked 
so that when they change the view updates.

Our UIKitNavigation library comes with a tool that brings this power to UIKit, and it's called
``observe(isolation:_:)-9xf99``:

```swift
observe { [weak self] in
  guard let self else { return }
  
  countLabel.text = "Count: \(model.count)"
  factLabel.isHidden = model.fact == nil 
  if let fact = model.fact {
    factLabel.text = fact
  }
  activityIndicator.isHidden = !model.isLoadingFact
}
```

Whichever fields are accessed inside `observe` (such as `count`, `fact` and `isLoadingFact` above)
are automatically tracked, so that whenever they are mutated the trailing closure of `observe`
will be invoked again, allowing us to update the UI with the freshest data.

All of these tools are built on top of Swift's powerful Observation framework. However, that 
framework only works on newer versions of Apple's platforms: iOS 17+, macOS 14+, tvOS 17+ and
watchOS 10+. However, thanks to our back-port of Swift's observation tools (see 
[Perception](http://github.com/pointfreeco/swift-perception)), you can make use of our tools 
right away, going all the way back to the iOS 13 era of platforms.


## Topics

### Animations

- ``withUIKitAnimation(_:_:completion:)``
- ``UIKitAnimation``

### Controls

- ``UIControlProtocol``
- ``UIKit/UIColorWell``
- ``UIKit/UIDatePicker``
- ``UIKit/UIPageControl``
- ``UIKit/UISegmentedControl``
- ``UIKit/UISlider``
- ``UIKit/UIStepper``
- ``UIKit/UISwitch``
- ``UIKit/UITextField``

### Navigation

- ``UIKit/UIViewController``
- ``UIKit/UIAlertController``
- ``UIKit/UITraitCollection``

### Xcode previews

- ``UIViewRepresenting``
- ``UIViewControllerRepresenting``

---

