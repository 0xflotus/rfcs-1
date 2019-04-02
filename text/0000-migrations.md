- Start Date: 2019-03-22
- RFC PR: (leave this empty)
- Prisma Issue: (leave this empty)

# Summary

In order to make Prismas existing migration system more powerful, we introduce a migration folder which includes datamodel snapshots per migration.

# Basic example

When starting out with a simple project like this:

```
.
├── datamodel.mdl
└── prisma.yml
```

Executing `prisma migrate` will create the following folder structure and migrate the underlying database structure:

```
.
├── datamodel.mdl
├── migrate
│   └── 20190322092247
│       └── datamodel.mdl
└── prisma.yml
```

So in addition to migrating the database, `prisma migrate` now also creates a new migration folder with the timestamp
of the migration.

# Motivation

These are the main reasons we want to add migration files:

- Being able to run arbitrary data migration scripts
- Having SQL / other database native escape hatches to perform structural database changes which Prisma doesn't support yet

# Terminology

Assuming that the reader knows, what Prisma 1.x looks like, the following
terms are new concepts which we introduce with this spec.

## Migration

When changing the datamodel of a Prisma project, we now have a new datamodel in the local filesystem, while the database and running Prisma instance don't know yet about this change. The action of applying this diff between the old datamodel and the new one is called "performing a migration".
In other words, you could see the diff between two data models, that is going to be applied as a migration.
While this is a concept that has been mostly hidden until now within Prisma 1.x, users of Prisma are going to be exposed to this concept.
This allows more fine-grained control over how datamodel changes get applied in the database (migrated).

## Declarative vs imperative migrations

### Declarative migration

An example for a declarative migration is this:
You start with datamodel `A` and change it to datamodel `B`, while not providing the actual steps needed to achieve this change.
The only thing you need to provide is the following:

`Datamodel A`

```graphql
model User {
  id: ID! @id
  name: String
}
```

`Datamodel B`

```graphql
model User {
  id: ID! @id
  name: String!
}
```

In this example we made the `name` field required.
The actual steps needed to achieve the change will be _inferred_ by Prisma.
Therefore we can also talk about declarative migrations as **automatic migrations**.

### Imperative migration

In traditional migration systems as the one provided by [Active Record](https://edgeguides.rubyonrails.org/active_record_migrations.html), the actual change is instead expressed in an imperative manner:

```ruby
class ChangeUserName < ActiveRecord::Migration[5.0]
  def change
    reversible do |dir|
      change_table :User do |t|
        t.string :name, null: false
      end
    end
  end
end
```

With imperative migrations, the user needs to both calculate the "diff" in his head and write the needed actions down, which will result in the new schema.

While in Prismas model the declarative datamodel is the source of truth for migrations, Active Record takes the imperative migrations as the source of truth for migrations.
Therefore one can call the imperative migrations also **manual migrations**.

## Migration Scripts

While Prisma solves the 95% use-case with the automatic migrations based on the user changing the datamodel, there are cases, in which this is not a sufficient solution.
One case could for example be, that I want to use a database primitive, which is not yet being supported by Prisma. In this case I would like to run a database native script, e.g. a SQL script.
I might also want to use a SQL script which will run after the migration was successful to insert data.
If the abstraction level of SQL is too low for me, having access to the Prisma Client would be very beneficial. Before introducing a `unique` constraint, I might for example run a TypeScript script, which makes sure, that there are no duplicates in the database.
So both migration scripts with the favorite language as TypeScript, JavaScript or Go are possible, while we also support low-level database access in SQL and other database languages.

## Migration Hook

A migration hook is used to run migration scripts. The two possible hooks are "before" and "after". Prisma decides, which file to execute at which point in time based on a filename convention. If there e.g. is a `after.go` file in the migration folder, Prisma will execute that Go script after the migration of the datamodel was successful.

## Datamodel Snapshots

As mentioned in [Declarative migration](#how-does-naming-of-migrations-work), the user adjusts the `datamodel.prisma` file, which results in a new migration folder when running `prisma migrate`. In order for the migration engine to know which changes need to be applied, it needs a "copy" of the datamodel. As this "copy" describes the datamodel in a specific point in time, we also refer to it as a "snapshot".

## Migration History

When a migration is running or has been executed successfully, Prisma writes this information into a table or collection in the database.
This table is also known as the "Migration History".
It includes all successful and failed migrations of the specific Prisma Project.
The benefits of storing this migration history are the following:

- Prisma can make sure, that the local migrations are in sync with the databases migrations
- Prisma can ensure, that migrations don't get run twice
- Prisma can give an overview in a GUI interface about the currently running migration and migrations that already have been running
- Prisma can rollback migrations when there has been an error

# Detailed design

As mentioned in the [basic example](#basic-example), a simple `prisma migrate` is needed to migrate the database and create a new migration.

These are the commands:

```bash
prisma migrate                                Creates a migration, clears the draft and migrates the database

       migrate draft                          Pushes the new changes into a draft without creating a migration
       migrate rollback                       Rolls back migration(s)
       migrate plan                           Creates a migration without applying it
       migrate apply                          Applies all unapplied migrations
       migrate discard                        Discard changes of the draft mode
       migrate run-post-hook [migration]      Runs a specific post hook of a migration (Useful for implementing hooks)
       migrate run-pre-hook [migration]       Runs a specific pre hook of a migration  (Useful for implementing hooks)
```

Let's discuss a few questions.

## How does naming of migrations work?

When executing `prisma migrate` without providing any args, a migration with the name `yyyyMMddHHmmss` (e.g. `20190322092247`) will be created.
Especially when working in a team, it can be very useful to add a custom name to the migration.
A name can be provided like this:

```bash
$ prisma migrate --name my-initial-migration
```

This will create the migration folder `migrate/20190322092247-my-initial-migration/` with the following content:

```bash
.
└── 20190322092247-init
    ├── client.ts
    └── datamodel.mdl
```

This means that every migration will have its own generated Prisma Client in the chosen language (JS, TS or Go).

## How can I hook in with my own migration logic?

There are a few use-cases when you want to run your own custom migration logic:

- Adding specific database primitives like functions in Postgres, which are not yet supported by Prisma
- Initializing data after a migration has been executed. Both SQL or the Prisma Client could be useful here

Let's say that after Prisma has migrated the database to the datamodel defined in `migrate/20190322092247-my-initial-migration/`,
you want to perform an action.
Prisma provides convention-based hooks. If you e.g. create a `after.ts`, it will be picked up by Prisma and executed at the right point in time.
These are the possible migration hooks:

```bash
├── 20190322092247-my-initial-migration
│   ├── after.down.sql
│   ├── after.go
│   ├── after.ts
│   ├── after.up.sql
│   ├── before.down.sql
│   ├── before.go
│   ├── before.ts
│   ├── before.up.sql
│   ├── client.go
│   ├── client.ts
│   └── datamodel.mdl
```

## How can I perform a rename?

The very nature of the declarative datamodel introduces ambiguities for transitions between two datamodels.
One of these ambiguities is renaming a field.

If we have the following datamodel:

```graphql
model User {
  id: ID @id
  name: String
  address: String
}
```

And want to rename the `name` field, it's not clear if it should be renamed based on the `address` field or the `name` field:

```graphql
model User {
  id: ID @id
  name2: String
  address2: String
}
```

For the reader it's very clear what should happen, whereas it's not trivial to detect this programmatically as we can't rely on the order of fields in the datamodel.
The migration engine doesn't know, if it should rename `name` to `name2` or `name` to `address2` and vise versa.
Maybe we even want to delete the `name` field with all its data and create a new field called `name2`?
Information for this transition between datamodels is needed.

One way to solve this is using the `@db` directive:

```graphql
model User {
  id: ID @id
  name: String @db(name: "name2")
  address: String @db(name: "address2")
}
```

Another way would be to express the renaming with SQL. With the SQL hooks that we provide, we now would perform the rename _before_
the datamodel will be applied.

This is how the SQL could look like:

`before.up.sql`

```SQL
ALTER TABLE User CHANGE "name" "name2"
```

`before.down.sql`

```SQL
ALTER TABLE User CHANGE "name2" "name"
```

## Can I just delete old migrations?

Let's say you use the migrations system for a couple of years and you accumulated over 1000 migrations.
All your Prisma instances already include these changes, so there is no point of storing migrations that are years old.
If you decide that you don't need the first 500 migrations, you can simply delete these folders.
Note that you can't delete folders in between migrations, it always has to happen right from the beginning.

## Can I lock the database during the migration to prevent data corruption?

While performing a migration like turning an optional relational into a required one, it may be beneficial to apply a lock on the database
to prevent data corruption.
We need to find out here if this should run on Prisma application level or in the individual database.
It should be configurable to add this lock.
Probably it's something Prisma will provide in the future with the Prisma server but not with the Prisma binary.

## In which order are migrations executed?

Migrations are executed lexicographically and with that in the order that they're stored in the filesystem.

## Could I just use Prisma as a migration runner without the datamodel?

There speaks nothing against that, yes. The only disadvantage is, that Prisma can't give you any of the guarantees anymore.
Your migration folder could just contain SQL scripts if you would like to.

## How to solve complex merge conflicts?

Let's say Alice and Bob start developing in their own branches based on the following datamodel:

```graphql
model User {
  id: ID! @id
  name: String
  address: String
}
```

And these migrations:

```bash
.
└── migrate
    ├── 1
    │   └── datamodel.mdl
    └── 2
        └── datamodel.mdl
```

`1` and `2` are just used for illustration, we're using timestamps in the real world.

Alice removes the `address` field and creates a new migration `4`.

```graphql
model User {
  id: ID @id
  name: String
}
```

```bash
.
└── migrate
    ├── 1
    │   └── datamodel.mdl
    ├── 2
    │   └── datamodel.mdl
    └── 4
        └── datamodel.mdl
```

Her changes get deployed to production.
Bob now pulls her changes and finds a merge conflict.

His `migrate` folder will look like this:

```bash
.
└── migrate
    ├── 1
    │   └── datamodel.mdl
    ├── 2
    │   └── datamodel.mdl
    ├── 3
    │   └── datamodel.mdl
    └── 4
        └── datamodel.mdl
```

The main `datamodel.mdl` file will look like this:

```graphql
model User {
  id: ID @id
  name: String
<<<<<<< HEAD
  address: String?
=======
>>>>>>> master
  posts: [Post]
}

model Post {
  id: ID @id
  title: String
}
```

He now can clearly see, that in the `master` branch the field `posts` has been removed. Git is pointing him here to the merge conflict.
Another important conflict, where Git doesn't help us are the migration folders `3` and `4`.

The right way to solve this conflict is the following:

```
$ prisma migrate rollback # rollbacks 3
```

Now he renames `3` to `5`, so that his changes comes after `4`.
He can confirm locally with a `prisma migrate` to his local machine that the change works and push to production.
The production system will now pick up the new `5` migration and apply it.

There is an infinite amount of conflict scenarios, so we're stopping here and won't go deeper into them.
What matters is this: Prisma will be able to help developers solving migration conflicts based on the _local migration history_ and the _remote migration history_.

## The draft mode

In the previous scenario it was quite easy to reason about the conflict resolution, as we just had to look into one migration from Alice and one from Bob.
Let's say that Alice had pushed 10 new migrations and the same for Bob, he also performed 10 migrations.
This would be a very complicated situation to reason about.
In order to circumvent this issue, we strongly advice users to do two things:

1. New your migrations. This will help your team to understand what happened in there
2. Group all your changes into as few migrations as possible. This way conflict resolution is way easier.

In order to make "grouping" or accumulating multiple schema changes into one migration, we introduce the `prisma migrate draft` command.
With this command, changes will be applied to the database without creating a new migration.
As soon as you're happy with all the changes, you can execute `prisma migrate`, which empties the draft and puts all accumulated changes into one new migration.

# Drawbacks

Providing a migration value or renaming requires SQL.

# Alternatives

All other systems that are known to us have an imperative approach of defining migrations.
That means, their migrations include something like `createField`, whereas with Prisma you
just store the desired datamodel and Prisma infers the imperative actions automatically.

Popular other migration systems include:

## Go

- [Go Migrate](https://github.com/golang-migrate/migrate) Up & Down migrations
- [Goose](https://github.com/pressly/goose) Up & Down migrations
- [Go PG Migrations](https://github.com/robinjoseph08/go-pg-migrations) Mix between [go-pg/migrations](https://github.com/go-pg/migrations) and [Knex](https://knexjs.org/)
- [Gormigrate](https://github.com/go-gormigrate/gormigrate) Programmatic API with structs

## Python

- [Django](https://docs.djangoproject.com/en/2.1/topics/migrations/) Django style fixtures
- [Alembic](https://pypi.org/project/alembic/) Belongs to SQLAlchemy

## PHP

- [CakePHP - Phinx](https://github.com/cakephp/phinx) Belongs to CakePHP
- [Doctrine](https://www.doctrine-project.org/projects/doctrine-migrations/en/2.0/reference/managing-migrations.html#managing-migrations) Mix between PHP and SQL

## Java

- [Flyway](https://flywaydb.org/) Standalone migration tool, enterprise-grade. Up & Down.

## Node.js

- [Knex](https://knexjs.org/#Migrations)
- [Sequelize](http://docs.sequelizejs.com/manual/migrations.html)

## Ruby

- [Active Record](https://edgeguides.rubyonrails.org/active_record_migrations.html)

# Adoption strategy

This is a new paradigm of how to do migrations and should probably land in Prisma 2.

# How we teach this

Most of the migration flows are fairly easy to understand and are similar to what we already have.
The difference is that now we're introducing files in the filesystem and allow now use-cases.
In addition with this migration system more complex merge conflicts can occur.

Both use-cases of hooks and conflicts need to be properly documented.

# Unresolved questions

- [x] Summarize and describe new concepts and terminology
- [ ] Spec out the workflow of generating a migration
  - [ ] Generate client into `node_modules` (also consider versioning e.g. `npx prisma@2.x generate`)
- [ ] Spec out how migration "hooks" are working (e.g. `before.up.sql`) as it's depending on individual connectors
- [ ] Spec out CLI output of each migration related command
- [ ] 3 migration modes (1. delete all data, 2. keep data + downtime, 3. keep data + zero downtime)
  - [ ] How to simplify renames for local development. Ideas:
    - `prisma dev`: interactive double list select?
    - Prisma Admin feature
- [ ] Transaction behavior (also when running scripts)
