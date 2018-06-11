+++
title = "Fun with foreign data wrappers"
author = "Peter Bex"
slug = "fun-with-fdw"
date = "2018-06-11"
description = ""
aliases = []
keywords = ["fdw", "foreign data wrappers", "postgresql"]
tags = ["fdw", "foreign data wrappers", "postgresql"]
+++

One of our clients is expanding their business into different
countries.  At Code Yellow we've created custom production software
for this client.  Of course, they would like to use our software in
the other countries, too.

At first, we considered modelling companies explicitly in the
database, and adding scoping to each and every view and query in the
system.  This would be a valid approach, but quite prone to mistakes
when adding such filters to an existing large system.  It would be
really awkward if the system started showing the data from another
company!

To avoid such issues, and to improve scalability and robustness, we
decided that a better approach would be to have separate deployments
for each company.  Our next issue would be to figure out how to
connect these deployments into an overall reporting tool and user
management.  One option would be to use an internal API between the
tool and the production system deployments.  This would require us to
add this API to the existing system and the new admin tool, which
would be fine, but a lot of work.

## Enter FDW

Given that we've always been wanting to play with Postgres Foreign
Data Wrappers (FDW), we gave this a try.  The idea behind FDW is that
you can query a table on a remote server (in any RDBMS!) as if it were
a local table.

We represent each company as a model in the administration tool, and
it automatically generates an FDW server definition in the database
for it.  The cool thing is, this allows us to query all the users in
all the databases like it was a local table, as in:

```sql
SELECT *
FROM all_connections.auth_user
ORDER BY username
LIMIT 10
OFFSET 0;
```

The magic here is handled by the `all_connections` schema which
contains tables that contain the data of every installation.

## FDW combined with table inheritance

FDW and table inheritance are two really cool Postgres features which
we can combine to make querying all instances easy.

First, we set up this schema:

```sql
CREATE SCHEMA all_connections;
```

Then we make a `auth_user` table which holds the definition.  You'll
notice this is just your ordinary Django auth user table.

```sql
CREATE TABLE all_connections.auth_user (
    id INTEGER NOT NULL,
    password VARCHAR(128) NOT NULL,
    last_login TIMESTAMP WITH TIME ZONE,
    is_superuser BOOLEAN NOT NULL,
    username VARCHAR(150) NOT NULL,
    first_name VARCHAR(30) NOT NULL,
    last_name VARCHAR(30) NOT NULL,
    email VARCHAR(254) NOT NULL,
    is_staff BOOLEAN NOT NULL,
    is_active BOOLEAN NOT NULL,
    date_joined TIMESTAMP WITH TIME ZONE NOT NULL
);
```

We could have used a different table name, but we'd prefer to keep the
name the same as the ones in the remote servers, to avoid confusion.
We use a different schema because `public` already contains the
default Django user table for the admin tool itself.

Let's say we define a new business entity.  The following happens
behind the scenes to set up foreign data wrappers for that entity's
server:

```sql
CREATE SERVER %name%
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'localhost', dbname %name%, port %local_port%);
```

And we ensure the remote user maps to the local one:

```sql
CREATE USER MAPPING FOR CURRENT_USER
SERVER %name%
OPTIONS (user 'production', password 'password');
```

Next, we create a schema named after the server to hold the mirrored
remote tables:

```sql
CREATE SCHEMA %name%;
```

Then, for every table in the `all_connections` schema, we set up a
table with the same name, but in this server's schema:

```sql
IMPORT FOREIGN SCHEMA public
LIMIT TO (%table1%, %table2%, ....)
FROM SERVER %name%
INTO %name%;
```

Finally, we set up each table to inherit from the base table:

```sql
ALTER TABLE %name%.%table1% INHERIT all_connections.%table1%;
ALTER TABLE %name%.%table2% INHERIT all_connections.%table2%;
...
```

This is the fancy magic that ties everything together.  This is what
allows us to transparently merge users from all the servers together.
We can query them as if they came from one big table.


## Struggles and challenges

Of course, things are never as simple as they appear!  While
implementing the proof of concept, I ran into a few issues.

### Finding to which entity a row belongs

You'll notice that the query on `auth_user` I showed you originally
will return multiple records with the same ID.  A typical result will
look something like this:

| `id` | `login` | `first_name` | `last_name` | ... |
|------|---------|--------------|-------------|-----|
|    1 |  admin  |           Ad |         Min | ... |
|    1 |  admin  |           Ad |         Min | ... |
|    2 |    jan  |          Jan |     Janssen | ... |
|    2 |   piet  |         Piet |   Pietersen | ... |

Now we know that there is **some** database in which there is a user
called `piet` with ID 2, but we don't know which one!  Luckily,
Postgres can do introspection on everything, which we can use to
figure out where a row came from.  Each row has a "hidden" field
called `tableoid`.  This is one of several so-called
["system columns"](https://www.postgresql.org/docs/10/static/ddl-system-columns.html).

This `oid` is just a number.  By itself, this isn't the most useful
thing, but we can query the [system
catalog](https://www.postgresql.org/docs/10/static/catalogs.html|system
catalog) to figure out to which database it belongs:

```sql
SELECT e.slug AS entity_slug, u.*
FROM all_connections.auth_user u
INNER JOIN pg_class pc
        ON pc.oid = u.tableoid
INNER JOIN pg_namespace pn
        ON pn.oid = pc.relnamespace
INNER JOIN admin_tool_entity e
        ON e.slug = pn.nspname
ORDER BY u.username, e.slug
LIMIT 10
OFFSET 0;
```

Let's say we have a business entity in the Netherlands which handles
sales, and one the UK which handles production.  These are identified
by their slugs `nl-sales` and `uk-prod`.  In this situation, the
results will look like this:

| `entity_slug` | `id` | `login` | `first_name` | `last_name` | ... |
|---------------|------|---------|--------------|-------------|-----|
|      nl-sales |    1 |  admin  |           Ad |         Min | ... |
|       uk-prod |    1 |  admin  |           Ad |         Min | ... |
|       uk-prod |    2 |    jan  |          Jan |     Janssen | ... |
|      nl-sales |    2 |   piet  |         Piet |   Pietersen | ... |

This works because we named the schemas for each remote database
according to the corresponding entity's slug.  We match the schema via
`pg_namespace` on the entity's `slug`.  This allows us to join the
schema name of the table from which the row came to the entity via the
slug.  To ensure this is stable, we disallow the slug from being
changed after an entity has been created.


### Missing sequences

The first thing I tried when querying my first remote table is
obviously a `SELECT` query, and this indeed works exactly as
envisioned.  But as soon as I tried to actually `INSERT` something
into the remote table, I noticed that `SERIAL` columns (which are used
for primary key ID fields by Django) have a
[sequence](https://www.postgresql.org/docs/10/static/functions-sequence.html|sequence),
which assigns the next value when inserting a new record.

Unfortunately, there's no way to import the sequence into a remote
table, which means there is no way to insert a new record.  It will
try to insert a new record with a `NULL` value for the `id`, but this
is obviously not allowed, as the primary key must always be present
and non-`NULL`.  You could explicitly assign an `id` value yourself,
but then this will cause conflicts when the sequence isn't incremented
to the new value.

The solution I eventually came up with is to import a **second** copy
of the table **without** the `id` column into a separate schema.  This
can then be inserted into, and instead of trying to insert a `NULL`
value into the remote table, it will simply omit the column value and
allow the remote server to assign the value, just as you do in the
`INSERT` statement.  First we create the schema:

```sql
CREATE SCHEMA %name%_insertable;
```

And then we import the table yet again into the new schema:

```sql
IMPORT FOREIGN SCHEMA public
LIMIT TO (%table1%, %table2%, ....)
FROM SERVER %name%
INTO %name%_insertable;
```

And finally we drop the `id` column from the tables we just imported:

```sql
ALTER FOREIGN TABLE %table1% DROP COLUMN id;
ALTER FOREIGN TABLE %table2% DROP COLUMN id;
...
```

When we want to insert a new user for server `foo`, we do something along the lines of:

```sql
INSERT INTO foo_insertable.auth_user (username, email, password, ...) 
VALUES ('user1', 'user1@example.com', '$2$....');
```

It's not very elegant, but it's straightforward and works like a
charm, especially considering Django guarantees an `id` field if you
don't manually override it (which we typically don't do in our
projects).


### Trouble in unit tests

Now, I'm thoroughly infected by the testing virus.  In fact, at the
office I have a reputation for always writing unit tests to the point
that people **expect** to break a unit tests when touching any of my
code :)

So, of course, I wanted to write unit tests for this FDW stuff.
Unfortunately, [`CREATE DATABASE` does not work in a
transaction](https://www.postgresql.org/docs/10/static/sql-createdatabase.html).
This is one of **very** few limitations in Postgres, so I was a bit
disappointed to find this out.  In Django, all tests usually run
[inside a transaction](https://docs.djangoproject.com/en/2.0/topics/db/transactions/#use-in-tests),
which makes tests self-contained and less dependent on state:
if something breaks, you won't end up with leftovers from previous
test runs.

While looking into this, I also figured out that [`postgres_fdw`
foreign data wrappers use their own transaction
management](https://www.postgresql.org/docs/10/static/postgres-fdw.html),
which has some implications for tests: when you insert a record in the
remote database's table (which happens from another connection in my
tests, as this is how it will work in practice), this is not seen
during the same test run.

So, I had to use the dreaded `TransactionTestCase` class, which means
all those benefits of the normal `TestCase` class go flying out the
window.


### Querying and connection management

The query above where we merge in the entity name through
introspection is hand-written and (as far as I know) cannot be
expressed with the Django ORM.  Because Django's query builder is
tightly coupled to the ORM, there is no way you can cleanly query the
database, so I ended up having to rely on hand-crafted SQL.  This is a
bit of a pain when dynamically composing queries with filters, and it
also means we can't rely on
[Binder](https://github.com/CodeYellowBV/django-binder/), our
homegrown REST framework for Django.

I briefly tried to use [SQL Alchemy](http://www.sqlalchemy.org/), but
couldn't figure out how to cleanly integrate it into our test suite
and how to integrate it with Django's connection management, and were
in somewhat of a hurry to make the deadline.


## Conclusion

So far, FDW combined with table inheritance seems like a very cool
approach that takes less time to implement than building an internal
API on two sides (client and server).  There are still some remaining
challenges, like how to deal with migrations, and downtime of servers,
but all in all I'm quite happy with the results so far.

At the very least, working through this implementation gave us the
opportunity to try out some of Postgres' more cutting-edge stuff.  We
can certainly use this knowledge in future projects.
