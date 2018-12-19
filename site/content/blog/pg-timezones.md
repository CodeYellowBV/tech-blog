+++
title = "Subtleties of time zone calculations in Postgres"
author = "Peter Bex"
slug = "pg-timezones"
date = "2018-12-19"
description = ""
aliases = ["/pg-timezones.html"]
keywords = ["Development", "postgresql", "timezones"]
tags = ["Development", "postgresql", "timezones"]
+++


As any programmer well knows, time zones can be a real pain.  They seem
almost invented just to make programmers' lives more difficult!
Luckily, if your tools are good, a lot of pain can be spared.

At Code Yellow, we work a lot with
[PostgreSQL](https://www.postgresql.org/).  This really has time zones
__handled__.  Imagine my surprise when a customer opened a ticket that
they got unexpected dates in their CSV files.

## Short problem description

Let's first start with a short overview of the problem, so we
understand what's going on.  One of our projects involves accepting
subscriptions for newspapers.

There are rules for when a subscription may start.  First of all, the
future subscriber can decide when their subscription starts.  For
example, they can ask for the subscription to be started two months
from now and we have to honor that.  However, it takes time to process
subscriptions.  So if I subscribe today and ask for my paper to be
delivered tomorrow, that's not going to work.  Our customer wants the
start date to begin on the first 

In principle, we want payment for subscriptions to start in the same
month the paper starts to be delivered.  To make things more
difficult, payments can only start on specific dates in the month.
Payments in the first half of the month (the 1st until the 15th) are
allowed.  If a payment date would fall between the 16th and the last
day of the month, the payment should start on the 1st of the next
month.  Of course we only want payments to start **after** the
subscription start date.

In pseudo-code, we have the following:

```pascal
first_delivery_date := max(date_of_registration + interval('1 week'), requested_date)

IF first_delivery_date.day_of_month() >= 16 THEN
    first_payment_date := date_truncate('month', first_delivery_date + interval('1 month'))
ELSE
    first_payment_date := first_delivery_date
ENDIF
```

Each of our customer's clients have different requirements of how
their files are formatted.  This is why this application has an
interface to build export files in a sort of graphical DSL which gets
compiled down to SQL.  The code generates quite efficient queries,
which get streamed straight to multiple files using server-side
cursors.  We can stream to multiple files from the same result set but
different filters can be applied to each file, and each file can be in
a different format (CSV, XML, JSON, ...).  This code has been
battle-tested and is very robust by now.  Only the user interface for
building these definitions could use some love...

Anyway, we store all our time stamps in the database with time zones
(i.e., [`timestamptz`](https://www.postgresql.org/docs/current/datatype-datetime.html)).
That is best practice and also what Django automatically uses for its
[`DateTimeField`](https://docs.djangoproject.com/en/2.1/ref/models/fields/#django.db.models.DateTimeField).
Furthermore, as far as I've been able to tell Django forces all
datetime input and output to be in UTC for consistency reasons.  This
should not technically be needed; if you do it correctly, Postgres
will always output the time zone information in result sets.

In the bug report, the subscription's first delivery date was the 24th
of November.  Because that date falls after the 15th of the month, the
first payment date should be the 1st of December.  In the export the
date appeared as the 31st of November, which is clearly wrong.


## Expected value differs between development and production

I studied the definition for a bit, and could not find fault with it.
The export code has extensive debugging built-in, so I also studied
the query.  These queries can be monstrously intimidating, if you
aren't familiar with code generation.

In the actual export, a lot was going on; a `CASE` statement to see if
the target date was after the 15th or not, a `date_trunc` call to drop
the day of month and then an addition of an interval of one month to
fast forward to the next month's first day.  After whittling it down
to simplify debugging, the test case I came up with looked like this:

```sql
SELECT to_char(('2018-10-30 23:00:00Z'::timestamptz + 'P1M'::interval)
               AT TIME ZONE 'Europe/Amsterdam', 'dd-mm-yyyy');
```

To make life more interesting (because time zones aren't difficult
enough already), in my Vagrant box the output of this was
`01-12-2018`, which is exactly as expected.  On the live server, the
output was `30-11-2018`.

Like I already said, as far as I can tell, Django forces the client
connection's time zone to be `UTC`, so I was surprised to see a
difference on such a simple query!  In fact, if you try to explicitly
invoke `SET TIME ZONE 'Europe/Amsterdam'`, for example, Django will
issue an assertion failure which complains the connection is not in
UTC.

Now, there is a PostgreSQL setting that influences the default time
zone to be used for input and output, which is simply called
[`timezone`](https://www.postgresql.org/docs/current/runtime-config-client.html#RUNTIME-CONFIG-CLIENT-FORMAT).
On Debian, you can find it in
`/etc/postgresql/9.6/main/postgresql.conf`.  The value of this setting
gets determined when the cluster is initialised.  For unknown reasons,
it was set to `localtime` on the server and to `UTC` in my Vagrant
box.

Running the above query from `psql` using either a `timezone` setting
of `UTC` or issuing `SET TIME ZONE 'UTC'` beforehand will result in
the expected output of `01-12-2018`.  If the time zone was `localtime`
or `Europe/Amsterdam`, the output was `31-11-2018`.

## Root cause analysis

To understand a bit better what is going on, we need to dig in a bit
more.

Even though the `timestamptz` column stores a time zone with the
time stamp, the output of the query will **always** be echoed back in
the selected current time zone:

```sql
psql=> SET TIME ZONE 'Europe/Amsterdam';
psql=> CREATE TEMPORARY TABLE foo ( dt timestamptz );
psql=> INSERT INTO foo VALUES
       ('2018-10-31T00:00:00+01'), --- entered in Europe/Amsterdam format
       ('2018-10-31T00:00:00Z');   --- entered in UTC format (one hour earlier)
INSERT 0 2
psql=> SELECT dt
       FROM foo;
           dt
------------------------
 2018-10-31 00:00:00+01
 2018-10-31 01:00:00+01
(2 rows)
psql=> SELECT to_char(dt, 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 31-10-2018
 31-10-2018
(2 rows)
psql=> SET TIME ZONE 'UTC';
psql=> SELECT dt
       FROM foo;
           dt
------------------------
 2018-10-30 23:00:00+00
 2018-10-31 00:00:00+00
(2 rows)
psql=> SELECT to_char(dt, 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 30-10-2017
 31-10-2018
(2 rows)
```

We ran into this before, so when we know we will emit a `to_char`
call, we require the user to state the time zone in which the output
should be formatted.  This causes the compiled query to look like
this:

```sql
psql=> SELECT to_char(dt AT TIME ZONE 'Europe/Amsterdam', 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 31-10-2018
 31-10-2018
(2 rows)
```

The output of this is the same, regardless of the selected output
time zone in Postgres.  When you use `AT TIME ZONE $tz`, one of two
things will happen:

If you have a `timestamptz`, it will be stripped of time zone and
become a time zone-unaware `timestamp`, but it will store the time that
was in effect in time zone `$tz`.

If you have a time zone-unaware `timestamp`, it will be interpreted as
that time in time zone `$tz` and cast into a time zone-aware
`timestamptz`.  But when displayed, it will still output using the
session time zone, **not** `$tz`.


## Date calculations are zone-dependent

The current time zone is more than just an __output__ format.  It
turns out that __calculations__ are also done in the current time zone
(but only on time zone-aware time stamps).  You might think that this
should not matter, but it does!

I already knew that for an operation like `date_trunc`, the truncation
needs to take care of the time zone, because truncating the time stamp
`2018-01-01T00:00:00+01` in **Europe/Amsterdam** to the month gives
you simply the same time stamp.  In **UTC**, however, the time stamp
reads as `2017-12-31T23:00:00Z`, and truncating it to the month will
yield `2017-12-01T00:00:00Z` which differs by a whole month and
doesn't even fall in the same year!

So let's say you want to truncate a given datetime in **UTC** but then
display it in **Europe/Amsterdam** (not very common, but it could
happen), you can do it like this:

```sql
psql=> SET TIME ZONE 'UTC';
pqsl=> SELECT to_char((date_trunc('month', dt AT TIME ZONE 'UTC') AT TIME ZONE 'UTC')
                       AT TIME ZONE 'Europe/Amsterdam',
                       'dd-mm-yyyy')
       FROM foo;
  to_char
-----------
 01-10-2018
 01-10-2018
(2 rows)
psql=> SET TIME ZONE 'Europe/Amsterdam';
pqsl=> SELECT to_char((date_trunc('month', dt AT TIME ZONE 'UTC') AT TIME ZONE 'UTC')
                       AT TIME ZONE 'Europe/Amsterdam',
                       'dd-mm-yyyy')
       FROM foo;
  to_char
-----------
 01-10-2018
 01-10-2018
(2 rows)
```

With three zone conversions, this looks overly complex.  But like I
said, this is generated SQL, which needs to ensure that each output
expression is consistently cast back to a zone-aware time stamp before
(maybe) being fed into an outer expression.  If you don't do this,
you'll get inconsistent results depending on your session's time zone:

```sql
psql=> SET TIME ZONE 'UTC';
pqsl=> SELECT to_char(date_trunc('month', dt) AT TIME ZONE 'UTC', 'dd-mm-yyyy')
       FROM foo;
  to_char
-----------
 01-10-2018
 01-10-2018

(2 rows)
psql=> SET TIME ZONE 'Europe/Amsterdam';
pqsl=> SELECT to_char(date_trunc('month', dt) AT TIME ZONE 'UTC', 'dd-mm-yyyy')
       FROM foo;
  to_char
-----------
 30-09-2018
 30-09-2018
(2 rows)
```

## Really, ALL date calculations are zone-dependent!

This finally offers an explanation of the code snippet we started with:

```sql
psql=> SET TIME ZONE 'UTC';
psql=> SELECT to_char((dt + 'P1M'::interval) AT TIME ZONE 'Europe/Amsterdam', 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 01-12-2018
 30-11-2018
(2 rows)
psql=> SET TIME ZONE 'Europe/Amsterdam';
psql=> SELECT to_char((dt + 'P1M'::interval) AT TIME ZONE 'Europe/Amsterdam', 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 30-11-2018
 30-11-2018
(2 rows)
```

The reason the UTC one displays two different values is as follows:

- The first time stamp is `2018-10-31 00:00:00` in `Europe/Amsterdam`.  When
  interpreted in UTC, this is `2018-10-30 23:00:00` (the previous day!).
- The second time stamp is `2018-10-30 23:00:00` in `Europe/Amsterdam`. When
  interpreted in UTC, that is `2018-10-30 22:00:00` (the same day).

Keep this in mind!  Let's continue with what happens in `Europe/Amsterdam`:

- When you add a month to `2018-10-31 00:00:00`, you get `2018-11-30 00:00:00`.
  Formatting only the date gives you `2018-11-30`.
- When you add a month to `2018-10-30 23:00:00`, you get `2018-11-30 23:00:00`.
  Formatting only the date gives you `2018-11-30`.

What happens in `UTC`:

- When you add a month to `2018-10-30 23:00:00`, you get `2018-11-30 23:00:00`.
  Formatting only the date in `UTC` would give you `2018-11-30`, but we do it in
  Europe/Amsterdam, so we get `2018-11-31 00:00:00` and formatting as a date
  that gives us `2018-11-31`.
- When you add a month to `2018-10-30 22:00:00`, you get `2018-11-30 22:00:00`.
  Formatting only the date in `UTC` would give you `2018-11-30`.  We do it in
  Europe/Amsterdam, but this is still the "same" date at `2018-11-30 23:00:00`,
  so the result is `2018-11-30`.

So, it's rather simple actually; formatting is a form of truncation,
but it's the **calculation** that introduces the unexpected value
differences.  It's a bit like rounding errors in floating point.  They
stack up the more operations you do, and weird results can pop up at
unexpected points.


## Solution

The solution to this problem was simple enough: we simply do the
addition after stripping time zone info and then later restore time
zone info:

```sql
psql=> SET TIME ZONE 'UTC';
psql=> SELECT to_char(((dt AT TIME ZONE 'Europe/Amsterdam' + 'P1M'::interval) 
                       AT TIME ZONE 'Europe/Amsterdam')
					   AT TIME ZONE 'Europe/Amsterdam', 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 30-11-2018
 30-11-2018
(2 rows)
psql=> SET TIME ZONE 'Europe/Amsterdam';
psql=> SELECT to_char(((dt AT TIME ZONE 'Europe/Amsterdam' + 'P1M'::interval) 
                       AT TIME ZONE 'Europe/Amsterdam')
					   AT TIME ZONE 'Europe/Amsterdam', 'dd-mm-yyyy')
       FROM foo;
  to_char
------------
 30-11-2018
 30-11-2018
(2 rows)
```

Consistency at last!  It looks unwieldy because of the three casts,
but conceptually it's the same as what we do with `date_trunc`.

In hindsight, it was rather obvious to me in the case of `date_trunc`.
I simply didn't realise that simple arithmetic with time stamps and
intervals also needs to strip and restore time zone information.

I think this can be simplified if you only strip and restore time
zones on input and output.  This works if you're always interested in
one time zone for everything.  However, if you want to do your
arithmetic in time zone A and output in time zone B, I think there is
no way around converting back and forth.

So far there hasn't been a need to convert between different time
zones, but the code has "accidentally" grown in this flexible
direction.  This happend as I explored the complexity of time stamp
manipulations.  It was only later that I realised that it can be
simplified by doing the conversions at strategic points.  This
realisation happened while I was working on this latest bug.

It is still tricky though!  Even writing this blog post was harder
than it should be.  I already (though I) understood the problem, but
coming up with good examples that illustrate the issue was **hard**.
