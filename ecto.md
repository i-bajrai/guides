# Ecto Basics

This guide is an introduction to [Ecto](https://github.com/elixir-lang/ecto),
the database wrapper and query generator for Elixir. Ecto provides a
standardised API for talking to all the different kinds of databases, so that
Elixir developers can query whatever database they're using in the same
fashion. If one application uses MySQL and another uses PostgreSQL but both
use Ecto, then the database querying for both of those applications will be
almost identical.

If you've come from the Ruby language, the equivalent there would be Active
Record, or Sequel. Java has Hibernate, and so on.

In this guide, we're going to learn some basics about Ecto, such as creating,
reading, updating and destroying records from a PostgreSQL database. 

**This guide will require you to have setup PostgreSQL beforehand.**

## Adding Ecto to an application

To start off with, we'll generate a new Elixir application by running this command:

```
mix new friends --sup
```

The `--sup` option ensures that this application has [a supervision tree](http://elixir-lang.org/getting-started/mix-otp/supervisor-and-application.html), which we'll need for Ecto a little later on.

To add Ecto to this application, there are a few steps that we need to take. The first step will be adding Ecto and an adapter called Postgrex to our `mix.exs` file, which we'll do by changing the `deps` definition in that file to this:

```elixir
defp deps do
  [
    {:ecto, "2.0.0-beta.2"},
    {:postgrex, "0.11.1"}
  ]
end
```

Ecto provides the common querying API, but we need the Postgrex adapter installed too, as that is what Ecto uses to speak in terms a PostgreSQL database can understand.

To install these dependencies, we will run this command:

```
mix deps.get
```

In this same file, we'll need to add `postgrex` to our applications list:

```elixir
def application do
  [applications: [:logger, :postgrex],
   mod: {Friends, []}]
end
```

The Postgrex application will receive queries from Ecto and execute them
against our database. If we didn't do this step, we wouldn't be able to do any
querying at all.

That's the first two steps taken now. We have installed Ecto and Postgrex as
dependencies of our application. We now need to setup some configuration for
Ecto so that we can perform actions on a database from within the
application's code.

The first bit of configuration is going to be in `config/config.exs`. On a new line in this file, put this content

```elixir
config :friends, Friends.Repo,
  adapter: Ecto.Adapters.Postgres,
  database: "friends",
  username: "postgres",
  password: "postgres"
```

**NOTE**: Your PostgreSQL database may be setup to not require a username and password. If the above configuration doesn't work, try removing the username and password fields.

This piece of configuration configures how Ecto will connect to our database, called "friends". Specifically, it configures a "repo". More information about [Ecto.Repo can be found in its documentation](https://hexdocs.pm/ecto/Ecto.Repo.html).

The next thing we'll need to do is to setup the repo itself, which goes into `lib/friends/repo.ex`:

```elixir
defmodule Friends.Repo do
  use Ecto.Repo,
    otp_app: :friends
end
```

This module is what we'll be using to query our database shortly. It uses the `Ecto.Repo` module, and the `otp_app` tells Ecto which Elixir application it can look for database configuration in. In this case, we've told it's the `:friends` application where Ecto can find that configuration and so Ecto will use the configuration that we set up in `config/config.exs`.

The final piece of configuration is to setup the `Friends.Repo` as a worker within the application's supervision tree, which we can do in `lib/friends.ex`, inside the `start/2` function:

```elixir
def start(_type, _args) do
  import Supervisor.Spec, warn: false

  children = [
    worker(Friends.Repo, []),
  ]

  ...
```

This piece of configuration will start the Ecto process which receives and executes our application's queries. Without it, we wouldn't be able to query the database at all!

We've now configured our application so that it's able to make queries to our database. Let's now create our database, add a table to it, and then perform some queries.

## Setting up the database

To be able to query a database, it first needs to exist. We can create the database with this command:

```
mix ecto.create
```

If the database has been created successfully, then you will see this message:

```
The database for Friends.Repo has been created.
```

**NOTE:** If you get an error, you should try changing your configuration in `config/config.exs`, as it may be an authentication error.

A database by itself isn't very queryable, so we will need to create a table within that database. To do that, we'll use what's referred to as a _migration_. If you've come from Active Record (or similar), you will have seen these before. A migration is a single step in the process of constructing your database.

Let's create a migration now with this command:

```
mix ecto.gen.migration create_people
```

This command will generate a brand new migration file in `priv/repo/migrations`, which is empty by default:

```elixir
defmodule Friends.Repo.Migrations.CreatePeople do
  use Ecto.Migration

  def change do

  end
end
```

Let's add some code to this migration to create a new table called "people", with a few columns in it:

```elixir
defmodule Friends.Repo.Migrations.CreatePeople do
  use Ecto.Migration

  def change do
    create table(:people) do
      add :first_name, :string
      add :last_name, :string
      add :age, :integer
    end
  end
end
```

This new code will tell Ecto to create a new table called people, and add three new fields: `first_name`, `last_name` and `age` to that table. The types of these fields are `string` and `integer`. (The different types that Ecto supports are covered in the [Ecto.Schema](https://hexdocs.pm/ecto/Ecto.Schema.html) documentation.)

**The naming convention for tables in Ecto databases is to use a pluralized name.**

To run this migration and create the `people` table, we will run this command:

```
mix ecto.migrate
```

If we found out that we made a mistake in this migration, we could run `mix ecto.rollback` to undo the changes in the migration. We could then fix the changes in the migration and run `mix ecto.migrate` again. If we ran `mix ecto.rollback` now, it would delete the table that we just created.

We now have a table created in our database. The next step that we'll need to do is to create the model.

## Creating the model

Let's create the model within our application at `lib/friends/person.ex`:

```elixir
defmodule Friends.Person do
  use Ecto.Schema

  schema "people" do
    field :first_name, :string
    field :last_name, :string
    field :age, :integer
  end
end
```

This model defines the schema from the database that this model maps to. In this case, we're telling Ecto that the `Friends.Person` model maps to the `people` table in the database, and the `first_name`, `last_name` and `age` fields in that table. The second argument passed to `field` tells Ecto how we want the information from the database to be represented in our model.

**We've called this model `Person` because the naming convention in Ecto for models is a singularized name.**

We can play around with this model in an IEx session by starting one up with `iex -S mix` and then running this code in it:

```elixir
person = %Friends.Person{}
```

This code will give us a new `Friends.Person` struct, which will have `nil` values for all the fields. We can set values on these fields by generating a new struct:

```elixir
person = %Friends.Person{age: 28}
```

Or with syntax like this:

```elixir
%{person | age: 28}
```

The model struct returned here is essentially a glorified Map. Let's take a look at how we can insert data into the database.

## Inserting data

We can insert a new record into our `people` table with this code:

```elixir
person = %Friends.Person{}
Friends.Repo.insert person
```

To insert the data into our database, we call `insert` on `Friends.Repo`, which is the module that uses Ecto to talk to our database. The `person` struct here represents the data that we want to insert into the database.

A successful insert will return a tuple, like so:

```elixir
{:ok,
 %Friends.Person{__meta__: #Ecto.Schema.Metadata<:loaded>, age: nil,
  first_name: nil, id: 1, last_name: nil}}
```

The `:ok` atom can be used for pattern matching purposes to ensure that the insert succeeds. A situation where the insert may not succeed is if you have a constraint on the database itself. For instance, if the database had a unique constraint on a field called `email` so that an email can only be used for one person record, then the insertion would fail.

You may wish to pattern match on the tuple in order to refer to the record inserted into the database:

```elixir
{ :ok, person } = Friends.Repo.insert person
```

## Validating changes

In Ecto, you may wish to validate changes before they go to the database. For instance, you may wish that a person has provided both a first name and a last name before a record can be entered into the database. For this, Ecto has [_changesets_](https://hexdocs.pm/ecto/Ecto.Changeset.html).





## Querying the database
