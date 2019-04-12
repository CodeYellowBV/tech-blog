+++
title = "How to use streaming replication with PostgreSQL"
author = "Peter Bex"
slug = "pg-replication"
date = "2019-04-12"
description = ""
aliases = ["/pg-replication.html"]
keywords = ["Development", "postgresql", "replication", "WAL", "backup"]
tags = ["Development", "postgresql", "replication", "WAL", "backup"]
+++

In this post, I want to explain how to set up streaming replication
for PostgreSQL.  We want to be able to do (read-only) reporting on
live data, but reduce load on the main database server.

Some background: We offer [Metabase](https://www.metabase.com/) to
many of our customers for data analysis and reporting.  This means we
don't have to hand-code everything into our application.  It even
allows savvy customers to write their own reports.  We set up a
read-only PostgreSQL user on the production server.  Metabase connects
with the database as this user over an SSH-tunnelled PostgreSQL port.

For most of our systems, this setup is just fine.  A few of our
clients have so many users that the databases are in constant use.
Some reports can be quite heavy, but they should not affect
operations.  So, we decided to replicate these databases to our
Metabase servers.  This allows us to query the local copy of the
database, reducing system load on production.  We currently rely on
`pg_dump` for backups, which is getting a bit slow too.  We are
considering replication for backups, too.  When the database becomes
even more unwieldy we could use this for load-balancing, too.

## Documentation overload

In general, the [PostgreSQL documentation](https://www.postgresql.org/docs/current/)
is to the point, clear and even usable as a tutorial.  For replication though,
I found the docs overwhelming and even a little confusing.

There are two chapters of interest: The first is under "backup", called
[Continuous Archiving and Point-in-Time Recovery (PITR)](https://www.postgresql.org/docs/11/continuous-archiving.html).
The next chapter is [High Availability, Load Balancing and Replication](https://www.postgresql.org/docs/11/high-availability.html).
Both chapters are a bit heavy on terminology, but for the most
part, I was just overwhelmed by all the options.

The backup chapter starts out by explaining manual WAL archiving.  You
generally *don't* want that.  Streaming replication with replication
slots was added later to the docs, as the feature is newer.  This is
what we're going to use.

### Untangling the confusion

Let's start by demystifying some terminology.  First of all, we want
to *replicate* the database.  The 
[definition of replication](https://www.merriam-webster.com/dictionary/replication)
is: "copy, reproduction".  That's exactly what this is: we run a second
server which will be an identical copy of the original database server.
As you can see from [the documentation](https://www.postgresql.org/docs/11/different-replication-solutions.html),
there are multiple solutions to perform replication.  I won't go into all
of them.  The one we're concerned with is *Write-Ahead Log Shipping*.

Postgres uses the *Write-Ahead Log* (or *WAL*) internally to guarantee
consistency and prevent data loss.  For a detailed explanation, see
[the chapter on reliability](https://www.postgresql.org/docs/11/wal.html).
Normally, Postgres keeps only a few WAL "segments", enough to replay the log
after a power failure or crash.  It uses these to restore consistency, much
like logged file systems such as `ext3` do.  If the database kept the entire
log since first boot, you could recreate the database from scratch up to the
current state.  WAL replication is based on this idea.

Log shipping can be done in two ways: the oldest one is to set up
hooks in Postgres to copy ("ship" or "archive") the WAL log files to
another server using, for example, rsync or scp.  This is what the 
[WAL Archiving](https://www.postgresql.org/docs/11/continuous-archiving.html#BACKUP-ARCHIVING-WAL)
section in the manual is all about.  To me, this feels somewhat
fiddly, like a duct tape solution.

The original database server is called the *master* or *primary*.  The
one that runs the copy is called the *standby* or *secondary*.  If you
use *streaming replication*, the standby server uses the streaming
replication protocol inside Postgres instead of manual file copying,
which makes things simpler.  However, with this mechanism you may miss
WAL segments if the primary starts deleting the corresponding files.

To prevent this deletion of logs, Postgres offers *replication slots*.
A replication slot is a record in the database with a name and a "last
seen WAL segment" (plus more, but none of that matters to us).  Once
all of the replication slots have seen a WAL segment, Postgres can
delete it.


## How to set it up

Once you know what to do, it is straightforward to set up replication.
Let's start by creating a read-only user on the primary (but please,
pick a good password!):

```sh
primary$ sudo -u postgres psql -c "CREATE USER readonly WITH password 'test123'"
```

Now, we can create a replication cluster on the secondary.  For
streaming replication, the versions of PostgreSQL **must** match
between primary and secondary.  The examples assume Postgres 11;
change the number if you are on a different version:

```sh
secondary$ sudo -u postgres pg_createcluster 11 replica

.... output ....

Success. You can now start the database server using:

    /usr/lib/postgresql/11/bin/pg_ctl -D /var/lib/postgresql/11/replica -l logfile start

Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory                 Log file
11  replica 5434 down   postgres /var/lib/postgresql/11/replica /var/log/postgresql/postgresql-11-replica.log

secondary$ sudo systemctl daemon-reload
```

Now, we're going to do something somewhat strange; we'll delete the
database files in this cluster (which is currently not running):

```sh
secondary$ rm -rf /var/lib/postgresql/11/replica
```

Use whatever directory the `pg_createcluster` command printed if you
don't have it in `/var/lib/postgresql`.

After deleting the files, we start by mirroring the existing database
from the primary using [`pg_basebackup`](https://www.postgresql.org/docs/11/app-pgbasebackup.html):

```sh
secondary$ sudo -u postgres pg_basebackup -D /var/lib/postgresql/11/replica -U readonly -h primary -PRCS my_replication_slot
Password: 
 2211288/13584399 kB (16%), 0/1 tablespace
```

The `-C -S my_replication_slot` options tell it to create and use the
replication slot called `my_replication_slot`.  This means that no
matter how long it takes us to get the secondary server up and
running, the primary will hold on to WAL segments for us in the mean
time.

Also, be sure to supply the correct connection settings: `-h primary`
indicates the hostname, you probably need to change at least that
part.  Now wait until the counter reaches 100%.

After this, we can check out the configuration file which
`pg_basebackup` created (this is what the `-R` option does):

```sh
secondary$ sudo cat /var/lib/postgresql/11/replica/recovery.conf
primary_conninfo = 'user=readonly password=''test123'' host=primary port=5432 sslmode=prefer sslcompression=0 krbsrvname=postgres target_session_attrs=any'
primary_slot_name = 'my_replication_slot'
```

Looks good?  Then, let's start the secondary replication cluster!

```sh
secondary$ sudo systemctl start postgresql@11-replica.service
```

## Operational notes

The simplest way to check if the secondary is correctly processing WAL
events from the primary is to check that changes you made after
setting everything up are reflected in the secondary (ie, checking
MAX(id) from a table or something).

You can also the built-in [`pg_replication_slots` view](https://www.postgresql.org/docs/current/view-pg-replication-slots.html)
in Postgres to check the status of a replication slot:

```sh
primary$ psql -c "SELECT slot_name, slot_type, active, restart_lsn FROM pg_replication_slots;"
      slot_name      | slot_type | active | restart_lsn
---------------------+-----------+--------+--------------
 my_replication_slot | physical  | t      | 10D/DA000000
(1 row)
```

The `restart_lsn` column shows the oldest *Log Sequence Number* which
the secondary has seen.  To verify that the secondary is currently
connected, check that `active` is true.  To check that it is
processing the WAL entries, check that `restart_lsn` is changing.

To verify the LSN values, you can check the primary's current log
location like so:

```sh
primary$ sudo -u postgres psql -c "select pg_current_wal_lsn()"
 pg_current_wal_lsn 
--------------------
 10D/E697D450
(1 row)

```

To get the latest LSN the secondary received (should always be the
same as `restart_lsn`):

```sh
secondary$ sudo -u postgres psql -c "SELECT pg_last_wal_receive_lsn()"
 pg_last_wal_receive_lsn 
-------------------------
 10D/E68501F0
(1 row)
```

To check the replication lag (in bytes) between the primary and
secondary, use `pg_wal_lsn_diff`:

```sh
primary$ sudo -u postgres psql -c "select pg_size_pretty(pg_wal_lsn_diff('10D/E697D450', '10D/E68501F0'))"
 pg_size_pretty 
----------------
 1205 kB
(1 row)
```

On the primary, you can also check the size of the
`/var/lib/postgresql/11/main/pg_wal` directory.  This is where WAL
segments are stored until they can be deleted.

If something does not work, always check the standard log file.  It
will contain error messages about missing WAL log files or when it
can't connect to the primary.

Remember (and this is **important**), when you take a secondary out of
commission, also delete its replication slot.  This slot blocks
deletion of WAL logs, so eventually your disk will fill up!  To delete
it:

```sh
primary$ sudo -u postgres psql -c "SELECT pg_drop_replication_slot('my_replication_slot')"
```

That's it. Enjoy your new replication server!
