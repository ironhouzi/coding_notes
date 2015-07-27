Notes taken for the Pycon 2015 talk: "PostgreSQL Proficiency for Python People", by Christophe Pettus.

### Just Do This.

* Always create databases as UTF-8.
    - Once created, cannot be changed.
    - Nightmare to convert from SQL ASCII.
* Don't use "C locale"!
* Verify system locale and UTF-8, which are defaults.

### Checksums

* Doesn't fix corruption, but will notify.
* Very small performance hit, so use it.
* initdb option (-k)
* Default is "off"!

### `pg_ctl`

* wrapper for starting postgres.

### Stopping PostgreSQL

* Three modes: smart, fast, immediate.
* Don't use smart, since you cannot log in while it is waiting for users to logout.
* Use fast (cancels queries, does shutdown)
* Immediate crashes PostgreSQL, which is OK, but recovery takes longer time.

### PostgreSQL directories

* Grepping for postgres/postmaster user in ps will reveal the path to the data dir.
* Data lives in "base".
* The transaction logs are in `pg_xlog` - don't remove these!

### Configuration files

* `postgresql.conf`
* `pg_hba.conf`

### Important configuration parameter classes

(Only configured on production, no need on dev)

* Logging.
* Memory.
* Checkpoints.
* Planner.

### Logging

* Be generous with logging, very low impact on the system.
* Use rigorous logging before other configuration, to know the system you're configuring.
* Can use syslog for dumping logs.
* Standard format to files is depricated.
* CSV is common.

Recommended settings:

```
log_destination = 'csvlog'
log_directory = 'pg_log'    # relative to `data` directory
logging_collector = on
log_filename = 'postgres-%Y-%m-%d_%H%M%S'
log_rotation_age = 1d
log_rotation_size = 1GB     # aggressive, but possible if doing perfomance analysis
log_min_duration_statement = 250ms  # log if statement takes longer than 250ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0          # log every temp file
```

### Memory configuration

#### `shared_buffers`

* Below 2GB, set to 20% of total system memory.
* Below 32GB, set to 25% of total system memory.
* Above 32GB, set to 8GB.

#### `work_mem`

Upper threshold for total memory allocation.

* Start low: 32-64MB.
* Look for `temporary file` lines in logs.
* Set to 2-3x the largest temp file you see.
* Can cause a huge speed-up if set properly.
* Be careful: It can use that amount
Every single sort operation, even if a single query needs more than one sort operation, can take 64MB - in theory there's no upper limit to how much memory can be used, but realistically this is hard to pull off. The point is to not over allocate memory.

#### `maintenance_work_mem`

* 10% of system memory, up to 1GB.
* Maybe even higher if you're having VACUUM problems.

#### `effective_cache_size`

* Set to the amount of file system cache available. (`free -m`)
* If you don't know, set it to 50% of total system memory.
* This isn't an allocation, it's a hint to the planner of how much stuff can be kept in memory total.
* Rule of thumb: If `effective_cache_size`, based on this setting, is smaller than your biggest index, you probably want more memory. (Look for your biggest index and you want enough memory to keep that whole thing in the file system cache)

### Checkpoints

* A complete flush of dirty shared buffers to disk.
* Potentially a lot of I/O.
* Done when the first of two thresholds are hit:
    1. A partial number of WAL (Write Ahead Log) segments have been written. (How much has been changed in the DB)
    2. A timeout occurs.

Recommended settings:

```
wal_buffers = 16MB  # Not default!
checkpoint_completion_target = 0.9
checkpoint_timeout = 10m-30m    # depends on restart time, don't use span, choose one time setting
checkpoint_segments = 32        # to start
```

The higher `checkpoint_timeout` is set, the longer a restart will take after a crash.

* Look for checkpoint entries in the logs.
* Are they happening more often than `checkpoint_timeout`?
    - Adjust `checkpoint_segments` so that checkpoints happen due to timeouts, rather than filling segments. (Checkpoints happening less frequently than checkpoint timeout)
    - Adjust means: doubling.
* The WAL can take up to 3 * 16MB * `checkpoint_segments` on disk. One reason to constrain `checkpoint_segments` is that you're low on disk.

### Planner settings.

The planner is what takes an SQL query and turns it into its internal format.

* `effective_io_concurrency` -- Set to the number of I/O channels; otherwise, ignore it. I.e: the nr of raid arrays, or the number of channels for the SSD.
* `random_page_cost` -- 3.0 for a typical RAID10 array, 2.0 for a SAN, 1.1 for Amazon EBS. Setting it too high can cause the planner to start ignoring indexes it should actually use. Default is 4, which is often too high.

### Do not touch!

* `fsync = on`
    - Never change this.
* `synchronous_commit = on` (when committing, there'll be a slight pause and all changes pushed to WAL to make ensure a replay, if a crash occurs. When commit returns, it's done and the transaction is committed to disk. If setting is `off`, the change is only queued and a crash could lead to lost data.
    - Change this, but only if you understand the data loss potential.


Changing settings

* Most settings just require a server reload to take effect.
* Some require a full server restart (such as `shared_buffers`).
* Many can be set on a per-session basis!

## `pg_hba.conf`

### Users and roles

* A "role" is a database object that can own other objects (tables, etc.) and that has privileges (able to write to a table).
* Roles are global, spanning all DB's in the system.
* A "user" is just a role that can log into the system; otherwise they're synonyms.
* You can create roles that cannot log in. (Django does not utilize this, but there are some Java applications that can)
* PostgreSQL's security model is based around users and roles.

### Basic user management.

* Don't use the "postgres" superuser for anything application related.
* Create an application user, grant it all the permissions it needs over the database.
* If you use migrations (like Django migrations), you sadly have to grant schema modification privileges to your application user.
* If you don't have to, don't. Use a separate user for the application and have a dedicated user for schema modifications.

### User security

* By default, database traffic is not encrypted.
* Turn on SSL if you're running in a cloud provider.

## The WAL (Write-Ahead Log)

* Building block for many other PostgreSQL technologies.
* Replication, crash recovery, etc..

### The basics

* When each transaction is committed, it is logged to the WAL.
* The changes in that transaction are flushed to disk. At this point the commit is complete.
* If the system crashes, the WAL is "replayed" to bring the database to a consistent state.

### A continuous record of changes

* The WAL is a continuous record of changes since the last checkpoint.
* If you have the disk image of the database and every WAL record since that was created, you can recreate the database to the end of the WAL.

### `pg_xlog`

* The WAL is stored in 16MB segments, which PostgreSQL calls "segments", in the `pg_xlog` directory.
* Don't mess with it!
* Records are automatically recycled when they are no longer required.

### WAL archiving

* `archive_command`
* Runs a command each time a WAL segment is complete.
* This command can do whatever you want.
* What you want is to move the WAL segment to someplace safe, on a different system.

### `pg_dump`

* Built-in dump-restore tool.
* takes a logical snapshot of the database.
* Does not lock the database or prevent writes to disk.
* Low, but not zero load on the database.

### `pg_restore`

* Not a fast operation.
* Great for simple backups, not suitable for fast recovery from major failures.

### `pg_dump` / `pg_restore` advice

* `pg_dump` only backs up a single DB at a time.
* Back up globals with `pg_dumpall --globals-only`.
* Back up each database with `pg_dump` using `--format=custom`.
* Two types of dump:
    1. Raw: SQL format. (Generally not useful!)
    2. Custom: Gives parallell restores and compresses data, using `pg_restore`.

### `pg_restore`

* Restore using `--jobs=<# of cores + 1>`.
* Most of the time in a restore is spent rebuilding indexes; this will parallelize that operation.

### PITR (Point In Time Recovery) backup / recovery

* If you take a snapshot or copy of the data directory, it will not be consistent, but with the archived WAL records, a consistent restore is possible.
* Decide where the WAL segments and the backups will live.
* Configure `archive_command` properly to do the copying.

## Creating a PITR backup

* Issue this command: `SELECT pg_start_backup(...);`. This only marks the beginning of a backup, by creating a checkpoint.
* Copy the disk image and any WAL files that are created.
* Issue the command: `SELECT pg_stop_backup();`. Creates a new checkpoint.
* Make sure you have all the WAL segments.
* The disk image + WAL segments are your backup.

### WAL-E

* Provides a full set of appropriate scripting.
* Automates create PITR backups into AWS S3.
* Highly recommended!

### PITR Restore

* Copy the disk image back to where you need it.
* Set up recovery.conf (top level inside `pg_data`) to point to where the WAL files are.
* Start up PostgreSQL and let it recover.

### How long will this take?

* The more WAL files, the longer it will take.
* Generally takes 10-20% of the time it took to create the WAL files in the first place.
* The more frequent snapshots = faster recovery time.

### "PITR"?

* You don't have to replay the entire WAL stream.
* It can be stopped at a particular timestamp, or transaction ID.
* Very handy for application level problems.

### Replication

* What if we sent the WAL directly to another server?
* We could have that server keep up to date with the primary server.
* This is how PostgreSQL replication works.

### WAL Archiving

* Each 16MB segment is sent to the secondary when complete.
* The secondary reads it and applies it to its copy.
* Make sure the WAL file copied atomically.
* Use rsync, WAL-E, etc, not scp, since it's not atomic.

### Streaming Replication Basics

* The secondary connects via a standard PostgreSQL connection to the primary.
* As changes happen on the primary, they are semt to the secondary.
* The secondary applies them to its local copy of the database.

### `recovery.conf`

* All replication is orchestrated through the `recovery.conf` file.
* Always lives in your data directory.
* Controls how to connect to the primary, how far to recover for PITR, etc..
* Also used if you are bringing the server after a crash as a PITR recovery instead of replication.

### Disaster recovery

* Always have a disaster recovery strategy.
* What if your data center / AWS region goes down?
* Have a plan for recovery from a remote site.
* WAL archiving is a great way to handle that.

### `pg_basebackup`

* Utility for doing a snapshot of a running server.
* Easiest  way to take a snapshot to start a new recovery.
* Masters do not keep track of their secondaries, so if a secondary goes down for long enough time, the primary might have removed the WAL segments that the secondary might ask for. 9.4 has new features to allow a primary to know about its secondaries: "replication slots", but these will fill up your disk if a secondary goes down forever!
* Can also be used as an archival backup.

### Replication, the good.

* Easy to set up.
* Schema changes are automatically replicated.
* Secondary can be used to handle RO queries for lead balancing.
* Very few gotchas; it either works or it doesn't and it's vocal about not working.

### Replication, the bad.

* Entire databse or none of it.
* No writes of any kind to the secondary.
    - This includes temporary tables.
* Some things aren't replicated.
    - Temporary tables, unlogged tables.

## Advice

* Start with WAL-E
    - The README tells you everything you need to know.
* Handles a very large number of complex replication problems easily.
* As you scale out of it, you'll have the relevant experience.

### Trigger based replication

* Installs triggers on tables on master.
* A daemon process picks up the changes and applies them to the secondaries.
* Third-party add-ons to PostgreSQL.
* Highly configurable.
* Can push all or part of the tables; don't have to replicate everything.
* Multi-master setups possible (Bucardo).
* Complicated to set up.
* Schema changes must be pushed out manually.
* Imposes overhead on the master.
* 9.4 has a framework for doing logical replication directly in the PostgreSQL core.
    - No triggers.
    - Need C programming skills.

## Transactions, MVCC (Multiversion Concurrency Control) and VACUUM

### Transaction

* A unit of which must be:
    - Applied atomically to the DB.
    - Invisible to other DB clients until it is committed.
    - e.g:

``` sql
BEGIN;
INSERT INTO transactions(account_id, value, offset_id)
    VALUES (11, 120.00, 14);
INSERT INTO transactions(account_id, value, offset_id)
    VALUES (14, -120.00, 11);
COMMIT;
```

* Once the COMMIT completes, the data has been written to permanent storage.
* If a database crash occurs, any transactions will be COMMITed or not; no half-done transactions.
* In PostgreSQL _everything_ runs inside of a transaction.
* If no explicit transaction, each statement is wrapped in a transaction for you.
    - This has certain consequences for database-modifying functions.
* Everything that modifies the databases are transactional, even schema changes.

### Warning

* Many resources are held until the end of a transaction.
    - Temporary tables, working memory, locks, etc.
* Keep transactions brief and to the point.
* Be aware of IDLE IN TRANSACTION sessions. Don't start a transaction in your console and walk away.

### Challenges

* How to handle concurrency, when two sessions are trying to use the same data?
    - Process 1 begins a transaction.
    - Process 2 begins a transaction.
    - Process 1 updates a tuple (row).
    - Process 2 tries to read that tuple.
* What would happen?
* Process 2 can't get the new version of the tuple since ACID generally prohibits dirty reads.
* Where does it get the old version of the tuple from?
    - Lock whole DB?
    - Lock the table?
    - Lock the tuple?

### Multi-Version Concurrency Control

* Creates multiple "versions" of the database.
* Each transaction only sees its own "version", called a "snapshot" in PostgreSQL.
* Snapshots are a first-class memeber of the database
    - There is no privileged "real" snapshot.

#### Implications

* Readers don't block readers.
* Readers don't block writers.
* Writers don't block readers.
* Writers only block writers to the same tuple.

#### Snapshots

* Logically, it is the set of all transactions that have committed at a particular point in time.
* You can even manipulate snapshots, by saving/loading them, though this is quite advanced.
* Each transaction maintains its own snapshot of the database.
* This snapshot is created when a statement or transaction starts, depending on the transaction isolation mode.
* The client only sees the changes in its own snapshot.

#### Limitations

* PostgreSQL will not allow two snapshots to "fork" the database.
* If this happens, it resolves the conflict with locking or with an error, depending on the isolation mode.

#### Isolation modes

* READ COMMITTED, the default.
* REPEATABLE READ.
* SERIALIZABLE.
* Dirty reads are not supported.

#### Snapshot issuing

* In READ COMMITTED, each statement starts its own snapshot, it sees anything that has committed since the last statement.
* If it attempts to update a tuple another transaction has touched, it blocks until the transaction commits.

#### Higher isolation modes
If you want a to use the DB as if you have a private copy of the database.

* REPEATABLE READ and SERIALIZABLE take the snapshot when the transaction begins.
* Snapshot lasts until the end.
* An attempt to modify a tuple another transaction has changed blocks and returns an error if that transaction commits.
* When the illusion of a perfect snapshot cannot be maintained, an error is thrown and the application can retry the transaction against the new, updated snapshot.

#### SERIALIZABLE

* Not every "conflict" can be detected at the single tuple level.
    - INSERTing calculated values.
* SERIALIZABLE detects these using predicate locking.
    - Requires some extra overhaed, but is remarkably efficient.
* Recommended isolation mode for a new application.

### MVCC consequences

* Deleted tuples are not usually freed immediately.
    - Tuples on disk might not be able to be readily checked.
* This results in dead tuples in the database.
* Which means: VACUUM!

### VACUUM

* VACUUM's primary job is to scavenge tuples that are no longer visible to any transactions.
* They are returned to the free space for reuse.
* `autovacuum` generally handles this for you without intervention.

#### ANALYZE

* The planner requires statistics on each table to make good guesses for how to execute queries.
* ANALYZE collects these statistics.
* This is done as part of VACUUM.
* Always do it after major database change -- especially a restore from a backup restore.

### VACUUM behavior

* VACUUM is always working.
* The DB generally stabilize at 20% to 50% bloat. That's acceptable.
* If you see autovacuum workers running, that's generally not a problem.
* VACUUM can be blocked by long-running transactions, or "idle-in-transaction" sessions.
* Manual table locking can block VACUUM.
* Very high write-rate (DELETEs and UPDATEs) tables will trigger a VACUUM.
* Many, many tables (10 000 or more) is problematic for VACUUM.

### Fixing VACUUM performance

* Reduce the autovacuum sleep time.
* Increase the number of autovacuum workers.
* Do low period manual VACUUMs.

## Schema Design

* Normalization is important, but don't obsess. It flows naturally from proper separation of data.
* Normalization is basically having things in one place only.
* Pick "Entities"
    - An entity is the top level logical object in your data model, e.g. Customer, Order, InventoryItem. Create tables for these.
    - Flow down from there to subsidiary items.
* Make sure that no entity-level information gets pushed into the subsidiary items.

### Joins

* PostgreSQL executes joins very efficiently.
* Don't worry about joins, specially joining large tables with small tables.
* Use PostgreSQL's type system.
* Use domains to create custom types.
* Don't have polymorphic fields: fields whose interpretation is dependent on another field.
    - E.g: ContactInfo field depends on ContactType field is bad. Have explicit PhoneField, EmailField, etc..
* Don't use fields which use strings to store multiple types.

### Constraints

* Use them. They're cheap and fast.

### Key selection

* SERIAL is convenient and straight forward but ..
    - What happens if you merge two tables?
* Use natural keys in preference to synthetic keys if you possibly can.
* Consider UUIDs instead of serials and synthetic keys.

### Don't OOP your schema

* Table hierarchies with "base classes" being factored out looks normalized, but are painful to use.

### The fast / slow rule

* If a table has a frequently updated section and a slowly updated section, consider splitting these between the fast and the slow with a 1:1 relationship.
* It keeps foreign key locking under control.
    - Every time you change that fast moving data, you're taking a lock on that row and FK's can sometimes hit the lock and create pile-ups.

### Arrays

* First class type in PostgreSQL.
* Can be searched and indexed.
* Often a good substitute to a subsidiary table: e.g. Storing "likes" would create an enormous many-to-many relationship, which would be too much to handle for a normal relational style DB.

### hstore

* Much, much better than an EAV schema.
* Great for optional, variable attributes.
* Can be indexed and searched.
* Don't use it as a replacement for schema modification!

### JSON

* A core type.
* Two types
    - json: pure text representation.
    - jsonb: parsed binary representation.

### The troubles of NULL

* Only use it to mean "missing value".
* Never, ever have it as a meaningful value in a key field.
* `WHERE NOT IN (SELECT ...)` will return NULL, which can be very surprising.

### Very Large Objects (1MB or more)

* Store them in files, store metadata in the database.
* The database is not designed for passing large objects around.

### Many-to-Many Tables

* Can get extremely large.
* Consider replacing with array fields; Either one way, or both directions.
* Can use a trigger to maintain integrity.
* Much smaller and more efficient.

### Time Representation

* _Always_ use TIMESTAMPTZ, TIMESTAMP is a bad idea.
* TIMESTAMPTZ is timestamp converted to UTC.
* Cannot know what timezone is used for TIMESTAMP.

### Indexing

... skipped

## Query optimization

* EXPLAIN or EXPLAIN ANALYZE
* Both take the SQL query string as an argument. Former only shows the planner works, latter performs the query.
* http://explain.depesz.com
* The output is cryptic;
    - Tree structure, each arrow is a level in the tree.
    - Each level runs a machine that feeds the level above.
    - Read the execution plan from the bottom up.
    - Look for nodes that are processing a lot of data; especially if the data set is being reduced considerably on the way up.

### Costs

* Measured in arbitrary units.
* First number is the startup cost for the first tuple, second is the total cost.
* Comparable with other plans using the same planner configuration parameter.
* Costs are estimates, what the planner estimates the operations to take.
* Big jumps in costs denotes something expensive.
* Costs are inclusive of subnodes.

... skipped

## `psycopg2`

* The result set of a query is loaded into client memory when the query completes, regardless of its size.
* Use named cursors if you want to iterate through the results.

## Django notes

* If using 1.6 or higher, always use the `@atomic` decorator.
* Cluster write operations into small transactions, leave read operations outside.
* Do all your writes at the very end of the view function.
* Multi-DB works very nicely with hot standby; Point the writes at the primary and the reads at the secondary.
* Sloppy transaction management can cause the dreaded Django idle-in-transaction problem.

## Special situations

### Minor version upgrade

* Do this promptly!
* Only requires installing new binaries apt-get update..

### Major version upgrade

* Requires planning.
* `pg_upgrade` is reliable.
* Trigger-based replication is another option for zero downtime.
* A full `pg_dump` / `pg_restore` is always safest, if practical.
* Always read the release notes!

### Bulk loading data

* Use COPY, not INSERT.
* `psycopg2` has a very nice COPY interface.
* COPY does full integrity checking and trigger processing.
* Do a VACUUM afterwards.

### Very high insert rates

* Reduce shared buffers by 25-75%.
* Reduce checkpoint timeouts to 3min or less.
* Make sure to do enough ANALYZEs to keep the statistics up to date, manual if required.

### AWS

* Remember that instances can disappear and come back up without storage.
* Always have a good backup / replication implementation on AWS!
* PIOPS are useful, but pricey if using EBS.

### Larger scale AWS Deployments

* Script everything: Instance creation, PostgreSQL setup, etc.
* Put everything inside a VPC.
* Scale up and down as required to meet load.

### PostgreSQL RDS

* Overall, not a bad product.
* Automatic failover.
* Bad performance compared to EC2.
* Expensive.

## Sharding

* Postgres-XL
* Bucardo

## Pooling

* Opening a connection to PostgreSQL is expensive.
* It can easily be longer than the actual query time.
* Above 200-300 connections: use a pooler.
    - pgbouncer: no failover though..
    - pgpool II: higher overhead, more complex to configure


## Monitoring

* Use Nagios / Ganglia to monitor:
    - Disk space
    - CPU usage
    - Memory usage
    - Replication lag

* Log analysis:
    - `pgbadger`.
    - `pg_stat_statements`

## Q&A

* Drop down from python to raw SQL when:
    - Joining three or more tables.
    - Iterating with writes over a queryset.


