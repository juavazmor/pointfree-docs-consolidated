# pointfreeco/sqlite-data Documentation

Auto-generated from https://github.com/pointfreeco/sqlite-data
Generated on: Sun Jan  4 11:08:51 UTC 2026

## Documentation from Sources/SQLiteData/Documentation.docc

### AddingToGRDB

# Adding to an existing GRDB application

Learn how to add SQLiteData to an existing app that uses GRDB.

## Overview

[GRDB] is a powerful SQLite library for Swift applications, and it is what is used by SQLiteData
to interact with SQLite under the hood, such as performing queries and observing changes to the
database. If you have an existing application using GRDB, and would like to use the tools of this
library, such as [`@FetchAll`](<doc:FetchAll>), the SQL query builder, and
[CloudKit synchronization](<doc:CloudKit>), then there are a few steps you must take.

## Replace PersistableRecord and FetchableRecord with @Table

The `PersistableRecord` and `FetchableRecord` protocols in GRDB facilitate saving data to the
database and querying for data in the database. In SQLiteData, the `@Table` macro is responsible
for this functionality.

```diff
-struct Reminder: MutablePersistableRecord, Encodable {
+@Table("reminder")
+struct Reminder {
   …
 }
```

> Note: The `"reminder"` argument is provided to `@Table` due to a naming convention difference
> between SQLiteData and GRDB. More details below.

> Tip: For an incremental migration you can use all 3 of `PersistableRecord`, `FetchableRecord`
> _and_ `@Table`. That will allow you to use the query building tools from both GRDB and SQLiteData
> as you transition.

Once that is done you will be able to make use of the type-safe and schema-safe query building
tools of this library:

```swift
RemindersList
  .group(by: \.id)
  .leftJoin(Reminder.all) { $0.id.eq($1.remindersListID) }
  .select {
    ($0.title, $1.count())
  }
}
```

And you can use the various property wrappers for fetching data from the database in your views
and observable models:

```swift
@Observable
class RemindersModel {
  @ObservationIgnored
  @FetchAll(Reminder.order(by: \.isCompleted)) var reminders
}
```

> Note: Due to the fact that macros and property wrappers do not play nicely together, we are forced
> to use `@ObservationIgnored`. However, [`@FetchAll`](<doc:FetchAll>) handles all of its own
> observation internally and so this does not affect observation.

There are 3 main things to be aware of when applying `@Table` to an existing schema:

  * The `@Table` macro infers the name of the SQL table from the name of the type by lowercasing the
    first letter and attempting to pluralize the type. This differs from GRDB's naming conventions,
    which only lowercases the first letter of the type name. So, you will need to override `@Table`'s
    default behavior by providing a string argument to the macro:

    ```swift
    @Table("reminder")
    struct Reminder {
      // ...
    }
    @Table("remindersList")
    struct RemindersList {
      // ...
    }
    ```

  * If the column names of your SQLite table do not match the name of the fields in your Swift type,
    then you can provide custom names _via_ the `@Column` macro:

    ```swift
    @Table
    struct Reminder {
      let id: UUID
      var title = ""
      @Column("is_completed")
      var isCompleted = false
    }
    ```

  * If your tables use UUID then you will need to add an extra decoration to your Swift data type
    to make it compatible with SQLiteData. This is due to the fact that by default GRDB encodes UUIDs
    as bytes whereas SQLiteData encodes UUIDs as text. To keep this compatibility you will need to use
    `@Column(as:)` on any fields holding UUIDs:

    ```swift
    @Table
    struct Reminder {
      @Column(as: UUID.BytesRepresentation.self)
      let id: UUID
      // ...
    }
    ```

    And if your table has an optional UUID, then you will handle that similarly:

    ```swift
    @Table
    struct ChildReminder {
      @Column(as: UUID?.BytesRepresentation.self)
      let parentID: UUID?
      // ...
    }
    ```

## Non-optional primary keys

Some of your data types may have an optional primary key and a `didInsert` callback for setting the
ID after insert:

```swift
struct Reminder: MutablePersistableRecord, Encodable {
  var id: Int?
  var title = ""
  mutating func didInsert(_ inserted: InsertionSuccess) {
    id = inserted.rowID
  }
}
```

These can be updated to use non-optional types for the primary key, and the field can be bound as
an immutable `let`:

```swift
@Table
struct Reminder {
  let id: Int
  var title = ""
}
```

The `@Table` macro automatically generates a `Draft` type that can be used when you want to be
able to construct a value without the ID specified:

```swift
let draft = Reminder.Draft(title: "Get milk")
```

Then when this draft value is inserted its ID will be determined by the database:

```swift
try Reminder.insert {
  Reminder.Draft(title: "Get milk")
}
.execute(db)
```

You can even use a `RETURNING` clause to grab the ID of the freshly inserted record:

```swift
try Reminder.insert {
  Reminder.Draft(title: "Get milk")
}
.returning(\.id)
.fetchOne(db)
```

## CloudKit synchronization

The library's [CloudKit](<doc:CloudKit>) synchronization tools require that the tables being
synchronized have a primary key, and this is enforced through the `PrimaryKeyedTable` protocol.
The `@Table` macro automatically applies this protocol for you when your type has an `id` field,
but if you use a different name for your primary key you will need to use the `@Column` macro
to specify that:

```swift
@Table struct Reminder {
  @Column(primaryKey: true)
  let identifier: String
  …
}
```

The library further requires your tables use globally unique identifiers (such as UUID) for their
primary keys, and in particular auto-incrementing integer IDs do _not_ work. You will need to
migrate your tables to use UUIDs, see
<doc:CloudKit#Preparing-an-existing-schema-for-synchronization> for more information.

[GRDB]: http://github.com/groue/GRDB.swift

---

### CloudKit

# Getting started with CloudKit

Learn how to seamlessly add CloudKit synchronization to your SQLiteData application.

## Overview

SQLiteData allows you to seamlessly synchronize your SQLite database with CloudKit. After a few
steps to set up your project and a ``SyncEngine``, your database can be automatically synchronized
to CloudKit. However, distributing your app's schema across many devices is an impactful decision
to make, and so an abundance of care must be taken to make sure all devices remain consistent
and capable of communicating with each other. Please read the documentation closely and thoroughly
to make sure you understand how to best prepare your app for cloud synchronization.

  - [Setting up your project](#Setting-up-your-project)
  - [Setting up a SyncEngine](#Setting-up-a-SyncEngine)
  - [Designing your schema with synchronization in mind](#Designing-your-schema-with-synchronization-in-mind)
    - [Globally unique primary keys](#Globally-unique-primary-keys)
    - [Primary keys on every table](#Primary-keys-on-every-table)
    - [Foreign key relationships](#Foreign-key-relationships)
    - [Uniqueness constraints](#Uniqueness-constraints)
    - [Avoid reserved CloudKit keywords](#Avoid-reserved-CloudKit-keywords)
  - [Backwards compatible migrations](#Backwards-compatible-migrations)
    - [Adding tables](#Adding-tables)
    - [Adding columns](#Adding-columns)
    - [Disallowed migrations](#Disallowed-migrations)
  - [Record conflicts](#Record-conflicts)
  - [Sharing records with other iCloud users](#Sharing-records-with-other-iCloud-users)
  - [Assets](#Assets)
  - [Accessing CloudKit metadata](#Accessing-CloudKit-metadata)
  - [Unit testing and Xcode previews](#Unit-testing-and-Xcode-previews)
  - [Preparing an existing schema for synchronization](#Preparing-an-existing-schema-for-synchronization)
  - [Tips and tricks](#Tips-and-tricks)
    - [Updating triggers to be compatible with synchronization](#Updating-triggers-to-be-compatible-with-synchronization)
    - [Developing in the simulator](#Developing-in-the-simulator)

## Setting up your project

The steps to set up your SQLiteData project for CloudKit synchronization are the
[same for setting up][setup-cloudkit-apple] any other kind of project for CloudKit:

  * Follow the [Configuring iCloud services] guide for enabling iCloud entitlements in your project.
  * Follow the [Configuring background execution modes] guide for adding the "Background Modes"
    capability to your project and turning on "Remote notifications".
  * If you want to enable sharing of records with other iCloud users, be sure to add a
    `CKSharingSupported` key to your Info.plist with a value of `true`. This is subtly documented
    in [Apple's documentation for sharing].
  * Once you are ready to deploy your app be sure to read Apple's documentation on
    [Deploying an iCloud Container’s Schema].

With those steps completed, you are ready to configure a ``SyncEngine`` that will facilitate
synchronizing your database to and from CloudKit.

[Deploying an iCloud Container’s Schema]: https://developer.apple.com/documentation/CloudKit/deploying-an-icloud-container-s-schema
[Apple's documentation for sharing]: https://developer.apple.com/documentation/cloudkit/sharing-cloudkit-data-with-other-icloud-users#Create-and-Share-a-Topic
[setup-cloudkit-apple]: https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices#Add-the-iCloud-and-Background-Modes-capabilities
[Configuring iCloud services]: https://developer.apple.com/documentation/Xcode/configuring-icloud-services
[Configuring background execution modes]: https://developer.apple.com/documentation/Xcode/configuring-background-execution-modes

## Setting up a SyncEngine

The foundational tool used to synchronize your SQLite database to CloudKit is a ``SyncEngine``.
This is a wrapper around CloudKit's `CKSyncEngine` and performs all the necessary work to listen
for changes in your database to play them back to CloudKit, and listen for changes in CloudKit to
play them back to SQLite.

Before constructing a ``SyncEngine`` you must have already created and migrated your app's local
SQLite database as detailed in <doc:PreparingDatabase>. Immediately after that is done in the
`prepareDependencies` of the entry point of your app you will override the
``Dependencies/DependencyValues/defaultSyncEngine`` dependency with a sync engine that specifies
the database to synchronize, as well as the tables you want to synchronize:

```swift
@main
struct MyApp: App {
  init() {
    try! prepareDependencies {
      $0.defaultDatabase = try appDatabase()
      $0.defaultSyncEngine = try SyncEngine(
        for: $0.defaultDatabase,
        tables: RemindersList.self, Reminder.self
      )
    }
  }

  // ...
}
```

The `SyncEngine`
[initializer](<doc:SyncEngine/init(for:tables:privateTables:containerIdentifier:defaultZone:startImmediately:delegate:logger:)>)
has more options you may be interested in configuring.

> Important: You must explicitly provide all tables that you want to synchronize. We do this so that
> you can have the option of having some local tables that are not synchronized to CloudKit, such as
> full-text search indices, cached data, etc.

Once this work is done the app should work exactly as it did before, but now any changes made
to the database will be synchronized to CloudKit. You will still interact with your local SQLite
database the same way you always have. You can use ``FetchAll`` to fetch data to be used in a view
or `@Observable` model, and you can use the `defaultDatabase` dependency to write to the database.

There is one additional step you can optionally take if you want to gain access to the underlying
CloudKit metadata that is stored by the library. When constructing the connection to your database
you can use the `prepareDatabase` method on `Configuration` to attach the metadatabase:

```swift
func appDatabase() -> any DatabaseWriter {
  var configuration = Configuration()
  configuration.prepareDatabase { db in
    try db.attachMetadatabase()
    …
  }
}
```

This will allow you to query the ``SyncMetadata`` table, which gives you access to the `CKRecord`
stored for each of your records, as well as the `CKShare` for any shared records.

See the ``GRDB/Database/attachMetadatabase(containerIdentifier:)`` for more information, as well
as <doc:CloudKit#Accessing-CloudKit-metadata> below.

## Designing your schema with synchronization in mind

Distributing your app's schema across many devices is a big decision to make for your app, and
care must be taken. It is not true that you can simply take any existing schema, add a
``SyncEngine`` to it, and have it magically synchronize data across all devices and across all
versions of your app. There are a number of principles to keep in mind while designing and evolving
your schema to make sure every device can synchronize changes to every other device, no matter the
version.

#### Globally unique primary keys

> TL;DR: Primary keys must be globally unique identifiers, such as UUID, and cannot be an
> autoincrementing integer. Further, a `NOT NULL` constraint must be specified with an
> `ON CONFLICT REPLACE` action.

Primary keys are an important concept in SQL schema design, and SQLite makes it easy to add a
primary key by using an `AUTOINCREMENT` integer. This makes it so that newly inserted rows get
a unique ID by simply adding 1 to the largest ID in the table. However, that does not play nicely
with distributed schemas. That would make it possible for two devices to create a record with
`id: 1`, and when those records synchronize there would be an irreconcilable conflict.

For this reason, primary keys in SQLite tables should be _globally_ unique, such as a UUID. The
easiest way to do this is to store your table's ID in a `TEXT` column, adding a
default with a freshly generated UUID, and further adding a `ON CONFLICT REPLACE` constraint:

```sql
CREATE TABLE "reminders" (
  "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
  …
)
```

> Tip: The `ON CONFLICT REPLACE` clause must be placed directly after `NOT NULL`.

This allows you to insert a row with a NULL value for the primary key and SQLite will compute
the primary key from the default value specified. This kind of pattern is commonly used with the
`Draft` type generated for primary keyed tables:

```swift
try database.write { db in
  try Reminder.upsert {
    // ℹ️ Omitting 'id' allows the database to initialize it for you.
    Reminder.Draft(title: "Get milk")
  }
  .execute(db)
}
```

If you would like to use a unique identifier other than the `UUID` provided by Foundation, you can
conform your identifier type to ``IdentifierStringConvertible``. We still recommend using
`NOT NULL ON CONFLICT REPLACE` on your column, as well as a default, but the default will need
to be provided outside of SQLite. You can do this by registering a function in SQLite and calling
out to it for the default value of your column:

```sql
CREATE TABLE "reminders" (
  "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (customUUIDv7()),
  …
)
```

Registering custom database functions for ID generation also makes it possible to generate
deterministic IDs for tests, making it easier to test your queries.

> Important: The primary key of a row is encoded into the `recordName` of a `CKRecord`, along with
> the table name. There are [restrictions][CKRecord.ID] on the value of `recordName`:
>
> * It may only contain ASCII characters
> * It must be less than 255 characters
> * It must not begin with an underscore
>
> If your primary key violates any of these rules, a `DatabaseError` will be thrown with a message
> of ``SyncEngine/invalidRecordNameError``.

[CKRecord.ID]: https://developer.apple.com/documentation/cloudkit/ckrecord/id

#### Primary keys on every table

> TL;DR: Each synchronized table must have a single, non-compound primary key to aid in
> synchronization, even if it is not used by your app.

_Every_ table being synchronized must have a single primary key and cannot have compound primary
keys. This includes join tables that typically only have two foreign keys pointing to the two
tables they are joining. For example, a `ReminderTag` table that joins reminders to tags should be
designed like so:

```sql
CREATE TABLE "reminderTags" (
  "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
  "reminderID" TEXT NOT NULL REFERENCES "reminders"("id") ON DELETE CASCADE,
  "tagID" TEXT NOT NULL REFERENCES "tags"("id") ON DELETE CASCADE
) STRICT
```

Note that the `id` column might not be needed for your application's logic, but it is necessary to
facilitate synchronizing to CloudKit.

#### Foreign key relationships

> TL;DR: Foreign key constraints can be enabled and you can use `ON DELETE` actions to
> cascade deletions.

Foreign keys are a SQL feature that allow one to express relationships between tables. This library
uses that information to correctly implement synchronization behavior, such as knowing what order
to syncrhonize records (parent first, then children), and knowing what associated records to
share when sharing a root record.

To express a foreign key relationship between tables you use the `REFERENCES` clause in the table's
schema, along with optional `ON DELETE` and `ON UPDATE` qualifiers:

```sql
CREATE TABLE "reminders"(
  "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
  "title" TEXT NOT NULL DEFAULT '',
  "remindersListID" TEXT NOT NULL REFERENCES "remindersLists"("id") ON DELETE CASCADE
) STRICT
```

> Tip: See SQLite's documentation on [foreign keys](https://sqlite.org/foreignkeys.html) for more information.

SQLiteData can synchronize many-to-one and many-to-many relationships to CloudKit,
and you can enforce foreign key constraints in your database connection. While it is possible for
the sync engine to receive records in an order that could cause a foreign key constraint failure,
such as receiving a child record before its parent, the sync engine will cache the child record
until the parent record has been synchronized, at which point the child record will also be
synchronized.

Currently the only actions supported for `ON DELETE` are `CASCADE`, `SET NULL` and `SET DEFAULT`.
In particular, `RESTRICT` and `NO ACTION` are not supported, and if you try to use those actions
in your schema an error will be thrown when constructing ``SyncEngine``.

#### Uniqueness constraints

> TL;DR: SQLite tables cannot have `UNIQUE` constraints on their columns in order to allow
> for distributed creation of records.

Tables with unique constraints on their columns, other than on the primary key, cannot be
synchronized. As an example, suppose you have a `Tag` table with a unique constraint on the
`title` column. It is not clear how the application should handle if two different devices create
a tag with the title "Family" at the same time. When the two devices synchronize their data
they will have a conflict on the uniqueness constraint, but it would not be correct to
discard one of the tags.

For this reason uniqueness constraints are not allowed in schemas, and this will be validated
when a ``SyncEngine`` is first created. If a uniqueness constraint is detected an error will be
thrown.

Sometimes it is possible to make the column that you want to be unique into the primary key of
your table. For example, if you wanted to associate a `RemindersListAsset` type to a
`RemindersList` type, you can make the primary key of the former also act as the foreign key:

```swift
@Table
struct RemindersListAsset {
  @Column(primaryKey: true)
  let remindersListID: RemindersList.ID
  let image: Data
}
```

This will make it so that at least one asset can be associated with a reminders list.

#### Avoid reserved CloudKit keywords

In the process of sending data from your database to CloudKit, the library turns rows into
`CKRecord`s, which is loosely a `[String: Any]` dictionary. However, certain key names are used
internally by CloudKit and are reserved for their use only. This means those keys cannot be used
as field names in your Swift data types or SQLite tables.

While Apple has not published an exhaustive list of reserved keywords, the following should cover
most known cases:

* `creationDate`
* `creatorUserRecordID`
* `etag`
* `lastModifiedUserRecordID`
* `modificationDate`
* `modifiedByDevice`
* `recordChangeTag`
* `recordID`
* `recordType`

## Backwards compatible migrations

> TL;DR: Database migrations should be done carefully and with full backwards compatibility
> in mind in order to support multiple devices running with different schema versions.

Migrations of a distributed schema come with even more complications than what is mentioned above.
If you ship a 1.0 of your app, and then in 1.1 you add a column to a table, you will need to
contend with the fact that users of the 1.0 will be creating records without that column. This can
cause problems if your migration is not designed correctly.

#### Adding tables

Adding new tables to a schema is perfectly safe thing to do in a CloudKit application. If a record
from a device is synchronized to a device that does not have that table it will cache the record
for later use. Then, when a device updates to the newest version of the app and detects a new table
has been added to the schema, it will populate the table with the cached records it received.

#### Adding columns

> TL;DR: When adding columns to a table that has already been deployed to users' devices, you will
either need to make the column nullable, or a default value must be provided with an
`ON CONFLICT REPLACE` clause.

As an example, suppose the 1.0 of your app shipped a table for a reminders list:

```swift
@Table
struct RemindersList {
  let id: UUID
  var title = ""
}
```

…and you created the SQL table for this like so:

```sql
CREATE TABLE "remindersLists" (
  "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
  "title" TEXT NOT NULL DEFAULT ''
) STRICT
```

Next suppose in 1.1 you want to add a column to the `RemindersList` type:

```diff
 @Table
 struct RemindersList {
   let id: UUID
   var title = ""
+  var position = 0
 }
```

…with the corresponding SQL migration:

```sql
ALTER TABLE "remindersLists"
ADD COLUMN "position" INTEGER NOT NULL DEFAULT 0
```

Unfortunately this schema is problematic for synchronization. When a device running the 1.0 of the
app creates a record, it will not have the `position` field. And when that synchronizes to devices
running the 1.1 of the app, the ``SyncEngine`` will attempt to run a query that is essentially this:

```sql
INSERT INTO "remindersLists"
("id", "title", "position")
VALUES
(NULL, 'Personal', NULL)
```

This will generate a SQL error because the "position" column was declared as `NOT NULL`, and so this
record will not properly synchronize to devices running a newer version of the app.

The fix is to allow for inserting `NULL` values into `NOT NULL` columns by using the default of the
column. This can be done like so:

```sql
ALTER TABLE "remindersLists"
ADD COLUMN "position" INTEGER NOT NULL ON CONFLICT REPLACE DEFAULT 0
```

> Important: The `ON CONFLICT REPLACE` clause must come directly after `NOT NULL` because it
> modifies that constraint.

Now when this query is executed:

```sql
INSERT INTO "remindersLists"
("id", "title", "position")
VALUES
(NULL, 'Personal', NULL)
```

…it will use 0 for the `position` column.

Sometimes it is not possible to specify a default for a newly added column. Suppose in version 1.2
of your app you add groups for reminders lists. This can be expressed as a new field on the
`RemindersList` type:

```diff
 @Table
 struct RemindersList {
   let id: UUID
   var title = ""
   var position = 0
+  var remindersListGroupID: RemindersListGroup.ID
 }
```

However, there is no sensible default that can be used for this schema. But, if you migrate your
table like so:

```sql
ALTER TABLE "remindersLists"
ADD COLUMN "remindersListGroupID" TEXT NOT NULL
REFERENCES "remindersListGroups"("id")
```

…then this will be problematic when older devices create reminders lists with no
`remindersListGroupID`. In this situation you have no choice but to make the field optional in
the type:

```diff
 @Table
 struct RemindersList {
   let id: UUID
   var title = ""
   var position = 0
-  var remindersListGroupID: RemindersListGroup.ID
+  var remindersListGroupID: RemindersListGroup.ID?
 }
```

And your migration will need to add a nullable column to the table:

```diff
 ALTER TABLE "remindersLists"
-ADD COLUMN "remindersListGroupID" TEXT NOT NULL
+ADD COLUMN "remindersListGroupID" TEXT
 REFERENCES "remindersListGroups"("id")
```

It may be disappointing to have to weaken your domain modeling to accommodate synchronization, but
that is the unfortunate reality of a distributed schema. In order to allow multiple versions of your
schema to be run on devices so that each device can create new records and edit existing records
that all devices can see, you will need to make some compromises.

#### Disallowed migrations

Certain kinds of migrations are simply not allowed when synchronizing your schema to multiple
devices. They are:

  * Removing columns
  * Renaming columns
  * Renaming tables

## Record conflicts

> TL;DR: Conflicts are handled automatically using a "last edit wins" strategy for each
> column of the record.

Conflicts between record edits will inevitably happen, and it's just a fact of dealing with
distributed data. The library handles conflicts automatically, but does so with a single strategy
that is currently not customizable. When a column is edited on a record, the library keeps track
of the timestamp for that particular column. When merging two conflicting records, each column
is analyzed, and the column that was most recently edited will win over the older data.

We do not employ more advanced merge conflict strategies, such as CRDT synchronization. We may
allow for these kinds of strategies in the future, but for now "field-wise last edit wins" is
the only strategy available and we feel serves the needs of the most number of people.

## Sharing records with other iCloud users

SQLiteData provides the tools necessary to share a record with another iCloud user so that
multiple users can collaborate on a single record. Sharing a record with another user brings
extra complications to an app that go beyond the existing complications of sharing a schema
across many devices. Please read the documentation carefully and thoroughly to understand
how to best situate your app for sharing that does not cause problems down the road.

See <doc:CloudKitSharing> for more information.

## Assets

> TL;DR: The library packages all `BLOB` columns in a table into `CKAsset`s and seamlessly decodes
> `CKAsset`s back into your tables. We recommend putting large binary blobs of data in their own
> tables.

All BLOB columns in a table are automatically turned into `CKAsset`s and synchronized to CloudKit.
This process is completely seamless and you do not have to take any explicit steps to support
assets.

However, general database design guidelines still apply. In particular, it is not recommended to
store large binary blobs in a table that is queried often. If done naively you may accidentally
load large amounts of data into memory when querying your table. Furthermore, large binary blobs
can slow down SQLite's ability to efficiently access the rows in your tables.

It is recommended to hold binary blobs in a separate, but related, table. For example, if you are
building a reminders app that has lists, and you allow your users to assign an image to a list.
One way to model this is a table for the reminders list data, without the image, and then another
table for the image data associated with a reminders list. Further, the primary key of the cover
image table can be the foreign key pointing to the associated reminders list:

```swift
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var title = ""
}

@Table
struct RemindersListCoverImage {
  @Column(primaryKey: true)
  let remindersListID: RemindersList.ID
  var image: Data
}
/*
CREATE TABLE "remindersListCoverImages" (
  "remindersListID" TEXT PRIMARY KEY NOT NULL REFERENCES "remindersLists"("id") ON DELETE CASCADE,
  "image" BLOB NOT NULL
)
*/
```

This allows you to efficiently query `RemindersList` while still allowing you to load the image
data for a list when you need it.

## Accessing CloudKit metadata

While the library tries to make CloudKit synchronization as seamless and hidden as possible,
there are times you will need to access the underlying CloudKit types for your tables and records.
The ``SyncMetadata`` table is the central place where this data is stored, and it is publicly
exposed for you to query it in whichever way you want.

> Important: In order to query the `SyncMetadata` table from your database connection you will need
to attach the metadatabase to your database connection. This can be done with the
``GRDB/Database/attachMetadatabase(containerIdentifier:)`` method defined on `Database`. See
<doc:CloudKit#Setting-up-a-SyncEngine> for more information on how to do this.

With that done you can use the ``StructuredQueriesCore/PrimaryKeyedTable/syncMetadataID`` property
to construct a SQL query for fetching the metadata associated with one of your records.

For example, if you want to retrieve the `CKRecord` that is associated with a particular row in
one of your tables, say a reminder, then you can use ``SyncMetadata/lastKnownServerRecord`` to
retrieve the `CKRecord` and then invoke a CloudKit database function to retrieve all of the details:

```swift
let lastKnownServerRecord = try database.read { db in
  try SyncMetadata
    .find(remindersList.syncMetadataID)
    .select(\.lastKnownServerRecord)
    .fetchOne(db)
    ?? nil
}
guard let lastKnownServerRecord
else { return }

let ckRecord = try await container.privateCloudDatabase
  .record(for: lastKnownServerRecord.recordID)
```

> Important: In the above snippet we are explicitly using `privateCloudDatabase`, but that is
> only appropriate if the user is the owner of the record. If the user is only a participant in
> a shared record, which can be determined from [SyncMetadata.share](<doc:SyncMetadata/share>),
> then you must use `sharedCloudDatabase` to fetch the newest record.

You are free to invoke any CloudKit functions you want with the `CKRecord` retrieved from
``SyncMetadata``. Any changes made directly with CloudKit will be automatically synced to your
SQLite database by the ``SyncEngine``.

It is also possible to fetch the `CKShare` associated with a record if it has been shared, which
will give you access to the most current list of participants and permissions for the shared record:

```swift
let share = try database.read { db in
  try RemindersList
    .find(remindersList.syncMetadataID)
    .select(\.share)
    .fetchOne(db)
}
guard let share
else { return }

let ckRecord = try await container.sharedCloudDatabase
  .record(for: share.recordID)
```

> Important: In the above snippet we are using the `sharedCloudDatabase` and this is always
appropriate to use when fetching the details of a `CKShare` as they are always stored in the
shared database.

It is also possible to join the ``SyncMetadata`` table directly to your tables so that you can
select this additional information on a per-record basis. For example, if you want to select all
reminders lists, along with a boolean that determines if it is shared or not, you can do the
following:

```swift
@Selection struct Row {
  let remindersList: RemindersList
  let isShared: Bool
}

@FetchAll(
  RemindersList
    .leftJoin(SyncMetadata.all) { $0.syncMetadataID.eq($1.id) }
    .select {
      Row.Columns(
        remindersList: $0,
        isShared: $1.isShared ?? false
      )
    }
)
var rows
```

Here we have used the ``StructuredQueriesCore/PrimaryKeyedTableDefinition/syncMetadataID`` helper
that is defined on all primary key tables so that we can join ``SyncMetadata`` to `RemindersList`.

<!--
## How SQLiteData handles distributed schema scenarios

todo: finish
-->

## Unit testing and Xcode previews

It is possible to run your features in tests and previews even when using the ``SyncEngine``. You
will need to prepare it for dependencies exactly as you do in the entry point of your app. This
can lead to some code duplication, and so you may want to extract that work to a mutating
`bootstrapDatabase` method on `DependencyValues` like so:

```swift
extension DependencyValues {
  mutating func bootstrapDatabase() throws {
    defaultDatabase = try Reminders.appDatabase()
    defaultSyncEngine = try SyncEngine(
      for: defaultDatabase,
      tables: RemindersList.self,
      RemindersListAsset.self,
      Reminder.self,
      Tag.self,
      ReminderTag.self
    )
  }
}
```

Then in your app entry point you can use it like so:

```swift
@main
struct MyApp: App {
  init() {
    try! prepareDependencies {
      try! $0.bootstrapDatabase()
    }
  }

  // ...
}
```

In tests you can use it like so:

```swift
@Suite(.dependencies { try! $0.bootstrapDatabase() })
struct MySuite {
  // ...
}
```

And in previews you can use it like so:

```swift
#Preview {
  try! prepareDependencies {
    try! $0.bootstrapDatabase()
  }
  // ...
}
```

> Tip: If you configure your ``SyncEngine`` with a ``SyncEngineDelegate``, you can pass it to the
> bootstrap function for configuration:
>
> ```diff
>  extension DependencyValues {
>    mutating func bootstrapDatabase(
> +    syncEngineDelegate: (any SyncEngineDelegate)? = nil
>    ) throws {
>      defaultDatabase = try Reminders.appDatabase()
>      defaultSyncEngine = try SyncEngine(
>        for: defaultDatabase,
>        tables: // ...
> +      delegate: syncEngineDelegate
>      )
>    }
>  }
> ```

## Preparing an existing schema for synchronization

If you have an existing app deployed to the app store using SQLite, then you may have to perform
a migration on your schema to prepare it for synchronization. The most important requirement
detailed above in <doc:CloudKit#Designing-your-schema-with-synchronization-in-mind> is that
all tables _must_ have a primary key, and all primary keys must be globally unique identifiers
such as UUID, and cannot be simple auto-incrementing integers.

The steps required to perform such a process are quite lengthy (the SQLite docs describe it in
[12 parts]), and those steps are easy to get wrong, which can either result in the migration
failing or your app accidentally corrupting your user's data.

SQLiteData provides a tool called ``SyncEngine/migratePrimaryKeys(_:tables:uuid:)`` that
makes it possible to perform this migration in just 2 steps:

  * Update your Swift data types (then used annotated with `@Table`) to use UUID identifiers instead
  of `Int`, and fix all of the resulting compiler errors in your features.
  * Create a new migration and invoke ``SyncEngine/migratePrimaryKeys(_:tables:uuid:)`` with the
  database handle from your migration and a list of all of your tables:

    ```swift
    try SyncEngine.migratePrimaryKeys(
      db,
      tables: Reminder.self, RemindersList.self, Tag.self
    )
    ```

That will perform the many step process of migrating each table from integer-based primary keys
to UUIDs.

This migration tool tries to be conservative with its efforts so that if it ever detects a
schema it does not know how to handle properly, it will throw an error. If this happens, then
you must migrate your tables manually using the introduces in <doc:ManuallyMigratingPrimaryKeys>.

## Tips and tricks

### Updating triggers to be compatible with synchronization

If you have triggers installed on your tables, then you may want to customize their definitions
to behave differently depending on whether a write is happening to your database from your own
code or from the sync engine. For example, if you have a trigger that refreshes an `updatedAt`
timestamp on a row when it is edited, it would not be appropriate to do that when the sync engine
updates a row from data received from CloudKit. But, if you have a trigger that updates a local
[FTS] index, then you would want to perform that work regardless if your app is updating the data
or CloudKit is updating the data.

[FTS]: https://sqlite.org/fts5.html

To customize this behavior you can use the ``SyncEngine/isSynchronizingChanges()`` SQL expression.
It represents a custom database function that is installed in your database connection, and it will
return true if the write to your database originates from the sync engine. You can use it in a
trigger like so:

```swift
#sql(
  """
  CREATE TEMPORARY TRIGGER "…"
  AFTER DELETE ON "…"
  FOR EACH ROW WHEN NOT \(SyncEngine.isSynchronizingChanges())
  BEGIN
    …
  END
  """
)
```

Or if you are using the trigger building tools from [StructuredQueries] you can use it like so:

[StructuredQueries]: https://github.com/pointfreeco/swift-structured-queries

```swift
Model.createTemporaryTrigger(
  after: .insert { new in
    // ...
  } when: { _ in
    !SyncEngine.isSynchronizingChanges()
  }
)
```

This will skip the trigger's action when the row is being updated due to data being synchronized
from CloudKit.

### Developing in the simulator

It is possible to develop your app with CloudKit synchronization using the iOS simulator, but
you must be aware that simulators do not support push notifications, and so changes do not
synchronize from CloudKit to simulator automatically. Sometimes you can simply close and re-open
the app to have the simulator sync with CloudKit, but the most certain way to force synchronization
is to kill the app and relaunch it fresh.

---

### CloudKitSharing

# Sharing data with other iCloud users

Learn how to allow your users to share certain records with other iCloud users for collaboration.

## Overview

SQLiteData provides the tools necessary to share a record with another iCloud user so that multiple
users can collaborate on a single record. Sharing a record with another user brings extra
complications to an app that go beyond the existing complications of sharing a schema across many
devices. Please read the documentation carefully and thoroughly to understand how to best design
your schema for sharing that does not cause problems down the road.

> Important: To enable sharing of records be sure to add a `CKSharingSupported` key to your
Info.plist with a value of `true`. This is subtly documented in [Apple's documentation for sharing].

[Apple's documentation for sharing]: https://developer.apple.com/documentation/cloudkit/sharing-cloudkit-data-with-other-icloud-users#Create-and-Share-a-Topic

  - [Creating CKShare records](#Creating-CKShare-records)
  - [Accepting shared records](#Accepting-shared-records)
  - [Diving deeper into sharing](#Diving-deeper-into-sharing)
    - [Sharing root records](#Sharing-root-records)
    - [Sharing foreign key relationships](#Sharing-foreign-key-relationships)
      - [One-to-many relationships](#One-to-many-relationships)
      - [Many-to-many relationships](#Many-to-many-relationships)
      - [One-to-"at most one" relationships](#One-to-at-most-one-relationships)
  - [Sharing permissions](#Sharing-permissions)
  - [Controlling what data is shared](#Controlling-what-data-is-shared)

## Creating CKShare records

To share a record with another user one must first create a `CKShare`. SQLiteData provides the
method ``SyncEngine/share(record:configure:)`` on ``SyncEngine`` for generating a `CKShare` for a
record. Further, the value returned from this method can be stored in a view and be used to drive a
sheet to display a ``CloudSharingView``, which is a wrapper around UIKit's
`UICloudSharingController`.

As an example, a reminders app that wants to allow sharing a reminders list with another user can do
so like this:

```swift
struct RemindersListView: View {
  let remindersList: RemindersList
  @State var sharedRecord: SharedRecord?
  @Dependency(\.defaultSyncEngine) var syncEngine

  var body: some View {
    Form {
      …
    }
    .toolbar {
      Button("Share") {
        Task {
          await withErrorReporting {
            sharedRecord = try await syncEngine.share(record: remindersList) { share in
              share[CKShare.SystemFieldKey.title] = "Join '\(remindersList.title)'!"
            }
          }
        }
      }
    }
    .sheet(item: $sharedRecord) { sharedRecord in
      CloudSharingView(sharedRecord: sharedRecord)
    }
  }
}
```

When the "Share" button is tapped, a ``SharedRecord`` will be generated and stored as local state in
the view. That will cause a ``CloudSharingView`` sheet to be presented where the user can configure
how they want to share the record. A record can be _unshared_ by presenting the same
``CloudSharingView`` to the user so that they can tap the "Stop sharing" button in the UI.

If you would like to provide a custom sharing experience outside of what `UICloudSharingController`
offers, you can find more info in [Apple's documentation].

[Apple's documentation]: https://developer.apple.com/documentation/cloudkit/shared-records

## Accepting shared records

Extra steps must be taken to allow a user to _accept_ a shared record. Once the user taps on the
share link sent to them (whether that is by text, email, etc.), the app will be launched with
special options provided or a special delegate method will be invoked in the app's scene delegate.
You must implement these delegate methods and invoke the ``SyncEngine/acceptShare(metadata:)``
method.

As a simplified example, a `UIWindowSceneDelegate` subclass can implement the delegate method like
so:

```swift
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
  @Dependency(\.defaultSyncEngine) var syncEngine
  var window: UIWindow?

  func windowScene(
    _ windowScene: UIWindowScene,
    userDidAcceptCloudKitShareWith cloudKitShareMetadata: CKShare.Metadata
  ) {
    Task {
      try await syncEngine.acceptShare(metadata: cloudKitShareMetadata)
    }
  }

  func scene(
    _ scene: UIScene,
    willConnectTo session: UISceneSession,
    options connectionOptions: UIScene.ConnectionOptions
  ) {
    guard let cloudKitShareMetadata = connectionOptions.cloudKitShareMetadata
    else {
      return
    }
    Task {
      try await syncEngine.acceptShare(metadata: cloudKitShareMetadata)
    }
  }
}
```

The unstructured task is necessary because the delegate method does not work with an async context,
and the `acceptShare` method is async.

## Diving deeper into sharing

The above gives a broad overview of how one shares a record with a user, and how a user accepts a
shared record. There is, however, a lot more to know about sharing. There are important restrictions
placed on what kind of records you are allowed to share, and what associations of those records are
shared.

In a nutshell, only "root" records can be directly shared, _i.e._ records with no foreign keys.
Further, an association of a root record can only be shared if it has only one foreign key pointing
to the root record. And this last rule applies recursively: a leaf association is shared only if
it has exactly one foreign key pointing to a record that also satisfies this property.

For more in-depth information, keep reading.

### Sharing root records

> Important: It is only possible to share "root" records, _i.e._ records with no foreign keys.

A record can be shared only if it is a "root" record. That means it cannot have any
foreign keys whatsoever. As an example, the following `RemindersList` table is a root record because
it does not have any fields pointing to other tables:

```swift
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var title = ""
}
```

On the other hand, a `Reminder` table with a foreign key pointing to the `RemindersList` is _not_
a root record:

```swift
@Table
struct Reminder: Identifiable {
  let id: UUID
  var title = ""
  var isCompleted = false
  var remindersListID: RemindersList.ID
}
```

Such records cannot be shared because it is not appropriate to also share the parent record (_i.e._
the reminders list).

For example, suppose you have a list named "Personal" with a reminder "Get milk". If you share this
reminder with someone, then it becomes difficult to figure out what to do when they make certain
changes to the reminder:

  * If they decide to reassign the reminder to their personal "Life" list, what should
    happen? Should their "Life" list suddenly be synchronized to your device?
  * Or what if they delete the list? Would you want that to delete your list and all of the
    reminders in the list?

For these reasons, and more, it is not possible to share non-root records, like reminders. Instead,
you can share root records, like reminders lists. If you do invoke
``SyncEngine/share(record:configure:)`` with a non-root record, an error will be thrown.

> Note: A reminder can still be shared as an association to a shared reminders list, as discussed
> [in the next section](<doc:CloudKit#Sharing-foreign-key-relationships>). However, a single
> reminder cannot be shared on its own.

For a more complex example, consider the following diagrammatic schema for a reminders app:

@Image(source: "sync-diagram-root-record.png") {
  The green node represents a "root" record, i.e. a record with no foreign key relationships.
}

In this schema, a `RemindersList` can have many `Reminder`s, can have a `CoverImage`, and a
`Reminder` can have multiple `Tag`s, and vice-versa. The only table in this diagram that constitutes
a "root" is `RemindersList`. It is the only one with no foreign key relationships. None of
`Reminder`, `CoverImage`, `Tag` or `ReminderTag` can be directly shared on their own because they
are not root tables.

### Sharing foreign key relationships

> Important: Foreign key relationships are automatically synchronized, but only if the related
> record has a single foreign key. Records with multiple foreign keys cannot be synchronized.

Relationships between models will automatically be shared when sharing a root record, but with some
limitations. An associated record of a shared record will only be shared if it has exactly one
foreign key pointing to the root shared record, whether directly or indirectly through other records
satisfying this property.

Below we describe some of the most common types of relationships in SQL databases, as well as
which are possible to synchronize, which cannot be synchronized, and which can be adapted to
play nicely with synchronization.

##### One-to-many relationships

One-to-many relationships are the simplest to share with other users. As an example, consider a
`RemindersList` table that can have many `Reminder`s associated with it:

```swift
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var title = ""
}

@Table
struct Reminder: Identifiable {
  let id: UUID
  var title = ""
  var isCompleted = false
  var remindersListID: RemindersList.ID
}
```

Since `RemindersList` is a [root record](#Sharing-root-records) it can be shared, and since
`Reminder` has only one foreign key pointing to `RemindersList`, it too will be shared.

Further, suppose there was a `ChildReminder` table that had a single foreign key pointing to a
`Reminder`:

```swift
@Table
struct ChildReminder: Identifiable {
  let id: UUID
  var title = ""
  var isCompleted = false
  var parentReminderID: Reminders.ID
}
```

This too will be shared because it has one single foreign key pointing to a table that also has one
single foreign key pointing to the root record being shared.

As a more complex example, consider the following diagrammatic schema:

@Image(source: "sync-diagram-one-to-many.png") {
  The green node is a shareable root record, and all blue records are relationships that will also
  be shared when the root is shared.
}

In this schema, a `RemindersList` can have many `Reminder`s and a `CoverImage`, and a `Reminder` can
have many `ChildReminder`s. Sharing a `RemindersList` will share all associated reminders, cover
image, and even child reminders. The child reminders are synchronized because it has a single
foreign key pointing to a table that also has a single foreign key pointing to the root record.

##### Many-to-many relationships

Many-to-many relationships pose a significant problem to sharing and cannot be supported. If a table
has multiple foreign keys, then it will not be shared even if one of those  foreign keys points to
the shared record.

As an example, suppose we had a many-to-many association of a `Tag` table to `Reminder` via a
`ReminderTag` join table:

```swift
@Table
struct Tag: Identifiable {
  let id: UUID
  var title = ""
}
@Table
struct ReminderTag: Identifiable {
  let id: UUID
  var reminderID: Reminder.ID
  var tagID: Tag.ID
}
```

In diagrammatic form, this schema looks like the following:

@Image(source: sync-diagram-many-to-many.png) {
  The green record is a shareable record, the blue record will be shared when the root is shared,
  and the light purple records cannot be shared.
}

The `ReminderTag` records will _not_ be shared because it has two foreign key relationships,
represented by the two arrows leaving the `ReminderTag` node. As a consequence, the `Tag` records
will also not be shared. Sharing these records cannot be done in a consistent and logical manner.

> Note: `CKShare` in CloudKit, which is what our tools are built on, does not support sharing
> many-to-many relationships. This is also how the Reminders app works on Apple's platforms. Sharing
> a list of reminders with another use does not share its tags with that user.

To see why this is an acceptable limitation, suppose you share a "Personal" list with someone, which
holds a "Get milk" reminder, and that reminder has a "weekend" tag associated with it. If the tag
were shared with your friend, then what happens when they delete the tag? Would it be appropriate to
delete that tag from all of your reminders, even the ones that were not shared? For this reason,
and more, records with multiple foreign keys cannot be shared with a record.

If you want to support many tags associated with a single reminder, you will have no choice
but to turn it into a one-to-many relationship so that each tag belongs to exactly one reminder:

```swift
@Table
struct Tag: Identifiable {
  let id: UUID
  var title = ""
  var reminderID: Reminder.ID
}
```

In diagrammatic form this schema now looks like the following:

@Image(source: sync-diagram-many-to-many-refactor.png) {
  The green record is a shareable root record, and the blue records will be shared when the root is
  shared.
}

This kind of relationship will now be synchronized automatically. Sharing a `RemindersList` will
automatically share all of its `Reminder`s, which will subsequently also share all of their
`Tag`s.

But, this does now mean it's possible to have multiple `Tag` rows in the database that have the
same title and thus represent the same tag. You wil have to put extra care in your queries and
application logic to properly aggregate these tags together, but luckily this is something that SQL
excels at.

##### One-to-"at most one" relationships

One-to-"at most one" relationships in SQLite allow you to associate zero or one records with
another record. For an example of this, suppose we wanted to hold onto a cover image for reminders
lists (see <doc:CloudKit#Assets> for more information on synchronizing assets such as images). It
is perfectly fine to hold onto large binary data in SQLite, such as image data, but typically one
should put this data in a separate table.

The way to model this kind of relationship in SQLite is by making a foreign key point from the image
table to the reminders list table, _and_ to make that foreign key the primary key of the table. That
enforces that at most one image is associated with a reminders list.

In diagrammatic form, it looks like this:

![One-to-"at most one" relationship with uniqueness](sync-diagram-one-to-at-most-one-unique.png)
<!--
```mermaid
graph BT
  Reminder ---\>|remindersListID| RemindersList
  CoverImage ---\>|PRIMARY KEY remindersListID| RemindersList
  classDef root color:#000,fill:#4cccff,stroke:#333,stroke-width:2px;
  classDef shared color:#000,fill:#98EFB5,stroke:#333,stroke-width:2px;
  class RemindersList root
  class Reminder,CoverImage shared
```
-->

Here the `CoverImage` table has a foreign key pointing to the root table `RemindersList`, but since
it is also the primary key of the table it enforces that at most one cover image belongs to a list.

## Sharing permissions

CloudKit sharing supports permissions so that you can give read-only or read-write access to the
data you share with other users. These permissions are automatically observed by the library and
enforced when writing to your database. If your application tries to write to a record that it
does not have permission for, a `DatabaseError` will be emitted.

To check for this error you can catch `DatabaseError` and compare its message to
``SyncEngine/writePermissionError``:

```swift
do {
  try await database.write { db in
    Reminder.find(id)
      .update { $0.title = "Personal" }
      .execute(db)
  }
} catch let error as DatabaseError where error.message == SyncEngine.writePermissionError {
  // User does not have permission to write to this record.
}
```

See <doc:CloudKit#Accessing-CloudKit-metadata> for more information on accessing the metadata
associated with your user's data.

Ideally your app would not allow the user to write to records that they do not have permissions for.
To check their permissions for a record, you can join the root record table to ``SyncMetadata`` and
select the ``SyncMetadata/share`` value:

```swift
let share = try await database.read { db in
  SyncMetadata
    .find(remindersList.syncMetadataID)
    .select(\.share)
    .fetchOne(db)
    ?? nil
}
guard
  share?.currentUserParticipant?.permission == .readWrite
    || share?.publicPermission == .readWrite
else {
  // User does not have permissions to write to record.
  return
}
```

This allows you to determine the sharing permissions for a root record.

## Controlling what data is shared

It is possible to specify that certain associations that are shareable not be shared. For example,
suppose that you want reminders lists to be sorted by your user, and so add a `position` column to
the table:

```swift
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var position = 0
  var title = ""
}
```

Sharing this record will mean also sharing the position of the list. That means when one user
reorders their local lists, even ones that are private to them, it will reorder the lists for
everyone shared. This is probably not what you want.

So, private and non-shareable information about this record can be stored in a separate table, and
we can use the trick mentioned in <doc:CloudKitSharing#One-to-at-most-one-relationships> by making
the foreign key of the table also be the table's primary key:

```swift
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var title = ""
}
@Table
struct RemindersListPrivate: Identifiable {
  @Column(primaryKey: true)
  let remindersListID: RemindersList.ID
  var position = 0
}
```

And then when creating the ``SyncEngine`` we can specifically ask it to not share this record when
the reminders list is shared by specifying the `privateTables` argument:

```swift
@main
struct MyApp: App {
  init() {
    try! prepareDependencies {
      $0.defaultDatabase = try appDatabase()
      $0.defaultSyncEngine = try SyncEngine(
        for: $0.defaultDatabase,
        tables: RemindersList.self, Reminder.self,
        privateTables: RemindersListPrivate.self
      )
    }
  }

  …
}
```

This table will still be synchronized across all of a single user's devices, but if that user
shares a list with a friend, it will _not_ share the private table, allowing each user to have
their own personal ordering of lists.

---

### ComparisonWithSwiftData

# Comparison with SwiftData

Learn how SQLiteData compares to SwiftData when solving a variety of problems.

## Overview

The SQLiteData library can replace SwiftData for many kinds of apps, and provide additional
benefits such as direct access to the underlying SQLite schema, and better integration outside of
SwiftUI views (including UIKit, `@Observable` models, _etc._). This article describes how the two
approaches compare in a variety of situations, such as setting up the data store, fetching data,
associations, and more.

  * [Defining your schema](#Defining-your-schema)
  * [Setting up external storage](#Setting-up-external-storage)
  * [Fetching data for a view](#Fetching-data-for-a-view)
  * [Fetching data for an @Observable model](#Fetching-data-for-an-Observable-model)
  * [Dynamic queries](#Dynamic-queries)
  * [Creating, updating and deleting data](#Creating-updating-and-deleting-data)
  * [Associations](#Associations)
  * [Booleans and enums](#Booleans-and-enums)
  * [Migrations](#Migrations)
    * [Lightweight migrations](#Lightweight-migrations)
    * [Manual migrations](#Manual-migrations)
  * [CloudKit](#CloudKit)
  * [Supported Apple platforms](#Supported-Apple-platforms)

### Defining your schema

Both SQLiteData and SwiftData come with tools to expose your data types' fields to the compiler
so that type-safe and schema-safe queries can be written. SQLiteData uses another library of ours
to provide these tools, called [StructuredQueries][sq-gh], and its `@Table` macro works similarly
to SwiftData's `@Model` macro:

[sq-gh]: http://github.com/pointfreeco/swift-structured-queries

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Table
    struct Item {
      let id: UUID
      var title = ""
      var isInStock = true
      var notes = ""
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Model
    class Item {
      var title: String
      var isInStock: Bool
      var notes: String
      init(
        title: String = "",
        isInStock: Bool = true,
        notes: String = ""
      ) {
        self.title = title
        self.isInStock = isInStock
        self.notes = notes
      }
    }
    ```
  }
}

Some key differences:

  * The `@Table` macro works with struct data types, whereas `@Model` only works with classes.
  * Because the `@Model` version of `Item` is a class it is necessary to provide an initializer.
  * The `@Model` version of `Item` does not need an `id` field because SwiftData provides a
    `persistentIdentifier` to each model.

See the [documentation][sq-defining-schema] from StructuredQueries for more information on how
to define your schema.

[sq-defining-schema]: https://swiftpackageindex.com/pointfreeco/swift-structured-queries/main/documentation/structuredqueriescore/definingyourschema

### Setting up external storage

Both SQLiteData and SwiftData require some work to be done at the entry point of the app in order
to set up the external storage system that will be used throughout the app. In SQLiteData we use
the `prepareDependencies` function to set up the default database used, and in SwiftUI you construct
a `ModelContainer` and propagate it through the environment:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @main
    struct MyApp: App {
      init() {
        prepareDependencies {
          // Create/migrate a database
          let db = try! DatabaseQueue(/* ... */)
          $0.defaultDatabase = db
        }
      }
      // ...
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @main
    struct MyApp: App {
      let container = {
        // Create/configure a container
        try! ModelContainer(/* ... */)
      }()

      var body: some Scene {
        WindowGroup {
          ContentView()
            .modelContainer(container)
        }
      }
    }
    ```
  }
}

See <doc:PreparingDatabase> for more advice on the various ways you will want to create and
configure your SQLite database for use with SQLiteData.

### Fetching data for a view

To fetch data from a SQLite database you use the `@FetchAll` property wrapper in SQLiteData,
whereas you use the `@Query` macro with SwiftData:

@Row {
  @Column {
    ```swift
    // SQLiteData
    struct ItemsView: View {
      @FetchAll(Item.order(by: \.title))
      var items

      var body: some View {
        ForEach(items) { item in
          Text(item.name)
        }
      }
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    struct ItemsView: View {
      @Query(sort: \Item.title)
      var items: [Item]

      var body: some View {
        ForEach(items) { item in
          Text(item.name)
        }
      }
    }
    ```
  }
}

The `@FetchAll` property wrapper takes a variety of options and allows you to write queries using a
type-safe and schema-safe builder syntax, or you can write safe SQL strings that are schema-safe and
protect you from SQL injection.

The library also ships a few other property wrappers that have no equivalent in SwiftData. For
example, the [`@FetchOne`](<doc:FetchOne>) property wrapper allows you to query for just a single
value, which can be useful for computing aggregate data:

```swift
@FetchOne(Item.where(\.isInStock).count())
var inStockItemsCount = 0
```

And the [`@Fetch`](<doc:Fetch>) property wrapper allows you to execute multiple queries in a single
database transaction to gather your data into a single data type. SwiftData has no equivalent for
either of these operations. See <doc:Fetching> for more detailed information on how to fetch
data from your database using the tools of this library.

### Fetching data for an @Observable model

There are many reasons one may want to move logic out of the view and into an `@Observable` model,
such as allowing to unit test your feature's logic, and making it possible to deep link in your
app. The `@FetchAll` property warpper, and other [data fetching tools](<doc:Fetching>) work just as
well in an `@Observable` model as they do in a SwiftUI view. The state held in the property wrapper
automatically updates when changes are made to the database.

The `@Query` macro, on the other hand, only works in SwiftUI views. This means if you want to move
some of your feature's logic out of the view and into an `@Observable` model you must recreate
its functionality from scratch:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Observable
    class FeatureModel {
      @ObservationIgnored
      @FetchAll(Item.order(by: \.title)) var items
      // ...
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Observable
    class FeatureModel {
      var modelContext: ModelContext
      var items = [Item]()
      var observer: (any NSObjectProtocol)!

      init(modelContext: ModelContext) {
        self.modelContext = modelContext
        observer = NotificationCenter.default.addObserver(
          forName: ModelContext.willSave,
          object: modelContext,
          queue: nil
        ) { [weak self] _ in
          self?.fetchItems()
        }
        fetchItems()
      }

      deinit {
        NotificationCenter.default.removeObserver(observer)
      }

      func fetchItems() {
        do {
          items = try modelContext.fetch(
            FetchDescriptor<Item>(sortBy: [SortDescriptor(\.title)])
          )
        } catch {
          // Handle error
        }
      }
      // ...
    }
    ```
  }
}

> Note: It is necessary to annotate `@FetchAll` with `@ObservationIgnored` when using the
> `@Observable` macro due to how macros interact with property wrappers. However, `@FetchAll`
> handles its own observation, and so state will still be observed when accessed in a view.

### Dynamic queries

Dynamic queries are important for updating the data fetched from the database based on information
that is not known at compile time. The prototypical example of this is a UI that allows the user to
search for rows in a table:

@Row {
  @Column {
    ```swift
    // SQLiteData
    struct ItemsView: View {
      @State var searchText = ""
      @FetchAll var items: [Item]

      var body: some View {
        ForEach(items) { item in
          Text(item.name)
        }
        .searchable(text: $searchText)
        .task(id: searchText) {
          await updateSearchQuery()
        }
      }

      func updateSearchQuery() {
        await $items.load(
          .fetchAll(
            Item.where {
              $0.title.contains(searchText)
            }
          )
        )
      }
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    struct ItemsView: View {
      @State var searchText = ""

      var body: some View {
        SearchResultsView(
          searchText: searchText
        )
        .searchable(text: $searchText)
      }
    }

    struct SearchResultsView: View {
      @Query var items: [Item]

      init(searchText: String) {
        _items = Query(
          filter: #Predicate<Item> {
            $0.title.contains(searchText)
          }
        )
      }

      var body: some View {
        ForEach(items) { item in
          Text(item.name)
        }
      }
    }
    ```
  }
}

Note that the SwiftData version of this code must have two views. The outer view, `ItemsView`,
holds onto the `searchText` state that the user can change and uses the `searchable` SwiftUI view
modifier. Then, the inner view, `SearchResultsView`, holds onto the `@Query` state so that it can
initialize with a dynamic predicate based on the `searchText`. These two views are necessary
because `@Query` state is not mutable after it is initialized. The only way to change `@Query`
state is if the view holding it is reinitialized, which requires a parent view to recreate the
child view.

On the other hand, the same UI made with `@FetchAll` can all happen in a single view. We can
hold onto the `searchText` state that the user edits, use the `searchable` view modifier for the
UI, and update the `@FetchAll` query when the `searchText` state changes.

See <doc:DynamicQueries> for more information on how to execute dynamic queries in the library.

### Creating, updating and deleting data

To create, update and delete data from the database you must use the `defaultDatabase` dependency.
This is similar to what one does with SwiftData too, where all changes to the database go through
the `ModelContext` and is not done through the `@Query` macro at all.

For example, to get access to `defaultDatabase`, you use the `@Dependency` property wrapper:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Dependency(\.defaultDatabase) var database
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Environment(\.modelContext) var modelContext
    ```
  }
}

Then, to create a new row in a table you use the `write` and `insert` methods from SQLiteData:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Dependency(\.defaultDatabase) var database

    try database.write { db in
      try Item.insert(Item(/* ... */))
        .execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Environment(\.modelContext) var modelContext

    let newItem = Item(/* ... */)
    modelContext.insert(newItem)
    try modelContext.save()
    ```
  }
}

To update an existing row you can use the `write` and `update` methods from SQLiteData:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Dependency(\.defaultDatabase) var database

    existingItem.title = "Computer"
    try database.write { db in
      try Item.update(existingItem).execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Environment(\.modelContext) var modelContext

    existingItem.title = "Computer"
    try modelContext.save()
    ```
  }
}

And to delete an existing row, you can use the `write` and `delete` methods from SQLiteData:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Dependency(\.defaultDatabase) var database

    try database.write { db in
      try Item.delete(existingItem).execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Environment(\.modelContext) var modelContext

    modelContext.delete(existingItem))
    try modelContext.save()
    ```
  }
}

### Associations

The biggest difference between SwiftData and SQLiteData is that SwiftData provides tools for an
Object Relational Mapping (ORM), whereas SQLiteData is largely just a nice API for interacting with SQLite
directly.

For example, SwiftData allows you to model a `Sport` type that belongs to many `Team`s like
so:

```swift
@Model class Sport {
  @Relationship(inverse: \Team.sport)
  var teams = [Team]()
}
@Model class Team {
  var sport: Sport
}
```

The data for `Sport` and `Team` are stored in separate tables of a SQLite database, and if you
fetch a `Sport`, you can immediate access the `teams` property on the sport in order to execute
another query to fetch all of the sport's teams:

```swift
let sport = try modelContext.fetch(FetchDescriptor<Sport>())
for sport in sports {
  print("\(sport) has \(sport.teams.count) teams")
}
```

This is powerful, but it can also lead to a number of problems in apps. First, the only way for this
mechanism to work is for `Team` and `Sport` to be classes, and the `@Model` macro enforces that.
Second, because the SQLite execution is so abstracted from us, it makes it easy to execute many,
_many_ queries, leading to inefficient code. In this case, we are first executing a query to
get all sports, and then executing a query for each sport to get the number of teams in each
sport. And on top of that, we are loading every team into memory just to compute the number of
teams.  We don't actually need any data from the team, only their aggregate count.

SQLiteData does not provide these kinds of tools, and for good reason. Instead, if you know you
want to fetch all of the teams with their corresponding sport, you can simply perform a single
query that joins the two tables together:

```swift
@Selection
struct SportWithTeamCount {
  let sport: Sport
  let teamCount: Int
}

@FetchAll(
  Sport
    .group(by: \.id)
    .leftJoin(Team.all) { $0.id.eq($1.sportID) }
    .select {
      SportWithTeamCount.Columns(sport: $0, teamCount: $1.count())
    }
)
var sportsWithTeamCounts
```

If either of the "sports" or "teams" tables change, this query will be executed again and the
state will update to the freshest values.

This style of handling associations does require you to be knowledgable in SQL to wield it
correctly, but that is a benefit! SQL (and SQLite) are some of the most proven pieces of
technologies in the history of computers, and knowing how to wield their powers is a huge benefit.

### Booleans and enums

While it may be hard to believe at first, SwiftData does not fully support boolean or enum values
for fields of a model. Take for example this following model:

```swift
@Model
class Reminder {
  var isCompleted = false
  var priority: Priority?
  init(isCompleted: Bool = false, priority: Priority? = nil) {
    self.isCompleted = isCompleted
    self.priority = priority
  }

  enum Priority: Int, Codable {
    case low, medium, high
  }
}
```

This model compiles just fine, but it is very limited in what you can do with it. First, you cannot
sort by the `isCompleted` column when constructing a `@Query` because `Bool` is not `Comparable`:

```swift
@Query(sort: [SortDescriptor(\.isCompleted)])
var reminders: [Reminder]  // 🛑
```

There is no way to sort by boolean columns in SwiftData.

Further, you cannot filter by enum columns, such as selecting only high priority reminders:

```swift
@Query(filter: #Predicate { $0.priority == Priority.high })
var highPriorityReminders: [Reminder]
```

This will compile just fine yet crash at runtime. The only way to make this code work is to greatly
weaken your model by modeling both `isCompleted` _and_ `priority` as integers:

```swift
@Model
class Reminder {
  var isCompleted = 0
  var priority: Int?
  init(isCompleted: Int = 0, priority: Int? = nil) {
    self.isCompleted = isCompleted
    self.priority = priority
  }
}

@Query(
  filter: #Predicate { $0.priority == 2 },
  sort: [SortDescriptor(\.isCompleted)]
)
var highPriorityReminders: [Reminder]
```

This will now work, but of course these fields can now hold over 9 quintillion possible values when
only a few values are valid.

On the other hand, booleans and enums work just fine in SQLiteData:

```swift
@Table
struct Reminder {
  var isCompleted = false
  var priority: Priority?
  enum Priority: Int, QueryBindable {
    case low, medium, high
  }
}

@FetchAll(
  Reminder
    .where { $0.priority == Priority.high }
    .order(by: \.isCompleted)
)
var reminders
```

This compiles and selects all high priority reminders ordered by their `isCompleted` state. You
can even leave off the type annotation for `reminders` because it is inferred from the query.

### Migrations

[grdb-migration-docs]: https://swiftpackageindex.com/groue/grdb.swift/master/documentation/grdb/migrations

[Migrations in GRDB][grdb-migration-docs] and SwiftData are very different. GRDB makes migrations
explicit where you make direct changes to the schemas in your database. This includes creating
tables, adding, removing or altering columns, adding or removing indices, and more.

Whereas SwiftData has two flavors of migrations. The simplest, "lightweight" migrations, work
implicitly by comparing your data types to the database schema and updating the schema accordingly.
That cannot always work, and so there are "manual" migrations where you explicitly describe how
to change the database schema.

#### Lightweight migrations

Lightweight migrations in SwiftData work for simple situations, such as adding a new data type:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Table
    struct Item {
      let id: UUID
      var title = ""
      var isInStock = true
    }

    migrator.registerMigration("Create 'items' table") { db in
      try #sql(
        """
        CREATE TABLE "items" (
          "id" INTEGER PRIMARY KEY AUTOINCREMENT,
          "title" TEXT NOT NULL,
          "isInStock" INTEGER NOT NULL DEFAULT 1
        )
        """
      )
      .execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Model
    class Item {
      var title = ""
      var isInStock = true
    }
    ```
  }
}

Note that in GRDB we must explicitly create the table, specify its columns, as well as its
constraints, such as if it is nullable or has a default value.

Similarly, adding a column to a data type is also a lightweight migration in SwiftData, such as
adding a `description` field to the `Item` type:

@Row {
  @Column {
    ```swift
    @Table
    struct Item {
      let id: UUID
      var title = ""
      var description = ""
      var isInStock = true
    }

    migrator.registerMigration("Add 'description' column to 'items'") { db in
      try #sql(
        """
        ALTER TABLE "items"
        ADD COLUMN "description" TEXT
        """
      )
      .execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Model
    class Item {
      var title = ""
      var description = ""
      var isInStock = true
    }
    ```
  }
}

In each of these cases, the lightweight migration of SwiftData is less code and the actual
migration logic is implicit and hidden away from you.

#### Manual migrations

However, unfortunately, not all migrations can be "lightweight". In fact, from our experience,
real world apps tend to require complex logic when performing most migrations. Something as simple
as changing an optional field to be a non-optional field cannot be done as a lightweight migration
since SwiftData does not know what value to insert into the database for any rows with a NULL
value. Even adding a unique index to a column is not possible because that may introduce constraint
errors if two rows have the same value.

For the times that a lightweight migration is not possible in SwiftData, one must turn to
"manual" migrations via the `VersionedSchema` protocol. As an example, consider adding a unique
index on the "title" column of the "items" table.

In GRDB this is a simple two-step process:

  1. Delete all items that have duplicate titles keeping the first created. Alternatively one could
    rename the titles to
    incorporate a "#" suffix to differentiate between items with the same name.
  1. Add the unique index.

In SwiftData this is a much more involved process since migrations are implicitly tied to the
structure of your data types. The overall steps to follow are as such:

  1. Create a type that conforms to the `VersionedSchema` protocol, which represents the current
     schema of the `Item` model. It is also customary to nest the `Item` model in this type.
  1. Create another type that conforms to the `VersionedSchema` protocol to represents the
     new schema of the `Item` data type.
  1. Duplicate the entire `@Model` data type so that you can specify the unique index. This type
     will need a new name so as to not conflict with the current, and so often it is nested in
     the type created in the previous step.
  1. Because you now have different data types representing `Item` it is customary to add a
     type alias that represents the most "current" version of the `Item`.
  1. Create a type that conforms to the `SchemaMigrationPlan` which allows you to specify the
    "stages" that will be executed when a migration is performed.
  1. Create a `MigrationStage` to implement the logic you want to perform when a migration occurs.
    This is where you will delete the items with duplicate titles, or however you want to handle
    the duplicates.
  1. Provide the migration plan to the `ModelContainer` you create at the entry point of your app.

@Row {
  @Column {
    ```swift
    // SQLiteData
    migrator.registerMigration("Make 'title' unique") { db in
      // 1️⃣ Delete all items that have duplicate title, keeping the first created one:
      try Item
        .delete()
        .where {
          !$0.id.in(
            Item
              .select { $0.id.min() }
              .group(by: \.title)
          )
        }
        .execute()
      // 2️⃣ Create unique index
      try #sql(
        """
        CREATE UNIQUE INDEX
        "items_title" ON "items" ("title")
        """
      )
      .execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    // 1️⃣ Create a type to conform to VersionedSchema and nest current Item inside:
    enum Schema1: VersionedSchema {
      static var versionIdentifier = Schema.Version(1, 0, 0)
      static var models: [any PersistentModel.Type] { [Item.self] }
      @Model
      class Item {
        var title = ""
        var isInStock = true
      }
    }

    // 2️⃣ Create type to conform to VersionedSchema:
    enum Schema2: VersionedSchema {
      static var versionIdentifier = Schema.Version(2, 0, 0)
      static var models: [any PersistentModel.Type] { [Item.self] }

      // 3️⃣ Duplicate Item type for new schema version with unique index:
      @Model
      class Item {
        @Attribute(.unique)
        var title = ""
        var isInStock = true
      }
    }

    // 4️⃣ Create a type alias for the newest Item schema:
    typealias Item = Schema2.Item

    // 5️⃣ Create a type to conform to the SchemaMigrationPlan protocol:
    enum MigrationPlan: SchemaMigrationPlan {
      static var schemas: [any VersionedSchema.Type] {
        [
          Schema1.self,
          Schema2.self
        ]
      }

      // 6️⃣ Create MigrationStage values to implement the logic for migration from one schema
      //    to the next:
      static var stages: [MigrationStage] {
        [
          MigrationStage.custom(
            fromVersion: Schema1.self,
            toVersion: Schema2.self
          ) { context in
            // Delete items with duplicate titles, keeping the first created.
            // Fetch all items but hydrating only their titles:
            var fetchDescriptor = FetchDescriptor<Item>()
            fetchDescriptor.propertiesToFetch = [\.title]
            let items = try context.fetch(fetchDescriptor)
            // Keep track of unique titles so that we know when to delete an item:
            var uniqueTitles: Set<String> = []
            for item in items {
              if uniqueTitles.contains(item.title) {
                // If title is not unique, delete the item:
                context.delete(item)
              } else {
                // If title is unique, add it to the set so that we know to delete
                // items with this title:
                uniqueTitles.insert(item.title)
              }
            }
            try context.save()
          } didMigrate: { _ in
          }
        ]
      }
    }

    // 7️⃣ Create ModelContainer with migration plan in entry point of app:
    @main
    struct MyApp: App {
      let container: ModelContainer
      init() {
        container = try ModelContainer(
          for: Schema(versionedSchema: Schema2.self),
          migrationPlan: MigrationPlan.self
        )
      }
      // ...
    }
    ```
  }
}

Some things to note about the above comparison:

  * In the SQLite version we can make use of SQL's powerful features for easily deleting all items
    with a duplicate title (keeping the first) by using a subquery.
  * The SwiftData migration is many, many times longer than the equivalent SQLite version involving
    many intricate steps that are hard to remember and easy to get wrong.
  * Because database schemas are tightly coupled to type definitions we have no choice but to
    duplicate our data type so that we can apply the `@Attribute(.unique)` macro.
  * Further, we will need to move all helper methods and computed properties from the previous
    version of the data type to the new version.
  * The work in step #6 that deletes items if they have a duplicate titles is very inefficient, but
    it's not possible to make much more efficient. SwiftData does not provide us with tools to run
    raw SQL on the tables, and so we have no choice but to load all of the items into memory and
    manually check for unique titles. This is memory intensive and CPU intensive work and may
    require extra attention if there are thousands of items in the table. On the other hand, SQLite
    can perform this work efficiently on millions of rows without ever loading a single `Item` into
    memory.

So, while lightweight migrations are one of the "magical" features of SwiftData, we feel that
complex "manual" migrations are common enough that one should optimize for them rather than the
other way around.

### CloudKit

Both SQLiteData and SwiftData support basic synchronization of models to CloudKit so that data
can be made available on all of a user's devices. However, SQLiteData also supports sharing records
with other iCloud users, and it exposes the underlying CloudKit data types (e.g. `CKRecord`) so
that you can interact directly with CloudKit if needed.

Setting up a database and sync engine in SQLiteData isn't much different from setting up a
SwiftData stack with CloudKit. The main difference is that one must explicitly provide the
container identifier in SQLiteData because SwiftData has been privileged in being able to
inspect the Entitlements.plist in order to automatically extract that information:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @main
    struct MyApp: App {
      init() {
        try! prepareDependencies {
          $0.defaultDatabase = try appDatabase()
          $0.defaultSyncEngine = try SyncEngine(
            for: $0.defaultDatabase,
            tables: RemindersList.self, Reminder.self
          )
        }
      }

      …
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @main
    struct MyApp: App {
      let modelContainer: ModelContainer
      init() {
        let schema = Schema([
          Reminder.self,
          RemindersList.self,
        ])
        let modelConfiguration = ModelConfiguration(schema: schema)
        modelContainer = try! ModelContainer(
          for: schema,
          configurations: [modelConfiguration]
        )
      }

      …
    }
    ```
  }
}

Once this initial set up is performed, all insertions, updates and deletions from the database
will be automatically synchronized to CloudKit.

SwiftData also has a few limitations in what features you are allowed to use in your schema:

* Unique constraints are not allowed on columns.
* All properties on a model must be optional or have a default value.
* All relationships must be optional.

SQLiteData has only one of these limitations:

* Unique constraints on columns (except for the primary key) cannot be upheld on a distributed
schema. For example, if you have a `Tag` table with a unique `title` column, then what
are you to do if two different devices create a tag with the title "family" at the same time?
See <doc:CloudKit#Uniqueness-constraints> for more information.
* Columns on freshly created tables do not need to have default values or be nullable. Only
newly added columns to existing tables need to either be nullable or have a default. See
<doc:CloudKit#Adding-columns> for more info.
* Relationships on freshly created do not need to be nullable. Only newly added columns to
existing tables need to be nullable. See <doc:CloudKit#Adding-columns> for more info.

For more information about requirements of your schema in order to use CloudKit synchronization,
see <doc:CloudKit#Designing-your-schema-with-synchronization-in-mind> and
<doc:CloudKit#Backwards-compatible-migrations>, and for more general
information about CloudKit synchronization, see <doc:CloudKit>.

### Supported Apple platforms

SwiftData and the `@Query` macro require iOS 17, macOS 14, tvOS 17, watchOS 10 and higher, and
some newer features require even more recent versions of iOS.

Meanwhile, SQLiteData has a broad set of deployment targets supporting all the way back to iOS 13,
macOS 10.15, tvOS 13, and watchOS 6. This means you can use these tools on essentially any
application today with no restrictions.

---

### DynamicQueries

# Dynamic queries

Learn how to load model data based on information that isn't known at compile time.

## Overview

It is very common for an application to provide a particular "view" into its model data depending on
some user-provided input. For example, you may want to filter a query using the text specified in a
search field, or you may want to provide a variety of sort options for displaying the data in a
particular order.

If you were to fetch all data up front with a static query, and then filter and sort using Swift's
collection algorithms, you would not only load more data into memory than necessary, but you would
also perform work in Swift that is more efficiently done in SQLite.

Take the following example:

```swift
struct ContentView: View {
  @FetchAll var items: [Item]
  @State var filterDate: Date?
  @State var order: SortOrder = .reverse

  var displayedItems: [Item] {
    items
      .filter { $0.timestamp > filterDate ?? .distantPast }
      .sorted {
        order == .forward
          ? $0.timestamp < $1.timestamp
          : $0.timestamp > $1.timestamp
      }
      .prefix(10)
  }

  // ...
}
```

It fetches _all_ items from the backing database into memory and then Swift does the work of
filtering, sorting, and truncating this data before it is displayed to the user. This means if the
table contains thousands, or even hundreds of thousands of rows, every single one will be loaded
into memory and processed, which is incredibly inefficient to do. Worse, this work will be performed
every single time `displayedItems` is evaluated, which will be at least once for each time the
view's body is computed, but could also be more.

This kind of data processing is exactly what SQLite excels at, and so we can offload this work by
modifying the query itself. One can do this with SQLiteData by using the `load` method on
``FetchAll``, ``FetchOne`` or ``Fetch`` in order to load a new key, and hence execute a new query:

```swift
struct ContentView: View {
  @FetchAll var items: [Item]
  @State var filterDate: Date?
  @State var order: SortOrder = .reverse

  var body: some View {
    List {
      // ...
    }
    .task(id: [filter, ordering] as [AnyHashable]) {
      await updateQuery()
    }
  }

  private func updateQuery() async {
    do {
      try await $items.load(
        Items
          .where { $0.timestamp > #bind(filterDate ?? .distantPast) }
          .order {
            if order == .forward {
              $0.timestamp
            } else {
              $0.timestamp.desc()
            }
          }
          .limit(10)
      )
    } catch {
      // Handle error...
    }
  }

  // ...
}
```

> Important: If a parent view refreshes, a dynamically-updated query can be overwritten with the
> initial `@FetchAll`'s value, taken from the parent. To manage the state of this dynamic query
> locally to this view, we use `@State @FetchAll`, instead, and to access the underlying
> `FetchAll` value you can use `wrappedValue`.
>
> This only happens when using `@FetchAll`/`@FetchOne`/`@Fetch` directly in a view, and does not
> affect using these tools elsewhere in your application.

---

### Fetching

# Fetching model data

Learn how to use the `@FetchAll`, `@FetchOne` and `@Fetch` property wrappers for performing
SQL queries to load data from your database.

## Overview

All data fetching happens by using the `@FetchAll`, `@FetchOne`, or `@Fetch` property wrappers.
The primary difference between these choices is whether if you want to fetch a collection of
rows, or fetch a single row (_e.g._, an aggregate computation), or if you want to execute multiple
queries in a single transaction.

  * [`@FetchAll`](#FetchAll)
  * [`@FetchOne`](#FetchOne)
  * [`@Fetch`](#Fetch)

### @FetchAll

The [`@FetchAll`](<doc:FetchAll>) property wrapper allows you to fetch a collection of results from
your database using a SQL query. The query is created using our
[StructuredQueries][structured-queries-gh] library, which can build type-safe queries that safely
and performantly decode into Swift data types.

To get access to these tools you must apply the `@Table` macro to your data type that represents
your table:

```swift
@Table
struct Reminder {
  let id: UUID
  var title = ""
  var dueAt: Date?
  var isCompleted = false
}
```

With that done you can already fetch all records from the `Reminder` table in their default order by
simply doing:

```swift
@FetchAll var reminders: [Reminder]
```

If you want to execute a more complex query, such as one that sorts the results by the reminder's
title, then you can use the various query building APIs on `Reminder`:

```swift
@FetchAll(Reminder.order(by: \.title))
var reminders
```

Or if you want to only select the completed reminders, sorted by their titles in a descending
fashion:

```swift
@FetchAll(
  Reminder.where(\.isCompleted).order { $0.title.desc() }
)
var completedReminders
```

This is only the basics of what you can do with the query building tools of this library. To
learn more, be sure to check out the [documentation][structured-queries-docs] of StructuredQueries.

You can even execute a SQL string to populate the data in your features:

```swift
@FetchAll(#sql("SELECT * FROM reminders where isCompleted ORDER BY title DESC"))
var completedReminders: [Reminder]
```

This uses the `#sql` macro for constructing [safe SQL strings][sq-safe-sql-strings]. You are
automatically protected from SQL injection attacks, and it is even possible to use the static
description of your schema to prevent accidental typos:

```swift
@FetchAll(
  #sql(
    """
    SELECT \(Reminder.columns)
    FROM \(Reminder.self)
    WHERE \(Reminder.isCompleted)
    ORDER BY \(Reminder.title) DESC
    """
  )
)
var completedReminders: [Reminder]
```

These interpolations are completely safe to do because they are statically known at compile time,
and it will minimize your risk for typos. Be sure to read the [documentation][sq-safe-sql-strings]
of StructuredQueries to see more of what `#sql` is capable of.

It is also possible to join tables together and query for multiple pieces of data at once. For
example, suppose we have another table for lists of reminders, and each reminder belongs to
exactly one list:

```swift
@Table
struct Reminder {
  let id: UUID
  var title = ""
  var dueAt: Date?
  var isCompleted = false
  var remindersListID: RemindersList.ID
}
@Table
struct RemindersList: Identifiable {
  let id: UUID
  var title = ""
}
```

And further suppose we have a feature that wants to load the title of every reminder, along with
the title of its associated list. Rather than loading all columns of all rows of both tables, which
is inefficient, we can select just the data we need. First we define a data type to hold just that
data, and decorate it with the `@Selection` macro:

```swift
@Selection
struct Record {
  let reminderTitle: String
  let remindersListTitle: String
}
```

And then we construct a query that joins the `Reminder` table to the `RemindersList` table and
selects the titles from each table:

```swift
@FetchAll(
  Reminder
    .join(RemindersList.all) { $0.remindersListID.eq($1.id) }
    .select {
      Record.Columns(
        reminderTitle: $0.title,
        remindersListTitle: $1.title
      )
    }
)
var records
```

This is a very efficient query that selects only the bare essentials of data that the feature
needs to do its job. This kind of query is a lot more cumbersome to perform in SwiftData because
you must construct a dedicated `FetchDescriptor` value and set its `propertiesToFetch`.

[sq-safe-sql-strings]: https://swiftpackageindex.com/pointfreeco/swift-structured-queries/~/documentation/structuredqueriescore/safesqlstrings
[structured-queries-gh]: https://github.com/pointfreeco/swift-structured-queries
[structured-queries-docs]: https://swiftpackageindex.com/pointfreeco/swift-structured-queries/main/documentation/structuredqueriescore/

### @FetchOne

The [`@FetchOne`](<doc:FetchOne>) property wrapper works similarly to `@FetchAll`, but fetches
only a single record from the database and you must provide a default for when no record is found or
use an optional value. This tool can be handy for computing aggregate data, such as the number of
reminders in the database:

```swift
@FetchOne(Reminder.count())
var remindersCount = 0
```

You can perform any query you want in `@FetchOne`, including "where" clauses:

```swift
@FetchOne(Reminder.where(\.isCompleted).count())
var completedRemindersCount = 0
```

You can use the `#sql` macro with `@FetchOne` to execute a safe SQL string:

```swift
@FetchOne(#sql("SELECT count(*) FROM reminders WHERE isCompleted"))
var completedRemindersCount = 0
```

### @Fetch

It is also possible to execute multiple database queries to fetch data for your features. This can
be useful for performing several queries in a single database transaction:

Each instance of `@FetchAll` and `@FetchOne` executes their queries in a separate transaction and
manage separate observations of the database. So, if we wanted to query for all completed reminders,
along with a total count of reminders (completed and uncompleted), we could do so like this:

```swift
@FetchOne(Reminder.count())
var remindersCount = 0

@FetchAll(Reminder.where(\.isCompleted)))
var completedReminders
```

…this is technically 2 separate database transactions with 2 separate observations.

Often this can be just fine, but if you have multiple queries that tend to change at the same time
(_e.g._, when reminders are created or deleted, `remindersCount` and `completedReminders` will
change at the same time), then you can bundle these two queries into a single transaction.

To do this, one defines a conformance to our ``FetchKeyRequest`` protocol, and in that
conformance one can use the builder tools to query the database:

```swift
struct Reminders: FetchKeyRequest {
  struct Value {
    var completedReminders: [Reminder] = []
    var remindersCount = 0
  }
  func fetch(_ db: Database) throws -> Value {
    try Value(
      completedReminders: Reminder.where(\.isCompleted).fetchAll(db),
      remindersCount: Reminder.fetchCount(db)
    )
  }
}
```

Here we have defined a ``FetchKeyRequest/Value`` type inside the conformance that represents all the
data we want to query for in a single transaction, and then we can construct it and return it from
the ``FetchKeyRequest/fetch(_:)`` method.

With this conformance defined we can use the
[`@Fetch`](<doc:Fetch>) property wrapper to execute the query specified by
the `Reminders` type, and we can access the `completedReminders` and `remindersCount` properties
to get to the queried data:

```swift
@Fetch(Reminders()) var reminders = Reminders.Value()
reminders.completedReminders  // [Reminder(/* ... */), /* ... */]
reminders.remindersCount      // 100
```

> Note: A default must be provided to `@Fetch` since it is querying for a custom data type
> instead of a collection of data.

Typically the conformances to ``FetchKeyRequest`` can even be made private and nested inside
whatever type they are used in, such as SwiftUI view, `@Observable` model, or UIKit view controller.
The only time it needs to be made public is if it's shared amongst many features.

---

### ManuallyMigratingPrimaryKeys

# Manually migrating primary keys

The steps needed to manually migrate your tables so that all tables have a primary key, and so
that all primary keys are UUIDs.

## Overview

If the [manual migration](<doc:SyncEngine/migratePrimaryKeys(_:tables:uuid:)>) tool provided
by this library does not work for you, then you will need to migrate your tables manually.
This consists of converting integer primary keys to UUIDs, and adding a primary key to all tables
that do not have one.

### Convert Int primary keys to UUID

The most important step for migrating an existing SQLite database to be compatible with CloudKit
synchronization is converting any `Int` primary keys in your tables to UUID, or some other
globally unique identifier. This can be done in a new migration that is registered when provisioning
your database, but it does take a few queries to accomplish because SQLite does not support
changing the definition of an existing column.

The steps are roughly: 1) create a table with the new schema, 2) copy data over from old
table to new table and convert integer IDs to UUIDs, 3) drop the old table, and finally 4) rename
the new table to have the same name as the old table.

```swift
migrator.registerMigration("Convert 'remindersLists' table primary key to UUID") { db in
  // Step 1: Create new table with updated schema
  try #sql("""
    CREATE TABLE "new_remindersLists" (
      "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
      -- all other columns from 'remindersLists' table
    ) STRICT
    """)
    .execute(db)

  // Step 2: Copy data from 'remindersLists' to 'new_remindersLists' and convert integer
  // IDs to UUIDs
  try #sql("""
    INSERT INTO "new_remindersLists"
    (
      "id",
      -- all other columns from 'remindersLists' table
    )
    SELECT
      -- This converts integers to UUIDs, e.g. 1 -> 00000000-0000-0000-0000-000000000001
      '00000000-0000-0000-0000-' || printf('%012x', "id"),
      -- all other columns from 'remindersLists' table
    FROM "remindersLists"
    """)
    .execute(db)

  // Step 3: Drop the old 'remindersLists' table
  try #sql("""
    DROP TABLE "remindersLists"
    """)
    .execute(db)

  // Step 4: Rename 'new_remindersLists' to 'remindersLists'
  try #sql("""
    ALTER TABLE "new_remindersLists" RENAME TO "remindersLists"
    """)
    .execute(db)
}
```

This will need to be done for every table that uses an integer for its primary key. Further,
for tables with foreign keys, you will need to adapt step 1 to change the types of those
columns to TEXT and will need to perform the integer-to-UUID conversion for those columns in
step 2:

```swift
migrator.registerMigration("Convert 'reminders' table primary key to UUID") { db in
  // Step 1: Create new table with updated schema
  try #sql("""
    CREATE TABLE "new_reminders" (
      "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
      "remindersListID" TEXT NOT NULL REFERENCES "remindersLists"("id") ON DELETE CASCADE,
      -- all other columns from 'reminders' table
    ) STRICT
    """)
    .execute(db)

  // Step 2: Copy data from 'reminders' to 'new_reminders' and convert integer
  // IDs to UUIDs
  try #sql("""
    INSERT INTO "new_reminders"
    (
      "id",
      "remindersListID",
      -- all other columns from 'reminders' table
    )
    SELECT
      -- This converts integers to UUIDs, e.g. 1 -> 00000000-0000-0000-0000-000000000001
      '00000000-0000-0000-0000-' || printf('%012x', "id"),
      '00000000-0000-0000-0000-' || printf('%012x', "remindersListID"),
      -- all other columns from 'reminders' table
    FROM "remindersLists"
    """)
    .execute(db)

  // Step 3 and 4 are unchanged...
}
```

### Add primary key to all tables

All tables must have a primary key to be synchronized to CloudKit, even typically you would not
add one to the table. For example, a join table that joins reminders to tags:

```swift
@Table
struct ReminderTag {
  let reminderID: Reminder.ID
  let tagID: Tag.ID
}
```

…must be updated to have a primary key:


```diff
 @Table
 struct ReminderTag {
+  let id: UUID
   let reminderID: Reminder.ID
   let tagID: Tag.ID
 }
```

And a migration must be run to add that column to the table. However, you must perform a multi-step
migration similar to what is described above in <doc:CloudKit#Convert-Int-primary-keys-to-UUID>.
You must 1) create a new table with the new primary key column, 2) copy data from the old table
to the new table, 3) delete the old table, and finally 4) rename the new table.

Here is how such a migration can look like for the `ReminderTag` table above:

```swift
migrator.registerMigration("Add primary key to 'reminderTags' table") { db in
  // Step 1: Create new table with updated schema
  try #sql("""
    CREATE TABLE "new_reminderTags" (
      "id" TEXT PRIMARY KEY NOT NULL ON CONFLICT REPLACE DEFAULT (uuid()),
      "reminderID" TEXT NOT NULL REFERENCES "reminders"("id") ON DELETE CASCADE,
      "tagID" TEXT NOT NULL REFERENCES "tags"("id") ON DELETE CASCADE
    ) STRICT
    """)
    .execute(db)

  // Step 2: Copy data from 'reminderTags' to 'new_reminderTags'
  try #sql("""
    INSERT INTO "new_reminderTags"
    ("reminderID", "tagID")
    SELECT "reminderID", "tagID"
    FROM "reminderTags"
    """)
    .execute(db)

  // Step 3: Drop the old 'reminderTags' table
  try #sql("""
    DROP TABLE "reminderTags"
    """)
    .execute(db)

  // Step 4: Rename 'new_reminderTags' to 'reminderTags'
  try #sql("""
    ALTER TABLE "new_reminderTags" RENAME TO "reminderTags"
    """)
    .execute(db)
}
```


---

### MigrationGuides

# Migration guides

Learn how to upgrade your application to the latest version of SQLiteData.

## Overview

SQLiteData is under constant development, and we are always looking for ways to simplify the
library and make it more powerful. As such, we often need to deprecate certain APIs in favor of
newer ones. We recommend people update their code as quickly as possible to the newest APIs, and
these guides contain tips to do so.

> Important: Before following any particular migration guide be sure you have followed all the
> preceding migration guides.

## Topics

- <doc:MigratingTo1.4>

---

### MigratingTo1.4

# Migrating to 1.4

SQLiteData 1.4 introduces a new tool for tying the lifecycle database subscriptions to the
lifecycle of the surrounding async context, but it may incidentally cause "Result of call …
is unused" warnings in your project.

## Overview

The `load` method defined on [`@FetchAll`](<doc:FetchAll>) / [`@FetchOne`](<doc:FetchOne>) /
[`@Fetch`](<doc:Fetch>) all now return a discardable result, ``FetchSubscription``. Awaiting the
``FetchSubscription/task`` of that result ties the lifecycle of the subscription to the database
to the lifecycle of the surrounding async context, which can help views to automatically
unsubscribe from the database when they are not visible.

However, when used with `withErrorReporting` you are likely to get the following warning:

```swift
private func updateQuery() async {
  // ⚠️ Result of call to 'withErrorReporting(_:to:fileID:filePath:line:column:isolation:catching:)' is unused
  await withErrorReporting {
    try await $rows.load(…)
  }
}
```

This is happening because although `load` has a discardable result, Swift does not propagate that
to `withErrorReporting`, and so Swift thinks you have an unused value. To fix you will need to
explicitly ignore the result with `_ = `:

```swift
private func updateQuery() async {
  _ = await withErrorReporting {
    try await $rows.load(…)
  }
}
```

---

### Observing

# Observing changes to model data

Learn how to observe changes to your database in SwiftUI views, UIKit view controllers, and
more.

## Overview

This library can be used to fetch and observe data from a SQLite database in a variety of places
in your application, and is not limited to just SwiftUI views as is the case with the `@Query`
macro from SwiftData.

### SwiftUI

The [`@FetchAll`](<doc:FetchAll>), [`@FetchOne`](<doc:FetchOne>), and [`@Fetch`](<doc:Fetch>)
property wrappers work in SwiftUI views similarly to how the `@Query` macro does from SwiftData.
You simply add a property to the view that is annotated with one of the various ways of
 [querying your database](<doc:Fetching>):

```swift
struct ItemsView: View {
  @FetchAll var items: [Item]

  var body: some View {
    ForEach(items) { item in
      Text(item.name)
    }
  }
}
```

The SwiftUI view will automatically re-render whenever the database changes that causes the
queried data to update.

### @Observable models

SQLiteData's property wrappers also works in `@Observable` models (and `ObservableObject`s for
pre-iOS 17 apps). You can add a property to an `@Observable` class, and its data will automatically
update when the database changes and cause any SwiftUI view using it to re-render:

```swift
@Observable
class ItemsModel {
  @ObservationIgnored
  @FetchAll var items: [Item]
}
struct ItemsView: View {
  let model: ItemsModel

  var body: some View {
    ForEach(model.items) { item in
      Text(item.name)
    }
  }
}
```

> Note: Due to how macros work in Swift, property wrappers must be annotated with
> `@ObservationIgnored`, but this does not affect observation as SQLiteData handles its own
> observation.

### UIKit

It is also possible to use this library's tools in a UIKit view controller. For example, if you
want to use a `UICollectionView` to display a list of items, powered by a diffable data source,
then you can do roughly the following:

```swift
class ItemsViewController: UICollectionViewController {
  @FetchAll var items: [Item]

  override func viewDidLoad() {
    // Set up data source and cell registration...

    // Observe changes to items in order to update data source:
    $items.publisher.sink { items in
      guard let self else { return }
      dataSource.apply(
        NSDiffableDataSourceSnapshot(items: items),
        animatingDifferences: true
      )
    }
    .store(in: &cancellables)
  }
}
```

This uses the `publisher` property that is available on every fetched value to update the collection
view's data source whenever the `items` change.

> Tip: There is an alternative way to observe changes to `items`. If you are already depending on
> our [Swift Navigation][swift-nav-gh] library to make use of powerful navigation APIs for SwiftUI
> and UIKitNavigation, then you can use the [`observe`][observe-docs] tool to update the database
> without using Combine:
>
> ```swift
> override func viewDidLoad() {
>   // Set up data source and cell registration...
>
>   // Observe changes to items in order to update data source:
>   observe { [weak self] in
>     guard let self else { return }
>     dataSource.apply(
>       NSDiffableDataSourceSnapshot(items: items),
>       animatingDifferences: true
>     )
>   }
> }
> ```

[swift-nav-gh]: http://github.com/pointfreeco/swift-navigation
[observe-docs]: https://swiftpackageindex.com/pointfreeco/swift-navigation/main/documentation/swiftnavigation/objectivec/nsobject/observe(_:)-94oxy

---

### PreparingDatabase

# Preparing a SQLite database

Learn how to create, configure and migrate the SQLite database that holds your application’s
data.

## Overview

Before you can use any of the tools of this library you must create and configure the SQLite
database that will be used throughout the app. There are a few steps to getting this right, and
a few optional steps you can perform to make the database you provision work well for testing
and Xcode previews.

* [Step 1: Static database connection](#Step-1-Static-database-connection)
* [Step 2: Create configuration](#Step-2-Create-configuration)
* [Step 3: Create database connection](#Step-3-Create-database-connection)
* [Step 4: Migrate database](#Step-4-Migrate-database)
* [Step 5: Set database connection in entry point](#Step-5-Set-database-connection-in-entry-point)
* [(Optional) Step 6: Set up CloudKit SyncEngine](#Optional-Step-6-Set-up-CloudKit-SyncEngine)

### Step 1: App database connection

We will begin by defining a static `appDatabase` function that returns a connection to a local
database stored on disk. We like to define this at the module level wherever the schema is defined:

```swift
func appDatabase() -> any DatabaseWriter {
  // ...
}
```

> Note: Here we are returning an `any DatabaseWriter`. This will allow us to return either a
> [`DatabaseQueue`][db-q-docs] or [`DatabasePool`][db-pool-docs] from within.

[db-q-docs]: https://swiftpackageindex.com/groue/grdb.swift/master/documentation/grdb/databasequeue
[db-pool-docs]: https://swiftpackageindex.com/groue/grdb.swift/master/documentation/grdb/databasepool

### Step 2: Create configuration

Inside this static variable we can create a [`Configuration`][config-docs] value that is used to
configure the database if there is any custom configuration you want to perform. This is an
optional step:

```diff
 func appDatabase() -> any DatabaseWriter {
+  var configuration = Configuration()
 }
```

One configuration you may want to enable is query tracing in order to log queries that are executed
in your application. This can be handy for tracking down long-running queries, or when more queries
execute than you expect. We also recommend only doing this in debug builds to avoid leaking
sensitive information when the app is running on a user's device, and we further recommend using
OSLog when running your app in the simulator/device and using `Swift.print` in previews:

```diff
 import OSLog
 import SQLiteData

 func appDatabase() -> any DatabaseWriter {
+  @Dependency(\.context) var context
   var configuration = Configuration()
+  #if DEBUG
+    configuration.prepareDatabase { db in
+      db.trace(options: .profile) {
+        if context == .preview {
+          print("\($0.expandedDescription)")
+        } else {
+          logger.debug("\($0.expandedDescription)")
+        }
+      }
+    }
+  #endif
 }

+private let logger = Logger(subsystem: "MyApp", category: "Database")
```

> Note: `expandedDescription` will also print the data bound to the SQL statement, which can include
> sensitive data that you may not want to leak. In this case we feel it is OK because everything
> is surrounded in `#if DEBUG`, but it is something to be careful of in your own apps.

> Tip: `@Dependency(\.context)` comes from the [Swift Dependencies][swift-dependencies-gh] library,
> which SQLiteData uses to share its database connection across fetch keys. It allows you to
> inspect the context your app is running in: live, preview or test.

[swift-dependencies-gh]: https://github.com/pointfreeco/swift-dependencies

For more information on configuring the database connection, see [GRDB's documentation][config-docs]
on the matter.

[config-docs]: https://swiftpackageindex.com/groue/grdb.swift/master/documentation/grdb/configuration
[trace-docs]: https://swiftpackageindex.com/groue/grdb.swift/master/documentation/grdb/database/trace(options:_:)

### Step 3: Create database connection

Once a `Configuration` value is set up we can construct the actual database connection. The simplest
way to do this is to construct the database connection using the ``defaultDatabase(path:configuration:)`` function:

```diff
-func appDatabase() -> any DatabaseWriter {
+func appDatabase() throws -> any DatabaseWriter {
   @Dependency(\.context) var context
   var configuration = Configuration()
   #if DEBUG
     configuration.prepareDatabase { db in
       db.trace(options: .profile) {
         if context == .preview {
           print("\($0.expandedDescription)")
         } else {
           logger.debug("\($0.expandedDescription)")
         }
       }
     }
   #endif
+  let database = try defaultDatabase(configuration: configuration)
+  logger.info("open '\(database.path)'")
+  return database
 }
```

This function provisions a context-dependent database for you, _e.g._ in previews and tests it
will provision unique, temporary databases that won't conflict with your live app's database.

### Step 4: Migrate database

Now that the database connection is created we can migrate the database. GRDB provides all the
tools necessary to perform [database migrations][grdb-migration-docs], but the basics include
creating a `DatabaseMigrator`, registering migrations with it, and then using it to migrate the
database connection:

```diff
 func appDatabase() throws -> any DatabaseWriter {
   @Dependency(\.context) var context
   var configuration = Configuration()
   #if DEBUG
     configuration.prepareDatabase { db in
       db.trace(options: .profile) {
         if context == .preview {
           print("\($0.expandedDescription)")
         } else {
           logger.debug("\($0.expandedDescription)")
         }
       }
     }
   #endif
   let database = try defaultDatabase(configuration: configuration)
   logger.info("open '\(database.path)'")
+  var migrator = DatabaseMigrator()
+  #if DEBUG
+    migrator.eraseDatabaseOnSchemaChange = true
+  #endif
+  migrator.registerMigration("Create tables") { db in
+    // Execute SQL to create tables
+  }
+  try migrator.migrate(database)
   return database
 }
```

As your application evolves you will register more and more migrations with the migrator.

It is up to you how you want to actually execute the SQL that creates your tables. There are
[APIs in the community][grdb-table-definition] for building table definition statements using Swift
code, but we personally feel that it is simpler, more flexible and more powerful to use
[plain SQL strings][table-definition-tools]:

[grdb-table-definition]: https://swiftpackageindex.com/groue/grdb.swift/v7.6.1/documentation/grdb/database/create(table:options:body:)

```swift
migrator.registerMigration("Create tables") { db in
  try #sql("""
    CREATE TABLE "remindersLists"(
      "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
      "title" TEXT NOT NULL
    ) STRICT
    """)
    .execute(db)

  try #sql("""
    CREATE TABLE "reminders"(
      "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
      "isCompleted" INTEGER NOT NULL DEFAULT 0,
      "title" TEXT NOT NULL,
      "remindersListID" INTEGER NOT NULL REFERENCES "remindersLists"("id") ON DELETE CASCADE
    ) STRICT
    """)
    .execute(db)
}
```

It may seem counterintuitive that we recommend using SQL strings for table definitions when so much
of the library provides type-safe and schema-safe tools for executing SQL. But table definition SQL
is fundamentally different from other SQL as it is frozen in time and should never be edited
after it has been deployed to users. Read [this article][table-definition-tools] from our
StructuredQueries library to learn more about this decision.

[table-definition-tools]: https://swiftpackageindex.com/pointfreeco/swift-structured-queries/main/documentation/structuredqueriescore/definingyourschema#Table-definition-tools

That is all it takes to create, configure and migrate a database connection. Here is the code
we have just written in one snippet:

```swift
import OSLog
import SQLiteData

func appDatabase() throws -> any DatabaseWriter {
  @Dependency(\.context) var context
  var configuration = Configuration()
  #if DEBUG
    configuration.prepareDatabase { db in
      db.trace(options: .profile) {
        if context == .preview {
          print("\($0.expandedDescription)")
        } else {
          logger.debug("\($0.expandedDescription)")
        }
      }
    }
  #endif
  let database = try defaultDatabase(configuration: configuration)
  logger.info("open '\(database.path)'")
  var migrator = DatabaseMigrator()
  #if DEBUG
    migrator.eraseDatabaseOnSchemaChange = true
  #endif
  migrator.registerMigration("Create tables") { db in
    // ...
  }
  try migrator.migrate(database)
  return database
}

private let logger = Logger(subsystem: "MyApp", category: "Database")
```

[grdb-migration-docs]: https://swiftpackageindex.com/groue/grdb.swift/master/documentation/grdb/migrations

### Step 5: Set database connection in entry point

Once you have defined your `appDatabase` helper for creating a database connection, you must set
it as the `defaultDatabase` for your app in its entry point. This can be in done in SwiftUI by using
`prepareDependencies` in the `init` of your `App` conformance:

```swift
import SQLiteData
import SwiftUI

@main
struct MyApp: App {
  init() {
    prepareDependencies {
      $0.defaultDatabase = try! appDatabase()
    }
  }
  // ...
}
```

> Important: You can only prepare the default database a single time in the lifetime of your
> application. It is best to do this as early as possible after the app launches.

If using app or scene delegates, then you can prepare the `defaultDatabase` in one of those
conformances:

```swift
import SQLiteData
import UIKit

class AppDelegate: NSObject, UIApplicationDelegate {
  func applicationDidFinishLaunching(_ application: UIApplication) {
    prepareDependencies {
      $0.defaultDatabase = try! appDatabase()
    }
  }
  // ...
}
```

And if using something besides UIKit or SwiftUI, then simply set the `defaultDatabase` as early as
possible in the application's lifecycle.

It is also important to prepare the database in Xcode previews. This can be done like so:

```swift
#Preview {
  let _ = prepareDependencies {
    $0.defaultDatabase = try! appDatabase()
  }
  // ...
}
```

And similarly, in tests, this can be done using the `.dependency` testing trait:

```swift
@Test(.dependency(\.defaultDatabase, try appDatabase())
func feature() {
  // ...
}
```

### (Optional) Step 6: Set up CloudKit SyncEngine

If you plan on synchronizing your local database to CloudKit so that your user's data is available
on all of their devices, there is an additional step you must take. See
<doc:CloudKit> for more information.

---

### Database

# ``GRDB/Database``

## Topics

### Seeding model data

- ``seed(_:)``

### User-defined functions

- ``add(function:)``
- ``remove(function:)``

### Querying CloudKit metadata

- ``attachMetadatabase(containerIdentifier:)``

---

### Fetch

# ``SQLiteData/Fetch``

## Overview

## Topics

### Fetching data

- ``FetchKeyRequest``
- ``init(wrappedValue:_:database:)``
- ``init(wrappedValue:)``
- ``load(_:database:)``

### Accessing state

- ``wrappedValue``
- ``projectedValue``
- ``isLoading``
- ``loadError``

### SwiftUI integration

- ``init(wrappedValue:_:database:animation:)``
- ``load(_:database:animation:)``

### Combine integration

- ``publisher``

### Custom scheduling

- ``init(wrappedValue:_:database:scheduler:)``
- ``load(_:database:scheduler:)``

### Sharing integration

- ``sharedReader``
- ``subscript(dynamicMember:)``

---

### FetchAll

# ``SQLiteData/FetchAll``

## Overview

## Topics

### Fetching data

- ``init(wrappedValue:database:)``
- ``init(wrappedValue:_:database:)``
- ``load(_:database:)``

### Accessing state

- ``wrappedValue``
- ``projectedValue``
- ``isLoading``
- ``loadError``

### SwiftUI integration

- ``init(wrappedValue:database:animation:)``
- ``init(wrappedValue:_:database:animation:)``
- ``load(_:database:animation:)``

### Combine integration

- ``publisher``

### Custom scheduling

- ``init(wrappedValue:database:scheduler:)``
- ``init(wrappedValue:_:database:scheduler:)``
- ``load(_:database:scheduler:)``

### Sharing infrastructure

- ``sharedReader``
- ``subscript(dynamicMember:)``

---

### FetchCursor

# ``StructuredQueriesCore/Statement/fetchCursor(_:)``

## Topics

### Iterating over rows

- ``QueryCursor``

---

### FetchKeyRequest

# ``SQLiteData/FetchKeyRequest``

## Topics

### Error handling

- ``NotFound``

---

### FetchOne

# ``SQLiteData/FetchOne``

## Overview

## Topics

### Fetching data

- ``init(wrappedValue:database:)``
- ``init(wrappedValue:_:database:)``
- ``load(_:database:)``

### Accessing state

- ``wrappedValue``
- ``projectedValue``
- ``isLoading``
- ``loadError``

### SwiftUI integration

- ``init(wrappedValue:database:animation:)``
- ``init(wrappedValue:_:database:animation:)``
- ``load(_:database:animation:)``

### Combine integration

- ``publisher``

### Custom scheduling

- ``init(wrappedValue:database:scheduler:)``
- ``init(wrappedValue:_:database:scheduler:)``
- ``load(_:database:scheduler:)``

### Sharing infrastructure

- ``sharedReader``
- ``subscript(dynamicMember:)``

---

### IdentifierStringConvertible

# ``IdentifierStringConvertible``

## Topics

### Conformances

- ``Swift/String``
- ``Swift/Substring``
- ``Foundation/UUID``
- ``Tagged/Tagged``

---

### SystemFieldsRepresentation

# ``CloudKit/CKRecord/SystemFieldsRepresentation``

---

### SQLiteData

# ``SQLiteData``

A fast, lightweight replacement for SwiftData, powered by SQL and supporting CloudKit
synchronization.

## Overview

SQLiteData is a [fast](#Performance), lightweight replacement for SwiftData, supporting CloudKit
synchronization (and even CloudKit sharing), built on top of the popular
[GRDB](https://github.com/groue/GRDB.swift) library.

@Row {
  @Column {
    ```swift
    // SQLiteData
    @FetchAll
    var items: [Item]

    @Table
    struct Item {
      let id: UUID
      var title = ""
      var isInStock = true
      var notes = ""
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Query
    var items: [Item]

    @Model
    class Item {
      var title: String
      var isInStock: Bool
      var notes: String
      init(
        title: String = "",
        isInStock: Bool = true,
        notes: String = ""
      ) {
        self.title = title
        self.isInStock = isInStock
        self.notes = notes
      }
    }
    ```
  }
}

Both of the above examples fetch items from an external data store using Swift data types, and both
are automatically observed by SwiftUI so that views are recomputed when the external data changes,
but SQLiteData is powered directly by SQLite and is usable from anywhere, including UIKit,
`@Observable` models, and more.

> Note: For more information on SQLiteData's querying capabilities, see <doc:Fetching>.

## Quick start

Before SQLiteData's property wrappers can fetch data from SQLite, you need to provide---at
runtime---the default database it should use. This is typically done as early as possible in your
app's lifetime, like the app entry point in SwiftUI, and is analogous to configuring model storage
in SwiftData:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @main
    struct MyApp: App {
      init() {
        prepareDependencies {
          // Create/migrate a database connection
          let db = try! DatabaseQueue(/* ... */)
          $0.defaultDatabase = db
        }
      }
      // ...
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @main
    struct MyApp: App {
      let container = {
        // Create/configure a container
        try! ModelContainer(/* ... */)
      }()

      var body: some Scene {
        WindowGroup {
          ContentView()
            .modelContainer(container)
        }
      }
    }
    ```
  }
}

> Note: For more information on preparing a SQLite database, see <doc:PreparingDatabase>.

This `defaultDatabase` connection is used implicitly by SQLiteData's property wrappers, like
``FetchAll``, which are similar to SwiftData's `@Query` macro, but more powerful:

@Row {
  @Column {
    ```swift
    @FetchAll
    var items: [Item]

    @FetchAll(Item.order(by: \.title))
    var items

    @FetchAll(Item.where(\.isInStock))
    var items



   @FetchAll(Item.order(by: \.isInStock))
   var items

    @FetchOne(Item.count())
    var itemsCount = 0
    ```
  }
  @Column {
    ```swift
    @Query
    var items: [Item]

    @Query(sort: [SortDescriptor(\.title)])
    var items: [Item]

    @Query(filter: #Predicate<Item> {
      $0.isInStock
    })
    var items: [Item]

    // No @Query equivalent of ordering
    // by boolean column.

    // No @Query equivalent of counting
    // entries in database without loading
    // all entries.
    ```
  }
}

And you can access this database throughout your application in a way similar to how one accesses
a model context, via a property wrapper:

@Row {
  @Column {
    ```swift
    // SQLiteData
    @Dependency(\.defaultDatabase) var database

    try database.write { db in
      try Item.insert { Item(/* ... */) }
      .execute(db)
    }
    ```
  }
  @Column {
    ```swift
    // SwiftData
    @Environment(\.modelContext) var modelContext

    let newItem = Item(/* ... */)
    modelContext.insert(newItem)
    try modelContext.save()
    ```
  }
}

> Important: SQLiteData uses [GRDB](https://github.com/groue/GRDB.swift) under the hood for
> interacting with SQLite, and you will use its tools for creating transactions for writing
> to the database, such as the `database.write` method above.

For more information on how SQLiteData compares to SwiftData, see <doc:ComparisonWithSwiftData>.

Further, if you want to synchronize the local database to CloudKit so that it is available on
all your user's devices, simply configure a `SyncEngine` in the entry point of the app:

```swift
@main
struct MyApp: App {
  init() {
    prepareDependencies {
      $0.defaultDatabase = try! appDatabase()
      $0.defaultSyncEngine = SyncEngine(
        for: $0.defaultDatabase,
        tables: /* ... */
      )
    }
  }
  // ...
}
```

> For more information on synchronizing the database to CloudKit and sharing records with iCloud
> users, see <doc:CloudKit>.

This is all you need to know to get started with SQLiteData, but there's much more to learn. Read
the [articles](#Essentials) below to learn how to best utilize this library.

## Performance

SQLiteData leverages high-performance decoding from
[StructuredQueries](https://github.com/pointfreeco/swift-structured-queries) to turn fetched data
into your Swift domain types, and has a performance profile similar to invoking SQLite's C APIs
directly.

See the following benchmarks against
[Lighter's performance test suite](https://github.com/Lighter-swift/PerformanceTestSuite) for a
taste of how it compares:

```
Orders.fetchAll                           setup    rampup   duration
   SQLite (generated by Enlighter 1.4.10) 0        0.144    7.183
   Lighter (1.4.10)                       0        0.164    8.059
┌──────────────────────────────────────────────────────────────────┐
│  SQLiteData (1.0.0)                     0        0.172    8.511  │
└──────────────────────────────────────────────────────────────────┘
   GRDB (7.4.1, manual decoding)          0        0.376    18.819
   SQLite.swift (0.15.3, manual decoding) 0        0.564    27.994
   SQLite.swift (0.15.3, Codable)         0        0.863    43.261
   GRDB (7.4.1, Codable)                  0.002    1.07     53.326
```

## SQLite knowledge required

SQLite is one of the
[most established and widely distributed](https://www.sqlite.org/mostdeployed.html) pieces of
software in the history of software. Knowledge of SQLite is a great skill for any app developer to
have, and this library does not want to conceal it from you. So, we feel that to best wield this
library you should be familiar with the basics of SQLite, including schema design and normalization,
SQL queries, including joins and aggregates, and performance, including indices.

With some basic knowledge you can apply this library to your database schema in order to query
for data and keep your views up-to-date when data in the database changes, and you can use
[StructuredQueries](https://github.com/pointfreeco/swift-structured-queries) to build queries,
either using its type-safe, discoverable query building APIs, or using its `#sql` macro for writing
safe SQL strings.

Further, this library is built on the popular and battle-tested
[GRDB](https://github.com/groue/GRDB.swift) library for interacting with SQLite, such as executing
queries and observing the database for changes.

## What is StructuredQueries?

[StructuredQueries](https://github.com/pointfreeco/swift-structured-queries) is a library for
building SQL in a safe, expressive, and composable manner, and decoding results with high
performance. Learn more about designing schemas and building queries with the library by seeing its
[documentation](https://swiftpackageindex.com/pointfreeco/swift-structured-queries/~/documentation/structuredqueriescore/).

SQLiteData contains an official StructuredQueries driver that connects it to SQLite _via_ GRDB,
though its query builder and decoder are general purpose tools that can interface with other
databases (MySQL, Postgres, _etc._) and database libraries.

## What is GRDB?

[GRDB](https://github.com/groue/GRDB.swift) is a popular Swift interface to SQLite with a rich
feature set and
[extensive documentation](https://swiftpackageindex.com/groue/GRDB.swift/documentation/grdb). This
library leverages GRDB's observation APIs to keep the `@FetchAll`, `@FetchOne`, and `@Fetch`
property wrappers in sync with the database and update SwiftUI views.

If you're already familiar with SQLite, GRDB provides thin APIs that can be leveraged with raw SQL
in short order. If you're new to SQLite, GRDB offers a great introduction to a highly portable
database engine. We recommend
([as does GRDB](https://github.com/groue/GRDB.swift?tab=readme-ov-file#documentation)) a familiarity
with SQLite to take full advantage of GRDB and SQLiteData.

## Topics

### Essentials

- <doc:Fetching>
- <doc:Observing>
- <doc:PreparingDatabase>
- <doc:DynamicQueries>
- <doc:AddingToGRDB>
- <doc:ComparisonWithSwiftData>

### Database configuration and access

- ``defaultDatabase(path:configuration:)``
- ``GRDB/Database``
- ``Dependencies/DependencyValues/defaultDatabase``

### Querying model data

- ``StructuredQueriesCore/Statement``
- ``StructuredQueriesCore/SelectStatement``
- ``StructuredQueriesCore/Table``
- ``StructuredQueriesCore/PrimaryKeyedTable``
- ``QueryCursor``

### Observing model data

- ``FetchAll``
- ``FetchOne``
- ``Fetch``
- ``FetchSubscription``

### CloudKit synchronization and sharing

- <doc:CloudKit>
- <doc:CloudKitSharing>
- ``SyncEngine``
- ``SyncEngineDelegate``
- ``Dependencies/DependencyValues/defaultSyncEngine``
- ``IdentifierStringConvertible``
- ``SyncMetadata``
- ``StructuredQueriesCore/PrimaryKeyedTableDefinition/syncMetadataID``
- ``StructuredQueriesCore/PrimaryKeyedTableDefinition/hasMetadata(in:)``
- ``SharedRecord``

---

