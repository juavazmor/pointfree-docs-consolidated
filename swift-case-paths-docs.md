# pointfreeco/swift-case-paths Documentation

Auto-generated from https://github.com/pointfreeco/swift-case-paths
Generated on: Sun Jan  4 11:08:12 UTC 2026

## Documentation from Sources/CasePaths/Documentation.docc

### CasePathableMacro

# ``CasePaths/CasePathable()``

## Further reading

See the [`CasePathsCore`](../casepathscore) module for the `CasePathable` protocol and other core
library types.

---

### CasePaths

# ``CasePaths``

Case paths bring the power and ergonomics of key paths to enums.

## Overview

This module exports the core functionality of the Case Paths library, as well as the `@CasePathable`
macro.

To read the core documentation, see the [`CasePathsCore`](./casepathscore) module.

## Topics

### Creating case paths

- ``CasePathable()``

### Deprecated interfaces

- <doc:Deprecations>

---

### Deprecations

# Deprecated interfaces

Review unsupported CasePaths APIs and their replacements.

## Overview

Avoid using deprecated APIs in your app. See the replacement that you should use instead.

## Topics

### Functions

- ``extract(case:from:)-262ip``
- ``extract(case:from:)-ylkc``
- ``extract(_:)-34fjh``
- ``extract(_:)-3twft``

### Creating paths

- ``CasePathsCore/AnyCasePath``

### Type aliases

- ``CasePath``

### Test helpers

- ``XCTModify(_:case:_:_:file:line:)-14526``
- ``XCTModify(_:case:_:_:file:line:)-1pef3``
- ``XCTModify(_:_:_:file:line:)``

---

### XCTModify

# ``CasePaths/XCTModify(_:case:_:_:file:line:)-14526``

## Topics

### Testing optionals

- ``XCTModify(_:_:_:file:line:)``

### Deprecated test helpers

- <doc:XCTModifyDeprecations>

---

### XCTModifyDeprecations

# Deprecated test helpers

Review unsupported CasePaths APIs and their replacements.

## Overview

Avoid using deprecated APIs in your app. See the replacement that you should use instead.

## Topics

### XCTModify

- ``XCTModify(_:case:_:_:file:line:)-1pef3``

### XCTUnwrap

- ``XCTUnwrap(_:case:_:file:line:)``

---

## Documentation from Sources/CasePathsCore/Documentation.docc

### CasePathsCore

# ``CasePathsCore``

Case paths bring the power and ergonomics of key paths to enums.

## Overview

This module contains the core functionality of the Case Paths library, minus the `@CasePathable`
macro, and is automatically imported when you `import CasePaths`

See the [`CasePaths`](./casepaths) module for information about the `@CasePathable` macro and
other non-core functionality.

## Topics

### Creating case paths

- ``CasePathable``
- ``CaseKeyPath``

### Swift support

- ``Swift/Optional``
- ``Swift/Result``
- ``Swift/Never``

### Migration guides

- <doc:MigrationGuides>

---

### AnyCasePath

# ``CasePathsCore/AnyCasePath``

## Topics

### Creating paths

- ``init(embed:extract:)``
- ``init(_:)``
- ``init()``

### Extracting and embedding values

- ``extract(from:)``
- ``embed(_:)``

### Appending paths

- ``subscript(dynamicMember:)``

---

### CaseKeyPath

# ``CaseKeyPath``

## Topics

### Creating root values

- ``Swift/KeyPath/callAsFunction(_:)``
- ``Swift/KeyPath/callAsFunction()``

### Matching associated values

- ``Swift/KeyPath/~=(_:_:)``

### Partial case paths

- ``PartialCaseKeyPath``
- ``CasePathable/subscript(case:)-73fim``

---

### CasePathable

# ``CasePathsCore/CasePathable``

## Topics

### Deriving case key paths

- ``Cases``

### Extracting, replacing, and modifying values

- ``subscript(case:)-3yqx3``
- ``subscript(case:)-2t4f8``
- ``modify(_:yield:fileID:filePath:line:column:)``

### Case properties

- ``is(_:)``
- ``subscript(dynamicMember:)-emck``
- ``subscript(dynamicMember:)-dm5y``

### Iteration and reflection

- ``CasePathIterable``
- ``CasePathReflectable``

### Manual conformances

- ``AllCasePaths``
- ``allCasePaths``
- ``AnyCasePath``

---

### MigrationGuides

# Migration guides

Learn how to upgrade your application to the newest version of Case Paths.

## Overview

Case Paths is under constant development, and we are always looking for ways to simplify the
library, and make it more powerful. As such, we often need to deprecate certain APIs in favor of
newer ones. We recommend people update their code as quickly as possible to the newest APIs, and
these guides contain tips to do so.

> Important: Before following any particular migration guide be sure you have followed all the 
> preceding migration guides.

## Topics

- <doc:MigratingTo1.1>

---

### MigratingTo1.1

# Migrating to 1.1

Learn how to migrate existing case path code to utilize the new `@CasePathable` macro and
``CaseKeyPath``s.

## Overview

CasePaths 1.1 introduces new APIs for deriving case paths that are safer, more ergonomic, more
performant, and more powerful than the existing APIs.

In past versions of the library, the primary way to derive a case path was via the form:

```
/<#enum name#>.<#case#>
```

It kind of looks like a key path with the `\` tilting the wrong way, but is actually an invocation
of a `/` prefix operator with an `Enum.case` initializer. Given just this initializer, the function
uses runtime reflection to produce a `CasePath` value.

So given an enum:

```swift
enum UserAction {
  case home(HomeAction)
}
```

One can produce a case path:

```swift
/UserAction.home
```

While the library has strived to optimize this reflection mechanism and work around bugs in the
runtime, it now offers a much better solution that is free of reflection-based code.

Deriving case paths is now a two-step process that is still mostly free of boilerplate:

1. You attach the `@CasePathable` macro to your enum:

```swift
@CasePathable
enum UserAction {
  case home(HomeAction)
}
```

2. You derive a case path using an actual key path expression:

```swift
\UserAction.Cases.home
```

This key path expression returns a ``CaseKeyPath``, which is a special kind of key path for enums
that can extract, modify, and embed the associated value of an enum case.

### Passing case key paths to APIs that take case paths

While libraries that use case paths should be updated to take ``CaseKeyPath``s directly, and should
deprecate APIs that take `CasePath`s (now ``AnyCasePath``s), you can continue to use these existing
APIs by converting case key paths to type-erased case paths via ``AnyCasePath/init(_:)``:

```swift
// Before:
action: /UserAction.home

// After:
action: AnyCasePath(\.home)
```

And when a library begins to provide APIs that take case key paths, you can pass a key path
expression directly:

```swift
action: \.home
```

### Working with case key paths

If you maintain APIs that take `CasePath` (now ``AnyCasePath``) values, you should introduce new
APIs that take ``CaseKeyPath``s instead. ``CaseKeyPath``s have all the functionality of
``AnyCasePath``s (and more), but you work with them more like regular key paths:

#### Extracting associated values

```swift
// Before:
casePath.extract(from: root)

// After:
root[case: casePath]
```

#### Embedding associated values

```swift
// Before:
casePath.embed(value)

// After:
casePath(value)
```

Case key paths can also replace an enum's existing associated value via
``CasePathable/subscript(case:)-2t4f8``:

```swift
root[case: casePath] = value
```

#### Modifying associated values

```swift
// Before:
casePath.modify(&root) {
  $0.count += 1
}

// After:
root.modify(casePath) {
  $0.count += 1
}
```

---

