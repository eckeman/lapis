
## Database Access

Lapis comes with a set of classes and functions for working with
[PostgreSQL][5]. In the future other databases might be directly supported.

### Configuring The Upstream & Location

Every query is performed asynchronously by sending an internal sub-request to a
special location defined in our Nginx configuration. This location communicates
with an upstream, which automatically manages a pool of PostgreSQL database
connections. This is handled by the
[`ngx_postgres`](https://github.com/FRiCKLE/ngx_postgres) module that is
bundled with OpenResty.

First we'll add the upstream to our `nginx.conf`, it's how we specify the
host and authentication of the database. Place the following in the `http`
block:

```nginx
upstream database {
  postgres_server ${{pg POSTGRESQL_URL}};
}
```

In this example the `pg` filter is applied to our `POSTGRESQL_URL`
configuration variable. Let's go ahead and add a value to our `config.moon`

```lua
config("development", {
  postgresql_url = "postgres://pg_user:user_password@127.0.0.1/my_database"
})
```

```moon
config "development", ->
  postgresql_url "postgres://pg_user:user_password@127.0.0.1/my_database"
```

The `pg` filter will convert the PostgreSQL URL to the right format for the
Nginx PostgreSQL module.

> Change `pg_user`, `user_password`, `127.0.0.1` and `my_database` to the
> correct values depending on your requirements.

Lastly, we add the location. Place the following in your `server` block:

```nginx
location = /query {
  internal;
  postgres_pass database;
  postgres_query $echo_request_body;
}
```

> The location must be named `/query` by default. And `postgres_pass` must
> match the name of the upstream. In this example we use `database`.



The `internal` setting is very important. This allows the location to only be
used within the context of a sub-request.

You're now ready to start making queries.

### Making A Query

There are two ways to make queries. The first way is to use the raw query
interface, a collection of functions to help you write SQL.

The second way is to use the `Model` class, a wrapper around a Lua table that
helps you synchronize it with a row in a database table.

Here's a base example using the raw query interface:

```lua
local lapis = require "lapis"
local db = require("lapis.db")

local app = lapis.Application()

app:get("/", function()
  local res = db.query("select * from my_table where id = ?", 10)
  return "ok!"
end)
```

```moon
lapis = require "lapis"
db = require "lapis.db"

class extends lapis.Application
  "/": =>
    res = db.query "select * from my_table where id = ?", 10
    "ok!"
```

By default all queries will log to the Nginx log. You'll be able to see each
query as it happens.

## Query Interface

```lua
local db = require("lapis.db")
```

```moon
db = require "lapis.db"
```

### Functions

The `db` module provides the following functions:

#### `query(query, params...)`

Performs a raw query. Returns the result set if successful, returns `nil` if
failed.

The first argument is the query to perform. If the query contains any `?`s then
they are replaced in the order they appear with the remaining arguments. The
remaining arguments are escaped with `escape_literal` before being
interpolated, making SQL injection impossible.

```lua
local res

res = db.query("SELECT * FROM hello")
res = db.query("UPDATE things SET color = ?", "blue")
res = db.query("INSERT INTO cats (age, name, alive) VALUES (?, ?, ?)", 25, "dogman", true)
```

```moon
res = db.query "SELECT * FROM hello"
res = db.query "UPDATE things SET color = ?", "blue"
res = db.query "INSERT INTO cats (age, name, alive) VALUES (?, ?, ?)", 25, "dogman", true
```

```sql
SELECT * FROM hello
UPDATE things SET color = 'blue'
INSERT INTO cats (age, name, alive) VALUES (25, 'dogman', TRUE)
```

> Due to a limitation in the PostgreSQL Nginx extension, it is not possible to
> get the error message in your code. You can however see the error in the
> logs.

#### `select(query, params...)`

The same as `query` except it appends `"SELECT"` to the front of the query.

```lua
local res = db.select("* from hello where active = ?", db.FALSE)
```

```moon
res = db.select "* from hello where active = ?", db.FALSE
```

```sql
SELECT * from hello where active = FALSE
```

#### `insert(table, values, returning...)`

Inserts a row into `table`. `values` is a Lua table of column names and values.

```lua
db.insert("my_table", {
  age = 10,
  name = "Hello World"
})
```


```moon
db.insert "my_table", {
  age: 10
  name: "Hello World"
}
```

```sql
INSERT INTO "my_table" ("age", "name") VALUES (10, 'Hello World')
```

A list of column names to be returned can be given after the value table:

```lua
local res = db.insert("some_other_table", {
  name = "Hello World"
}, "id")
```

```moon
res = db.insert "some_other_table", {
  name: "Hello World"
}, "id"
```

```sql
INSERT INTO "some_other_table" ("name") VALUES ('Hello World') RETURNING "id"
```

#### `update(table, values, conditions, params...)`

Updates `table` with `values` on all rows that match `conditions`.

```lua
db.update("the_table", {
  name = "Dogbert 2.0",
  active = true
}, {
  id = 100
})

```

```moon
db.update "the_table", {
  name: "Dogbert 2.0"
  active: true
}, {
  id: 100
}
```

```sql
UPDATE "the_table" SET "name" = 'Dogbert 2.0', "active" = TRUE WHERE "id" = 100
```

`conditions` can also be a string, and `params` will be interpolated into it:

```lua
db.update("the_table", {
  count = db.raw("count + 1")
}, "count < ?", 10)
```

```moon
db.update "the_table", {
  count: db.raw"count + 1"
}, "count < ?", 10
```

```sql
UPDATE "the_table" SET "count" = count + 1 WHERE count < 10
```

#### `delete(table, conditions, params...)`

Deletes rows from `table` that match `conditions`.

```lua
db.delete("cats", { name: "Roo"})
```

```moon
db.delete "cats", name: "Roo"
```

```sql
DELETE FROM "cats" WHERE "name" = 'Roo'
```

`conditions` can also be a string

```moon
db.delete("cats", "name = ?", "Gato")
```

```moon
db.delete "cats", "name = ?", "Gato"
```

```sql
DELETE FROM "cats" WHERE name = 'Gato'
```

#### `raw(str)`

Returns a special value that will be inserted verbatim into query without being
escaped:

```lua
db.update("the_table", {
  count = db.raw("count + 1")
})

db.select("* from another_table where x = ?", db.raw("now()"))
```

```moon
db.update "the_table", {
  count: db.raw"count + 1"
}

db.select "* from another_table where x = ?", db.raw"now()"
```

```sql
UPDATE "the_table" SET "count" = count + 1
SELECT * from another_table where x = now()
```

#### `escape_literal(value)`

Escapes a value for use in a query. A value is any type that can be stored in a
column. Numbers, strings, and booleans will be escaped accordingly.

```lua
local escaped = db.escape_literal(value)
local res = db.query("select * from hello where id = " .. escaped")
```

```moon
escaped = db.escape_literal value
res = db.query "select * from hello where id = #{escaped}"
```

`escape_literal` is not appropriate for escaping column or table names. See
`escape_identifier`.

#### `escape_identifier(str)`

Escapes a string for use in a query as an identifier. An identifier is a column
or table name.

```lua
local table_name = db.escape_identifier("table")
local res = db.query("select * from " .. table_name)
```

```moon
table_name = db.escape_identifier "table"
res = db.query "select * from #{table_name}"
```

`escape_identifier` is not appropriate for escaping values. See
`escape_literal` for escaping values.

### Constants

The following constants are also available:

 * `NULL` -- represents `NULL` in SQL
 * `TRUE` -- represents `TRUE` in SQL
 * `FALSE` -- represents `FALSE` in SQL


```lua
db.update("the_table", {
  name = db.NULL
})
```

```moon
db.update "the_table", {
  name: db.NULL
}
```

## Models

Lapis provides a `Model` baseclass for making Lua tables that can be
synchronized with a database row. The class is used to represent a single
database table, an instance of the class is used to represent a single row of
that table.

The most primitive model is a blank model:


```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users")
```

```moon
import Model from require "lapis.db.model"

class Users extends Model
```

The name of the class is used to determine the name of the table. In this case
the class name `Users` represents the table `users`. A class name of
`HelloWorlds` would result in the table name `hello_worlds`. It is customary to
make the class name plural.

If you want to use a different table name you can overwrite the `@table_name`
class method:


```moon
class Users extends Model
  @table_name: => "active_users"
```

### Primary Keys

By default all models have the primary key "id". This can be changed by setting
the `@primary_key` class variable.


```lua
local Users = Model:extend("users", {
  primary_key = "login"
})
```

```moon
class Users extends Model
  @primary_key: "login"
```

If there are multiple primary keys then a array table can be used:

```lua
local Followings = Model:extend("followings", {
  primary_key = { "user_id", "followed_user_id" }
})
```

```moon
class Followings extends Model
  @primary_key: { "user_id", "followed_user_id" }
```

### Finding A Row

For the following examples assume we have the following models:

```lua
local Model = require("lapis.db.model").Model

local Users = Model:extend("users")

local Tags = Model:extend("tags", {
  primary_key = {"user_id", "tag"}
})

```


```moon
import Model from require "lapis.db.model"

class Users extends Model

class Tags extends Model
  @primary_key: {"user_id", "tag"}
```

When you want to find a single row the `find` class method is used. In the
first form it takes a variable number of values, one for each primary key in
the order the primary keys are specified:

```lua
local user = Users:find(23232)
local tag = Tags:find(1234, "programmer")
```

```moon
user = Users\find 23232
tag = Tags\find 1234, "programmer"
```

```sql
SELECT * from "users" where "id" = 23232 limit 1
SELECT * from "tags" where "user_id" = 1234 and "tag" = 'programmer' limit 1
```

`find` returns an instance of the model. In the case of the user, if there was a
`name` column, then we could access the users name with `user.name`.

We can also pass a table as an argument to `find`. The table will be converted to a `WHERE` clause in the query:


```lua
local user = Users:find({ email = "person@example.com"})
```

```moon
user = Users\find email: "person@example.com"
```

```sql
SELECT * from "users" where "email" = 'person@example.com' limit 1
```

### Finding Many Rows

When searching for multiple rows the `select` class method is used. It works
similarly to the `select` function from the raw query interface except you
specify the part of the query after the list of columns to select.

```lua
local tags = Tags:select("where tag = ?", "merchant")
```

```moon
tags = Tags\select "where tag = ?", "merchant"
```

```sql
SELECT * from "tags" where tag = 'merchant'
```

Instead of a single instance, an array table of instances is returned.

If you want to restrict what columns are selected you can pass in a table as
the last argument with the `fields` key set:

```lua
local tags = Tags:select("where tag = ?", "merchant", { fields = "created_at as c" })
```

```moon
tags = Tags\select "where tag = ?", "merchant", fields: "created_at as c"
```

```sql
SELECT created_at as c from "tags" where tag = 'merchant'
```

Alternatively if you want to find many rows by their primary key you can use
the `find_all` method. It takes an array table of primary keys. This method
only works on tables that have singular primary keys.

```lua
local users = Users:find_all({ 1,2,3,4,5 })
```

```moon
users = Users\find_all { 1,2,3,4,5 }
```

```sql
SELECT * from "users" where "id" in (1, 2, 3, 4, 5)
```

### Inserting Rows

The `create` class method is used to create new rows. It takes a table of
column values to create the row with. It returns an instance of the model. The
create query fetches the values of the primary keys and sets them on the
instance using the PostgreSQL `RETURN` statement. This is useful for getting
the value of an auto-incrementing key from the insert statement.

```lua
local user = Users:create({
  login = "superuser",
  password = "1234"
})
```

```moon
user = Users\create {
  login: "superuser"
  password: "1234"
}
```

```sql
INSERT INTO "users" ("password", "login") VALUES ('1234', 'superuser') RETURNING "id"
```

### Updating A Row

Instances of models have the `update` method for updating the row. The values
of the primary keys are used to uniquely identify the row for updating.

The first form of update takes variable arguments. A list of strings that
represent column names to be updated. The values of the columns are taken from
the current values in the instance.

```lua
local user = Users:find(1)
user.login = "uberuser"
user.email = "admin@example.com"
user:update("login", "email")
```

```moon
user = Users\find 1
user.login = "uberuser"
user.email = "admin@example.com"

user\update "login", "email"
```

```sql
UPDATE "users" SET "login" = 'uberuser', "email" = 'admin@example.com' WHERE "id" = 1
```

Alternatively we can pass a table as the first argument of `update`. The keys
of the table are the column names, and the values are the values to update the
columns too. The instance is also updated. We can rewrite the above example as:

```lua
local user = Users:find(1)
user:update({
  login = "uberuser",
  email = "admin@example.com",
})
```

```moon
user = Users\find 1
user\update {
  login: "uberuser"
  email: "admin@example.com"
}
```

```sql
UPDATE "users" SET "login" = 'uberuser', "email" = 'admin@example.com' WHERE "id" = 1
```

> The table argument can also take positional values, which are treated the
> same as the variable argument form.

### Deleting A Row

Just call `delete` on the instance:

```lua
local user = Users:find(1)
user:delete()
```

```moon
user = Users\find 1
user\delete!
```

```sql
DELETE FROM "users" WHERE "id" = 1
```

### Timestamps

Because it's common to store creation and update time models have
support for managing these columns automatically.

When creating your table make sure your table has the following columns:


```sql
CREATE TABLE ... (
  ...
  "created_at" timestamp without time zone NOT NULL,
  "updated_at" timestamp without time zone NOT NULL
)
```

Then define your model with the `@timestamp` class variable set to true:

```lua
local Users = Model:extend("users", {
  timestamp = true
})
```

```moon
class Users extends Model
  @timestamp: true
```

Whenever `create` and `update` are called the appropriate timestamp column will
also be set.


### Preloading Associations

A common pitfall when using active record type systems is triggering many
queries inside of a loop. In order to avoid situations like this you should
load data for as many objects as possible in a single query before looping over
the data.

We'll need some models to demonstrate: (The columns are annotated in a comment
above the model).

```lua
local Model = require("lapis.db.model").Model

-- table with columns: id, name
local Users = Model:extend("users")
local Posts = Model:extend("posts")
```

```moon
import Model from require "lapis.db.model"

-- table with columns: id, name
class Users extends Model

-- table with columns: id, user_id, text_content
class Posts extends Model
```

Given all the posts, we want to find the user for each post. We use the
`include_in` class method to include instances of that model in the array of
models instances passed to it.

```lua
local posts = Posts:select() -- this gets all the posts
Users:include_in(posts, "user_id")

print(posts[1].user.name) -- print the fetched data
```

```moon
posts = Posts\select! -- this gets all the posts

Users\include_in posts, "user_id"

print posts[1].user.name -- print the fetched data
```

```sql
SELECT * from "posts"
SELECT * from "users" where "id" in (1,2,3,4,5,6)
```

Each post instance is mutated to have a `user` property assigned to it with an
instance of the `Users` model. The first argument of `include_in` is the array
table of model instances. The second argument is the column name of the foreign
key found in the array of model instances that maps to the primary key of the
class calling the `include_in`.

The name of the inserted property is derived form the name of the foreign key.
In this case, `user` was derived from the foreign key `user_id`. If we want to
manually specify the name we can do something like this:


```lua
Users:include_in(posts, "user_id", { as: "author" })
```

```moon
Users\include_in posts, "user_id", as: "author"
```

Now all the posts will contain a property named `author` with an instance of
the `Users` model.

Sometimes the relationship is flipped. Instead of the list of model instances
having the foreign key column, the model we want to include might have it. This
is common in one-to-one relationships.

Here's another set of example models:

```lua
local Model = require("lapis.db.model").Model

-- table with columns: id, name
local Users = Model:extend("users")

-- table with columns: user_id, twitter_account, facebook_username
local UserData = Model:extend("user_data")

```

```moon
import Model from require "lapis.db.model"

-- columns: id, name
class Users extends Model

-- columns: user_id, twitter_account, facebook_username
class UserData extends Model
```

Now let's say we have a collection of users and we want to fetch the associated
user data:

```lua
local users = Users:select()
UserData:include_in(users, "user_id", { flip: true })

print(users[1].user_data.twitter_account)
```

```moon
users = Users\select!
UserData\include_in users, "user_id", flip: true

print users[1].user_data.twitter_account
```

```sql
SELECT * from "user_data" where "user_id" in (1,2,3,4,5,6)
```

In this example we set the `flip` option to true in the `include_in` method.
This causes the search to happen against our foreign key, and the ids to be
pulled from the `id` of the array of model instances.

Additionally, the derived property name that is injected into the model
instances is created from the name of the included table. In the example above
the `user_data` property contains the included model instances. (Had it been
plural the table name would have been made singular)

### Constraints

Often before we insert or update a row we want to check that some conditions
are met. In Lapis these are called constraints. For example let's say we have a
user model and users are not allowed to have the name "admin".

We might define it like this:


```moon
import Model from require "lapis.db.models"

class Users extends Model
  @constraints: {
    name: (value) =>
      if value\lower! == "admin"
        "User can not be named admin"
  }


assert Users\create {
  name: "Admin"
}
```

The `@constraints` class variable is a table that maps column name to a
function that should check if the constraint is broken. If anything truthy is
returned from the function then the update/insert fails, and that is returned
as the error message.

In the example above, the call to `assert` will fail with the error `"User can
not be named admin"`.

The constraint check function is passed 4 arguments. The model class, the value
of the column being checked, the name of the column being checked, and lastly
the object being checked. On insertion the object is the table passed to the
create method. On update the object is the instance of the model.

### Pagination

Using the `paginated` method on models we can easily paginate through a query
that might otherwise return many results. The arguments are the same as the
`select` method but instead of the result it returns a special `Paginator`
object.

For example, say we have the following table and model: (For documentation on
creating tables see the [next section](#database-schemas-creating-and-dropping-tables))

```moon
create_table "users", {
  { "id", types.serial }
  { "name", types.varchar }
  { "group_id", types.integer }

  "PRIMARY KEY(id)"
}

class Users extends Model

```

We can create a paginator like so:

```moon
paginated = Users\paginated [[where group_id = ? order by name asc]], 123
```

A paginator can be configured by passing a table as the last argument.
The following options are supported:

`per_page`: sets the number of items per page

```moon
paginated_alt = Users\paginated [[where group_id = ?]], 4, per_page: 100
```

`prepare_results`: a function that is passed the results of `get_page` and
`get_all` for processing before they are returned. This is useful for bundling
preloading information into the paginator. The prepare function takes 1
argument, the results, and it must return the results after they have been
processed:


```moon
preloaded = Posts\paginated [[where category = ?]], "cats", {
  per_page: 10
  prepare_results: (posts) ->
    Users\include_in posts, "user_id"
    posts
}
```

The paginator has the following methods:

#### `get_all()`

Gets all the items that the query can return, is the same as calling the
`select` method directly. Returns an array table of model instances.

```moon
users = paginated\get_all!
```

```sql
SELECT * from "users" where group_id = 123 order by name asc
```

#### `get_page(page_num)`

Gets `page_num`th page, where pages are 1 indexed. The number of items per page
is controlled by the `per_page` option, and defaults to 10. Returns an array
table of model instances.

```moon
page1 = paginated\get_page 1
page6 = paginated\get_page 6
```

```sql
SELECT * from "users" where group_id = 123 order by name asc limit 10 offset 0
SELECT * from "users" where group_id = 123 order by name asc limit 10 offset 50
```

#### `num_pages()`

Returns the total number of pages.

#### `total_items()`

Gets the total number of items that can be returned. The paginator will parse
the query and remove all clauses except for the `WHERE` when issuing a `COUNT`.

```moon
users = paginated\total_items!
```

```sql
SELECT COUNT(*) as c from "users" where group_id = 123
```

## Database Schemas

Lapis comes with a collection of tools for creating your database schema inside
of the `lapis.db.schema` module.

### Creating And Dropping Tables

#### `create_table(table_name, { table_declarations... })`

The first argument to `create_table` is the name of the table and the second
argument is an array table that describes the table.

```moon
db = require "lapis.db"
schema = require "lapis.db.schema"

import create_table, types from schema

create_table "users", {
  {"id", types.serial}
  {"username", types.varchar}

  "PRIMARY KEY (id)"
}
```

This will generate the following SQL:

```sql
CREATE TABLE IF NOT EXISTS "users" (
  "id" serial NOT NULL,
  "username" character varying(255) NOT NULL,
  PRIMARY KEY (id)
);
```

The items in the second argument to `create_table` can either be a table, or a
string. When the value is a table it is treated as a column/type tuple:

    { column_name, column_type }

They are both plain strings. The column name will be escaped automatically.
The column type will be inserted verbatim after it is passed through
`tostring`. `schema.types` has a collection of common types that can be used.
For example, `schema.types.varchar` is evaluates to `character varying(255) NOT
NULL`. See more about types below.

If the value to the second argument is a string then it is inserted directly
into the `CREATE TABLE` statement, that's how we create the primary key above.

#### `drop_table(table_name)`

Drops a table.

```moon
import drop_table from schema

drop_table "users"
```

```sql
DROP TABLE IF EXISTS "users";
```

### Indexes

#### `create_index(table_name, col1, col2..., [options])`

`create_index` is used to add new indexes to a table. The first argument is a
table, the rest of the arguments are the ordered columns that make up the
index. Optionally the last argument can be a Lua table of options.

There are two options `unique: BOOL`, `where: clause_string`.

`create_index` will also check if the index exists before attempting to create
it. If the index exists then nothing will happen.

Here are some example indexes:

```moon
import create_index from schema

create_index "users", "created_at"
create_index "users", "username", unique: true

create_index "posts", "category", "title"
create_index "uploads", "name", where: "not deleted"
```

This will generate the following SQL:

```sql
CREATE INDEX ON "users" (created_at);
CREATE UNIQUE INDEX ON "users" (username);
CREATE INDEX ON "posts" (category, title);
CREATE INDEX ON "uploads" (name) WHERE not deleted;
```

#### `drop_index(table_name, col1, col2...)`

Drops an index from a table. It calculates the name of the index from the table
name and columns. This is the same as the default index name generated by PostgreSQL.

```moon
import drop_index from schema

drop_index "users", "created_at"
drop_index "posts", "title", "published"
```

This will generate the following SQL:

```sql
DROP INDEX IF EXISTS "users_created_at_idx"
DROP INDEX IF EXISTS "posts_title_published_idx"
```

### Altering Tables

#### `add_column(table_name, column_name, column_type)`

Adds a column to a table.

```moon
import add_column, types from schema

add_column "users", "age", types.integer
```

Generates the SQL:

```sql
ALTER TABLE "users" ADD COLUMN "age" integer NOT NULL DEFAULT 0
```

#### `drop_column(table_name, column_name)`

Removes a column from a table.

```moon
import drop_column from schema

drop_column "users", "age"
```

Generates the SQL:

```sql
ALTER TABLE "users" DROP COLUMN "age"
```

#### `rename_column(table_name, old_name, new_name)`

Changes the name of a column.

```moon
import rename_column from schema

rename_column "users", "age", "lifespan"
```

Generates the SQL:

```sql
ALTER TABLE "users" RENAME COLUMN "age" TO "lifespan"
```

#### `rename_table(old_name, new_name)`

Changes the name of a table.

```moon
import rename_table from schema

rename_table "users", "members"
```

Generates the SQL:

```sql
ALTER TABLE "users" RENAME TO "members"
```

### Column Types

All of the column type generators are stored in `schema.types`. All the types
are special objects that can either be turned into a type declaration string
with `tostring`, or called like a function to be customized.

Here are all the default values:

```moon
import types from require "lapis.db.schema"

types.boolean       --> boolean NOT NULL DEFAULT FALSE
types.date          --> date NOT NULL
types.double        --> double precision NOT NULL DEFAULT 0
types.foreign_key   --> integer NOT NULL
types.integer       --> integer NOT NULL DEFAULT 0
types.numeric       --> numeric NOT NULL DEFAULT 0
types.real          --> real NOT NULL DEFAULT 0
types.serial        --> serial NOT NULL
types.text          --> text NOT NULL
types.time          --> timestamp without time zone NOT NULL
types.varchar       --> character varying(255) NOT NULL
```

You'll notice everything is `NOT NULL` by default, and the numeric types have
defaults of 0 and boolean false.

When a type is called like a function it takes one argument, a table of
options. The options include:

* `default: value` -- sets default value
* `null: boolean` -- determines if the column is `NOT NULL`
* `unique: boolean` -- determines if the column has a unique index
* `primary_key: boolean` -- determines if the column is the primary key

Here are some examples:

```moon
types.integer default: 1, null: true  --> integer DEFAULT 1
types.integer primary_key: true       --> integer NOT NULL DEFAULT 0 PRIMARY KEY
types.text null: true                 --> text
types.varchar primary_key: true       --> character varying(255) NOT NULL PRIMARY KEY
```

## Database Migrations

Because requirements typically change over the lifespan of a web application
it's useful to have a system to make incremental schema changes to the
database.

We define migrations in our code as a table of functions where the key of each
function in the table is the name of the migration. You are free to name the
migrations anything but it's suggested to give them Unix timestamps as names:

```moon
{
  [1368686109]: =>
    add_column "my_table", "hello", integer

  [1368686843]: =>
    create_index "my_table", "hello"
}
```

A migration function is a plain function. Generally they will call the
schema functions described above, but they don't have to.

Only the functions that not have already been executed before will be called
when we tell our migrations to run. The migrations that have already been run
are stored in the migrations table, a database table that holds the names of
the migrations that have already been run. Migrations are run in the order of
their keys sorted ascending.

### Running Migrations

The Lapis command line tool has a special command for running migrations. It's
called `lapis migrate`.

This command expects a module called `migrations` that returns a table of
migrations in the format described above.

Let's create this file with a single migration as an example.

```moon
-- migrations.moon

import create_table, types from require "lapis.db.schema"

{
  [1]: =>
    create_table "articles", {
      { "id", types.serial }
      { "title", types.text }
      { "content", types.text }

      "PRIMARY KEY (id)"
    }
}
```

After creating the file, ensure that it is compiled to Lua and run `lapis
migrate`. The command will first create the migrations table if it doesn't
exist yet then it will run every migration that hasn't been executed yet.

Read more about [the migrate command](#command-line-interface-lapis-migrate).

### Manually Running Migrations

We can manually create the migrations table using the following code:

```moon
migrations = require "lapis.db.migrations"
migrations.create_migrations_table!
```

It will execute the following SQL:

```sql
CREATE TABLE IF NOT EXISTS "lapis_migrations" (
  "name" character varying(255) NOT NULL,
  PRIMARY KEY(name)
);
```

Now we can run migrations like so:

```moon
import run_migrations from require "lapis.db.migrations"
run_migrations require "migrations"
```
