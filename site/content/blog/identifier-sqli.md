+++
title = "What your framework never told you about SQL injection protections"
author = "Peter Bex"
slug = "identifier-sqli"
date = "2014-05-20"
description = ""
aliases = ["/identifier-sqli.html"]
keywords = ["security", "frameworks", "SQL injection", "Laravel", "FuelPHP", "CakePHP", "CodeIgniter", "Lithium"]
tags = ["security", "frameworks", "SQL injection", "Laravel", "FuelPHP", "CakePHP", "CodeIgniter", "Lithium"]
+++


We've discovered that SQL injection is to this day not a fully solved
problem, even in most popular frameworks.  In this post, we'll explain
how these frameworks fail at escaping parts of a query, culminating in
the discovery of a critical vulnerability in the popular Laravel
framework which affects a large percentage of applications.

Let's start with an innocent example, which provides the starting
point of our journey.  This is a typical simple use case: a
filterable, sortable list.

From safe to vulnerable in one step
===================================

In [FuelPHP](http://www.fuelphp.com), a (much simplified) controller
action to fetch a JSON result might look like this:

```php
<?php
public function get_products()
{
    $conditions = array('where' => array(array('deleted', '=', false)));
    $minprice = \Input::get('minprice', null);
    if (isset($minprice))
      $conditions['where'][] = array('price', '>=', $minprice);

    $result = Model_Product::find('all', $conditions);
    return $this->response($result);
}
```

This would return products, optionally filtered by a minimum price.
It uses the input directly, without checking whether the price is
syntactically a valid number.  Now, that's pretty lazy, but regardless
of not validating, this code is protected by the framework against SQL
injection.  If the input is non-numerical, and we're using a sane
database, this would cause a data constraint error, resulting in an
exception being thrown in PHP.  It can be argued that this is fine:
there is nothing sane you can do with bogus input, so returning a HTTP
status of 500 is acceptable.  The main point is: this code is *not*
vulnerable to SQL injection.

Now, let's suppose we wanted to allow the user to sort the product
list by any field:

```php
<?php
public function get_products()
{
    $conditions = array('where' => array(array('deleted', '=', false)));
    $minprice = \Input::get('minprice', null);
    if (isset($minprice))
      $conditions['where'][] = array('price', '>=', $minprice);

    $conditions['order_by'] = array(Input::get('sort_field', 'id') => Input::get('sort_order', 'asc'));

    $result = Model_Product::find('all', $conditions);
    return $this->response($result);
}
```

Again, you might argue that it's a bad idea to allow the user to
filter by arbitrary column names, and we would agree: this would open
up a relatively minor information leak if the user can sort on a field
that's not supposed to be exposed to them (for example, a field that
contains the cost price of the item).  But if the user truly is
allowed to sort on *any* column from the table, this code should be
safe: it would just cause an exception if the column doesn't exist.
In that respect, it behaves in a way similar to our first example.

By now you must have guessed that FuelPHP did't escape identifiers
correctly.  Basically it simply wraps the identifier in (back)quotes,
but that is trivial to break out of, of course.  In fact, it's worse:
the sort direction string would be simply appended, **as-is**, to the
SQL statement!

To make things worse, the documentation explicitly states that code
using the query builder is safe because [escaping is automatically
done](http://www.fuelphp.com/docs/general/security.html#sql).  To be
fair, it only mentions that *values* are escaped, but this isn't very
clear, and there is no reason to assume that identifiers are treated
differently.  The framework **could** escape identifiers, and it is
strongly implied that it will, so it **should** do it.

We believe that the failure mode of accidentally passing user
controlled names to filtering or ordering functions should be
proportionate to the mistake: at worst, it should cause a limited
information leak.  Instead, this breach of the abstraction provided by
the query builder exposes us to a critical vulnerability by allowing
arbitrary SQL to be injected.  Besides, we believe that completely
putting the responsibility of filtering the input on the user is
unacceptable.

Finally, the behaviour is simply *incorrect*: what if we really
**want** to query a column or create a column alias containing a
special character?  This might happen, for example, when working with
legacy databases with weird naming conventions, or when naming columns
for a COPY OUT statement.

Widespread problem
------------------

At first, the problem was simply reported to FuelPHP with a patch and
proof of concept.  The FuelPHP core team was quick to respond and
agreed with us that you shouldn't be doing this, but even if you did,
it should not be exploitable in this way.

Because at Code Yellow we are currently looking at a few other
frameworks for our projects, we decided to have a quick look to see
how *they* dealt with it.  It turned out to be worse than we had
imagined.  Everywhere we looked we saw essentially the same code:

```PHP
<?php
function quoteIdent($name) {
    return sprintf('"%s"', $name);
}
```

The implementations in CakePHP, CodeIgniter and FuelPHP before the
patch were the worst of all.  They were so incredibly large and
complex they couldn't *possibly* be safe.  Just have a look at the
[`name`](https://github.com/cakephp/cakephp/blob/2eadb89ff6fdc29754a8a0b7686852a245d66a8d/lib/Cake/Model/Datasource/DboSource.php#L784)
function from CakePHP or the
[`escape_identifiers`](https://github.com/EllisLab/CodeIgniter/blob/640ff4e348f6370b84b5d23c811f087e22d13819/system/database/DB_driver.php#L1271)
function from CodeIgniter.

I certainly wouldn't be able to confidently say these functions are
secure, even if they *did* fix all the issues in them.  It's not even
clear what exactly they're trying to do, there's just too much code
getting in the way.  And overly *clever* code, at that: I blinked and
had to read the CodeIgniter function several times before I even
understood the basic idea.

It looks like these frameworks are trying to "do what I mean".  For
example, they attempt to quote identifiers which occur inside
aggregate expressions, like ```SUM("x")```, and they split ```x.y```
into ```"x"."y"```.  Unfortunately, this causes ambiguities: what if
you have a column with the *name* ```SUM(x)```, or ```x.y```, however
unlikely?  Instead, these functions should be doing only one simple
thing: quote the identifier safely for use in a query.  Arbitrary SQL
fragments can be made in FuelPHP using
[DB::expr](http://fuelphp.com/docs/classes/database/db.html#/method_expr),
so there's already a provision for inserting a literal ```SUM(x)```
into a query.  This obviates the need for more magic in the identifier
quotation function.

Finally, if it's decided that it really **is** the user's
responsibility to avoid "strange" column names, and identifiers will
never get quoted correctly (which is a cheap cop-out, in our opinion),
it should at least be mentioned clearly in the documentation.  Most of
the frameworks we looked at simply didn't mention anything about this
in the manual or API docs.  A simple statement along the lines of
"only use trusted input as identifiers: they are not always escaped
correctly" should be enough.

It gets worse
-------------

The above example is a bit of a stretch, because querying on a
user-controlled field name wouldn't (or *shouldn't*?) occur much in
practice.  However, thanks to a tip from [Alex
Soban](https://github.com/SobanVuex) we started looking at "create" and
"update" functions as well.  This led to the discovery that in
[Laravel](http://laravel.com/), incorrect identifier quotation caused
a much bigger problem: its ORM uses the field quotation function for
```INSERT``` and ```UPDATE``` queries, without checking that the
fields exist.  This means that the following little snippet opens up a
huge vulnerability in your application:

```php
<?php
    $u = User::find($id);
    $u->fill($_POST);
    $u->save();
```

If the POST body contains a key/value pair where the key is
```email"='foo@example.com',"password```, Laravel will simply place
quotes around it, which results in the following query:

```sql
    UPDATE "users" SET "email"='foo@example.com',"password" = ? WHERE "id" = ?
```

This allows an attacker to update the password via the first
positional argument, which will be filled in with the value
corresponding to that weird key in our POST.  This works even if you
have a ```$guarded``` clause in your user model which should prevent
that, like this:

```php
<?php
    protected $guarded = array('id', 'password');
```

The only way to fix this is by using a whitelist instead of a
blacklist (which is recommended as a security best practice, anyway):

```php
<?php
    protected $fillable = array('name', 'email');
```

Please note that **all** models must use ```$fillable```: if there's
even one model that doesn't, an attacker can insert the result of
arbitrary subqueries into records, which allows them to systematically
scrape your entire database.

The other frameworks generally check the list of supplied values
against the list of columns in the database, and only store those.
That means that they can't be attacked in this particular way, but
programs using attacker-controlled column names in queries like we've
seen in the introduction are still vulnerable.  Let's see how bad the
situation is for those frameworks.

First off, there aren't many cases when arbitrary attacker-controlled
strings are sent as identifiers in a query.  Furthermore, if it is, it
is most likely going to be in a column argument for an ```ORDER BY```
clause.  This is one of the final clauses that occur in a ```SELECT```
query, and the SQL grammar allows only a handful of trailing clauses
to an ```ORDER BY```: only ```LIMIT```/```FETCH``` and ```OFFSET```
are allowed there, and an optional locking clause.  Anything else will
cause a syntax error to be signaled: you can't put a ```UNION```
following the ```ORDER BY```, for example (which is one of the most
common ways to exploit SQL injection bugs), but you *can* order by a
subquery expression, which makes for an information leak which can be
exploited by enumerating things of interested.

Of course, if the ```SELECT``` happens to be embedded in a larger
query (as a subquery or a [common table
expression](http://www.postgresql.org/docs/current/interactive/queries-with.html)),
or when the query procedure being used allows for multiple SQL
commands to be sent off to the database (separated by semicolons),
there will be **plenty** of opportunity to abuse it.  And of course
there may be some (future?) extension of SQL that will allow trailing
query commands to be embedded.  Even then, there's always someone more
clever than you who may come up with creative ways to squeeze out more
information from the database through the ways described above.


Fixing the problem
==================

Of course, we contacted the various vendors which we found to be
affected by this problem.  It proved to be more difficult than
expected: The [Laravel](http://www.laravel.com),
[Lithium](http://www.li3.me) and
[CodeIgniter](http://www.codeigniter.com) frameworks did not provide
any security contact information on their websites, and when we asked
on IRC, we were suggested to file a ticket on GitHub.  In our opinion,
posting a security issue on a public issue tracker is not a
[responsible way](https://en.wikipedia.org/wiki/Responsible_disclosure)
of disclosing vulnerabilities!

In the end we decided to contact Lithium and CodeIgniter through the
websites of the companies behind them.  In the case of Laravel we
contacted its original author.  As fortune would have it, we reported
the issue one day before [Laracon
2014](https://conference.laravel.com/).  He was unavailable due to
this conference, so our e-mail went unseen for 5 days.  This shows why
it is important to have a security team, with contact details clearly
posted on the framework's website.

Since reporting this issue, Laravel has
[fixed](https://github.com/laravel/framework/commit/c942a1bc30999e3d7a5567f264bfc99c77843919)
the issue and updated the
[documentation](https://github.com/laravel/docs/commit/5afdbb899c714cb3ece6fe3eaecb54c5945be765)
to more clearly state this pitfall, FuelPHP has
[fixed](https://github.com/fuel/core/commit/7574d5c78aae52ecdbedd138b2614bb2345ecff5)
the issue,
[CodeIgniter](https://github.com/EllisLab/CodeIgniter/commit/8c833f4c096a1fa9187c599943159eacb3f7133a)
and
[CakePHP](https://github.com/cakephp/docs/commit/bf6ab74954b1852c53d52573a76acf3ac198482d)
documented it as a potential developer pitfall.  The
[Lithium](http://li3.me/) core team are still deliberating on what to
do.

We'd like to tip our hat to Thijs from
[WhiteHats](https://www.whitehats.nl/) for discussing with us how to
best report the issue with so many frameworks, and for contacting the
[Drupal](http://www.drupal.org) team about improving their
documentation about this issue.

Surprisingly few frameworks quote identifiers correctly.  To find
frameworks which do it correctly, we looked outside the PHP ecosystem,
and even there we found that only [Ruby on
Rails](http://rubyonrails.org/) correctly quotes identifiers
throughout.  Even [Django](https://www.djangoproject.com/) doesn't
quote correctly, but it consistently checks all attributes against
database columns which mean it's probably not vulnerable.  Of course,
we haven't been able to look at all frameworks (there are so many of
them!), so we would like to suggest that you investigate how your
framework of choice does it.  While you're at it, of course you'll
need to know how to *properly* fix this issue, so let's finish up by
explaining that.

How to correctly quote identifiers
----------------------------------

It turns out that quoting *identifiers* in plain ANSI SQL under an
ASCII character encoding is as simple as quoting *values* in ANSI SQL
with ASCII. As you may know, values must be wrapped in single quotes,
with embedded single quotes doubled.  For identifiers, the same
happens, but with double quotes.  So if the *value* ```O'Neill``` gets
quoted as ```'O''Neill'```, the *identifier* ```This is the "simplest"
thing``` gets quoted as ```"This is the ""simplest"" thing"```.

Unfortunately, there are some complications.  As you may know,
escaping *values* should be done via
[```PQescapeStringConn()```](http://www.postgresql.org/docs/current/interactive/libpq-exec.html#LIBPQ-PQESCAPESTRINGCONN)
or
[```mysql_real_escape_string()```](http://dev.mysql.com/doc/refman/5.6/en/mysql-real-escape-string.html)
and **not** via the deprecated
[```PQescapeString()```](http://www.postgresql.org/docs/current/interactive/libpq-exec.html#LIBPQ-PQESCAPESTRING)
or
[```mysql_escape_string()```](http://dev.mysql.com/doc/refman/5.6/en/mysql-escape-string.html).
The reason is that you can't treat strings in multi-byte character
encodings one byte at a time, because that way you could be blindly
replacing bytes *within a character*: after reinterpretation of those
bytes as characters in the chosen encoding, your carefully quoted
string [might suddenly not be quoted
anymore](http://shiflett.org/blog/2006/jan/addslashes-versus-mysql-real-escape-string).
The database client library knows which encoding is being used to
communicate with the server, so it knows how to treat multi-byte
characters correctly.

To this end, PostgreSQL's ```libpq``` offers the
[```PQescapeIdentifier()```](http://www.postgresql.org/docs/current/static/libpq-exec.html#LIBPQ-PQESCAPEIDENTIFIER)
function.  Unfortunately, neither MySQL's nor SQLite's client
libraries offer a function for escaping identifiers.  In the corporate
enterprise(TM) world, it gets worse: in the Oracle database it
apparently is **impossible** to safely quote identifiers!  The
documentation states ["neither quoted nor nonquoted identifiers can
contain double quotation marks or the null character
(\0)."](http://docs.oracle.com/cd/E16655_01/server.121/e17209/sql_elements008.htm).
Microsoft SQL Server has two ways of escaping, the regular ANSI way
(which, like with MySQL, is only enabled if an option is set) and its
proprietary way of wrapping the identifier in square brackets.  The
[documentation](http://msdn.microsoft.com/en-us/library/ms175874%28v=sql.120%29.aspx)
doesn't mention it, but you can escape square brackets within
identifiers by doubling the *closing* bracket.  This can be gleaned
from the documentation on the [in-database identifier quotation
function](http://technet.microsoft.com/en-us/library/ms176114.aspx),
which doesn't seem to be available in client libraries.

Because most DB client libraries have no identifier quotation
function, most DB-agnostic abstractions don't offer it either.  This
includes PDO, which is currently the most popular PHP database
abstraction library.  By offering only the lowest common denominator,
the power to correctly quote identifiers is taken away.  Instead, the
library could emulate identifier escaping if it enforces a sane
character encoding, and throw an exception in cases where a special
character occurs in the string and escaping it is truly impossible
(for example when quoting for the Oracle database).

Luckily, it turns out that in practice, multi-byte encodings are
rarely used.  The most used encoding is probably UTF-8, which also
happens to be somewhat self-correcting, so the multi-byte issue might
not be that big of a deal, especially because the encoding can usually
not be changed by an attacker.  So in practice, escaping the quotation
ASCII character at the byte level is "good enough".

Since most modern code uses parameterised queries or prepared
statements, the risk of SQL injection is much lower than it used to
be.  Unfortunately, you can only pass values as parameters; there is
no provision for parameterising identifiers, which would solve the
problem more thoroughly.

All in all, there's still some work to be done by the database
vendors.  The underlying problem, as I've argued
[before](http://www.more-magic.net/posts/structurally-fixing-injection-bugs.html),
is that we're trying to build up strings containing statements of a
full-fledged query language, with embedded user input.  If the
database vendors provided a way to bypass SQL strings, and instead
provided direct access to the underlying abstract syntax tree, all SQL
injection problems would disappear in one fell swoop.
