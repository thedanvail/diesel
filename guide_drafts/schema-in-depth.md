Schema in Depth
===============

In this guide we're going to look at what exactly `infer_schema!`, `infer_table_from_schema!`, and `table!` do. For `table!` we will show a simplified version of the actual code that gets generated, and explain how each piece is relevant to you. If you've ever been confused about what exactly is getting generated, or what `use schema::posts::dsl::*` means, this is the right place to be.

`infer_schema!` is a macro provided by `diesel_infer_schema` when you have enabled the feature for one or more database backends. The macro will establish a database connection at compile time, query for a list of all the tables, and generate `infer_table_from_schema!` for each one. `infer_schema` will skip any table names which start with `__`.

`infer_table_from_schema!` has a similar role. It will establish a database connection at compile time, and query the database for the list of columns for the given table. It will then use this information to generate an invocation of `table!` for you.

You can get the same `table!` invocations that `infer_schema!` would have generated by running `diesel print-schema`. If you do not want to require a database with the proper schema be present to compile your app, `diesel print-schema` is the recommended way to create your schema file. Typically these invocations would be placed in `src/schema.rs`.

`table!` is where the bulk of the code gets generated. If you wanted to, you could see the actual exact code that gets generated by running `cargo rustc -- -Z unstable-options --pretty=expanded`. However, the output will be quite noisy, and there's a lot of code that won't actually be relevant to you. Instead we're going to go step by step through a *simplified* version of this output, which only has the code which you would use directly.

For this example, we'll look at the code generated by this `table!` invocation:

```rust
table! {
    users {
        id -> Integer,
        name -> Text,
        hair_color -> Nullable<Text>,
    }
}
```

If you just want to see the full simplified code and look through it yourself, you will find it at the end of this guide.

The output of `table!` is always a Rust module with the same name. If you are using `infer_schema!` there will actually be one additional level of nesting like this:

```rust
mod infer_users {
    pub mod users {
        /* ... */
    }
}
pub use self::infer_users::users;
```

However, if you're using `table!` directly, you will just have `mod users`. The first and most important part of this module will be the definition of the table itself:

```rust
pub struct table;
```

This is the struct that represents the user's table for the purpose of constructing SQL queries. It's usually referenced in code as `users::table` (or sometimes just `users`, more on that in a bit). Next, we'll see there's a module called `columns`, with one struct per column of the table.

```rust
pub struct id;
pub struct name;
pub struct hair_color;
```

Each of these structs uniquely represents each column of the table for the purpose of constructing SQL queries. Each of these structs will implement a trait called [`Expression`][], which indicates the SQL type of the column.

[`Expression`]: https://docs.diesel.rs/diesel/expression/trait.Expression.html

```rust
impl Expression for id {
    type SqlType = Integer;
}

impl Expression for name {
    type SqlType = Text;
}

impl Expression for hair_color {
    type SqlType = Nullable<Text>;
}
```

The `SqlType` type is at the core of how Diesel ensures that your queries are correct. This type will be used by [`ExpressionMethods`][] to determine what things can and cannot be passed to methods like `eq`. It will also be used by [`Queryable`][] to determine what types can be deserialized when this column appears in the select clause.

[`ExpressionMethods`]: https://docs.diesel.rs/diesel/expression_methods/trait.ExpressionMethods.html
[`Queryable`]: https://docs.diesel.rs/diesel/query_source/trait.Queryable.html

In the columns module you'll also see a special column called `star`. Its definition looks like this:

```rust
pub struct star;

impl Expression for star {
    type SqlType = ();
}
```

The `star` struct represents `users.*` in the query builder. This struct is only intended to be used for generating count queries. It should never be used directly. Diesel loads your data from a query by index, not by name. In order to ensure that we're actually getting the data for the column we think we are, Diesel never uses `*` when we actually want to get the data back out of it. We will instead generate an explicit select clause such as `SELECT users.id, users.name, users.hair_color`.

Everything in the columns module will be re-exported from the parent module. This is why we can reference columns as `users::id`, and not `users::columns::id`.

```rust
pub use self::columns::*;

pub struct table;

pub mod columns {
    /* ... */
}
```

Queries can often get quite verbose when everything has to be prefixed with `users::`. For this reason, Diesel also provides a convenience module called `dsl`.

```rust
pub mod dsl {
    pub use super::columns::{id, name, hair_color};
    pub use super::table as users;
}
```

This module re-exports everything in `columns` module (except for `star`), and also re-exports the table but renamed to the actual name of the table. This means that instead of writing `users::table.filter(users::name.eq("Sean")).filter(users::hair_color.eq("black"))`, we can instead write `users.filter(name.eq("Sean")).filter(hair_color.eq("black"))`. The `dsl` module should only ever be imported for single functions. You should never have `use schema::users::dsl::*;` at the top of a module. Code like `#[derive(Insertable)]` will assume that `users` points to the module, not the table struct.

Since `star` is otherwise inaccessible if you have `use schema::users::dsl::*;`, it is also exposed as an instance method on the table.

```rust
impl table {
    pub fn star(&self) -> star {
        star
    }
}
```

Next, there are several traits that get implemented for `table`. You generally will never interact with these directly, but they are what enable most of the query builder functions found in [the `prelude` module][], as well as use with `insert`, `update`, and `delete`.

[the `prelude` module]: https://docs.diesel.rs/diesel/prelude/index.html

```rust
impl AsQuery for table {
    /* body omitted */
}

impl Table for table {
    /* body omitted */
}

impl IntoUpdateTarget for table {
    /* body omitted */
}
```

Finally, there are a few small type definitions and constants defined to make your life easier.

```rust
pub const all_columns: (id, name, hair_color) = (id, name, hair_color);

pub type SqlType = (Integer, Text, Nullable<Text>);

pub type BoxedQuery<'a, DB, ST = SqlType> = BoxedSelectStatement<'a, ST, table, DB>;
```

`all_columns` is just a tuple of all of the column on the table. It is what is used to generate the `select` statement for a query on this table when you don't specify one explicitly. If you ever want to reference `users::star` for things that aren't count queries, you probably want `users::all_columns` instead.

`SqlType` will be the SQL type of `all_columns`. It's rare to need to reference this directly, but it's less verbose than `<<users::table as Table>::AllColumns as Expression>::SqlType` when you need it.

Finally, there is a helper type for referencing boxed queries built from this table. This means that instead of writing `BoxedSelectStatement<'static, users::SqlType, users::table, Pg>` you can instead write `users::BoxedQuery<'static, Pg>`. You can optionally specify the SQL type as well if the query has a custom select clause.

And that's everything! Here is the full code that was generated for this table:

```rust
pub mod users {
    pub use self::columns::*;

    pub mod dsl {
        pub use super::columns::{id, name, hair_color};
        pub use super::table as users;
    }

    pub const all_columns: (id, name, hair_color) = (id, name, hair_color);

    pub struct table;

    impl table {
        pub fn star(&self) -> star {
            star
        }
    }

    pub type SqlType = (Integer, Text, Nullable<Text>);

    pub type BoxedQuery<'a, DB, ST = SqlType> = BoxedSelectStatement<'a, ST, table, DB>;

    impl AsQuery for table {
        /* body omitted */
    }

    impl Table for table {
        /* body omitted */
    }

    impl IntoUpdateTarget for table {
        /* body omitted */
    }

    pub mod columns {
        pub struct star;

        impl Expression for star {
            type SqlType = ();
        }

        pub struct id;

        impl Expression for id {
            type SqlType = Integer;
        }

        pub struct name;

        impl Expression for name {
            type SqlType = Text;
        }

        pub struct hair_color;

        impl Expression for hair_color {
            type SqlType = Nullable<Text>;
        }
    }
}
```