# Fluent Migrations

Migrations allow you to make organized, testable, and reliable changes to your database's structure--
even while it's in production.

Migrations are often used for preparing a database schema for your models. However, they can also be used to 
make normal queries to your database.

In this guide we will cover creating both types of migrations.

## Create / Delete Schema

Let's take a look at how we can use migrations to prepare a schema supporting database to store a theoretical `Galaxy` model.

```swift
import Fluent<#Database#>

struct Galaxy: <#Database#>Model {
    var id: ID?
    var name: String
}
```

### Automatic

Models provide a shortcut for declaring database migrations. If you conform a type that conforms to [`Model`](#fixme) to migration, Fluent can infer the model's properties and automatically implement the `prepare(...)` and `revert(...)` methods.

```swift
import Fluent<#Database#>

extension Galaxy: <#Database#>Migration { }
```

This method is especially useful for quick prototyping and simple setups. For most other situations you should consider creating a normal, custom migration. 

#### Config

Add this automatic migration to your [`MigrationConfig`](#fixme) using the `add(model:database:)` method. This is done in your [`configure.swift`](../getting-started/structure.md#configureswift) file.

```swift
var migrations = MigrationConfig()
migrations.add(model: Galaxy.self, database: .<#dbid#>)
services.register(migrations)
```

The `add(model:database:)` method will automatically set the model's [`defaultDatabase`](#fixme) property. 

### Custom

We can customize the table created for our model by creating a migration and using the static `create` and `delete` methods on [`Database`](#fixme).

```swift
import Fluent<#Database#>

struct CreateGalaxy: <#Database#>Migration {
    // ... 
}
```

#### Prepare

The most important method in a migration is `prepare(...)`. This is responsible for effecting the migration's changes. For our `CreateGalaxy` migration, we will use our database's static `create` method to create a schema.

```swift
import Fluent<#Database#>

struct CreateGalaxy: <#Database#>Migration {
    // ... 

    static func prepare(on conn: <#Database#>Connection) -> Future<Void> {
        return <#Database#>Database.create(Galaxy.self, on: conn) { builder in
            builder.field(for: \.id, isIdentifier: true)
            builder.field(for: \.name)
        }
    }
}
```

To create a schema, you must pass a model type and connection as the first two parameters. The third parameter is a closure that accepts the [`SchemaBuilder`](#fixme). This builder has convenience methods for declaring fields in the schema.

You can use the `field(for: <#KeyPath#>)` method to quickly create fields for each of your model's properties. Since this method accepts key paths to the model (indicated by `\.`), Fluent can see what type those properties are. For most common types (`String`, `Int`, `Double`, etc) Fluent will automatically be able to determine the best database field type to use.

You can also choose to manually select which database field type to use for a given field.

```swift
try builder.field(for: \.name, type: <#DataType#>)
```

Each database has it's own unique data types, so refer to your database's documentation for more information.

Learn more about creating, updating, and deleting schemas in [Fluent &rarr; Schema Builder](../schema-builder).

#### Revert

Each migration should also include a method for _reverting_ the changes it makes. It is used when you boot your 
app with the `--revert` option. 

For a migration that creates a table in the database, the reversion is quite simple: delete the table.

To implement `revert` for our model, we can use our database's static `delete(...)` method to indicate that we would like to delete the schema a schema.

```swift
import Fluent<#Database#>

struct CreateGalaxy: <#Database#>Migration {
    // ... 
    static func revert(on connection: <#Database#>Connection) -> Future<Void> {
        return <#Database#>Database.delete(Galaxy.self, on: connection)
    }
}
```

To delete a schema, you pass a model type and connection as the two required parameters. That's it.

You can always choose to skip a reversion by simplying returning `conn.future(())`. But note that they are especially useful when testing and debugging your migrations.

#### Config

Add this custom migration to your [`MigrationConfig`](#fixme) using the `add(migration:database:)` method. This is done in your [`configure.swift`](../getting-started/structure.md#configureswift) file.

```swift
var migrations = MigrationConfig()
migrations.add(migration: CreateGalaxy.self, database: .<#dbid#>)
services.register(migrations)
```
Make sure to also set the `defaultDatabase` property on your model when using a custom migration. 

```swift
Galaxy.defaultDatabase = .<#dbid#>
```

## Update Schema

After you deploy your application to production, you may find it necessary to add or remove fields on an existing model. You can achieve this by creating a new migration. 

For this example, let's assume we want to add a new property `mass` to the `Galaxy` model from the previous section.

```swift
import Fluent<#Database#>

struct Galaxy: <#Database#>Model {
    var id: ID?
    var name: String
    var mass: Int
}
```

Since our previous migration created a table with fields for both `id` and `name`, we need to update that table and add a field for `mass`. We can do this by using the static `update` method on [`Database`](#fixme).

```swift
import Fluent<#Database#>

struct AddGalaxyMass: <#Database#>Migration {
    // ... 
}
```

### Prepare

Our prepare method will look very similar to the prepare method for a new table, except it will only contain our newly added field.

```swift
struct AddGalaxyMass: <#Database#>Migration {
    // ... 
    
    static func prepare(on conn: <#Database#>Connection) -> Future<Void> {
        return <#Database#>Database.update(Galaxy.self, on: conn) { builder in
            builder.field(for: \.mass)
        }
    }
}
```

All methods available when creating a schema will be available while updating alongside some new methods for deleting fields. See [`SchemaUpdater`](#fixme) for a list of all available methods.

### Revert

To revert this change, we must delete the `mass` field from the table.

```swift
struct AddGalaxyMass: <#Database#>Migration {
    // ... 

    static func revert(on conn: <#Database#>Connection) -> Future<Void> {
        return <#Database#>Database.update(Galaxy.self, on: conn) { builder in
        builder.deleteField(for: \.mass)
    }
}
```

### Config

Add this migration to your [`MigrationConfig`](#fixme) using the `add(migration:database:)` method. This is done in your [`configure.swift`](../getting-started/structure.md#configureswift) file.

```swift
var migrations = MigrationConfig()
// ...
migrations.add(migration: AddGalaxyMass.self, database: .<#dbid#>)
services.register(migrations)
```

## Data

While migrations are useful for creating and updating schemas in SQL databases, they can also be used for more general purposes in any database. Migrations are passed a connection upon running which can be used to perform arbitrary database queries.

For this example, let's assume we want to do a data cleanup migration on our `Galaxy` model and delete any galaxies with a mass of `0`. 

The first step is to create our new migration type.

```swift
struct GalaxyMassCleanup: <#Database#>Migration {

}
```

### Prepare

In the prepare method of this migration, we will perform a query to delete all galaxies which have a mass equal to `0`.

```swift
struct GalaxyMassCleanup: <#Database#>Migration {
    static func prepare(on conn: <#Database#>Connection) -> Future<Void> {
        return Galaxy.query(on: conn).filter(\.mass == 0).delete()
    }
}
```

### Revert

There is no way to undo this migration since it is destructive. You can omit the `revert(...)` method by returning a pre-completed future.

```swift
struct GalaxyMassCleanup: <#Database#>Migration {
    // ...
    
    static func revert(on conn: <#Database#>Connection) -> Future<Void> {
        return conn.future(())
    }
}
```

### Config

Add this migration to your [`MigrationConfig`](#fixme) using the `add(migration:database:)` method. This is done in your [`configure.swift`](../getting-started/structure.md#configureswift) file.

```swift
var migrations = MigrationConfig()
// ...
migrations.add(migration: GalaxyMassCleanup.self, database: .<#dbid#>)
services.register(migrations)
```
