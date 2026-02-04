# pointfreeco/swift-dependencies Documentation

Auto-generated from https://github.com/pointfreeco/swift-dependencies
Generated on: Wed Feb  4 06:19:15 UTC 2026

## Documentation from Sources/Dependencies/Documentation.docc

### DesigningDependencies

# Designing dependencies

Learn techniques on designing your dependencies so that they are most flexible for injecting into
features and overriding for tests.

* [Protocol-based dependencies](#Protocol-based-dependencies)
* [Struct-based dependencies](#Struct-based-dependencies)
* [@DependencyClient macro](#DependencyClient-macro)

## Overview

Making it possible to control your dependencies is the most important step you can take towards
making your features isolatable and testable. The second most important step after that is to design
your dependencies in a way that maximizes their flexibility in tests and other situations.

> Tip: We have an [entire series of episodes][designing-deps] dedicated to the topic of dependencies
> and how to best design and construct them.

## Protocol-based dependencies

The most popular way to design dependencies in Swift is to use protocols. For example, if your
feature needs to interact with an audio player, you might design a protocol with methods for
playing, stopping, and more:

```swift
protocol AudioPlayer {
  func loop(url: URL) async throws
  func play(url: URL) async throws
  func setVolume(_ volume: Float) async
  func stop() async
}
```

Then you are free to make as many conformances of this protocol as you want, such as a
`LiveAudioPlayer` that actually interacts with AVFoundation, or a `MockAudioPlayer` that doesn't
play any sounds, but does suspend in order to simulate that something is playing. You could even
have an `UnimplementedAudioPlayer` conformance that invokes `reportIssue` when any method is 
invoked:

```swift
struct LiveAudioPlayer: AudioPlayer {
  let audioEngine: AVAudioEngine
  // ...
}
struct MockAudioPlayer: AudioPlayer {
  // ...
}
struct UnimplementedAudioPlayer: AudioPlayer {
  func loop(url: URL) async throws {
    reportIssue("AudioPlayer.loop is unimplemented")
  }
  // ...
}
```

And all of those conformances can be used to specify the live, preview and test values for the
dependency:

```swift
private enum AudioPlayerKey: DependencyKey {
  static let liveValue: any AudioPlayer = LiveAudioPlayer()
  static let previewValue: any AudioPlayer = MockAudioPlayer()
  static let testValue: any AudioPlayer = UnimplementedAudioPlayer()
}
```

> Tip: See <doc:LivePreviewTest> for more information on how to best leverage live, preview and test
> implementations of your dependencies.

This style of dependencies works just fine, and if it is what you are most comfortable with then
there is no need to change.

## Struct-based dependencies

However, there is a small change one can make to this dependency to unlock even more power. Rather
than designing the audio player as a protocol, we can use a struct with closure properties to
represent the interface:

```swift
struct AudioPlayerClient {
  var loop: (_ url: URL) async throws -> Void
  var play: (_ url: URL) async throws -> Void
  var setVolume: (_ volume: Float) async -> Void
  var stop: () async -> Void
}
```

Then, rather than defining types that conform to the protocol you construct values:

```swift
extension AudioPlayerClient {
  static var live: Self {
    let audioEngine: AVAudioEngine
    return Self(/*...*/)
  }

  static let mock = Self(/* ... */)

  static let unimplemented = Self(
    loop: { _ in reportIssue("AudioPlayerClient.loop is unimplemented") },
    // ...
  )
}
```

Then, to register this dependency you can leverage the `AudioPlayerClient` struct to conform
to the ``DependencyKey`` protocol. There's no need to define a new type. In fact, you can even 
define the live, preview and test values directly in the conformance, all at once:

```swift
extension AudioPlayerClient: DependencyKey {
  static var liveValue: Self {
    let audioEngine: AVAudioEngine
    return Self(/* ... */)
  }

  static let previewValue = Self(/* ... */)

  static let testValue = Self(
    loop: unimplemented("AudioPlayerClient.loop"),
    play: unimplemented("AudioPlayerClient.play"),
    setVolume: unimplemented("AudioPlayerClient.setVolume"),
    stop: unimplemented("AudioPlayerClient.stop")
  )
}

extension DependencyValues {
  var audioPlayer: AudioPlayerClient {
    get { self[AudioPlayerClient.self] }
    set { self[AudioPlayerClient.self] = newValue }
  }
}
```

> Tip: We are using the `unimplemented` method from our 
> [Issue Reporting][issue-reporting-gh] library to provide closures that cause an
> XCTest failure if they are ever invoked. See <doc:LivePreviewTest> for more information on this
> pattern.

If you design your dependencies in this way you can pick which dependency endpoints you need in your
feature. For example, if you have a feature that needs an audio player to do its job, but it only
needs the `play` endpoint, and doesn't need to loop, set volume or stop audio, then you can specify
a dependency on just that one function:

```swift
@Observable
final class FeatureModel {
  @ObservationIgnored
  @Dependency(\.audioPlayer.play) var play
  // ...
}
```

This can allow your features to better describe the minimal interface they need from dependencies,
which can help a feature seem less intimidating.

You can also override the bare minimum of the dependency in tests. For example, suppose that one
user flow of your feature you are testing invokes the `play` endpoint, but you don't think any other
endpoint will be called. Then you can write a test that overrides only that one single endpoint:

```swift
func testFeature() {
  let isPlaying = ActorIsolated(false)

  let model = withDependencies {
    $0.audioPlayer.play = { _ in await isPlaying.setValue(true) }
  } operation: {
    FeatureModel()
  }

  await model.play()
  XCTAssertEqual(isPlaying.value, true)
}
```

If this test passes you can be guaranteed that no other endpoints of the dependency are used in the
user flow you are testing. If someday in the future more of the dependency is used, you will
instantly get a test failure, letting you know that there is more behavior that you must assert on.

## @DependencyClient macro

The library ships with a macro that can help improve the ergonomics of struct-based dependency
interfaces. The macro ships as a separate library within this package because it depends on 
SwiftSyntax, and that increases the build times by about 20 seconds. We did not want to force
everyone using this library to incur that cost, so if you want to use the macro you will need to
explicitly add the `DependenciesMacros` product to your targets.

Once that is done you can apply the `@DependencyClient` macro directly to your dependency struct:

```swift
import DependenciesMacros

@DependencyClient
struct AudioPlayerClient {
  var loop: (_ url: URL) async throws -> Void
  var play: (_ url: URL) async throws -> Void
  var setVolume: (_ volume: Float) async -> Void
  var stop: () async -> Void
}
```

This does a few things for you. First, it automatically provides a default for each endpoint that
simply throws an error and triggers an XCTest failure. This means you get an "unimplemented" client
for free with no additional work. This allows you to simplify the `testValue` of your 
``TestDependencyKey`` conformance like so:

```diff
 extension AudioPlayerClient: TestDependencyKey {
-  static let testValue = Self(
-    loop: unimplemented("AudioPlayerClient.loop"),
-    play: unimplemented("AudioPlayerClient.play"),
-    setVolume: unimplemented("AudioPlayerClient.setVolume"),
-    stop: unimplemented("AudioPlayerClient.stop")
-  )
+  static let testValue = Self()
 }
```

This behaves the exact same as before, but now all of the code is generated for you.

Further, when you provide argument labels to the client's closure endpoints, the macro turns that 
information into methods with argument labels. This means you can invoke the `play` endpoint
like so:

```swift
try await player.play(url: URL(filePath: "..."))
```

And finally, the macro also generates a public initializer for you with all of the client's 
endpoints. One typically needs to maintain this initializer when separate the interface of the 
dependency from the implementation (see 
<doc:LivePreviewTest#Separating-interface-and-implementation> for more information). But now there
is no need to maintain that code as it is automatically provided for you by the macro.

[designing-deps]: https://www.pointfree.co/collections/dependencies
[issue-reporting-gh]: http://github.com/pointfreeco/swift-issue-reporting

---

### Lifetimes

# Dependency lifetimes

Learn about the lifetimes of dependencies, how to prolong the lifetime of a dependency, and how
dependencies are inherited.

## Overview

When the ``Dependency`` property wrapper is initialized it captures the current state of the
dependency at that moment. This provides a kind of "scoping" mechanism that is similar to how
`@TaskLocal` values are inherited by new asynchronous tasks, but has some new caveats of its own.

* [How task locals work](#How-task-locals-work)
* [How @Dependency lifetimes work](#How-Dependency-lifetimes-work)
* [Accessing a @Dependency from pre-structured concurrency](#Accessing-a-Dependency-from-pre-structured-concurrency)

## How task locals work

Task locals are what power this library under the hood, and so it can be important to first
understand how task locals work and how task local inheritance works.

Task locals are values that are implicitly associated with a task. They make it possible to push
values deep into every part of an application without having to explicitly pass the values around.
This makes task locals sound like a "global" variable, which you may have heard is bad, but task
locals have 3 features that make them safe to use and easy to reason about:

  * Task locals are safe to use from concurrent contexts. This means multiple tasks can access the
    same task local without fear of a race condition.
  * Task locals can be mutated only in specific, well-defined scopes. It is not allowed to forever
    mutate a task local in a way that all parts of the application observe the change.
  * Task locals are inherited by new tasks that are spun up from existing tasks.

For example, suppose you had the following task local:

```swift
enum Locals {
  @TaskLocal static var value = 1
}
```

The value can only be "mutated" by using the task locals [`withValue`][tasklocal-withvalue-docs]
method, which allows changing `value` only for the scope of a non-escaping closure:

```swift
print(Locals.value)  // 1
Locals.$value.withValue(42) {
  print(Locals.value)  // 42
}
print(Locals.value)  // 1
```

The above shows that `Locals.value` is changed only for the duration of the `withValue` closure.

This may seem very restrictive, but it is also what makes task locals safe and easy to reason about.
You are not allowed to make task local changes to extend for any amount of time, such as mutating it
directly:

```swift
Locals.value = 42
// 🛑 Cannot assign to property: 'value' is a get-only property
```

If this were possible it would make changes to `value` instantly observable from every part of the
application. It could even cause two consecutive reads of `Locals.value` to report different values:

```swift
print(Locals.value)  // 1
print(Locals.value)  // 42
```

This would make code very difficult to reason about, and so is why task locals can be changed for
only very specific scopes.

However, there is a tool that Swift provides that allows task locals to prolong their changes
outside the scope of a non-escaping closure, and does so in a way without making it difficult to
reason about. That tool is known as "task local inheritance." Any child tasks created via
`TaskGroup` or `async let`, as well as tasks created with `Task { }`, inherit the task locals at the
moment they were created.

For example, the following example shows that a task local remains overridden even when accessed
from a `Task` a second later, and even though that closure is escaping:

```swift
enum Locals {
  @TaskLocal static var value = 1
}

print(Locals.value)  // 1
Locals.$value.withValue(42) {
  print(Locals.value)  // 42
  Task {
    try await Task.sleep(for: .seconds(1))
    print(Locals.value)  // 42
  }
  print(Locals.value)  // 42
}
```

Even though the closure handed to `Task` is escaping, and even though the print happens long after
`withValue`'s scope has ended, somehow still "42" is printed. This happens because task locals are
inherited in tasks.

This gives us the ability to prolong the lifetime of a task local change, but in a well-defined and
easy to reason about way.

It is important to note that task locals are not inherited in _all_ escaping contexts. It does work
for [`Task.init`][task-init-docs] and [`TaskGroup.addTask`][group-add-task-docs], which make use of
escaping closures, but only because the standard library special cases those tools to inherit task
locals (see `copyTaskLocals` in [this][task-copy-locals-code] code).

But generally speaking, task local overrides are lost when crossing escaping boundaries. For
example, if instead of using `Task` we used `DispatchQueue.main.asyncAfter` in the above code, we
will observe that the task local resets back to 1 in the escaped closure:

```swift
print(Locals.value)  // 1
Locals.$value.withValue(42) {
  print(Locals.value)  // 42
  DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    print(Locals.value)  // 1
  }
  print(Locals.value)  // 42
}
```

So, in conclusion, Swift does extra work to propagate task locals to certain escaping, unstructured
contexts, but does not do so universally, and so care must be taken.

## How @Dependency lifetimes work

Now that we understand how task locals work, we can begin to understand how `@Dependency` lifetimes
work, and how they can be extended. Under the hood, dependencies are held as a `@TaskLocal`, and so
many of the rules from task locals also apply to dependencies, _e.g._ dependencies are inherited in
tasks but not generally across escaping boundaries. But there are a few additional caveats.

Just like with task locals, a dependency's value can be changed for the scope of the trailing,
non-escaping closure of ``withDependencies(_:operation:)-4uz6m``, but the library also ships
with a few tools to prolong the change in a well-defined manner.

For example, suppose you have a feature that needs access to an API client for fetching a user:

```swift
@Observable
class FeatureModel {
  var user: User?

  @ObservationIgnored
  @Dependency(\.apiClient) var apiClient

  func onAppear() async {
    do {
      user = try await apiClient.fetchUser()
    } catch {}
  }
}
```

Sometimes we may want to construct this model in a "controlled" environment, where we use a
different implementation of `apiClient`. 

> Note: Tests are probably the most prototypical example of overriding dependencies to control them.
Be sure to read the dedicated article <doc:Testing> for more information on that topic.

Controlling dependencies isn't only useful in tests. It can also be used directly in your feature's
logic in order to run some child feature in a controlled environment, and can even be used in Xcode
previews.

Let's first see how controlling dependencies can be used directly in a feature's logic. Suppose we
wanted to show this feature in the application as a part of an "onboarding" experience. During the
onboarding experience, we want the user to be able to make use of the feature without executing real
life API requests, which may cause data to be written to a remote database.

Accomplishing this can be difficult because models are created in one scope and then dependencies
are used in another scope. However, as mentioned above, the library does extra work to make it so
that later referencing dependencies of a model uses the dependencies captured at the moment of
creating the model.

For example, if you create the features model in the following way:

```swift
let onboardingModel = withDependencies {
  $0.apiClient = .mock
} operation: {
  FeatureModel()
}
```

...then all references to the `apiClient` dependency inside `FeatureModel` will be using the mock
API client. This is true even though the `FeatureModel`'s `onAppear` method will be called outside
the scope of the `operation` closure.

However, care must be taken when creating a child model from a parent model. In order for the
child's dependencies to inherit from the parent's dependencies, you must make use of
``withDependencies(from:operation:fileID:filePath:line:column:)`` when creating the child model:

```swift
let onboardingModel = withDependencies(from: self) {
  $0.apiClient = .mock
} operation: {
  FeatureModel()
}
```

This makes `FeatureModel`'s dependencies inherit from the parent feature, and you can further
override any additional dependencies you want.

In general, if you want dependencies to be properly inherited through every layer of feature in your
application, you should make sure to create any observable models inside a
``withDependencies(from:operation:fileID:filePath:line:column:)`` scope.

If you do this, it also allows you to run previews in a very specific environment. Dependencies
already support the concept of a ``TestDependencyKey/previewValue-8u2sy``, which is an
implementation of the dependency used when run in an Xcode preview (see <doc:LivePreviewTest> for
more info). It is most appropriate to implement the ``TestDependencyKey/previewValue-8u2sy`` by
immediately returning some basic, mock data.

But sometimes you want to customize dependencies for the preview so that you can see how your
feature behaves in very specific states. For example, if you wanted to see how your feature reacts
when the `fetchUser` endpoint throws an error, you can update the preview like so:

```swift
#Preview {
  let _ = prepareDependencies {
    $0.apiClient.fetchUser = { _ in throw SomeError() }
  }
  FeatureView(model: FeatureModel())
}
```

## Accessing a @Dependency from pre-structured concurrency

Because dependencies are held in a task local, they only automatically propagate within structured
concurrency and in `Task`s. In order to access dependencies across escaping closures, _e.g._ in a
callback or Combine operator, you must do additional work to "escape" the dependencies so that they
can be passed into the closure.

For example, suppose you use `DispatchQueue.main.asyncAfter` to execute some logic after a delay,
and that logic needs to make use of dependencies. In order to guarantee that dependencies used in
the escaping closure of `asyncAfter` reflect the correct values, you must use
``withEscapedDependencies(_:)-5xvi3``:

```swift
withEscapedDependencies { dependencies in
  DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
    dependencies.yield {
      // All code in here will use dependencies at the time of calling withEscapedDependencies.
    }
  }
}
```

[task-copy-locals-code]: https://github.com/apple/swift/blob/60952b868d46fc9a83619f747a7f92b5534fb632/stdlib/public/Concurrency/Task.swift#L500-L509
[task-init-docs]: https://developer.apple.com/documentation/swift/task/init(priority:operation:)-5k89c
[group-add-task-docs]: https://developer.apple.com/documentation/swift/taskgroup/addtask(priority:operation:)
[tasklocal-withvalue-docs]: https://developer.apple.com/documentation/swift/tasklocal/withvalue(_:operation:file:line:)-1xjor

---

### LivePreviewTest

# Live, preview, and test dependencies

Learn how to provide different implementations of your dependencies for use in the live application,
as well as in Xcode previews, and even in tests.

## Overview

In the previous section we showed that to conform to ``DependencyKey`` you must provide _at least_ a
``DependencyKey/liveValue``, which is the default version of the dependency that is used when
running on a device or simulator. The ``DependencyKey`` protocol inherits from a base protocol,
``TestDependencyKey``, which has two other optional properties that can be implemented 
``TestDependencyKey/testValue`` and ``TestDependencyKey/previewValue-8u2sy``, both of which will 
delegate to ``DependencyKey/liveValue`` if left unimplemented.

Leveraging these alternative dependency implementations allow to run your features in safer
environments for tests, previews, and more.

* [Live value](#Live-value)
* [Test value](#Test-value)
* [Preview value](#Preview-value)
* [Separating interface and implementation](#Separating-interface-and-implementation)
* [Cascading rules](#Cascading-rules)

## Live value

The ``DependencyKey/liveValue`` static property from the ``DependencyKey`` protocol is the only
truly _required_ requirement from the protocol. This is the value that is used when running your
feature in the simulator or on a device. It is appropriate to use an implementation of your
dependency for this value that actually interacts with the outside world. That is, it can make
network requests, perform time-based asynchrony, interact with the file system, and more.

However, if you only implement ``DependencyKey/liveValue``, then it means your feature will use the
live dependency when run in tests, which can be problematic. That will cause live API requests to be 
made, which are slow and flakey, analytics will be tracked, which will muddy your data, files will
be written to disk, which will bleed into other tests, and more.

Using live dependencies in tests are so problematic that the library will cause a test failure
if you ever interact with a live dependency while tests are running:

```swift
@Test
func feature() async throws {
  let model = FeatureModel()

  model.addButtonTapped()
  // 🛑  A dependency has no test implementation, but was accessed from a 
  //     test context:
  //
  //         Dependency:
  //           APIClient
  //
  //     Dependencies registered with the library are not allowed to use 
  //     their default, live implementations when run from tests.
}
```

If you truly want to use live dependencies in tests you have to make it explicit by overriding the
dependency and setting the live value:

```swift
@Test
func feature() async throws {
  let model = withDependencies {
    // ⚠️ Explicitly say you want to use a live dependency.
    $0.apiClient = .liveValue
  } operation: {
    FeatureModel()
  }

  // ...
}
```

## Test value

The ``TestDependencyKey/testValue`` static property from the ``TestDependencyKey`` protocol should
be implemented if you want to provide a specific implementation of your dependency for all tests. At
a bare minimum you should provide an implementation of your dependency that does not reach out to
the real world. This means it should not make network requests, should not sleep for real-world
time, should not touch the file system, etc.

This can guarantee that a whole class of bugs do not happen in your code when running tests. For
example, suppose you have a dependency for tracking user events with your analytics server. If you
allow this dependency to be used in an uncontrolled manner in tests you run the risk of accidentally
tracking events that do not actually correspond to user actions, and therefore will result in bad,
unreliable data.

Another example of a dependency you want to control during tests is access to the file system. If
your feature writes a file to disk during a test, then that file will remain there for subsequent
runs of other tests. This causes testing artifacts to bleed over into other tests, which can cause
confusing failures.

So, providing a ``TestDependencyKey/testValue`` can be very useful, but even better, we highly
encourage users of our library to provide what is known as "unimplemented" versions of their
dependencies for their ``TestDependencyKey/testValue``. These are implementations that cause a test
failure if any of its endpoints are invoked.

You can use our [Issue Reporting][issue-reporting-gh] library to aid in this, which is
immediately accessible as a transitive dependency. It comes with a function called
[`unimplemented`][unimplemented-docs] that can return a function of nearly any signature with the
property that if it is invoked it will cause a test failure. For example, the hypothetical analytics
dependency we considered a moment ago can be given such a `testValue` like so:

```swift
struct AnalyticsClient {
  var track: (String, [String: String]) async throws -> Void
}

import Dependencies

extension AnalyticsClient: TestDependencyKey {
  static let testValue = Self(
    track: unimplemented("AnalyticsClient.track")
  )
}
```

This makes it so that if your feature ever makes use of the `track` endpoint on the analytics client
without you specifically overriding it, you will get a test failure. This makes it easy to be
notified if you ever start tracking new events without writing a test for it, which can be
incredibly powerful.

## Preview value

We've now seen that ``DependencyKey/liveValue`` is an appropriate place to put dependency
implementations that reach out to the outside world, and ``TestDependencyKey/testValue`` is an
appropriate place to put dependency implementations that refrain from interacting with the outside
world. Even better if the `testValue` actually causes a test failure if any of its endpoints are
accessed.

There's a third kind of implementation that you can provide that sits somewhere between
``DependencyKey/liveValue`` and ``TestDependencyKey/testValue``: it's called
``TestDependencyKey/previewValue-8u2sy``. It will be used whenever your feature is run in an Xcode
preview.

Xcode previews are similar to tests in that you usually do not want to interact with the outside 
world, such as making network requests. In fact, many of Apple's frameworks do not work in previews, 
such as Core Location, and so it will be hard to interact with your feature in previews if it 
touches those frameworks.

However, Xcode previews are dissimilar to tests in that it's fine for dependencies to return some 
mock data. There's no need to deal with "unimplemented" clients for proving which dependencies are
actually used.

For example, suppose you have an API client with some endpoints for fetching users. You do not want
to make live, network requests in Swift previews because that will cause previews to run slowly. So,
you can provide a ``TestDependencyKey/previewValue-8u2sy`` implementation that synchronously and
immediately returns some mock data:

```swift
extension APIClient: TestDependencyKey {
  static let previewValue = Self(
    fetchUsers: {
      [
        User(id: 1, name: "Blob"),
        User(id: 2, name: "Blob Jr."),
        User(id: 3, name: "Blob Sr."),
      ]
    },
    fetchUser: { id in
      User(id: id, name: "Blob, id: \(id)")
    }
  )
}
```

> Note: The `previewValue` implementation must be defined in the same module as the 
``TestDependencyKey`` conformance. If you end up separating the interface and implementation of your
dependency, as shown in <doc:LivePreviewTest#Separating-interface-and-implementation>, then it
must be defined the interface module, not the implementation module.

Then when running a feature that uses this dependency in an Xcode preview, it will immediately get
data provided to it, making it easier for you to iterate on your feature's logic and styling.

You can also always override dependencies for the preview if you want to test out a specific 
configuration of data. For example, if you want to test the empty state of your feature when the 
API client returns an empty array, you can do so like this:

```swift
struct Feature_Previews: PreviewProvider {
  static var previews: some View {
    FeatureView(
      model: withDependencies {
        $0.apiClient.fetchUsers = { _ in [] }
      } operation: {
        FeatureModel()
      }
    )
  }
}
```

Or if you want to preview how your feature deals with errors returned from the API:

```swift
struct Feature_Previews: PreviewProvider {
  static var previews: some View {
    FeatureView(
      model: withDependencies {
        $0.apiClient.fetchUser = { _ in
          struct SomeError: Error {}
          throw SomeError()
        }
      } operation: {
        FeatureModel()
      }
    )
  }
}
```

## Separating interface and implementation

It is common for the interface of an dependency to be super lightweight and compile quickly (as
usually it consists of some simple data types), but for the "live" implementation to be heavyweight
and take a long time to compile (usually when 3rd party libraries are involved). In such cases it is
recommended to put the interface and live implementation in separate modules, and then
implementation can depend on the interface.

In order to accomplish this you can conform your dependency to the ``TestDependencyKey`` protocol in
the interface module, like this:

```swift
// Module: AnalyticsClient
struct AnalyticsClient: TestDependencyKey {
  // ...

  static let testValue = Self(/* ... */)
}
```

And then in the implementation module you can extend the dependency to further conform to the
``DependencyKey`` protocol and provide a live implementation:

```swift
// Module: LiveAnalyticsClient
extension AnalyticsClient: DependencyKey {
  static let liveValue = Self(/* ... */)
}
```

## Cascading rules

Depending on which of ``TestDependencyKey/testValue``, ``TestDependencyKey/previewValue-8u2sy`` and
``DependencyKey/liveValue`` you implement, _and_ depending on which conformance to
``TestDependencyKey`` and ``DependencyKey`` is visible to the compiler, there are rules that decide
which actual dependency will be used at runtime.

  * A default implementation of ``TestDependencyKey/testValue`` is provided, and it simply calls out
    to ``TestDependencyKey/previewValue-8u2sy``. This means that in a testing context, the preview
    version of the dependency will be used.

  * Further, if a conformance to ``DependencyKey`` is provided in addition to ``TestDependencyKey``,
    then ``TestDependencyKey/previewValue-8u2sy`` has a default implementation provided, and it
    calls out to ``DependencyKey/liveValue``. This means that in a preview context, the live version
    of the dependency will be used.

Note that a consequence of the above two rules is that if only ``DependencyKey/liveValue`` is
implemented when conforming to ``DependencyKey``, then both ``TestDependencyKey/testValue`` and
``TestDependencyKey/previewValue-8u2sy`` will call out to the `liveValue` under the hood. This means
your dependency will be interacting with the outside world during tests and in previews, which may
not be ideal.

There is one thing the library will do to help you catch using a live dependency in tests. If a live
dependency is used in a test context, the test case will fail. This is done to make sure you
understand the risks of using a live dependency in tests. To confirm that you truly want to use a
live dependency you can override the dependency with `.liveValue`:

```swift
@Test
func feature() async throws {
  let model = withDependencies {
    // ⚠️ Explicitly say you want to use a live dependency.
    $0.apiClient = .liveValue
  } operation: {
    FeatureModel()
  }

  // ...
}
```

This will prevent the library from failing your test for using a live dependency in a testing
context.

On the flip side, the library also helps you catch when you have not provided a `liveValue`. When
running the application in the simulator or on a device, if a dependency is accessed for which a
`liveValue` has not been provided, a purple, runtime warning will appear in Xcode letting you know.

There is also a way to force a dependency context in an application target or test target. When
the environment variable `SWIFT_DEPENDENCIES_CONTEXT` is present, and is equal to either `live`,
`preview` or `test`, that context will be used. This can be useful in UI tests since the application
target runs as a separate process outside of the testing process.

In order to force the application target to run with test dependencies during a UI test, simply
perform the following in your UI test case:

```swift
func testFeature() {
  self.app.launchEnvironment["SWIFT_DEPENDENCIES_CONTEXT"] = "test"
  self.app.launch()
  …
}
```

[unimplemented-docs]: https://pointfreeco.github.io/xctest-dynamic-overlay/main/documentation/xctestdynamicoverlay/unimplemented(_:fileid:line:)-5098a
[issue-reporting-gh]: http://github.com/pointfreeco/xctest-dynamic-overlay

## Topics

### Previews

- ``DeveloperToolsSupport/PreviewTrait``

---

### OverridingDependencies

# Overriding dependencies

Learn how dependencies can be changed at runtime so that certain parts of your application can use
different dependencies.

## Overview

It is possible to change the dependencies for a particular feature running inside your application.
This can be handy when running a feature in a more controlled environment where it may not be
appropriate to communicate with the outside world. The most obvious examples of this is running a
feature in tests or Xcode previews, but there are other interesting examples too.

## The basics

For example, suppose you want to teach users how to use your feature through an onboarding
experience. In such an experience it may not be appropriate for the user's actions to cause data to
be written to disk, or user defaults to be written, or any number of things. It would be better to
use mock versions of those dependencies so that the user can interact with your feature in a fully
controlled environment.

To do this you need to make use of the
``withDependencies(from:operation:fileID:filePath:line:column:)`` function, which allows you to
inherit the dependencies from an existing object _and_ additionally override some of those
dependencies:

```swift
@Observable
final class AppModel {
  var onboardingTodos: TodosModel?

  func tutorialButtonTapped() {
    onboardingTodos = withDependencies(from: self) {
      $0.apiClient = .mock
      $0.fileManager = .mock
      $0.userDefaults = .mock
    } operation: {
      TodosModel()
    }
  }

  // ...
}
```

In the code above, the `TodosModel` is constructed with an environment that has all of the same
dependencies as the parent, `AppModel`, and further the `apiClient`, `fileManager` and
`userDefaults` have been overridden to be mocked in a controllable manner so that they do not
interact with the outside world. This way you can be sure that while the user is playing around in
the tutorial sandbox they are not accidentally making network requests, saving data to disk or
overwriting settings in user defaults.

> Note: The method ``withDependencies(from:operation:fileID:filePath:line:column:)`` used in the
> code snippet above is subtly different from ``withDependencies(_:operation:)``. It takes an extra
> argument, `from`, which is the object from which we propagate the dependencies before overriding 
> some. This allows you to propagate dependencies from object to object.
>
> In general you should _always_ use this method when constructing model objects from other model
> objects. See [Scoping dependencies](#Scoping-dependencies) for more information.

## Scoping dependencies

Extra care must be taken when overriding dependencies in order for the new dependencies to propagate
down to child models, and grandchild models, and on and on. All child models constructed should be
done so inside an invocation of ``withDependencies(from:operation:fileID:filePath:line:column:)`` so
that the child model picks up the exact dependencies the parent is using.

For example, taking the code sample from above, suppose that the `TodosModel` could drill down to an
edit screen for a particular todo. You could model that with an `EditTodoModel` and a piece of
optional state that when hydrated causes the drill down:

```swift
@Observable
final class TodosModel {
  var todos: [Todo] = []
  var editTodo: EditTodoModel?

  @ObservationIgnored
  @Dependency(\.apiClient) var apiClient
  @ObservationIgnored
  @Dependency(\.fileManager) var fileManager
  @ObservationIgnored
  @Dependency(\.userDefaults) var userDefaults

  func tappedTodo(_ todo: Todo) {
    editTodo = EditTodoModel(todo: todo)
  }

  // ...
}
```

However, when constructing `EditTodoModel` inside the `tappedTodo` method, its dependencies will go
back to the default ``DependencyKey/liveValue`` that the application launches with. It will not have
any of the overridden dependencies from when the `TodosModel` was created.

In order to make sure the overridden dependencies continue to propagate to the child feature, you
must wrap the creation of the child model in
``withDependencies(from:operation:fileID:filePath:line:column:)``:

```swift
func tappedTodo(_ todo: Todo) {
  editTodo = withDependencies(from: self) {
    EditTodoModel(todo: todo)
  }
}
```

Note that we are using `withDependencies(from: self)` in the above code. That is what allows the
`EditTodoModel` to be constructed with all the same dependencies as `self`, and should be used
even if you are not explicitly overriding dependencies.

## Testing

To override dependencies in tests you can use ``withDependencies(_:operation:)-4uz6m`` in the
same way you override dependencies in features. For example, if a model uses an API client to fetch
a user when the view appears, a test for this functionality could be written by overriding the
`apiClient` to return some mock data:

```swift
@Test
func onAppear() async {
  let model = withDependencies {
    $0.apiClient.fetchUser = { _ in User(id: 42, name: "Blob") }
  } operation: {
    FeatureModel()
  }

  #expect(model.user == nil)
  await model.onAppear()
  #expect(model.user == User(id: 42, name: "Blob"))
}
```

Sometimes there is a dependency that you want to override in a particular way for the entire test
case. For example, your feature may make extensive use of the ``DependencyValues/date`` dependency
and it may be cumbersome to override it in every test. Instead, it can be done a single time by
overriding `invokeTest` in your test case class:

```swift
final class FeatureTests: XCTestCase {
  override func invokeTest() {
    withDependencies {
      $0.date.now = Date(timeIntervalSince1970: 1234567890)
    } operation: {
      super.invokeTest()
    }
  }

  // All test functions will use the mock date generator.
}
```

Any dependencies overridden in `invokeTest` will be overridden for the entirety of the test case.

You can also implement a base test class for other test cases to inherit from in order to provide
a base set of dependencies for many test cases to use:

```swift
class BaseTestCase: XCTestCase {
  override func invokeTest() {
    withDependencies {
      // Mutate $0 to override dependencies for all test
      // cases that inherit from BaseTestCase.
      // ...
    } operation: {
      super.invokeTest()
    }
  }
}
```

[swift-identified-collections]: https://github.com/pointfreeco/swift-identified-collections
[environment-values-docs]: https://developer.apple.com/documentation/swiftui/environmentvalues

---

### QuickStart

# Quick start

Learn the basics of getting started with the library before diving deep into all of its features.

## Adding the Dependencies library as a dependency

To use this library in a SwiftPM project, add it to the dependencies of your Package.swift and
specify the `Dependencies` product in any targets that need access to the library:

```swift
let package = Package(
  dependencies: [
    .package(
      url: "https://github.com/pointfreeco/swift-dependencies",
      from: "1.0.0"
    ),
  ],
  targets: [
    .target(
      name: "<your-target-name>",
      dependencies: [
        .product(name: "Dependencies", package: "swift-dependencies")
      ]
    )
  ]
)
```

## Using your first dependency

The library allows you to register your own dependencies, but it also comes with many controllable
dependencies out of the box (see ``DependencyValues`` for a full list), and there
is a good chance you can immediately make use of one. If you are using `Date()`, `UUID()`,
`Task.sleep`, or Combine schedulers directly in your feature's logic, you can already start to use
this library.

```swift
@Observable
final class FeatureModel {
  var items: [Item] = []

  @ObservationIgnored
  @Dependency(\.continuousClock) var clock  // Controllable way to sleep a task
  @ObservationIgnored
  @Dependency(\.date.now) var now           // Controllable way to ask for current date
  @ObservationIgnored
  @Dependency(\.mainQueue) var mainQueue    // Controllable scheduling on main queue
  @ObservationIgnored
  @Dependency(\.uuid) var uuid              // Controllable UUID creation

  // ...
}
```

Once your dependencies are declared, rather than reaching out to the `Date()`, `UUID()`, etc.,
directly, you can use the dependency that is defined on your feature's model:

```swift
@Observable
final class FeatureModel {
  // ...

  func addButtonTapped() async throws {
    try await clock.sleep(for: .seconds(1))  // 👈 Don't use 'Task.sleep'
    items.append(
      Item(
        id: uuid(),  // 👈 Don't use 'UUID()'
        name: "",
        createdAt: now  // 👈 Don't use 'Date()'
      )
    )
  }
}
```

That is all it takes to start using controllable dependencies in your features. With that little
bit of upfront work done you can start to take advantage of the library's powers.

For example, you can easily control these dependencies in tests. If you want to test the logic
inside the `addButtonTapped` method, you can use the ``withDependencies(_:operation:)-4uz6m``
function to override any dependencies for the scope of one single test. It's as easy as 1-2-3:

```swift
@Test
func add() async throws {
  let model = withDependencies {
    // 1️⃣ Override any dependencies that your feature uses.
    $0.clock = .immediate
    $0.date.now = Date(timeIntervalSinceReferenceDate: 1234567890)
    $0.uuid = .incrementing
  } operation: {
    // 2️⃣ Construct the feature's model
    FeatureModel()
  }

  // 3️⃣ The model now executes in a controlled environment of dependencies,
  //    and so we can make assertions against its behavior.
  try await model.addButtonTapped()
  #expect(
    model.items == [
      Item(
        id: UUID(uuidString: "00000000-0000-0000-0000-000000000000")!,
        name: "",
        createdAt: Date(timeIntervalSinceReferenceDate: 1234567890)
      )
    ]
  )
}
```

Here we controlled the `date` dependency to always return the same date, and we controlled the
`uuid` dependency to return an auto-incrementing UUID every time it is invoked, and we even 
controlled the `clock` dependency using an [`ImmediateClock`][immediate-clock-docs] to squash all
of time into a single instant. If we did not control these dependencies this test would be very 
difficult to write since there is no way to accurately predict what will be returned by `Date()` 
and `UUID()`, and we'd have to wait for real world time to pass, making the test slow.

But, controllable dependencies aren't only useful for tests. They can also be used in Xcode
previews. Suppose the feature above makes use of a clock to sleep for an amount of time before
something happens in the view. If you don't want to literally wait for time to pass in order to see
how the view changes, you can override the clock dependency to be an "immediate" clock using
``prepareDependencies(_:)``:

```swift
#Preview {
  let _ = prepareDependencies { $0.continuousClock = ImmediateClock() }
  // All access of '@Dependency(\.continuousClock)' in this preview will
  // use an immediate clock.
  FeatureView(model: FeatureModel())
}
```

This will make it so that the preview uses an immediate clock when run, but when running in a
simulator or on device it will still use a live `ContinuousClock`. This makes it possible to
override dependencies just for previews without affecting how your app will run in production.

That is the basics to getting started with using the library, but there is still a lot more you
can do. You can learn more in depth about <doc:WhatAreDependencies> as well as
<doc:UsingDependencies>. Once comfortable with that you can learn about
<doc:RegisteringDependencies> as well as how to best leverage <doc:LivePreviewTest>. And finally,
there are more advanced topics to explore, such as <doc:DesigningDependencies>,
<doc:OverridingDependencies>, <doc:Lifetimes> and <doc:SingleEntryPointSystems>.

[immediate-clock-docs]: https://pointfreeco.github.io/swift-clocks/main/documentation/clocks/immediateclock

---

### RegisteringDependencies

# Registering dependencies

Learn how to register your own dependencies with the library so that they immediately become
available from any part of your code base.

## Overview

Although the library comes with many controllable dependencies out of the box, there are still times
when you want to register your own dependencies with the library so that you can use them with the
``Dependency`` property wrapper. There are a couple ways to achieve this, and the process is quite
similar to registering a value with [the environment][environment-values-docs] in SwiftUI.

First you create a ``DependencyKey`` protocol conformance. The minimum implementation you must
provide is a ``DependencyKey/liveValue``, which is the value used when running the app in a
simulator or on device, and so it's appropriate for it to actually make network requests to an
external server. It is usually convenient to conform the type of dependency directly to this
protocol:

```swift
extension APIClient: DependencyKey {
  static let liveValue = APIClient(/*
    Construct the "live" API client that actually makes network 
    requests and communicates with the outside world.
  */)
}
```

> Tip: There are two other values you can provide for a dependency. If you implement
> ``DependencyKey/testValue`` it will be used when running features in tests, and if you
> implement `previewValue` it  will be used while running features in an Xcode preview. You don't
> need to worry about those values when you are just getting started, and instead can add them
> later. See <Doc:LivePreviewTest> for more information.

With that done you can instantly access your API client dependency from any part of your code base:

```swift
@Observable
final class TodosModel {
  @ObservationIgnored
  @Dependency(APIClient.self) var apiClient
  // ...
}
```

This will automatically use the live dependency in previews, simulators and devices, and in tests
you can override the dependency to return mock data:

```swift
@MainActor
@Test
func fetchUser() async {
  let model = withDependencies {
    $0[APIClient.self].fetchTodos = { _ in Todo(id: 1, title: "Get milk") }
  } operation: {
    TodosModel()
  }

  await store.loadButtonTapped()
  #expect(
    model.todos == [Todo(id: 1, title: "Get milk")]
  )
}
```

## Advanced techniques

### Dependency key paths

You can take one additional step to register your dependency value at a particular key path, and
that is by extending `DependencyValues` with a property:

```swift
extension DependencyValues {
  var apiClient: APIClient {
    get { self[APIClientKey.self] }
    set { self[APIClientKey.self] = newValue }
  }
}
```

This allows you to access and override the dependency in way similar to SwiftUI environment values,
as a property that is discoverable from autocomplete:

```diff
-@Dependency(APIClient.self) var apiClient
+@Dependency(\.apiClient) var apiClient

 let model = withDependencies {
-  $0[APIClient.self].fetchTodos = { _ in Todo(id: 1, title: "Get milk") }
+  $0.apiClient.fetchTodos = { _ in Todo(id: 1, title: "Get milk") }
 } operation: {
   TodosModel()
 }
```

Another benefit of this style is the ability to scope a `@Dependency` to a specific sub-property:

```swift
// This feature only needs to access the API client's logged-in user
@Dependency(\.apiClient.currentUser) var currentUser
```

### Indirect dependency key conformances

It is not always appropriate to conform your dependency directly to the `DependencyKey` protocol,
for example if it is a type you do not own. In such cases you can define a separate type that
conforms to `DependencyKey`:

```swift
enum UserDefaultsKey: DependencyKey {
  static let liveValue = UserDefaults.standard
}
```

You can then access and override your dependency through this key type, instead of the value's type:

```swift
@Dependency(UserDefaultsKey.self) var userDefaults

let model = withDependencies {
  let defaults = UserDefaults(suiteName: "test-defaults")
  defaults.removePersistentDomain(forName: "test-defaults")
  $0[UserDefaultsKey.self] = defaults
} operation: {
  TodosModel()
}
```

If you extend dependency values with a dedicated key path, you can even make this key private:

```diff
-enum UserDefaultsKey: DependencyKey { /* ... */ }
+private enum UserDefaultsKey: DependencyKey { /* ... */ }
+
+extension DependencyValues {
+  var userDefaults: UserDefaults {
+    get { self[UserDefaultsKey.self] }
+    set { self[UserDefaultsKey.self] = newValue }
+  }
+}
```

[environment-values-docs]: https://developer.apple.com/documentation/swiftui/environmentvalues

---

### SingleEntryPointSystems

# Single entry point systems

Learn about "single entry point" systems, and why they are best suited for this dependencies
library, although it is possible to use the library with non-single entry point systems.

## Overview

A system is said to have a "single entry point" if there is one place to invoke all of its logic and
behavior. Such systems make it easy to alter the execution context a system runs in, which can be
powerful.

## Examples of single entry point systems

By far the most popular example of this in the Apple ecosystem is SwiftUI views. A view is a type
conforming to the `View` protocol and exposing a single `body` property that returns the view
hierarchy:

```swift
struct FeatureView: View {
  var body: some View {
    // All of the view is constructed in here...
  }
}
```

There is only one way to create the actual views that SwiftUI will render to the screen, and that
is by invoking the `body` property, though we never need to actually do that. SwiftUI hides all of
that from us in the `@main` entry point of the application or in `UIHostingController`.

[The Composable Architecture][tca-gh] is another example of a single entry point system, but this
time for implementing logic and behavior of a view. It provides a protocol that one conforms to and
it has a single requirement, `reduce`, which is responsible for mutating the feature's state and
returning effects to execute:

```swift
import ComposableArchitecture

@Reducer
struct Feature {
  struct State {
    // ...
  }
  enum Action {
    // ...
  }
  var body: some Reducer<State, Action> {
    // All of the feature's logic and behavior is implemented here...
  }
}
```

Again, there is only one way to execute this feature's logic, and that is by invoking the `reduce`
method. However, you never actually need to do that in practice. The Composable Architecture
hides all of that from you, and instead you just construct a `Store` at the root of the application.

Another example of a single entry point system is a server framework. Such frameworks usually
have a simple request-to-response lifecycle. It starts by the framework receiving a request from an
external client. Then one uses the framework's tools in order to interpret that request and build
up a response to send back to the client. This again describes just a single point for all logic to
be executed for a particular request.

So, there are a lot of examples of "single entry point" systems out there, but it's also not the
majority. There are plenty of examples that do not fall into this paradigm, such as observable
objects, all of UIKit and more. If you _are_ dealing with a single entry point system, then there
are some really great superpowers that can be unlocked...

## Altered execution environments

One of the most interesting aspects of single entry point systems is that they have a well-defined
scope from beginning to end, and that makes it possible to easily alter their execution context.

For example, SwiftUI views have a powerful feature known as ["environment values"][env-values-docs].
They allow you to propagate values deep into a view hierarchy and can be overridden for just one
small subset of the view tree.

The following SwiftUI view stacks a header view on top of a footer view, and overrides the
foreground color for the header:

```swift
struct ContentView: View {
  var body: some View {
    VStack {
      HeaderView()
        .foregroundColor(.red)
      FooterView()
    }
  }
}
```

The `.red` foreground color will be applied to every view in `HeaderView`, including deeply nested
views. And most importantly, that style is applied only to the header and not to the
`FooterView`.

The `foregroundColor` view modifier is powered by [environment values][env-values-docs] under the
hood, as can be seen by printing the type of `ContentView`'s body:

```swift
print(ContentView.Body.self)
// VStack<
//   TupleView<(
//     ModifiedContent<
//       HeaderView,
//       _EnvironmentKeyWritingModifier<Optional<Color>>
//     >,
//     FooterView
//   )>
// >
```

The presence of `_EnvironmentKeyWritingModifier` shows that an environment key is being written.

This is an incredibly powerful feature of SwiftUI, and the only reason it works so well and is so
easy to understand is specifically because SwiftUI views form a single entry point system. That
makes it possible to alter the execution environment of `HeaderView` so that its foreground color
is red, and that altered state does not affect the other parts of the view tree, such as
`FooterView`.

The same is possible with the Composable Architecture and the dependencies of features. For example,
suppose some feature's logic and behavior was decomposed into the logic for the "header" and
"footer," and that we wanted to alter the dependencies used in the header. This can be done using
the `.dependency` method on reducers, which acts similarly to the
[`.environment`][env-view-modifier-docs] view modifier from SwiftUI:

```swift
@Reducer
struct Feature {
  struct State {
    // ...
  }
  enum Action {
    // ...
  }
  var body: some Reducer<State, Action> {
    Header()
      .dependency(\.fileManager, .mock)
      .dependency(\.userDefaults, .mock)

    Footer()
  }
}
```

This will override the `fileManager` and `userDefaults` dependency to be mocks for the `Header`
feature (as well as all features called to from inside `Header`), but will leave the dependencies
untouched for all other features, including `Footer`.

This pattern can also be repeated for server applications. It is possible to alter the execution
environment on a per-request basis, and even for just a subset of the request-to-response lifecycle.

It is incredibly powerful to be able to do this, but it all hinges on being able to express your
system as a single point of entry. Without that it becomes a lot more difficult to alter the
execution context of the system, or a sub-system, because there is not only one place to do so.

## Non-single entry point systems

While this library thrives when applied to "single entry point" systems, it is still possible to use
with other kinds of systems. You just have to be a little more careful. In particular, you must be
careful where you add dependencies to your features and how you construct features that use
dependencies.

When adding a dependency to a feature modeled in an observable object, you should make use of
`@Dependency` only for the object's instance properties:

```swift
@Observable
final class FeatureModel {
  @ObservationIgnored
  @Dependency(\.apiClient) var apiClient
  @ObservationIgnored
  @Dependency(\.date) var date
  // ...
}
```

And similarly for `UIViewController` subclasses:

```swift
final class FeatureViewController: UIViewController {
  @Dependency(\.apiClient) var apiClient
  @Dependency(\.date) var date
  // ...
}
```

Then you are free to use those dependencies from anywhere within the model and controller.

Then, if you create a new model or controller from within an existing model or controller, you
will need to take an extra step to make sure that the parent feature's dependencies are propagated
to the child.

For example, if your SwiftUI model holds a piece of optional state that drives a sheet, then when
hydrating that state you will want to wrap it in
``withDependencies(from:operation:fileID:filePath:line:column:)``:

```swift
@Observable
final class FeatureModel {
  var editModel: EditModel?

  @ObservationIgnored
  @Dependency(\.apiClient) var apiClient
  @ObservationIgnored
  @Dependency(\.date) var date

  func editButtonTapped() {
    editModel = withDependencies(from: self) {
      EditModel()
    }
  }
}
```

This makes it so that if `FeatureModel` were constructed with some of its dependencies overridden
(see <doc:OverridingDependencies>), then those changes will also be visible to `EditModel`.

The same principle holds for UIKit. When constructing a child view controller to be presented,
be sure to wrap its construction in
``withDependencies(from:operation:fileID:filePath:line:column:)``:

```swift
final class FeatureViewController: UIViewController {
  @Dependency(\.apiClient) var apiClient
  @Dependency(\.date) var date

  func editButtonTapped() {
    let controller = withDependencies(from: self) {
      EditViewController()
    }
    present(controller, animated: true, completion: nil)
  }
}
```

If you make sure to always use ``withDependencies(from:operation:fileID:filePath:line:column:)``
when constructing child models and controllers you can be sure that changes to dependencies at
any layer of your application will be visible at any layer below it. See <doc:Lifetimes> for
more information on how dependency lifetimes work.

[tca-gh]: http://github.com/pointfreeco/swift-composable-architecture
[env-values-docs]: https://developer.apple.com/documentation/swiftui/environment-values
[env-view-modifier-docs]: https://developer.apple.com/documentation/swiftui/view/environment(_:_:)

---

### Testing

# Testing

One of the main reasons to control dependencies is to allow for easier testing. Learn some tips and 
tricks for writing better tests with the library.

## Overview

In the article <doc:LivePreviewTest> you learned how to define a ``TestDependencyKey/testValue``
when registering your dependencies, which will be automatically used during tests. In this article
we cover more detailed information about how to actually write tests with overridden dependencies,
as well as some tips and gotchas to keep in mind.

* [Swift's native Testing framework](#Swifts-native-Testing-framework)
* [Xcode's XCTest framework](#Xcodes-XCTest-framework)
* [Changing dependencies during tests](#Changing-dependencies-during-tests)
* [Testing gotchas](#Testing-gotchas)
  * [Testing host application](#Testing-host-application)
  * [Statically linking your tests target to Dependencies](#Statically-linking-your-tests-target-to-Dependencies)
  * [Test case leakage](#Test-case-leakage)
  * [Static @Dependency](#Static-Dependency)
  * [Parameterized and repeated @Test runs](#Parameterized-and-repeated-Test-runs)

## Swift's native Testing framework

This library has full support for Swift's native Testing framework, in addition to Xcode's XCTest
framework. It even works with in process concurrent tests without tests bleeding over to other 
tests.

The most direct way to override dependencies in a test is to simply wrap the entire test function
in ``withDependencies(_:operation:)`` in order to override the dependencies for the duration of that
test:

```swift
@Test func basics() {
  withDependencies {
    $0.uuid = .incrementing
  } operation: {
    let model = FeatureModel()
    // Invoke methods on 'model' and make assertions
  }
}
```

The library also ships with test traits that can help allivate the nesting incurred by 
``withDependencies(_:operation:)``. In order to get access to the test traits you must link the
DependenciesTestSupport library to your test target, after which you can do the following:

```swift
@Test(.dependency(\.uuid, .incrementing)) 
func basics() {
  let model = FeatureModel()
  // Invoke methods on 'model' and make assertions
}
```

> Important: Make sure to link DependenciesTestSupport only to your test targets and never your 
> app targets. Linking that library to an app target will cause the project to fail to compile.

It is also possible to override dependencies for an entire `@Suite` using a suite trait:

```swift
@Suite(.dependency(\.uuid, .incrementing))
struct MySuite {
  @Test func basics() {
    let model = FeatureModel()
    // Invoke methods on 'model' and make assertions
  }
}
```

If you need to override multiple dependencies you can do so using the `.dependencies` test trait:

```swift
@Suite(.dependencies {
  $0.date.now = Date(timeIntervalSince1970:12324567890)
  $0.uuid = .incrementing
})
struct MySuite {
  @Test func basics() {
    let model = FeatureModel()
    // Invoke methods on 'model' and make assertions
  }
}
```

Because tests in Swift's native Testing framework run in parallel and in process, it is possible
for multiple tests running to access the same dependency. This can be troublesome if those 
dependencies are stateful, making it possible for one test to make changes to a dependency that
another test sees. This will cause mysterious test failures and the type of test failure you get
may even depend on the order the tests ran.

To properly handle this we recommend having a "base suite" that all of your tests and suites are
nested inside. That will allow you to provide a fresh set of dependencies to each test, and it will
not be possible for changes in one test to bleed over to another test.

To do this, simply define a `@Suite` and use the `.dependencies` trait:

```swift
@Suite(.dependencies) struct BaseSuite {}
```

This type does not need to have anything in its body, and the `.dependencies` trait is responsible
for giving each test in the suite its own scratchpad of dependencies.

Then nest all of your `@Suite`s and `@Test`s in this type:

```swift
extension BaseSuite {
  @Suite struct FeatureTests {
    @Test func basics() {
      // ...  
    }
  }
}
```

This will allow all tests to run in parallel and in process without them affecting each other.

## Xcode's XCTest framework

This library also works with Xcode's testing framework, known as XCTest. Just as with Swift's 
Testing framework, one can override dependencies for a test by wrapping the body of a test in
``withDependencies(_:operation:)``:

```swift
func testBasics() {
  withDependencies {
    $0.uuid = .incrementing
  } operation: {
    let model = FeatureModel()
    // Invoke methods on 'model' and make assertions
  }
}
```

XCTest does not support traits, and so it is not possible to override dependencies on a per-test
basis without incurring the indentation of ``withDependencies(_:operation:)``. However, you can
override all dependencies for an entire test case by implementing the `invokeTest` method:

```swift
class FeatureTests: XCTestCase {
  override func invokeTest() {
    withDependencies { 
      $0.uuid = .incrementing
    } operation: {
      super.invokeTest()
    }
  }

  func testBasics() {
    // Test has 'uuid' dependency overridden.
  }
}
```

## Changing dependencies during tests

While it is most common to set up all dependencies at the beginning of a test and then make 
assertions, sometimes it is necessary to also change the dependencies in the middle of a test.
This can be very handy for modeling test flows in which a dependency is in a failure state at
first, but then later becomes successful. To do this one can simply use 
``withDependencies(_:operation:)`` again inside the body of your test.

For example, suppose we have a login feature such that if you try logging in and an error is thrown
causing a message to appear. But then later, if login succeeds that message goes away. We can
test that entire flow, from end-to-end, but starting the API client dependency in a state where
login fails, and then later change the dependency so that it succeeds using 
``withDependencies(_:operation:)``:

```swift
@Test(.dependency(\.apiClient.login, { _, _ in throw LoginFailure() }))
func retryFlow() async {
  let model = LoginModel()
  await model.loginButtonTapped()
  #expect(model.errorMessage == "We could not log you in. Please try again")

  withDependencies {
    $0.apiClient.login = { email, password in 
      LoginResponse(user: User(id: 42, name: "Blob"))
    }
  } operation: {
    await model.loginButtonTapped()
    #expect(model.errorMessage == nil)
  }
}
```

Even though the `LoginModel` was created in the context of the API client failing it still sees 
the updated dependency when run in the new `withDependencies` context.

## Testing gotchas

### Testing host application

This is not well known, but when an application target runs tests it actually boots up a simulator
and runs your actual application entry point in the simulator. This means while tests are running,
your application's code is separately also running. This can be a huge gotcha because it means you
may be unknowingly making network requests, tracking analytics, writing data to user defaults or
to the disk, and more.

This usually flies under the radar and you just won't know it's happening, which can be problematic.
But, once you start using this library to control your dependencies the problem can surface in a 
very visible manner. Typically, when a dependency is used in a test context without being overridden,
a test failure occurs. This makes it possible for your test to pass successfully, yet for some
mysterious reason the test suite fails. This happens because the code in the _app host_ is now
running in a test context, and accessing dependencies will cause test failures.

This only happens when running tests in a _application target_, that is, a target that is 
specifically used to launch the application for a simulator or device. This does not happen when
running tests for frameworks or SwiftPM libraries, which is yet another good reason to modularize
your code base.

However, if you aren't in a position to modularize your code base right now, there is a quick
fix. Our [Issue Reporting][issue-reporting-gh] library, which is transitively included
with this library, comes with a property you can check to see if tests are currently running. If
they are, you can omit the entire entry point of your application.

For example, for a pure SwiftUI entry point you can do the following to keep your application from
running during tests:

```swift
import IssueReporting
import SwiftUI

@main
struct MyApp: App {
  var body: some Scene {
    WindowGroup {
      if !isTesting {
        // Your real root view
      }
    }
  }
}
```

And in an `UIApplicationDelegate`-based entry point you can do the following:

```swift
func application(
_ application: UIApplication,
didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
  guard !isTesting else { return true }
  // ...
}
```

That will allow tests to run in the application target without your actual application code 
interfering.

### Statically linking your tests target to Dependencies

If you statically link the `Dependencies` module to your tests target, its implementation may clash
with the implementation that is statically linked to the app itself. It then may use a different
`DependencyValues` base type in the app and in tests, and you may encounter test failures where
dependency overrides performed with `withDependencies` seem ineffective.

In such cases Xcode will display multiple warnings similar to:

> Class _TtC12Dependencies[…] is implemented in both […] and […].
> One of the two will be used. Which one is undefined.

The solution is to remove the static link to `Dependencies` from your test target, as you
transitively get access to it through the app itself. In Xcode, go to "Build Phases" and remove
"Dependencies" from the "Link Binary With Libraries" section. When using SwiftPM, remove the
"Dependencies" entry from the `testTarget`'s' `dependencies` array in `Package.swift`.

### Test case leakage

Sometimes it is possible to have tests that pass successfully when run in isolation, but somehow 
fail when run together as a suite. This can happen when using escaping closures in tests, which 
creates an alternate execution flow, allowing a test's code to continue running long after the test 
has  finished.

This can happen in any kind of test, not just when using this dependencies library. For example, 
each of the following test methods passes when run in isolation, yet running the whole test suite
fails:

```swift
final class SomeTest: XCTestCase {
  func testA() {
    Task {
      try await Task.sleep(for: .seconds(0.1))
      XCTFail()
    }
  }
  func testB() async throws {
    try await Task.sleep(for: .seconds(0.15))
  }
}
```

This happens because `testA` escapes some work to be executed and then finishes immediately with
no failure. Then, while `testB` is executing, the escaped work from `testA` finally gets around
to executing and causes a failure.

You can also run into this issue while using this dependencies library. In particular, you may
see test a failure for accessing a ``TestDependencyKey/testValue`` of a dependency that your
test is not even using. If running that test in isolation passes, then you probably have some
other test accidentally leaking its code into your test. You need to check every other test in the 
suite to see if any of them use escaping closures causing the leakage.

### Static @Dependency

You should never use the `@Dependency` property wrapper as a static variable:

```swift
class Model {
  @Dependency(\.date) static var date
  // ...
}
```

You will not be able to override this dependency in the normal fashion. In general there is no need
to ever have a static dependency, and so you should avoid this pattern.

### Parameterized and repeated @Test runs

> Important: If targeting Swift 6.1+ (Xcode 16.3+), then this gotcha does not apply and can be
> ignored.

The library comes with support for Swift's new native Testing framework. However, as there are still
still features missing from the Testing framework that XCTest has, there may be some additional
steps you must take.

If you are are writing a _parameterized_ test using the `@Test` macro, you will need to surround the
entire body of your test in [`withDependencies`](<doc:withDependencies(_:operation:)>) that
resets the entire set of values to guarantee that a fresh set of dependencies is used per parameter:

```swift
@Test(arguments: [1, 2, 3])
func feature(_ number: Int) {
  withDependencies {
    $0 = DependencyValues()
  } operation: {
    // All test code in here...
  }
}
```

This will guarantee that dependency state does not bleed over to each parameter of the test.

[issue-reporting-gh]: http://github.com/pointfreeco/swift-issue-reporting

---

### UsingDependencies

# Using dependencies

Learn how to use the dependencies that are registered with the library.

## Overview

Once a dependency is registered with the library (see <doc:RegisteringDependencies> for more info),
one can access the dependency with the ``Dependency`` property wrapper. This is most commonly done
by adding `@Dependency` properties to your feature's model, such as an observable object, or
controller, such as `UIViewController` subclass. It can be used in other scopes too, such as
functions, methods and computed properties, but there are caveats to consider, and so doing that
is not recommended until you are very comfortable with the library.

The library comes with many common dependencies that can be used in a controllable manner, such as
date generators, clocks, random number generators, UUID generators, and more.

For example, suppose you have a feature that needs access to a date initializer, a continuous
clock for time-based asynchrony, and a UUID initializer. All 3 dependencies can be added to your
feature's model:

```swift
@Observable
final class TodosModel {
  @ObservationIgnored @Dependency(\.continuousClock) var clock
  @ObservationIgnored @Dependency(\.date) var date
  @ObservationIgnored @Dependency(\.uuid) var uuid

  // ...
}
```

Then, all 3 dependencies can easily be overridden with deterministic versions when testing the
feature:

```swift
@MainActor
@Test
func todos() async {
  let model = withDependencies {
    $0.continuousClock = .immediate
    $0.date.now = Date(timeIntervalSinceReferenceDate: 1234567890)
    $0.uuid = .incrementing
  } operation: {
    TodosModel()
  }

  // Invoke methods on `model` and make assertions...
}
```

All references to `continuousClock`, `date`, and `uuid` inside the `TodosModel` will now use the
controlled versions.

---

### WhatAreDependencies

# What are dependencies?

Learn what dependencies are, how they complicate your code, and why you want to control them.

## Overview

Dependencies in an application are the types and functions that need to interact with outside
systems that you do not control. Classic examples of this are API clients that make network requests
to servers, but also seemingly innocuous things such as the `UUID` and `Date` initializers, and even
clocks and timers, can be thought of as dependencies.

By controlling the dependencies our features need to do their jobs we gain the ability to completely
alter the execution context a feature runs in. This means in tests and Xcode previews you can
provide a mock version of an API client that immediately returns some stubbed data rather than
making a live network request to a server.

## The need for controlled dependencies

Suppose that you are building a feature that displays a message to the user after 10 seconds. This
logic can be packaged up into an observable object:

```swift
@Observable
final class FeatureModel {
  var message: String?

  func onAppear() async {
    do {
      try await Task.sleep(for: .seconds(10))
      message = "Welcome!"
    } catch {}
  }
}
```

And a view can make use of that model:

```swift
struct FeatureView: View {
  let model: FeatureModel

  var body: some View {
    Form {
      if let message = model.message {
        Text(message)
      }

      // ...
    }
    .task { await model.onAppear() }
  }
}
```

This code works just fine at first, but it has some problems:

First, if you want to iterate on the styling of the message in an Xcode preview you will have to
wait for 10 whole seconds of real world time to pass before the message appears. This completely
destroys the fast, iterative nature of previews.

Second, if you want to write a test for this feature, you will again have to wait for 10 whole
seconds of real world time to pass. This slows down your test suite, making it less likely you will
add new tests in the future if the whole suite takes a long time to run.

The reason this code does not play nicely with Xcode previews or tests is because it has an
uncontrolled dependency on an outside system: `Task.sleep`. That API can only sleep for a real world
amount of time.

## Controlling the dependency

It would be far better if we could swap out different notions of "sleeping" in our feature so that
when run in the simulator or device, `Task.sleep` could be used, but in previews or tests other
forms of sleeping could be used.

The tool to do this is known as the `Clock` protocol, which is a tool from the Swift standard
library. Instead of reaching out to `Task.sleep` directly, we can "inject" our dependency on
time-based asynchrony by holding onto a clock in the feature's model by using the ``Dependency``
property  wrapper and ``DependencyValues/continuousClock`` dependency value:

```swift
@Observable
final class FeatureModel {
  var message: String?

  @ObservationIgnored
  @Dependency(\.continuousClock) var clock

  func onAppear() async {
    do {
      try await clock.sleep(for: .seconds(10))
      message = "Welcome!"
    } catch {}
  }
}
```

> Note: Using the `@ObservationIgnored` macro is necessary when using `@Observable` because 
> `@Dependency` is a property wrapper. 

That small change makes this feature much friendlier to Xcode previews and testing.

For previews, you can use ``prepareDependencies(_:)`` to override the
``DependencyValues/continuousClock`` dependency to be an "immediate" clock, which is a clock that
does not actually sleep for any amount of time:

```swift
#Preview {
  let _ = prepareDependencies { $0.continuousClock = ImmediateClock() }
  FeatureView(model: FeatureModel())
}
```

This will cause the message to appear immediately. No need to wait 10 seconds.

> Tip: We have a [series of episodes][clocks-collection] discussing the `Clock` protocol in depth
and showing how it can be used to control time-based asynchrony.

Further, in tests you can also override the clock dependency to use an immediate clock, also using
the ``withDependencies(_:operation:)-4uz6m`` helper:

```swift
@Test
func message() async {
  let model = withDependencies {
    $0.continuousClock = .immediate
  } operation: {
    FeatureModel()
  }

  #expect(model.message == nil)
  await model.onAppear()
  #expect(model.message == "Welcome!")
}
```

This test will pass quickly, and deterministically, 100% of the time. This is why it is so
important to control dependencies that interact with outside systems.

[clocks-collection]: https://www.pointfree.co/collections/concurrency/clocks

---

### Dependencies

# ``Dependencies``

A dependency management library inspired by SwiftUI's "environment."

## Additional Resources

- [GitHub Repo](https://github.com/pointfreeco/swift-dependencies)
- [Discussions](https://github.com/pointfreeco/swift-dependencies/discussions)
- [Point-Free Videos](http://pointfree.co)

## Overview

Dependencies are the types and functions in your application that need to interact with outside
systems that you do not control. Classic examples of this are API clients that make network
requests to servers, but also seemingly innocuous things such as `UUID` and `Date` initializers,
file access, user defaults, and even clocks and timers, can all be thought of as dependencies.

You can get really far in application development without ever thinking about dependency management 
(or, as some like to call it, "dependency injection"), but eventually uncontrolled dependencies can 
cause many problems in your code base and development cycle:

  * Uncontrolled dependencies make it **difficult to write fast, deterministic tests** because you 
    are susceptible to the vagaries of the outside world, such as file systems, network 
    connectivity, internet speed, server uptime, and more.
    
  * Many dependencies **do not work well in SwiftUI previews**, such as location managers and speech
    recognizers, and some **do not work even in simulators**, such as motion managers, and more. 
    This prevents you from being able to easily iterate on the design of features if you make use of 
    those frameworks.

  * Dependencies that interact with 3rd party, non-Apple libraries (such as Firebase, web socket
    libraries, network libraries, etc.) tend to be heavyweight and take a **long time to compile**. 
    This can slow down your development cycle.

For these reasons, and a lot more, it is highly encouraged for you to take control of your
dependencies rather than letting them control you.

But, controlling a dependency is only the beginning. Once you have controlled your dependencies,
you are faced with a whole set of new problems:

  * How can you **propagate dependencies** throughout your entire application in a way that is more
    ergonomic than explicitly passing them around everywhere, but safer than having a global
    dependency?

  * How can you **override dependencies** for just one portion of your application? This can be 
    handy for overriding dependencies for tests and SwiftUI previews, as well as specific user 
    flows such as onboarding experiences.
    
  * How can you be sure you **overrode _all_ dependencies** a feature uses in tests? It would be
    incorrect for a test to mock out some dependencies but leave others as interacting with the
    outside world.

This library addresses all of the points above, and much, _much_ more.

## Topics

### Getting started

- <doc:QuickStart>
- <doc:WhatAreDependencies>

### Essentials

- <doc:UsingDependencies>
- <doc:RegisteringDependencies>
- <doc:LivePreviewTest>
- <doc:Testing>

### Advanced

- <doc:DesigningDependencies>
- <doc:OverridingDependencies>
- <doc:Lifetimes>
- <doc:SingleEntryPointSystems>

### Dependency management

- ``Dependency``
- ``DependencyValues``
- ``DependencyKey``
- ``DependencyContext``

---

### Dependency

# ``Dependencies/Dependency``

## Topics

### Using a dependency

- ``init(_:fileID:filePath:line:column:)-1f0mh``
- ``init(_:fileID:filePath:line:column:)-1ytea``

### Getting the value

- ``wrappedValue``

---

### DependencyKey

# ``Dependencies/DependencyKey``

## Topics

### Registering a dependency

- ``Value``
- ``liveValue``
- ``testValue-535kh``
- ``previewValue-36s5j``

### Modularizing a dependency

- ``TestDependencyKey``

---

### DependencyValues

# ``Dependencies/DependencyValues``

## Topics

### Creating and accessing values

- ``init()``
- ``subscript(_:fileID:filePath:line:column:function:)``
- ``subscript(_:)``

### Overriding values

- ``withDependencies(_:operation:)-4uz6m``
- ``withDependencies(from:operation:fileID:filePath:line:column:)``
- ``prepareDependencies(_:)``

### Escaping contexts

- ``withEscapedDependencies(_:)-5xvi3``

### Dependency values

- ``assert``
- ``assertionFailure``
- ``calendar``
- ``context``
- ``continuousClock``
- ``date``
- ``fireAndForget``
- ``locale``
- ``mainQueue``
- ``mainRunLoop``
- ``notificationCenter``
- ``openURL``
- ``precondition``
- ``suspendingClock``
- ``timeZone``
- ``urlSession``
- ``uuid``
- ``withRandomNumberGenerator``

### Default contexts

- ``live``
- ``preview``
- ``test``

### Deprecations

- <doc:DependencyValuesDeprecations>

---

### DependencyValuesAssert

# ``Dependencies/DependencyValues/assert``

## Topics

### Dependency values

- ``AssertionEffect``
- ``AssertionFailureEffect``

---

### DependencyValuesContext

# ``Dependencies/DependencyValues/context``

## Topics

### Dependency context

- ``DependencyContext``

---

### DependencyValuesDate

# ``Dependencies/DependencyValues/date``

## Topics

### Dependency value

- ``DateGenerator``

---

### DependencyValuesDeprecations

# Deprecations

Review unsupported dependency values APIs and their replacements.

## Overview

Avoid using deprecated APIs in your app. Select a method to see the replacement that you should use
instead.

## Topics

### Overriding values

- ``DependencyValues/withTestValues(_:assert:)-6erij``
- ``DependencyValues/withTestValues(_:assert:)-6qwh1``
- ``DependencyValues/withValues(_:operation:)-9prz8``
- ``DependencyValues/withValues(_:operation:)-4omsq``
- ``DependencyValues/withValue(_:_:operation:)-4lu7m``
- ``DependencyValues/withValue(_:_:operation:)-65nv4``

---

### DependencyValuesFireAndForget

# ``Dependencies/DependencyValues/fireAndForget``

## Topics

### Dependency value

- ``FireAndForget``

---

### DependencyValuesOpenURL

# ``Dependencies/DependencyValues/openURL``

## Topics

### Dependency value

- ``OpenURLEffect``

---

### DependencyValuesUUID

# ``Dependencies/DependencyValues/uuid``

## Topics

### Dependency value

- ``UUIDGenerator``

### Helpers

- ``Foundation/UUID``

---

### DependencyValuesWithRandomNumberGenerator

# ``Dependencies/DependencyValues/withRandomNumberGenerator``

## Topics

### Dependency value

- ``WithRandomNumberGenerator``

---

### TestDependencyKey

# ``Dependencies/TestDependencyKey``

## Topics

### Registering a dependency

- ``Value``
- ``testValue``
- ``previewValue-8u2sy``

---

### WithDependencies

# ``Dependencies/withDependencies(_:operation:)``

## Topics

### Overloads

- ``withDependencies(isolation:_:operation:)``

---

### WithDependenciesFrom

# ``Dependencies/withDependencies(from:operation:fileID:filePath:line:column:)``

## Topics

### Overloads

- ``withDependencies(from:isolation:operation:fileID:filePath:line:column:)``
- ``withDependencies(from:_:operation:fileID:filePath:line:column:)``
- ``withDependencies(from:isolation:_:operation:fileID:filePath:line:column:)``

---

### WithEscapedDependencies

# ``Dependencies/withEscapedDependencies(_:)-5xvi3``

## Topics

### Yielding escaped dependencies

- ``DependencyValues/Continuation``

### Overloads

- ``withEscapedDependencies(_:)-74wb7``

---

